---
layout: post
title: "I Profiled Our Vault Plugin and the Bottleneck Wasn't Our Code"
date: 2026-10-08
---

I assumed, going in, that if the STS path was slower than the static path,
it would be because of something *we* wrote — maybe the policy-validation
logic, maybe something inefficient in how we talk to Vault's storage
layer. I was wrong, and the actual answer turned into one of the more
useful debugging stories from this project.

### The first surprising number

Running both paths under the same load (autocannon, 10 connections, 100
seconds):

- **Static path:** ~4,195 requests/sec average, 1.88ms average latency
- **STS path:** ~11.6 requests/sec average, 860ms average latency

That's not a small gap. That's roughly 360x.

My first instinct was "okay, AssumeRole is a real network call to MinIO,
of course it's slower than reading a cached credential out of storage."
True, but 360x felt too big to just shrug at. So I profiled it.

### What CPU profiling actually showed

I captured a CPU profile of the STS path with Go's `pprof` — a 30-second
window with about 2.8 seconds of actual sampled CPU time. Here's the
breakdown that mattered:

- **68.95%** of sampled time: inside the TLS handshake
- Within that, **62.45%**: specifically X.509 certificate-chain
  verification
- Underneath *that*: RSA signature checking, bottoming out in big-integer
  modular exponentiation

Our own code — the actual credential-issuance logic — accounted for well
under 10% of total sampled time.

In other words: almost none of the cost was "ours." It was TLS setup. Our
implementation opens a fresh connection for every single `AssumeRole`
call, which means a full handshake and a full certificate chain
verification, every time, instead of reusing a connection. That's a
completely different fix than "go optimize your business logic" — it's
"add a persistent HTTP client with connection pooling," a much smaller and
more mechanical change than I was bracing for.

### The load-testing result that actually worried me

The CPU profile explained *why* each request was slow. It didn't tell me
what happens when you throw real concurrent traffic at it. For that I
needed a different kind of load test.

Most load-testing tools, including the one I'd been using (autocannon),
are **closed-loop**: they open N connections and only send a new request
once the last one on that connection finishes. That's a self-throttling
model — it can never actually offer more load than the server can
absorb, because it waits.

I switched to [vegeta](https://github.com/tsenart/vegeta), which is
**open-loop**: you tell it a fixed rate — say, 10 requests/second — and it
fires requests at exactly that rate regardless of whether earlier ones
have finished. This is a much more honest test of "what happens if traffic
actually arrives at this rate."

The results were worse than I expected:

| Offered rate | Actual throughput | Success rate | Mean latency |
|---|---|---|---|
| 10 req/s | 4.82 req/s | 71.8% | 25.9 seconds |
| 15 req/s | 3.94 req/s | 39.3% | 22.9 seconds |
| 20 req/s | 2.13 req/s | 16.0% | 28.8 seconds |

At an offered rate barely above what the closed-loop test said was
"sustainable" (~11.6 req/s), success collapsed to 72%, and the average
wait time jumped to nearly 26 seconds. Push it a little further and you're
down to 16% success with most requests just timing out.

This is the part that actually changed how I think about load testing in
general: **closed-loop benchmarks can make a system look far healthier
than it is**, because they can't reveal what happens once you cross the
real capacity line. Open-loop testing showed a hard cliff that the
"sustainable" throughput number completely hid.

### A debugging detour: the mystery 2.6-second spike

One more thing worth mentioning, because it's a good lesson in not
jumping to conclusions. An early test run of the *static* path — the fast
one — had one single request take 2,611ms, a wild outlier against an
otherwise ~2ms-typical latency. For a while I assumed GC pause, or some
transient network blip.

It wasn't. I re-ran the exact same test (same concurrency, same role) and
got a clean 22ms max. Turns out: the very first request against a brand
new role has to actually *create* the backing MinIO user and persist
state to Vault — real provisioning work. Every request after that is just
reading an already-existing credential, which is fast. The "slow" run
just happened to be the first one to ever hit that role. Once I understood
that, the spike wasn't scary anymore — it's a one-time, predictable cost
on a role's first use, not a systemic problem.

### What I'd actually change

The fix that falls out of all this is specific and small: reuse a
persistent HTTP client (and its connection pool) across STS requests
instead of paying for a fresh TLS handshake and certificate check on every
single call. It won't change the security properties of the STS path at
all — it just stops re-verifying the same certificate chain hundreds of
times for no reason.
