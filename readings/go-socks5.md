# The Art of the Useful Subset

**Project:** [armon/go-socks5](https://github.com/armon/go-socks5)
**Language:** Go
**Size:** 765 lines across 6 files

---

Armon Dadgar wrote a SOCKS5 server in five hours on a Thursday afternoon in January 2014. Twenty commits, 765 lines, six files. Then he stopped. The project has received a handful of contributions over the next two years — pluggable auth, context support, half-close — and nothing since September 2016. It has 2,100 stars and 556 forks. It is, quietly, the canonical SOCKS5 implementation in Go.

The interesting thing about go-socks5 is not the protocol implementation. SOCKS5 is a well-understood RFC from 1996. The interesting thing is the decision-making. What to build, what to skip, and how to make the boundaries so clean that the skipped parts never matter.

## Six Interfaces in 765 Lines

The project defines six interfaces: `Authenticator`, `CredentialStore`, `NameResolver`, `RuleSet`, `AddressRewriter`, and an implicit dial function signature. Each is one or two methods. Each has a default implementation. The `Config` struct has seven fields, all optional:

```go
type Config struct {
    AuthMethods []Authenticator
    Credentials CredentialStore
    Resolver    NameResolver
    Rules       RuleSet
    Rewriter    AddressRewriter
    BindIP      net.IP
    Logger      *log.Logger
    Dial        func(ctx context.Context, network, addr string) (net.Conn, error)
}
```

You can start a working SOCKS5 server with `socks5.New(&socks5.Config{})`. Every field has a sensible zero-value behavior: no auth required, DNS resolution via the standard library, all connections permitted, no address rewriting, bind to all interfaces, log to stderr, dial with `net.Dial`. This is the Go proverb made structural — make the zero value useful — applied not to a single type but to an entire server architecture.

But the interfaces are not there for abstraction's sake. Each one represents a real extension point that someone actually needs. `CredentialStore` exists because you might check passwords against LDAP or a database. `NameResolver` exists because you might want to intercept DNS. `RuleSet` exists because you might want ACLs. `AddressRewriter` exists because you might need to remap destinations in a service mesh. These are not speculative. Armon was building HashiCorp. He knew exactly which knobs infrastructure operators turn.

The `CredentialStore` interface is the most instructive in its minimalism:

```go
type CredentialStore interface {
    Valid(user, password string) bool
}

type StaticCredentials map[string]string

func (s StaticCredentials) Valid(user, password string) bool {
    pass, ok := s[user]
    if !ok {
        return false
    }
    return password == pass
}
```

Seventeen lines. An interface, a type, a method. The static implementation is obviously not production-grade — plaintext passwords in a map — but it's a complete working example that makes the interface self-documenting. You look at `StaticCredentials` and you immediately understand what `CredentialStore` expects from you. This is a pattern throughout: each interface comes with an implementation that serves double duty as documentation and as a reasonable default.

## The File Creation Order

The git history reveals that Armon built this project bottom-up. The commit sequence: `credentials.go` → `resolver.go` → `ruleset.go` → `auth.go` → `socks5.go` → `request.go`. Interfaces and leaf types first. Then authentication, which depends on credentials. Then the server shell, which depends on everything. Then the request handler, which is the actual wire protocol.

This is not how most people write code. Most people start with the server, realize they need auth, realize auth needs credentials, and end up with a tangled dependency graph they refactor later. Armon started from the leaves of the dependency tree and worked toward the root. By the time he wrote `socks5.go`, every type it references already existed. By the time he wrote `request.go` — the most complex file, handling the actual SOCKS5 protocol — the entire supporting infrastructure was in place.

This suggests he had the architecture mapped out before he touched the keyboard. Five hours is not enough time to design and implement a protocol server. It is enough time to implement one you've already designed.

## What He Didn't Build

SOCKS5 defines three commands: CONNECT, BIND, and ASSOCIATE. CONNECT creates an outbound TCP connection through the proxy. BIND waits for an inbound connection (used by protocols like FTP that need the server to call back). ASSOCIATE sets up UDP relay. Armon implemented CONNECT and left the other two as explicit TODOs:

```go
func (s *Server) handleBind(ctx context.Context, conn conn, req *Request) error {
    // TODO: Support bind
    if err := sendReply(conn, commandNotSupported, nil); err != nil {
        return fmt.Errorf("Failed to send reply: %v", err)
    }
    return nil
}
```

In ten years, across 556 forks, nobody has contributed BIND or ASSOCIATE to the main repository. This is not neglect. This is market signal. CONNECT handles HTTP proxying, HTTPS tunneling, SSH proxying — essentially every modern use case. BIND was designed for FTP active mode, which the world has moved past. ASSOCIATE is for UDP, which most SOCKS5 users don't need because they're proxying TCP traffic.

Armon shipped the useful subset. Not the complete subset. Not the correct-by-spec subset. The useful one. And a decade of usage data has confirmed the judgment.

## Seven Lines That Do All the Work

The actual proxying — the thing the entire server exists to do — is startlingly small:

```go
func proxy(dst io.Writer, src io.Reader, errCh chan error) {
    _, err := io.Copy(dst, src)
    if tcpConn, ok := dst.(closeWriter); ok {
        tcpConn.CloseWrite()
    }
    errCh <- err
}
```

Called twice, once in each direction:

```go
errCh := make(chan error, 2)
go proxy(target, req.bufConn, errCh)
go proxy(conn, target, errCh)

for i := 0; i < 2; i++ {
    e := <-errCh
    if e != nil {
        return e
    }
}
```

Two goroutines, one buffered channel, `io.Copy`. The `closeWriter` interface — a type assertion checking if the connection supports `CloseWrite()` — was added by stbuehler in 2016 for half-close support. Before that, the proxy function was even simpler: just `io.Copy` and an error send. The entire data path of a SOCKS5 proxy is a standard library call.

This is what good abstraction looks like in practice. The complexity of go-socks5 is not in moving bytes. It's in the negotiation, authentication, and connection setup that happens before the bytes start moving. Once the connection is established, the kernel does the work through `io.Copy`'s `splice(2)` optimization. Armon put the complexity where the decisions are, not where the volume is.

## Pragmatic Error Classification

After a dial failure in `handleConnect`, the code needs to send back the appropriate SOCKS5 error code. The RFC defines specific codes for "connection refused" versus "network unreachable" versus generic failure. Here is how the code distinguishes them:

```go
msg := err.Error()
resp := hostUnreachable
if strings.Contains(msg, "refused") {
    resp = connectionRefused
} else if strings.Contains(msg, "network is unreachable") {
    resp = networkUnreachable
}
```

String matching on error messages. In a textbook, this would be wrong. You should use `errors.Is` or type assertions on `net.OpError` or check the underlying syscall error. But in January 2014, Go's error system was young and `net.Dial` returned formatted strings without structured wrapping. Armon worked with what existed, not with what should have existed.

The pragmatism is the point. This code runs inside HashiCorp tools. It handles real traffic. And the string matching works, because the Go standard library has been formatting these error messages the same way for a decade. Sometimes the quick solution is the durable one.

## The Context Pipeline

The context support added by ymmt2005 in 2016 reveals an elegant architectural property. Each stage of request handling accepts and returns a `context.Context`:

```go
// In RuleSet
Allow(ctx context.Context, req *Request) (context.Context, bool)

// In AddressRewriter
Rewrite(ctx context.Context, request *Request) (context.Context, *AddrSpec)
```

This means the processing pipeline — Auth → Rules → Resolve → Rewrite → Connect — is also a context decoration pipeline. The auth stage can stash user identity in the context. The rule evaluation stage can read it. The address rewriter can add routing metadata. The dial function can read everything.

This is a generalized middleware pattern without any middleware framework. No handler chains, no wrapper types, no next-function callbacks. Just `context.Context` flowing through typed interfaces. It's the simplest possible expression of "pass arbitrary data through a processing pipeline" and it works because Go's context was specifically designed for this.

One detail: the imports still reference `golang.org/x/net/context` rather than the standard library `context` package. The context package moved into the standard library in Go 1.7, released August 2016. The last commit to go-socks5 was September 2016. The project froze one month after it could have dropped the external dependency. Time capsule.

## 556 Forks

The fork-to-star ratio of 1:4 is unusually high. Most popular Go libraries run 1:10 or higher. A high fork ratio means people are not just using the code — they're modifying it. They're embedding it into their own projects, adding the BIND command for their specific use case, swapping the logger for `zap`, updating the context import, adding metrics.

This is the signature of infrastructure code. It's not a library you import and call. It's a library you fork and adapt. Armon built the skeleton and the joints. Five hundred people have attached their own muscles.

And that, ultimately, is what 765 lines can do when the interfaces are right. You don't need to implement everything. You need to make the right things replaceable. Armon knew which parts were universal — the protocol negotiation, the connection lifecycle, the proxy loop — and which parts were particular — the auth backend, the DNS resolver, the ACL rules, the dialer. He carved the boundary between them with six interfaces, each one or two methods wide. Then he went home.

The project hasn't needed a commit in ten years. That's not abandonment. That's completion.
