# Cloud Export and Connectivity

Quick-reference for Workflow History Export (S3/GCS), private connectivity (AWS PrivateLink / GCP PSC), connectivity rules, and the Cloud Ops API.

---

## Workflow History Export

Workflow History Export sends closed Workflow Histories from Temporal Cloud to cloud object storage (AWS S3 or GCP GCS). <!-- docs/cloud/export.mdx:20 -->

- Export format is **protobuf binary**, using the [`WorkflowExecutions`](https://github.com/temporalio/api/blob/master/temporal/api/export/v1/message.proto) proto. <!-- docs/cloud/export.mdx:34 -->
- Exports run **hourly, beginning 10 minutes after the hour**. Allow up to 24 hours for a closed Workflow to appear. <!-- docs/cloud/export.mdx:28-29 -->
- Delivery is guaranteed **at least once**. <!-- docs/cloud/export.mdx:30 -->
- Archival (self-hosted feature) is **not** supported in Temporal Cloud; Export is the equivalent. <!-- docs/cloud/export.mdx:25-26 -->

### Export directory structure

```
//[bucket-name]/temporal-workflow-history/export/[Namespace]/[Year]/[Month]/[Day]/[Hour]/[Minute]/
```
<!-- docs/cloud/export.mdx:163 -->

The time in the path is when the export uploads to object storage, not the Workflow completion time. <!-- docs/cloud/export.mdx:166 -->

### Prerequisites

1. A cloud account in the cloud provider where the Namespace is hosted. <!-- docs/cloud/export.mdx:131 -->
2. An object storage bucket available to receive the exported history. <!-- docs/cloud/export.mdx:132 -->
3. The S3 bucket or GCS bucket **must reside in the same region as the Namespace**. <!-- docs/cloud/aws-export-s3.mdx:30, docs/cloud/gcp-export-gcs.mdx:33 -->

### AWS S3 export setup via tcld

Create an export sink:

```bash
tcld namespace export s3 create \
    --namespace <namespace_id> \
    --sink-name <sink_name> \
    --s3-bucket-name <bucket_name> \
    --role-arn <role_arn>
```
<!-- docs/cloud/tcld/namespace.mdx:503-508 -->

Required flags: `--namespace` (`-n`), `--sink-name`, `--s3-bucket-name`, `--role-arn`. <!-- docs/cloud/tcld/namespace.mdx:519-536 -->

Optional: `--kms-arn` (ARN of KMS key for encryption). <!-- docs/cloud/tcld/namespace.mdx:545-548 -->

Check status:

```bash
tcld namespace export s3 get \
    --namespace <namespace_id> \
    --sink-name <sink_name>
```
<!-- docs/cloud/tcld/namespace.mdx:557-559 -->

Example output fields: `state` (`Active`), `health` (`Ok`), `latestDataExportTime`, `lastHealthCheckTime`. <!-- docs/cloud/aws-export-s3.mdx:124-146 -->

Other S3 subcommands:

| Subcommand | Purpose |
|---|---|
| `tcld namespace export s3 list` | List all export sinks for a Namespace <!-- docs/cloud/tcld/namespace.mdx:619 --> |
| `tcld namespace export s3 update` | Modify an existing sink (e.g. `--enabled true`) <!-- docs/cloud/tcld/namespace.mdx:649 --> |
| `tcld namespace export s3 delete` | Remove an export sink <!-- docs/cloud/tcld/namespace.mdx:579 --> |
| `tcld namespace export s3 validate` | Validate the sink configuration <!-- docs/cloud/tcld/namespace.mdx:708 --> |

Alias for the `export` subgroup: `es`. <!-- docs/cloud/tcld/namespace.mdx:486 -->

### GCP GCS export setup via tcld

GCS export uses a separate subcommand group: `tcld namespace export gcs`. <!-- docs/cloud/gcp-export-gcs.mdx:99 -->

Create an export sink:

```bash
tcld namespace export gcs create \
    --namespace <namespace_id> \
    --sink-name <sink_name> \
    --service-account-email <sa_email> \
    --gcs-bucket <bucket_name>
```
<!-- docs/cloud/gcp-export-gcs.mdx:119-129 -->

GCS subcommands: `create`, `update`, `validate`, `get`, `delete`, `list`. <!-- docs/cloud/gcp-export-gcs.mdx:106-117 -->

GCS prerequisites specific to GCP:
- Only **single-region** buckets are supported (not multi-region or dual-region). <!-- docs/cloud/gcp-export-gcs.mdx:32 -->
- A service account in the same project must grant Temporal permission to write to the bucket. <!-- docs/cloud/gcp-export-gcs.mdx:37 -->
- CMEK (customer-managed encryption keys) is optional. <!-- docs/cloud/gcp-export-gcs.mdx:31 -->

### Export with High Availability Namespaces

- Export is tied to the **specific region** where it was originally configured; it does **not** failover automatically. <!-- docs/cloud/export.mdx:201-202 -->
- If the primary region has an outage, exports are unavailable until that region recovers, then resume and include Workflow histories that occurred during the outage. <!-- docs/cloud/export.mdx:212 -->

### Monitoring exports

- **Object storage**: inspect files under the directory structure above. <!-- docs/cloud/export.mdx:158 -->
- **Cloud UI**: "Last Successful Export" and "Last Status Check" timestamps on the Export screen. <!-- docs/cloud/export.mdx:170-173 -->
- **Metrics**: `temporal_cloud_v1_total_action_count` with label `is_background="true"`. <!-- docs/cloud/export.mdx:180 -->
- **Email**: alerts sent to Namespace Administrator, Account Owner, and Global Administrator roles on export failure due to user-related errors. <!-- docs/cloud/export.mdx:183 -->

### Verify export setup

From the Export configuration page in the Cloud UI, select **Verify** to validate that Temporal can write a test file to the object store. <!-- docs/cloud/export.mdx:149-151 -->

---

## Private Connectivity

Temporal Cloud supports private connectivity via **AWS PrivateLink** or **GCP Private Service Connect (PSC)** in addition to public internet endpoints. <!-- docs/cloud/connectivity/index.mdx:26 -->

Namespace access is always authenticated via API keys or mTLS regardless of connectivity method. <!-- docs/cloud/connectivity/index.mdx:28 -->

### Three-step setup process

1. **Set up the private connection** from your VPC to the region where the Namespace is located. <!-- docs/cloud/connectivity/index.mdx:36 -->
2. **Update private DNS and/or client configuration** to use the private connection. Activating private connectivity does not change Namespace or Regional Endpoints automatically. <!-- docs/cloud/connectivity/index.mdx:37 -->
3. **Create a Connectivity Rule** (required for GCP PSC, optional for AWS PrivateLink) and attach it to the target Namespace(s). <!-- docs/cloud/connectivity/index.mdx:38 -->

### AWS PrivateLink key facts

- The PrivateLink endpoint **must be in the same region** as the Temporal Cloud Namespace. Cross-region endpoints are not supported. <!-- docs/cloud/connectivity/aws-connectivity.mdx:36-39 -->
- PrivateLink endpoint services are **regional** -- individual Namespaces do not use separate services. <!-- docs/cloud/connectivity/aws-connectivity.mdx:57-58 -->
- Security group must accept **TCP ingress on port 7233**. <!-- docs/cloud/connectivity/aws-connectivity.mdx:68 -->
- After the VPC endpoint status is `Available`, configure private DNS or direct VPCE targeting. <!-- docs/cloud/connectivity/aws-connectivity.mdx:77 -->
- **Direct VPCE targeting** (without per-Namespace DNS) works for single-region Namespaces only; set `ServerName` / SNI override to the Namespace Endpoint. Not compatible with HA Namespaces. <!-- docs/cloud/connectivity/aws-connectivity.mdx:196-213 -->

### GCP Private Service Connect key facts

- PSC endpoint must be in the **same region** as the Namespace. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:39 -->
- PSC endpoint stays in **`Pending`** until a matching Connectivity Rule is created -- the Connectivity Rule is the approval step. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:77, 83-84 -->
- Automatic failover via Temporal Cloud DNS is **not currently supported** with GCP PSC; manual worker updates are required on failover. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:33-36 -->

### Client configuration without private DNS

If you cannot set up private DNS, update two settings in your Temporal clients: <!-- docs/cloud/connectivity/index.mdx:215-216 -->

1. Set endpoint server address to the PrivateLink DNS name or PSC IP address, port `7233`. <!-- docs/cloud/connectivity/index.mdx:217 -->
2. Set TLS server name override: <!-- docs/cloud/connectivity/index.mdx:218 -->

| Auth method | TLS server name |
|---|---|
| mTLS (single-region) | Namespace Endpoint, e.g. `my-namespace.my-account.tmprl.cloud` |
| API key (single-region) | Regional API endpoint, e.g. `us-east-1.aws.api.temporal.io` |
| Multi-region (mTLS or API key) | Active region endpoint, e.g. `aws-us-east-1.region.tmprl.cloud` |
<!-- docs/cloud/connectivity/index.mdx:222-226 -->

### Control plane connectivity

- The control plane (`saas-api.tmprl.cloud`) is accessible via public internet and optionally via AWS PrivateLink. <!-- docs/cloud/connectivity/index.mdx:346-347 -->
- Control plane PrivateLink is in `us-west-2` with service name `com.amazonaws.vpce.us-west-2.vpce-svc-0c57a5930b6f6be0e`. <!-- docs/cloud/connectivity/index.mdx:358-359 -->
- Control plane private connectivity does **not** block public internet access to the control plane. <!-- docs/cloud/connectivity/index.mdx:351 -->

---

## Connectivity Rules

Connectivity Rules restrict the network paths that can reach a Namespace. They are enforced by Temporal Cloud and do not create or modify the underlying network connection. <!-- docs/cloud/connectivity/index.mdx:69 -->

### Default behavior

A Namespace with **zero** Connectivity Rules is reachable over the public internet and any private connections already configured to the region. <!-- docs/cloud/connectivity/index.mdx:71 -->

When one or more rules are attached, Temporal Cloud **immediately blocks** any traffic that does not match a rule. <!-- docs/cloud/connectivity/index.mdx:73 -->

The Web UI is **not** subject to connectivity rule enforcement. <!-- docs/cloud/connectivity/index.mdx:61-63 -->

### When you need a Connectivity Rule

| Provider | Required? | Why |
|---|---|---|
| AWS PrivateLink | Optional -- add only to enforce private-only access | PrivateLink becomes usable when VPC endpoint is `Available` without any rule |
| GCP PSC | **Required** | PSC endpoint stays `Pending` until a matching rule is created |
<!-- docs/cloud/connectivity/index.mdx:79-83 -->

### Rule parameters

**Public rule**: no parameters needed. Only one public rule allowed per account. <!-- docs/cloud/connectivity/index.mdx:84, 111 -->

**AWS PrivateLink private rule** requires:
- `--connection-id`: VPC endpoint identifier (`vpce-...` value), not the endpoint service or DNS name. <!-- docs/cloud/connectivity/index.mdx:88 -->
- `--region`: Region prefixed with `aws-` (e.g. `aws-us-east-1`). Must match Namespace region. <!-- docs/cloud/connectivity/index.mdx:89 -->

**GCP PSC private rule** requires:
- `--connection-id`: PSC connection identifier (numeric string, e.g. `1234567890123456789`). <!-- docs/cloud/connectivity/index.mdx:92 -->
- `--region`: Region prefixed with `gcp-` (e.g. `gcp-us-east1`). Must match Namespace region. <!-- docs/cloud/connectivity/index.mdx:93 -->
- `--gcp-project-id`: GCP project where the PSC connection was created. <!-- docs/cloud/connectivity/index.mdx:95 -->

### Permissions and limits

- Only **Account Admins and Account Owners** can create/manage connectivity rules. <!-- docs/cloud/connectivity/index.mdx:107 -->
- Default: 5 private rules per Namespace, 50 private rules per account. <!-- docs/cloud/connectivity/index.mdx:109 -->
- Contact support to raise limits. <!-- docs/cloud/connectivity/index.mdx:109 -->

### tcld connectivity-rule commands

Alias: `cr`. <!-- docs/cloud/tcld/connectivity-rule.mdx:20 -->

Create a private rule (AWS):

```bash
tcld connectivity-rule create --connectivity-type private --connection-id "vpce-00939a7ed9EXAMPLE" --region "aws-us-east-1"
```
<!-- docs/cloud/connectivity/index.mdx:120 -->

Create a private rule (GCP):

```bash
tcld connectivity-rule create --connectivity-type private --connection-id "1234567890" --region "gcp-us-central1" --gcp-project-id "my-project-123"
```
<!-- docs/cloud/connectivity/index.mdx:126 -->

Create a public rule (once per account):

```bash
tcld connectivity-rule create --connectivity-type public
```
<!-- docs/cloud/connectivity/index.mdx:132 -->

Other subcommands:

| Subcommand | Purpose |
|---|---|
| `tcld connectivity-rule get --connectivity-rule-id <id>` | Get a rule <!-- docs/cloud/tcld/connectivity-rule.mdx:70-76 --> |
| `tcld connectivity-rule delete --connectivity-rule-id <id>` | Delete a rule <!-- docs/cloud/tcld/connectivity-rule.mdx:60-66 --> |
| `tcld connectivity-rule list` | List all rules (optionally filter by `--namespace`) <!-- docs/cloud/tcld/connectivity-rule.mdx:82-90 --> |

`--connectivity-type` values: `private`, `public`. <!-- docs/cloud/tcld/connectivity-rule.mdx:40 -->

### Attaching rules to a Namespace

```bash
tcld namespace set-connectivity-rules \
    --namespace "my-namespace.abc123" \
    --connectivity-rule-ids "rule-id-1" \
    --connectivity-rule-ids "rule-id-2"
```
<!-- docs/cloud/connectivity/index.mdx:162-163 -->

Alias: `tcld n scrs`. <!-- docs/cloud/tcld/namespace.mdx:1899 -->

Rules are attached **as a set** -- to remove one rule while keeping others, re-specify only the rules to keep. <!-- docs/cloud/connectivity/index.mdx:171-175 -->

Remove all rules (makes Namespace public again):

```bash
tcld namespace set-connectivity-rules --namespace "my-namespace.abc123" --remove-all
```
<!-- docs/cloud/connectivity/index.mdx:180 -->

Rules can also be set at Namespace creation time with `--connectivity-rule-ids`:

```bash
tcld namespace create \
    --namespace test-namespace.a1b2c \
    --region us-east-1 \
    --auth-method api_key \
    --connectivity-rule-ids <rule_id1> \
    --connectivity-rule-ids <rule_id2>
```
<!-- docs/cloud/tcld/namespace.mdx:185-191 -->

View rules for a Namespace:

```bash
tcld connectivity-rule list -n "my-namespace.abc123"
```
<!-- docs/cloud/connectivity/index.mdx:204-205 -->

Or view them as part of `tcld namespace get`. <!-- docs/cloud/connectivity/index.mdx:196-198 -->

---

## Cloud Ops API

For the full Cloud Ops API reference (endpoints, Go SDK, protobuf compilation, rate limits, use cases), see the standalone [cloud-ops-api.md](cloud-ops-api.md).

Quick reference:

- **URL** (HTTP and gRPC): `saas-api.tmprl.cloud` (port 443 for gRPC). <!-- docs/cloud/operation-api.mdx:25, 135 -->
- **Account-level rate limit**: 160 RPS. <!-- docs/cloud/operation-api.mdx:152 -->
- **Terraform**: The [Temporal Cloud Terraform Provider](https://registry.terraform.io/providers/temporalio/temporalcloud/latest) uses the Cloud Ops API. For the full Terraform reference, see [cloud-terraform.md](cloud-terraform.md). <!-- docs/cloud/terraform-provider.mdx:17-18 -->

---

## Troubleshooting pointers

### Export not delivering files

1. Check export sink state and health: `tcld namespace export s3 get` / `tcld namespace export gcs get`. Look for `state: Active` and `health: Ok`. <!-- docs/cloud/aws-export-s3.mdx:114-146 -->
2. Verify IAM role / service account permissions allow Temporal to write to the bucket.
3. Confirm bucket is in the **same region** as the Namespace. <!-- docs/cloud/aws-export-s3.mdx:30 -->
4. For HA Namespaces, export runs from the region where it was originally configured, not the active region. <!-- docs/cloud/export.mdx:201-202 -->

### PSC endpoint stuck in Pending

- Most common cause: no Connectivity Rule exists for the connection ID. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:87 -->
- Check that `--connection-id`, `--region`, and `--gcp-project-id` in the Connectivity Rule match the endpoint exactly. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:89 -->

### PrivateLink TLS handshake fails

- If using API key auth over PrivateLink/PSC with the wrong TLS server name, the handshake fails with `connection reset by peer` even though `nc` shows the port is open. <!-- docs/cloud/connectivity/index.mdx:228 -->
- Verify the TLS server name override matches the auth-method table above.

### Network connectivity check

```bash
nc -zv <endpoint_host> 7233
```
<!-- docs/cloud/connectivity/index.mdx:335 -->

For full connectivity diagnosis, see the triage `../triage/connectivity.md` reference.
