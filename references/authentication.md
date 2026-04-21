# Authentication

Diagnose failures that happen *after* TLS has completed — the TCP connection is up, the handshake finished, and the peer returned a gRPC error about who you are or what you can do. This file covers layer 4 of the [diagnostic ladder](diagnostic-ladder.md).

Prerequisite: rule out layers 1–3 first. If `openssl s_client` (see [certificates.md → openssl recipes](certificates.md#openssl-recipes)) does not print `Verify return code: 0 (ok)`, the problem is not in this file.

Out of scope here (link, don't absorb):
- DNS / TCP / endpoint family → [connectivity.md](connectivity.md) (layers 1–2)
- TLS handshake / x509 / SNI override → [certificates.md](certificates.md) (layer 3)
- gRPC `RESOURCE_EXHAUSTED` → [rate-limits.md](rate-limits.md)
- `context deadline exceeded` (ambiguous) → [runtime-errors.md](runtime-errors.md)

## Table of Contents

- [Authentication vs authorization](#authentication-vs-authorization)
- [UNAUTHENTICATED vs PERMISSION_DENIED](#unauthenticated-vs-permission_denied)
- [API-key authentication](#api-key-authentication)
- [mTLS authentication after TLS completes](#mtls-authentication-after-tls-completes)
- [Cloud role and permission model](#cloud-role-and-permission-model)
- [Quick routing](#quick-routing)

## Authentication vs authorization

Temporal Cloud distinguishes two post-TLS checks, and the gRPC status code tells you which one failed:

- **Authentication** — *who you are*. The API key is valid and active, or the mTLS client cert chains to an accepted CA and matches any configured certificate filters. Failure → `UNAUTHENTICATED` <!-- grpc: UNAUTHENTICATED -->.
- **Authorization** — *what you can do*. The identity is known, but the role / Namespace permission / cert-filter-derived identity does not permit the requested action. Failure → `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED -->.

The authorization pathway for API keys is documented as "API key (authentication) → Identity (user or Service Account) → RBAC (authorization)" <!-- docs/cloud/get-started/api-keys.mdx:47 -->.

## UNAUTHENTICATED vs PERMISSION_DENIED

The two gRPC codes in this file, per the gRPC spec <!-- grpc: https://grpc.io/docs/guides/status-codes/ -->:

| Code | Meaning in the spec | What it tells you on Temporal Cloud |
|---|---|---|
| `UNAUTHENTICATED` <!-- grpc: UNAUTHENTICATED --> | The request does not have valid authentication credentials for the operation. | The API key was not sent, not accepted, disabled, or expired; or the mTLS cert was not trusted / not accepted. |
| `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> | The caller does not have permission to execute the specified operation. Distinct from `UNAUTHENTICATED`. | The identity is known (API key valid, cert accepted) but lacks the Namespace-level permission (`Read` / `Write` / `Admin`) or the account-level role required for the action. |

Two traps this distinction prevents:

- **`UNAVAILABLE` is not an auth code.** If your error is gRPC `UNAVAILABLE` <!-- grpc: UNAVAILABLE -->, TLS may have failed before any auth happened. Peel the wrapped cause; if it starts with `tls:`, `x509:`, or `remote error: tls:`, jump to [certificates.md](certificates.md). The troubleshooting guide names an expired TLS certificate as a common root cause of "looks like auth but isn't." <!-- docs/troubleshooting/last-connection-error.mdx:15 -->
- **`RESOURCE_EXHAUSTED` is not `PERMISSION_DENIED`.** A rate-limited caller is *allowed* to make the call but is being throttled. See [rate-limits.md](rate-limits.md).

Do not use `FORBIDDEN` or `FAILED_PRECONDITION` as auth codes. They are either not gRPC codes at all, or unrelated to auth.

## API-key authentication

### How the Temporal CLI and SDKs pick up the key

The Temporal CLI reads the API key either from the `--api-key` flag <!-- docs/cli/cmd-options.mdx:141 --> or from the `TEMPORAL_API_KEY` environment variable <!-- docs/cli/index.mdx:278 -->. The Cloud docs: "The CLI automatically picks up the `TEMPORAL_API_KEY` environment variable from your shell." <!-- docs/cloud/get-started/api-keys.mdx:365 -->

`tcld` uses the same two forms — `--api-key` flag or `TEMPORAL_API_KEY` env var. <!-- docs/cloud/get-started/api-keys.mdx:407-410 -->

The Terraform provider uses a separate env var (`TEMPORAL_CLOUD_API_KEY`) — do not confuse it with `TEMPORAL_API_KEY`. This file only covers `TEMPORAL_API_KEY`.

### Required address form for API-key connections

API-key connections must use the **Regional Endpoint** form, not the Namespace Endpoint. From the Cloud API-keys guide: "For API key connections, use the format `<region>.<cloud_provider>.api.temporal.io:7233`." <!-- docs/cloud/get-started/api-keys.mdx:370 --> The Namespace Endpoint (`<namespace>.<account>.tmprl.cloud`) is the correct form for mTLS. See the endpoint table in [connectivity.md → endpoint formats](connectivity.md#endpoint-formats) for the full comparison.

If a user reports `UNAUTHENTICATED` with an API key while also pointing `--address` at `<namespace>.<account>.tmprl.cloud:7233`, the endpoint is the first thing to check. <!-- VERIFY: exact server-side status text when a Namespace Endpoint is used with an API key varies by release; observe the returned code rather than pinning a string. -->

### Things to check when `UNAUTHENTICATED` is returned with an API key

Per the Cloud troubleshooting note: "Invalid API key errors: Check that you copied the key correctly and that it hasn't been revoked or expired." <!-- docs/cloud/get-started/api-keys.mdx:431 -->

- **Key not delivered to the process.** Inside the failing environment (pod/container/host), confirm `TEMPORAL_API_KEY` is set: `env | grep -i TEMPORAL_API_KEY`. A shell-level export on the developer's laptop is not inherited by a container.
- **Key typo or truncation.** Leading/trailing whitespace, a trailing newline from a copy-paste, or a shell that split the key on whitespace will all produce `UNAUTHENTICATED`.
- **Key disabled.** A disabled key cannot authenticate — per the Cloud docs: "When disabled, an API key cannot authenticate with Temporal Cloud." <!-- docs/cloud/get-started/api-keys.mdx:167 --> Check with `tcld apikey list` <!-- docs/cloud/tcld/apikey.mdx:130 --> or `tcld apikey get --id <apikey_id>` <!-- docs/cloud/tcld/apikey.mdx:108 -->.
- **Key deleted.** Per the Cloud docs: "Deleting an API key stops it from authenticating with Temporal Cloud." <!-- docs/cloud/get-started/api-keys.mdx:190 -->
- **Key expired.** API keys expire based on the `--duration` or `--expiry` set at creation time <!-- docs/cloud/tcld/apikey.mdx:62-89 -->. The FAQ caps expiry at 2 years. <!-- docs/cloud/get-started/api-keys.mdx:447-449 -->
- **Wrong address family.** See above — use the Regional Endpoint.
- **API keys disabled at the account level.** A Global Administrator or Account Owner can disable the *creation* of new API keys with the **Disable Create API Keys** control; existing keys continue to work until disabled, deleted, or expired. <!-- docs/cloud/get-started/api-keys.mdx:108-110 --> This does not on its own turn a working key into `UNAUTHENTICATED`.

### API-key lifecycle commands

Every command here is grounded in `docs/cloud/tcld/apikey.mdx`. The `tcld apikey` group alias is `ak`. <!-- docs/cloud/tcld/apikey.mdx:19 -->

```bash
# Create
tcld apikey create --name <name>
# Command and required flag:             <!-- docs/cloud/tcld/apikey.mdx:32 -->
# Optional: --description, --duration, --expiry, --request-id
#   --description: <!-- docs/cloud/tcld/apikey.mdx:50 -->
#   --duration:    <!-- docs/cloud/tcld/apikey.mdx:62 -->
#   --expiry:      <!-- docs/cloud/tcld/apikey.mdx:77 -->
#   --request-id:  <!-- docs/cloud/tcld/apikey.mdx:91 -->

# List (to find the ID for disable/enable/delete)
tcld apikey list
# Command: <!-- docs/cloud/tcld/apikey.mdx:130 -->

# Inspect a specific key
tcld apikey get --id <apikey_id>
# Command: <!-- docs/cloud/tcld/apikey.mdx:108 -->

# Disable / enable
tcld apikey disable --id <apikey_id>
tcld apikey enable  --id <apikey_id>
# disable: <!-- docs/cloud/tcld/apikey.mdx:194 -->
# enable:  <!-- docs/cloud/tcld/apikey.mdx:240 -->

# Delete
tcld apikey delete --id <apikey_id>
# Command: <!-- docs/cloud/tcld/apikey.mdx:146 -->
```

### Rotating without downtime

The Cloud docs prescribe this sequence: create a new key; verify both keys work; switch clients to the new key; delete the old key. <!-- docs/cloud/get-started/api-keys.mdx:219-224 -->

Key behavioral note: "Deleting or disabling a key removes its ability to authenticate into Temporal Cloud. If you delete or disable an API key being used by Workers to run a Workflow, those Workers will be unable to connect to Temporal until a new API key secret is created and configured." <!-- docs/cloud/get-started/api-keys.mdx:118-121 --> That is why the rotation order is "deploy new, verify, then delete old" and not the reverse.

### Discriminating with a CLI smoke test

```bash
temporal workflow list --limit 1 \
  --address <region>.<cloud_provider>.api.temporal.io:7233 \
  --namespace <namespace>.<account> \
  --api-key "$TEMPORAL_API_KEY"
# --address:  <!-- docs/cli/cmd-options.mdx:137 -->
# --namespace: <!-- docs/cli/cmd-options.mdx:420 -->
# --api-key:  <!-- docs/cli/cmd-options.mdx:141 -->
# Address form for API-key: <!-- docs/cloud/get-started/api-keys.mdx:370 -->
```

Interpret the result:

| CLI result | Where the chain broke | Where to go |
|---|---|---|
| Returns a list (possibly empty) | Auth and authorization both succeeded | Issue is elsewhere — look at the SDK / worker config |
| `UNAUTHENTICATED` <!-- grpc: UNAUTHENTICATED --> | Key itself is rejected (disabled, deleted, expired, typo, wrong env var, wrong endpoint family) | Re-run the [things-to-check list above](#things-to-check-when-unauthenticated-is-returned-with-an-api-key) |
| `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> | Key is authenticated but the identity lacks Namespace permission | [Cloud role and permission model](#cloud-role-and-permission-model) |
| An `x509:` or `tls:` error | Not an auth issue | [certificates.md](certificates.md) |

A lighter probe is `temporal operator cluster health --address <addr>` — if it returns `SERVING`, the client cleared every layer up through gRPC auth. <!-- docs/cli/operator.mdx:56 -->

## mTLS authentication after TLS completes

If the TLS handshake succeeded (the peer did not send a `tls:` alert and the client did not emit an `x509:` error — see [certificates.md](certificates.md) for those), there are still two post-handshake reasons an mTLS connection can be refused:

1. **The client cert chains to an accepted CA but is filtered out by a Namespace certificate filter.** See [Certificate filters](#certificate-filters) below.
2. **The identity derived from the cert is not authorized for the requested action.** See [Cloud role and permission model](#cloud-role-and-permission-model).

Case (1) typically surfaces as `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go -->, which is a TLS-layer alert but triggered by a post-validation filter check. Because the symptom is a TLS alert, `openssl s_client` output is the right first tool; the *cause* is at the Namespace-filter layer. See [certificates.md → Accepted client CA set](certificates.md#accepted-client-ca-set-mtls-cloud) for `remote error: tls: unknown certificate authority` and `remote error: tls: bad certificate` disambiguation.

Case (2) surfaces as gRPC `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> *after* a clean TLS handshake.

### Certificate filters

Cloud Namespace certificate filters are configured at the Namespace level and restrict which end-entity (leaf) certificates may authenticate, even when the issuing CA is in the accepted-client-ca set. Per the Cloud docs: "To limit access to specific [end-entity certificates](#end-entity-certificates), create certificate filters. Each filter contains values for one or more of the following fields: commonName (CN), organization (O), organizationalUnit (OU), subjectAlternativeName (SAN)." <!-- docs/cloud/get-started/certificates.mdx:404-410 --> "Corresponding fields in the client certificate must match every specified value in the filter." <!-- docs/cloud/get-started/certificates.mdx:412 -->

Matching rules worth knowing when diagnosing a filter mismatch:

- Values are case-insensitive. <!-- docs/cloud/get-started/certificates.mdx:414 -->
- Without wildcards, each value must match exactly. <!-- docs/cloud/get-started/certificates.mdx:414-415 -->
- A single `*` wildcard may appear at the beginning or end of a value (but not both, and not alone). <!-- docs/cloud/get-started/certificates.mdx:417-418 -->
- Maximum 25 filters per Namespace. <!-- docs/cloud/get-started/certificates.mdx:420 -->

Inspect the DN fields on the client cert:

```bash
openssl x509 -in client.pem -noout -subject
# -subject prints the cert's Subject (CN, O, OU, etc.) — see certificates.md openssl recipes
```

Inspect / change filters with `tcld`:

```bash
# View current filters
tcld namespace certificate-filters export \
  --namespace <namespace_id> \
  --certificate-filter-file <path>
# Command: <!-- docs/cloud/tcld/namespace.mdx:1271-1276 -->

# Clear all filters (allows any cert that chains to an accepted CA)
tcld namespace certificate-filters clear \
  --namespace <namespace_id>
# Command: <!-- docs/cloud/tcld/namespace.mdx:1216 -->

# Replace filters with a JSON file
tcld namespace certificate-filters import \
  --namespace <namespace_id> \
  --certificate-filter-file <path>
# Command: <!-- docs/cloud/tcld/namespace.mdx:1339-1344 -->

# Add additional filters
tcld namespace certificate-filters add \
  --namespace <namespace_id> \
  --certificate-filter-file <path>
# Command: <!-- docs/cloud/tcld/namespace.mdx:1133-1153 -->
```

Cloud UI path is documented alongside these commands. <!-- docs/cloud/get-started/certificates.mdx:458-468 -->

Caution on clearing: "Using this command allows _any_ client certificate that chains up to a configured CA certificate to connect to the Namespace." <!-- docs/cloud/tcld/namespace.mdx:1219-1224 -->

### Cert identity not mapped to a role

mTLS identifies the caller but does not by itself grant Namespace-level permissions. Authorization still flows through the Cloud role / Namespace-permission model in the next section. Symptom: a TLS handshake that succeeds (`Verify return code: 0 (ok)` from `openssl s_client`) followed by `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> on the first data-plane call.

## Cloud role and permission model

Cloud access is governed on two independent axes — account-level roles (who can do account operations) and Namespace-level permissions (who can do data-plane operations in a given Namespace). Reference: `docs/cloud/manage-access/roles-and-permissions.mdx`.

### Account-level roles

The CLI `--account-role` flag on `tcld user invite` and `tcld user set-account-role` accepts three values:

> Available account roles: `admin` | `developer` | `read`. <!-- docs/cloud/tcld/user.mdx:125 --><!-- docs/cloud/tcld/user.mdx:244 -->

The concept documentation lists additional role *names* — Account Owner, Global Admin, Developer, Finance Admin, Read-Only — alongside their primary purposes. <!-- docs/cloud/manage-access/roles-and-permissions.mdx:34-40 --> Treat the CLI flag enum (`admin | developer | read`) as the surface you interact with in tcld; the role concept docs describe what those roles *do*.

### Namespace-level permissions

Namespace permissions are set via `--namespace-permission <ns>=<permission-type>` on `tcld user invite` and `tcld user set-namespace-permissions`. The enum is:

> Available namespace permissions: `Admin` | `Write` | `Read`. <!-- docs/cloud/tcld/user.mdx:136 --><!-- docs/cloud/tcld/user.mdx:338 -->

The values are case-sensitive (the docs write them capitalized; the invite example uses `ns1=Admin --namespace-permission ns2=Write`). <!-- docs/cloud/tcld/user.mdx:149 -->

What each permission grants is summarized in the concept table: Read observes activity; Write starts / signals / cancels / terminates / resets Workflows and polls Task Queues; Namespace Admin does all of that plus Namespace administration. <!-- docs/cloud/manage-access/roles-and-permissions.mdx:83-87 -->

### Inviting and adjusting users with tcld

```bash
# Invite a user with account role + per-namespace permissions
tcld user invite \
  --user-email <user@example.com> \
  --account-role developer \
  --namespace-permission <ns1>=Admin \
  --namespace-permission <ns2>=Write
# Command:                                        <!-- docs/cloud/tcld/user.mdx:102 -->
# --user-email (required, NOT --email):           <!-- docs/cloud/tcld/user.mdx:110-117 -->
# --account-role (required), enum admin|developer|read: <!-- docs/cloud/tcld/user.mdx:119-125 -->
# --namespace-permission (repeatable), format <ns>=<Admin|Write|Read>: <!-- docs/cloud/tcld/user.mdx:129-138 -->

# Change a user's account role after the fact
tcld user set-account-role \
  --user-email <user@example.com> \
  --account-role developer
# Command: <!-- docs/cloud/tcld/user.mdx:229 -->

# Change a user's Namespace permissions
tcld user set-namespace-permissions \
  --user-email <user@example.com> \
  --namespace-permission <ns>=Write
# Command: <!-- docs/cloud/tcld/user.mdx:287 -->

# Look up what a user has
tcld user get --user-email <user@example.com>
# Command: <!-- docs/cloud/tcld/user.mdx:73 -->

# List users scoped to a Namespace
tcld user list --namespace <namespace_id>
# Command: <!-- docs/cloud/tcld/user.mdx:152 -->
```

User groups follow a parallel shape but a different namespace-role syntax — `<namespaceid>-<role>` with `<role>` in `admin | read | write`, instead of the `<ns>=<Admin|Write|Read>` used on `tcld user`. <!-- docs/cloud/tcld/user-group.mdx:66-67 --><!-- docs/cloud/tcld/user-group.mdx:151-154 --> The user-group account-role enum is also broader than the `tcld user` one. <!-- docs/cloud/tcld/user-group.mdx:62 --> If a triage turns up `PERMISSION_DENIED` for a principal that inherits its access from a group, reach for `tcld user-group get --group-id <id>` <!-- docs/cloud/tcld/user-group.mdx:81 --> rather than `tcld user get`.

### Discriminating with a read-vs-write smoke test

If a user can do one operation but not another in the same session, the question is almost always which Namespace permission they hold:

- `temporal workflow list --limit 1 ...` — requires `Read` or higher. Success implies the identity is authenticated and has at least `Read` on the Namespace.
- `temporal workflow start ...` or `temporal workflow signal ...` — requires `Write` or higher.
- Namespace administration (e.g. `tcld namespace get --namespace <ns>`) requires the appropriate account role and/or Namespace `Admin`, and hits the control plane, not the data plane. A control-plane-only failure with data-plane access intact points at the account role, not the Namespace permission.

If the read smoke test returns data and the write smoke test returns `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED -->, the diagnosis is "identity authenticated, permission insufficient" rather than "broken auth."

## Quick routing

| Error text shape | Layer | Go to |
|---|---|---|
| gRPC `UNAUTHENTICATED` <!-- grpc: UNAUTHENTICATED --> with API key | 4 | [API-key authentication](#api-key-authentication) |
| gRPC `UNAUTHENTICATED` <!-- grpc: UNAUTHENTICATED --> with mTLS | 3–4 boundary | First confirm TLS with `openssl s_client` per [certificates.md](certificates.md); if TLS is clean, this is a post-TLS rejection — see [mTLS authentication after TLS completes](#mtls-authentication-after-tls-completes) |
| gRPC `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> on data-plane call | 4 | [Cloud role and permission model](#cloud-role-and-permission-model) |
| gRPC `PERMISSION_DENIED` <!-- grpc: PERMISSION_DENIED --> on `tcld` control-plane call | 4 (control plane) | Account-level role — see [Account-level roles](#account-level-roles) |
| `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go --> | 3 / filter | [certificates.md → Accepted client CA set](certificates.md#accepted-client-ca-set-mtls-cloud) and then [Certificate filters](#certificate-filters) here |
| `remote error: tls: unknown certificate authority` <!-- go: crypto/tls/alert.go --> | 3 | [certificates.md → Accepted client CA set](certificates.md#accepted-client-ca-set-mtls-cloud) |
| gRPC `UNAVAILABLE` <!-- grpc: UNAVAILABLE --> | 2–3 | Not an auth code — peel the wrapped cause; see [connectivity.md](connectivity.md) and [certificates.md](certificates.md) |
| gRPC `RESOURCE_EXHAUSTED` <!-- grpc: RESOURCE_EXHAUSTED --> | 4+ | Not an auth code — see [rate-limits.md](rate-limits.md) |
