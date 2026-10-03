---
layout: post
title: "Why I Gave MinIO Credentials Two Different Lifecycles"
date: 2026-10-01
---

Static API keys are the easiest thing in the world to get wrong. You create
one, it works, and then it just... keeps working. Forever, unless someone
remembers to rotate it. I've seen this exact pattern cause real incidents:
a key leaks into a log line, a config file gets committed somewhere it
shouldn't, and the key is still valid six months later because nothing
about MinIO forces it to expire.

That's not really a criticism of MinIO — it's an S3-compatible object
store, and the whole point of an access key is that it's a stable,
long-lived credential an application can hold onto. MinIO even supports an
optional expiry you can set on a key. But nothing *forces* you to set
one, there's no way to enforce a maximum lifetime across all your keys,
and there's definitely no concept of "rotate this key gracefully without
breaking whoever's currently using it."

So when I extended an existing open-source Vault plugin
([kula/vault-plugin-secrets-minio](https://github.com/kula/vault-plugin-secrets-minio))
that bridges HashiCorp Vault and MinIO, I didn't want to just wrap MinIO's
existing key model in a nicer API. I wanted two genuinely different ways
to get credentials, each suited to a different kind of workload.

### Two paths, one role

Every role you configure in the plugin picks exactly one `credential_type`:
`static` or `sts`. You don't get a mix — a role is one or the other. If a
service needs both behaviors, you configure two roles for it.

**Static credentials** behave like a normal MinIO access key, except the
plugin tracks a real lifecycle for them: `Creation → Active → Retired →
Evicted`. When a credential crosses its `max_ttl`, it doesn't just vanish —
it moves into a grace period where it's `Retired` but still works. That
overlap matters. If I killed a credential the instant it expired, I'd break
every in-flight request, every long-running batch job that cached the old
key, every client that hasn't gotten around to refreshing yet. The grace
period buys everyone time to catch up before the old key is actually
deleted from MinIO.

**STS credentials** are short-lived sessions, modeled directly on AWS's
STS `AssumeRole` pattern (MinIO's STS API is deliberately
AWS-compatible, so any existing AWS SDK client works against it without
modification). These expire on their own, enforced by MinIO itself on
every request — not by the plugin remembering to clean something up. No
grace period needed, because nothing is waiting for an eviction job to
run.

### The part I didn't expect to have to explain

Here's a detail that tripped me up when I was writing this up more
formally: STS credentials aren't actually independent of the static
credential machinery. Every STS request *still* goes through the exact
same credential-resolution logic the static path uses — the plugin
resolves or creates a backing MinIO user for that role, exactly like it
would for a static-type role. The difference is what happens next: a
static-type role just hands that credential back to you. An STS-type role
uses it internally to call MinIO's `AssumeRole` API, and hands you back a
short-lived session instead — the backing credential itself never leaves
the plugin.

So "two credential models" doesn't mean two separate systems. It means
one lifecycle machine, with two different endings.

### Why bother with two paths at all?

Because the tradeoff is real and workload-dependent. A background job that
runs for six hours doesn't want to deal with a credential that expires in
fifteen minutes. A short-lived CI job that only needs five seconds of
access shouldn't be holding a key that's valid for a year. Giving both
options from the same role-based control plane — instead of forcing
everyone onto whichever model is more convenient to implement — was the
actual point.

It also turned out to have a much bigger performance difference than I
expected. That's the next post.
