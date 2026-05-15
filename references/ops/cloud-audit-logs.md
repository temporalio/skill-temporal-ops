# Cloud Audit Logs

Audit Logs provide forensic access information for operations in the Temporal Cloud control plane. They answer "who, when, and what" questions about Temporal Cloud resources. <!-- docs/cloud/audit-logs.mdx:22-24 -->

Required role: Account Owner or Global Administrator to view Audit Logs via UI, use the API, or configure an Audit Log Integration. <!-- docs/cloud/audit-logs.mdx:27 -->

**Audit Logs do NOT capture data plane events** (Workflow Start, Workflow Terminate, Schedule Create, etc.). For closed Workflow Histories, use the Export feature instead. <!-- docs/cloud/audit-logs.mdx:30-33 -->

---

## Supported events

### Account
- `ChangeAccountPlanType`: Change Account Plan Type <!-- docs/cloud/audit-logs.mdx:39 -->
- `UpdateAccountAPI`: Configure Audit Logs, Configure Observability Endpoint <!-- docs/cloud/audit-logs.mdx:40 -->

### API Keys
- `CreateAPIKey`: Create API Key <!-- docs/cloud/audit-logs.mdx:42 -->
- `DeleteAPIKey`: Delete API Key <!-- docs/cloud/audit-logs.mdx:43 -->
- `UpdateAPIKey`: Update API Key <!-- docs/cloud/audit-logs.mdx:44 -->

### Connectivity Rules
- `CreateConnectivityRule`: Create Connectivity Rule <!-- docs/cloud/audit-logs.mdx:46 -->
- `DeleteConnectivityRule`: Delete Connectivity Rule <!-- docs/cloud/audit-logs.mdx:47 -->

### Namespace
- `CreateNamespaceAPI`: Create Namespace <!-- docs/cloud/audit-logs.mdx:49 -->
- `DeleteNamespaceAPI`: Delete Namespace <!-- docs/cloud/audit-logs.mdx:50 -->
- `FailoverNamespacesAPI`: Failover (for High Availability Namespaces) <!-- docs/cloud/audit-logs.mdx:51 -->
- `RenameCustomSearchAttributeAPI`: Rename Custom Search Attribute <!-- docs/cloud/audit-logs.mdx:52 -->
- `UpdateNamespaceAPI`: Retention period changes, replica edits, authentication method updates, custom search attribute updates, connectivity rule bindings <!-- docs/cloud/audit-logs.mdx:53 -->

### Namespace Export
- `CreateNamespaceExportSink`: Create Namespace Export Sink <!-- docs/cloud/audit-logs.mdx:55 -->
- `DeleteNamespaceExportSink`: Delete Namespace Export Sink <!-- docs/cloud/audit-logs.mdx:56 -->
- `UpdateNamespaceExportSink`: Update Namespace Export Sink <!-- docs/cloud/audit-logs.mdx:57 -->
- `ValidateNamespaceExportSink`: Validate Namespace Export Sink <!-- docs/cloud/audit-logs.mdx:58 -->

### Nexus Endpoint
- `CreateNexusEndpoint`: Create Nexus Endpoint <!-- docs/cloud/audit-logs.mdx:60 -->
- `DeleteNexusEndpoint`: Delete Nexus Endpoint <!-- docs/cloud/audit-logs.mdx:61 -->
- `UpdateNexusEndpoint`: Update Nexus Endpoint <!-- docs/cloud/audit-logs.mdx:62 -->

### Service Accounts
- `CreateServiceAccount`: Create Service Account <!-- docs/cloud/audit-logs.mdx:64 -->
- `CreateServiceAccountAPIKey`: Create Service Account API Key <!-- docs/cloud/audit-logs.mdx:65 -->
- `DeleteServiceAccount`: Delete Service Account <!-- docs/cloud/audit-logs.mdx:66 -->
- `UpdateServiceAccount`: Update Service Account <!-- docs/cloud/audit-logs.mdx:67 -->

### User
- `CreateUserAPI`: Create Users <!-- docs/cloud/audit-logs.mdx:69 -->
- `DeleteUserAPI`: Delete Users <!-- docs/cloud/audit-logs.mdx:70 -->
- `InviteUsersAPI`: Invite Users <!-- docs/cloud/audit-logs.mdx:71 -->
- `SetUserNamespaceAccessAPI`: Set User Namespace Access <!-- docs/cloud/audit-logs.mdx:72 -->
- `UpdateIdentityNamespacePermissionsAPI`: Update Identity Namespace Permissions <!-- docs/cloud/audit-logs.mdx:73 -->
- `UpdateUserAPI`: Update User Account-level Roles <!-- docs/cloud/audit-logs.mdx:74 -->
- `UpdateUserNamespacePermissionsAPI`: Update User Namespace Permissions <!-- docs/cloud/audit-logs.mdx:75 -->

### User Groups
- `CreateUserGroup`: Create User Group <!-- docs/cloud/audit-logs.mdx:77 -->
- `DeleteUserGroup`: Delete User Group <!-- docs/cloud/audit-logs.mdx:78 -->
- `SetUserGroupNamespaceAccess`: Set User Group Namespace Access <!-- docs/cloud/audit-logs.mdx:79 -->
- `UpdateUserGroup`: Update User Group <!-- docs/cloud/audit-logs.mdx:80 -->

---

## Audit Log format

```json
{
  "operation":          // Operation that was performed
  "principal":          // Information about who initiated the operation
  "raw_details":        // Details about the request
  "x_forwarded_for":    // The IP address(es) making the call
  "emit_time":          // Time the operation was recorded
  "log_id":             // Unique ID of the log entry
  "async_operation_id": // Optional async operation id set by the user when sending a request
  "request_id":         // DEPRECATED, use async_operation_id
  "status":             // Status, such as OK or ERROR
  "version":            // Version of the log entry
}
```
<!-- docs/cloud/audit-logs.mdx:92-104 -->

**Deprecation notice:** The `request_id` field is deprecated and is planned for removal on or after November 1 2026. Use `async_operation_id` instead. <!-- docs/cloud/audit-logs.mdx:84-87 -->

The `x_forwarded_for` field uses the `X-Forwarded-For` format: a comma-separated list of IP addresses, evaluated from last to first until meeting the first untrusted IP address. <!-- docs/cloud/audit-logs.mdx:109-111 -->

---

## Viewing Audit Logs

### Via the Cloud UI

1. Select **Settings**.
2. On the **Settings** page, select **Audit Logs**.

Up to 1000 events can be downloaded from the Audit Log UI to a local file. <!-- docs/cloud/audit-logs.mdx:197-200 -->

### Via the API

Audit Logs can be accessed using the Cloud Ops API. Use the API to build dashboards for viewing Audit Logs outside of Temporal Cloud. If your goal is to export logs continuously, use an Audit Log sink instead. <!-- docs/cloud/audit-logs.mdx:204-207 -->

Audit Logs are accessible for the past 30 days using the API. <!-- docs/cloud/audit-logs.mdx:209 -->

API filter parameters: <!-- docs/cloud/audit-logs.mdx:212-215 -->

| Parameter | Description |
|---|---|
| `StartTimeInclusive` | Filter for UTC time >= (defaults to 30 days ago) - optional |
| `EndTimeExclusive` | Filter for UTC time < (defaults to current time) - optional |
| `PageSize` | Cannot exceed 1000. Defaults to 100. - optional |
| `PageToken` | Page token for continuing from another response - optional |

---

## Audit Log sink configuration

Audit Logs can be sent to AWS Kinesis or GCP Pub/Sub. <!-- docs/cloud/audit-logs.mdx:161-164 -->

### AWS Kinesis

Prerequisites: an AWS account and Kinesis Data Streams. <!-- docs/cloud/audit-logs-aws.mdx:26 -->

An [AWS CloudFormation template](https://temporal-auditlogs-config.s3.us-west-2.amazonaws.com/cloudformation/iam-role-for-temporal-audit-logs.yaml) is available to create an IAM role with access to a Kinesis stream. <!-- docs/cloud/audit-logs-aws.mdx:31 -->

Kinesis has a rate limit of 1,000 messages per second. <!-- docs/cloud/audit-logs-aws.mdx:33 -->

Setup via Cloud UI: <!-- docs/cloud/audit-logs-aws.mdx:38-46 -->

1. Select **Settings** > **Audit Logs** > **Setup**.
2. Choose your **Access method**: **Auto** (configure CloudFormation from the Cloud UI) or **Manual** (download a template).
3. Enter the **Kinesis ARN**, **Role name**, and **AWS region**.
4. Follow the Auto or Manual steps to complete CloudFormation stack creation.

Use the **Verify** button to confirm Temporal can write to the stream. <!-- docs/cloud/audit-logs-aws.mdx:64 -->

First logs appear within 10 minutes after configuring the sink. <!-- docs/cloud/audit-logs-aws.mdx:72 -->

### GCP Pub/Sub

For manual setup: create a Pub/Sub topic and a service account in the same GCP project. <!-- docs/cloud/audit-logs-gcp.mdx:36-40 -->

Setup via Cloud UI: <!-- docs/cloud/audit-logs-gcp.mdx:43-55 -->

1. Select **Settings** > **Audit Logs** > **Setup**.
2. Select **Pub/Sub**.
3. Enter the **service account email** and **Topic name**.
4. Choose **Manual** or **Deploy with Terraform** to configure permissions.
5. Use the **Verify** button to confirm Temporal can write to the topic.
6. Click **Create**.

Audit Logs appear in Pub/Sub within 10 minutes. <!-- docs/cloud/audit-logs-gcp.mdx:56 -->

If using Terraform for deployment, the manual prerequisites (topic and service account creation) can be skipped. <!-- docs/cloud/audit-logs-gcp.mdx:30-33 -->

---

## Troubleshooting

### Sink status

The Audit Logs page of the Cloud UI shows the current status: <!-- docs/cloud/audit-logs.mdx:168-171 -->

- If an error is detected, a summary appears below the page title.
- If functioning normally, an **On** badge appears next to the page heading.

Temporal retains Audit Log information for up to 30 days. To retrieve logs up to the past 30 days, file a request. <!-- docs/cloud/audit-logs.mdx:176 -->

If you experience an issue with a sink, Temporal can provide missing audit information via a support ticket. <!-- docs/cloud/audit-logs.mdx:179-180 -->

### Deleting a sink

In the Cloud UI: **Settings** > **Audit Logs** > **Edit** > **Delete** at the bottom of the page. After confirmation, the sink is removed and logs stop flowing to the stream. <!-- docs/cloud/audit-logs.mdx:186-191 -->
