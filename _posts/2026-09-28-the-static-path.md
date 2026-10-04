---
layout: post
title: "The Static Path: Long-Lived Keys With a Real Lifecycle"
series: vault-minio
date: 2026-09-28
description: "A MinIO access key that still behaves like one, but with a tracked lifecycle, a grace period for rotation, and a hard cap on how many can exist."
---

Some workloads genuinely want a stable credential. A background job that
runs for six hours, or a long-running service that holds a connection
open for days, doesn't want to re-authenticate every fifteen minutes. The
static path is for them: from the client's side, it's a normal MinIO
access key. The difference is everything the plugin does around it.

### Creation → Active → Retired → Evicted

A role configured with `credential_type` set to `static` gets a credential
that moves through four states, tracked by the plugin:

- **Creation** — the first request against a role provisions a backing
  MinIO user and persists its state to Vault's storage.
- **Active** — the credential is handed out to callers and works
  normally.
- **Retired** — once it crosses its `max_ttl`, it doesn't just vanish. It
  moves into a grace period where it still works, but a new credential
  becomes the active one.
- **Evicted** — after the `grace_period` ends, the old credential is
  actually deleted from MinIO.

### Why the grace period exists

If I killed a credential the instant it expired, I'd break every
in-flight request, every long-running batch job that cached the old key,
and every client that hasn't gotten around to refreshing yet. The grace
period buys everyone time to catch up before the old key is actually
deleted. Rotation doesn't have to mean breakage.

### A hard cap on accumulation

Each role holds at most one active and one retired credential. Repeated
requests can't quietly pile up MinIO identities over time, which matters
both for hygiene and for the threat model later in the series: a static
role's exposure is bounded by design, not by someone remembering to clean
up.

### State that survives a restart

The lifecycle state (which credentials are active, retired, or evicted
for each role) lives both in memory and in Vault's own storage. If Vault
fails over to a new node, or an entire site goes down, the plugin picks
up where it left off instead of losing track of a credential that's still
live in MinIO. The disaster recovery drill later in the series tests
exactly that.

### Cleanup is lazy, on purpose

There's no background job sweeping expired credentials. Lifecycle
transitions are evaluated when someone requests that role. That keeps the
plugin simple, but it has a real consequence: a credential that should
have been evicted stays live in MinIO until the role is requested again.
I come back to this in the final post.

### How it performs

Once a credential is active, serving it is a storage lookup, not a live
call to MinIO. Under autocannon at 10 connections for 100 seconds, the
static path averaged about **4,195 requests/sec** with **1.88ms** average
latency. Across a sweep from 5 to 15 connections, throughput ranged from
roughly 3,300 to 5,200 req/s, with latency staying in single-digit
milliseconds.

One early run showed a single 2,611ms request against an otherwise
~2ms-typical latency. It turned out to be the very first request against
a brand-new role, which has to actually create the backing MinIO user and
persist state to Vault. Re-running the same test gave a clean 22ms
maximum. It's a one-time, predictable cost on a role's first use, not a
recurring problem.

### The trade-off

The static path's expiration is enforced by the plugin's own
bookkeeping, not by MinIO. If that bookkeeping is ever wrong, stale, or
never re-evaluated because a role went quiet, nothing else in the system
catches it. That's the gap the other credential type closes, at a
different cost.

Next: the STS path.
