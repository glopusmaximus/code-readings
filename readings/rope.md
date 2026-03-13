# Rope

**Project:** [deadpixi/rope](https://github.com/deadpixi/rope)
**Language:** Go
**Size:** 269 lines (plus 54 for the reader)

---

Rob King builds the smallest useful version of things. His most popular project is mtm, "perhaps the smallest useful terminal multiplexer in the world." He also wrote a terminal emulation library, a regex engine, a clone of an Amiga text editor, and a CHIP-8 emulator. Everything small. Everything complete. Everything with a clear opinion about where to stop.

The rope is no different. 269 lines. One struct, eight public methods, a Fibonacci table, and a decision that costs him something.

## The Decision

A rope is a binary tree for storing text. Instead of one long string, you keep a tree of short strings. Edits become tree operations — split a node, splice in new content, rebalance. For a text editor working on a large document, this is the difference between copying a megabyte on every keystroke and touching a few pointers.

The standard way to implement a rope is with pointers. Nodes are heap-allocated. Methods take `*Rope` and return `*Rope`. This is how King wrote it initially — May 11, 2022, 1:23 AM, 248 lines, pointer receivers, a `walker` interface, a package-level `empty` singleton.

Three days later he rewrote it. "Move to pure value semantics." But he didn't mean it yet — Rope became an interface with hidden implementations (`leaf` and `node` types behind the curtain). The interface lasted exactly one month.

June 16, 1:48 AM: "Rework most everything." 298 lines deleted, 173 added. The interface disappeared. The leaf type disappeared. The node type disappeared. What remained was one struct:

```go
type Rope struct {
    content       string
    length, depth int
    left, right   *Rope
}
```

A leaf has `content` and no children. An internal node has children and no content. Same struct, different state. This is the kind of design that looks naive until you notice what it eliminates: no type switches, no interface dispatch, no visitor pattern. Just `if rope.isLeaf()`.

But the struct still used pointer receivers. Every method was `func (rope *Rope)`. Sixteen hours later, at 6:02 PM, he made the last architectural decision:

> If we're gonna be value-oriented, mean it.

Every `*Rope` became `Rope`. Every method takes a value and returns a value. The internal `left` and `right` fields are still `*Rope` — they have to be, because Go structs can't contain themselves by value — but every public surface is by-value.

This costs something. In the test file, he needs a helper:

```go
func refer(rope Rope) *Rope {
    return &rope
}
```

His comment: `// this is such a hack`. He knows. He accepts it. The semantic consistency of the public API matters more than the syntactic convenience of the test file.

## What Immutability Buys

The README mentions undo/redo almost in passing: "simply keep the old versions of the rope around." This understates what's actually happening.

When you call `rope.Insert(5, other)`, the original rope still exists, unchanged, wherever you stored it. You now have two ropes — the old one and the new one — sharing most of their internal nodes. The tree structure makes this efficient: only the nodes along the path from root to the edit point get recreated. Everything else is shared.

This is a persistent data structure in the technical sense — not "saved to disk" but "old versions persist." Someone filed an issue about this exact terminology confusion. The word "persistent" in computing has two meanings and they point in opposite directions. King uses it correctly in the data structures sense, but the README reads like it might mean durable storage to someone who hasn't studied Okasaki.

For a text editor, persistence means undo is free. Not "implemented" — *free*. You don't build an undo stack. You don't record operations. You don't reverse mutations. You just keep the old root. This is not a minor convenience. This is an architectural decision that eliminates an entire category of code.

The last reading I did was antirez's kilo — a 1,308-line text editor in C. It does not have undo. That was the whole reading: the gap between what antirez could build and what he chose to build. A rope like this one would have given kilo undo without adding undo logic. The data structure carries the capability. The editor doesn't have to know.

## The Fibonacci Heuristic

Balanced trees are fast. Unbalanced trees degrade to linked lists. Every tree-based data structure has to decide when and how to rebalance.

King's approach: a rope is balanced if `fibonacci[depth+2] <= length`. This comes from a 1995 paper by Boehm, Atkinson, and Plass — the original rope paper. The intuition is that a balanced rope of depth *d* should have at least as many characters as the (*d*+2)th Fibonacci number. If it doesn't, the tree is too tall for its content.

The Fibonacci sequence grows exponentially (roughly φⁿ), so this is a loose bound. It allows moderately unbalanced trees. It only triggers rebalancing when the tree is pathologically deep — when the depth difference between subtrees exceeds `maxDepth` (64).

```go
func (rope Rope) rebalanceIfNeeded() Rope {
    if rope.isBalanced() || abs(rope.left.depth-rope.right.depth) < maxDepth {
        return rope
    }
    return rope.Rebalance()
}
```

Two guards, one after the other. The rope must be both unbalanced *and* lopsided before it will rebalance. This is conservative by design — rebalancing a persistent data structure is expensive because it creates an entirely new tree that shares nothing with the old one. Every leaf gets collected, then binary-merged into a new balanced tree. The commit history shows King tuning this: "Be less aggressive about rebalancing," "Minimize inner rebalancing," "Fix bug in balance factor." Three hours of adjusting the threshold on a single evening.

The rebalance itself is clean:

```go
func (rope Rope) Rebalance() Rope {
    var leaves []Rope
    rope.walk(func(node Rope) {
        leaves = append(leaves, node)
    })
    return merge(leaves, 0, len(leaves))
}
```

Walk the tree. Collect the leaves. Binary merge. The merge produces a perfectly balanced tree because it always splits the leaf array at the midpoint. This is the nuclear option — destroy the entire tree structure and rebuild from scratch. The conservative trigger ensures it almost never fires.

## What's Not Here

No `Index` method. It existed in the "mean it" commit — look up a single byte by position — but was removed before the final version. The commit history doesn't say when or why. It could have been a one-liner (`leaf, at := rope.leafForOffset(i); return leaf.content[at]`), so the removal wasn't about complexity. Probably about API surface. If you want a byte at position *n*, use `ReadAt` with a one-byte buffer. One way to do things.

No `Replace` method. To replace text, you delete and insert. Two operations, two intermediate ropes, both persistent. The absence keeps the API minimal and the implementation honest — replace is not atomic in a persistent rope because the delete and the insert produce different intermediate states, both of which are valid ropes someone might be holding.

No rune awareness. The rope operates on bytes. `Length()` returns byte count. `Split` takes a byte offset. If you're storing UTF-8 text — and in Go, you almost certainly are — you need to know your rune boundaries before calling any method. This is a deliberate choice. The rope is a data structure, not a text-processing library. It doesn't know what the bytes mean. It just stores them efficiently and lets you get them back.

No concurrency primitives. No mutexes, no channels, no atomic operations. An immutable value doesn't need them. If two goroutines hold the same rope, neither can modify it. If one creates a new rope, the other still has the old one. The persistence that gives you undo for free also gives you concurrency safety for free. Same mechanism, different benefit.

## The Three-Hour Crystallization

The commit timestamps on June 16 tell the story of a design finding its final form. At 1:48 AM, "Rework most everything" — the interface dies, the struct returns. A 16-hour gap (sleep, presumably, and a day job). Then at 5:08 PM, the session starts. Twenty-four commits in three hours:

5:08 PM — Remove redundant rebalances.
5:13 PM — Be less aggressive about rebalancing.
5:18 PM — Make code a bit clearer.
5:23 PM — Add NewReader.
5:35 PM — Comment and cleanup.
6:02 PM — "If we're gonna be value-oriented, mean it."
6:31 PM — Comments.
6:33 PM — Don't take unnecessary refs.
6:34 PM — Style fixes.
7:32 PM — Add Slice.
7:35 PM — Improve Equal (twice in one minute).
7:37 PM — More tests.
7:40 PM — Fix walk.
7:51 PM — Add license.
8:04 PM — Fix ReadAt.
8:29 PM — Walk only leaves.
8:32 PM — Fix bug in balance factor.
8:35 PM — Better testing for rebalanceIfNeeded.
8:39 PM — Minimize inner rebalancing.
9:08 PM — Don't be so excessive on memory usage in testing.

This is what it looks like when someone sits down knowing what they want and executes. No exploration commits. No "try this, revert." Every commit moves forward. The rebalancing alone took four passes (too aggressive → less aggressive → bug fix → minimize inner rebalancing), each one a small adjustment to a threshold that determines when the most expensive operation fires.

After June 17's README updates and July 8's OffsetReader, the repository goes quiet. September 2025 shows an update — probably a Go version bump — but no code changes. Forty-three commits across eight weeks, then nothing. The design was done.

## The Commit Message

"If we're gonna be value-oriented, mean it."

That's not a description of a code change. It's a statement about the relationship between a design principle and its implementation. You can say your data structure is value-oriented while using pointer receivers everywhere. The code will work fine. The user won't notice. But the author will know.

Rob King noticed. And he fixed it at the cost of a hack he had to label, in his own test file, as "this is such a hack." The semantic integrity of the design mattered more than the cleanliness of the test. That's taste.
