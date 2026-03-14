# xsync

**Project:** [puzpuzpuz/xsync](https://github.com/puzpuzpuz/xsync)
**Language:** Go
**Size:** 2,072 lines across 7 files

---

Andrei Pechkurov builds concurrent data structures that solve the same problem five different ways. xsync is a collection of six things — a counter, a mutex, three queues, and a hash map — that each address the same underlying issue: the standard library's synchronization primitives, designed for correctness and generality, leave performance on the table when you know something specific about your access pattern.

The library has 1,600 stars on GitHub and ships inside projects like Otter (a high-performance cache) and various production systems where `sync.Map` and `sync.RWMutex` became bottlenecks. Pechkurov writes about the internals on his blog at puzpuzpuz.dev, which means the code comes with its own commentary — a rare thing for a library that lives in the gap between textbook concurrency and Go's runtime.

## The Foundation: Cache Lines and False Sharing

Every data structure in xsync is built on top of a 67-line utility file that establishes the physics of the library. Two constants and five functions define the rules:

```go
const cacheLineSize = 64
```

A CPU cache line is 64 bytes on most modern hardware. When two goroutines on different cores modify data that lives on the same cache line, the cores must synchronize — even if they're modifying different fields of different structs. This is false sharing, and it can turn theoretically parallel work into effectively serial work.

The comment says: "64B are used instead of 128B as a compromise between memory footprint and performance; 128B usage may give ~30% improvement on NUMA machines." Pechkurov measured both. He chose the smaller number. Every padded struct in the library trades 56-60 bytes of wasted space per instance for the guarantee that no two hot fields share a cache line.

The other critical function:

```go
//go:noescape
//go:linkname runtime_cheaprand runtime.cheaprand
func runtime_cheaprand() uint32
```

This reaches into Go's runtime to use its internal fast random number generator — faster than `math/rand` because it doesn't need to be cryptographically strong or even statistically excellent. It just needs to distribute goroutines across stripes. The `//go:linkname` directive is a trick: it aliases a private runtime function into the package's namespace. It's fragile — the Go team can rename or remove `cheaprand` at any point — but it's fast. Pechkurov switched to this from `fastrand` in September 2025 when the Go runtime renamed the function.

## Counter: The Simplest Idea (99 lines)

The counter is the clearest expression of xsync's core thesis: if you can tolerate a slightly stale read, you can make writes much faster.

```go
type Counter struct {
    stripes []cstripe
    mask    uint32
}

type cstripe struct {
    c int64
    _ [cacheLineSize - 8]byte
}
```

Instead of a single `atomic.Int64` that every core fights over, the counter spreads its value across multiple stripes — one per CPU, each on its own cache line. To increment:

```go
func (c *Counter) Add(delta int64) {
    t, ok := ptokenPool.Get().(*ptoken)
    if !ok {
        t = new(ptoken)
        t.idx = runtime_cheaprand()
    }
    for {
        stripe := &c.stripes[t.idx&c.mask]
        cnt := atomic.LoadInt64(&stripe.c)
        if atomic.CompareAndSwapInt64(&stripe.c, cnt, cnt+delta) {
            break
        }
        t.idx = runtime_cheaprand()
    }
    ptokenPool.Put(t)
}
```

Pick a random stripe. Try to CAS (compare-and-swap). If it fails — because another goroutine hit the same stripe — pick a different random stripe and try again. The `ptoken` is pooled via `sync.Pool` to avoid allocating on every call.

To read the counter's value, sum all stripes. The result might miss a concurrent increment, but it's eventually consistent. This is Java's `LongAdder` translated to Go, minus the thread-local optimization that Java has and Go doesn't. The random stripe selection is the best Go can do without actual thread-local storage — hence Pechkurov's blog post titled "Thread-local State in Go, Huh?"

## RBMutex: The Academic Paper (188 lines)

The reader-biased mutex is the most intellectually ambitious piece of the library. It implements a modified version of BRAVO (Biased Locking for Reader-Writer Locks), a 2019 paper from Dave Dice of Oracle Labs. The paper's key insight: most reader-writer locks pay a cost on every read lock even when writes are rare.

```go
type RBMutex struct {
    rslots       []rslot
    rmask        uint32
    rbias        int32
    inhibitUntil time.Time
    rw           sync.RWMutex
}
```

When reader bias is active (`rbias == 1`), readers don't touch the underlying `sync.RWMutex` at all. Instead, they atomically increment a slot counter — each reader gets a random slot, each slot is on its own cache line. No shared state between readers. The fast path:

```go
func (mu *RBMutex) fastRlock() *RToken {
    if atomic.LoadInt32(&mu.rbias) == 1 {
        t := rtokenPool.Get().(*RToken)
        for i := 0; i < len(mu.rslots); i++ {
            slot := t.slot + uint32(i)
            rslot := &mu.rslots[slot&mu.rmask]
            if atomic.CompareAndSwapInt32(&rslot.mu, rslotmu, rslotmu+1) {
                if atomic.LoadInt32(&mu.rbias) == 1 {
                    return t
                }
                atomic.AddInt32(&rslot.mu, -1)
                return nil
            }
        }
    }
    return nil
}
```

The double-check on `rbias` is critical: between the CAS success and the bias check, a writer might have disabled bias. If so, the reader rolls back its slot increment and falls through to the standard `sync.RWMutex` path.

When a writer arrives, it sets `rbias = 0`, acquires the standard write lock, then spin-waits for all reader slots to drain to zero. The writer measures how long this drain takes and sets `inhibitUntil` — a timestamp after which reader bias can be re-enabled:

```go
mu.inhibitUntil = time.Now().Add(time.Since(start) * nslowdown)
```

The inhibit period is 7× the drain time. This is the BRAVO paper's core mechanism: after a write, suppress the reader bias long enough that the writer's cost is amortized. If writes are rare, the inhibit period is short (fast drain = short penalty). If writes cause long drains, the inhibit grows, and readers stay on the standard path longer. The system self-tunes.

The constant `nslowdown = 7` is labeled "slow-down guard" with no further explanation. It comes from the paper's empirical tuning. Pechkurov trusts the paper's number.

## Three Queues: Three Access Patterns

The queues are the clearest demonstration that there is no single best concurrent data structure — only specific ones for specific constraints.

**SPSCQueue** (98 lines): One producer, one consumer. Based on Erik Rigtorp's ring buffer design. The key optimization: cached indices. The producer caches the consumer's index, and vice versa. Instead of reading the other side's atomic index on every operation, they check the cached value first. Only when the cached value says "full" (or "empty") do they reload the atomic. This turns most operations into local cache hits.

```go
func (q *SPSCQueue[I]) TryEnqueue(item I) bool {
    idx := atomic.LoadUint64(&q.pidx)
    next_idx := idx + 1
    cached_idx := q.ccachedIdx  // local read, no atomic
    if next_idx == cached_idx {
        cached_idx = atomic.LoadUint64(&q.cidx)  // only on suspected full
        q.ccachedIdx = cached_idx
        if next_idx == cached_idx {
            return false
        }
    }
    q.items[idx] = item
    atomic.StoreUint64(&q.pidx, next_idx)
    return true
}
```

The padding between `pidx`, `pcachedIdx`, `cidx`, and `ccachedIdx` ensures each lives on its own cache line. Four fields, four cache lines, 256 bytes of padding for a queue that handles millions of operations per second.

**MPMCQueue** (107 lines): Multiple producers, multiple consumers. Based on Rigtorp's C++ MPMC queue. Uses a turn-based protocol: each slot has a `turn` counter. A producer can write to a slot only when the slot's turn matches the expected value. A consumer can read only when the turn matches a different expected value. This eliminates the need for a global lock while preventing two producers from writing to the same slot:

```go
turn := q.turn(head) * 2
if slot.turn.Load() == turn {
    if atomic.CompareAndSwapUint64(&q.head, head, head+1) {
        slot.item = item
        slot.turn.Store(turn + 1)
        return true
    }
}
```

The turn alternates: even turns are for producers, odd turns are for consumers. A slot becomes writable on turn 0, readable on turn 1, writable again on turn 2, and so on. The CAS on `head` resolves contention between multiple producers — exactly one wins, writes its item, advances the turn.

**UMPSCQueue** (151 lines): Unbounded multi-producer, single consumer. The most complex of the three. Where the bounded queues use ring buffers, this one uses a linked list of segments — each segment is a slice of 4,096 `queueValue` entries. Multiple producers atomically increment a segment index to claim a slot, write their value, then signal readiness via a `sync.WaitGroup`. The consumer reads slots in order, blocking on each WaitGroup until the value is ready.

```go
type queueValue[T any] struct {
    item  T
    ready sync.WaitGroup
}
```

Using a WaitGroup for per-slot synchronization is unusual. Most lock-free queues use atomic flags or spinning. Pechkurov uses WaitGroup because it integrates with Go's goroutine scheduler — `Wait()` parks the goroutine instead of spinning, which is cheaper when the consumer outruns the producers. The trade-off: each slot costs a WaitGroup, which is 12 bytes of state plus the runtime overhead of Add/Wait/Done. For a segment of 4,096 entries, that's meaningful memory. But the alternative — spinning — burns CPU cycles.

The segment pool recycles value slices when the consumer finishes reading a segment. The segment struct itself can't be reused because its `nextOnce` is spent and its `next` pointer can't be safely reset while writers might still hold references. This is a subtle distinction: the data is recyclable, the coordination mechanism isn't.

## The Map: 1,362 Lines in One File

The Map is the centerpiece — larger than all other data structures combined. It implements a concurrent hash table based on CLHT (Cache-Line Hash Table), a design from EPFL. The map is too large for a line-by-line reading, but the design choices are visible in the constants:

```go
entriesPerMapBucket = 5  // 5 entries fit in one 64-byte cache line
mapShrinkFraction  = 128
mapLoadFactor      = 0.75
```

Five entries per bucket is not arbitrary — it's the number that makes a bucket exactly 64 bytes on a 64-bit machine. One hash, one key, one value times five, packed into a single cache line. A lookup reads one cache line, checks up to five entries, and either finds the key or follows a chain pointer.

Recent commits (2025-2026) show the map's evolution: parallel resize, cooperative rehashing, lock-free shrink operations. The map grew from 800 lines in 2021 to 1,362 lines in 2026, gaining features while maintaining the same external interface. This is the opposite trajectory of the other data structures, which have been stable since their introduction.

## What the Code Reveals

Every data structure in xsync cites its source. BRAVO for the mutex. Rigtorp for the queues. CLHT for the map. Java's `LongAdder` (implicitly) for the counter. Pechkurov is not inventing — he's translating. The academic papers solve the concurrency theory. The Go runtime solves the memory model. Pechkurov solves the gap between them: how do you implement a cache-line-aware data structure in a language that doesn't expose cache lines? How do you approximate thread-local storage in a language that doesn't have threads?

The answer, repeated across every data structure: `runtime_cheaprand()` for stripe selection, `sync.Pool` for token recycling, cache-line padding for isolation, and CAS loops for lock-free progress. These four techniques appear in the counter, the mutex, the queues, and the map. They're the vocabulary of Go concurrency when you've outgrown the standard library.

The library has been actively developed for five years — June 2021 to February 2026 — with the map receiving the most attention. The simpler data structures (counter, queues) have been stable since 2021. The mutex was stable until 2024 when `TryRLock` was added. The map is still evolving, growing a cooperative resize protocol in September 2025 that lets multiple goroutines participate in rehashing rather than blocking on a single resizer.

The commit that adds `runtime_cheaprand` in September 2025 — replacing the previous `fastrand` — is a one-line change that took Go's runtime function rename and adapted. This is the cost of reaching into the runtime: you ship faster code, but you accept maintenance obligations that most libraries never face. Pechkurov accepts this, and the `//go:linkname` directive sits at the bottom of `util.go`, quietly coupling the library's performance to Go's internal implementation details, one version at a time.
