---
layout: post
title: "The STS Path: Sessions That Expire on Their Own"
date: 2026-09-29
description: "Short-lived sessions issued through MinIO's AssumeRole API, where the expiry is enforced by MinIO itself and the caller never holds a long-lived key."
---

A short CI job that only needs a few seconds of access shouldn't be
holding a key that's valid for a year. The STS path is for workloads like
that, and for anything where nothing should be left standing around to
leak once the work is done.

### Modeled on AssumeRole

STS credentials are short-lived sessions, modeled directly on AWS's STS
`AssumeRole` pattern. MinIO's STS API is deliberately AWS-compatible, so
any existing AWS SDK client works against it without modification.

These sessions expire on their own, enforced by MinIO on every request,
not by the plugin remembering to clean something up. There's no grace
period and no eviction step, because nothing is waiting for a background
job to run.

### How a session gets issued

A role configured with `credential_type` set to `sts` starts the same
way a static role does: the plugin resolves, or creates if none exists, a
backing MinIO credential for that role. Then the paths split. Instead of
handing that credential back, the plugin uses the backing credential's
own keys to call MinIO's `AssumeRole` API, and returns the resulting
short-lived session.

Two details matter here:

- **The plugin's root/admin credential is never used for AssumeRole.**
  The call is made with the role's own backing identity.
- **The backing credential never leaves the plugin.** The caller only
  ever holds a derived session.

### Requests can only narrow access

An STS request can ask for a policy, but only one that's a subset of what
the role already allows. The issuance path can't be used to escalate
privileges beyond what the role was configured with.

### The blast radius is one role

Each role's backing identity is private to that role. Compromising one
STS role's backing credential doesn't expose any other role's
credentials, so the damage from a bad day is bounded to that one role,
not the whole mount.

### How it performs

Every STS request is a live `AssumeRole` call to MinIO over TLS. Under
autocannon at 10 connections for 100 seconds, the STS path averaged about
**11.6 requests/sec** with **860ms** average latency. Across a sweep from
5 to 15 connections, average latency ranged from roughly 850ms to 1.9
seconds.

I profiled it with Go's `pprof` (a 30-second window, about 2.8 seconds
of sampled CPU time) to see where that time goes:

- **68.95%** of sampled time was inside the TLS handshake.
- Within that, **62.45%** was X.509 certificate-chain verification,
  bottoming out in RSA signature checks.
- The plugin's own credential-issuance logic accounted for well under 10%.

The implementation opens a fresh connection for every `AssumeRole` call,
which means a full handshake and certificate verification every time.
Reusing a persistent HTTP client with connection pooling would remove
most of that cost without changing the path's security properties at
all. It's on the list in the final post.

### The trade-off

Its lifetime boundary is enforced by MinIO, not by bookkeeping. But you
can't revoke a single compromised session early. The only way to cut off
access before natural expiry is disabling the role's backing identity,
which invalidates every other live session for that same role. At least
that blast radius stays contained to the one role.

Next: what it took to actually trust both paths enough to call them
tested.
