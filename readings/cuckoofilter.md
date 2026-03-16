# Cuckoo Filter

**Project:** [seiflotfy/cuckoofilter](https://github.com/seiflotfy/cuckoofilter)
**Language:** Go
**Size:** 486 lines across 5 files

---

Seif Lotfy's cuckoo filter is a paper transcription with exactly one original decision, and that decision determines everything else. The 2014 CMU paper "Cuckoo Filter: Practically Better Than Bloom" by Fan, Andersen, and Kaminsky presents a family of possible filters parameterized by fingerprint size and bucket size. Lotfy picked one byte. That single commitment — `type fingerprint byte` — collapses the design space into a specific, concrete artifact where the bucket is four bytes wide, the alternate-index table has 256 entries, serialization is a memcpy, and the false positive rate is fixed at roughly 1/255. The entire library is the consequences of that line.

## The one decision Seif actually made

Read the git history and you find a project that arrived nearly complete on July 6, 2015, then entered a decade of maintenance. The bucket size of four comes from the paper. The XOR-based alternate index comes from the paper. The 500-iteration eviction limit comes from the paper. The power-of-two sizing comes from the paper. What Lotfy contributed is in `doc.go`, where he wrote it down explicitly: "a static bucket size of 4 fingerprints and a fingerprint size of 1 byte based on my understanding of an optimal bucket/fingerprint/size ratio from the aforementioned paper."

This is not a criticism. This is the interesting thing. The paper presents tables of tradeoffs — two-byte fingerprints cut false positives, eight-entry buckets change the load factor, semi-sorting compresses storage. Lotfy read those tables and decided that for a general-purpose Go library, the simplest point in the design space was the right one. One byte per fingerprint means the entire data structure is `[]bucket` where `bucket` is `[4]byte`. Four bytes per bucket. No bit-packing, no variable-width encoding, no configuration. You allocate a slice of four-byte arrays and you are done.

The consequence is that the library cannot be tuned. You cannot ask for a lower false positive rate by widening the fingerprint. You cannot trade memory for throughput by changing the bucket width. These are compile-time constants. And because they are constants, every function in the codebase is a linear scan over exactly four elements — small enough that the Go compiler can unroll it, small enough that the programmer never needed to think about loop optimization. The `insert` method on `bucket` checks four slots. The `delete` method checks four slots. The `getFingerprintIndex` method checks four slots. Nobody who reads this code wonders what the loop bounds are.

## A byte buys you a lookup table

The fingerprint-as-byte decision cascades most visibly into `util.go`. The alternate index formula is `i2 = i1 XOR hash(fingerprint)` — the paper's key insight that lets you recover either bucket index from the other using only the stored fingerprint. In a variable-width fingerprint implementation, you'd compute that hash at runtime. With a one-byte fingerprint, there are exactly 256 possible inputs. So Lotfy precomputes all of them:

```go
var altHash [256]uint
```

Populated in `init()`, this table maps every possible fingerprint value to its MetroHash output, masked to the right width. The alternate-index calculation becomes two lookups and an XOR. No hashing on the hot path for the second bucket probe. The table fits in a single cache line or two. This is the kind of optimization that only exists because the fingerprint space is small enough to enumerate exhaustively.

The same narrowness makes the null sentinel work cleanly. Fingerprint zero means "empty slot." To avoid ever generating a zero fingerprint from real data, `getFingerprint` computes `hash % 255 + 1`, mapping the hash output into the range [1, 255]. One value out of 256 is sacrificed, and in exchange, every function that scans a bucket can use `fp != 0` as the emptiness check. No separate occupancy bitmap, no option types, no secondary bookkeeping. The cost is precisely one bit of entropy — the false positive rate shifts from 1/256 to 1/255, a difference that matters to nobody.

## The single-hash trick someone else found

For five years, the `Lookup` and `Insert` paths called the hash function twice: once to get the index, once to get the fingerprint. In October 2020, a contributor named panmari opened a commit with the observation that a 64-bit hash contains enough bits for both. The upper 32 bits give the index; the lower bits give the fingerprint. One hash call instead of two.

This is the optimization Lotfy didn't make, and it's worth noticing why. The original code was correct. It was clear. Two hash calls mapped obviously to two conceptual operations: "where does this item go" and "what does this item look like." Merging them into a single call and splitting the bits requires you to reason about the independence of the upper and lower halves of MetroHash's output. Panmari did that reasoning; Lotfy accepted it. The pattern repeats throughout the commit history: the core author wrote the faithful transliteration, and the community found the engineering optimizations.

The same pattern produced `scalable_cuckoofilter.go`. Seven4X added it in November 2020 — a wrapper that chains multiple filters together, creating a new one when the current filter's load factor exceeds a threshold. The base filter has a fixed capacity set at construction time; the scalable variant grows. This is the feature a library user would eventually need, and it's the feature Lotfy didn't build. He built the data structure. Someone else built the product.

The scalable filter also introduces a quiet inconsistency: it serializes with Go's `gob` encoding, while the base filter uses raw byte dumps. Two serialization strategies in the same package, contributed by different authors, with different tradeoffs. The base filter's `Encode` is beautiful in its simplicity — flatten the bucket array to bytes, done, no headers, no version numbers, the count is reconstructed on decode by counting non-zero entries. The scalable filter needs to encode a slice of filters with metadata, so it reaches for `gob`. Neither author had reason to reconcile with the other's choice. The code accreted rather than being designed as a whole.

## What absence tells you

There are no mutexes. No `sync.RWMutex` wrapping the filter, no atomic operations on the count field, no documentation warning about concurrent access beyond what a reader can infer. `InsertUnique` calls `Lookup` and then `Insert` as two separate operations — a textbook TOCTOU race under concurrency. This is a deliberate absence. The Go standard library's `map` type is also not safe for concurrent use; the caller provides synchronization. Lotfy made the same choice, and it's the right one for a building block. A mutex baked into the data structure forces every single-threaded caller to pay for synchronization they don't need. Leaving it out means the concurrent caller writes `mu.Lock(); filter.Insert(data); mu.Unlock()` and the single-threaded caller writes nothing.

There are also no benchmarks in the repository. For a data structure whose entire reason to exist is performance — cuckoo filters replace Bloom filters when you need deletion support without sacrificing speed — this is a notable gap. Lotfy trusted the paper's analysis. The paper says four-entry buckets with one-byte fingerprints achieve 95% occupancy and sub-microsecond operations. The code implements what the paper specifies. Benchmarking would verify the implementation but wouldn't change the design, and the design is what Lotfy cared about.

## The thesis in the code

Every codebase reveals what its author values by what it refuses to parameterize. Lotfy refused to parameterize fingerprint size, bucket width, eviction limits, and hash functions (that last one held firm for nine years before the community added `SetDefaultHasher` in 2024). The result is a library where the data structure is literally an array of four-byte arrays. There is nothing to configure because there is nothing to get wrong.

This is a specific philosophy of library design: implement the paper's optimal point, not the paper's parameter space. A more ambitious library would let you choose two-byte fingerprints for lower false positive rates, or eight-entry buckets for higher load factors. It would be more useful and harder to use correctly. Lotfy's version gives you one false positive rate (0.39%), one maximum load factor, one eviction strategy, and forces every caller into the same operating point. Twelve hundred people starred it. Debian packaged it. It works.

The most revealing artifact in the repository is the commit frequency. A burst of activity in July 2015, then nothing substantial from the original author for years. The code did not need to change because the decisions were made correctly at the start, and the only decision that was really Lotfy's — one byte — turned out to be the one that made everything else inevitable. When your fingerprint is a byte, your bucket is four bytes, your lookup table is 256 entries, your serialization is a byte dump, and your filter is a flat slice of compact arrays. There is no second design to discover. You read the paper, you pick your point, and if you pick well, the implementation writes itself.

That is what happened here. Lotfy's cuckoo filter is proof that the hardest part of implementing a paper is not the algorithm — it's choosing which version of the algorithm to build, and then having the discipline to build only that.
