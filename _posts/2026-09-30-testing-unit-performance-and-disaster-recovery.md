---
layout: post
title: "Testing It: Unit, Performance, and Disaster Recovery"
series: vault-minio
date: 2026-09-30
description: "51 test cases, throughput and latency for each credential path under load, and a live failover drill against a real cluster."
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

The STS path makes a live `AssumeRole` call to MinIO on every request.
At the same concurrency levels, it handles around 11 requests/sec, with
average latency around 850ms–1.9 seconds depending on load. Profiling
traced almost all of that time to TLS handshake and certificate
verification, not to anything in the plugin's own logic.

I also ran an open-loop load test against the STS path. Those results
matter most as a denial-of-service question, so they're covered in the
threat model post.

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
