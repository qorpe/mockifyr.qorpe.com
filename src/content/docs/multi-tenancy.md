---
title: Multi-tenancy
description: Run isolated sets of stubs on one Mockifyr host, selected per request with a header.
---

One Mockifyr host can serve several independent sets of stubs. A tenant is chosen per request with the
`X-Mockifyr-Tenant` header, and a tenant can never see another tenant's stubs.

```bash
curl -X POST http://localhost:8080/__admin/mappings \
  -H 'X-Mockifyr-Tenant: team-payments' \
  -d '{"request":{"method":"GET","url":"/hello"},"response":{"status":200,"body":"payments"}}'

curl http://localhost:8080/hello -H 'X-Mockifyr-Tenant: team-payments'
# → payments
```

The header is honoured on every `/__admin/*` route, on the mock-serving facade, in the
[gRPC](/grpc/) middleware and in the [WebSocket](/websocket/) facade. When the header is absent the
request goes to the **default tenant** — which is what a host that never sets the header uses for
everything.

Isolation is structural, not a filter applied late: every store and engine entry point takes an
explicit tenant, so a code path that forgot to scope itself would not compile.

## Listing tenants

```bash
curl http://localhost:8080/__admin/tenants
```

```json
{ "tenants": ["default", "team-payments"] }
```

This reports the tenants that currently hold stubs. A tenant is not created or registered up front —
it exists because something was stored under it.

## What is not tenant-scoped

Some admin surfaces are deliberately **host-level**: they configure the process, not a set of stubs.
Sending `X-Mockifyr-Tenant` to these routes changes nothing.

| Route | Scope |
|-------|-------|
| `/__admin/recordings/*` | Host-level |
| `/__admin/git/*` | Host-level |
| `/__admin/outbound-trust*` | Host-level |
| `/__admin/ext/*` | Host-level |
| `/__admin/health` | Host-level |
| `/__admin/tenants` | Host-level |

:::caution
This is the part that surprises people. [Record and playback](/record-and-playback/),
[git sync](/persistence/#git-sync), [outbound certificate trust](/https-and-mtls/#outbound-certificate-trust)
and extension management are shared by every tenant on the host. Two teams sharing one Mockifyr
instance share those settings, and a change one makes applies to the other.
:::

## Loading stubs into a named tenant

Stub files read from disk at startup go into the **default tenant**. There is no per-directory tenant
convention at load time.

To populate a named tenant, either use the dashboard's **Import** while that tenant is selected, or
post the bundle with the header:

```bash
curl -X POST http://localhost:8080/__admin/mappings/import \
  -H 'X-Mockifyr-Tenant: team-payments' \
  --data-binary @mappings.json
```

## What follows the tenant

- [Scenario](/scenarios/) state is keyed by **(tenant, scenario name)**, so two tenants running the
  same scenario name advance independently.
- [Environments](/environments/) are tenant-scoped — each tenant owns its own keys and active values.

## In the dashboard

The sidebar has a tenant switcher. Selecting a tenant scopes the whole UI — stubs, scenarios,
environments and imports all act on that tenant until you switch again.

## No per-user identity

Tenancy separates stubs; it does not separate people. Authentication is a **single host-level admin
credential**, so anyone who can log in can switch to any tenant. Tenants are an isolation boundary for
data, not a security boundary between users — see [securing the admin API](/securing-the-admin-api/).

## Related

- [Environments](/environments/) — tenant-scoped configuration values.
- [Admin API](/admin-api/) — the full endpoint reference.


## Declaring a tenant

A tenant exists the moment something is stored under its name — that is still true, and nothing about
it has changed. What you can also do now is **declare** one, which makes it an object you can name,
suspend, bound and offboard:

```bash
curl -X POST http://localhost:8080/__admin/tenants \
  -H 'Content-Type: application/json' \
  -d '{"id":"acme","displayName":"Acme Ltd","storageLimitBytes":10000000}'
```

Declaring is additive. A host that never calls this route behaves exactly as before, and the tenant
listing still reports everything holding stubs.

**Suspending** stops serving without deleting anything:

```bash
curl -X POST http://localhost:8080/__admin/tenants/acme/suspend
```

Requests for a suspended tenant get **403** with `Tenant.Suspended` — deliberately not the 401 an
unknown key gets, because the credential is fine and the account is not, and telling a partner
"unauthorised" sends them to re-check a key with nothing wrong with it. Their stubs, documents and
keys are all still there; `/resume` puts it back exactly as it was.

**Offboarding** removes everything scoped to the tenant and tells you what went:

```bash
curl -X DELETE http://localhost:8080/__admin/tenants/acme
```

```json
{ "removed": { "stubs": 12, "documents": 340, "environmentKeys": 3, "apiKeys": 2, "messages": 18 } }
```

A receipt rather than an "ok": a destructive operation that only says it ran leaves you to go and
check whether it did what you meant.

## Bounding a tenant

`--resource-max-body` caps one document and `--resource-limit` caps one collection, which leaves the
case that matters on a shared host: one tenant seeding a loop across many collections.
`--tenant-storage-limit <bytes>` is the ceiling on everything one tenant holds, and a declared tenant
can carry its own instead.

The refusal names both numbers — the limit and what is already used — and the tenant listing reports
usage before it is reached. Editing an existing document only counts the difference, so a tenant at
its ceiling can still correct what it has rather than being forced to delete first.
