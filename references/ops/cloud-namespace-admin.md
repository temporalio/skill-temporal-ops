# Cloud Namespace Administration via tcld

Quick-reference for Cloud namespace lifecycle operations using `tcld namespace`.

Alias: `n` <!-- docs/cloud/tcld/namespace.mdx:22 -->

---

## Identity formats

| Concept | Format | Example |
|---|---|---|
| Namespace Name | `<namespace_name>` (2-39 chars, lowercase, letters/numbers/hyphens, must start with letter, end with letter or number) | `accounting-production` <!-- docs/cloud/get-started/namespaces.mdx:47-50 --> |
| Account ID | `<account_suffix>` (5+ chars) | `123de` <!-- docs/cloud/get-started/namespaces.mdx:65 --> |
| Namespace ID | `<namespace_name>.<account_suffix>` | `accounting-production.123de` <!-- docs/cloud/tcld/namespace.mdx:27 --> |
| Namespace endpoint | `<ns>.<acct>.tmprl.cloud:7233` | `accounting-production.123de.tmprl.cloud:7233` <!-- docs/cloud/get-started/namespaces.mdx:331 --> |
| Regional endpoint | `<region>.<cloud_provider>.api.temporal.io:7233` | `us-east-1.aws.api.temporal.io:7233` <!-- docs/cloud/get-started/namespaces.mdx:335 --> |

All `--namespace` / `-n` flags accept the **Namespace ID** (full form), not the short Namespace Name. <!-- docs/cloud/tcld/namespace.mdx:27 -->

If `--namespace` is omitted, the environment variable `$TEMPORAL_CLOUD_NAMESPACE` is used. <!-- docs/cloud/tcld/namespace.mdx:63 -->

---

## Limits

- Default account namespace limit: 10 (auto-increases as you create namespaces; for large-scale needs open a support ticket) <!-- docs/cloud/get-started/namespaces.mdx:184 -->
- Retention range: 1-90 days <!-- docs/cloud/get-started/namespaces.mdx:216 -->
- Max tags per namespace: 10 <!-- docs/cloud/get-started/namespaces.mdx:480 -->
- Tag key/value length: 1-63 characters <!-- docs/cloud/get-started/namespaces.mdx:483 -->
- Soft limit of 1000 unique tag keys per account <!-- docs/cloud/get-started/namespaces.mdx:487 -->

---

## Account commands (`tcld account`)

Commands for inspecting the Cloud account itself, before working with namespaces.

### Get account info

```bash
tcld account get
```
<!-- docs/cloud/tcld/account.mdx:311-319 -->

Returns account ID, name, and related metadata. No modifiers. <!-- docs/cloud/tcld/account.mdx:313,319 -->

### List available regions

```bash
tcld account list-regions
```
<!-- docs/cloud/tcld/account.mdx:321-325 -->

Lists all regions where the account can provision namespaces, including the cloud provider (AWS or GCP). Alias: `l` <!-- docs/cloud/tcld/account.mdx:323-325 -->

---

## Polling async operations (`tcld request`)

Namespace creation, deletion, region additions/removals, and failover are **asynchronous** operations. Poll their status with `tcld request get`. <!-- docs/cloud/tcld/request.mdx:16-17 -->

```bash
tcld request get --request-id <request_id>
```
<!-- docs/cloud/tcld/request.mdx:26 -->

Alias: `g`. The `--request-id` (alias `-r`) flag is required. <!-- docs/cloud/tcld/request.mdx:28,34 -->

**Pattern:** submit a mutation with a client-supplied `--request-id`, then poll until the state becomes terminal (fulfilled or rejected). Guard against persistent failure: if `tcld request get` itself returns a non-zero exit code, stop and report rather than looping indefinitely.

```bash
# 1. Submit with an idempotency key
tcld namespace create --namespace payments.a1b2c --region us-east-1 \
    --auth-method api_key --request-id create-01

# 2. Poll
tcld request get --request-id create-01

# 3. Inspect the namespace once ready
tcld namespace get --namespace payments.a1b2c
```
<!-- docs/cloud/tcld/request.mdx:42-44, docs/cloud/tcld/namespace.mdx:466-468 -->

---

## Feature flags and version

```bash
# Inspect account-level feature flags
tcld feature get

# Toggle a specific flag (dynamic subcommand name: toggle-<flag>)
tcld feature toggle-apikey
```
<!-- docs/cloud/tcld/feature.mdx:25-69 -->

`tcld feature` has two subcommands: `get` (alias `g`) and `toggle-*`. The `*` is replaced by the feature name. Example output: `Feature flag enable-apikey is now true`. <!-- docs/cloud/tcld/feature.mdx:52-75 -->

```bash
# tcld binary version
tcld version
```
<!-- docs/cloud/tcld/version.mdx:16-22 -->

---

## tcld namespace create

Alias: `c` <!-- docs/cloud/tcld/namespace.mdx:103 -->

```bash
tcld namespace create \
    --namespace <namespace_id> \
    --region <region> \
    --auth-method api_key
```
<!-- docs/cloud/tcld/namespace.mdx:129-134 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | Becomes part of the Namespace ID |
| `--region` | `--re` | Yes | One for standard, two for HA. See Regions docs |
| `--auth-method` | | No | `mtls` (default) or `api_key` <!-- docs/cloud/tcld/namespace.mdx:122 --> |
| `--ca-certificate` | `-c` | Conditional | Required if `--auth-method mtls` and no `--ca-certificate-file` |
| `--ca-certificate-file` | `--cf` | Conditional | Path to PEM file |
| `--certificate-filter-file` | `--cff` | No | JSON file defining cert filters |
| `--certificate-filter-input` | `--cfi` | No | JSON string defining cert filters |
| `--cloud-provider` | `--cp` | No | `aws` (default) or `gcp` <!-- docs/cloud/tcld/namespace.mdx:170 --> |
| `--connectivity-rule-ids` | `--ids` | No | Can be specified multiple times <!-- docs/cloud/tcld/namespace.mdx:177 --> |
| `--enable-delete-protection` | `--edp` | No | Default `false` <!-- docs/cloud/tcld/namespace.mdx:193-199 --> |
| `--endpoint` | `-e` | No | Codec server endpoint (must be HTTPS) <!-- docs/cloud/tcld/namespace.mdx:203 --> |
| `--include-credentials` | `--ic` | No | Include cross-origin credentials for codec server. Default `false` |
| `--pass-access-token` | `--pat` | No | Pass user access token to codec server. Default `false` |
| `--request-id` | `-r` | No | Async operation request ID |
| `--retention-days` | `--rd` | No | Default `30` <!-- docs/cloud/tcld/namespace.mdx:233 --> |
| `--search-attribute` | `--sa` | No | `name=type` format; can repeat. Types: `Bool`, `Datetime`, `Double`, `Int`, `Keyword`, `Text` <!-- docs/cloud/tcld/namespace.mdx:240-241 --> |
| `--tag` | `--t` | No | `key=value` format; can repeat <!-- docs/cloud/tcld/namespace.mdx:258 --> |
| `--user-namespace-permission` | `-p` | No | `email=permission` format; `Admin`, `Write`, `Read` <!-- docs/cloud/tcld/namespace.mdx:277-279 --> |

Example with HA (two regions), tags, and search attributes:

```bash
tcld namespace create \
    --namespace my-namespace.a1b2c \
    --region us-east-1 \
    --region us-west-2 \
    --auth-method api_key \
    --retention-days 30 \
    --search-attribute "customer_id=Int" \
    --tag "env=production" \
    --user-namespace-permission "user@example.com=Admin"
```

---

## tcld namespace get

Alias: `g` <!-- docs/cloud/tcld/namespace.mdx:450 -->

```bash
tcld namespace get \
    --namespace <namespace_id>
```
<!-- docs/cloud/tcld/namespace.mdx:466-468 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | No | Falls back to `$TEMPORAL_CLOUD_NAMESPACE` |

Output is JSON by default (no `--format` flag exists).

---

## tcld namespace list

Alias: `l` <!-- docs/cloud/tcld/namespace.mdx:473 -->

```bash
tcld namespace list
```
<!-- docs/cloud/tcld/namespace.mdx:476-478 -->

No modifiers. Returns JSON with `namespaces` array and `nextPageToken`. <!-- docs/cloud/get-started/namespaces.mdx:121-128 -->

---

## tcld namespace delete

Alias: `d` <!-- docs/cloud/tcld/namespace.mdx:299 -->

```bash
tcld namespace delete \
    --namespace <namespace_id>
```
<!-- docs/cloud/tcld/namespace.mdx:325-327 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--request-id` | `-r` | No | |
| `--resource-version` | `-v` | No | ETag; if omitted uses latest |

Deletion is permanent. All Workflow Executions and Task Queues are removed immediately. Closed Workflow Histories remain until their retention period expires. <!-- docs/cloud/get-started/namespaces.mdx:433-438 -->

### Delete protection

Enable via `--enable-delete-protection` / `--edp` at create time. <!-- docs/cloud/tcld/namespace.mdx:193-199 -->

To toggle on an existing namespace:

```bash
tcld namespace lifecycle set \
    --namespace <namespace_id> \
    --enable-delete-protection <Boolean>
```
<!-- docs/cloud/get-started/namespaces.mdx:468-471 -->

<!-- RESOLVED: tcld namespace lifecycle set is documented in docs/cloud/get-started/namespaces.mdx:468 but not in the tcld CLI reference (namespace.mdx). The command exists per the concepts page; the tcld reference page omits it. -->

---

## tcld namespace add-region

Upgrades a namespace to support High Availability by adding a replica region. <!-- docs/cloud/tcld/namespace.mdx:49 -->

```bash
tcld namespace add-region \
    --namespace <namespace_id> \
    --region <replica_region_name>
```
<!-- docs/cloud/tcld/namespace.mdx:82-85 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--region` | `--re` | Yes | Region name, e.g. `us-east-1` |
| `--cloud-provider` | | No | `aws` (default) or `gcp` <!-- docs/cloud/tcld/namespace.mdx:94 --> |
| `--request-id` | `-r` | No | |

Temporal Cloud sends an email alert once the Namespace is ready. <!-- docs/cloud/tcld/namespace.mdx:90 -->

---

## tcld namespace delete-region

Removes a replica region, disabling HA. Imposes a mandatory 7-day waiting period before re-enabling HA in the same location. <!-- docs/cloud/tcld/namespace.mdx:331-334 -->

```bash
tcld namespace delete-region \
    --namespace <namespace_id> \
    --region <region_name>
```
<!-- docs/cloud/tcld/namespace.mdx:359-362 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--region` | `--re` | Yes | Region to remove |
| `--cloud-provider` | | No | `aws` (default) or `gcp` <!-- docs/cloud/tcld/namespace.mdx:375-377 --> |
| `--request-id` | `-r` | No | |

---

## tcld namespace failover

Switches a namespace from its primary region to a replica region (requires HA). <!-- docs/cloud/tcld/namespace.mdx:381-382 -->

```bash
tcld namespace failover \
    --namespace <namespace_id> \
    --region <target_region>
```
<!-- docs/cloud/tcld/namespace.mdx:387-390 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--region` | `--re` | Yes | Region to fail over TO |
| `--ca-certificate` | `-c` | Conditional | Required unless `--ca-certificate-file` specified <!-- docs/cloud/tcld/namespace.mdx:432-436 --> |
| `--cloud-provider` | | No | `aws` (default) or `gcp` <!-- docs/cloud/tcld/namespace.mdx:443-444 --> |
| `--request-id` | `-r` | No | |

---

## tcld namespace retention

Alias: `r` <!-- docs/cloud/tcld/namespace.mdx:1621 -->

### retention get

Alias: `g` <!-- docs/cloud/tcld/namespace.mdx:1631 -->

```bash
tcld namespace retention get \
    --namespace <namespace_id>
```
<!-- docs/cloud/tcld/namespace.mdx:1646-1647 -->

### retention set

Alias: `s` <!-- docs/cloud/tcld/namespace.mdx:1655 -->

```bash
tcld namespace retention set \
    --namespace <namespace_id> \
    --retention-days <days>
```
<!-- docs/cloud/tcld/namespace.mdx:1678-1680 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--retention-days` | `--rd` | Yes | Range: 1-90 days <!-- docs/cloud/get-started/namespaces.mdx:216 --> |

---

## tcld namespace update-codec-server

See [codec-server.md](codec-server.md#namespace-level-cloud-or-self-hosted----web-ui-or-tcld) for the full flag table, HTTP contract, CORS, and authorization details.

---

## tcld namespace update-high-availability

Alias: `uha` <!-- docs/cloud/tcld/namespace.mdx:1761 -->

```bash
tcld namespace update-high-availability \
    --namespace <namespace_id> \
    --disable-auto-failover=true
```
<!-- docs/cloud/tcld/namespace.mdx:1784-1787 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--disable-auto-failover` | `-daf` | No | `true` or `false` (default). Controls automatic failovers. <!-- docs/cloud/tcld/namespace.mdx:1775-1779 --> |

---

## tcld namespace tags

Alias: `t` <!-- docs/cloud/tcld/namespace.mdx:1805 -->

### tags upsert

Add new tags or update existing tag values. Alias: `u` <!-- docs/cloud/tcld/namespace.mdx:1814 -->

```bash
tcld namespace tags upsert \
    --namespace <namespace_id> \
    --tag "key1=value1" \
    --tag "key2=updated"
```
<!-- docs/cloud/tcld/namespace.mdx:1846-1849 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--tag` | `--t` | Yes | `key=value` format; repeatable <!-- docs/cloud/tcld/namespace.mdx:1836-1838 --> |
| `--request-id` | `-r` | No | |

### tags remove

Remove tags by key. Alias: `rm` <!-- docs/cloud/tcld/namespace.mdx:1855 -->

```bash
tcld namespace tags remove \
    --namespace <namespace_id> \
    --tag-key "key1" \
    --tag-key "key2"
```
<!-- docs/cloud/tcld/namespace.mdx:1887-1892 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | Yes | |
| `--tag-key` | `--tk` | Yes | Key string; repeatable <!-- docs/cloud/tcld/namespace.mdx:1877-1879 --> |
| `--request-id` | `-r` | No | |

### Tag constraints

- Allowed characters: lowercase `a-z`, `0-9`, `.`, `_`, `-`, `@` <!-- docs/cloud/get-started/namespaces.mdx:484 -->
- Keys must be unique per namespace <!-- docs/cloud/get-started/namespaces.mdx:481 -->
- Only Account Admins and Account Owners can create/edit tags <!-- docs/cloud/get-started/namespaces.mdx:491 -->

---

## tcld namespace set-connectivity-rules

Alias: `scrs` <!-- docs/cloud/tcld/namespace.mdx:1900 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `n` | Yes | |
| `--connectivity-rule-ids` | `ids` | No | Repeatable. `--ids id1 --ids id2` <!-- docs/cloud/tcld/namespace.mdx:1902-1906 --> |
| `--remove-all` | | No | Acknowledges removal of all rules, enabling connectivity from any source <!-- docs/cloud/tcld/namespace.mdx:1914-1916 --> |

---

## tcld namespace search-attributes

Alias: `sa` <!-- docs/cloud/tcld/namespace.mdx:1442 -->

### search-attributes add

Alias: `a` <!-- docs/cloud/tcld/namespace.mdx:1457 -->

```bash
tcld namespace search-attributes add \
    --namespace <namespace_id> \
    --search-attribute "YourSearchAttribute1=Text" \
    --search-attribute "YourSearchAttribute2=Double"
```
<!-- docs/cloud/tcld/namespace.mdx:1523-1526 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | No | Falls back to `$TEMPORAL_CLOUD_NAMESPACE` |
| `--search-attribute` | `--sa` | Yes | `name=type` format; repeatable. Types: `Bool`, `Datetime`, `Double`, `Int`, `Keyword`, `Text` <!-- docs/cloud/tcld/namespace.mdx:1509-1516 --> |
| `--request-id` | `-r` | No | |
| `--resource-version` | `-v` | No | |

To delete a search attribute, contact Support at support.temporal.io. <!-- docs/cloud/tcld/namespace.mdx:1447-1448 -->

### search-attributes rename

```bash
tcld namespace search-attributes rename \
    --namespace <namespace_id> \
    --existing-name <old_name> \
    --new-name <new_name>
```
<!-- docs/cloud/tcld/namespace.mdx:1547-1551 -->

| Flag | Alias | Required | Notes |
|---|---|---|---|
| `--namespace` | `-n` | No | |
| `--existing-name` | `--en` | Yes | <!-- docs/cloud/tcld/namespace.mdx:1589 --> |
| `--new-name` | `--nn` | Yes | <!-- docs/cloud/tcld/namespace.mdx:1601 --> |
| `--request-id` | `-r` | No | |
| `--resource-version` | `-v` | No | |

---

## tcld namespace accepted-client-ca

Manages client CA certificates used to verify mTLS connections. Alias: `ca` <!-- docs/cloud/tcld/namespace.mdx:762 -->

| Subcommand | Alias | Purpose |
|---|---|---|
| `add` | `a` | Add CA certs <!-- docs/cloud/tcld/namespace.mdx:785 --> |
| `list` | `l` | List current CA certs <!-- docs/cloud/tcld/namespace.mdx:875 --> |
| `set` | `s` | Replace all CA certs (used for rollover) <!-- docs/cloud/tcld/namespace.mdx:1009 --> |
| `remove` | `r` | Remove specific CA certs <!-- docs/cloud/tcld/namespace.mdx:899 --> |

All subcommands accept `--namespace` / `-n`, `--request-id` / `-r`, `--resource-version` / `-v`.

Certificate can be supplied as:
- `--ca-certificate` / `-c` (base64-encoded string)
- `--ca-certificate-file` / `-f` (path to PEM file)

If both are specified, `--ca-certificate` takes precedence. <!-- docs/cloud/tcld/namespace.mdx:838-839 -->

The `remove` subcommand additionally supports `--ca-certificate-fingerprint` / `--fp` for removal by fingerprint. <!-- docs/cloud/tcld/namespace.mdx:986-993 -->

### CA certificate rollover procedure

1. Create a single PEM file with both old and new CA certificate blocks concatenated <!-- docs/cloud/tcld/namespace.mdx:1017-1026 -->
2. Run `tcld namespace accepted-client-ca set --ca-certificate-file <path>` <!-- docs/cloud/tcld/namespace.mdx:1031-1033 -->
3. Monitor traffic until old cert usage ceases <!-- docs/cloud/tcld/namespace.mdx:1035 -->
4. Run `set` again with only the new certificate <!-- docs/cloud/tcld/namespace.mdx:1037-1039 -->

Do NOT use a CA certificate signed with SHA-1 -- such signatures are rejected. <!-- docs/cloud/tcld/namespace.mdx:772-776 -->

---

## tcld namespace certificate-filters

Manages certificate filters that authorize client certificates based on DN fields. Alias: `cf` <!-- docs/cloud/tcld/namespace.mdx:1127 -->

| Subcommand | Alias | Purpose |
|---|---|---|
| `add` | | Add certificate filters <!-- docs/cloud/tcld/namespace.mdx:1133 --> |
| `import` | `imp` | Set (replace all) certificate filters <!-- docs/cloud/tcld/namespace.mdx:1339 --> |
| `export` | `exp` | Export current filters to file <!-- docs/cloud/tcld/namespace.mdx:1273 --> |
| `clear` | | Clear all filters (allows any client cert that chains to a configured CA) <!-- docs/cloud/tcld/namespace.mdx:1216-1224 --> |

Filter fields (at least one required): `commonName`, `organization`, `organizationalUnit`, `subjectAlternativeName` <!-- docs/cloud/tcld/namespace.mdx:1350-1354 -->

Filter input via `--certificate-filter-file` / `-f` or `--certificate-filter-input` / `-i`. Cannot specify both. <!-- docs/cloud/tcld/namespace.mdx:1364-1365 -->

JSON format: `{ "filters": [ { "commonName": "test1" } ] }` <!-- docs/cloud/tcld/namespace.mdx:153 -->

---

## tcld namespace export (Workflow History Exports)

Manages Workflow History Export sinks to S3. Alias: `es` <!-- docs/cloud/tcld/namespace.mdx:486 -->

| Subcommand | Purpose |
|---|---|
| `tcld namespace export s3 create` | Create an export sink <!-- docs/cloud/tcld/namespace.mdx:497 --> |
| `tcld namespace export s3 get` | Get sink details <!-- docs/cloud/tcld/namespace.mdx:549 --> |
| `tcld namespace export s3 delete` | Delete a sink <!-- docs/cloud/tcld/namespace.mdx:579 --> |
| `tcld namespace export s3 list` | List all sinks <!-- docs/cloud/tcld/namespace.mdx:619 --> |
| `tcld namespace export s3 update` | Modify a sink <!-- docs/cloud/tcld/namespace.mdx:649 --> |
| `tcld namespace export s3 validate` | Validate a sink <!-- docs/cloud/tcld/namespace.mdx:708 --> |

Key flags for create/validate:
- `--sink-name` (required) <!-- docs/cloud/tcld/namespace.mdx:521 -->
- `--s3-bucket-name` (required) <!-- docs/cloud/tcld/namespace.mdx:534 -->
- `--role-arn` (required) <!-- docs/cloud/tcld/namespace.mdx:527 -->
- `--kms-arn` (optional) <!-- docs/cloud/tcld/namespace.mdx:545 -->

---

## Endpoint and authentication summary

| Auth method | Endpoint type | Format |
|---|---|---|
| API key or mTLS | Namespace endpoint (recommended) | `<ns>.<acct>.tmprl.cloud:7233` <!-- docs/cloud/get-started/namespaces.mdx:331 --> |
| API key or mTLS | Regional endpoint | `<region>.<cloud_provider>.api.temporal.io:7233` <!-- docs/cloud/get-started/namespaces.mdx:335 --> |

- Namespace endpoints auto-route during HA failover -- Workers and Clients do not need endpoint changes <!-- docs/cloud/get-started/namespaces.mdx:334 -->
- When using mTLS with a regional endpoint, set `server_name` to the Namespace endpoint value <!-- docs/cloud/get-started/namespaces.mdx:338 -->
- Web UI URL: `https://cloud.temporal.io/namespaces/<namespace_id>` <!-- docs/cloud/get-started/namespaces.mdx:352 -->

---

## Access and permissions

- Creating a namespace requires Developer, Account Owner, or Global Admin account-level role <!-- docs/cloud/get-started/namespaces.mdx:178 -->
- The creator is automatically granted Namespace Admin permission <!-- docs/cloud/get-started/namespaces.mdx:176 -->
- Deleting a namespace requires Namespace Admin permission <!-- docs/cloud/get-started/namespaces.mdx:419 -->
- Tags: only Account Admins and Account Owners can create/edit <!-- docs/cloud/get-started/namespaces.mdx:491 -->

---

## Common anti-patterns

| Wrong | Right | Why |
|---|---|---|
| `--namespace my-ns` | `--namespace my-ns.a1b2c` | Namespace ID requires account suffix |
| `tcld namespace get --format json` | `tcld namespace get` | No `--format` flag; output is JSON by default |
| `tcld namespace search-attributes create` | `tcld namespace search-attributes add` | Subcommand is `add`, not `create` |
| `tcld namespace update --retention-days 30` | `tcld namespace retention set --retention-days 30` | Retention is its own subcommand tree |
| Short-name endpoint `my-ns:7233` | `my-ns.a1b2c.tmprl.cloud:7233` | Cloud requires full Namespace ID in endpoint |
