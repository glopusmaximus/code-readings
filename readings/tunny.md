# Tunny

**Project:** [Jeffail/tunny](https://github.com/Jeffail/tunny)
**Language:** Go
**Size:** 435 lines across 2 files

---

Tunny is a goroutine pool where the workers tell the pool when they're ready, not the other way around. That inversion — workers push availability rather than pull work — is the decision that explains everything else about the library: why there's no work queue, why cancellation requires three select statements, why `BlockUntilReady` exists as an interface method, and why Ashley Jeffs spent a 2018 rewrite deleting features rather than adding them. The code is 435 lines because that inversion, followed to its consequences, turns out to be enough.

## The pool has no opinion about your work

Most pool implementations put a buffered channel between callers and workers. You submit a job, it enters a queue, a worker eventually picks it up. The pool owns the queue, and the queue is the coordination mechanism. Tunny refuses this. There is no work queue. The `reqChan` on the `Pool` struct is not a channel of jobs — it's a channel of `workRequest` structs, and the workers are the ones sending on it:

```go
// Inside workerWrapper.run(), the worker loop:
select {
case w.reqChan <- workRequest{
    jobChan:       jobChan,
    retChan:       retChan,
    interruptFunc: w.interrupt,
}:
case <-w.closeChan:
    return
}
```

The worker creates a fresh `jobChan` and `retChan` pair, wraps them with its own interrupt function, and pushes the whole bundle onto the shared request channel. On the other side, `Process()` receives this struct:

```go
func (p *Pool) Process(payload interface{}) interface{} {
    request, open := <-p.reqChan
    // ...
    request.jobChan <- payload
    payload, open = <-request.retChan
    // ...
}
```

The caller doesn't submit work to the pool. The caller waits for a worker to show up, then hands it work directly through that worker's private channels. This is a handshake, not a dispatch. The caller blocks until a worker is ready, and the worker blocks until a caller has something to give it.

The consequence is that backpressure is immediate and complete. If all workers are busy, callers block on `<-p.reqChan`. There's no buffer absorbing the pressure. There's no queue growing in the background, hiding the fact that your system is overloaded. `QueueLength()` doesn't count buffered items — it counts goroutines blocked waiting for a worker, which is a fundamentally different measurement. One tells you how deep the hole is. The other tells you how many people are standing at the edge.

This is a real position about what a pool should do. Most pool libraries assume their job is to decouple submission from execution — to give you a place to put work and walk away. Jeffs assumes the opposite: that the moment you submit work, you should feel the system's capacity constraints. If no one is available, you wait. That's not a limitation of the implementation. That's the point.

## Killing async made the library honest

The git history tells the story. The original Tunny (2014) had `SendWork`, `SendWorkTimed`, async job submission — the standard pool API where you fire and forget. The January 2018 rewrite deleted all of it. 778 lines removed, 554 added. The library lost features and got better.

The async API was a lie. If workers push availability and there's no internal queue, then "async submission" means the library has to create a goroutine to wait for a worker on your behalf. You aren't decoupling anything. You're just hiding the blocking behind a goroutine that the pool manages, which means the pool now has to track those goroutines, handle their lifetimes, deal with what happens when you close the pool while submissions are pending. The complexity isn't in the work — it's in managing the illusion that the work was accepted instantly.

The 2018 rewrite confronted this. If the design says callers should feel backpressure, then giving them an API that hides the backpressure is incoherent. `Process()` blocks. `ProcessTimed()` blocks with a deadline. `ProcessCtx()` blocks with a context. Every variant makes the caller wait. The API and the architecture agree with each other.

`Close()` demonstrates the same honesty. It's implemented as `SetSize(0)` followed by closing `reqChan`. The pool doesn't have a "running" or "stopped" flag, no state machine, no mutex-guarded boolean. A pool with zero workers and a closed request channel is a closed pool. This falls out of the design rather than being bolted on. `SetSize` already knows how to add and remove workers. Closing is just removing all of them and then shutting the door.

## Three selects is what cancellation costs when you take it seriously

`ProcessTimed` has three select statements. Not because the author didn't know how to consolidate them, but because there are three distinct phases where a timeout can arrive, and each requires different cleanup:

```go
// Phase 1: waiting for a worker
select {
case request, open := <-p.reqChan:
    // got a worker
case <-tout:
    return nil, ErrJobTimedOut
}

// Phase 2: sending the payload
select {
case request.jobChan <- payload:
    // worker accepted the job
case <-tout:
    request.interruptFunc()
    return nil, ErrJobTimedOut
}

// Phase 3: waiting for the result
select {
case payload, open = <-request.retChan:
    // got the result
case <-tout:
    request.interruptFunc()
    return nil, ErrJobTimedOut
}
```

If you timeout in phase 1, nothing has happened. No worker was claimed, no cleanup needed. If you timeout in phase 2, you've received a worker's `workRequest` but haven't sent it a job yet. The worker is sitting there with an open `jobChan`, waiting. You have to interrupt it — otherwise it hangs forever. If you timeout in phase 3, the worker is actively processing. You interrupt it, which closes its `interruptChan` and calls the `Worker.Interrupt()` method.

This is the honest cost of the handshake model. Because the caller and worker are directly connected through private channels, cancellation at any phase means you have to deal with what the other side is doing at that exact moment. A queue-based pool doesn't have this problem — you just don't dequeue the item. But Tunny's callers are holding a live connection to a specific worker. Walking away from that connection requires telling the worker you're leaving.

`ProcessCtx` (added in 2021 by a contributor) has the same three-select shape but with `ctx.Done()` instead of a timer channel. The pattern was so clearly the right structure that when someone else needed to implement the context variant, they reproduced it exactly. The architecture guided a contributor who wasn't the original author to the same design.

## BlockUntilReady is the interface method that justifies the interface

The `Worker` interface has four methods, and three of them — `Process`, `Interrupt`, `Terminate` — are what you'd expect. The fourth is the interesting one:

```go
type Worker interface {
    Process(interface{}) interface{}
    BlockUntilReady()
    Interrupt()
    Terminate()
}
```

`BlockUntilReady` is called at the top of the worker's run loop, before the worker sends its `workRequest` on `reqChan`. A worker that isn't ready doesn't advertise availability. This turns the push-availability model from a concurrency pattern into an extension point.

A rate-limited worker can sleep in `BlockUntilReady` until its rate window opens. A worker that depends on an external connection can block until the connection is healthy. A worker that processes batches can wait until it has flushed its previous batch. None of these are the pool's concern. The pool doesn't know about rates or connections or batches. It just knows that when a `workRequest` arrives on `reqChan`, the worker behind it is ready.

This is where the inversion pays its largest dividend. In a pull-based pool, if you want workers to pace themselves, you need the pool to understand pacing — or you put the pacing logic inside `Process` and waste a slot on a worker that's accepted a job but isn't actually working on it yet. With push-based availability, the worker doesn't claim a spot in the pool's attention until it's genuinely ready. Backpressure propagates correctly: if all workers are rate-limited, callers block, which is the true state of the system.

The two convenience constructors, `NewFunc` and `NewCallback`, wrap closures in a `Worker` implementation where `BlockUntilReady` is a no-op. Most users never think about it. But the method is the reason `Worker` is an interface instead of a function signature. Without it, `NewFunc(n, func(interface{}) interface{})` would be the entire API and there'd be no need for the abstraction. `BlockUntilReady` is the load-bearing method that justifies the design.

## The interruptChan remake is the one clever trick

There's one moment in the code that feels like a hack until you think about it:

```go
// After handling an interrupt in the worker loop:
w.interruptChan = make(chan struct{})
```

After an interrupt fires, the worker replaces its interrupt channel with a fresh one. This is because `close(chan)` is permanent in Go — a closed channel always reads immediately. If the worker kept the old channel, every subsequent select case on `interruptChan` would fire instantly. The worker would be permanently interrupted.

The alternative would be to use a sync primitive — a mutex-guarded boolean, an atomic flag with a check. But then the interrupt couldn't participate in select statements alongside `jobChan` and `closeChan`. The whole point of using a channel for interruption is that it composes with Go's select. Remaking the channel preserves that composability. It's not elegant, but it's correct, and it keeps the interrupt mechanism in the same idiom as everything else.

This is, in 435 lines, the only place where the code does something that requires a comment to understand. Everything else follows from the push-availability decision with almost mechanical inevitability. The architecture carries the implementation.

## A library that knows what it decided

Tunny is ten years old and stable not because it's abandoned but because it's complete. The push-availability model was a decision, not an accident, and the 2018 rewrite was the moment Jeffs recognized what he'd actually built and stripped away everything that contradicted it. The async API was removed because it fought the architecture. The work queue was never added because the architecture didn't want one. `BlockUntilReady` exists because the architecture created a natural place for it.

The result is a library where every piece explains every other piece. You can start from the `reqChan` inversion and derive the three-select cancellation pattern, the panic in `Process`, the `Close`-as-`SetSize(0)` equivalence, the `Worker` interface shape. You can start from `BlockUntilReady` and work backward to why workers must push availability. The dependency graph of ideas is fully connected.

Four hundred thirty-five lines is not a small library because the problem is simple. It's a small library because the author found the one decision that, if you commit to it fully, makes all the other decisions for you.
