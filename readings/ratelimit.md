# Ratelimit

**[uber-go/ratelimit](https://github.com/uber-go/ratelimit)** — 425 lines of Go across four files

Kris Kowal (Uber), Bohdan Storozhuk, Paweł Krolikowski. 2016–2024.

---

The library contains three implementations of the same interface. This is the reading.

```go
type Limiter interface {
    Take() time.Time
}
```

One method. You call `Take()` before each operation. It blocks until you're allowed to proceed and returns the time at which permission was granted. A leaky bucket: requests drip out at a fixed rate, and if you call faster than the rate, you wait. If you call slower, you may accumulate a small credit — configurable via `slack` — that lets you burst briefly.

Three files implement this interface. They all do the same thing. They disagree about how to do it under contention.

## The Mutex Version (88 lines)

```go
func (t *mutexLimiter) Take() time.Time {
    t.Lock()
    defer t.Unlock()
    // ...
}
```

The simplest version. Lock, compute the sleep duration, sleep while holding the lock, unlock. The algorithm is transparent: track when the last request was permitted, compute how long you should sleep to maintain the rate, clamp the accumulated slack, sleep if needed.

Sleeping while holding the lock is a choice. It means concurrent callers queue behind the mutex, which serializes them — exactly what a rate limiter should do. But it also means the lock is held for the duration of the sleep, not just the duration of the computation. On a 100 RPS limiter, that's 10ms per take. Fine for moderate concurrency. At scale, the contention point is the lock itself — every goroutine that calls `Take` blocks on the same mutex, even if the sleep would have resolved before they'd be serviced.

## The Atomic Pointer Version (110 lines)

```go
type state struct {
    last     time.Time
    sleepFor time.Duration
}

type atomicLimiter struct {
    state unsafe.Pointer
    padding [56]byte
    // ...
}
```

Bohdan Storozhuk added this in 2017 as `#15`. The idea: replace the mutex with a compare-and-swap loop. The state is a struct behind an `unsafe.Pointer`, atomically swapped on each take. No lock contention — if two goroutines race, one wins and the other retries with the new state. The sleep happens *after* the CAS succeeds, outside any critical section.

The `padding [56]byte` is there because without it, the `state` pointer might share a cache line with adjacent memory. On a multi-core machine, every CAS on `state` invalidates that cache line on all other cores. If other data lives on the same line, it gets evicted too — false sharing. The padding pushes everything else to a different cache line. Fifty-six bytes because the pointer is 8 bytes, and a cache line is 64.

The comment says it clearly: `// lint:ignore U1000 Padding is unused but it is crucial to maintain performance`. The linter wants to remove it. The linter is wrong.

But there's a subtlety: each CAS failure allocates a new `state` struct. Under high contention, this creates garbage collection pressure. The `unsafe.Pointer` swap means the old state becomes unreachable and must be collected. The mutex version has no allocations in the hot path.

## The Atomic Int64 Version (92 lines)

This is where the commit history gets interesting.

Commit `f04376c` (Bohdan, 2023): "Implement rate limiter based on atomic int64 operations (#85)."
Commit `8b3fccf` (Paweł, 2023): "Revert 'Implement rate limiter based on atomic int64 operations (#85)' (#91)."
Commit `783ade2` (Bohdan, 2023): "Restore int64 based atomic rate limiter (#94)."
Commit `a12885f` (Bohdan, 2023): "Make int64 based atomic ratelimiter default (#101)."

Added, reverted, restored, promoted. The int64 version is now the default — the one `New()` calls.

The insight: you don't need a struct. The entire state fits in a single `int64` — the Unix nanosecond timestamp of the next permission. No `unsafe.Pointer`, no struct allocation, no GC pressure. Just `atomic.CompareAndSwapInt64`.

```go
type atomicInt64Limiter struct {
    prepadding  [64]byte
    state       int64
    postpadding [56]byte
    // ...
}
```

Note the padding here is different. The pointer version has 56 bytes *after* the pointer. The int64 version has 64 bytes *before* and 56 bytes *after*. The pre-padding is a full cache line, ensuring the `state` field starts at a cache line boundary. The post-padding fills the rest of that cache line. The state lives on its own cache line, isolated from everything.

The `Take` method is a CAS loop with a three-way branch:

```go
switch {
case timeOfNextPermissionIssue == 0 || (t.maxSlack == 0 && now-timeOfNextPermissionIssue > int64(t.perRequest)):
    newTimeOfNextPermissionIssue = now
case t.maxSlack > 0 && now-timeOfNextPermissionIssue > int64(t.maxSlack)+int64(t.perRequest):
    newTimeOfNextPermissionIssue = now - int64(t.maxSlack)
default:
    newTimeOfNextPermissionIssue = timeOfNextPermissionIssue + int64(t.perRequest)
```

First case: first call ever, or no slack mode and we're past due — reset to now. Second case: slack mode but the gap exceeds the max slack — clamp to now minus the max slack. Third case: normal operation — next permission is the previous one plus the per-request interval. If the resulting timestamp is in the future, sleep until then.

The beauty is that all three paths produce a single `int64`, which gets CAS'd into the state. No allocation. No pointer indirection. No GC interaction. The tradeoff compared to the pointer version: slack handling is slightly different (the pointer version accumulates sleep debt across calls; the int64 version works purely from timestamps), and the code is arguably harder to reason about because duration arithmetic is hidden inside int64 subtraction.

## What the Three Versions Teach

The mutex version is correct, obvious, and slow under contention. The pointer version eliminates lock contention but introduces GC pressure. The int64 version eliminates both but compresses the state to the point where the algorithm changes shape — it's the same leaky bucket, but expressed as timestamp arithmetic instead of duration accumulation.

The revert-and-restore cycle in the commit log suggests the int64 version had bugs (the fix in `#95` — "Fix no slack option for int64 based option" — confirms this). Compressing state into a single integer removes slack-handling nuance. The pointer version can store both `last` and `sleepFor` independently. The int64 version has to encode the slack relationship into the single timestamp, and getting that encoding right took three passes through the repository.

All three versions remain in the codebase. The mutex version and pointer version are never constructed by the public API — `New()` calls `newAtomicInt64Based`. They survive as documentation: this is where we came from, and why each step was necessary.

The `unlimited` type at the bottom of `ratelimit.go` is the fourth implementation. Eight lines. `Take() time.Time { return time.Now() }`. The fastest rate limiter is the one that doesn't limit.

---

*Three ways to say "wait your turn." A mutex, a pointer, an integer. The commit log records each attempt and its failure mode, and leaves all of them in the source for the next person to understand why the simplest version isn't good enough.*
