# Self-Hosted Admin — `temporal operator`

Control-plane operations for self-hosted Temporal Services.
All commands use the `temporal operator` CLI prefix and connect to `--address` (default `localhost:7233`). <!-- docs/cli/operator.mdx:526 -->

> **Cloud vs Self-Hosted**: `temporal operator` manages self-hosted clusters.
> Cloud equivalents use `tcld` — see [comparison table](#cloud-equivalent-comparison) at the end.

---

## Cluster Commands

### Health check

```bash
temporal operator cluster health
```
<!-- docs/cli/operator.mdx:77-79 -->

Returns health status of the Temporal Service. No subcommand-specific flags; uses [global flags](#global-flags-summary) only.

### Describe cluster

```bash
temporal operator cluster describe [--detail]
```
<!-- docs/cli/operator.mdx:59-67 -->

Shows Cluster Name, persistence store, and visibility store.

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--detail` | No | **bool** | Show history shard count and Cluster/Service version information. | <!-- docs/cli/operator.mdx:72 -->

### System info

```bash
temporal operator cluster system
```
<!-- docs/cli/operator.mdx:115-129 -->

Shows Server version, scheduling support, and more. Defaults to local Service; use `--frontend-address` to target a remote endpoint.

### List clusters

```bash
temporal operator cluster list [--limit max-count]
```
<!-- docs/cli/operator.mdx:85-92 -->

Lists remote Temporal Clusters registered to the local Service. Reports: name, ID, address, History Shard count, Failover version, availability.

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--limit` | No | **int** | Maximum number of Clusters to display. | <!-- docs/cli/operator.mdx:98 -->

### Remove cluster

```bash
temporal operator cluster remove --name YourClusterName
```
<!-- docs/cli/operator.mdx:101-107 -->

Removes a registered remote Cluster from the local Service.

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--name` | Yes | **string** | Cluster/Service name. | <!-- docs/cli/operator.mdx:112 -->

### Upsert cluster

```bash
temporal operator cluster upsert \
    --frontend-address "YourRemoteEndpoint:YourRemotePort" \
    --enable-connection false
```
<!-- docs/cli/operator.mdx:131-145 -->

Add, remove, or update a registered remote Cluster.

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--enable-connection` | No | **bool** | Set the connection to "enabled". | <!-- docs/cli/operator.mdx:151 -->
| `--enable-replication` | No | **bool** | Set the replication to "enabled". | <!-- docs/cli/operator.mdx:152 -->
| `--frontend-address` | Yes | **string** | Remote endpoint. | <!-- docs/cli/operator.mdx:153 -->

---

## Namespace Commands

### Create namespace

```bash
temporal operator namespace create \
    --namespace YourNewNamespaceName \
    [options]
```
<!-- docs/cli/operator.mdx:170-177 -->

Create a Namespace with multi-region replication:

```bash
temporal operator namespace create \
    --global \
    --namespace YourNewNamespaceName
```
<!-- docs/cli/operator.mdx:182-185 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--active-cluster` | No | **string** | Active Cluster (Service) name. | <!-- docs/cli/operator.mdx:205 -->
| `--cluster` | No | **string[]** | Cluster names. Can be passed multiple times. | <!-- docs/cli/operator.mdx:206 -->
| `--data` | No | **string[]** | Namespace data as `KEY=VALUE` pairs. Keys must be identifiers, values must be JSON. | <!-- docs/cli/operator.mdx:207 -->
| `--description` | No | **string** | Namespace description. | <!-- docs/cli/operator.mdx:208 -->
| `--email` | No | **string** | Owner email. | <!-- docs/cli/operator.mdx:209 -->
| `--global` | No | **bool** | Enable multi-region data replication. | <!-- docs/cli/operator.mdx:210 -->
| `--history-archival-state` | No | **string-enum** | Accepted values: `disabled`, `enabled`. Default `disabled`. | <!-- docs/cli/operator.mdx:211 -->
| `--history-uri` | No | **string** | Archive history to this URI. Once enabled, can't be changed. | <!-- docs/cli/operator.mdx:212 -->
| `--retention` | No | **duration** | Time to preserve closed Workflows before deletion. Default `72h`. | <!-- docs/cli/operator.mdx:213 -->
| `--visibility-archival-state` | No | **string-enum** | Accepted values: `disabled`, `enabled`. Default `disabled`. | <!-- docs/cli/operator.mdx:214 -->
| `--visibility-uri` | No | **string** | Archive visibility to this URI. Once enabled, can't be changed. | <!-- docs/cli/operator.mdx:215 -->

Note: URI values for archival states can't be changed once enabled. <!-- docs/cli/operator.mdx:199 -->

### Delete namespace

```bash
temporal operator namespace delete --namespace YourNamespaceName
```
<!-- docs/cli/operator.mdx:217-229 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--yes`, `-y` | No | **bool** | Request confirmation before deletion. | <!-- docs/cli/operator.mdx:236 -->

### Describe namespace

```bash
temporal operator namespace describe --namespace YourNamespaceName
```
<!-- docs/cli/operator.mdx:239-252 -->

Can also identify by `--namespace-id`:

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--namespace-id` | No | **string** | Namespace ID. | <!-- docs/cli/operator.mdx:257 -->

### List namespaces

```bash
temporal operator namespace list
```
<!-- docs/cli/operator.mdx:261-265 -->

Displays a detailed listing for all Namespaces on the Service. No subcommand-specific flags.

### Update namespace

```bash
temporal operator namespace update --namespace YourNamespaceName [options]
```
<!-- docs/cli/operator.mdx:270-275 -->

Examples:

```bash
# Assign active cluster
temporal operator namespace update \
    --namespace YourNamespaceName \
    --active-cluster NewActiveCluster

# Promote for multi-region replication
temporal operator namespace update \
    --namespace YourNamespaceName \
    --promote-global
```
<!-- docs/cli/operator.mdx:277-292 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--active-cluster` | No | **string** | Active Cluster (Service) name. | <!-- docs/cli/operator.mdx:307 -->
| `--cluster` | No | **string[]** | Cluster (Service) names. | <!-- docs/cli/operator.mdx:308 -->
| `--data` | No | **string[]** | Namespace data as `KEY=VALUE` pairs. | <!-- docs/cli/operator.mdx:309 -->
| `--description` | No | **string** | Namespace description. | <!-- docs/cli/operator.mdx:310 -->
| `--email` | No | **string** | Owner email. | <!-- docs/cli/operator.mdx:311 -->
| `--history-archival-state` | No | **string-enum** | Accepted values: `disabled`, `enabled`. | <!-- docs/cli/operator.mdx:312 -->
| `--history-uri` | No | **string** | Archive history URI. Once enabled, can't be changed. | <!-- docs/cli/operator.mdx:313 -->
| `--promote-global` | No | **bool** | Enable multi-region data replication. | <!-- docs/cli/operator.mdx:314 -->
| `--replication-state` | No | **string-enum** | Accepted values: `normal`, `handover`. | <!-- docs/cli/operator.mdx:315 -->
| `--retention` | No | **duration** | Time to preserve closed Workflows before deletion. | <!-- docs/cli/operator.mdx:316 -->
| `--visibility-archival-state` | No | **string-enum** | Accepted values: `disabled`, `enabled`. | <!-- docs/cli/operator.mdx:317 -->
| `--visibility-uri` | No | **string** | Archive visibility URI. Once enabled, can't be changed. | <!-- docs/cli/operator.mdx:318 -->

---

## Search Attribute Commands

Supported types: `Text`, `Keyword`, `Int`, `Double`, `Bool`, `Datetime`, `KeywordList`. <!-- docs/cli/operator.mdx:461 -->

### Create search attribute

```bash
temporal operator search-attribute create \
    --name YourAttributeName \
    --type Keyword
```
<!-- docs/cli/operator.mdx:468-475 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--name` | Yes | **string[]** | Search Attribute name. | <!-- docs/cli/operator.mdx:481 -->
| `--type` | Yes | **string-enum[]** | Accepted values: `Text`, `Keyword`, `Int`, `Double`, `Bool`, `Datetime`, `KeywordList`. | <!-- docs/cli/operator.mdx:482 -->

### List search attributes

```bash
temporal operator search-attribute list
```
<!-- docs/cli/operator.mdx:485-491 -->

Displays active Search Attributes that can be assigned or used in Workflow Queries. No subcommand-specific flags.

### Remove search attribute

```bash
temporal operator search-attribute remove --name YourAttributeName
```
<!-- docs/cli/operator.mdx:496-501 -->

Skip confirmation prompt:

```bash
temporal operator search-attribute remove --name YourAttributeName --yes
```
<!-- docs/cli/operator.mdx:506-510 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--name` | Yes | **string[]** | Search Attribute name. | <!-- docs/cli/operator.mdx:516 -->
| `--yes`, `-y` | No | **bool** | Don't prompt to confirm removal. | <!-- docs/cli/operator.mdx:517 -->

> **Self-hosted deletion note**: `remove` de-registers custom attributes from the queryable set ("Remove custom Search Attributes from the options that can be assigned or used with Workflow Queries" <!-- docs/cli/operator.mdx:497-498 -->). Permanent deletion from the backing store may require additional steps. The Cloud docs note: "If you wish to delete a Search Attribute, please contact Support." <!-- docs/cli/operator.mdx:464-465 -->

---

## Nexus Endpoint Commands

### Create endpoint

```bash
temporal operator nexus endpoint create \
    --name your-endpoint \
    --target-namespace your-namespace \
    --target-task-queue your-task-queue \
    --description-file DESCRIPTION.md
```
<!-- docs/cli/operator.mdx:341-359 -->

Target is either a Worker (`--target-namespace` + `--target-task-queue`) or an external URL (`--target-url`). <!-- docs/cli/operator.mdx:348-350 -->

Fails if an Endpoint with the same name already exists. <!-- docs/cli/operator.mdx:351-352 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--description` | No | **string** | Endpoint description. May use Markdown. | <!-- docs/cli/operator.mdx:365 -->
| `--description-file` | No | **string** | Path to description file. May use Markdown. | <!-- docs/cli/operator.mdx:366 -->
| `--name` | Yes | **string** | Endpoint name. | <!-- docs/cli/operator.mdx:367 -->
| `--target-namespace` | No | **string** | Namespace where handler Worker polls for Nexus tasks. | <!-- docs/cli/operator.mdx:368 -->
| `--target-task-queue` | No | **string** | Task Queue that handler Worker polls for Nexus tasks. | <!-- docs/cli/operator.mdx:369 -->
| `--target-url` | No | **string** | External endpoint URL. _(Experimental)_ | <!-- docs/cli/operator.mdx:370 -->

### Delete endpoint

```bash
temporal operator nexus endpoint delete --name your-endpoint
```
<!-- docs/cli/operator.mdx:374-378 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--name` | Yes | **string** | Endpoint name. | <!-- docs/cli/operator.mdx:383 -->

### Get endpoint (EXPERIMENTAL)

```bash
temporal operator nexus endpoint get --name your-endpoint
```
<!-- docs/cli/operator.mdx:387-391 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--name` | Yes | **string** | Endpoint name. | <!-- docs/cli/operator.mdx:396 -->

### List endpoints

```bash
temporal operator nexus endpoint list
```
<!-- docs/cli/operator.mdx:400-405 -->

No subcommand-specific flags.

### Update endpoint

```bash
temporal operator nexus endpoint update \
    --name your-endpoint \
    --target-task-queue your-other-queue
```
<!-- docs/cli/operator.mdx:410-429 -->

Patches the endpoint; existing fields for which flags are not provided are left unchanged. <!-- docs/cli/operator.mdx:419-420 -->

| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--description` | No | **string** | Endpoint description. May use Markdown. | <!-- docs/cli/operator.mdx:441 -->
| `--description-file` | No | **string** | Path to description file. May use Markdown. | <!-- docs/cli/operator.mdx:442 -->
| `--name` | Yes | **string** | Endpoint name. | <!-- docs/cli/operator.mdx:443 -->
| `--target-namespace` | No | **string** | Namespace where handler Worker polls for Nexus tasks. | <!-- docs/cli/operator.mdx:444 -->
| `--target-task-queue` | No | **string** | Task Queue that handler Worker polls for Nexus tasks. | <!-- docs/cli/operator.mdx:445 -->
| `--target-url` | No | **string** | External endpoint URL. _(Experimental)_ | <!-- docs/cli/operator.mdx:446 -->
| `--unset-description` | No | **bool** | Unset the description. | <!-- docs/cli/operator.mdx:447 -->

---

## Global Flags Summary

Key global flags applicable to all `temporal operator` commands: <!-- docs/cli/operator.mdx:520-558 -->

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--address` | **string** | `localhost:7233` | Temporal Service gRPC endpoint. | <!-- docs/cli/operator.mdx:526 -->
| `--namespace`, `-n` | **string** | `default` | Temporal Service Namespace. | <!-- docs/cli/operator.mdx:544 -->
| `--api-key` | **string** | | API key for request. | <!-- docs/cli/operator.mdx:527 -->
| `--tls` | **bool** | | Enable base TLS encryption. Defaulted to true if api-key or other TLS options are present. | <!-- docs/cli/operator.mdx:549 -->
| `--tls-ca-path` | **string** | | Path to server CA certificate. | <!-- docs/cli/operator.mdx:551 -->
| `--tls-cert-path` | **string** | | Path to x509 certificate. | <!-- docs/cli/operator.mdx:553 -->
| `--tls-key-path` | **string** | | Path to x509 private key. | <!-- docs/cli/operator.mdx:557 -->
| `--tls-server-name` | **string** | | Override target TLS server name. | <!-- docs/cli/operator.mdx:558 -->
| `--output`, `-o` | **string-enum** | `text` | Non-logging data output format. Accepted values: `text`, `json`, `jsonl`, `none`. | <!-- docs/cli/operator.mdx:546 -->
| `--log-level` | **string-enum** | `never` | Log level. Accepted values: `debug`, `info`, `warn`, `error`, `never`. | <!-- docs/cli/operator.mdx:543 -->
| `--env` | **string** | `default` | Active environment name. | <!-- docs/cli/operator.mdx:538 -->
| `--config-file` | **string** | | TOML config file path. | <!-- docs/cli/operator.mdx:535 -->

See `docs/cli/operator.mdx` lines 520-558 for the full list.

---

## Cloud-Equivalent Comparison

| Operation | Self-Hosted (`temporal operator`) | Cloud (`tcld`) |
|-----------|-----------------------------------|----------------|
| Check cluster health | `temporal operator cluster health` | N/A (Cloud-managed) |
| Describe cluster | `temporal operator cluster describe` | N/A (Cloud-managed) |
| Server system info | `temporal operator cluster system` | N/A (Cloud-managed) |
| List clusters | `temporal operator cluster list` | N/A (Cloud-managed) |
| Create namespace | `temporal operator namespace create` | `tcld namespace create` |
| Delete namespace | `temporal operator namespace delete` | `tcld namespace delete` |
| Describe namespace | `temporal operator namespace describe` | `tcld namespace get` |
| List namespaces | `temporal operator namespace list` | `tcld namespace list` |
| Update namespace | `temporal operator namespace update` | No single equivalent; use per-attribute subcommands (`retention set`, `capacity update`, `auth-method set`, `tags`, …) <!-- docs/cloud/tcld/namespace.mdx#retention #capacity #auth-method #tags --> |
| Create search attribute | `temporal operator search-attribute create` | `tcld namespace search-attributes add` |
| List search attributes | `temporal operator search-attribute list` | No tcld subcommand; use Cloud UI or Cloud Ops API <!-- docs/cloud/tcld/namespace.mdx:1444-1445 (only add and rename exist) --> |
| Remove search attribute | `temporal operator search-attribute remove` | `tcld namespace search-attributes rename` <!-- Note: Cloud renames rather than removes; deletion requires Support --> |
| Create Nexus endpoint | `temporal operator nexus endpoint create` | `tcld nexus endpoint create` <!-- docs/cloud/tcld/nexus.mdx:147 --> |
| Delete Nexus endpoint | `temporal operator nexus endpoint delete` | `tcld nexus endpoint delete` <!-- docs/cloud/tcld/nexus.mdx:198 --> |
| Get Nexus endpoint | `temporal operator nexus endpoint get` | `tcld nexus endpoint get` <!-- docs/cloud/tcld/nexus.mdx:223 --> |
| List Nexus endpoints | `temporal operator nexus endpoint list` | `tcld nexus endpoint list` <!-- docs/cloud/tcld/nexus.mdx:235 --> |
| Update Nexus endpoint | `temporal operator nexus endpoint update` | `tcld nexus endpoint update` <!-- docs/cloud/tcld/nexus.mdx:241 --> |
