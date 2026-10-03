---
layout: post
title: "The Architecture, With Diagrams"
date: 2026-09-27
description: "How Vault talks to the plugin, why multiplexing matters, and the shared machinery behind both credential paths."
---

Vault plugins run as separate processes from Vault itself, not inside
Vault's own memory space. That's deliberate — a bug in a plugin can't
crash Vault core. Vault starts the plugin, verifies its binary against a
checksum it already has on file, and talks to it over a mutually
authenticated RPC connection. Here's what that looks like end to end for
this plugin:

![Vault core, the external plugin process, and MinIO communicate over mutually authenticated RPC; a single multiplexed plugin process serves every MinIO mount across the Vault cluster.](/assets/fig-architecture.png)

### Why one process instead of many

Without extra work, Vault spawns a separate OS process for every mount of
a plugin — even if ten teams are all running the identical plugin binary,
that's ten processes, ten sets of file descriptors, ten independent RPC
handshakes at mount-enable time. Vault supports *multiplexing* to avoid
this: a plugin that opts in runs as a single process serving every mount
of that type across the whole Vault cluster.

I implemented multiplexing support specifically because the cost it
avoids scales with how many teams adopt the plugin, not with how much
traffic any one of them sends. The catch: credential lifecycle state —
which credentials are active, retired, or evicted — had to be explicitly
keyed by mount, not treated as global state in the process. Otherwise two
teams' identically-named roles could collide inside the same shared
process.

### One role, one credential type, no mixing

Every role picks exactly one `credential_type`: `static` or `sts`. Not
both. If a service genuinely needs both behaviors, it gets two separate
roles, not one role doing double duty.

### The detail I didn't expect

Here's the thing that surprised me when I actually traced the code: the
static and STS paths are *not* two independent systems. Both of them
start by resolving — or creating, if none exists — a backing MinIO
credential for the role, using the exact same logic. A static-type role
just hands that credential back to the caller directly. An STS-type role
instead uses that same backing credential's own keys (never the plugin's
configured root/admin credential) to call MinIO's `AssumeRole` API, and
hands back a short-lived session instead of the long-lived key itself.

So "two credential models" really means one lifecycle machine with two
different endings, not two separate code paths that happen to share a
role definition. Here's the full request flow, start to finish:

![End-to-end data flow from client request to credential issuance and eventual expiration, for both static and STS paths.](/assets/fig-flow.png)

The shared box at the top — resolving or creating the backing
credential — runs for every single request, regardless of which
credential type the role is configured for. Only after that does the path
split.

### Why that matters beyond architecture trivia

It changes the actual security story. Compromising one STS-type role's
backing credential doesn't expose every other role's credentials — each
role's backing identity is private to that role, bounded the same way a
static role's credential is bounded. The blast radius of a bad day is one
role, not the whole mount.

Next up: the two credential paths, one at a time, starting with static.
