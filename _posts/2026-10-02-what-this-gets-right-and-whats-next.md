---
layout: post
title: "What This Gets Right, What's Still Missing, and What's Next"
series: vault-minio
date: 2026-10-02
description: "An honest accounting of both credential paths' trade-offs, seven real limitations, and where this goes from here."
---

Every design is a set of trade-offs, not a list of wins. Here's an honest
accounting of both sides, and where I'm taking this next.

### What the static path gets right

No per-request network round trip once a credential is active — it's a
storage lookup, not a live issuance call, which keeps its latency in
single-digit milliseconds under load. It suits long-running
or offline workloads that can't realistically re-authenticate every few
minutes. And the grace period means rotation doesn't mean breakage —
an old credential keeps working for a bounded window after a new one
exists, instead of cutting off instantly.

The honest trade-off: its expiration is enforced by the plugin's own
bookkeeping, not by MinIO itself. If that bookkeeping is ever wrong,
stale, or just never re-evaluated because a role went quiet, nothing else
in the system catches it.

### What the STS path gets right

Its lifetime boundary is cryptographic, not advisory — MinIO rejects an
expired session by construction, not because a background job got around
to evicting it. The caller never holds a long-lived key at all, only a
derived session. And because each role's backing credential is private to
that role, compromising one doesn't cascade to others.

The honest trade-off: you can't revoke a single compromised session
early. The only way to cut off access before natural expiry is disabling
the backing identity entirely — which invalidates every other live
session tied to that same role, though at least that blast radius stays
contained to the one role.

### The limitations I'm not going to pretend don't exist

A few of these only became obvious once I went looking for them
deliberately:

- **Vault and MinIO can silently disagree.** Nothing continuously checks
  that what Vault believes about a credential's state still matches
  reality in MinIO. If someone deletes a MinIO user out of band, Vault
  won't know until something forces a re-check.
- **No locking under concurrent requests.** Two near-simultaneous
  requests for the same role could theoretically race on the same
  lifecycle transition. The bounded credential count constrains the
  outcome, but it's not a proven race-free guarantee under load.
- **Partial failures between two systems aren't fully handled.**
  Provisioning a credential means writing to both MinIO and Vault
  storage, with no shared transaction across them. A crash between the
  two can leave one system knowing something the other doesn't.
- **Cleanup only happens when someone asks.** A credential that should
  have been evicted days ago stays fully live if nobody requests that
  role again. There's no background sweep, only lazy, on-demand
  evaluation.
- **It doesn't speak Vault's native lease language.** The static
  lifecycle is custom bookkeeping, not Vault's built-in lease/revocation
  system — so none of Vault's own lease tooling or dashboards see these
  credentials.
- **One mount, one MinIO cluster.** Multiple clusters means multiple
  mounts today; there's no single role that can span more than one
  cluster.
- **No early revocation for STS sessions**, as covered above.

### Where this actually goes next

Each of these maps to something concrete, not a vague "improve
reliability" bullet:

- A periodic reconciliation pass between Vault's view and MinIO's actual
  state.
- Proper locking or an atomic compare-and-swap on lifecycle transitions.
- A repair path for partial failures across the Vault/MinIO write
  boundary.
- An optional background sweeper that complements — not replaces — the
  current lazy cleanup.
- Migrating the static lifecycle onto Vault's own native lease framework.
- Cross-mount cluster targeting, so one control plane can route across
  multiple MinIO deployments.
- A compensating control for STS's revocation gap — a shorter default
  session TTL, or a documented signing-key-rotation runbook.
- Persistent HTTP client reuse for the STS path, to remove most of the
  TLS handshake cost that profiling found, without touching its security
  properties at all.

None of this was obvious when I started. Most of it came from actually
profiling, load-testing, and failing this thing on purpose, rather than
reasoning about it from the architecture alone. That's probably the real
takeaway, more than any individual finding: the gap between "this should
work" and "I watched it work, and watched it break" is where the useful
information actually lives.
