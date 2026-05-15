---
name: temporal-ops
description: Administer and diagnose running Temporal Cloud or self-hosted Temporal Server environments via CLI commands (temporal, tcld, openssl) — not SDK code. Operations: Temporal namespace CRUD, Temporal Cloud capacity/APS, Temporal API-key rotation, Temporal mTLS certificate rotation, workflow health queries, batch cancel/terminate/reset, export setup, search attributes, Cloud Ops API, billing and cost attribution, audit logs, Terraform provider, notifications, SAML SSO, SCIM provisioning. Diagnosis: bottom-up triage of Temporal-specific failures including stuck workflows, non-determinism errors, worker-health issues, task-queue problems, Temporal Cloud HA failover, payload/gRPC size limits, performance bottlenecks, and missed scheduled actions. Use when the user is working with Temporal infrastructure and mentions: "create a Temporal namespace", "check my APS", "rotate Temporal API key", "rotate Temporal certificates", "find hung workflows", "cancel workflows in batch", "my workflow is stuck", "can't connect to Temporal", "task queue has no pollers", "workers not picking up tasks", "set up PrivateLink", "Temporal RESOURCE_EXHAUSTED", "workflow replay failed", "Temporal billing", "audit logs", "Terraform Temporal", "SAML Temporal", "SCIM Temporal", "payload too large", "schedule missed", "performance bottleneck". Do NOT trigger for generic TLS/cert errors, generic gRPC errors (context deadline exceeded), or API key rotation unrelated to Temporal. Does NOT write or fix application code (temporal-developer) or explain CLI flags (temporal-cli).
version: 0.2.0
---

# Skill: temporal-ops

## Overview

This skill operates and diagnoses Temporal environments. It has two modes:

- **Operations:** the user wants to do something — create a namespace, rotate a key, check capacity, find unhealthy workflows, cancel a batch, set up export. The skill executes the right commands and interprets the output.
- **Diagnosis:** the user arrives with a symptom — a stuck workflow, a cert error, a connection timeout, a non-determinism panic. The skill routes the investigation through a layered, bottom-up diagnosis until a root cause is identified with a confidence score.

It does not teach how to write workflows or activities (use `skill-temporal-developer` for that), and it does not deep-dive into CLI flag semantics (use `skill-temporal-cli` for that). The boundary is: if the user needs to administer or troubleshoot a running Temporal environment, this skill applies.

## Out of scope

- **Writing workflows, activities, or SDK code** → `skill-temporal-developer`.
- **CLI command reference, flag semantics** → `skill-temporal-cli`.
- **Worker performance tuning, sizing, capacity planning** → `skill-temporal-workertuning`.
- **Helm, Kubernetes, database admin, monitoring stack config** for self-hosted — beyond the CLI surface.

If the conversation drifts into one of these areas, hand off to the relevant sibling skill rather than improvising.

## Philosophy

### Operator discipline

When the user wants to perform an operational task:

1. **Identify the intent and backend.** Is this a Cloud operation (`tcld`) or a self-hosted operation (`temporal operator`)? Data-plane operations (`temporal workflow`, `temporal batch`, etc.) work on both. **If the backend is ambiguous, ask before proceeding — do not assume Cloud or self-hosted and do not output environment-specific commands until you know.**
2. **Execute commands and interpret output.** Run the documented command, read the result, and report what it means — or act on it if the user asked for an action.
3. **Verify the result.** After a mutating operation, confirm the new state matches the user's intent.

### Diagnostic discipline

When the user arrives with a failure or anomaly:

1. **Bottom-up diagnosis.** Verify the lower layer before blaming the upper one. The layers, from bottom to top:
   1. DNS / network path
   2. TCP / port reachability
   3. TLS handshake
   4. Authentication (API key or mTLS client cert)
   5. gRPC health and Temporal frontend reachability
   6. Temporal namespace, task queues, workers
   7. Workflow code (determinism, signals, timers, child workflows)

   The full ladder lives in [diagnostic-ladder.md](references/triage/diagnostic-ladder.md).

2. **Always verify the next layer up** rather than prescribing a speculative fix. If TLS works, prove auth works before blaming the workflow. If pollers are present, prove the workflow's last event before blaming the worker.

3. **Attach a confidence score** (1-10) to every proposed diagnosis:
   - 9-10: symptoms, operation, and confirming signals line up cleanly.
   - 6-8: evidence is good but at least one alternative remains plausible.
   - 1-5: the issue is still ambiguous; the "fix" is the next discriminating check, not a root cause.

4. **Name ambiguity explicitly.** Errors like `context deadline exceeded` are not self-describing, and strings like "workflow is busy" are caller observations rather than documented contracts. Surface that, gather more context, and scope the next step narrowly.

These are skill conventions, not docs-derived facts.

## Intent routing

### Operations

Find the row that matches the user's intent. The reference file contains the commands and procedures.

| Intent | Category | Reference |
|---|---|---|
| Create, get, list, delete a Cloud namespace | Cloud namespace admin | [cloud-namespace-admin.md](references/ops/cloud-namespace-admin.md) |
| Add/remove region, failover, HA config | Cloud namespace admin | [cloud-namespace-admin.md](references/ops/cloud-namespace-admin.md) |
| Set retention, tags, codec-server, connectivity rules | Cloud namespace admin | [cloud-namespace-admin.md](references/ops/cloud-namespace-admin.md) |
| Add or rename Cloud search attributes | Cloud namespace admin | [cloud-namespace-admin.md](references/ops/cloud-namespace-admin.md) |
| Check current APS / capacity mode | Cloud capacity | [cloud-capacity.md](references/ops/cloud-capacity.md) |
| Switch On-Demand ↔ Provisioned, set TRUs | Cloud capacity | [cloud-capacity.md](references/ops/cloud-capacity.md) |
| Understand APS / RPS / OPS limits and throttling | Cloud capacity | [cloud-capacity.md](references/ops/cloud-capacity.md) |
| Create, disable, enable, delete an API key | Cloud IAM | [cloud-iam.md](references/ops/cloud-iam.md) |
| Invite, list, remove users; set roles/permissions | Cloud IAM | [cloud-iam.md](references/ops/cloud-iam.md) |
| Manage user groups and service accounts | Cloud IAM | [cloud-iam.md](references/ops/cloud-iam.md) |
| Generate mTLS certs, upload CA, set cert filters | Cloud certs | [cloud-certs.md](references/ops/cloud-certs.md) |
| Rotate mTLS certificates | Cloud certs | [cloud-certs.md](references/ops/cloud-certs.md) |
| Set up Workflow History Export (S3 / GCS) | Cloud export & connectivity | [cloud-export-and-connectivity.md](references/ops/cloud-export-and-connectivity.md) |
| Set up PrivateLink / PSC, manage connectivity rules | Cloud export & connectivity | [cloud-export-and-connectivity.md](references/ops/cloud-export-and-connectivity.md) |
| Cloud Ops API access | Cloud export & connectivity | [cloud-export-and-connectivity.md](references/ops/cloud-export-and-connectivity.md) |
| Self-hosted cluster health, describe, namespace CRUD | Self-hosted admin | [self-hosted-admin.md](references/ops/self-hosted-admin.md) |
| Self-hosted search attributes, Nexus endpoints | Self-hosted admin | [self-hosted-admin.md](references/ops/self-hosted-admin.md) |
| Find stuck/hung/unhealthy workflows via list queries | Workflow health | [workflow-health.md](references/ops/workflow-health.md) |
| Task queue poller status, workflow counts | Workflow health | [workflow-health.md](references/ops/workflow-health.md) |
| Cancel, terminate, or reset workflows | Batch & lifecycle | [batch-and-lifecycle.md](references/ops/batch-and-lifecycle.md) |
| Batch operations on workflows | Batch & lifecycle | [batch-and-lifecycle.md](references/ops/batch-and-lifecycle.md) |
| Schedule CRUD and operations | Batch & lifecycle | [batch-and-lifecycle.md](references/ops/batch-and-lifecycle.md) |
| Complete or fail an activity externally | Batch & lifecycle | [batch-and-lifecycle.md](references/ops/batch-and-lifecycle.md) |
| Cloud Ops API access, rate limits, Go SDK | Cloud Ops API | [cloud-ops-api.md](references/ops/cloud-ops-api.md) |
| View billing, generate billing report, cost attribution | Cloud billing | [cloud-billing.md](references/ops/cloud-billing.md) |
| Audit Logs: view, query via API, configure sink (AWS/GCP) | Cloud audit logs | [cloud-audit-logs.md](references/ops/cloud-audit-logs.md) |
| Terraform provider: Namespace/User/SA/API Key/Nexus CRUD | Cloud Terraform | [cloud-terraform.md](references/ops/cloud-terraform.md) |
| Expiry alerts (cert, API key, credit), status page | Cloud notifications | [cloud-notifications.md](references/ops/cloud-notifications.md) |
| SAML SSO, SCIM provisioning, IdP integration | Cloud SAML/SCIM | [cloud-saml-scim.md](references/ops/cloud-saml-scim.md) |
| End-to-end ops playbook (setup, rotation, audit, billing, Terraform) | Ops recipes | [ops/recipes.md](references/ops/recipes.md) |

### Diagnosis

Find the row that matches the user's symptom. Start the investigation at the first check, then read the linked reference.

| Symptom | Category | First check | Reference |
|---|---|---|---|
| `connection refused`, cannot reach frontend | Connectivity | `nc -zvw10 <host> 7233` | [connectivity.md#connection-refused](references/triage/connectivity.md#connection-refused) |
| `no such host`, DNS resolution fails | Connectivity | `dig +short <host>` or `nslookup <host>` | [connectivity.md#dns](references/triage/connectivity.md#dns) |
| `tls: handshake failure` <!-- go: crypto/tls -->, server rejects handshake | Certificates | `openssl s_client -connect <host>:7233 -servername <host> </dev/null` | [certificates.md#handshake-failure](references/triage/certificates.md#handshake-failure) |
| `x509: certificate has expired` or `not yet valid` <!-- go: crypto/x509 --> | Certificates | `openssl x509 -enddate -noout -in cert.pem` | [certificates.md#expired-or-not-yet-valid](references/triage/certificates.md#expired-or-not-yet-valid) |
| `x509: certificate signed by unknown authority` <!-- go: crypto/x509 --> | Certificates | `openssl verify -CAfile ca.pem client.pem` | [certificates.md#unknown-authority](references/triage/certificates.md#unknown-authority) |
| `tcld` session / auth fails, Cloud role unclear | Authentication | `tcld account get` | [authentication.md#cloud-role-and-permission-model](references/triage/authentication.md#cloud-role-and-permission-model) |
| `UNAUTHENTICATED`, API key rejected | Authentication | `env \| grep -i TEMPORAL_API_KEY`, then `tcld apikey get --id <apikey_id>` | [authentication.md#things-to-check-when-unauthenticated-is-returned-with-an-api-key](references/triage/authentication.md#things-to-check-when-unauthenticated-is-returned-with-an-api-key) |
| `namespace not found` / wrong namespace string with an API key | Authentication | Confirm Regional Endpoint form `<region>.<cloud_provider>.api.temporal.io:7233` | [authentication.md#required-address-form-for-api-key-connections](references/triage/authentication.md#required-address-form-for-api-key-connections) |
| `RESOURCE_EXHAUSTED` gRPC status | Rate limits | Identify which limit fired via the `resource_exhausted_cause` metric label | [rate-limits.md#identifying-which-limit-was-hit](references/triage/rate-limits.md#identifying-which-limit-was-hit) |
| Task queue shows no pollers | Worker health | `temporal task-queue describe --task-queue <q>` | [worker-health.md#what-no-pollers-looks-like](references/triage/worker-health.md#what-no-pollers-looks-like) |
| Workflow stuck on a pending activity / timer / child / signal | Workflow stuck | `temporal workflow describe --workflow-id <id>` | [workflow-stuck.md#the-primary-inspection-command-temporal-workflow-describe](references/triage/workflow-stuck.md#the-primary-inspection-command-temporal-workflow-describe) |
| `NondeterminismError`, repeating `WorkflowTaskFailed` | Non-determinism | Identify the last `WorkflowTaskFailed` cause in the Event History | [non-determinism.md#the-wft-failure-signature-of-non-determinism](references/triage/non-determinism.md#the-wft-failure-signature-of-non-determinism) |
| Replay fails locally but prod workflow was running | Non-determinism | Export history and run the SDK replayer | [replay-with-vscode.md#step-2a--sdk-replayer-all-supported-sdks-ci-friendly](references/triage/replay-with-vscode.md#step-2a--sdk-replayer-all-supported-sdks-ci-friendly) |
| HA failover did not route traffic to failover region | HA failover | `tcld namespace get --namespace <ns>.<acct>` vs. DNS CNAME | [ha-failover.md#verify-the-current-active-region](references/triage/ha-failover.md#verify-the-current-active-region) |
| `context deadline exceeded` (unknown layer) | Runtime errors | Identify which operation and SDK emitted it | [runtime-errors.md#deadline-exceeded](references/triage/runtime-errors.md#deadline-exceeded) |
| Caller reports "workflow is busy" / `RESOURCE_EXHAUSTED` on signal/update/query to one Workflow | Runtime errors | Classify via `resource_exhausted_cause`, not by the message text | [runtime-errors.md#workflow-busy-backpressure](references/triage/runtime-errors.md#workflow-busy-backpressure) |
| `PAYLOADS_TOO_LARGE`, `exceeds size limit`, payload/gRPC blob size error | Blob size limits | Check whether the issue is payload (2 MB) or gRPC message (4 MB) | [blob-size-limits.md](references/triage/blob-size-limits.md) |
| Workflow stuck in invisible retry loop (gRPC message too large) | Blob size limits | Check Worker logs for `ResourceExhausted`, reduce batch size | [blob-size-limits.md](references/triage/blob-size-limits.md) |
| High schedule-to-start latency, task slot depletion, slow Workflow Tasks | Performance bottlenecks | Check `temporal_workflow_task_schedule_to_start_latency` P95 | [performance-bottlenecks.md](references/triage/performance-bottlenecks.md) |
| High replay latency, cache evictions, deadlock detected | Performance bottlenecks | Check `workflow_task_replay_latency` and sticky cache metrics | [performance-bottlenecks.md](references/triage/performance-bottlenecks.md) |
| Schedule did not fire, missed catchup window | Missed Schedule Actions | Alert on `temporal_cloud_v1_schedule_missed_catchup_window_count` | [schedule-missed.md](references/triage/schedule-missed.md) |

If a symptom does not map to a row, start at [diagnostic-ladder.md](references/triage/diagnostic-ladder.md) and work up from whichever layer was last known healthy.

## The process

### Operations path

#### Step 1: Identify intent and backend

Determine what the user wants to do and whether it targets:
- **Temporal Cloud** → use `tcld` commands (requires `tcld login`)
- **Self-hosted cluster** → use `temporal operator` commands
- **Data plane (either backend)** → use `temporal workflow`, `temporal batch`, `temporal schedule`, etc.

If the backend is unambiguous from context — `.tmprl.cloud` address, `tcld` command, Cloud namespace format `ns.account` → Cloud; Kubernetes/Helm, `docker-compose`, self-hosted config files → self-hosted — proceed without asking. **Otherwise, stop and ask: "Are you on Temporal Cloud or self-hosted?" before outputting any environment-specific commands.** Do not default to either environment. Once known, save the answer to memory so you don't ask again in future conversations.

#### Step 2: Execute and interpret

Look up the intent in the Operations table above. Read the linked reference file for the exact commands, flags, and expected output. Run the command and interpret the result for the user.

#### Step 3: Verify

After a mutating operation (create, update, delete, rotate), confirm the new state:
- Re-run the corresponding `get` or `describe` command
- Confirm the output matches the user's intent
- Report the result

### Diagnosis path

#### Step 1: Identify the symptom

Ask the user for the exact, copy-pasted error text. Do not accept paraphrases — the exact string often encodes the layer (e.g., `x509:` prefix means TLS/cert layer, `RESOURCE_EXHAUSTED:` prefix means gRPC rate limit, `NondeterminismError` means workflow replay layer). Note that some user-reported phrases (e.g. "workflow is busy") are field observations rather than server-contracted strings; the gRPC code and any `resource_exhausted_cause` label are more reliable signals than the free-text message.

Confirm three things before continuing:
- What command was run, or what SDK call produced the error?
- What environment produced it (local dev server, self-hosted cluster, Temporal Cloud)? If clear from context (addresses, commands, namespace format), don't ask — **but if uncertain, ask now before proceeding with any diagnosis.** Save the answer to memory for future conversations.
- What changed recently (new deploy, new certs, new namespace, new region)?

#### Step 2: Gather context

The context the investigation needs depends on the category. At minimum:

- **For any Cloud auth / connectivity issue:** auth method (API key vs mTLS), exact address, exact namespace, SDK + version. The endpoint family differs by auth method — see [connectivity.md#endpoint-formats](references/triage/connectivity.md#endpoint-formats).
- **For a stuck workflow:** namespace, workflow ID, run ID, and the output of `temporal workflow describe --workflow-id <id>` (pending-operation state lives here, not in the Event History alone). Event History via `temporal workflow show` is the companion view.
- **For a worker health issue:** worker logs (registration errors, auth errors, panics), and the output of `temporal task-queue describe --task-queue <q>`.
- **For a non-determinism error:** the worker log line containing the error, the workflow type name, and access to the history JSON for replay.

#### Step 3: Validate pasted SDK config (if any)

If the user has pasted SDK connection code — even just the address/namespace/auth fields — review it against [sdk-snippet-review.md](references/triage/sdk-snippet-review.md) **before** descending the ladder. Wrong endpoint family, short namespace, or mismatched auth method will make every network-layer probe below look broken when nothing lower actually is.

Skip this step when the user has an established, previously-working config and the symptom is new — the snippet is not the culprit, the environment changed. Otherwise treat snippet validation as Layer 0.

#### Step 4: Descend the ladder

Use [diagnostic-ladder.md](references/triage/diagnostic-ladder.md) to pick the right starting layer. As a rule of thumb:

- Auth / connectivity / cert symptom → start at layer 1 (DNS) and walk up.
- Worker / task-queue symptom → start at layer 6 (namespace + pollers).
- Stuck workflow / determinism symptom → start at layer 7 (workflow code), but confirm layer 6 (worker is actually polling) first.

Each layer has a command that proves it healthy and a failure signature that tells you whether the problem lives at that layer or higher.

#### Step 5: Fix and verify

Prescribe the fix scoped to the root cause. Then verify by re-running the layer's healthy-check command and, if possible, the original user operation. Attach the confidence score to the diagnosis.

If the layer above the fix is still failing, return to step 4 and continue walking upward — the first broken layer is rarely the only one.

## Prerequisites

- **Temporal CLI** (`temporal`) — required for data-plane operations and self-hosted admin. Install: `brew install temporal` or see [Temporal CLI docs](https://docs.temporal.io/cli).
- **tcld** — required for Cloud operations. Install: `brew install temporal-cloud-cli` or see [tcld docs](https://docs.temporal.io/cloud/tcld). Authenticate with `tcld login` before use.

## Reference files

### Operations

- [cloud-namespace-admin.md](references/ops/cloud-namespace-admin.md) — Cloud namespace lifecycle via `tcld`: create, get, list, delete, failover, add-region, retention, tags, codec-server, HA config, connectivity rules, search attributes, accepted-client-ca, certificate filters, export.
- [cloud-capacity.md](references/ops/cloud-capacity.md) — Capacity modes (On-Demand / Provisioned), APS/RPS/OPS definitions, TRUs, `tcld namespace capacity update`, default limits, throttling, APS management best practices.
- [cloud-iam.md](references/ops/cloud-iam.md) — API key lifecycle (`tcld apikey`), users (`tcld user`), user groups (`tcld user-group`), service accounts, account operations (`tcld account`), roles, namespace permissions.
- [cloud-certs.md](references/ops/cloud-certs.md) — mTLS cert management: generating certs with `tcld generate-certificates`, uploading CAs, certificate filters, cert rotation, switching mTLS ↔ API keys.
- [cloud-export-and-connectivity.md](references/ops/cloud-export-and-connectivity.md) — Workflow History Export to S3/GCS, private connectivity (PrivateLink/PSC), connectivity rules. Cloud Ops API quick reference (pointer to standalone file).
- [cloud-ops-api.md](references/ops/cloud-ops-api.md) — Cloud Ops API: HTTP and gRPC endpoints (`saas-api.tmprl.cloud`), Go SDK, protobuf compilation, rate limits (160 RPS account, 40 user, 80 SA, 10 concurrent async), API version header, use cases.
- [cloud-billing.md](references/ops/cloud-billing.md) — Cloud billing: Billing Center (invoices, credits, plans, cost by namespace), Usage Dashboards, Billing API (Public Preview): async CSV report generation, FOCUS-friendly format, 26-column report schema, date range constraints.
- [cloud-audit-logs.md](references/ops/cloud-audit-logs.md) — Cloud Audit Logs: supported control plane events (Account, API Keys, Connectivity Rules, Namespace, Export, Nexus, Service Accounts, User, User Groups), JSON format, API access (30-day retention), AWS Kinesis and GCP Pub/Sub sink configuration.
- [cloud-terraform.md](references/ops/cloud-terraform.md) — Terraform provider: setup (`TEMPORAL_CLOUD_API_KEY`), `temporalcloud_namespace`/`temporalcloud_user`/`temporalcloud_service_account`/`temporalcloud_apikey`/`temporalcloud_nexus_endpoint` CRUD, import, data sources (regions, namespaces), limitations (API keys not importable, cannot manage Account Owner).
- [cloud-notifications.md](references/ops/cloud-notifications.md) — Cloud notifications: certificate expiry (15/10/5 days), API key expiry (30/20/10 days), credit consumption/expiry alerts, plan changes, failover events, recipient roles, `noreply@temporal.io` sender.
- [cloud-saml-scim.md](references/ops/cloud-saml-scim.md) — SAML SSO (Entra ID, Okta): entity identifier (`urn:auth0:prod-tmprl:ACCOUNT_ID-saml`), callback URL (`login.tmprl.cloud`), IdP configuration steps, support ticket workflow. SCIM: supported vendors, prerequisites (SAML first), 10-minute sync window, group-to-role mapping.
- [self-hosted-admin.md](references/ops/self-hosted-admin.md) — Self-hosted control plane via `temporal operator`: cluster health/describe, namespace CRUD, search-attribute create/list/remove, Nexus endpoint CRUD.
- [workflow-health.md](references/ops/workflow-health.md) — Data-plane health queries: `temporal workflow list` with List Filters, `temporal workflow describe`/`show`/`count`, `temporal task-queue describe` for poller status.
- [batch-and-lifecycle.md](references/ops/batch-and-lifecycle.md) — Bulk and lifecycle operations: `temporal workflow cancel/terminate/reset`, `temporal batch`, `temporal schedule`, `temporal activity complete/fail`.
- [ops/recipes.md](references/ops/recipes.md) — End-to-end ops playbooks: check APS, switch capacity mode, find hung workflows, rotate API key, audit access, set up new namespace, rotate mTLS certs, check self-hosted health, view billing / generate billing report, configure audit log sink, provision resources with Terraform, set up SAML SSO.

### Diagnosis

- [sdk-snippet-review.md](references/triage/sdk-snippet-review.md) — Layer-0 config check for pasted SDK connection snippets: endpoint form per auth method, namespace format, auth / TLS expectations, `TEMPORAL_*` env vars, common misconfigurations. Run before the diagnostic ladder.
- [diagnostic-ladder.md](references/triage/diagnostic-ladder.md) — the seven-layer bottom-up model, with one canonical command per layer and cross-links into the topical leaves.
- [connectivity.md](references/triage/connectivity.md) — DNS, TCP, endpoint families (Namespace Endpoint for mTLS vs. Regional Endpoint for API keys), firewall/proxy shapes, PrivateLink/PSC, quick diagnostic scripts.
- [certificates.md](references/triage/certificates.md) — x509 and TLS alert strings, expiry / unknown-authority / hostname-mismatch / key-mismatch diagnosis, Cloud accepted-client-CA set via `tcld namespace accepted-client-ca`, Cloud mTLS certificate requirements, rotation and expiry notifications, openssl recipes.
- [authentication.md](references/triage/authentication.md) — `UNAUTHENTICATED` vs `PERMISSION_DENIED`, API-key lifecycle (`tcld apikey` commands, env var propagation, required Regional Endpoint form), mTLS after TLS (certificate filters, identity-to-role mapping), Cloud account-level roles and namespace-level permissions.
- [workflow-stuck.md](references/triage/workflow-stuck.md) — Workflow Execution Status values, `temporal workflow describe` as the primary inspection command, Event History via `temporal workflow show`, pending activities / child workflows / signals / Nexus operations / Workflow Tasks, WorkflowTaskFailed retry loops, recovery commands (signal, terminate, cancel, reset, pause/unpause).
- [non-determinism.md](references/triage/non-determinism.md) — determinism definition, WFT-failure signature, ND-inducing code patterns, per-SDK error shapes, identifying ND from Event History, local replay reproduction, remediation via Worker Versioning / patching / reset.
- [worker-health.md](references/triage/worker-health.md) — no-pollers runbook via `temporal task-queue describe`, reachability and versioning, worker-level describe, schedule-to-start latency, worker task slots, sticky execution and sticky cache, worker heartbeating, Cloud namespace-level poller limits, worker log signatures.
- [rate-limits.md](references/triage/rate-limits.md) — what `RESOURCE_EXHAUSTED` means (and does not), Cloud APS / RPS / OPS under On-Demand and Provisioned capacity modes, self-hosted `frontend.rps` / `frontend.namespaceRPS` dynamic config, identifying which limit fired via the `resource_exhausted_cause` metric label.
- [ha-failover.md](references/triage/ha-failover.md) — Cloud HA routing via the Namespace Endpoint CNAME, verifying the active region (control-plane `tcld namespace get` vs. DNS view), clients that did not follow the failover, PrivateLink after failover, failover-not-executing, handover-window errors, platform limits, RPO/RTO semantics.
- [runtime-errors.md](references/triage/runtime-errors.md) — deadline-exceeded disambiguated by operation and by where the call was made, "workflow busy" backpressure framed via `resource_exhausted_cause` rather than a pinned string, routing for `no pollers` / `INVALID_ARGUMENT` / unspecified `UNAVAILABLE`.
- [replay-with-vscode.md](references/triage/replay-with-vscode.md) — exporting Event History, running the SDK replayer in any supported SDK, the TypeScript-only VS Code extension, `TEMPORAL_DEBUG` and the deadlock detector, interpreting divergent and successful replays.
- [blob-size-limits.md](references/triage/blob-size-limits.md) — Payload size limit (2 MB) and gRPC message size limit (4 MB): error messages, per-SDK behavior (Python 1.23.0+ vs. others), claim check pattern, External Storage (Pre-release), batch-size reduction.
- [performance-bottlenecks.md](references/triage/performance-bottlenecks.md) — Latency and throughput diagnosis via SDK metrics: schedule-to-start latency, workflow task execution latency, replay latency, activity execution latency, task slot depletion, network request metrics, sticky cache metrics.
- [schedule-missed.md](references/triage/schedule-missed.md) — Missed Schedule Actions: alerting via `temporal_cloud_v1_schedule_missed_catchup_window_count` / `schedule_missed_catchup_window`, investigation via `temporal schedule list` + `temporal schedule describe`, DescribeSchedule fields (`missedCatchupWindow`, `overlapSkipped`, `bufferDropped`), default catchup window (one year), root causes, overlap policies (6 values), backfill remediation.
- [recipes.md](references/triage/recipes.md) — four end-to-end triage walkthroughs: stuck workflow at 3am, cert expired with workers offline, task-queue backlog mystery, non-determinism caught in prod.

## Feedback

### Feedback prompt

When you have **first** loaded this skill, output this message to the user:

"Thank you for trying out the public preview of the Temporal Ops skill! We would love to hear your feedback - positive or negative - over in the [Community Slack](https://t.mp/slack), in the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY)."

Do not output this message multiple times in the same conversation.

### Reporting Issues in This Skill

If you (the AI) find this skill's explanations are unclear, misleading, or missing important information, draft a GitHub issue body describing the problem encountered and what would have helped, then ask the user to file it at https://github.com/temporalio/skill-temporal-ops/issues/new. Do not file the issue autonomously.
