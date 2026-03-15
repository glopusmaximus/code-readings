# ARP

**Project:** [mdlayher/arp](https://github.com/mdlayher/arp)
**Language:** Go
**Size:** 675 lines across 7 files (including two CLI tools)

---

Matt Layher builds protocol implementations the way other people build CRUD apps. His GitHub is a stack of networking packages — ethernet, packet, dhcp6, netlink, Wi-Fi — each one a single protocol rendered in Go. The ARP package sits near the bottom of this stack, both chronologically (July 2015) and architecturally (layer 2, just above raw ethernet frames). It implements RFC 826, the Address Resolution Protocol, in 675 lines. The interesting thing about those 675 lines isn't the protocol — ARP is one of the simplest in the entire networking stack. The interesting thing is the way the code reveals its author's opinions about where trust should go and where it shouldn't.

## The Wire Format Is a Liar

The ARP packet format includes two self-describing length fields: `HardwareAddrLength` tells you how long the hardware addresses are, and `IPLength` tells you how long the protocol addresses are. For standard Ethernet-over-IPv4, these are always 6 and 4. Always. Except when they're not.

In the original `MarshalBinary`, Layher hardcoded the IP address size:

```go
b := make([]byte, 2+2+1+1+2+4+4+(p.HardwareAddrLength*2))
```

The `4+4` is two IPv4 addresses, four bytes each. Then he ran go-fuzz. The commit message from July 30, 2015 reads: "packet: fix crasher found by go-fuzz due to misleading IP address lengths." The fix:

```go
// Though an IPv4 address should always 4 bytes, go-fuzz
// very quickly created several crasher scenarios which
// indicated that these values can lie.
b := make([]byte, 2+2+1+1+2+(p.IPLength*2)+(p.HardwareAddrLength*2))
```

The buffer is now sized from the packet's own claimed lengths, not from the programmer's knowledge of what those lengths should be. This is a subtle shift in trust. The original code trusted the programmer's understanding of the protocol. The fuzzed code trusts the wire format's self-description. The comment is honest about why: the values can lie. A packet can claim its IP addresses are 7 bytes long, or 0, or 255. If you size your buffer based on what you *know* the length should be and then copy based on what the header *says* it is, you get a bounds violation.

The fuzz harness itself is 17 lines:

```go
func Fuzz(data []byte) int {
    p := new(Packet)
    if err := p.UnmarshalBinary(data); err != nil {
        return 0
    }

    if _, err := p.MarshalBinary(); err != nil {
        panic(err)
    }

    return 1
}
```

Unmarshal arbitrary bytes, then marshal the result. If unmarshal succeeds, marshal must not panic. This is a round-trip property test — not for correctness (the output doesn't need to match the input) but for safety. Any input that survives unmarshal must be marshallable. The panic on marshal error is intentional: `MarshalBinary` is documented as never returning an error (`MarshalBinary never returns an error`), so if it does, something fundamental broke.

The fuzz harness was committed on the same day as the fix. Layher found the bug and wrote the regression test in one session. And the bug it found is the kind of thing that haunts C implementations of network protocols — trusting a header field to match your expectations about what valid data looks like. The fix is to trust nothing and size everything from what the packet claims, while validating that the claims are consistent with the actual data length.

## One Allocation, Four Fields

Six days after the fuzz fix, another commit: "packet: speed up Packet.UnmarshalBinary by allocating a single slice and re-slicing fields from it." This is the `UnmarshalBinary` code in its final form:

```go
// Allocate single byte slice to store address information, which
// is resliced into fields
bb := make([]byte, addrl-n)

// Sender hardware address
copy(bb[0:ml], b[n:n+ml])
p.SenderHardwareAddr = bb[0:ml]
n += ml

// Sender IP address
copy(bb[ml:ml+il], b[n:n+il])
senderIP, ok := netip.AddrFromSlice(bb[ml : ml+il])
```

One `make` call. One allocation. The resulting slice is carved up with slice expressions into four fields: sender hardware address, sender IP, target hardware address, target IP. Each field is a window into the same backing array.

In most Go code, you'd see four separate allocations — `make([]byte, 6)` for each address. That's what the pre-optimization version did. The single-allocation version is harder to read (you have to track the offset arithmetic through `ml`, `ml2`, `il`, `il2`) but produces one allocation instead of four. For a packet parser that might run on every ARP frame crossing a network interface, this matters.

What's notable is the timing. This optimization was committed on August 5, 2015 — day 30 of the project. The benchmarks were committed on the same day. Layher wasn't optimizing prematurely or in response to a production problem. He was writing benchmarks as part of the initial development and optimizing what they revealed. The `b.ReportAllocs()` in the benchmarks shows he was tracking allocations from the start.

## The API That Got Rewritten

The most significant commit in the project's history isn't by Layher. On August 10, 2015 — five weeks after the initial commit — Dominik Honnef submitted a pull request titled "restructure API" that deleted 698 lines and added 251. It removed the server, the multiplexer, and the request type. It unified the client and server into a single `Client`.

Honnef's commit message is a design document:

> This change completely overhauls the API. It gets rid of the server and client distinction. From a traditional networking standpoint, there are no ARP clients and servers. Each device will send requests, and it will answer to requests.

The original API modeled ARP as client/server — you either send requests or handle them. Honnef pointed out that this doesn't match the protocol. ARP is peer-to-peer. Every machine both asks questions and answers them. The rewrite gave the API five methods: `Read`, `WriteTo`, `Request`, `Reply`, `Resolve`. Two low-level (read a packet, write a packet), two mid-level (send a request, send a reply), one high-level (resolve an IP to a MAC address).

The split between `Resolve` and `Request` is the most deliberate design decision in the final API. `Resolve` sends a request and loops until it gets a matching reply:

```go
func (c *Client) Resolve(ip netip.Addr) (net.HardwareAddr, error) {
    err := c.Request(ip)
    if err != nil {
        return nil, err
    }

    for {
        arp, _, err := c.Read()
        if err != nil {
            return nil, err
        }

        if arp.Operation != OperationReply || arp.SenderIP != ip {
            continue
        }

        return arp.SenderHardwareAddr, nil
    }
}
```

`Request` just sends. The comment on `Resolve` is explicit: "Resolve must not be used concurrently with Read." Two methods because two usage patterns exist. If you just need one MAC address, call `Resolve`. If you're building a proxy ARP daemon that sits in a read loop, use `Request` and `Read` separately. Not a unified abstraction that handles both badly.

This is the kind of API that results from someone looking at a first draft and saying "this doesn't match the domain." Layher merged it the same day. The pull request discussion is one message long.

## The Injection Point

In April 2017, Paul Tagliamonte submitted a PR to allow creating a Client from an arbitrary `net.PacketConn`. The motivation: promiscuous mode. Tagliamonte wanted to sniff all traffic on an interface, not just ARP. The standard `Dial` function creates a raw socket filtered to ARP frames. Tagliamonte needed to pass in a socket that listens to everything.

The solution was to expose the `New` constructor alongside `Dial`:

```go
// newClient is the internal, generic implementation of newClient.  It is used
// to allow an arbitrary net.PacketConn to be used in a Client, so testing
// is easier to accomplish.
func newClient(ifi *net.Interface, p net.PacketConn, addrs []netip.Addr) (*Client, error) {
```

The comment says "so testing is easier to accomplish." And it is — the test file defines `noopPacketConn`, a mock that implements `net.PacketConn` with no-ops, and embeds it into test-specific types like `closeCapturePacketConn` and `deadlineCapturePacketConn`. No real network interfaces needed.

But the injection point serves double duty. It's for testing *and* for power users who need different socket configurations. The interface isn't a test-only abstraction — it's the actual seam in the architecture. `Client.p` is `net.PacketConn`, and anything that satisfies that interface can drive ARP operations. The tests and the promiscuous-mode use case fall out of the same design.

## 76 Lines of Daemon

The `cmd/proxyarpd` directory contains a complete proxy ARP daemon in 76 lines. The filtering logic is three `continue` statements:

```go
// Ignore ARP replies
if pkt.Operation != arp.OperationRequest {
    continue
}

// Ignore ARP requests which are not broadcast or bound directly for
// this machine
if !bytes.Equal(eth.Destination, ethernet.Broadcast) && !bytes.Equal(eth.Destination, ifi.HardwareAddr) {
    continue
}
```

Then one IP comparison, one log line, one call to `client.Reply()`. A proxy ARP daemon answers the question "who has IP address X?" with "I do" — even when the IP belongs to a different machine. It's used for bridging, VPNs, and network testing. In C, this would require raw sockets, manual ethernet frame construction, careful buffer management, and probably a few hundred lines. Here it's a `for` loop with some `if` statements.

The daemon also demonstrates the API that Honnef's rewrite enabled. It uses `Read` in a loop (the "server" pattern) and `Reply` to respond (the "client" pattern). Both through the same `Client`. Both in the same `for` loop. The unified API makes this natural in a way the original client/server split wouldn't have.

There's a correctness fix from Dominik Honnef here too. The original proxy ARP daemon checked the ARP-layer target address for broadcasts. Honnef changed it to check the ethernet-layer destination. This matters because the ARP target hardware address in a request is typically zero (you're asking "who has this IP?", you don't know the hardware address yet). The broadcast check belongs on the ethernet frame, not the ARP payload. Two layers of framing, and the filter needs to know which layer to check.

## Two Layers, Two Packages

Every ARP operation goes through two serialization cycles. The ARP packet is marshaled into bytes, then wrapped in an ethernet frame, then that frame is marshaled. Reading reverses it: unmarshal the ethernet frame, check the EtherType, unmarshal the ARP payload.

```go
func parsePacket(buf []byte) (*Packet, *ethernet.Frame, error) {
    f := new(ethernet.Frame)
    if err := f.UnmarshalBinary(buf); err != nil {
        return nil, nil, err
    }

    if f.EtherType != ethernet.EtherTypeARP {
        return nil, nil, errInvalidARPPacket
    }

    p := new(Packet)
    if err := p.UnmarshalBinary(f.Payload); err != nil {
        return nil, nil, err
    }
    return p, f, nil
}
```

Five lines of logic. The `ethernet` package is a separate Layher project (`mdlayher/ethernet`). The `packet` package that handles raw sockets is another (`mdlayher/packet`). ARP depends on both but owns neither. Each layer is independently testable, independently replaceable.

This is the pattern across Layher's entire collection of networking packages. He doesn't build monolithic protocol stacks. He builds layers. Each package handles one protocol at one level of the stack. ARP knows about ethernet framing only through the `ethernet.Frame` type. It doesn't know about raw sockets — that's `packet`'s job. The composition happens in `Client.Dial`, which calls `packet.Listen`, and in `WriteTo`, which wraps ARP bytes in an ethernet frame and hands the result to `PacketConn.WriteTo`.

## Seven Years of Maintenance

The initial burst of development lasted five weeks: July 6 through August 12, 2015. Layher built the packet serialization, the client, the fuzz testing, the CLI tools, and the benchmarks. Honnef rewrote the API. Then quiet.

Over the next seven years, the project received 17 more commits. A contributor moved the raw socket protocol constant into the library. Another fixed the ethernet destination address in `WriteTo`. Another added InfiniBand-over-IP support (20-byte hardware addresses instead of 6). Go modules were added. `gofumpt` formatting was applied. And in the final substantive commit, Seth Bromberger migrated from `net.IP` to `netip.Addr` — the standard library's newer, more efficient IP address type.

None of these are architectural changes. The structure that existed after Honnef's August 2015 rewrite is the structure that exists today. The ARP packet format hasn't changed since 1982. The code that implements it hasn't changed structurally since 2015. What changed was the Go ecosystem around it — better IP types, better formatting tools, better socket abstractions — and the code adapted to those changes without changing shape.

384 stars, seven years, and a `buf := make([]byte, 128)` that's generous enough for any ARP frame over any link layer. The read buffer in `Client.Read` is allocated per call, not pooled. For ARP traffic — which is low-volume, low-frequency, and local to a single broadcast domain — this is exactly right. A sync.Pool would save one small allocation per read and add complexity that would outlive the savings. Layher knows where his code runs and doesn't optimize for a load profile it will never see.

That restraint is the thesis. Every decision in this codebase is made with a clear sense of what ARP is: a simple, old, low-traffic protocol that maps IP addresses to hardware addresses on a local network. The code is exactly as complex as that problem requires. The fuzz testing exists because the wire format can't be trusted. The single-allocation optimization exists because the benchmarks revealed it. The two-function API exists because two usage patterns exist. Nothing is built speculatively. Nothing is built for a future that ARP doesn't have. The protocol was done in 1982. The code was done in 2015.
