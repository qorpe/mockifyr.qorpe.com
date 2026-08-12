---
title: CLI and configuration
description: Every Mockifyr setting is a command-line flag — and the same name works as an environment variable.
---

Mockifyr has **no configuration file**. Every setting is a command-line flag on the host process.

The host builds its configuration with the standard .NET configuration builder, so **every flag is also
readable as an environment variable of the same name**. That is why this works:

```bash
docker run -p 8080:8080 -e admin-user=alice -e admin-pass='s3cret' ghcr.io/qorpe/mockifyr
```

Command-line arguments win over environment variables when both supply the same key.

:::tip
Use environment variables for anything secret. A password passed on the command line is visible in
`ps` to every user on the machine.
:::

## Flags

### Listeners

| Flag | Default | Effect |
|------|---------|--------|
| `--port <n>` | `8080` | Mock-serving HTTP port. `0` picks an ephemeral port. |
| `--https-port <n>` | unset (no HTTPS listener) | Enables the TLS listener. Both listeners then negotiate HTTP/1.1 and HTTP/2. |
| `--https-keystore <path>` | unset → ephemeral self-signed RSA-2048 certificate | PFX/PKCS#12 server certificate. |
| `--https-keystore-password <p>` | none | Password for the keystore. |
| `--https-require-client-auth` | `false` | Requires a client certificate (mTLS). Applies to the HTTPS listener only. |
| `--https-truststore <path>` | unset | CA anchor the client certificate must chain to. With none set, any well-formed client certificate is accepted. |
| `--https-truststore-password <p>` | none | Password for the truststore. If empty, the file is loaded as a plain certificate. |

See [HTTPS, HTTP/2 and mTLS](/https-and-mtls/).

### Files and the dashboard

| Flag | Default | Effect |
|------|---------|--------|
| `--root-dir <dir>` | unset | Loads `<dir>/mappings/*.json` at startup, persists stub mutations there, and provides `<dir>/__files`, `<dir>/grpc/*.dsc`, `<dir>/environments/` and `<dir>/outbound-trust.json`. |
| `--dashboard <dir>` | unset | Serves the built dashboard under `/__mockifyr`, only if the directory exists. |

### Message channels

| Flag | Default | Effect |
|------|---------|--------|
| `--smtp-port <n>` | unset | Starts the ESMTP capture listener: real mail in, realistic replies out, everything captured into the tenant inbox. The AUTH username selects the tenant. See [Email & SMS mocking](/messages/). |
| `--sms-profile twilio` | unset | Mounts Twilio's send-message endpoint on the mock surface — the official SDK works unchanged; every send is captured. A stub on the same URL still wins. |
| `--message-limit <n>` | `1000` | Per-tenant inbox bound; the oldest message is evicted first. |
| `--kafka-bootstrap <servers>` | unset | Connects the [broker channel](/brokers/): stubs can publish, and capture becomes available. Without it nothing connects. |
| `--kafka-subscribe <topics>` | unset | Comma-separated topics to capture into the message inbox. Publishing works without this; capture needs it. |
| `--kafka-group <id>` | `mockifyr` | Consumer group for capture, so replicas share a subscription rather than each receiving everything. |
| `--amqp-uri <uri>` | — | Connect to AMQP / RabbitMQ (e.g. `amqp://guest:guest@localhost:5672/`). Same channel, second transport — see [Message brokers](/brokers/). |
| `--amqp-subscribe <queues>` | — | Comma-separated queues to consume. Declared on connect, so a queue nobody has created yet is fine. |

### Security hardening

| Flag | Default | Effect |
|------|---------|--------|
| `--decrypt-key <base64>` | none | A 256-bit key (base64 or base64url) enabling **payload cryptography** — decrypting request fields or whole bodies, and protecting responses: a stub declaring `decrypt` matches and templates against the decrypted values. See [request matching](/request-matching/#encrypted-payloads). |
| `--sign-key <base64>` | none | A 256-bit secret enabling **signing**: a stub's `signature` block requires a validly signed request, and its `sign` block adds `Digest` + signature headers to the response. See [request matching](/request-matching/#signed-requests). |
| `--decrypt-key-file <path>` | none | Read decryption keys from a file instead of the command line. The file holds one key per line, newest first, optionally `id: base64`. Re-read on change — see [key rotation](/deploying-in-production/#key-rotation). |
| `--sign-key-file <path>` | none | The same, for signing secrets. |
| `--key-reload-seconds <n>` | `10` | How often a key file is re-read. |
| `--admin-pass-file <path>` | none | Read the admin password from a file, keeping it out of the process listing. Read at startup; the inline `--admin-pass` wins if both are given. |
| `--mask-headers <names>` | none | Comma-separated header names whose values are replaced with `***` **before the serve event is stored** — they never reach the journal, the dashboard, or an export. Case-insensitive. |
| `--mask-body-fields <names>` | none | Comma-separated JSON field names masked the same way, at any depth and inside arrays. A body that is not JSON is stored byte-for-byte. |
| `--tenant-credential <tenant>:<user>:<pass>` | none | Repeatable. An admin credential scoped to one tenant: naming another in `X-Mockifyr-Tenant` answers **403 `Admin.TenantForbidden`**. `--admin-user` stays the system scope that reaches every tenant. |
| `--partner-credential <tenant>:<user>:<pass>` | none | Repeatable. As above, plus **403** on the routes and stub fields through which this host acts on the network — `/__admin/recordings`, `/__admin/outbound-trust`, `/__admin/git`, and any stub declaring `proxyBaseUrl` or a post-serve action. See [handing a credential to a partner](/securing-the-admin-api/#handing-a-credential-to-a-partner). |
| `--allow-outbound-host <host>` | unrestricted | Repeatable. Restrict the hosts this instance may call — proxy stubs and webhooks. `host`, `host:port`, or `*.domain` (subdomains only). A refused webhook is recorded on the journal entry; a refused proxy answers **502**. See [reachable by people you do not employ](/deploying-in-production/#which-hosts-it-may-call). |
| `--max-request-body-bytes <n>` | Kestrel's ~30 MB | Host-wide ceiling on request bodies. Larger is refused with **413** naming which limit it hit. |
| `--tenant-max-request-body <tenant>:<bytes>` | none | Repeatable. Holds one tenant below the ceiling; a value above it is clamped. |
| `--allow-origin <origin>` | none | Repeatable. Browser origins allowed to call the mock and `/__sandbox`. `/__admin` stays same-origin. Matched whole — scheme, host and port. |
| `--tenant-allow-origin <tenant>=<origin>` | none | Repeatable. A tenant's own origin list, **replacing** the host-wide one. `=` and not `:`, since every origin contains a colon. |
| `--block-outbound-routes` | off | While the admin API is unauthenticated, refuse `POST`/`PUT`/`DELETE` on `/__admin/recordings`, `/__admin/outbound-trust` and `/__admin/git` with **403 `Admin.OutboundRoutesBlocked`**, so an open host cannot be used as a forward proxy. Inert once credentials are set — the host says so at startup rather than leaving you to discover it. To scope a *specific* credential away from those routes, issue it with `--partner-credential`. |

:::caution
Masking is opt-in because the journal also backs [verify](/admin-api/#request-journal) and near-miss
diagnostics: a masked `Authorization` header is invisible to a verification that asserts on it. Mask
what must never be retained, not everything.
:::

### Request journal

| Flag | Default | Effect |
|------|---------|--------|
| `--journal-limit <n>` | `1000` | Per-tenant request-journal bound; the oldest entry is evicted first. `<=0` means unbounded. `--max-request-journal-entries` is a kept alias. |
| `--journal-disabled` | off | Record nothing in the request journal — for load tests where the journal is pure overhead. `--no-request-journal` is a kept alias. |

### Operations

Covered end to end in [deploying in production](/deploying-in-production/).

| Flag | Default | Effect |
|------|---------|--------|
| `--metrics` | off | Expose Prometheus metrics at `/__admin/metrics`, on the existing port and outside admin auth — a scraper cannot carry credentials. |
| `--otel-endpoint <url>` | unset | Export traces and metrics over OTLP, e.g. `http://otel-collector:4317`. |
| `--log-json` | off | Structured JSON logs for a log pipeline or SIEM. |
| `--audit` | off | Record every administrative change at `/__admin/audit` — principal, tenant, action, target, outcome — and emit each as an `admin.audit` log line. Reads and unauthenticated attempts are not recorded. |
| `--audit-limit <n>` | `1000` | Per-tenant audit-trail bound; the oldest entry is evicted first. `<=0` means unbounded. |
| `--healthcheck` | — | One-shot probe mode: checks the host and exits `0` or `1`, then stops. Used by the image's `HEALTHCHECK` so no `curl` is needed. |

### Sandbox

| Flag | Default | Effect |
|------|---------|--------|
| `--resource-limit <n>` | `1000` | Per-collection sandbox document bound (`/__admin/resources`); the oldest document is evicted first. |
| `--resource-max-body <bytes>` | `1048576` | Per-document body cap for sandbox resources; larger documents are refused with **413**. |
| `--sandbox-auth` | off | Enables [sandbox API keys](/admin-api/#sandbox-api-keys): `mfk_…` tokens presented as `X-Api-Key` or `Bearer` select the tenant on the mock surface, with optional per-key hourly quotas (**429** + rate headers past the budget). |
| `--tenant-storage-limit <bytes>` | unlimited | Ceiling on the sandbox document bytes one tenant may hold. A declared tenant can carry its own instead. The refusal names the limit and the current usage. |
| `--idempotency` | off | Replay the first response when a write is retried with the same `Idempotency-Key`, instead of running it again. A declared tenant can keep it off while the host has it on. |
| `--idempotency-window <seconds>` | `86400` | How long a stored response stays replayable. |
| `--usage` | off | Keep bounded per-key request counts — total, matched, unmatched, and each refusal separately — plus the most-called paths, readable at `/__admin/usage` and by a partner at `/__sandbox/usage`. Counts only: no headers, no bodies, nothing per request. |
| `--rate-burst <n>/<seconds>` | off | A host-wide burst ceiling counted beside each key's hourly quota — e.g. `--rate-burst 50/10` for fifty requests per ten seconds. It applies to keys with **no** quota too, and whichever limit is about to stop the caller is the one reported in the rate headers. |

### Admin authentication

| Flag | Default | Effect |
|------|---------|--------|
| `--admin-user <u>` | unset | Admin username. |
| `--admin-pass <p>` | unset | Admin password. |

Both must be given together. If only one is set, auth stays off — see
[securing the admin API](/securing-the-admin-api/).

| Flag | Default | Effect |
|------|---------|--------|
| `--oidc-authority <url>` | unset | Authenticate the admin API with OIDC bearer tokens. Signing keys are read from the issuer's discovery document, so provider key rotation needs no restart. See [single sign-on](/securing-the-admin-api/#single-sign-on-oidc). |
| `--oidc-audience <aud>` | unset | The audience a token must carry. Unset accepts any, which is rarely what you want when the directory issues tokens for several applications. |
| `--oidc-client-id <id>` | unset | The **public** client the dashboard signs in with (authorization code + PKCE, no secret). |
| `--oidc-tenant-claim <claim>` | unset | The claim naming the tenant an identity may address; a scoped principal gets **403** on another tenant. No claim means system scope. |
| `--oidc-required-role <role>` | unset | Refuse tokens that do not carry this role. |
| `--oidc-role-claim <claim>` | `roles` | Which claim carries roles. |

OIDC and `--admin-user` can both be configured: people sign in through the provider, machines keep a
credential.

### Persistence

| Flag | Default | Effect |
|------|---------|--------|
| `--litedb <path>` | unset | LiteDB persistence and loader. |
| `--postgres <connstr>` | unset | PostgreSQL persistence and loader. |
| `--redis <connstr>` | unset | Redis persistence and loader. |
| `--change-feed` | `false` | Multi-instance coherence. Only wired when `--postgres` or `--redis` is set. |

See [persistence](/persistence/).

### Serving behaviour

| Flag | Default | Effect |
|------|---------|--------|
| `--global-response-templating` | `false` | Every response renders through the templating engine regardless of the per-stub `transformers` list. See [templating](/templating/). |

### Outbound calls

| Flag | Default | Effect |
|------|---------|--------|
| `--outbound-host-fallback <true\|false>` | `true` | Container-localhost retry for callbacks and proxies. |
| `--trust-proxy-target <host>` | none | Trust that host's certificate on outbound calls. Repeatable, and also accepts a comma- or semicolon-separated list. Exact host match, no wildcards. |
| `--trust-all-proxy-targets` | `false` | Disables outbound certificate verification entirely. |

`--webhook-host-fallback` is a kept alias for `--outbound-host-fallback` from v0.8.1. If both are
present, the new key wins.

:::caution
`--trust-all-proxy-targets` is flag-only — it is not settable from the dashboard. A host that relaxes
outbound TLS prints a `mockifyr: outbound TLS: …` line at startup, so check the log if you are unsure
what a running instance trusts.
:::

Relevant to [proxying](/proxying/) and [webhooks](/webhooks/).

### Git sync

| Flag | Default | Effect |
|------|---------|--------|
| `--git-remote <url>` | unset | Pins Git sync to that remote. Requires `--root-dir` or startup fails. |
| `--git-branch <name>` | `main` | Branch for Git sync. |
| `--git-work-dir <dir>` | `<cwd>/mockifyr-data` | Overrides the default Git working copy. |

:::note
A host started with no flags that finds a `.git` in the default working copy adopts that directory as
its root dir.
:::

## Environment variables that are not flags

Two settings exist only as environment variables:

| Variable | Purpose |
|----------|---------|
| `MOCKIFYR_GIT_TOKEN` | HTTPS Git credential token. |
| `MOCKIFYR_GIT_USERNAME` | HTTPS Git username. |

They are supplied to Git through an inline credential helper: never passed in `argv`, never written to
disk.

## Reserved URL prefixes

| Prefix | Surface |
|--------|---------|
| `/__admin` | The [admin REST API](/admin-api/), and the scope of Basic auth. |
| `/__mockifyr` | The [dashboard](/the-dashboard/). |

Everything else is the mock-serving surface.

## Docker

The image is `ghcr.io/qorpe/mockifyr`, built for `linux/amd64` and `linux/arm64`. It exposes
`8080` and its baked entrypoint is:

```bash
dotnet Mockifyr.Server.dll --port 8080 --dashboard /app/dashboard --root-dir /work
```

Extra flags appended to `docker run` are passed through to that entrypoint:

```bash
docker run -p 8080:8080 ghcr.io/qorpe/mockifyr --global-response-templating
```

To run engine-only with no dashboard, override the entrypoint so that `--dashboard` is dropped.

## Without Docker

Requires the .NET 10 SDK:

```bash
dotnet run --project src/Mockifyr.Server -- --port 8080 --root-dir .
```

## Related

- [Getting started](/getting-started/) — the one-line run.
- [Securing the admin API](/securing-the-admin-api/) — `--admin-user` and `--admin-pass` in context.
- [Persistence](/persistence/) — choosing a store.
- [Admin API](/admin-api/) — the runtime equivalents of several of these flags.
