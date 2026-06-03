# Cloud Export

Quick-reference for Workflow History Export to cloud object storage (AWS S3 or GCP GCS).

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

## Troubleshooting

### Export not delivering files

1. Check export sink state and health: `tcld namespace export s3 get` / `tcld namespace export gcs get`. Look for `state: Active` and `health: Ok`. <!-- docs/cloud/aws-export-s3.mdx:114-146 -->
2. Verify IAM role / service account permissions allow Temporal to write to the bucket.
3. Confirm bucket is in the **same region** as the Namespace. <!-- docs/cloud/aws-export-s3.mdx:30 -->
4. For HA Namespaces, export runs from the region where it was originally configured, not the active region. <!-- docs/cloud/export.mdx:201-202 -->
