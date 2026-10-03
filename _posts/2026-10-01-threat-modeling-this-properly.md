---
layout: post
title: "Threat Modeling This Properly"
date: 2026-10-01
description: "What's in scope, what's deliberately out of scope, and a full pass through STRIDE."
---

It's easy to say a credential system is "secure" without ever writing
down what that actually means — what it protects against, and just as
importantly, what it explicitly doesn't. I wanted both halves on paper
before calling this done.

### What's actually in scope

Four things this design is meant to hold up against:

- **Long-lived static credentials leaking** — bounded by the
  max_ttl/grace_period lifecycle, instead of a key that's valid
  indefinitely by default.
- **Stale credential replay** — a retired-but-not-yet-evicted static
  credential, or an expired STS session, presented after it should no
  longer work. MinIO rejects both on its own.
- **Unbounded credential accumulation** — the plugin caps each role at
  one active and one retired credential, so repeated requests can't quietly
  pile up MinIO identities over time.
- **Privilege escalation through the issuance path itself** — an STS
  request can only ask for a policy that's a subset of what the role
  already allows.

### What's deliberately out of scope

Compromise of Vault's own unseal mechanism or root token. Compromise of
the plugin's configured MinIO admin credential (used only for admin
operations like provisioning backing users — not for the AssumeRole call
itself, which uses a role-scoped credential instead). Network-level
man-in-the-middle attacks, assumed to be handled by TLS configuration
outside this system. Host-level compromise of the machines involved. And
one I debated adding, then decided against: a legitimate Vault operator
misusing their own access to tamper with role configuration. That's a
Vault governance question — who's allowed to change policy — not
something a plugin's own design can defend against. Scoping it in would
have been a category error against what this project is actually
responsible for.

### Running it through STRIDE

Beyond the narrative version above, I mapped the system against each of
Microsoft's STRIDE categories, since that forces you to check categories
you might not think to ask about on your own:

**Spoofing** — mitigated by Vault's ACL policy on the plugin's paths, plus
mutual TLS between Vault core and the plugin process. Residual risk: a
compromised Vault token with access to the mount looks identical to a
legitimate request.

**Tampering** — Vault's own ACL enforcement and storage integrity protect
role configuration. Residual: an operator with legitimately broad policy
access can still tamper with it (the same governance question flagged
above).

**Repudiation** — Vault's audit device logs every request on the plugin's
paths. Residual: MinIO's own access logs are a separate system from
Vault's audit trail, and correlating "who requested this in Vault" with
"what it did in MinIO" isn't automated.

**Information disclosure** — bounded lifetimes reduce the exposure
window, but neither path masks credentials in application-level logs
outside Vault's control. That's always going to be true for any system
handing a credential to a client.

**Denial of service** — this is the one the open-loop load test feeds
directly into. Autocannon, used for the benchmarks in the last post, is a
*closed-loop* tool: it only sends a new request once the last one on that
connection finishes, so it can never offer more load than the server can
absorb. To find out what happens when offered load actually exceeds
capacity, I used vegeta instead, an *open-loop* tool that injects
requests at a fixed rate no matter what the server is doing. On the STS
path:

| Offered rate | Actual throughput | Success rate | Mean latency |
|---|---|---|---|
| 10 req/s | 4.82 req/s | 71.8% | 25.9 seconds |
| 15 req/s | 3.94 req/s | 39.3% | 22.9 seconds |
| 20 req/s | 2.13 req/s | 16.0% | 28.8 seconds |

At an offered rate barely above what the closed-loop test called
sustainable, success collapsed to 72%, with average wait times jumping to
nearly 26 seconds. That's a real capacity cliff the closed-loop numbers
alone never revealed. No rate-limiting exists at the plugin layer itself,
so this is a real, currently unmitigated risk beyond whatever
network-level controls sit in front of it.

**Elevation of privilege** — the category this design handles most
completely. Policy-subset validation on the STS path, and the bounded
active/retired credential count on the static path, both constrain this
directly.

Writing this out this explicitly did something useful beyond just
documentation: it's what pointed me at "no rate-limiting" as a real gap
rather than a hypothetical one, and it's part of what shapes the future
work in the next post.
