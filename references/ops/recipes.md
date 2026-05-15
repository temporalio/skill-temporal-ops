# Ops Recipes

End-to-end operational playbooks that chain commands from the ops reference files.
For triage-focused walkthroughs, see `../recipes.md`.

---

## (a) Check current APS and capacity mode for a Cloud namespace

**When to use:** You need to know whether a namespace is On-Demand or Provisioned and what its current APS limit is.

1. Get the namespace details:

   ```bash
   tcld namespace get \
       --namespace <namespace_name>.<account_suffix>
   ```
   <!-- docs/cloud/tcld/namespace.mdx:466-468 -->

   Output is JSON by default (no `--format` flag exists). <!-- undocumented: source = tcld CLI behavior; tcld commands emit JSON without a --format flag -->

2. In the JSON output, look for the capacity configuration section. Key fields:

   - **Capacity mode**: `on_demand` or `provisioned`. <!-- docs/cloud/capacity-modes.mdx:212-213 -->
   - **TRU count** (if provisioned): the number of Temporal Resource Units allocated. Valid values: 2, 3, 4, 6, 8, 10, 12. <!-- docs/cloud/capacity-modes.mdx:142, 149 -->
   - **APS limit**: On-Demand default is 500; each TRU provides 500 APS. <!-- docs/cloud/capacity-modes.mdx:107, 140 -->

3. To check whether throttling is occurring, look for `temporal_cloud_v0_resource_exhausted_errors` in your metrics. <!-- docs/best-practices/managing-aps-limits.mdx:237 -->

---

## (b) Switch capacity mode from On-Demand to Provisioned

**When to use:** You are preparing for a planned spike (load test, promotion, migration) and need to pre-provision capacity beyond the On-Demand auto-scaling limit.

1. Confirm current capacity mode (see playbook (a)):

   ```bash
   tcld namespace get \
       --namespace <namespace_name>.<account_suffix>
   ```
   <!-- docs/cloud/tcld/namespace.mdx:466-468 -->

2. Switch to Provisioned mode with the desired TRU count:

   ```bash
   tcld namespace capacity update \
       --namespace <namespace_name>.<account_suffix> \
       --capacity-mode provisioned \
       --capacity-value <tru_count>
   ```
   <!-- docs/cloud/capacity-modes.mdx:208 -->

   Valid `--capacity-value` values: 2, 3, 4, 6, 8, 10, 12. <!-- docs/cloud/capacity-modes.mdx:142, 149 -->

   Temporal aims to provision the additional capacity within two minutes. <!-- docs/cloud/capacity-modes.mdx:150 -->

   For requests in excess of 4 TRUs in regions outside of the US, submit a support ticket to ensure capacity availability. <!-- docs/cloud/capacity-modes.mdx:153-156 -->

   Requires **Global Admin** or **Namespace Admin** role. <!-- docs/cloud/capacity-modes.mdx:179 -->

3. Verify the change took effect:

   ```bash
   tcld namespace get \
       --namespace <namespace_name>.<account_suffix>
   ```
   <!-- docs/cloud/tcld/namespace.mdx:466-468 -->

   Confirm the capacity mode is `provisioned` and the TRU count matches your request.

4. When the spike is over, switch back to On-Demand:

   ```bash
   tcld namespace capacity update \
       --namespace <namespace_name>.<account_suffix> \
       --capacity-mode on_demand
   ```
   <!-- docs/cloud/capacity-modes.mdx:208 -->

   When switching back to On-Demand mode, your APS limit resets to the running average from the last 7 days. Plan for this if your workload is sensitive to the transition. <!-- docs/best-practices/managing-aps-limits.mdx:205-207 -->

---

## (c) Find and triage all hung workflows in a namespace

**When to use:** You suspect workflows are stuck and need to locate them and understand why they are not making progress.

### Step 1: Count potentially stuck workflows

```bash
temporal workflow count \
    --query "ExecutionStatus = 'Running' AND StartTime < '2024-01-15T09:00:00Z'"
```
<!-- docs/cli/workflow.mdx:88-96, docs/encyclopedia/visibility/search-attributes.mdx:87 -->

Replace the timestamp with your threshold for "too long". A count greater than zero indicates workflows that have been running longer than expected.

### Step 2: List the stuck workflows

```bash
temporal workflow list \
    --query "ExecutionStatus = 'Running' AND StartTime < '2024-01-15T09:00:00Z'" \
    --limit 20
```
<!-- docs/cli/workflow.mdx:280-286, docs/encyclopedia/visibility/list-filter.mdx:174 -->

You can narrow further by Task Queue or Workflow Type:

```bash
temporal workflow list \
    --query "ExecutionStatus = 'Running' AND TaskQueue = 'my-task-queue' AND StartTime < '2024-01-15T09:00:00Z'"
```
<!-- docs/encyclopedia/visibility/search-attributes.mdx:89 -->

### Step 3: Check worker health on the relevant Task Queue

```bash
temporal task-queue describe \
    --task-queue my-task-queue
```
<!-- docs/cli/task-queue.mdx:106-109 -->

Look for: active pollers present, `LastAccessTime` within the last minute, no growing `ApproximateBacklogCount`. <!-- docs/cli/task-queue.mdx:125-149 -->

If there are no pollers, no Workers are running for this Task Queue and workflows on this queue cannot make progress.

### Step 4: Inspect individual stuck workflows

```bash
temporal workflow describe --workflow-id <id>
```
<!-- docs/cli/workflow.mdx:133-140 -->

```bash
temporal workflow show --workflow-id <id> --reverse
```
<!-- docs/cli/workflow.mdx:422-431 -->

```bash
temporal workflow stack --workflow-id <id>
```
<!-- docs/cli/workflow.mdx:527-533 -->

### Step 5: Diagnose root cause

For diagnosing *why* a specific workflow is stuck (pending activities, pending child workflows, non-determinism, etc.), follow the triage procedures in `workflow-stuck.md`.

---

## (d) Rotate an API key without downtime

**When to use:** An API key is approaching expiration or needs to be rotated for security hygiene.

1. Create a new API key (for a user):

   ```bash
   tcld apikey create --name <name> \
       --description "<description>" \
       --duration <duration>
   ```
   <!-- docs/cloud/tcld/apikey.mdx:30-102 -->

   Or for a Service Account:

   ```bash
   tcld apikey create \
       --name <name> \
       --description "<description>" \
       --duration <duration> \
       --service-account-id <service-account-id>
   ```
   <!-- docs/cloud/get-started/api-keys.mdx:258-267 -->

   You may reuse key names. <!-- docs/cloud/get-started/api-keys.mdx:219-225 -->

   Save the returned key secret -- it is only shown once.

2. Verify both the original and new key function properly:

   ```bash
   temporal workflow list \
       --address <namespace>.<account>.tmprl.cloud:7233 \
       --namespace <namespace_id>.<account_id>
   ```
   <!-- docs/cloud/get-started/api-keys.mdx:364-389 -->

   Set `TEMPORAL_API_KEY` to each key in turn and confirm the command succeeds.

3. Update clients and workers to load the new key. <!-- docs/cloud/get-started/api-keys.mdx:219-225 -->

4. Once no traffic uses the old key, delete it:

   ```bash
   tcld apikey delete --id <old_apikey_id>
   ```
   <!-- docs/cloud/tcld/apikey.mdx:144-188 -->

   Alternatively, disable before deleting to validate nothing breaks:

   ```bash
   tcld apikey disable --id <old_apikey_id>
   ```
   <!-- docs/cloud/tcld/apikey.mdx:192-234 -->

**Limits:** Up to 10 non-expired keys per user; up to 20 non-expired keys per Service Account. Maximum expiration: 2 years. <!-- docs/cloud/get-started/api-keys.mdx:440-449 -->

---

## (e) Audit namespace access (users + keys + service accounts)

**When to use:** You need a complete picture of who and what can access a namespace -- humans, API keys, and service accounts.

### Step 1: List all users with access to the namespace

```bash
tcld user list --namespace <namespace_name>.<account_suffix>
```
<!-- docs/cloud/tcld/user.mdx:154-160, docs/cloud/tcld/user.mdx:168-170 -->

This filters to users with permissions on the specified namespace.

### Step 2: Inspect individual user permissions

```bash
tcld user get --user-email <email>
```
<!-- docs/cloud/tcld/user.mdx:74-100 -->

Check the account role (`admin`, `developer`, `read`) and namespace-level permissions (`Admin`, `Write`, `Read`). <!-- docs/cloud/tcld/user.mdx:125, 137 -->

### Step 3: List all user groups

```bash
tcld user-group list
```
<!-- docs/cloud/tcld/user-group.mdx:94-105 -->

For each group with namespace access, list its members:

```bash
tcld user-group list-members --group-id <id>
```
<!-- docs/cloud/tcld/user-group.mdx:108-120 -->

### Step 4: List all service accounts

```bash
tcld service-account list
```
<!-- docs/cloud/get-started/service-accounts.mdx:118-119 -->

Review the output for service accounts that have permissions on the target namespace. Namespace-scoped Service Accounts always have a `Read` Account Role and are restricted to a single namespace. <!-- docs/cloud/get-started/service-accounts.mdx:195-198 -->

### Step 5: List all API keys

```bash
tcld apikey list
```
<!-- docs/cloud/tcld/apikey.mdx:128-140 -->

Cross-reference the API key owners (user IDs or service account IDs) against the users and service accounts identified above.

---

## (f) Set up a new Cloud namespace with API key auth (end-to-end)

**When to use:** Provisioning a new Temporal Cloud namespace from scratch, using API key authentication.

### Step 1: Create the namespace

```bash
tcld namespace create \
    --namespace <namespace_name>.<account_suffix> \
    --region <region> \
    --auth-method api_key \
    --retention-days 30
```
<!-- docs/cloud/tcld/namespace.mdx:129-134, docs/cloud/tcld/namespace.mdx:122, docs/cloud/tcld/namespace.mdx:233 -->

Requires Developer, Account Owner, or Global Admin account-level role. <!-- docs/cloud/get-started/namespaces.mdx:178 -->
The creator is automatically granted Namespace Admin permission. <!-- docs/cloud/get-started/namespaces.mdx:176 -->

Optional flags:
- `--search-attribute "name=type"` (types: `Bool`, `Datetime`, `Double`, `Int`, `Keyword`, `Text`). <!-- docs/cloud/tcld/namespace.mdx:240-241 -->
- `--tag "key=value"` (up to 10 tags per namespace). <!-- docs/cloud/get-started/namespaces.mdx:480 -->
- `--user-namespace-permission "email=permission"` (permissions: `Admin`, `Write`, `Read`). <!-- docs/cloud/tcld/namespace.mdx:277-279 -->

### Step 2: Create a service account for Workers

```bash
tcld service-account create -n "<name>" -d "<description>" --ar "developer" \
    --np "<namespace_name>.<account_suffix>=Write"
```
<!-- docs/cloud/get-started/service-accounts.mdx:88-89 -->

Note the returned `ServiceAccountId`. <!-- docs/cloud/get-started/service-accounts.mdx:96-97 -->

### Step 3: Create an API key for the service account

```bash
tcld apikey create \
    --name <key_name> \
    --description "<description>" \
    --duration <duration> \
    --service-account-id <service-account-id>
```
<!-- docs/cloud/get-started/api-keys.mdx:258-267 -->

Save the returned key secret.

### Step 4: Verify connectivity

Set the API key and test with the Temporal CLI:

```bash
export TEMPORAL_API_KEY=<key-secret>
temporal workflow list \
    --address <namespace_name>.<account_suffix>.tmprl.cloud:7233 \
    --namespace <namespace_name>.<account_suffix>
```
<!-- docs/cloud/get-started/api-keys.mdx:364-389 -->

### Step 5: (Optional) Grant additional user access

```bash
tcld user set-namespace-permissions \
    --user-email <email> \
    --namespace-permission <namespace_name>.<account_suffix>=<permission>
```
<!-- docs/cloud/tcld/user.mdx:287-340 -->

Permissions: `Admin`, `Write`, `Read`. <!-- docs/cloud/tcld/user.mdx:331-338 -->

### Step 6: (Optional) Enable delete protection

```bash
tcld namespace lifecycle set \
    --namespace <namespace_name>.<account_suffix> \
    --enable-delete-protection true
```
<!-- docs/cloud/get-started/namespaces.mdx:468-471 -->

---

## (g) Rotate mTLS certificates

**When to use:** A CA certificate is approaching expiration, or you need to switch to a new CA without disrupting running Workers.

Temporal Cloud sends email notifications 15 days before certificate expiration. <!-- docs/cloud/get-started/certificates.mdx:337 -->

### Step 1: Generate a new CA certificate

```bash
tcld generate-certificates certificate-authority-certificate \
    --organization <value> \
    --validity-period <duration> \
    --ca-certificate-file <new_ca>.pem \
    --ca-key-file <new_ca>.key
```
<!-- docs/cloud/tcld/generate-certificates.mdx:24-81 -->

Default key algorithm is ECDSA P-384. Maximum duration: 1 year. <!-- docs/cloud/tcld/generate-certificates.mdx:85, docs/cloud/get-started/certificates.mdx:129-130 -->

### Step 2: Generate new end-entity (leaf) certificates

```bash
tcld generate-certificates end-entity-certificate \
    --organization <value> \
    --validity-period <duration> \
    --ca-certificate-file <new_ca>.pem \
    --ca-key-file <new_ca>.key \
    --certificate-file <new_client>.pem \
    --key-file <new_client>.key
```
<!-- docs/cloud/tcld/generate-certificates.mdx:96-187 -->

End-entity certificate must expire before its root CA certificate. <!-- docs/cloud/get-started/certificates.mdx:130-131 -->

### Step 3: Create a combined PEM bundle with old and new CA certificates

Concatenate both CA certificates into a single PEM file: <!-- docs/cloud/get-started/certificates.mdx:378-388 -->

```
-----BEGIN CERTIFICATE-----
... old CA cert ...
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
... new CA cert ...
-----END CERTIFICATE-----
```

### Step 4: Upload the combined bundle (replaces all existing CAs)

```bash
tcld namespace accepted-client-ca set \
    --namespace <namespace_name>.<account_suffix> \
    --ca-certificate-file <combined>.pem
```
<!-- docs/cloud/get-started/certificates.mdx:390-393 -->

Both old and new end-entity certificates will now be accepted.

### Step 5: Roll out new end-entity certificates to Workers and Clients

Deploy the new leaf certificates to all Workers and Clients. Monitor traffic to the old certificate until it ceases. <!-- docs/cloud/get-started/certificates.mdx:395 -->

### Step 6: Remove the old CA certificate

Create a file containing only the new CA certificate and run `set` again: <!-- docs/cloud/get-started/certificates.mdx:397-399 -->

```bash
tcld namespace accepted-client-ca set \
    --namespace <namespace_name>.<account_suffix> \
    --ca-certificate-file <new_ca_only>.pem
```

### Step 7: Verify the namespace only has the new CA

```bash
tcld namespace accepted-client-ca list \
    --namespace <namespace_name>.<account_suffix>
```
<!-- docs/cloud/tcld/namespace.mdx:868-890 -->

Do NOT use a CA certificate signed with SHA-1 -- such signatures are rejected. <!-- docs/cloud/tcld/namespace.mdx:772-776 -->

---

## (h) Check self-hosted cluster health

**When to use:** You want to verify that a self-hosted Temporal cluster is operational and inspect its configuration.

### Step 1: Check cluster health

```bash
temporal operator cluster health
```
<!-- docs/cli/operator.mdx:77-79 -->

Returns health status of the Temporal Service. Connects to `--address` (default `localhost:7233`). <!-- docs/cli/operator.mdx:526 -->

### Step 2: Describe the cluster

```bash
temporal operator cluster describe --detail
```
<!-- docs/cli/operator.mdx:59-67, docs/cli/operator.mdx:72 -->

Shows Cluster Name, persistence store, visibility store, history shard count, and version information.

### Step 3: Get system info

```bash
temporal operator cluster system
```
<!-- docs/cli/operator.mdx:115-129 -->

Shows Server version, scheduling support, and more.

### Step 4: List namespaces

```bash
temporal operator namespace list
```
<!-- docs/cli/operator.mdx:261-265 -->

Displays a detailed listing for all Namespaces on the Service.

### Step 5: Check worker health on key Task Queues

```bash
temporal task-queue describe \
    --task-queue <task_queue_name>
```
<!-- docs/cli/task-queue.mdx:106-109 -->

```bash
temporal task-queue describe \
    --task-queue <task_queue_name> \
    --task-queue-type activity
```
<!-- docs/cli/task-queue.mdx:117-123 -->

Look for: active pollers present, `LastAccessTime` within the last minute, no growing `ApproximateBacklogCount` or positive `BacklogIncreaseRate`. <!-- docs/cli/task-queue.mdx:125-149 -->

### Step 6: Spot-check for stuck workflows

```bash
temporal workflow count \
    --query "ExecutionStatus = 'Running' AND StartTime < '2024-01-15T09:00:00Z'"
```
<!-- docs/cli/workflow.mdx:88-96, docs/encyclopedia/visibility/search-attributes.mdx:87 -->

If the count is unexpectedly high, follow playbook (c) above to investigate.

### Connecting to a remote cluster

For any of these commands, specify the target address:

```bash
temporal operator cluster health \
    --address <host>:<port>
```
<!-- docs/cli/operator.mdx:526 -->

If TLS is enabled:

```bash
temporal operator cluster health \
    --address <host>:<port> \
    --tls \
    --tls-ca-path <path_to_ca> \
    --tls-cert-path <path_to_cert> \
    --tls-key-path <path_to_key>
```
<!-- docs/cli/operator.mdx:549-558 -->
