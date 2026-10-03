---
layout: post
title: "Testing It: Unit, Performance, and Disaster Recovery"
date: 2026-09-30
description: "51 test cases, a 360x performance gap, an open-loop capacity cliff, and a live failover drill against a real cluster."
---

"It works on my machine" doesn't mean much for something that manages
credentials. I tested this at three different levels, each answering a
different question.

## Does the logic actually do what it claims?

51 test cases, across four test files, covering role configuration,
credential creation and reuse, TTL and grace-period transitions, cleanup
and failure handling, persistence, and STS policy validation. Overall
coverage lands around 80% of the plugin's logic.

Two numbers are worth explaining rather than just reporting. The binary's
own entrypoint (`main.go`) shows 0% coverage — not because it's untested
in spirit, but because it calls a function that blocks forever serving
RPC traffic. You can't unit test something designed to never return; it's
exercised by actually running the compiled plugin, not by `go test`.

And two files sit lower than the rest (roughly 70–76% vs. 90%+
elsewhere) for a specific, traceable reason: their uncovered branches are
failure paths that require a Vault storage write to fail, or a live MinIO
STS call to fail — neither of which the current test doubles can
actually trigger. These are genuine external-failure branches, not
missed application logic.

## How fast is it, actually?

I benchmarked both credential paths with autocannon across a concurrency
sweep (5, 10, and 15 connections), repeating key runs to check for
consistency rather than trusting a single sample.

The static path scales the way you'd want: latency stays in single-digit
milliseconds, throughput grows from about 3,300 to 5,200 req/s as
concurrency increases. One early run showed a single alarming 2.6-second
outlier — I chased that down separately (worth its own read, but short
version: it was the one-time cost of provisioning a brand-new role's
backing credential, not a recurring problem).

The STS path tells a different story. At the same concurrency levels, it
tops out around 11 requests/sec, with average latency around 850ms–1.9
seconds depending on load. That's roughly a 360x throughput gap against
the static path — and profiling traced almost all of that cost to TLS
handshake and certificate verification, not to anything in the plugin's
own logic.

### The test that actually worried me

Autocannon is a *closed-loop* tool: it only sends a new request once the
last one on that connection finishes, which means it can never truly
offer more load than the server can absorb. To find out what happens when
offered load actually exceeds capacity, I used vegeta instead — an
*open-loop* tool that injects requests at a fixed rate no matter what the
server is doing.

At an offered rate of just 10 requests/sec on the STS path — barely above
what the closed-loop test called "sustainable" — success collapsed to
72%, with average wait times jumping to nearly 26 seconds. Push it to 20
req/s and you're down to 16% success, with most requests simply timing
out. That's a real capacity cliff that the closed-loop numbers alone
never revealed.

## Does it survive losing a datacenter?

The plugin persists its credential lifecycle state both in memory and in
Vault's own storage, specifically so it can recover cleanly from a
restart or failover without losing track of anything. I didn't want to
just trust that in theory, so I tested it against a real multi-node HA
setup: a 3-node Vault Enterprise cluster backed by 5-node Consul, mirrored
at a separate DR site with an identical topology.

Two drills: first, take the primary site down entirely and promote the DR
secondary to active. Second, bring the original primary back, rejoin it,
and promote it back — a full round trip, not just a one-way failover.

Both directions: no credential state lost, no plugin crash. I'll be
upfront that this was a manual, hands-on drill rather than something
running continuously in CI — it tells me the behavior held for the
scenarios I actually exercised, not that it's verified on every commit
the way the unit tests are.

Testing performance and testing survivability told me the system works.
They didn't tell me what it's actually defending against. That's next.
