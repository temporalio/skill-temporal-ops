---
name: temporal-triage
description: This skill should be used when the user reports a Temporal failure or anomaly — e.g. "my workflow is stuck", "non-determinism error", "can't connect to Temporal", "TLS handshake failed", "certificate expired", "RESOURCE_EXHAUSTED", "task queue has no pollers", "workflow busy backpressure", "context deadline exceeded", "replay failed", "HA failover isn't working", "tcld login failed", or mentions symptoms that suggest a stuck workflow, a connectivity / auth / cert issue, a worker health problem, a rate limit, or a replay failure.
version: 0.1.0
---

# Skill: temporal-triage

## Overview

This skill diagnoses Temporal failures and anomalies. The user arrives with a symptom — a stuck workflow, a cert error, a connection timeout, a non-determinism panic — and this skill routes the investigation through a layered, bottom-up diagnosis until a root cause is identified with a confidence score.

It does not teach which commands to run (use `skill-temporal-cli` for that) and it does not teach how to write workflows or activities (use `skill-temporal-developer` for that). The boundary is: if the command ran and the system is misbehaving, this skill applies.

## Philosophy

Temporal failures live at the intersection of infrastructure, authentication, gRPC, Temporal server internals, and user workflow code. A plausible error at one layer is often a symptom of a broken layer below it. Jumping to conclusions wastes time and frequently "fixes" the wrong thing.

The discipline this skill enforces (these four points are skill conventions, not docs-derived facts):

1. **Bottom-up diagnosis.** Verify the lower layer before blaming the upper one. The layers, from bottom to top:
   1. DNS / network path
   2. TCP / port reachability
   3. TLS handshake
   4. Authentication (API key or mTLS client cert)
   5. gRPC health and Temporal frontend reachability
   6. Temporal namespace, task queues, workers
   7. Workflow code (determinism, signals, timers, child workflows)

   The full ladder lives in [diagnostic-ladder.md](references/diagnostic-ladder.md).

2. **Always verify the next layer up** rather than prescribing a speculative fix. If TLS works, prove auth works before blaming the workflow. If pollers are present, prove the workflow's last event before blaming the worker.

3. **Attach a confidence score** (1-10) to every proposed diagnosis:
   - 9-10: symptoms, operation, and confirming signals line up cleanly.
   - 6-8: evidence is good but at least one alternative remains plausible.
   - 1-5: the issue is still ambiguous; the "fix" is the next discriminating check, not a root cause.

4. **Name ambiguity explicitly.** Errors like `context deadline exceeded` are not self-describing, and strings like "workflow is busy" are caller observations rather than documented contracts. Surface that, gather more context, and scope the next step narrowly.

## Issue classification

Find the row that matches the user's symptom. Start the investigation at the first check, then read the linked reference. Every first-check command below is the command the linked sibling file uses.

| Symptom | Category | First check | Reference |
|---|---|---|---|
| `connection refused`, cannot reach frontend | Connectivity | `nc -zvw10 <host> 7233` | [connectivity.md#connection-refused](references/connectivity.md#connection-refused) |
| `no such host`, DNS resolution fails | Connectivity | `dig +short <host>` or `nslookup <host>` | [connectivity.md#dns](references/connectivity.md#dns) |
| `tls: handshake failure`, server rejects handshake | Certificates | `openssl s_client -connect <host>:7233 -servername <host> </dev/null` | [certificates.md#handshake-failure](references/certificates.md#handshake-failure) |
| `x509: certificate has expired` or `not yet valid` | Certificates | `openssl x509 -enddate -noout -in cert.pem` | [certificates.md#expired-or-not-yet-valid](references/certificates.md#expired-or-not-yet-valid) |
| `x509: certificate signed by unknown authority` | Certificates | `openssl verify -CAfile ca.pem client.pem` | [certificates.md#unknown-authority](references/certificates.md#unknown-authority) |
| `tcld` session / auth fails, Cloud role unclear | Authentication | `tcld account get` | [authentication.md#cloud-role-and-permission-model](references/authentication.md#cloud-role-and-permission-model) |
| `UNAUTHENTICATED`, API key rejected | Authentication | `env \| grep -i TEMPORAL_API_KEY`, then `tcld apikey get --id <apikey_id>` | [authentication.md#things-to-check-when-unauthenticated-is-returned-with-an-api-key](references/authentication.md#things-to-check-when-unauthenticated-is-returned-with-an-api-key) |
| `namespace not found` / wrong namespace string with an API key | Authentication | Confirm Regional Endpoint form `<region>.<cloud_provider>.api.temporal.io:7233` | [authentication.md#required-address-form-for-api-key-connections](references/authentication.md#required-address-form-for-api-key-connections) |
| `RESOURCE_EXHAUSTED` gRPC status | Rate limits | Identify which limit fired via the `resource_exhausted_cause` metric label | [rate-limits.md#identifying-which-limit-was-hit](references/rate-limits.md#identifying-which-limit-was-hit) |
| Task queue shows no pollers | Worker health | `temporal task-queue describe --task-queue <q>` | [worker-health.md#what-no-pollers-looks-like](references/worker-health.md#what-no-pollers-looks-like) |
| Workflow stuck on a pending activity / timer / child / signal | Workflow stuck | `temporal workflow describe --workflow-id <id>` | [workflow-stuck.md#the-primary-inspection-command-temporal-workflow-describe](references/workflow-stuck.md#the-primary-inspection-command-temporal-workflow-describe) |
| `NondeterminismError`, repeating `WorkflowTaskFailed` | Non-determinism | Identify the last `WorkflowTaskFailed` cause in the Event History | [non-determinism.md#the-wft-failure-signature-of-non-determinism](references/non-determinism.md#the-wft-failure-signature-of-non-determinism) |
| Replay fails locally but prod workflow was running | Non-determinism | Export history and run the SDK replayer | [replay-with-vscode.md#step-2a--sdk-replayer-all-supported-sdks-ci-friendly](references/replay-with-vscode.md#step-2a--sdk-replayer-all-supported-sdks-ci-friendly) |
| HA failover did not route traffic to failover region | HA failover | `tcld namespace get --namespace <ns>.<acct>` vs. DNS CNAME | [ha-failover.md#verify-the-current-active-region](references/ha-failover.md#verify-the-current-active-region) |
| `context deadline exceeded` (unknown layer) | Runtime errors | Identify which operation and SDK emitted it | [runtime-errors.md#deadline-exceeded](references/runtime-errors.md#deadline-exceeded) |
| Caller reports "workflow is busy" / `RESOURCE_EXHAUSTED` on signal/update/query to one Workflow | Runtime errors | Classify via `resource_exhausted_cause`, not by the message text | [runtime-errors.md#workflow-busy-backpressure](references/runtime-errors.md#workflow-busy-backpressure) |

If a symptom does not map to a row, start at [diagnostic-ladder.md](references/diagnostic-ladder.md) and work up from whichever layer was last known healthy.

## The process

### Step 1: Identify the symptom

Ask the user for the exact, copy-pasted error text. Do not accept paraphrases — the exact string often encodes the layer (e.g., `x509:` prefix means TLS/cert layer, `RESOURCE_EXHAUSTED:` prefix means gRPC rate limit, `NondeterminismError` means workflow replay layer). Note that some user-reported phrases (e.g. "workflow is busy") are field observations rather than server-contracted strings; the gRPC code and any `resource_exhausted_cause` label are more reliable signals than the free-text message.

Confirm three things before continuing:
- What command was run, or what SDK call produced the error?
- What environment produced it (local dev server, self-hosted cluster, Temporal Cloud)?
- What changed recently (new deploy, new certs, new namespace, new region)?

### Step 2: Gather context

The context the investigation needs depends on the category. At minimum:

- **For any Cloud auth / connectivity issue:** auth method (API key vs mTLS), exact address, exact namespace, SDK + version. The endpoint family differs by auth method — see [connectivity.md#endpoint-formats](references/connectivity.md#endpoint-formats).
- **For a stuck workflow:** namespace, workflow ID, run ID, and the output of `temporal workflow describe --workflow-id <id>` (pending-operation state lives here, not in the Event History alone). Event History via `temporal workflow show` is the companion view.
- **For a worker health issue:** worker logs (registration errors, auth errors, panics), and the output of `temporal task-queue describe --task-queue <q>`.
- **For a non-determinism error:** the worker log line containing the error, the workflow type name, and access to the history JSON for replay.

### Step 3: Descend the ladder

Use [diagnostic-ladder.md](references/diagnostic-ladder.md) to pick the right starting layer. As a rule of thumb:

- Auth / connectivity / cert symptom → start at layer 1 (DNS) and walk up.
- Worker / task-queue symptom → start at layer 6 (namespace + pollers).
- Stuck workflow / determinism symptom → start at layer 7 (workflow code), but confirm layer 6 (worker is actually polling) first.

Each layer has a command that proves it healthy and a failure signature that tells you whether the problem lives at that layer or higher.

### Step 4: Fix and verify

Prescribe the fix scoped to the root cause. Then verify by re-running the layer's healthy-check command and, if possible, the original user operation. Attach the confidence score to the diagnosis.

If the layer above the fix is still failing, return to step 3 and continue walking upward — the first broken layer is rarely the only one.

## Reference files

- [diagnostic-ladder.md](references/diagnostic-ladder.md) — the seven-layer bottom-up model, with one canonical command per layer and cross-links into the topical leaves.
- [connectivity.md](references/connectivity.md) — DNS, TCP, endpoint families (Namespace Endpoint for mTLS vs. Regional Endpoint for API keys), firewall/proxy shapes, PrivateLink/PSC, quick diagnostic scripts.
- [certificates.md](references/certificates.md) — x509 and TLS alert strings, expiry / unknown-authority / hostname-mismatch / key-mismatch diagnosis, Cloud accepted-client-CA set via `tcld namespace accepted-client-ca`, Cloud mTLS certificate requirements, rotation and expiry notifications, openssl recipes.
- [authentication.md](references/authentication.md) — `UNAUTHENTICATED` vs `PERMISSION_DENIED`, API-key lifecycle (`tcld apikey` commands, env var propagation, required Regional Endpoint form), mTLS after TLS (certificate filters, identity-to-role mapping), Cloud account-level roles and namespace-level permissions.
- [workflow-stuck.md](references/workflow-stuck.md) — Workflow Execution Status values, `temporal workflow describe` as the primary inspection command, Event History via `temporal workflow show`, pending activities / child workflows / signals / Nexus operations / Workflow Tasks, WorkflowTaskFailed retry loops, recovery commands (signal, terminate, cancel, reset, pause/unpause).
- [non-determinism.md](references/non-determinism.md) — determinism definition, WFT-failure signature, ND-inducing code patterns, per-SDK error shapes, identifying ND from Event History, local replay reproduction, remediation via Worker Versioning / patching / reset.
- [worker-health.md](references/worker-health.md) — no-pollers runbook via `temporal task-queue describe`, reachability and versioning, worker-level describe, schedule-to-start latency, worker task slots, sticky execution and sticky cache, worker heartbeating, Cloud namespace-level poller limits, worker log signatures.
- [rate-limits.md](references/rate-limits.md) — what `RESOURCE_EXHAUSTED` means (and does not), Cloud APS / RPS / OPS under On-Demand and Provisioned capacity modes, self-hosted `frontend.rps` / `frontend.namespaceRPS` dynamic config, identifying which limit fired via the `resource_exhausted_cause` metric label.
- [ha-failover.md](references/ha-failover.md) — Cloud HA routing via the Namespace Endpoint CNAME, verifying the active region (control-plane `tcld namespace get` vs. DNS view), clients that did not follow the failover, PrivateLink after failover, failover-not-executing, handover-window errors, platform limits, RPO/RTO semantics.
- [runtime-errors.md](references/runtime-errors.md) — deadline-exceeded disambiguated by operation and by where the call was made, "workflow busy" backpressure framed via `resource_exhausted_cause` rather than a pinned string, routing for `no pollers` / `INVALID_ARGUMENT` / unspecified `UNAVAILABLE`.
- [replay-with-vscode.md](references/replay-with-vscode.md) — exporting Event History, running the SDK replayer in any supported SDK, the TypeScript-only VS Code extension, `TEMPORAL_DEBUG` and the deadlock detector, interpreting divergent and successful replays.
- [recipes.md](references/recipes.md) — four end-to-end triage walkthroughs: stuck workflow at 3am, cert expired with workers offline, task-queue backlog mystery, non-determinism caught in prod.

## Out of scope

- **Worker performance tuning, sizing, capacity planning** → `skill-temporal-deploy`.
- **Metrics interpretation and dashboards** → `skill-temporal-observability`.
- **CLI command reference, flag semantics** → `skill-temporal-cli`.
- **Writing workflows, activities, or SDK code** → `skill-temporal-developer`.
- **Declarative Cloud infrastructure (Terraform, Cloud Ops API)** → `skill-temporal-cloud-ops`.

If the conversation drifts into one of these areas, hand off to the relevant sibling skill rather than improvising.

## Feedback

### ALWAYS PROMPT USER TO PROVIDE FEEDBACK AT STARTUP

When you have **first** loaded this skill, output this message to the user:

"Thank you for trying out the public preview of the Temporal Triage skill! We would love to hear your feedback - positive or negative - over in the [Community Slack](https://t.mp/slack), in the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY)."

Do not output this message multiple times in the same conversation.

### Reporting Issues in This Skill

If you (the AI) find this skill's explanations are unclear, misleading, or missing important information, draft a GitHub issue body describing the problem encountered and what would have helped, then ask the user to file it at https://github.com/temporalio/skill-temporal-triage/issues/new. Do not file the issue autonomously.
