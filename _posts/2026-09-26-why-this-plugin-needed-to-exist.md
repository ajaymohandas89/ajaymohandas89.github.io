---
layout: post
title: "Why This Plugin Needed to Exist"
series: vault-minio
date: 2026-09-26
description: "Static keys that never expire by default, and the gap between what MinIO provides and what a real credential lifecycle needs."
---

Static API keys are the easiest thing in the world to get wrong. You create
one, it works, and then it just... keeps working. Forever, unless someone
remembers to rotate it. I've seen this exact pattern cause real incidents:
a key leaks into a log line, a config file gets committed somewhere it
shouldn't, and the key is still valid six months later because nothing
forced it to expire.

### MinIO's part in this

MinIO is an S3-compatible object store, and like S3, its access keys are
built to be stable, long-lived credentials an application can hold onto.
MinIO does support an optional expiry you can set on a key — but nothing
forces you to set one, there's no way to enforce a maximum lifetime across
all your keys, and there's no concept of "rotate this key gracefully
without breaking whoever's currently using it." By MinIO's own
documentation, a key created without an explicit expiry simply doesn't
expire until someone manually removes it.

That's not a flaw in MinIO — it's just not the problem MinIO set out to
solve. Someone else has to own the lifecycle.

### What Vault already does, and doesn't

HashiCorp Vault has a generic lease and expiration subsystem built in:
dynamic secrets engines can attach a lease to a secret and Vault will
track and revoke it on a schedule. But that subsystem exists for engines
that opt into it — it doesn't retroactively give MinIO's own access keys
an expiration concept they don't have. An existing open-source plugin
([kula/vault-plugin-secrets-minio](https://github.com/kula/vault-plugin-secrets-minio))
already bridged Vault and MinIO at a basic level. What it didn't have was a real lifecycle, or a way to issue
genuinely short-lived credentials instead of just long-lived ones with a
nicer API in front of them.

### Two kinds of "I need access to MinIO"

In practice, "give me a credential" means two very different things
depending on who's asking:

- A long-running service that's going to hold a credential for hours or
  days wants something stable — it doesn't want to re-authenticate every
  fifteen minutes.
- A short CI job, or a security-conscious workload, wants the opposite:
  access that expires on its own, with nothing standing around to leak
  after the job finishes.

Trying to force both of these into one credential model always means
compromising one of them. So the plugin I built offers both, from the
same role-based control plane, and lets you pick per role which one you
actually need — not which one happened to be easier to implement first.

### Where "zero trust" actually comes in

I didn't want to just rename the problem "zero trust" and call it done.
The actual principle that mattered here, from NIST's own framing of it, is
that no credential should be trusted for longer than it's operationally
necessary, and access should be scoped and time-bounded by default rather
than persistent. That's a design constraint, not a marketing label — it's
the reason the static path has a bounded lifecycle with real eviction
instead of just "a key that happens to have an expiry field," and it's
the reason the short-lived path exists at all instead of everyone just
getting static keys because they're simpler to issue.

The next post covers how the actual architecture pulls this off —
including a detail about how the two credential types are a lot less
independent than I expected going in.

---

*Thanks to Kula for the original
[vault-plugin-secrets-minio](https://github.com/kula/vault-plugin-secrets-minio),
which this work builds on. The plugin's foundation (MinIO configuration,
roles, and per-request user provisioning) started there. The credential
lifecycle with its grace period, the STS path, multiplexing support, and
the testing work described in this series are my extensions.*
