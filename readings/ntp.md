# NTP

**Project:** [beevik/ntp](https://github.com/beevik/ntp)
**Language:** Go
**Size:** 1,092 lines across 2 files

---

The most honest way to ask for the time is to lie about yours.

That is the design thesis of Brett Vickers's NTP library, and it is not a metaphor. When the client sends a query to an NTP server, it fills the TransmitTime field — the field that should contain the client's current time — with cryptographically random bytes. The server echoes this nonce back. The client checks the echo to confirm the response isn't spoofed, then overwrites it with the real transmit time for the offset calculation. The wire never carries the client's actual clock. Your clock is none of the server's business.

This single decision, pulled from an IETF data-minimization draft, is the library in miniature. Every design choice in these 1,092 lines is about the gap between what the protocol asks you to reveal and what you actually need to reveal, between what the server tells you and what you can trust, between what the spec describes and what matters when you're sending one packet and computing an offset.

## Lying to the server is a security decision; lying to yourself is a math one

The TransmitTime lie works because the NTP offset calculation doesn't care about absolute timestamps — it cares about differences. The four timestamps in the classic NTP exchange are: when the client sent, when the server received, when the server replied, when the client received. The offset formula cancels out everything except the asymmetry. So the client can record its real transmit time locally, send garbage on the wire, and substitute the real value after the response arrives. The math doesn't know.

```go
// To help prevent spoofing and client fingerprinting, use a
// cryptographically random 64-bit value for the TransmitTime. See:
// https://www.ietf.org/archive/id/draft-ietf-ntp-data-minimization-04.txt
bits := make([]byte, 8)
_, err = rand.Read(bits)
```

But the substitution happens with a line that deserves attention:

```go
// Correct the received message's origin time using the actual
// transmit time.
recvHdr.OriginTime = toNtpTime(xmitTime)
```

The server echoed back the random nonce as OriginTime. After validation, the client replaces it with the real local timestamp. From this point on, the response header is a fiction — it no longer matches what the server sent. The code comments this without apology. The header is not a historical record; it is an intermediate data structure being prepared for arithmetic.

## Twelve years early is exactly on time

```go
var (
    ntpEra0 = time.Date(1900, 1, 1, 0, 0, 0, 0, time.UTC)
    ntpEra1 = time.Date(2036, 2, 7, 6, 28, 16, 0, time.UTC)
)
```

NTP timestamps are 64-bit fixed-point seconds since January 1, 1900. The 32-bit seconds field rolls over on February 7, 2036. In June 2024, Vickers added era-rollover handling: if a raw timestamp suggests a year before 1970, the code assumes NTP era 1 (starting 2036) and adds the duration to the era 1 epoch instead of the era 0 epoch.

```go
func (t ntpTime) Time() time.Time {
    const t1970 = 0x83aa7e8000000000
    if uint64(t) < t1970 {
        return ntpEra1.Add(t.Duration())
    }
    return ntpEra0.Add(t.Duration())
}
```

The heuristic is elegant because of what it assumes: no NTP timestamp you encounter in practice should represent a time before 1970. Unix didn't exist. Networks didn't exist. If the raw bits decode to pre-1970, they must be post-rollover values from era 1. The window of ambiguity is the 70-year span from 1900 to 1970 — a window where NTP servers were not operational and therefore no legitimate timestamp could originate.

Vickers committed this twelve years before the rollover will happen. The commit message says so explicitly. This is the kind of maintenance that only makes sense if you expect to be maintaining the library in 2036 — or if you expect someone else will be, and you want them to find it already handled. Either way it reflects a relationship with time that is itself temporal: you write the fix when you understand the problem, not when the problem arrives.

## One packet is a whole philosophy

The library sends one UDP packet and computes an offset. It does not maintain state between queries, track multiple servers, weight responses, or implement the Mills clock discipline algorithm. The `rootDistance` function explains why:

```go
// For an SNTP client which sends only a single packet, most of these
// terms are irrelevant and become 0.
totalDelay := rtt + rootDelay
return totalDelay/2 + rootDisp
```

A full NTP implementation is a feedback loop: query multiple servers, filter outliers, weight by quality, discipline the local oscillator. The reference implementation is thousands of lines of state management and statistical estimation. Vickers built none of it. The library answers one question — "how far off is my clock right now?" — and answers it well enough for the vast majority of callers who will use it to check drift, adjust an application timer, or display a corrected time.

The design shows in what's exported. The `Response` struct carries `ClockOffset`, `RTT`, `RootDistance`, `MinError` — everything you need to decide whether to trust this particular answer. The library computes the metadata for trust but does not make the trust decision for you. A kiss-of-death response (stratum 0) produces a `Response` with a `KissCode` field; it does not panic or return only an error. `Validate()` exists as a separate method. You call it if you want the library's opinion. You don't have to.

## The validation function is a paranoia checklist

`Validate()` reads like a sequential enumeration of the ways an NTP response might be lying:

```go
func (r *Response) Validate() error {
    if r.authErr != nil {
        return r.authErr
    }
    if r.Stratum == 0 {
        return ErrKissOfDeath
    }
    if r.Stratum >= maxStratum {
        return ErrInvalidStratum
    }
    // ...
    if r.Time.Before(r.ReferenceTime) {
        return ErrInvalidTime
    }
    if r.Leap == LeapNotInSync {
        return ErrInvalidLeapSecond
    }
    return nil
}
```

Each check is independent. None are combined. The freshness check — server transmit time minus reference time must be less than ~36 hours — catches a server that hasn't synced recently. The dispersion check — `RootDelay/2 + RootDispersion` must be under 16 seconds — catches a server whose error bounds are too wide to be useful. The backwards-clock check — server can't transmit before its own reference time — catches basic nonsense.

These aren't defensive abstractions. Each one corresponds to a specific failure mode that NTP servers exhibit in the wild. The error names are the documentation: `ErrServerClockFreshness`, `ErrServerTickedBackwards`, `ErrInvalidDispersion`. Read the error list and you've read a catalog of the ways time servers fail.

## Hand-rolled CMAC, because the boundary is the point

The authentication code in `auth.go` implements AES-CMAC from scratch in about 35 lines. Not because there's no CMAC library for Go — there is — but because the implementation is compact enough that the dependency isn't worth it.

```go
func double(dst, src []byte, xor int) {
    _ = src[15] // compiler hint: bounds check
    s0 := binary.BigEndian.Uint64(src[0:8])
    s1 := binary.BigEndian.Uint64(src[8:16])

    carry := int(s0 >> 63)
    d0 := (s0 << 1) | (s1 >> 63)
    d1 := (s1 << 1) ^ uint64(subtle.ConstantTimeSelect(carry, xor, 0))
```

The `double()` function — which left-shifts a 128-bit value and conditionally XORs with a constant — uses `subtle.ConstantTimeSelect` for the carry. This prevents timing side-channels. The `_ = src[15]` lines are compiler hints that eliminate bounds checks in the subsequent code. The `verifyMAC` function uses `subtle.ConstantTimeCompare`. None of this is ceremony; each line addresses a specific class of attack.

The digest functions for SHA-256 and SHA-512 both truncate to 20 bytes: `digest[:20]`. This matches the NTP spec's MAC format, which allocates a fixed-size field regardless of hash output. The truncation is a protocol constraint, not a security decision, and the code doesn't explain it because it doesn't need to — anyone reading `auth.go` is already holding the RFC.

## Extensions unwind like a stack

When building the query, extensions are processed in order:

```go
for _, e := range opt.Extensions {
    err = e.ProcessQuery(&xmitBuf)
```

When processing the response, they're iterated in reverse:

```go
for i := len(opt.Extensions) - 1; i >= 0; i-- {
    err = opt.Extensions[i].ProcessResponse(recvBuf)
```

This models extensions as layers. The last extension added to the outgoing packet wraps the outermost layer; unwrapping the response means peeling from the outside in. It's a three-line architectural decision that implicitly defines a composition model: extensions are a stack, not a pipeline.

## The voice of a decade

Vickers's GitHub spans XML parsers, a 64-bit operating system, a 6502 CPU emulator, and this NTP library. The range suggests someone who works at boundaries — between software and hardware, between software and protocols, between software and time. The NTP library is the most patient of these projects. Fifty commits over a decade. Forty-six of them are his.

The July 2023 burst is visible in the git log: authentication support, extensions, data minimization, address parsing, kiss-of-death handling, all in one month. Then quiet. A few commits per year. June 2024: the era rollover, twelve years early. September 2025: accepting a contributor's `GetSystemTime` callback. January 2026: fixing crypto-NAK detection. Each commit addresses a specific edge of the protocol that someone hit in production or that Vickers noticed while reading a draft RFC.

The code has never been rewritten. The `header` struct from the earliest version maps to the same wire format. The `rtt()` and `offset()` functions have the same four-parameter signatures. The additions — auth, extensions, era handling — are incremental. They don't restructure; they extend. This is maintenance in the original sense: keeping a thing in working order, not rebuilding it.

The offset function carries a comment that links to a David Mills paper about era boundary crossings. The `getTime` function notes that Go 1.9's monotonic clock means `recvTime - xmitTime` should never go negative, then clamps it to zero anyway. The `Precision` field on the outgoing header is set to `0x20` — a value that says "I'm not telling you my real precision." Every detail is considered. None are accidental.

What makes this library worth reading is not that it implements NTP correctly — many libraries do. It's that every line reveals a specific opinion about what "correctly" means when you're asking a stranger on the internet what time it is. You lie about your clock. You validate their response against six independent criteria. You handle a timestamp rollover that won't happen for a decade. You implement your own CMAC rather than trust a dependency. And you do all of this in 1,092 lines that have remained structurally stable for ten years, because the protocol is stable, the problems are known, and the author has the patience to solve them one commit at a time.
