# SDK Snippet Review

A Layer-0 config check that runs **before** the [diagnostic ladder](diagnostic-ladder.md). When a user pastes SDK connection code (or just the address / namespace / auth fields), the snippet itself is frequently the whole bug — wrong endpoint family, short namespace, API key paired with the wrong address. Running DNS / TCP / TLS probes at that point "fixes" nothing, because nothing lower is broken.

This file is a **cross-SDK** checklist. SDK-specific field names (`tls.Config{}` in Go, `Connection.connect({ tls })` in TypeScript, `TLSConfig` in Python) belong to `skill-temporal-developer`. Stay at the level of: endpoint, namespace, auth method, TLS expectations, env vars.

Out of scope here (link, don't absorb):
- Network-layer probes (DNS / TCP / TLS / auth) → [diagnostic-ladder.md](diagnostic-ladder.md)
- Endpoint-family rationale and DNS failure shapes → [connectivity.md → Endpoint formats](connectivity.md#endpoint-formats)
- API-key lifecycle and required address form → [authentication.md → Required address form for API-key connections](authentication.md#required-address-form-for-api-key-connections)
- mTLS certificate validation → [certificates.md](certificates.md)

## Table of Contents

- [When to use this check](#when-to-use-this-check)
- [Review order](#review-order)
- [Endpoint form](#endpoint-form)
- [Namespace format](#namespace-format)
- [Auth method](#auth-method)
- [TLS expectations](#tls-expectations)
- [Environment variables](#environment-variables)
- [Common misconfigurations](#common-misconfigurations)
- [Quick routing](#quick-routing)

## When to use this check

Run snippet review when the user pastes SDK connection code **and** reports any of:

- Cannot connect to Cloud, `UNAVAILABLE` on first attempt <!-- grpc: UNAVAILABLE -->
- `UNAUTHENTICATED` / `PERMISSION_DENIED` on first attempt <!-- grpc: UNAUTHENTICATED --> <!-- grpc: PERMISSION_DENIED -->
- `context deadline exceeded` with no prior-success baseline <!-- grpc: DEADLINE_EXCEEDED -->
- `namespace not found` / `INVALID_ARGUMENT` <!-- grpc: INVALID_ARGUMENT -->
- "This worked yesterday" **after** a config change

Skip snippet review (go straight to the ladder) when the user has an established, previously-working config and the symptom is new — the snippet is not the culprit, something in the environment changed.

## Review order

Check in this order. Each step is cheaper than the one below it, and a failure at a higher step makes the lower ones moot.

1. **Auth method** — API key vs mTLS. This determines everything else.
2. **Endpoint form** — must match the auth method (see below).
3. **Namespace** — full `<namespace>.<account>` form.
4. **TLS expectations** — principle-level, per auth method.
5. **Env-var propagation** — are the `TEMPORAL_*` values actually reaching the client?

If any of steps 1–3 is wrong, fix that first. Do not start the diagnostic ladder until endpoint, namespace, and auth method are plausible — the ladder will produce misleading results.

## Endpoint form

Two Cloud endpoint families, each tied to how the client authenticates:

| Auth | Endpoint | Form | Source |
|---|---|---|---|
| mTLS | Namespace Endpoint (recommended) | `<namespace>.<account>.tmprl.cloud:7233` | <!-- docs/cloud/get-started/namespaces.mdx:325 --> |
| API key | Regional Endpoint | `<region>.<cloud_provider>.api.temporal.io:7233` | <!-- docs/cloud/get-started/api-keys.mdx:370 --> |
| Control plane (`tcld`, Cloud Ops API, Terraform) | `saas-api.tmprl.cloud` | port `443`, **not** 7233 | <!-- docs/cloud/operation-api.mdx:135 --> |

The Namespace Endpoint follows HA failovers transparently. <!-- docs/cloud/get-started/namespaces.mdx:328 --> When an mTLS client pins to a Regional Endpoint for explicit region routing, the client **must** override the TLS `server_name` to the Namespace Endpoint value — see [certificates.md → Server name override](certificates.md#server-name-override).

**Snippet smells:**

- Short hostname without `.<account>` or `.tmprl.cloud` — stale docs / wrong form.
- `saas-api.tmprl.cloud` as the data-plane address — that's the control plane on port 443, not a workflow endpoint. Pointing a worker or `temporal` CLI data-plane command at it will not connect. <!-- docs/cloud/connectivity/index.mdx:321 -->
- API key + Namespace Endpoint — the Cloud API-keys doc prescribes the Regional Endpoint. See [authentication.md → Required address form for API-key connections](authentication.md#required-address-form-for-api-key-connections).
- mTLS + Regional Endpoint **without** SNI override — the TLS handshake will fail with `x509: certificate is valid for <SANs>, not <requested host>`. See [certificates.md → Hostname mismatch](certificates.md#hostname-mismatch).
- Empty `HostPort` / address with a Cloud namespace set — no explicit Cloud endpoint; the client will attempt a local-dev default that won't reach Cloud.
- URL form (`https://…`) in `--address` / `TEMPORAL_ADDRESS` — the flag takes `host:port`, not a URL. <!-- docs/cli/cmd-options.mdx:137-139 --> <!-- docs/cli/index.mdx:269 -->

## Namespace format

Cloud namespace format is `<namespace_name>.<account_id>`. <!-- docs/cloud/get-started/api-keys.mdx:372-373 --> The account suffix is visible in the Cloud UI and in `tcld namespace list`.

**Snippet smells:**

- Short name (`payments`) without the account suffix — fails resolution, or produces `INVALID_ARGUMENT: namespace not found`.
- Namespace set in the address but not in the namespace field (or vice versa) — both values are required.
- Case mismatch — namespace strings are case-sensitive. Match the Cloud UI exactly.
- Namespace from the wrong account — looks correct but the user isn't logged into that account. Verify with `tcld account get` (see [authentication.md → Cloud role and permission model](authentication.md#cloud-role-and-permission-model)).

## Auth method

One auth method per client. A snippet that sets both mTLS cert flags **and** an API key is a configuration smell — the SDK will pick one and silently ignore the other, and which one wins is SDK-dependent and version-dependent.

**Snippet smells:**

- Both `tls-cert-path` / `tls-key-path` **and** `api-key` set — pick one.
- API key in the snippet but the namespace was created with `--auth-method mtls` (or vice versa). Either the namespace must be recreated / migrated, or the namespace is in the pre-release dual-auth mode. <!-- docs/cloud/get-started/namespaces.mdx:316-320 -->
- Auth method mismatched against the endpoint form (see [Endpoint form](#endpoint-form) above).

## TLS expectations

Principle-level only. Do not diagnose SDK-specific struct fields here — that belongs to `skill-temporal-developer`.

- **API key auth.** TLS is **required**. The client opens a TLS connection (server TLS only, no client cert) and presents the API key as a bearer credential at the gRPC layer. An API-key snippet with TLS explicitly disabled will not connect.
- **mTLS auth.** Client certificate and private key are **required**. The certificate must chain to a CA that the namespace accepts — verify with `tcld namespace accepted-client-ca list --namespace <ns>` (see [certificates.md → Accepted client CA set (mTLS Cloud)](certificates.md#accepted-client-ca-set-mtls-cloud)). The key file must match the cert; see [certificates.md → Key does not match cert](certificates.md#key-does-not-match-cert).

**Snippet smells:**

- API key connection with TLS disabled or with an `insecure` flag set — the server will close the connection.
- mTLS snippet without a key file, or key and cert from different generations.
- `--tls-disable-host-verification` / `TEMPORAL_TLS_DISABLE_HOST_VERIFICATION=true` <!-- docs/cli/index.mdx:275 --> in a snippet the user expects to work in production — this masks hostname-mismatch bugs rather than fixes them.

## Environment variables

Names verified against `docs/cli/index.mdx:269-277`. No `_CLIENT_` segment; no `_PATH` suffix on TLS paths.

| Variable | Purpose | CLI flag |
|---|---|---|
| `TEMPORAL_ADDRESS` | `host:port` (not a URL) | `--address` |
| `TEMPORAL_NAMESPACE` | `<namespace>.<account>` | `--namespace` |
| `TEMPORAL_API_KEY` | API-key secret (also read by `tcld`, **not** `TEMPORAL_CLOUD_API_KEY`) | `--api-key` |
| `TEMPORAL_TLS_CA` | Server CA certificate path | `--tls-ca-path` |
| `TEMPORAL_TLS_CERT` | Client x509 certificate path | `--tls-cert-path` |
| `TEMPORAL_TLS_KEY` | Client private key path | `--tls-key-path` |
| `TEMPORAL_TLS_SERVER_NAME` | SNI override (Regional Endpoint + mTLS) | `--tls-server-name` |
| `TEMPORAL_TLS_DISABLE_HOST_VERIFICATION` | Default `false` | `--tls-disable-host-verification` |

**Snippet smells:**

- Snippet hardcodes a value that an env var also sets. Precedence is SDK-specific; the effective value may not be what the snippet shows. Ask the user to echo the env var from the exact shell / container the client runs in.
- Env vars set in the user's interactive shell but the worker runs in a different shell / container / pod where they are absent.
- Typo in the variable name — `TEMPORAL_CLOUD_API_KEY`, `TEMPORAL_TLS_CLIENT_CERT_PATH`, `TEMPORAL_TLS_CERT_FILE`. The CLI and SDK silently ignore unknown variables; the symptom is "my config doesn't take effect."
- Mixing env vars and explicit flags across auth methods — e.g. `TEMPORAL_API_KEY` set in the environment, but the snippet also passes `--tls-cert-path`. See [Auth method](#auth-method) above.

## Common misconfigurations

| Snippet shape | Likely root cause | First fix |
|---|---|---|
| API key + `*.tmprl.cloud:7233` address | API-keys doc prescribes Regional Endpoint <!-- docs/cloud/get-started/api-keys.mdx:370 --> | Switch to `<region>.<provider>.api.temporal.io:7233` |
| mTLS + `*.api.temporal.io:7233` without SNI override | Regional Endpoint + mTLS needs `server_name = <ns>.<acct>.tmprl.cloud` <!-- docs/cloud/get-started/namespaces.mdx:332 --> | Add SNI override, or switch to Namespace Endpoint |
| Short namespace (`payments`) | Missing account suffix | Use `<ns>.<account>` from `tcld namespace list` |
| Empty HostPort / address, Cloud namespace set | No explicit Cloud endpoint | Add an address matching the auth method |
| API key **and** mTLS cert both configured | Ambiguous; one silently wins | Remove whichever you are not using |
| `saas-api.tmprl.cloud` as address | Control plane mistaken for data plane | Switch to the Namespace or Regional Endpoint |
| `https://…` in `--address` | Flag takes `host:port`, not a URL <!-- docs/cli/cmd-options.mdx:137-139 --> | Strip the scheme |
| `TEMPORAL_CLOUD_API_KEY` set, nothing else | Variable not read; `tcld` / SDK read `TEMPORAL_API_KEY` | Rename to `TEMPORAL_API_KEY` |
| `TEMPORAL_TLS_CLIENT_CERT_PATH` / `_PATH` suffix | Variable does not exist; correct name is `TEMPORAL_TLS_CERT` | Rename |

## Quick routing

- Snippet looks plausible on all five review points → proceed to [diagnostic-ladder.md](diagnostic-ladder.md).
- Snippet has an obvious wrong endpoint, namespace, or auth method → fix that first; do not run network-layer probes until it's corrected.
- Snippet fix applied and symptom persists → descend the ladder from layer 1 (DNS). The environment may have a second, independent problem.
- Snippet references SDK-specific struct fields, connection-builder objects, or runtime-specific TLS APIs that aren't covered here → hand off to `skill-temporal-developer`.
