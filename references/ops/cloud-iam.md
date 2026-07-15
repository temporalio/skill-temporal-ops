# Cloud IAM Reference

Quick-reference for Temporal Cloud identity and access management via `tcld`.
Covers API keys, users, user groups, service accounts, account operations, roles, and namespace permissions.

> For authentication failures during workflow execution, see the triage ladder in `../triage/authentication.md`.

---

## API Key Lifecycle (`tcld apikey`)

Alias: `ak` <!-- docs/cloud/tcld/apikey.mdx:19 -->

### Create

```bash
tcld apikey create --name <name> \
    --description "<description>" \
    --duration <duration>        # e.g. 24h; ignored if --expiry set; default 0s
    # --expiry <RFC3339>         # e.g. '2023-11-28T09:23:24-08:00'
    # --request-id <request_id>
```
<!-- docs/cloud/tcld/apikey.mdx:30-102 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--name` | `-n` | Yes | Display name of the API key <!-- docs/cloud/tcld/apikey.mdx:40 --> |
| `--description` | `-desc` | No | <!-- docs/cloud/tcld/apikey.mdx:52 --> |
| `--duration` | `-d` | No | Duration from now until expiry. Ignored if `--expiry` is set. Default `0s` <!-- docs/cloud/tcld/apikey.mdx:64-68 --> |
| `--expiry` | `-e` | No | Absolute expiry timestamp (RFC3339) <!-- docs/cloud/tcld/apikey.mdx:78 --> |
| `--request-id` | `-r` | No | Server assigns one if not set <!-- docs/cloud/tcld/apikey.mdx:92 --> |

To create an API key for a **Service Account**, add `--service-account-id <id>`:
<!-- docs/cloud/get-started/api-keys.mdx:258-267 -->

```bash
tcld apikey create \
    --name <name> \
    --description "<description>" \
    --duration <duration> \
    --service-account-id <service-account-id>
```

### Get

```bash
tcld apikey get --id <apikey_id>
```
<!-- docs/cloud/tcld/apikey.mdx:106-124 -->

| Flag | Alias | Required |
|------|-------|----------|
| `--id` | `-i` | Yes |

### List

```bash
tcld apikey list
```
<!-- docs/cloud/tcld/apikey.mdx:128-140 -->

Alias: `l` <!-- docs/cloud/tcld/apikey.mdx:135 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--owner-id` | `-oid` | No | Filter API keys by owner ID <!-- docs/cloud/tcld/apikey.mdx:87-89 --> |
| `--owner-type` | `-ot` | No | Filter by owner type: `user` \| `service-account` <!-- docs/cloud/tcld/apikey.mdx:93-95 --> |

### Delete

```bash
tcld apikey delete --id <apikey_id>
```
<!-- docs/cloud/tcld/apikey.mdx:144-188 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--id` | `-i` | Yes | <!-- docs/cloud/tcld/apikey.mdx:152 --> |
| `--resource-version` | `-v` | No | ETag; uses latest if not set <!-- docs/cloud/tcld/apikey.mdx:166 --> |
| `--request-id` | `-r` | No | Server assigns if not set <!-- docs/cloud/tcld/apikey.mdx:178 --> |

### Disable

```bash
tcld apikey disable --id <apikey_id>
```
<!-- docs/cloud/tcld/apikey.mdx:192-234 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--id` | `-i` | Yes | <!-- docs/cloud/tcld/apikey.mdx:200 --> |
| `--resource-version` | `-v` | No | ETag; uses latest if not set <!-- docs/cloud/tcld/apikey.mdx:214 --> |
| `--request-id` | `-r` | No | Server assigns if not set <!-- docs/cloud/tcld/apikey.mdx:226 --> |

### Enable

```bash
tcld apikey enable --id <apikey_id>
```
<!-- docs/cloud/tcld/apikey.mdx:238-282 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--id` | `-i` | Yes | <!-- docs/cloud/tcld/apikey.mdx:248 --> |
| `--resource-version` | `-v` | No | ETag; uses latest if not set <!-- docs/cloud/tcld/apikey.mdx:260 --> |
| `--request-id` | `-r` | No | Server assigns if not set <!-- docs/cloud/tcld/apikey.mdx:272 --> |

### Key Rotation Procedure

<!-- docs/cloud/get-started/api-keys.mdx:219-225 -->

1. Create a new key (you may reuse key names).
2. Verify both original and new key function properly.
3. Switch clients to load the new key.
4. Delete the old key after it is no longer in use.

### API Key Limits

<!-- docs/cloud/get-started/api-keys.mdx:440-449 -->

- Up to **10** non-expired keys per user.
- Up to **20** non-expired keys per Service Account.
- Maximum expiration time: **2 years**.

---

## API Key Connectivity Setup

To authenticate SDK or CLI connections to Temporal Cloud using an API key:

<!-- docs/cloud/get-started/api-keys.mdx:364-389 -->

### Environment variable approach (recommended)

```bash
export TEMPORAL_API_KEY=<key-secret>
temporal workflow list \
    --address <namespace>.<account>.tmprl.cloud:7233 \
    --namespace <namespace_id>.<account_id>
```

### tcld authentication

<!-- docs/cloud/get-started/api-keys.mdx:407-410 -->

Pass the key with `--api-key` flag or set the `TEMPORAL_API_KEY` environment variable:

```bash
tcld --api-key <key-secret> apikey list
# or
export TEMPORAL_API_KEY=<key-secret>
tcld apikey list
```

### Namespace gRPC endpoint format

<!-- docs/cloud/get-started/api-keys.mdx:347-349 -->

```
<namespace>.<account>.tmprl.cloud:7233
```

For Temporal CLI, the `--address` format for API key connections uses:
`<region>.<cloud_provider>.api.temporal.io:7233`
<!-- docs/cloud/get-started/api-keys.mdx:370-371 -->

---

## User Management (`tcld user`)

Alias: `u` <!-- docs/cloud/tcld/user.mdx:20 -->

### Invite

```bash
tcld user invite \
    --user-email <email> \
    --account-role <role> \
    --namespace-permission <namespace>=<permission>
```
<!-- docs/cloud/tcld/user.mdx:104-150 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--user-email` | `-e` | Yes | Can be supplied multiple times <!-- docs/cloud/tcld/user.mdx:113-116 --> |
| `--account-role` | `--ar` | Yes | Case-insensitive: `Admin` \| `Developer` \| `Read` \| `Owner` \| `FinanceAdmin` \| `MetricsRead` <!-- docs/cloud/tcld/user.mdx:84-86 --> |
| `--namespace-permission` | `-p` | No | Format: `namespace=permission-type`. Can be repeated. Permissions: `Admin` \| `Write` \| `Read` <!-- docs/cloud/tcld/user.mdx:129-137 --> |
| `--request-id` | `-r` | No | <!-- docs/cloud/tcld/user.mdx:141 --> |

Example with multiple namespace permissions:
```bash
tcld user invite \
    --user-email <test@example.com> \
    --account-role developer \
    --namespace-permission ns1=Admin \
    --namespace-permission ns2=Write \
    --request-id <123456>
```
<!-- docs/cloud/tcld/user.mdx:148-150 -->

### Get

```bash
tcld user get --user-email <email>
# or
tcld user get --user-id <user-id>
```
<!-- docs/cloud/tcld/user.mdx:74-100 -->

Must set either `--user-email` or `--user-id`. <!-- docs/cloud/tcld/user.mdx:77 -->

### List

```bash
tcld user list
```
<!-- docs/cloud/tcld/user.mdx:154-160 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--namespace` | `-n` | No | Filter: users with permissions to this namespace <!-- docs/cloud/tcld/user.mdx:168-170 --> |
| `--page-token` | `-p` | No | Pagination token <!-- docs/cloud/tcld/user.mdx:180 --> |
| `--page-size` | `-s` | No | Defaults to 10 <!-- docs/cloud/tcld/user.mdx:186-188 --> |

### Delete

```bash
tcld user delete --user-email <email>
# or
tcld user delete --user-id <user-id>
```
<!-- docs/cloud/tcld/user.mdx:30-71 -->

Must set either `--user-email` or `--user-id`. <!-- docs/cloud/tcld/user.mdx:33 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--user-email` | | Conditional | <!-- docs/cloud/tcld/user.mdx:39 --> |
| `--user-id` | | Conditional | <!-- docs/cloud/tcld/user.mdx:49 --> |
| `--request-id` | `-r` | No | <!-- docs/cloud/tcld/user.mdx:59 --> |
| `--resource-version` | `-v` | No | ETag; uses latest if not set <!-- docs/cloud/tcld/user.mdx:66-69 --> |

### Resend Invite

```bash
tcld user resend-invite --user-email <email>
# or
tcld user resend-invite --user-id <user-id>
```
<!-- docs/cloud/tcld/user.mdx:192-219 -->

Alias: `ri` <!-- docs/cloud/tcld/user.mdx:196 -->

Must set either `--user-email` or `--user-id`. <!-- docs/cloud/tcld/user.mdx:193-194 -->

### Set Account Role

```bash
tcld user set-account-role --user-email <email> --account-role <role>
# or
tcld user set-account-role --user-id <user-id> --account-role <role>
```
<!-- docs/cloud/tcld/user.mdx:229-285 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--account-role` | `-ar` | Yes | Case-insensitive: `Admin` \| `Developer` \| `Read` \| `Owner` \| `FinanceAdmin` \| `MetricsRead` <!-- docs/cloud/tcld/user.mdx:186-188 --> |
| `--user-email` | `-e` | Conditional | <!-- docs/cloud/tcld/user.mdx:250 --> |
| `--user-id` | `--id` | Conditional | <!-- docs/cloud/tcld/user.mdx:264 --> |
| `--request-id` | `-r` | No | <!-- docs/cloud/tcld/user.mdx:275 --> |
| `--resource-version` | `-v` | No | ETag <!-- docs/cloud/tcld/user.mdx:281 --> |

### Set Namespace Permissions

```bash
tcld user set-namespace-permissions \
    --user-email <email> \
    --namespace-permission <namespace>=<permission>
```
<!-- docs/cloud/tcld/user.mdx:287-340 -->

Alias: `snp` <!-- docs/cloud/tcld/user.mdx:293 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--user-email` | | Conditional | <!-- docs/cloud/tcld/user.mdx:297 --> |
| `--user-id` | | Conditional | <!-- docs/cloud/tcld/user.mdx:307 --> |
| `--namespace-permission` | `-p` | No | Format: `namespace=permission-type`. Can be repeated. Permissions: `Admin` \| `Write` \| `Read`. Empty removes all namespace permissions <!-- docs/cloud/tcld/user.mdx:222-224 --> |
| `--request-id` | `-r` | No | <!-- docs/cloud/tcld/user.mdx:319 --> |
| `--resource-version` | `-v` | No | ETag <!-- docs/cloud/tcld/user.mdx:325 --> |

---

## Roles and Permissions

### Account-Level Roles

<!-- docs/cloud/tcld/user.mdx:125 and docs/cloud/tcld/user-group.mdx:62 -->

| Role (tcld value) | Notes |
|--------------------|-------|
| `admin` | Global Administrator <!-- docs/cloud/tcld/user.mdx:86 --> |
| `developer` | <!-- docs/cloud/tcld/user.mdx:86 --> |
| `read` | Read-only <!-- docs/cloud/tcld/user.mdx:86 --> |
| `owner` | Account Owner <!-- docs/cloud/tcld/user.mdx:86, docs/cloud/tcld/user-group.mdx:62 --> |
| `financeadmin` | Finance Admin <!-- docs/cloud/tcld/user.mdx:86, docs/cloud/tcld/user-group.mdx:62 --> |
| `metricsread` | Metrics read access <!-- docs/cloud/tcld/user.mdx:86 --> |
| `none` | User-group only; removes account-level role <!-- docs/cloud/tcld/user-group.mdx:62 --> |

Account-role values are case-insensitive in `tcld user` commands; canonical forms are
`Admin`, `Developer`, `Read`, `Owner`, `FinanceAdmin`, `MetricsRead`. <!-- docs/cloud/tcld/user.mdx:86,188 -->
`tcld user invite` and `tcld user set-account-role` accept all six of these values; the server may
still reject a role that is not valid for a given identity. `none` is accepted only by
`tcld user-group` commands. <!-- docs/cloud/tcld/user.mdx:86,188, docs/cloud/tcld/user-group.mdx:62 -->

### Namespace-Level Permissions

<!-- docs/cloud/tcld/user.mdx:137 -->

| Permission (tcld value) | Notes |
|--------------------------|-------|
| `Admin` | Full namespace control <!-- docs/cloud/tcld/user.mdx:137 --> |
| `Write` | <!-- docs/cloud/tcld/user.mdx:137 --> |
| `Read` | Read-only <!-- docs/cloud/tcld/user.mdx:137 --> |

Format for `--namespace-permission` flag: `<namespace_id>=<permission>`
where `<namespace_id>` is the full Cloud namespace ID (e.g. `mynamespace.abc123`).
<!-- docs/cloud/tcld/user.mdx:135 -->

---

## User Groups (`tcld user-group`)

Alias: `ug` <!-- docs/cloud/tcld/user-group.mdx:18 -->

### Create

```bash
tcld user-group create \
    --display-name <name> \
    --account-role <role> \
    --namespace-role <namespaceid>-<role>
```
<!-- docs/cloud/tcld/user-group.mdx:49-67 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--display-name` | | Yes | Display name of the group <!-- docs/cloud/tcld/user-group.mdx:58 --> |
| `--account-role` | | Yes | `admin` \| `read` \| `developer` \| `owner` \| `financeadmin` \| `none` <!-- docs/cloud/tcld/user-group.mdx:62 --> |
| `--namespace-role` | `-nr` | No | Repeatable. Format: `<namespaceid>-<role>` where role is `admin` \| `read` \| `write`. Example: `mynamespace.abc123-read` <!-- docs/cloud/tcld/user-group.mdx:66-67 --> |

Alias: `c` <!-- docs/cloud/tcld/user-group.mdx:52 -->

**Important**: the `--namespace-role` format uses a **hyphen** separator (`<namespaceid>-<role>`), not `=`.
This differs from `tcld user` commands which use `<namespace>=<permission>`. <!-- docs/cloud/tcld/user-group.mdx:67 -->

### Get

```bash
tcld user-group get --group-id <id>
```
<!-- docs/cloud/tcld/user-group.mdx:84-90 -->

Alias: `g` <!-- docs/cloud/tcld/user-group.mdx:86 -->

### List

```bash
tcld user-group list
```
<!-- docs/cloud/tcld/user-group.mdx:94-105 -->

| Flag | Alias | Notes |
|------|-------|-------|
| `--page-size` | `-s` | Defaults to 10 <!-- docs/cloud/tcld/user-group.mdx:101 --> |
| `--page-token` | `-p` | <!-- docs/cloud/tcld/user-group.mdx:105 --> |

### Delete

```bash
tcld user-group delete --group-id <id>
```
<!-- docs/cloud/tcld/user-group.mdx:69-79 -->

Alias: `d` <!-- docs/cloud/tcld/user-group.mdx:74 -->

### Add Users

```bash
tcld user-group add-users --group-id <id> --user-email <email>
```
<!-- docs/cloud/tcld/user-group.mdx:29-46 -->

Alias: `au` <!-- docs/cloud/tcld/user-group.mdx:34 -->

| Flag | Alias | Notes |
|------|-------|-------|
| `--group-id` | `-id` | Required <!-- docs/cloud/tcld/user-group.mdx:40 --> |
| `--user-email` | `-e` | Can be specified multiple times <!-- docs/cloud/tcld/user-group.mdx:44-45 --> |

### Remove Users

```bash
tcld user-group remove-users --group-id <id> --user-email <email>
```
<!-- docs/cloud/tcld/user-group.mdx:122-136 -->

Alias: `ru` <!-- docs/cloud/tcld/user-group.mdx:126 -->

| Flag | Alias | Notes |
|------|-------|-------|
| `--group-id` | `-id` | Required <!-- docs/cloud/tcld/user-group.mdx:130 --> |
| `--user-email` | `-e` | Can be specified multiple times <!-- docs/cloud/tcld/user-group.mdx:134-135 --> |

### List Members

```bash
tcld user-group list-members --group-id <id>
```
<!-- docs/cloud/tcld/user-group.mdx:108-120 -->

Alias: `lm` <!-- docs/cloud/tcld/user-group.mdx:112 -->

### Set Access

```bash
tcld user-group set-access --group-id <id> \
    --account-role <role> \
    --namespace-role <namespaceid>-<role>
```
<!-- docs/cloud/tcld/user-group.mdx:138-163 -->

Alias: `sa` <!-- docs/cloud/tcld/user-group.mdx:141 -->

| Flag | Alias | Required | Notes |
|------|-------|----------|-------|
| `--group-id` | `-id` | Yes | <!-- docs/cloud/tcld/user-group.mdx:145 --> |
| `--account-role` | | No | `admin` \| `read` \| `developer` \| `owner` \| `financeadmin` \| `none` <!-- docs/cloud/tcld/user-group.mdx:149 --> |
| `--namespace-role` | `-nr` | No | Repeatable. Same `<namespaceid>-<role>` format as create <!-- docs/cloud/tcld/user-group.mdx:153-154 --> |
| `--append` | `-a` | No | Append namespace roles instead of replacing all existing roles <!-- docs/cloud/tcld/user-group.mdx:157-158 --> |
| `--remove` | `-r` | No | Remove the given namespace roles instead of replacing <!-- docs/cloud/tcld/user-group.mdx:161-162 --> |

Without `--append` or `--remove`, set-access **replaces** all existing roles. <!-- docs/cloud/tcld/user-group.mdx:157-158, 161-162 -->

---

## Service Accounts (`tcld service-account`)

<!-- docs/cloud/get-started/service-accounts.mdx:48 -->

Service Accounts are non-human identities that use API keys to authenticate.
Use `tcld service-account --help` for a full list of subcommands. <!-- docs/cloud/get-started/service-accounts.mdx:48 -->

### Create

```bash
tcld service-account create -n "<name>" -d "<description>" --ar "<account-role>"
# Optional: --np "<namespace>=<permission>"
```
<!-- docs/cloud/get-started/service-accounts.mdx:88-89 -->

Returns a `ServiceAccountId` used for subsequent operations. <!-- docs/cloud/get-started/service-accounts.mdx:96-97 -->

### Create Scoped (Namespace-scoped)

```bash
tcld service-account create-scoped -n "<name>" --np "<namespace>=<permission>"
```
<!-- docs/cloud/get-started/service-accounts.mdx:230-231 -->

Namespace-scoped Service Accounts always have a `Read` Account Role and are restricted to a single namespace. <!-- docs/cloud/get-started/service-accounts.mdx:195-198 -->
Cannot be reassigned to a different namespace after creation. <!-- docs/cloud/get-started/service-accounts.mdx:200 -->

### List

```bash
tcld service-account list
```
<!-- docs/cloud/get-started/service-accounts.mdx:118-119 -->

### Get

```bash
tcld service-account get --service-account-id "<id>"
```
<!-- docs/cloud/tcld/service-account.mdx:115-127 -->

Alias: `g`. `--service-account-id` (alias `--id`) is required. <!-- docs/cloud/tcld/service-account.mdx:121-127 -->

### Delete

```bash
tcld service-account delete --service-account-id "<id>"
```
<!-- docs/cloud/get-started/service-accounts.mdx:147-148 -->

Deleting a Service Account automatically deletes all associated API keys. <!-- docs/cloud/get-started/service-accounts.mdx:128-129 -->

### Update

Three update commands exist: <!-- docs/cloud/get-started/service-accounts.mdx:177-179 -->

```bash
# Update name or description
tcld service-account update --id "<id>" -d "<new description>"

# Update account role
tcld service-account set-account-role --id "<id>" --ar "<role>"

# Update namespace permissions
tcld service-account set-namespace-permissions --id "<id>" -p "<namespace>=<permission>"
```
<!-- docs/cloud/get-started/service-accounts.mdx:177-185 -->

### Namespace-Scoped Lifecycle

When a namespace is deleted, all associated Namespace-scoped Service Accounts and their API keys are automatically deleted. <!-- docs/cloud/get-started/service-accounts.mdx:238-239 -->

---

## Account Operations (`tcld account`)

Alias: `a` <!-- docs/cloud/tcld/account.mdx:20 -->

### Get

```bash
tcld account get
```
<!-- docs/cloud/tcld/account.mdx:311-318 -->

Returns information about the Temporal Cloud account you are logged into. No modifiers. <!-- docs/cloud/tcld/account.mdx:313,319 -->

### List Regions

```bash
tcld account list-regions
```
<!-- docs/cloud/tcld/account.mdx:322-325 -->

Lists all regions where the account can provision namespaces. Alias: `l` <!-- docs/cloud/tcld/account.mdx:325 -->

### Audit Log

Subcommands for configuring audit log sinks: <!-- docs/cloud/tcld/account.mdx:28-30 -->

- `tcld account audit-log kinesis` (alias: `k`) -- Kinesis sinks: create, delete, get, list, update, validate <!-- docs/cloud/tcld/account.mdx:37-46 -->
- `tcld account audit-log pubsub` (alias: `ps`) -- Pub/Sub sinks: create, delete, get, list, update, validate <!-- docs/cloud/tcld/account.mdx:182-193 -->

### Metrics

```bash
tcld account metrics enable    # Enable metrics endpoint
tcld account metrics disable   # Disable metrics endpoint
```
<!-- docs/cloud/tcld/account.mdx:594-614 -->

End-entity certificates must be configured before enabling.
Managed via `tcld account metrics accepted-client-ca` subcommands: `add`, `list`, `set`, `remove`. <!-- docs/cloud/tcld/account.mdx:338-341,349-350 -->

---

## Key Differences: `tcld user` vs `tcld user-group` Namespace Permission Format

| Context | Flag | Format | Example |
|---------|------|--------|---------|
| `tcld user` commands | `--namespace-permission` | `<namespace>=<permission>` | `ns1.abc123=Admin` <!-- docs/cloud/tcld/user.mdx:135 --> |
| `tcld user-group` commands | `--namespace-role` | `<namespaceid>-<role>` | `mynamespace.abc123-read` <!-- docs/cloud/tcld/user-group.mdx:67 --> |

The permission values also differ in case:
- `tcld user`: `Admin` | `Write` | `Read` (title case) <!-- docs/cloud/tcld/user.mdx:137 -->
- `tcld user-group`: `admin` | `read` | `write` (lower case) <!-- docs/cloud/tcld/user-group.mdx:67 -->
