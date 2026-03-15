# Consistent

**Project:** [buraksezer/consistent](https://github.com/buraksezer/consistent)
**Language:** Go
**Size:** 393 lines in a single file

---

Most consistent hashing libraries implement a ring and stop. Hash the key, walk to the nearest node, done. The problem everyone knows about — that adding or removing a server should move as few keys as possible — is solved by virtual nodes and binary search. Clean, well-understood, implemented a hundred times.

Burak Sezer's `consistent` implements the ring, then builds two more mechanisms on top of it, because the ring alone doesn't solve the problem he actually has. The problem is load. A consistent hash ring distributes keys, but it makes no promise about distributing them *evenly*. A node can end up with twice the average, three times, ten times. If you're building a distributed cache — which Burak is, in Olric — that's not a theoretical concern. That's a production incident at 2am.

Google published the fix in 2017: "Consistent Hashing with Bounded Loads." The idea is simple. When placing a key, walk the ring as usual, but if the node you land on is already above a configurable load threshold, skip it and try the next one. This is what Burak implements. The interesting part is how.

## Three Layers in One File

The 393 lines contain three distinct mechanisms, layered on top of each other:

**Layer 1: The ring.** A sorted slice of uint64 hash values (`sortedSet`), a map from those values to members (`ring`), and binary search to find the nearest node for a given hash. Standard consistent hashing. Virtual nodes are created by hashing `member.String() + "0"`, `member.String() + "1"`, etc., with `ReplicationFactor` controlling how many copies of each member appear on the ring.

**Layer 2: Partitions.** Instead of hashing keys directly to ring positions at lookup time, the library introduces a layer of indirection. Keys hash to partition IDs via modulo: `hash(key) % partitionCount`. Each partition is pre-assigned to a member. `LocateKey` becomes two operations:

```go
func (c *Consistent) LocateKey(key []byte) Member {
    partID := c.FindPartitionID(key)
    return c.GetPartitionOwner(partID)
}
```

One hash, one map lookup. No ring walking at lookup time. All the ring-walking happens in `distributePartitions()`, which runs only when you call `Add` or `Remove` — the write path, not the read path. The README makes this explicit: "No memory is allocated by consistent except hashing when you want to locate a key."

**Layer 3: Load balancing.** `distributePartitions` doesn't just assign partitions to their nearest ring position. It enforces the bounded loads constraint. Each partition assignment is routed through `distributeWithLoad`, which walks the ring and skips any member that would exceed the average load ceiling.

Most consistent hashing libraries ship layer 1. Some add layer 2. Burak implements the full paper, and the three layers compose cleanly enough that it all fits in a single file.

## The Heart of the Algorithm

`distributeWithLoad` is where the bounded loads paper becomes code:

```go
func (c *Consistent) distributeWithLoad(partID, idx int, partitions map[int]*Member, loads map[string]float64) {
    avgLoad := c.averageLoad()
    var count int
    for {
        count++
        if count >= len(c.sortedSet) {
            panic("not enough room to distribute partitions")
        }
        i := c.sortedSet[idx]
        member := *c.ring[i]
        load := loads[member.String()]
        if load+1 <= avgLoad {
            partitions[partID] = &member
            loads[member.String()]++
            return
        }
        idx++
        if idx >= len(c.sortedSet) {
            idx = 0
        }
    }
}
```

Start at the ring position where the partition's hash naturally lands. Check if the member there can accept one more partition without exceeding the ceiling. If yes, assign it and increment the load counter. If no, advance to the next position on the ring. Wrap around if you reach the end. If you walk the entire ring and find nowhere to place the partition, panic.

The load ceiling is calculated as the average load per member, multiplied by the load factor, rounded up:

```go
func (c *Consistent) averageLoad() float64 {
    if len(c.members) == 0 {
        return 0
    }
    avgLoad := float64(c.partitionCount/uint64(len(c.members))) * c.config.Load
    return math.Ceil(avgLoad)
}
```

With 271 partitions, 8 members, and load factor 1.25: `271/8 = 33`, `33 * 1.25 = 41.25`, `ceil(41.25) = 42`. No member gets more than 42 partitions. The integer division truncates — `271/8` is 33, not 33.875 — which means the ceiling is slightly lower than a precise calculation would give. This is conservative: members get slightly less room, which means load is slightly more even, at the cost of slightly more ring-walking during redistribution.

## The Deliberate Panic

The panic in `distributeWithLoad` is the most opinionated line in the codebase. Go generally prefers error returns. But `distributePartitions` is called from `Add()` and `Remove()`, which return nothing:

```go
func (c *Consistent) Add(member Member) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if _, ok := c.members[member.String()]; ok {
        return
    }
    c.add(member)
    c.distributePartitions()
}
```

The panic says: if you configured 271 partitions, 2 members, and a load factor of 1.01, that's not a runtime error — it's a programming error. You asked for a mathematically impossible distribution. The comment in the panic handler spells it out: "User needs to decrease partition count, increase member count or increase load factor." Better to crash loudly than silently misassign keys.

There is another panic, equally deliberate, in `New`:

```go
if config.Hasher == nil {
    panic("Hasher cannot be nil")
}
```

Most hashing libraries ship a default hash function. Burak forces you to choose. The README example uses `cespare/xxhash`. The test suite uses `hash/fnv`. The choice matters because the hash function determines distribution quality, and burying that choice behind a default would hide the most important decision the caller makes. This is the same engineering philosophy as the panic: make the programmer confront the decision rather than letting them drift into a bad one.

## Full Redistribution

`Add()` and `Remove()` both call `distributePartitions()`, which redistributes *all* partitions from scratch. Fresh load map, fresh partition map, iterate every partition ID and assign it. No incremental rebalancing.

For a library designed to minimize key redistribution, this is surprising. The whole point of consistent hashing is that adding a node should move only `K/n` keys. But the bounded loads constraint changes the calculus. When you add a member, it's not just the partitions near the new member's ring positions that might move. Reducing one member's load might free capacity that causes a different partition — on the other side of the ring — to snap back to a closer member that was previously full. The cascading effects mean incremental rebalancing would need to track second-order movements, and getting it right would be far more complex than just starting over.

The cost is manageable because you're iterating over partitions, not keys. With `DefaultPartitionCount` at 271, redistribution is 271 iterations of hash-and-walk. The README example with 8 members shows 6% of partitions relocating when a 9th is added. The `LocateKey` benchmarks show 252 nanoseconds per lookup; redistribution happens at topology-change time, not per-request time. The design trades write-path simplicity for read-path speed, and the partition count keeps the write path cheap enough that the trade works.

## The Magic Number 271

`DefaultPartitionCount` is 271, a prime. The comment says: "Prime numbers are good to distribute keys uniformly." This is classic hash table wisdom — modulo by a prime avoids clustering when hash functions have patterns in their lower bits.

But why 271 specifically? The library was extracted from Olric, Burak's distributed in-memory cache (3K+ stars, used in production). The defaults aren't theoretical — they're the values that worked in a real distributed system. The test suite uses 23 partitions (also prime), suggesting the algorithm works at different scales but the defaults target the production case.

271 is small enough that full redistribution is trivially fast. It's large enough to distribute load across a cluster of realistic size. With 20 nodes and 271 partitions, each node gets roughly 13-14 partitions — enough granularity to balance, not so many that the partition map becomes the bottleneck. The number sits at a sweet spot that you find by running a system, not by doing math.

## One Evening in Istanbul

Burak built this in a single evening in March 2018 — first commit at 6:07pm, config integration at 11:34pm, Istanbul time. Then four years of sparse maintenance: an off-by-one fix in `GetClosestN`, a deadlock fix, a divide-by-zero guard for empty rings, default config values. The commit history of a library that was written correctly the first time and needed only the fixes that production surfaced.

The notable users list tells the rest of the story: OpenTelemetry, SeaweedFS, KubeEdge, a blockchain, a serverless cache. Each one needed consistent hashing with a load guarantee, found this library, and never needed to replace it. That's what 393 well-chosen lines can do.
