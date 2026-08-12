---
title: The integration sandbox
description: Give a partner a live, stateful API to build against — seeded data, real CRUD, their own key, and a quota — without standing up the real service.
---

A stub answers the same way every time. That is exactly what you want in a test and exactly what you
do not want when somebody has to *build* against your API for two weeks.

The sandbox is the other half: tenant-scoped JSON collections your stubs read and write, so a `POST`
creates something a later `GET` returns. Add an API key and a quota and it becomes something you can
hand to a partner integration team.

Everything here is opt-in. A host that never touches it behaves exactly as before.

## The pieces

| | What it is |
|---|---|
| **Resources** | Tenant- and collection-scoped JSON documents — the data |
| **The `state` directive** | What turns a matched stub into a CRUD operation on those documents |
| **OpenAPI import** | A spec in, a working stateful sandbox out |
| **API keys** | A credential the partner carries; it selects their tenant and carries their quota |

## Seed some data

Collections are created by writing to them — there is nothing to declare first.

```bash
curl -X POST http://localhost:8080/__admin/resources/orders/seed \
  -H 'Content-Type: application/json' \
  -d '[{"id":"ord-1","total":42},{"id":"ord-2","total":17}]'
```

Seeding is transactional: every item is validated before anything lands, so a bad element leaves the
collection untouched. Elements may carry a string `id`; absent ones are generated.

```bash
curl http://localhost:8080/__admin/resources          # collections + counts
curl http://localhost:8080/__admin/resources/orders   # {documents, total}
```

An unknown collection lists as an **empty page**, not a 404 — "no rows yet" is what a sandbox
operator means. Full routes and error codes are in the [admin API reference](/admin-api/#sandbox-resources).

## Make stubs read and write it

A stub response declares a `state` directive and becomes a CRUD operation. The created document is
addressable by the next request:

```json
{
  "request": { "method": "POST", "urlPath": "/api/orders" },
  "response": {
    "status": 201,
    "body": "{\"id\":\"{{state.id}}\",\"order\":{{state.body}} }",
    "state": { "operation": "create", "collection": "orders" }
  }
}
```

```json
{
  "request": { "method": "GET", "urlPathPattern": "/api/orders/[^/]+" },
  "response": {
    "status": 200,
    "body": "{{state.body}}",
    "state": { "operation": "read", "collection": "orders", "id": "{{request.pathSegments.[2]}}" }
  }
}
```

`create`, `read`, `update`, `delete` and `list` are the operations; the result renders as
`{{state.id}}`, `{{state.body}}`, `{{state.version}}`, `{{state.count}}` and `{{state.list}}`.
Declaring the directive **is** the templating opt-in — no `response-template` transformer needed. The
full syntax, including `missStatus`, lives with the [response directives](/responses/#stateful-responses-the-state-directive).

What a stub writes, the admin API and the dashboard see immediately: it is one store, not a copy.

## From a spec to a working sandbox

If you already have an OpenAPI 3.x document, you do not have to author any of the above:

```bash
curl -X POST 'http://localhost:8080/__admin/openapi/import?stateful=true' \
  -H 'Content-Type: application/yaml' \
  --data-binary @petstore.yaml
```

- **Without `stateful`** — one stub per operation. Declared examples serve as-is; example-less
  schemas get a synthesized sample, with `uuid`, `email` and `uri` formats backed by the
  [random helpers](/template-helpers/) so values look alive rather than identical.
- **With `stateful=true`** — resource-shaped path pairs (`/pets` plus `/pets/{id}`) additionally
  become a wired CRUD set on a collection named after the path. Import, then `POST` a pet and `GET`
  it back.

JSON and YAML both work. Import is transactional, and imported stubs are ordinary mappings
afterwards — listable, editable, exportable, indistinguishable from hand-written ones.

Three refusals are worth knowing, because they are deliberate and typed rather than mysterious:
external `$ref`s (URL **or** file) are refused before parsing with the offending pointer named — the
host never fetches anything — a spec over 5 MiB is refused, and schema recursion deeper than 32
levels (a cyclic `$ref`) is refused instead of hanging.

The dashboard's **Add stub → OpenAPI** channel does the same thing with a file picker.

## Filtering, sorting and summary shapes

Real APIs let you narrow a list. So does the sandbox — on the documents themselves, with no extra
stub per query shape:

```bash
curl 'localhost:8080/orders?status=settled&_sort=-total&_fields=id,total'
```

| Form | Means |
|---|---|
| `?status=settled` | the field equals the value |
| `?note:contains=urgent` | the field's text contains it |
| `?ref:matches=^INV-\d+$` | the field matches a regular expression |
| `?note:absent=true` | the field is not there at all (`false` — it is) |
| `?_sort=total` · `?_sort=-total` | ascending · descending |
| `?_fields=id,total` | return only these fields |

Those first four are the same words a stub uses to match a request, deliberately: one vocabulary, not
two. Several filters combine with AND. `limit`, `offset`, `_sort` and `_fields` are the list's own
parameters, so a document field with one of those names cannot be filtered on.

The same query works on `GET /__admin/resources/{collection}`, where `total` counts the documents that
**matched** — not the ones in the collection — so paging over a filtered list is honest.

:::caution
For filtering to work on a served route, the stub must match with **`urlPath`**, not `url`. `url`
matches the path *and* the query string, so a stub written as `"url": "/orders"` stops matching the
moment somebody filters — the request that most wants this returns a 404.

```json
{ "request": { "method": "GET", "urlPath": "/orders" },
  "response": { "body": "{{state.list}}", "state": { "operation": "list", "collection": "orders" } } }
```
:::

A few deliberate details: numbers sort as numbers (`9` before `250`, not after), a document missing the
sort field goes last whichever way you sorted, and a selected field the document does not have is
simply absent from the result rather than present and null.

There is no query language here — no joins, nothing across collections. Filter, sort, select. A mock
that is harder to reason about than the service it replaces has stopped being useful.

## Relations between collections

Real APIs are hierarchical. An order belongs to a customer, an account to a client, a transaction to
an account — and a sandbox that does not know this is worse than one that has no data at all,
because it answers confidently with somebody else's.

If your specification contains `/customers/{customerId}/orders`, the import above already declared
the relation: that path *is* the sentence "orders belong to customers, keyed by `customerId`". You
get the behaviour without writing anything.

```bash
curl -X POST localhost:8080/customers -d '{"name":"Ada"}'      # → Location: /customers/<a>
curl -X POST localhost:8080/customers -d '{"name":"Grace"}'    # → Location: /customers/<g>

curl -X POST localhost:8080/customers/<a>/orders -d '{"total":100}'
curl -X POST localhost:8080/customers/<g>/orders -d '{"total":250}'

curl localhost:8080/customers/<a>/orders     # → just the 100
curl localhost:8080/customers/<g>/orders     # → just the 250
```

Three more things follow from the same declaration:

- **An order cannot be created under a customer who does not exist.** `POST /customers/99/orders`
  answers `404` — the route names something that is not there. A reference in the *body* that does
  not resolve answers `422` instead: the request reached a real place and its payload is what is
  wrong.
- **An id does not travel between parents.** Reading one customer's order through another
  customer's path is a `404`, not a lucky hit.
- **Deleting a customer who still has orders is refused** with a `409` naming what is in the way.

### Choosing what a delete does

`restrict` is the default because that is what the APIs you are standing in for mostly do — deleting
a customer at a payment provider does not delete their past charges. Change it when the API you are
modelling really does cascade:

```bash
curl -X PUT localhost:8080/__admin/relations/orders \
  -d '{"belongsTo":[{"collection":"customers","via":"customerId","onDelete":"cascade"}]}'
```

`orphan` is the third option: delete the parent, leave the children with a key that no longer
resolves. Some real APIs do exactly that, so it is available rather than assumed to be a mistake.

### Declaring one by hand

For a sandbox you built without a specification, declare the relation yourself. `via` is the field in
the child document that holds the parent's id:

```bash
curl -X PUT localhost:8080/__admin/relations/invoices \
  -d '{"belongsTo":[{"collection":"customers","via":"billTo"}]}'
```

If your documents do not carry such a field at all, that is fine — nothing is added to them. The
sandbox records the parent alongside the document instead, so what you stored is returned back to
you byte for byte and stays exactly what your contract describes.

### What it deliberately does not do

Relations make the sandbox behave like the API it stands in for. They do not turn it into a database:
there are no joins, no transactions across collections, no query language and no schema migrations.
A mock that is harder to reason about than the service it replaces has stopped being useful.

Two collections may reference each other, and a collection may reference itself (`managerId` on
`employees` is a real model) — a relation is checked when its key is present, so there is never a
chicken-and-egg problem creating the first document.

## Datasets — a whole scenario in one call

"The delinquent customer" is not a customer. It is a customer, three orders, two failed payments and a
dunning record — across four collections and only meaningful together. Seeding them one collection at a
time means a script, and then the scenario lives in the script rather than in the sandbox.

Declare it once — the document templates contain single quotes, so this is a file rather than an
inline `-d` string:

```json title="delinquent.json"
{
  "seed": 42,
  "items": [
    { "collection": "customers", "count": 1, "id": "customer-{{index}}",
      "document": { "name": "{{random 'Name.fullName'}}", "status": "delinquent" } },
    { "collection": "orders", "count": 3, "id": "order-{{index}}",
      "document": { "customerId": "customer-0", "total": 100 } }
  ]
}
```

```bash
curl -X PUT localhost:8080/__admin/datasets/delinquent \
  -H 'Content-Type: application/json' --data-binary @delinquent.json
```

Then load it, run your tests, and put the sandbox back:

```bash
curl -X POST localhost:8080/__admin/datasets/delinquent/load     # → { "loaded": 4 }
curl -X POST localhost:8080/__admin/datasets/delinquent/unload   # → { "removed": 4 }
```

### What it does for you

- **Order is worked out for you.** Write the items in any order; the loader puts parents before the
  children that [belong to them](#relations-between-collections), because integrity would otherwise
  refuse an order whose customer does not exist yet. Unloading reverses that, since a `restrict`
  relation refuses to delete a customer who still has orders.
- **All or nothing.** If anything fails — a template that will not render, a reference to a document
  that is not there — every document the load already wrote is removed. A half-loaded scenario is one
  nobody can reason about.
- **Loading twice leaves one copy.** The previous load is removed first, so `load` before each test run
  is a single line rather than a line and a guard.
- **Unload only removes what that load created.** Documents you or a colleague added by hand stay.

### Generating plausible data

`count` and the [Faker helpers](/template-helpers/) turn "two hundred customers" into a line:

```json
{ "collection": "customers", "count": 200,
  "document": { "name": "{{random 'Name.fullName'}}", "email": "{{random 'Internet.email'}}" } }
```

Templates also see `{{index}}` — the document's position, which is what makes `"id": "customer-{{index}}"`
work — and `{{dataset}}`, the dataset's own name.

**`seed` makes it reproducible.** The same dataset with the same seed produces the same two hundred
customers on every load, which is what lets a dataset be the basis of a regression test rather than
just a pile of data. Leave it out and each load generates fresh values. Seeding affects only that load —
stubs serving random values elsewhere keep generating new ones.

:::note
A document template is written as **JSON**, not as an escaped string, and it does not have to be valid
JSON before rendering: `{"total": {{random 'Number.digit'}}}` is fine, and is checked after the helpers
have run. A dataset is capped at 10 000 documents in total.
:::

## Check that it still tells the truth

A mock that has quietly drifted from the API it models is worse than no mock, because it manufactures
confidence: the upstream adds a required field, the stubs do not follow, the tests stay green, and
production breaks.

The same document you imported from can be used to check the stubs you have now:

```bash
curl -X POST http://localhost:8080/__admin/openapi/verify --data-binary @openapi.yaml
```

```json
{ "conforms": false, "operationsInSpec": 12, "operationsCovered": 9,
  "findings": [
    { "kind": "schemaViolation", "method": "GET", "path": "/orders/{id}", "stubId": "…",
      "detail": "/total: Required properties [\"total\"] were not present" },
    { "kind": "undeclaredOperation", "method": "GET", "path": "/legacy/orders", "stubId": "…",
      "detail": "The specification declares no such operation…" } ] }
```

Four kinds of finding — a stub answering an operation the spec no longer declares, an operation no
stub answers, an undeclared status, and a body that violates the declared schema — plus coverage
counts, because "conforms" on an empty stub set is true and useless.

**It reports; it never changes anything.** Which side is wrong is a judgement about your system.

Wiring it into CI is the point: fail the build when the mock and the contract disagree, before the
disagreement reaches somebody's integration test.

### Against reality, not just against the document

A document can be stale too. With a [recording session](/record-and-playback/) live against the real
upstream, the same question can be asked of reality:

```bash
curl -X POST http://localhost:8080/__admin/recordings/start -d '{"targetBaseUrl":"https://api.real.example.com"}'
# …drive your integration tests through the mock as usual…
curl -X POST http://localhost:8080/__admin/recordings/verify
```

It compares what the upstream just returned against what your stubs *would* have answered: a field the
upstream grew, a field only the stub has, a changed type, a changed status, a call no stub matches.

The comparison is **structural, never literal** — ids, timestamps and totals differ between
environments and between minutes, and reporting them would bury the findings that matter. And it
serves nothing while it looks: no journal entry, no scenario advances.

### Is the client staying inside the contract?

The same document, pointed at your traffic instead of your stubs:

```bash
curl -X POST http://localhost:8080/__admin/requests/verify --data-binary @openapi.yaml
```

Reports calls to operations the contract never declared, missing required query parameters and
headers, and request bodies the schema forbids — all of which work perfectly against a mock that is
more permissive than the real service, and fail the first time they meet it.

:::note
Three things are deliberately **not** reported, because a report with false findings is one nobody
reads: a templated body (it is not JSON until a request renders it), a stub matching by regular
expression (a spec path cannot be compared to one without guessing), and an operation whose schema the
document omits.
:::

## Hand it to a partner

Start the host with `--sandbox-auth` and issue a key per consumer:

```bash
curl -X POST http://localhost:8080/__admin/apikeys \
  -H 'Content-Type: application/json' \
  -d '{"name":"acme-integration","quotaPerHour":1000}'
```

```json
{ "id": "…", "name": "acme-integration", "key": "mfk_…", "prefix": "mfk_a1b2c3d4", "quotaPerHour": 1000 }
```

**The token appears once.** Every later view carries only the 12-character display prefix — it is
stored salted and hashed, so a leaked backup is not a leaked key, and neither is a screenshot of the
Access screen.

The partner sends it as `X-Api-Key` or `Authorization: Bearer`:

```bash
curl -H 'X-Api-Key: mfk_…' http://localhost:8080/api/orders
```

The key **selects the tenant**, ahead of the host/header rules — so a partner cannot reach another
tenant's sandbox by changing a header, and does not need to know your tenancy scheme at all. An
invalid or revoked key is an honest **401**, never a silent fall-through to somebody else's data.

A key with a `quotaPerHour` is limited over a fixed hourly window. Counted responses carry
`X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; going over gets a **429** with
`Retry-After` — the behaviour a real gateway has, so the partner's retry logic gets exercised here
instead of in production. `quotaPerHour: 0` (or absent) means unlimited, and unlimited keys emit no
rate headers.

`--rate-burst <n>/<seconds>` adds a second window beside that one, for the whole host. The two
protect different things — fifty requests in a second is a runaway loop, fifty thousand in a day is a
consumer who should be paying — and one number cannot say both. Because it protects the host rather
than a partner's budget, it applies to keys with **no** `quotaPerHour` as well: "unlimited" describes
a budget, not permission to melt the host. Both windows count every request, even one the other
already refused, and the rate headers report whichever limit is about to stop the caller. When both
refuse, `Retry-After` names the later reopening — a `Retry-After` that is too short just invites a
retry into a door that is still shut.

```bash
mockifyr --sandbox-auth --rate-burst 50/10
```

Revoking is immediate:

```bash
curl -X DELETE http://localhost:8080/__admin/apikeys/{id}
```

:::caution
A sandbox key is a **data-plane** credential. It never authenticates `/__admin/*` — the admin surface
only accepts its own [admin authentication](/securing-the-admin-api/), on both carriers. Control
plane and data plane never blur.
:::

### What a partner can see for themselves

Calling the mock is half of integrating against it. The other half is looking: *did the OTP arrive?
what did that webhook actually contain? why did my call 404?* Without that, every question becomes a
message to you.

`/__sandbox/*` answers those, with the same key, for that key's tenant only:

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/__sandbox/` | Which tenant this key belongs to |
| `GET` | `/__sandbox/requests?unmatched=true` | Their request journal — the "why did it 404" surface |
| `GET` | `/__sandbox/messages` | Captured e-mail, SMS and broker messages |
| `GET` | `/__sandbox/messages/otp?recipient=…` | The one-time code from the newest matching message |
| `GET` | `/__sandbox/resources` · `/{collection}` · `/{collection}/{id}` | Their sandbox data |
| `GET` | `/__sandbox/environments` | Which value each key currently resolves to (never a [secret](#secret-values)) |
| `POST` | `/__sandbox/resources/reset` · `/messages/reset` · `/requests/reset` | Start the next test run clean |

```bash
# the gesture this exists for
curl -H 'X-Api-Key: mfk_…' 'http://localhost:8080/__sandbox/messages/otp?recipient=+15551234567'
# → { "otp": "482915", "messageId": "…", "receivedAt": "…" }
```

Three things worth knowing:

- **It is not `/__admin` with a smaller menu.** It is a separate surface, and the rule above still
  holds without exception — a sandbox key authenticates nothing on the admin API.
- **There is no tenant header here at all.** The tenant comes from the key. Sending
  `X-Mockifyr-Tenant` naming somebody else is simply ignored, so there is nothing to get wrong.
- **It exists only with `--sandbox-auth`.** Without keys there is no way to tell one partner from
  another, so the whole namespace answers 404 rather than trusting everyone.

Reads and resets only. Nothing here creates stubs, changes configuration, or reaches the network —
for a credential that also needs to *write* configuration, see
[`--partner-credential`](/securing-the-admin-api/#handing-a-credential-to-a-partner).

## In the dashboard

The **Sandbox** group has two screens: **Resources** browses collections and documents and lets you
edit or reset them, and **Access** issues keys, shows each key's usage against its quota, and revokes
them. A newly issued token is revealed once, with a copy button — after that the screen shows the
prefix like everything else does.

## Operating it

**Durability.** Seeded documents survive a restart on every [persistence backend](/persistence/)
(file system, LiteDB, PostgreSQL, Redis) — deletes and resets included. Issued keys persist the same
way. A host with no persistence keeps everything in memory, which is usually what you want in CI.

**Several replicas.** With `--change-feed`, documents written on one replica reach the others without
a restart, keeping the version and timestamps the writing replica gave them.

**Bounds.** A collection holds `--resource-limit` documents (default 1000, oldest evicted first when
a *create* overflows — an update never evicts), and one document may be `--resource-max-body` bytes
(default 1 MiB, an honest **413** beyond it). Bodies must be well-formed JSON (**422** otherwise) and
are otherwise opaque: they round-trip byte for byte, unicode and all.

**Tenancy.** Collections, documents and keys are all [tenant-scoped](/multi-tenancy/). The same
collection name and the same document id in two tenants are two different documents.

**Quotas across replicas.** With `--redis`, the request count lives in Redis: two replicas enforce
the number on the key rather than that number each, and a restart mid-hour does not refund what was
already spent. Without a shared backend the count is in memory and starts fresh on restart — fine for
one host, and the reason a multi-replica deployment should point at Redis.

**Keys across replicas need `--change-feed`.** Issuing and revoking are announced on the same feed
that carries stubs, environment keys and sandbox documents. Without it, another replica will not see
an issued key — or a **revoked** one — until it restarts, and the host says so at startup rather than
leaving you to find it in the logs.

## Related

- [Responses](/responses/#stateful-responses-the-state-directive) — the `state` directive in full.
- [Admin API](/admin-api/#sandbox-resources) — every route, parameter and error code.
- [Multi-tenancy](/multi-tenancy/) — how a tenant is chosen when there is no key.
- [Persistence](/persistence/) — what survives a restart, and where it is stored.
- [CLI](/cli/) — `--sandbox-auth`, `--rate-burst`, `--resource-limit`, `--resource-max-body`.
