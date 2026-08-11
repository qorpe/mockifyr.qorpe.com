---
title: Securing the admin API
description: Require a username and password on the admin API and the dashboard with HTTP Basic auth.
---

By default Mockifyr is **open** — anyone who can reach it can manage stubs. Set an admin username and
password to require **HTTP Basic auth** on the admin API (`/__admin/*`) and show a login screen on the
dashboard.

:::note
The **mock-serving surface stays open** — clients hit your stubs without credentials, as they should.
Only the admin API and dashboard are protected.
:::

:::note
`/__admin/health` is **exempt** from admin Basic auth: Kubernetes/OpenShift probes cannot carry
credentials, and a 401 health check would send the pod into a restart loop. The endpoint is
read-only and exposes only name, version, persistence, and tenant count; every other `/__admin`
route stays guarded.
:::

Three flags cover the hardening this page's guidance implies — see the
[CLI reference](/cli/#security-hardening) for the full table:

- **`--tenant-credential <tenant>:<user>:<pass>`** — per-tenant admin credentials. Without it the
  tenant header is a claim any admin caller can rewrite; with it, a principal scoped to one tenant
  gets **403** when it names another. `--admin-user` remains the system scope.
- **`--mask-headers` / `--mask-body-fields`** — keep credentials and sensitive fields out of the
  request journal entirely.
- **`--partner-credential <tenant>:<user>:<pass>`** — the same scoping, for a credential you hand to
  someone outside your team. See [handing one out](#handing-a-credential-to-a-partner).
- **`--block-outbound-routes`** — on an unauthenticated host, refuse the routes that make outbound
  calls or change outbound trust. On an authenticated host it does nothing, and says so at startup.

An unauthenticated host also prints a startup line naming what is reachable.

## Enable it

Two settings, always given together — `admin-user` and `admin-pass`. If only one is set, auth stays
off.

Either form works, because **every Mockifyr flag is also readable as an environment variable of the
same name** — the host builds its configuration with the standard .NET builder, so `--admin-user` on
the command line and `admin-user` in the environment reach the same key. Command-line arguments win
when both are present. Prefer the environment for credentials; see the [CLI reference](/cli/).

### Docker — environment variables (recommended)

```bash
docker run -p 8080:8080 \
  -e admin-user=alice -e admin-pass='s3cret' \
  ghcr.io/qorpe/mockifyr
```

### Docker Compose

Keep the password out of the file with a git-ignored `.env`:

```yaml
# docker-compose.yml
services:
  mockifyr:
    image: ghcr.io/qorpe/mockifyr:latest
    ports: ['8080:8080']
    environment:
      admin-user: ${MOCKIFYR_USER}
      admin-pass: ${MOCKIFYR_PASS}
```

```ini
# .env  (add to .gitignore)
MOCKIFYR_USER=alice
MOCKIFYR_PASS=s3cret
```

### Command-line flags

```bash
docker run -p 8080:8080 ghcr.io/qorpe/mockifyr --admin-user alice --admin-pass 's3cret'
# local:
dotnet run --project src/Mockifyr.Server -- --admin-user alice --admin-pass 's3cret'
```

## Using it

**CLI clients** send Basic auth proactively:

```bash
curl -u alice:s3cret http://localhost:8080/__admin/mappings
```

Without credentials the admin API returns **401**; the mock surface is unaffected.

**The dashboard** shows a login screen — enter the same username and password. It stores the
credentials locally and attaches Basic auth to its admin calls. (Mockifyr deliberately omits the
`WWW-Authenticate` header so the browser's native Basic-auth popup never blocks the dashboard.)

## Handing a credential to a partner

`--tenant-credential` answers "which tenant's data may this credential touch". It does not answer
"may this credential make the host call out", and for a credential you keep yourself that question
does not come up — you already trust the holder.

It comes up the moment you hand one to somebody else. A few admin routes act on the network rather
than on data: starting a recording points this host at a target of the caller's choosing, outbound
trust changes which certificates it accepts, and Git sync reaches a remote. So does a stub: a
`proxyBaseUrl` forwards to whatever it names, and a post-serve action calls out after responding.

`--partner-credential` takes the same `tenant:user:pass` form and refuses all of it:

```bash
mockifyr --admin-user ops --admin-pass "$OPS_PASS" \
         --partner-credential "acme:acme-team:$ACME_PASS"
```

The holder can do everything with `acme`'s data — stubs, sandbox resources, environments, the
journal, captured messages — and gets **403** on `/__admin/recordings`, `/__admin/outbound-trust` and
`/__admin/git`, and on any stub declaring `proxyBaseUrl` or a post-serve action. The refusal names
the field, so if they genuinely need a proxy stub they can tell you what to add rather than guessing
why it failed.

Importing an OpenAPI document still works: nothing the importer generates reaches outward, so a
partner can turn their specification into a working sandbox unaided.

:::note
Refused *changes* appear in the [audit trail](/admin-api/#audit-trail) as `partner:<tenant>`, distinct
from an operator's `tenant:<tenant>` — so a tenant with both credentials still tells you which one
acted. Refused reads are not recorded, the same as any other read.
:::

## Best practices

- **Don't hard-code the password** in a committed Compose file or in shell history — use an
  environment variable from a git-ignored `.env`, or a Docker/host secret.
- A command-line password is visible in `ps`; prefer env vars in production.
- Put Mockifyr behind TLS (`--https-port`) or a TLS-terminating proxy so Basic credentials aren't sent
  in the clear.


## Single sign-on (OIDC)

Authenticate people through your identity provider instead of a shared password:

```bash
mockifyr --oidc-authority https://login.example.com \
         --oidc-audience mockifyr \
         --oidc-client-id mockifyr-dashboard
```

The dashboard then shows **Sign in with your identity provider** instead of a username and password,
using authorization code + PKCE. Register the dashboard's URL as a redirect URI on a **public** client
— there is no client secret, because anything shipped to a browser is readable.

API callers send the access token as `Authorization: Bearer <token>`.

### Scoping an identity to a tenant

```bash
--oidc-tenant-claim mockifyr_tenant
```

The named claim decides which tenant that identity may address — the same rule
[`--tenant-credential`](#per-tenant-credentials) enforces, applied to a claim instead of a password. A
principal scoped to `acme` gets **403** when it names `globex`, and omitting the tenant header does not
help: that addresses the default tenant, which it also does not own.

An identity **without** the claim keeps system scope and reaches every tenant — the OIDC equivalent of
`--admin-user`, so an operator's own account still works while individual teams are scoped.

### Requiring a role

```bash
--oidc-required-role mockifyr-admin        # --oidc-role-claim defaults to `roles`
```

Tokens without the role are refused, so a directory-wide sign-in is not automatically admin access
here.

### Notes

- **Basic credentials keep working alongside OIDC.** Run SSO for people and `--admin-user` for CI —
  adopting it does not have to be a flag day.
- **Signing keys come from the provider's discovery document**, so key rotation needs no restart and
  nothing is pinned in configuration.
- **Everything that is not a valid token is a 401**, including an unreachable provider — a provider
  outage must not turn every admin call into a 500.
- **The [audit trail](/admin-api/#audit-trail) records the person** (`oidc:jane@example.com`), never
  the token.
- **`/__admin/health` reports the auth mode** and the public client parameters, unauthenticated by
  necessity — a login screen cannot authenticate before it knows where to send the user.
- Probes stay open, as always: a kubelet cannot carry a token.

:::note
Not yet supported: token refresh (an expired session returns you to sign-in), back-channel logout, and
mapping claims to anything finer than a tenant.
:::
