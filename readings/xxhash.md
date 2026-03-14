# xxhash

**Project:** [cespare/xxhash](https://github.com/cespare/xxhash)
**Language:** Go (with amd64 and arm64 assembly)
**Size:** 505 lines of Go, 394 lines of assembly

---

Caleb Spare works at Liftoff, where he makes Go servers fast. His most-used project is an implementation of a hash function. These two facts are related.

xxhash implements the XXH64 algorithm — a non-cryptographic hash designed by Yann Collet in 2012 for speed. Not security, not distribution quality, not novelty. Speed. The algorithm processes input in 32-byte blocks using four accumulators, mixed with five prime constants and a specific sequence of multiplies and rotates. It's the hash function inside LZ4, Zstandard, and a long list of databases and caches. Spare's Go implementation is used by Prometheus, Badger, Ristretto, VictoriaMetrics, and FastCache — projects where hashing sits in the hottest path.

The codebase is small enough to fit in a single reading and interesting enough to sustain one, because the 505 lines of Go are not really the point. The point is that those 505 lines exist in three parallel implementations, each making a different bet about what a Go program is allowed to know about the machine it runs on.

## Three Implementations of the Same Algorithm

The build tags tell the story:

**Pure Go** (`xxhash_other.go`): 76 lines. Runs everywhere. The fallback. Uses `encoding/binary.LittleEndian.Uint64` for byte reads, `math/bits.RotateLeft64` for rotations, and trusts the compiler to do the right thing. It triggers when you're on an architecture without assembly support, or when you build with `-tags purego`, or on App Engine where unsafe operations aren't allowed.

**Assembly** (`xxhash_amd64.s`, `xxhash_arm64.s`): 394 lines combined. The performance path. On amd64 and arm64 — which is to say, on nearly every server and laptop that matters — the two functions `Sum64` and `writeBlocks` are implemented in hand-written Plan 9 assembly. The Go files (`xxhash_asm.go`) declare these as stubs with `//go:noescape`, and the linker wires them to the `.s` files.

**Unsafe Go** (`xxhash_unsafe.go`): 58 lines. The strangest of the three. Not an implementation of the algorithm but of string-to-byte-slice conversion, using `unsafe.Pointer` to avoid a copy when hashing strings. Exists because `Sum64([]byte(s))` copies `s`, and for a function that runs billions of times per day across the projects that depend on it, that copy matters.

The pure Go version and the assembly version implement the same algorithm. They produce identical output. The only difference is speed — and the speed difference is large enough that Spare maintains 394 lines of hand-written assembly across two architectures to capture it.

## What the Assembly Buys

The core loop in both implementations does the same thing: load four 64-bit words from the input, feed each through a round function (multiply by prime2, rotate left 31, multiply by prime1), advance the pointer, repeat. In Go:

```go
for len(b) >= 32 {
    v1 = round(v1, u64(b[0:8:len(b)]))
    v2 = round(v2, u64(b[8:16:len(b)]))
    v3 = round(v3, u64(b[16:24:len(b)]))
    v4 = round(v4, u64(b[24:32:len(b)]))
    b = b[32:len(b):len(b)]
}
```

The three-index slicing (`b[0:8:len(b)]`) is a bounds-check elimination trick. By setting the capacity equal to the length, the compiler can prove that subsequent operations won't exceed bounds and elides the check. This is a commit from 2022 — six years after the initial version — suggesting Spare found the bounds checks by profiling and eliminated them one by one.

In assembly, the same loop is:

```asm
loop:
    MOVQ +0(p), x
    round(v1, x)
    MOVQ +8(p), x
    round(v2, x)
    MOVQ +16(p), x
    round(v3, x)
    MOVQ +24(p), x
    round(v4, x)
    ADDQ $32, p
    CMPQ p, end
    JLE  loop
```

No bounds checks to eliminate because there are no bounds. No function call overhead because `round` is a macro. No register allocation decisions because Spare made them:

```asm
#define h      AX
#define p      SI
#define n      DX
#define end    BX
#define v1     R8
#define v2     R9
#define v3     R10
#define v4     R11
#define x      R12
#define prime1 R13
#define prime2 R14
#define prime4 DI
```

Every register has a name. The primes that get reused in the loop (`prime1`, `prime2`) live in registers. The prime that's only used in the tail (`prime4`) uses `DI` — which is also the pointer register in Go's calling convention, repurposed here because `Sum64` is `NOSPLIT|NOFRAME` (no stack frame, no preemption point). This is a function that knows exactly how many registers it has and uses all of them.

The arm64 assembly goes further. Where amd64 loads one 64-bit word at a time with `MOVQ`, arm64 uses `LDP` — load pair — to grab two 64-bit words in a single instruction:

```asm
LDP.P 16(p), (x1, x2)
LDP.P 16(p), (x3, x4)
```

Four words in two instructions. The arm64 version also uses `MADD` (multiply-accumulate) to fuse operations that take two instructions on amd64. And it uses ARM's rotated register operands to combine an XOR and a rotate into a single instruction: `EOR x1 @> 64-27, h, h`. A comment in the code notes that "sequencing the EOR after the ROR (using a rotated register) is worth a small but measurable speedup for small inputs."

The arm64 tail loop is structurally different too. Where amd64 uses comparison and conditional jumps (`CMPQ`/`JG`), arm64 uses `TBZ` — test bit and branch if zero. To check whether there are at least 16 remaining bytes, it tests bit 4 of the length. To check for 8 bytes, bit 3. For 4 bytes, bit 2. This is possible because the tail processing handles power-of-two chunks, and checking a specific bit is cheaper than a comparison on ARM. The amd64 version doesn't do this — it uses conventional comparisons, adjusting the `end` pointer between each section.

## The Unsafe Trick

The most interesting code isn't in the algorithm. It's in `xxhash_unsafe.go`, and it's about the Go compiler's inliner.

The standard way to convert a string to a `[]byte` without copying is through `reflect.SliceHeader`:

```go
var b []byte
bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
bh.Data = (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
bh.Len = len(s)
bh.Cap = len(s)
```

This works but it's expensive in a specific way: the Go compiler's inliner assigns each expression a cost, and this sequence is heavy enough that any function using it won't be inlined. For `Sum64String` — which should be a thin wrapper around `Sum64` — this means a function call at every callsite.

Spare's solution:

```go
type sliceHeader struct {
    s   string
    cap int
}

func Sum64String(s string) uint64 {
    b := *(*[]byte)(unsafe.Pointer(&sliceHeader{s, len(s)}))
    return Sum64(b)
}
```

This exploits Go's memory layout: a string is a pointer and a length. A slice is a pointer, a length, and a capacity. By creating a struct that embeds a string (pointer + length) followed by a cap field, and casting it to `*[]byte`, you get a valid slice header without touching `reflect` at all. The inliner sees a struct literal, a pointer cast, and a function call — all cheap enough to inline.

There's also a separate trick in `WriteString`:

```go
func (d *Digest) WriteString(s string) (n int, err error) {
    d.Write(*(*[]byte)(unsafe.Pointer(&sliceHeader{s, len(s)})))
    return len(s), nil
}
```

The comment explains: "Ignoring the return output and returning these fixed values buys a savings of 6 in the inliner's cost model." `d.Write` always returns `len(s), nil`, but capturing those return values costs the inliner more than hardcoding them. Spare knows the exact cost model and writes code against it.

A test (`TestInlining`) verifies that both functions remain inlineable by running `go build -gcflags="-m"` and parsing the output. If a future Go version changes the inliner's cost model and breaks inlining, the test catches it. The code is written against a specific version of the compiler, and it knows this about itself.

The whole file is guarded by `//go:build !appengine`, with a safe fallback (`xxhash_safe.go`) that just does `Sum64([]byte(s))` — the obvious thing, with the copy. Sixteen lines versus fifty-eight, same interface, different performance characteristics, split by build tag.

## The Eight-Year Arc

The commit history spans from August 2016 to April 2024 — eight years, 60 commits, for 505 lines. The initial commit on August 27, 2016 is a pure Go implementation. The same day: benchmarks, direct `Sum64` implementation (avoiding `New`/`Write`/`Sum64`), bounds check elimination. The next day: assembly for amd64. Three days later: optimizing which primes to keep in registers.

Then quiet. A few commits per year. Each one a specific optimization with benchmarks attached. November 2020: the inlining work. November 2022: arm64 assembly, bounds check removal from the pure Go path, and a flurry of assembly refinement — cleaning up amd64 using ideas from the arm64 implementation.

April 2024: the most recent code change adds seed support, breaking the assumption that zero-seed is the only use case. 32 commits after the initial commit, which is 0.33 commits per month over eight years.

This is what mature infrastructure looks like. Not abandoned — maintained. Not actively developed — complete. Each commit exists because someone measured something and found it wasn't fast enough.

## What the Code Reveals

The code is not self-expressive in the way that a well-designed API is self-expressive. You cannot read `xxhash.go` and understand why the primes are those specific numbers, or why the rotation amounts are 31, 27, 23, 11, and not others. The algorithm was designed by someone else, and this is a faithful implementation. The authorial voice lives not in the algorithm but in the implementation decisions: when to drop into assembly, how to trick the inliner, where to spend a register.

The three-implementation architecture is the thesis. Go promises portability and safety. The pure Go version delivers both. The assembly version breaks portability for speed. The unsafe version breaks safety for speed. Each layer peels back a guarantee that Go provides by default, and each layer is isolated by build tags so the guarantees remain intact for everyone who doesn't need the last 25% of throughput.

The arm64 assembly arrived six years after the amd64 assembly. In 2016, arm64 servers were rare. By 2022, Graviton processors were running a meaningful fraction of AWS. The architecture that required assembly was determined not by the algorithm but by the hardware market. Spare wrote arm64 assembly when arm64 mattered, not before.

And the unsafe string trick — a struct literal that exploits Go's memory layout to avoid a copy, verified by a test that parses compiler output — is the kind of thing that exists because this function runs in the inner loop of Prometheus. Not because it's elegant. Because someone measured, and the copy was there, and removing it mattered.

Caleb Spare's contribution to the Go ecosystem is invisible in the way that good infrastructure is invisible. You don't think about your hash function. You think about the thing you're hashing. But somewhere underneath, four accumulators are spinning through your data in hand-written assembly, on a register map that someone chose by name eight years ago, and the string you passed in didn't get copied because of a struct that lies about being a slice.
