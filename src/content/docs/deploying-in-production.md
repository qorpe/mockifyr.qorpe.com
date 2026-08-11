---
title: Deploying in production
description: Container posture, Kubernetes probes, the Helm chart, metrics and traces, and the admin audit trail.
---

Running Mockifyr on a laptop needs nothing but the binary. Running it as shared infrastructure — a
sandbox several teams and partners depend on — raises different questions: how does the platform know
the pod is healthy, who is allowed to change a stub, and where does the evidence go. This page covers
the answers.

Everything here is **opt-in**. A host started without these flags behaves exactly as it always has.

## The container

The published image runs as an unprivileged user (UID 1001, GID 0), which satisfies OpenShift's
arbitrary-UID model as well as a plain Kubernetes `runAsNonRoot` policy. Writable state lives under
`/work`, group-writable so the container works whatever UID the platform assigns.

There is no `curl` in the image; the binary probes itself instead:

```bash
docker run --rm -p 8080:8080 ghcr.io/qorpe/mockifyr:latest
```

```bash
docker run --rm ghcr.io/qorpe/mockifyr:latest --healthcheck
```

`--healthcheck` performs one probe and exits 0 or 1 — the image's `HEALTHCHECK` uses it directly.

Each release publishes an SBOM (CycloneDX), build provenance, and a keyless
[cosign](https://docs.sigstore.dev/) signature, so an image can be verified before it is admitted.

## Health and readiness probes

Three endpoints, all outside admin auth, because a probe cannot carry credentials:

| Endpoint | Answers |
| --- | --- |
| `/__admin/health` | name, version, persistence provider, tenant count, enabled capabilities |
| `/__admin/live` | is the process alive — never fails while the host is running |
| `/__admin/ready` | should traffic be routed here (**503** while starting or draining) |

The split matters during a rolling update. On shutdown the host flips readiness off **first**, so the
platform stops sending it new requests while it finishes the ones already in flight; liveness stays
green throughout, so nothing kills the pod mid-drain.

```yaml
livenessProbe:
  httpGet: { path: /__admin/live, port: http }
readinessProbe:
  httpGet: { path: /__admin/ready, port: http }
```

## The Helm chart

The chart in `deploy/helm/mockifyr` renders a Deployment, Service, and — when asked for — an Ingress
or OpenShift Route, a PVC for durable state, and a Prometheus `ServiceMonitor`.

```bash
helm install mockifyr ./deploy/helm/mockifyr --set ingress.enabled=true
```

Admin auth is **on by default**; the password is generated if you do not supply one, stored in a
Secret, and preserved across upgrades. Two guards are deliberate: enabling an Ingress while admin auth
is off fails to render rather than publishing an open admin API, and a `ServiceMonitor` without
metrics enabled fails rather than pointing at nothing. Both are checked in CI on every change.

## Metrics, traces and logs

```bash
mockifyr --metrics --otel-endpoint http://otel-collector:4317 --log-json
```

- **`--metrics`** exposes Prometheus metrics at `/__admin/metrics`, on the existing port and outside
  admin auth, so a scraper needs no configuration beyond the address.
- **`--otel-endpoint`** exports traces and metrics over OTLP.
- **`--log-json`** switches logs to structured JSON for a log pipeline or SIEM.

Serving metrics are labelled by `tenant`, `matched` and `method`. Stub id and URL are deliberately
**not** labels: a mock host can hold thousands of stubs, and a metrics backend that receives one time
series per stub will fall over.

## Key rotation

Cryptographic keys and the admin password can come from files rather than the command line, which
keeps them out of the process listing — and, for keys, makes rotation something you do without
restarting anything.

```bash
mockifyr --decrypt-key-file /keys/decrypt --sign-key-file /keys/sign --admin-pass-file /keys/admin
```

A key file holds a **ring**: one key per line, newest first, optionally with an id.

```
# /keys/decrypt
partner-2026-08: TmV3IGtleSBtYXRlcmlhbCBnb2VzIGhlcmUgLi4uLi4uLi4=
partner-2026-02: T2xkIGtleSwgc3RpbGwgYWNjZXB0ZWQgd2hpbGUgd2UgZHI=
```

**New payloads are produced with the newest key; every key in the file is still accepted.** That is
what makes a rollover safe:

1. **Add** the new key as the first line. Partners that have already switched work immediately.
2. **Drain** — leave the old key in place while partners migrate. Both work, at the same time.
3. **Remove** the old line. Removing it is what actually retires the key; until then it keeps working.

No step restarts the host. The file is re-read when it changes (`--key-reload-seconds`, default 10),
and `/__admin/health` reports how many keys are active per capability, so you can confirm a rollover
landed instead of assuming it:

```json
{ "cryptography": { "payloadDecryption": true, "decryptKeys": 2, "signKeys": 1 } }
```

It reports counts, never key material.

:::note
Comment a line out with `#` to withdraw a key temporarily — a commented key is not active. A line
that is not a valid 256-bit key is skipped rather than failing the file, so a typo during a rollover
cannot stop the host from starting; the active-key count is how you notice.
:::

### In Kubernetes

```bash
helm install mockifyr ./deploy/helm/mockifyr \
  --set cryptography.existingSecret=mockifyr-keys \
  --set cryptography.mountAsFiles=true
```

The Secret is mounted read-only at `/keys` and no key appears in the pod's arguments or environment.
Updating the Secret rewrites the mounted file and the host picks it up on its next poll — which is
why the file source polls the modification time rather than watching for filesystem events, since
Kubernetes updates a mounted Secret by swapping a symlink.

A key file that is briefly unreadable or half-written leaves the last good keys in place, so a
rotation script that truncates before writing cannot disarm a running host.

## Reachable by people you do not employ

Three settings a host on the open internet needs that a host on your laptop does not. All three are
**off by default** — a host without them behaves exactly as it always has.

### Which hosts it may call

A mock server calls out for a living: proxy stubs forward, webhooks fire. On a shared network that is
also a way *into* it. Name the hosts you actually integrate with and everything else is refused:

```bash
mockifyr --allow-outbound-host partner.example \
         --allow-outbound-host '*.hooks.partner.example' \
         --allow-outbound-host internal-qa.example:8443
```

An entry without a port allows any port on that host. `*.domain` covers **subdomains only** — add the
bare domain as its own entry if you want it too, since allowing `*.internal.example` rarely means
`internal.example` itself.

The check runs against the URL a webhook actually resolves to, not the template it was written as, so
a callback address that comes from the request (`{{request.headers.X-Callback}}`) is checked at the
moment it is known. A refused webhook is recorded on the request that triggered it — look at the
journal entry's detail, where a failed delivery would appear. A refused proxy answers **502** naming
the host and what is allowed.

:::note
A `publish` action names a topic on the broker you started the host with, so there is no per-stub
outbound host for an allowlist to decide. Restrict broker access with your broker's own controls.
:::

### How large a request body it will read

Without this, the default is roughly 30 MB for everyone:

```bash
mockifyr --max-request-body-bytes 1048576 \
         --tenant-max-request-body acme:262144
```

The host value is a **ceiling**: a per-tenant value above it is clamped, not honoured. Something
larger is refused with **413**, and the message says which of the two limits it hit — so you know
whether to raise the tenant's number or the host's.

### Which browsers may call it

A partner's browser application cannot call your sandbox until you say so. Name the origins:

```bash
mockifyr --allow-origin https://app.partner.example \
         --tenant-allow-origin 'acme=https://acme-portal.example'
```

Origins are matched whole — scheme, host and port. A tenant that names its own origins uses **only**
those; a tenant that names none inherits the host-wide list. The separator for a per-tenant entry is
`=` rather than `:`, because every origin already contains a colon.

This covers the mock and [`/__sandbox`](/sandbox/#what-a-partner-can-see-for-themselves), so a browser
partner can read their own OTP. It deliberately does **not** cover `/__admin`, which stays same-origin:
you reach the admin API from the dashboard that served it.

:::caution
An origin nobody allowed simply gets no CORS headers — the browser is what enforces the rule, so
non-browser clients are unaffected. If a `fetch` fails with a CORS error, the origin is missing from
this list; the request itself was answered.
:::

## Backup and restore

```bash
curl -s http://mockifyr:8080/__admin/backup > backup-$(date +%F).json
```

One archive per tenant, covering everything an operator authored: stubs, environment keys, sandbox
documents, API keys and scenario states. Settings → **Backup and restore** does the same from the
dashboard. Full request and response shapes are in the [admin API reference](/admin-api/#backup-and-restore).

A runbook that works:

1. **Before an upgrade**, take an archive per tenant and store it where your secrets live — it carries
   API key verifiers, which is what lets consumers keep using the keys they already hold.
2. **To restore**, bring up a host on the target version and `POST` each archive to
   `/__admin/restore` with that tenant's `X-Mockifyr-Tenant` header.
3. **Verify** by calling one stub per tenant and checking that an environment-templated response
   resolves — that proves the environment section landed, not just the stubs.

A restore **replaces** what the archive covers rather than merging, so a restored host is the host
that was backed up. Restoring is refused outright if the file is not a Mockifyr archive, so pointing
it at a stub bundle by mistake changes nothing.

Deliberately not in the archive: the request journal, the message inbox and quota counters. They are
observations of what happened, not configuration — restoring them would fabricate a history the new
host never served. Host configuration (outbound trust, TLS, flags) is not there either; that lives
with your Helm values.

## The audit trail

The request journal records what the host **served**. `--audit` records what was **changed**:

```bash
mockifyr --audit --audit-limit 1000 --log-json
```

Every mutating call under `/__admin` — from the API, the dashboard, or your own tooling — is appended
to a tenant-scoped, append-only trail:

```json
{
  "id": "62a8a64f-4f86-4d36-a30b-0f1e373a8753",
  "timestamp": "2026-07-30T13:25:14.32Z",
  "principal": "tenant:acme",
  "tenant": "acme",
  "action": "DELETE /__admin/mappings/5a5bb853-…",
  "target": "5a5bb853-…",
  "outcome": 200
}
```

Read it at `GET /__admin/audit?limit=200` (newest first, tenant-scoped) or on the dashboard's **Audit**
screen.

What is and is not recorded, and why:

- **Reads are not.** They are already in the request journal, and mixing them in would evict the
  changes you came looking for.
- **Refused changes are.** A **403** cross-tenant attempt or a **422** malformed stub is recorded with
  its real outcome — an entry never claims success for something the host rejected.
- **Unauthenticated attempts are not.** They have no principal to name, and recording them would let
  anyone flush the trail by repetition. They appear in metrics and access logs instead.
- **The principal is a label, never a credential**: `system`, `tenant:acme`, or `anonymous`. No part of
  a password or key is ever stored.
- **Nothing on the API can rewrite it.** Entries are written by the host as a side effect of the change
  they describe; there is no route that appends, edits, or clears one.

The trail is held in memory and bounded by `--audit-limit` (default 1000 per tenant, oldest evicted
first), so it disappears with the pod — on purpose. Each entry is **also** emitted as a structured
`admin.audit` log line, so with `--log-json` the durable copy lives wherever your SIEM's retention
policy says it should, not in a mock host's heap.

:::note
`/__admin/health` reports whether auditing is enabled. An empty trail is ambiguous on its own —
"nothing changed" and "nobody is recording" look identical — so the dashboard tells you which it is.
:::
