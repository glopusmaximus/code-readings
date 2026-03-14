# Groupcache

**Project:** [golang/groupcache](https://github.com/golang/groupcache)
**Language:** Go
**Size:** ~1,660 lines (excluding generated protobuf)

---

Brad Fitzpatrick wrote memcached in 2003. Ten years later he wrote its replacement in Go, and the replacement is smaller than most people's configuration files. Groupcache is a distributed cache where the clients are the servers, there is no separate daemon, no configuration file, no persistence to disk, no eviction policy beyond LRU, no way to update a cached value, and no way to delete one. It is 1,660 lines across eight files, and it contains five ideas that each became their own standard library pattern.

Fitzpatrick open-sourced it on July 23, 2013, while on the Go team at Google. In the twelve years since, it has received 70 commits, 13,300 stars, and exactly one architectural change (swapping in consistent hashing, contributed three months after release). Everything else was spelling fixes and Go version bumps. The code was done when it shipped.

## Five Subsystems, Each Complete

### Singleflight (64 lines)

The most influential piece of groupcache isn't in groupcache. `singleflight` was extracted into the Go standard library as `golang.org/x/sync/singleflight` and now ships with every Go installation. The original is 64 lines:

```go
type call struct {
    wg  sync.WaitGroup
    val interface{}
    err error
}

type Group struct {
    mu sync.Mutex
    m  map[string]*call
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        g.mu.Unlock()
        c.wg.Wait()
        return c.val, c.err
    }
    c := new(call)
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()

    c.val, c.err = fn()
    c.wg.Done()

    g.mu.Lock()
    delete(g.m, key)
    g.mu.Unlock()

    return c.val, c.err
}
```

The idea: if ten goroutines ask for the same key at the same time, only one does the work. The other nine wait on a `sync.WaitGroup` and receive the same result. The map holds in-flight calls. The mutex protects only the map operations — the actual work (`fn()`) runs without the lock held. When the work completes, `wg.Done()` wakes all waiters, then the entry is deleted from the map.

The map is lazily initialized (`if g.m == nil`). This is a pattern that appears in every subsystem of groupcache — nothing allocates until first use. It means a zero-value `Group` is valid, which means you don't need constructors, which means the API surface is smaller.

The elegance is in what's not here. No timeout. No cancellation. No error handling beyond passing through `fn()`'s error. No deduplication window — the suppression lasts exactly as long as the function call. No retry. The caller gets exactly one execution per key per concurrent overlap window, which is the only guarantee that matters for a cache fill.

### Consistent Hash (81 lines)

```go
type Map struct {
    hash     Hash
    replicas int
    keys     []int // Sorted
    hashMap  map[int]string
}
```

A hash ring. Each server gets `replicas` virtual nodes on the ring (default: 50). To find which server owns a key, hash the key, binary search the sorted ring for the next virtual node, look up which server owns it. The virtual nodes spread load even when servers have different capacities or when the number of servers is small.

The implementation uses `sort.Search` for the binary search — a standard library function that takes a predicate. Originally it was a linear scan; someone contributed the binary search in June 2014. The hash function defaults to CRC32 and is injectable. Eighty-one lines, no concurrency primitives (the caller handles locking), no removal (you recreate the ring with a new set of servers).

No removal is a design choice, not an omission. When a server leaves the pool, you call `Set` with the new list, which replaces the entire ring. This avoids the complexity of removing virtual nodes from a sorted slice and the hashMap. It's O(n·replicas) to rebuild, but n is the number of servers, which is small. Rebuilding is simpler than removal and produces the same result.

### LRU Cache (133 lines)

```go
type Cache struct {
    MaxEntries int
    OnEvicted  func(key Key, value interface{})
    ll         *list.List
    cache      map[interface{}]*list.Element
}
```

A linked list and a hash map. `Get` moves the accessed element to the front of the list. `Add` pushes to the front and evicts from the back if over capacity. `RemoveOldest` removes from the back. This is the textbook LRU implementation, using Go's `container/list` for the doubly-linked list.

The LRU cache is not safe for concurrent access. The comment says so on line 23. This is deliberate — the groupcache `cache` wrapper (in `groupcache.go`) adds its own `sync.RWMutex` and byte-counting logic. The LRU itself is just the eviction policy, isolated from everything else. The byte counting, the hit/miss stats, the eviction callbacks — all live one layer up.

The `OnEvicted` callback is used by the wrapper to decrement the byte counter when entries are evicted. This is the only connection between the LRU and the memory budget. The LRU doesn't know about bytes; it only knows about entry count. The wrapper doesn't know about list ordering; it only knows about bytes. Clean separation.

### ByteView (175 lines)

```go
type ByteView struct {
    b []byte
    s string
}
```

An immutable view of bytes that internally holds either a `[]byte` or a `string`. Every method checks which field is populated and dispatches accordingly. `Len()` returns `len(v.b)` or `len(v.s)`. `String()` returns `string(v.b)` or `v.s`. `Copy()` calls `copy(dest, v.b)` or `copy(dest, v.s)`.

This exists because of a fundamental tension in Go: strings are immutable, byte slices are mutable, and converting between them copies. A cached value should be immutable — you don't want one caller modifying a value that another caller received. But the data might arrive as either `[]byte` (from a network response) or `string` (from a local computation). ByteView holds whichever representation it received, avoiding the conversion cost, and presents an immutable interface regardless.

The `ByteSlice()` method always copies: `cloneBytes(v.b)` or `[]byte(v.s)`. If you want the raw bytes, you pay for a copy. If you want a string, and the data is already a string, you pay nothing. This asymmetry is intentional — strings are safe to share; byte slices aren't.

### The Orchestration (502 lines)

The core `Group` type ties everything together:

```go
type Group struct {
    name       string
    getter     Getter
    peers      PeerPicker
    cacheBytes int64
    mainCache  cache
    hotCache   cache
    loadGroup  flightGroup
}
```

Two caches, not one. `mainCache` holds values that this peer owns — keys that consistent-hash to this server. `hotCache` holds values fetched from other peers that are popular enough to warrant local replication. The hot cache prevents a single popular key from saturating a peer's network card.

The hot cache population uses a probabilistic approach: 10% of remote fetches are stored locally. The comment says `// TODO(bradfitz): use res.MinuteQps or something smart`. That TODO has been open for twelve years. The 10% heuristic works well enough that no one has needed to implement something smarter.

The eviction policy between the two caches is one line of insight:

```go
victim := &g.mainCache
if hotBytes > mainBytes/8 {
    victim = &g.hotCache
}
```

The hot cache is allowed to grow up to 1/8 of the main cache before it starts getting evicted. This ensures that most of the memory budget goes to authoritative data (which only this peer can serve) rather than replicated data (which other peers can also serve). If the hot cache exceeds the ratio, it becomes the eviction target. Simple, effective, documented as "good-enough-for-now logic."

The `load` method is where everything converges:

1. Check the cache (might have been filled by another goroutine while we were waiting)
2. Use singleflight to ensure only one load per key
3. Inside singleflight: check the cache *again* (a concurrent request might have filled it between our cache miss and our singleflight entry)
4. Ask the consistent hash which peer owns this key
5. If it's a remote peer, fetch over HTTP
6. If it's us (or the remote fetch failed), call the user's Getter
7. Populate the appropriate cache

The double cache check (steps 1 and 3) exists because singleflight only deduplicates concurrent requests, not sequential ones. The code includes a 14-line comment with a numbered sequence diagram explaining exactly how two goroutines can both miss the cache and both enter `load`. This is a comment written by someone who has debugged this race condition.

## What's Not Here

No TTL. No expiration. No way to invalidate a cache entry. The `Getter` interface requires that "key must uniquely describe the loaded data, without an implicit current time." If your data changes, your keys must change. This is not a limitation — it's the central design constraint. It means the cache never serves stale data, which means there's no need for invalidation, which means there's no distributed invalidation protocol, which means the system is simpler by an entire class of complexity.

No writes. Groupcache is read-only. You can't set a value — you can only define a `Getter` that loads it. This means there's no write-through, no write-behind, no consistency protocol for writes. The only consistency question is whether a fill is in progress, which singleflight handles.

No cluster membership protocol. You call `Set` with the list of peers. How you discover peers is your problem. How you detect failures is your problem. Groupcache handles data sharding and deduplication. The operational concerns — health checks, rolling restarts, service discovery — belong to the infrastructure layer.

## The Voice in the Code

The TODO comments are Brad Fitzpatrick at his most characteristic:

```go
// TODO(bradfitz): log the peer's error? keep
// log of the past few for /groupcachez?  It's
// probably boring (normal task movement), so not
// worth logging I imagine.
```

```go
// TODO(bradfitz): use res.MinuteQps or something smart to
// conditionally populate hotCache.  For now just do it some
// percentage of the time.
```

These are not deferred work items. They're the author thinking out loud about things he considered and decided against, preserved in the code so that the next person who has the same idea can see that it was already considered. The logging TODO has been open for twelve years because the conclusion ("probably boring") was already correct. The MinuteQps TODO has been open for twelve years because the 10% heuristic was already good enough.

The Sink interface in `sinks.go` has its own self-conversation:

```go
// if this code ever ends up tracking that at least one set*
// method was called, don't make it an error to call set
// methods multiple times. Lorry's payload.go does that, and
// it makes sense.
```

"Lorry" is presumably a colleague at Google who was using the internal version. The comment captures a design decision made by observing how the code was actually used, not how it was theoretically supposed to be used. It's been part of the public codebase since day one, a piece of Google's internal usage patterns preserved in an open-source README that no one ever wrote.

## The Memcached Replacement

Fitzpatrick built memcached as a separate daemon: start the daemon, configure clients to connect to it, manage the daemon's lifecycle independently of your application. Groupcache inverts this. The cache is a library. Your application is the cache server. There is no daemon to manage, no connection pool to configure, no operational overhead beyond the application itself.

This inversion eliminates an entire category of operational problems — version skew between client and server, separate scaling of cache and application tiers, connection management, monitoring, deployment. But it creates a constraint: the cache is coupled to the application's lifecycle. You can't scale the cache independently. You can't flush it without restarting the application. You can't share it across applications in different languages.

Fitzpatrick knew this. He built memcached for one set of constraints (shared cache across polyglot services at LiveJournal) and groupcache for a different set (embedded cache in a Go monoculture at Google). The design doesn't abstract over both use cases. It picks one and executes it in 1,660 lines.

Twelve years, 70 commits, and the code still works exactly as it was designed to work. The 10% heuristic for the hot cache is still 10%. The singleflight pattern is now in the standard library. The TODO comments are still open, still correct, still not worth implementing. This is what it looks like when someone who has already built the complex version builds the simple one.
