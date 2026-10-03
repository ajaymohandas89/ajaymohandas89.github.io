---
layout: post
title: "Testing Disaster Recovery for a Vault Plugin: Taking Down Production on Purpose"
date: 2026-10-15
---

It's one thing to say a plugin "handles Vault failover correctly." It's
another thing to actually kill the primary cluster and watch what happens.

### The setup

Two sites, mirrored:

- **Primary (Cluster A):** 3-node Vault Enterprise HA cluster, backed by a
  5-node Consul storage cluster
- **DR (Cluster B):** an identical 3 Vault / 5 Consul topology at a
  separate site

The plugin persists its credential lifecycle state (which credentials are
`Active`, `Retired`, or `Evicted` for each role) both in memory and in
Vault's own storage. In theory, that means if Vault fails over to a new
node — or an entire site goes down — the plugin should pick up exactly
where it left off once it's running again, without losing track of any
credential's state.

Theory is nice. I wanted to actually watch it happen.

### Test 1: kill the primary

I took Cluster A down entirely and promoted Cluster B — the DR
secondary — to active primary.

What I was watching for: did any credential lifecycle state get lost?
Did the plugin process crash or come back up in some broken state?

Neither happened. Vault's persisted storage carried the state over
cleanly, and the plugin picked it back up on the newly-active cluster
without needing to re-derive anything.

### Test 2: the full round trip

The more interesting test, honestly, was reversing it: bring Cluster A
back, rejoin it as secondary, then promote it *back* to active primary
while Cluster B returns to being the DR secondary — restoring the
original topology exactly.

Same result both directions: no state loss, no crash. The round trip
mattered because it's not just "can you fail over once" — it's "can you
fail back without quietly breaking something on the way."

### Why this was worth doing manually

I'll be upfront: this wasn't an automated regression test. It was a live,
hands-on drill against a real multi-node cluster, not something that runs
in CI on every commit. That's a real limitation — it tells me the
behavior held for the specific scenarios I exercised, not that it's
continuously verified the way the unit test suite is.

But for a plugin whose entire job is tracking credential state correctly
over time, "does this survive an actual failover" isn't optional
homework. It's the thing that matters most if it ever goes wrong in
production, and I'd rather have found problems in a deliberate drill than
during a real incident.
