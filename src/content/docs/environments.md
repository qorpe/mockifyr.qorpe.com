---
title: Environments
description: Tenant-scoped configuration keys that stubs reference as {{key}} and Mockifyr resolves at serve time.
---

An environment key is a named value a stub can reference instead of hard-coding it. A tenant owns a set
of keys; each key holds several **named values**, one of which is **active**. Switching the active value
changes every stub that references the key, without editing a single stub.

```json
{
  "request": { "method": "GET", "url": "/config" },
  "response": {
    "status": 200,
    "body": "{\"api\":\"{{baseUrl}}\"}"
  }
}
```

The reference is stored **verbatim** — the stub on disk still says `{{baseUrl}}` — and is resolved when
the request is served.

## Where resolution applies

| Surface | Resolved |
|---------|----------|
| Response body | Yes |
| Response headers | Yes |
| [Proxy](/proxying/) target | Yes |
| [Webhook](/webhooks/) URL, body and headers | Yes |

Environments are **tenant-scoped**: each tenant has its own keys and its own active values. See
[multi-tenancy](/multi-tenancy/).

## When it runs

Resolution happens **before** Handlebars and **before** the transformer guard.

:::tip
That ordering is the point: `{{baseUrl}}` works on a stub that has **no** `response-template`
transformer. You do not have to opt a stub into [templating](/templating/) to use a configuration key.
:::

## Substitution semantics

These rules are narrow on purpose — the goal is that a stub which does not use environments comes
through untouched.

- **Only bare identifiers that resolve to a defined key are replaced.** Everything else passes through
  byte-identical, so Handlebars expressions such as `{{jsonPath request.body '$.id'}}` are left alone
  and evaluated later by the template engine as normal.
- **Substituted values are not rescanned.** There is no chaining and no recursion: if a key's value
  itself contains `{{other}}`, that text stays literal.
- **Lookup is case-sensitive.** `{{baseUrl}}` and `{{baseurl}}` are different references.
- **An undefined reference survives as literal text.** `{{typo}}` comes back in the response as the
  characters `{{typo}}`, not as an empty string.

:::note
Leaving an undefined reference visible is deliberate. An empty string would look like a legitimate
value and a misspelt key would fail silently in a downstream assertion; the literal `{{typo}}` in the
response body points straight at the mistake.
:::

## Reserved key names

A key named after a built-in template helper is refused:

```json
{ "error": "Environment.ReservedKey" }
```

…with HTTP **400**.

:::caution
The reserved list is a deliberate **superset** of the actual helper list, so a few names are refused
even though no helper by that name exists. If a key name is rejected and you cannot find a matching
helper in the [template helper reference](/template-helpers/), that is why — pick another name.
:::

## Admin endpoints

| Method | Route | Body |
|--------|-------|------|
| `GET` | `/__admin/environments` | — |
| `PUT` | `/__admin/environments/{key}` | `{activeValue, values:[{name,value}]}` |
| `PUT` | `/__admin/environments/{key}/active` | `{"activeValue":"…"}` |
| `DELETE` | `/__admin/environments/{key}` | — |
| `POST` | `/__admin/environments/reset` | — |

`GET` returns every key with its values and the value currently in effect:

```json
{
  "environments": [
    {
      "key": "baseUrl",
      "activeValue": "staging",
      "resolved": "https://staging.example.com",
      "values": [
        { "name": "staging", "value": "https://staging.example.com" },
        { "name": "prod", "value": "https://api.example.com" }
      ]
    }
  ]
}
```

Define a key and its values:

```bash
curl -X PUT http://localhost:8080/__admin/environments/baseUrl \
  -H 'X-Mockifyr-Tenant: team-payments' \
  -d '{"activeValue":"staging","values":[
        {"name":"staging","value":"https://staging.example.com"},
        {"name":"prod","value":"https://api.example.com"}]}'
```

Switch which one is live:

```bash
curl -X PUT http://localhost:8080/__admin/environments/baseUrl/active \
  -d '{"activeValue":"prod"}'
```

### Errors

| Code | HTTP |
|------|------|
| `Environment.InvalidBody` | 400 |
| `Environment.ReservedKey` | 400 |
| `Environment.UnknownKey` | 404 |

## Secret values

A sandbox is where a webhook signing secret or a partner API token ends up, and a plain environment
value shows up in the admin API, on the dashboard and inside export bundles — the artefacts that get
attached to tickets and committed to repositories.

Mark a value `secret` and its literal is withheld from all three, while stubs keep resolving it
normally:

```bash
curl -X PUT localhost:8080/__admin/environments/signingKey \
  -d '{"activeValue":"live","values":[
        {"name":"live","value":"whsec_live_9c1f","secret":true},
        {"name":"test","value":"whsec_test_0000"}
      ]}'
```

Reading it back gives you the marker and no literal — including `resolved`, which would otherwise hand
back the same value by another name:

```json
{ "key": "signingKey", "activeValue": "live", "resolved": null, "secret": true,
  "values": [ { "name": "live", "secret": true },
              { "name": "test", "value": "whsec_test_0000", "secret": false } ] }
```

A stub using `{{signingKey}}` serves `whsec_live_9c1f` exactly as before. Hiding a value you can no
longer use would not be a feature.

### Changing one

Because reads withhold the literal, a write that carries the marker and no value means **unchanged**:

```bash
# renames a value, leaves the secret alone
curl -X PUT localhost:8080/__admin/environments/signingKey \
  -d '{"activeValue":"live","values":[{"name":"live","secret":true},{"name":"staging","value":"..."}]}'
```

Send an explicit `value` to rotate it. A value that is new, marked secret and carries no literal is
**dropped** rather than stored empty — an empty secret is a stub that signs with nothing and reports
success, which is worse than an error.

The dashboard follows the same rule: the field for a secret is masked and reads *unchanged — type to
replace*, so saving a key you did not edit leaves the secret exactly as it was.

### In an export bundle

Bundles carry the marker and never the literal. A restore therefore reports the value as present but
unset rather than writing an empty string, and you re-enter it once on the target host.

:::caution
This keeps a secret out of the surfaces that report it. It is not encryption at rest — the value is
stored as text like every other environment value, and anyone with filesystem or database access to
the host can read it. Environments remain test-data-only: do not put production credentials in a
sandbox.
:::

## Export and import

Exporting stubs from the dashboard carries the tenant's environments with them. With no keys defined
the export is a plain mapping array, unchanged; with keys it becomes a `{"mappings":[…]}` bundle with
a sibling `environments` section:

```json
{
  "mappings": [ /* the stub definitions */ ],
  "environments": [
    {
      "key": "baseUrl",
      "activeValue": "staging",
      "values": [
        { "name": "staging", "value": "https://staging.example.com" },
        { "name": "prod", "value": "https://api.example.com" }
      ]
    }
  ]
}
```

Importing such a bundle — through the dashboard's import tab or directly via
[`POST /__admin/mappings/import`](/admin-api/) — restores the keys, their values, and the active
selection **before** the mappings load, so the stubs' `{{key}}` references resolve immediately.

- An imported key **replaces** an existing key of the same name: the import restores the exported
  state rather than merging into it.
- Each key passes the same validation as `PUT /__admin/environments/{key}`; an entry that fails
  (a reserved name, no usable values) is skipped without failing the import — the mappings still load.
- Older exports — a bare mapping array, or a bundle without an `environments` section — import
  exactly as before.

## In the dashboard

The **Environments** page lists the current tenant's keys, their named values, and which value is
active, and lets you switch the active value.

## Limitations

- **Values are plaintext.** There is no secret type — a value is readable through `GET
  /__admin/environments` and visible in the dashboard. Do not put credentials in a key and expect them
  to be hidden.
- **Multi-instance propagation is asynchronous.** With [`--change-feed`](/persistence/) a key change
  reaches the other instances within moments, not atomically — a request in flight on another instance
  may still resolve the previous value.

## Related

- [Multi-tenancy](/multi-tenancy/) — the scope environments live in.
- [Persistence](/persistence/) — where environment values are stored.
- [Templating](/templating/) — what runs after resolution.
