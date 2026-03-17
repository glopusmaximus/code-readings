# Gobreaker

**[sony/gobreaker](https://github.com/sony/gobreaker)** — 380 lines of Go

Yoshiyuki Mineo, Sony. 2015–2026.

---

A circuit breaker has three states. Closed means traffic flows. Open means traffic is blocked. Half-open means a limited number of requests are allowed through to test whether the downstream service has recovered. If they succeed, the breaker closes. If any fail, it opens again.

The pattern has been described so many times that most implementations are interchangeable. Gobreaker's 380 lines don't reinvent it. What they do is solve a problem that most descriptions of the pattern don't mention: what happens to a request that started in one state and finishes in another?

## The Generation Counter

The `CircuitBreaker` struct carries a `generation uint64`. Every time the state changes — or the periodic counter-clearing interval expires in the closed state — the generation increments and the counts reset.

```go
func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
    cb.generation++
    cb.counts.clear()
    // ...
}
```

When a request begins, `beforeRequest` records the current generation:

```go
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
    cb.mutex.Lock()
    defer cb.mutex.Unlock()
    // ...
    cb.counts.onRequest()
    return generation, nil
}
```

When it ends, `afterRequest` checks whether the generation has changed:

```go
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
    cb.mutex.Lock()
    defer cb.mutex.Unlock()
    // ...
    state, generation := cb.currentState(now)
    if generation != before {
        return
    }
    // ...
}
```

If the generation doesn't match, the result is silently discarded.

This is the whole design. A request that took 30 seconds to fail shouldn't trip a breaker that has since recovered and reset its counts. A request that succeeded during a previous half-open window shouldn't count toward the current half-open window's success threshold. The generation counter makes time travel impossible — every result is stamped with the era it belongs to, and stale results from expired eras are dropped without ceremony.

Without this mechanism, a slow failure from a previous closed period could arrive during a new closed period and push the consecutive failure count over the threshold, opening the breaker even though nothing has actually failed recently. The generation counter is a single `uint64` that prevents an entire class of temporal bugs.

## Lazy State Transitions

The breaker doesn't use timers. There is no goroutine watching the clock and flipping states when timeouts expire. Instead, `currentState` checks the clock on every call:

```go
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
    switch cb.state {
    case StateClosed:
        if !cb.expiry.IsZero() && cb.expiry.Before(now) {
            cb.toNewGeneration(now)
        }
    case StateOpen:
        if cb.expiry.Before(now) {
            cb.setState(StateHalfOpen, now)
        }
    }
    return cb.state, cb.generation
}
```

Two transitions happen lazily. In the closed state, if the interval timer has expired, counts are cleared and a new generation starts — but the state stays closed. In the open state, if the timeout has expired, the state moves to half-open. Both transitions happen as side effects of checking the state, not as scheduled events.

This means a breaker that nobody talks to doesn't consume resources. No goroutines, no timers, no channels. A service with a thousand circuit breakers to different backends only pays for the ones that are actively receiving traffic. The cost is that the transition happens slightly late — on the next request rather than at the exact timeout moment — but for a circuit breaker, "slightly late" is indistinguishable from "on time."

## The Counts Struct

The `Counts` struct tracks five numbers: total requests, total successes, total failures, consecutive successes, consecutive failures. It's reset to zero on every generation change. The consecutive counters are the interesting ones — `onSuccess` zeroes `ConsecutiveFailures`, and `onFailure` zeroes `ConsecutiveSuccesses`:

```go
func (c *Counts) onSuccess() {
    c.TotalSuccesses++
    c.ConsecutiveSuccesses++
    c.ConsecutiveFailures = 0
}
```

The default trip condition is `ConsecutiveFailures > 5`. Five consecutive failures and the breaker opens. Not five total failures — five *in a row*. A single success resets the failure counter. This is forgiving to services that fail intermittently and strict with services that have actually gone down.

The `ReadyToTrip` callback receives a *copy* of `Counts`, not a pointer. This prevents the callback from accidentally mutating the breaker's internal state, but more importantly, it means you can write trip conditions based on any combination of the five counters. Failure rate above 50%? `return counts.TotalFailures > 10 && float64(counts.TotalFailures)/float64(counts.Requests) > 0.5`. The interface is five numbers and a boolean — everything else is your problem.

## TwoStepCircuitBreaker

The `TwoStepCircuitBreaker` wraps a `CircuitBreaker` and exposes a different API: instead of `Execute(func() (interface{}, error))`, it has `Allow() (done func(success bool), err error)`.

```go
func (tscb *TwoStepCircuitBreaker) Allow() (done func(success bool), err error) {
    generation, err := tscb.cb.beforeRequest()
    if err != nil {
        return nil, err
    }
    return func(success bool) {
        tscb.cb.afterRequest(generation, success)
    }, nil
}
```

The standard `Execute` wraps a function call — you hand it a closure and it runs the closure inside the breaker. `TwoStepCircuitBreaker` separates the permission check from the result reporting. You call `Allow()`, get permission, do your work however you want, then call the returned callback with the outcome.

The generation is captured in the closure. The callback calls `afterRequest` with the generation that was current when `Allow` was called. If the breaker has moved to a new generation by the time the callback fires, the result is discarded — same mechanism, different API surface.

This matters for cases where the protected operation isn't a simple function call. HTTP requests with streaming responses. Database transactions that span multiple operations. Anything where the "did it work?" answer doesn't come back as a return value from a single function.

## What 380 Lines Teaches

The code links to a Microsoft architecture guide for the circuit breaker pattern. That guide describes the three states, the transitions, and the counting logic. This implementation follows that guide faithfully and adds exactly two ideas: the generation counter for temporal correctness, and the two-step variant for API flexibility.

Yoshiyuki Mineo has maintained this code for eleven years. Seventy-six commits. The v2 added distribution (Redis-backed shared state), a rolling window counter, and file separation. But the v1 — this 380-line file — hasn't needed its core logic changed. The generation counter was there from the initial commit.

The things that don't change in a decade are usually the things that were right the first time. The three states are the pattern. The generation counter is the insight.

---

*Three states everyone knows. One counter nobody talks about. 380 lines, eleven years, and the core logic hasn't needed revision because it solved the temporal problem before anyone asked about it.*
