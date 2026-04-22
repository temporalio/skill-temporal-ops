# Runtime Errors

Disambiguate Temporal errors whose text does not by itself identify the layer at fault. Primary focus: `context deadline exceeded` (`DEADLINE_EXCEEDED` <!-- grpc: DEADLINE_EXCEEDED -->) and "workflow busy" backpressure. Secondary: where to route `no pollers`, `INVALID_ARGUMENT`, and unspecified `UNAVAILABLE` <!-- grpc: UNAVAILABLE --> when the layer isn't obvious from the message alone.

Unambiguous errors are covered in the layer-specific files — link, don't duplicate:
- DNS / TCP / endpoint / PrivateLink → [connectivity.md](connectivity.md)
- TLS / x509 / mTLS alerts → [certificates.md](certificates.md)
- `UNAUTHENTICATED` / `PERMISSION_DENIED` → [authentication.md](authentication.md)
- `RESOURCE_EXHAUSTED` rate-limit anatomy → [rate-limits.md](rate-limits.md)
- Pollers / schedule-to-start / sticky cache → [worker-health.md](worker-health.md)
- Pending activities / children / signals / Workflow Task failures → [workflow-stuck.md](workflow-stuck.md)
- Blob-size / history-size limits → `docs/troubleshooting/blob-size-limit-error.mdx` (not ambiguous; the error names itself)
- Performance-bottlenecks deep dive → `docs/troubleshooting/performance-bottlenecks.mdx`

## Table of Contents

- [Why these errors are hard](#why-these-errors-are-hard)
- [Deadline exceeded](#deadline-exceeded)
- [Workflow busy backpressure](#workflow-busy-backpressure)
- [Other frequently ambiguous errors](#other-frequently-ambiguous-errors)
- [Triage protocol](#triage-protocol)

## Why these errors are hard

`context deadline exceeded` is emitted by the Go `context` package <!-- go: context --> and propagates through gRPC as `DEADLINE_EXCEEDED` <!-- grpc: DEADLINE_EXCEEDED -->. It tells you only that the caller gave up waiting — not why the response did not arrive. The Temporal troubleshooting guide lists "network interruptions, timeouts, server overload, and Query errors" as causes in the same breath, and the fix catalog spans clock skew, Frontend Service reachability, rate-limit saturation, client/worker configuration, and connection-age tuning. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:17 --><!-- docs/troubleshooting/deadline-exceeded-error.mdx:20-118 -->

The "workflow busy" shape — `RESOURCE_EXHAUSTED` <!-- grpc: RESOURCE_EXHAUSTED --> on operations targeting a single Workflow Execution — is not documented verbatim in the local docs clone. <!-- VERIFY: grep of /Users/joe/sap/documentation/docs for "workflow is busy" / "Workflow is busy" / "BusyWorkflow" on 2026-04-21 returned zero hits. The string is known from server behavior in the field; treat it as an observed shape and use the `RESOURCE_EXHAUSTED` code plus the `resource_exhausted_cause` metric label to classify. -->

Treating either error as a single-layer failure is the most common triage mistake in this category. Identify the operation and the layer before prescribing a fix. Give the proposed root cause an explicit confidence label — "low confidence, next discriminating check is X" beats a guess dressed up as a diagnosis. (Confidence framing is a skill convention, not a Temporal contract.)

## Deadline exceeded

**Verbatim error shapes seen in the wild:**
- `Context: deadline exceeded` — surfaced by the Temporal troubleshooting guide. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:16 -->
- `rpc error: code = DeadlineExceeded desc = context deadline exceeded` — Go gRPC client. <!-- grpc: DEADLINE_EXCEEDED -->
- `Error: 4 DEADLINE_EXCEEDED: context deadline exceeded` — the `@grpc/grpc-js` TypeScript client form documented in the Temporal TS debugging guide. <!-- docs/develop/typescript/best-practices/debugging.mdx:291 -->

**What it means:** the caller's deadline fired before a response arrived. The code is `DEADLINE_EXCEEDED` <!-- grpc: DEADLINE_EXCEEDED --> regardless of which layer failed to respond.

**Causes documented in the Temporal troubleshooting guide for `deadline-exceeded`:** <!-- docs/troubleshooting/deadline-exceeded-error.mdx:21-118 -->

- **Clock skew** between a Worker and the Temporal Service exceeding an Activity's Start-To-Close Timeout — produces `Activity complete after timeout` alongside `Context: deadline exceeded`. Resolution: sync to NTP. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:21-27 -->
- **Frontend Service not reachable.** OSS users can check with `temporal operator cluster health --address 127.0.0.1:7233`; `grpc-health-probe` lets you probe Frontend, Matching, and History individually. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:29-56 --> Cloud users cannot access these logs directly — the guide directs them to open a support ticket with Namespace Name and sample Workflow IDs. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:33-37 -->
- **Resource-exhausted backpressure masquerading as deadline.** "A `resource exhausted` error can cause your client request to fail, which prompts the `deadline exceeded` error." <!-- docs/troubleshooting/deadline-exceeded-error.mdx:62-63 --> Discriminator query (self-hosted metrics): `sum(rate(service_errors_resource_exhausted{}[1m])) by (resource_exhausted_cause)`, watching for `RpsLimit`, `ConcurrentLimit`, `SystemOverloaded`. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:65-68 --> For the Cloud equivalent and the `RESOURCE_EXHAUSTED` anatomy see [rate-limits.md](rate-limits.md#identifying-which-limit-was-hit).
- **Invalid client/worker configuration** (wrong server name, address, or certificate). These produce `connection refused` alongside `deadline exceeded`; the guide explicitly pairs the two. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:74-83 --> Re-run the [connectivity ladder](connectivity.md) before anything else.
- **Service just restarted / roles not yet initialized.** Wait and retry; review Workflow Execution history and server logs if it persists. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:100-105 -->
- **Self-hosted: `frontend.keepAliveMaxConnectionAge` too short** for in-flight requests — increase it and monitor server load. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:107-118 -->

**TypeScript SDK guide adds two concrete triggers for a `context deadline exceeded` at call time:** <!-- docs/develop/typescript/best-practices/debugging.mdx:308-315 -->

- Network hiccup, timeout that's too short, or overloaded server.
- "Querying a Workflow Execution whose query handler causes an error can result in the query call timing out." <!-- docs/develop/typescript/best-practices/debugging.mdx:309 -->

**Nexus-specific:** "If a Nexus handler doesn't process a start or cancel request within 10 seconds, it will receive a context deadline exceeded error, and the caller will retry, with an exponential backoff, for the ScheduleToClose duration for the overall Nexus Operation." <!-- docs/evaluate/temporal-cloud/limits.mdx:293 -->

### Discriminating by where the call was made

**First question to ask the reporter:** which operation emitted this — workflow start, signal/update, query, a worker poll, a Nexus start/cancel, or an un-attributed log line? The answer narrows the layer.

| Operation | Likely-first check |
|---|---|
| Workflow start / signal / update / describe (RPC from a client) | Confirm DNS / TCP / TLS / auth succeed for the same endpoint from the failing environment. Start at [connectivity.md → Quick diagnostic scripts](connectivity.md#quick-diagnostic-scripts). If layers 1–4 pass, move to Frontend health <!-- docs/troubleshooting/deadline-exceeded-error.mdx:29-56 --> and the `resource_exhausted_cause` metric. |
| Workflow start with a large input | Rule out `BlobSizeLimitError` (2 MB per request, 4 MB per Event History transaction). <!-- docs/troubleshooting/blob-size-limit-error.mdx:15-18 --> That error is self-naming, not a bare `deadline exceeded`, but large payloads also inflate request latency (`temporal_request_latency`) per the bottlenecks guide. <!-- docs/troubleshooting/performance-bottlenecks.mdx:219-220 --> |
| Query | The Workflow's Query handler may itself be erroring out. <!-- docs/develop/typescript/best-practices/debugging.mdx:309 --> Queries run in the Worker and the call is synchronous. <!-- docs/encyclopedia/workflow-message-passing/sending-messages.mdx:175 --> Check Worker logs for an exception raised inside the handler; fix and redeploy. |
| Worker long-poll (`PollWorkflowTaskQueue`, `PollActivityTaskQueue`) | `temporal_long_request_failure` is counted against these poll RPCs; the bottlenecks guide lists network issues, rate limiting (often indicated by `ResourceExhausted`), and server errors as the three cause classes. <!-- docs/troubleshooting/performance-bottlenecks.mdx:183-187 --> Jump to [worker-health.md → Worker log signatures](worker-health.md#worker-log-signatures). |
| Nexus handler call | 10-second handler cap; see `docs/evaluate/temporal-cloud/limits.mdx` for Nexus timeout semantics. <!-- docs/evaluate/temporal-cloud/limits.mdx:293 --> |

### PrivateLink-specific deadline exceeded

`context deadline exceeded` from a Worker or CLI going through AWS PrivateLink or GCP Private Service Connect has a common set of layer-1–3 causes. The commands below are the ones already grounded in the sibling files — reuse them rather than re-deriving.

1. **VPC-endpoint path unreachable from the client subnet.**
   ```bash
   nc -zvw10 vpce-0123456789abcdef-abc.us-east-1.vpce.amazonaws.com 7233   # man: nc(1)
   ```
   If this times out, the VPC-endpoint security group is not permitting TCP/7233 from the client subnet. <!-- docs/cloud/connectivity/aws-connectivity.mdx:68 --> The exact probe command form is the one used in the Cloud connectivity guide. <!-- docs/cloud/connectivity/index.mdx:317-319 --> See [connectivity.md → PrivateLink and PSC](connectivity.md#privatelink-and-psc).
2. **TLS handshake fails because SNI is not overridden.** When connecting by the VPC-endpoint DNS name (i.e. not via private DNS), the client must set the TLS server name to the Namespace Endpoint — `<namespace>.<account>.tmprl.cloud`. The Cloud connectivity guide gives the exact env-var form: `TEMPORAL_ADDRESS=vpce-...:7233` paired with `TEMPORAL_TLS_SERVER_NAME=my-namespace.my-account.tmprl.cloud`. <!-- docs/cloud/connectivity/index.mdx:208-221 --> Full details: [certificates.md → Server name override](certificates.md#server-name-override).
3. **Private DNS missing for the region the Namespace is currently in (HA Namespaces).** After a failover, the `region.tmprl.cloud` private hosted zone must cover every region the Namespace can fail over to. <!-- docs/cloud/high-availability/ha-connectivity.mdx:58-61 --> Details: [ha-failover.md → PrivateLink stopped working after failover](ha-failover.md#symptom-privatelink-stopped-working-after-failover).
4. **PrivateLink not enabled on the Namespace.** Verify connectivity configuration on the Namespace; if the Namespace is not configured for PrivateLink, public DNS will route the caller somewhere the VPC cannot reach. <!-- docs/cloud/connectivity/index.mdx:201-212 -->

## Workflow busy backpressure

**Observed error shape:** a `RESOURCE_EXHAUSTED` <!-- grpc: RESOURCE_EXHAUSTED --> returned on operations (start / signal / update / query) targeting a single Workflow Execution, often reported by users as "workflow is busy". The verbatim string `workflow is busy` / `Workflow is busy` / `BusyWorkflow` is **not present** in the local docs clone; treat "workflow busy" as a field observation, not a documented contract. <!-- VERIFY: grepped /Users/joe/sap/documentation/docs on 2026-04-21 for "workflow is busy", "Workflow is busy", "BusyWorkflow", "WorkflowBusy", "workflow busy" — zero hits. Classify by code + `resource_exhausted_cause` label; don't pattern-match on the text. -->

**What the docs do say about operations competing on the same Workflow Execution:**

- Workflow Tasks are scheduled for a Workflow Execution, and the in-flight state is inspectable via `pendingWorkflowTask` in `temporal workflow describe`. <!-- docs/cli/index.mdx:397-402 --> An accumulation of pending operations against a single Workflow is an observable condition via describe, not via a named server error.
- "High Workflow lock latency. If many updates are made to a single execution, this can cause Workflow lock latency, which in turn affects the Schedule-to-start latency. Reduce the rate of Signals." <!-- docs/troubleshooting/performance-bottlenecks.mdx:38 --> This is the docs' framing of single-execution hot-spot pressure.

**What this is not:**
- Not a Namespace-wide rate limit — that is APS / RPS / OPS on Cloud or `frontend.rps` / `frontend.namespaceRPS` self-hosted. See [rate-limits.md](rate-limits.md).
- Not a Workflow failure. A `RESOURCE_EXHAUSTED` on a signal/update does not fail the Workflow Execution; the SDK's default gRPC retry policy retries the RPC with backoff. <!-- docs/evaluate/temporal-cloud/limits.mdx:97 -->
- Not "the workflow is blocked in a useful sense." The Workflow may be perfectly healthy and the pressure is on the caller's side.

**Things to discriminate:**
- **Which RPC returned the error?** Signals, updates, and queries against the same Workflow ID are the usual culprits when a caller (or a caller's retry loop) fans in.
- **Caller concurrency vs. the same Workflow ID.** Rate of operations per second from all callers against that one ID.
- **Is the caller retrying without backoff?** Retries count against the budget. <!-- docs/evaluate/temporal-cloud/limits.mdx:97-98 --> A raw gRPC client reimplementing retry must implement exponential backoff.

**Mitigation shapes (docs-anchored where possible):**
- Throttle or coalesce signals on the caller side; the bottlenecks guide's "Reduce the rate of Signals" wording applies. <!-- docs/troubleshooting/performance-bottlenecks.mdx:38 -->
- Fan out across multiple Workflow IDs when the entity is genuinely many things.
- Rely on SDK retry with backoff for transient bursts. <!-- docs/evaluate/temporal-cloud/limits.mdx:97 -->

**Which limiter actually fired?** Classify with the server-side cause label, not the message text. Cloud exposes `temporal_cloud_v0_resource_exhausted_error_count` labeled by `resource_exhausted_cause`. <!-- docs/cloud/metrics/reference.mdx:83-86 --><!-- docs/cloud/metrics/reference.mdx:217 --> Self-hosted exposes the same label via `service_errors_resource_exhausted`. <!-- docs/troubleshooting/deadline-exceeded-error.mdx:65-68 --> See [rate-limits.md → From the error](rate-limits.md#from-the-error) for the full protocol.

## Other frequently ambiguous errors

### `no pollers`

This phrase is a clue, not a diagnosis. Common shapes: no Worker reached the frontend for the queue+type within the last 5 minutes, Workers are polling a different queue or Namespace, or Workers connect but fail before the poll loop. <!-- docs/cli/task-queue.mdx:99-101 --> Verify from `temporal task-queue describe`, not from cached metrics. Full protocol: [worker-health.md → What "no pollers" looks like](worker-health.md#what-no-pollers-looks-like).

### `INVALID_ARGUMENT`

`INVALID_ARGUMENT` <!-- grpc: INVALID_ARGUMENT --> is a catch-all. The suffix is the useful part:

- `namespace not found` — the namespace string does not exist in this account. Confirm via `tcld namespace list` <!-- docs/cloud/tcld/namespace.mdx --> (see [connectivity.md → Endpoint formats](connectivity.md#endpoint-formats) for the correct Namespace Endpoint form, which is a common source of this shape).
- Field-specific validation errors — fix the input; the suffix names the bad field.

### Unspecified `UNAVAILABLE`

`UNAVAILABLE` <!-- grpc: UNAVAILABLE --> on its own does not pin a layer. Peel the wrapped cause: `tls:` / `x509:` / `remote error: tls:` → [certificates.md](certificates.md); `connection refused` / `no such host` / `i/o timeout` → [connectivity.md](connectivity.md). If the wrapped cause is absent, run the [diagnostic ladder](diagnostic-ladder.md) from layer 1.

## Triage protocol

For any ambiguous runtime error:

1. **Demand the exact text.** Copy-paste, not paraphrase. An `UNAVAILABLE` with `x509:` wrapped inside is a TLS problem, not a network one.
2. **Identify which operation produced it** — start / signal / update / query / poll / Nexus handler / internal. The operation narrows the candidate layers.
3. **Identify which environment produced it** — local dev, self-hosted, Cloud. Different error catalogs (e.g. Cloud's `resource_exhausted_cause` labels versus self-hosted `service_errors_resource_exhausted`).
4. **Walk the diagnostic ladder** up to the layer the evidence implicates. See [diagnostic-ladder.md](diagnostic-ladder.md).
5. **Attach confidence.** Below 6, the next action is a discriminating check (DNS lookup, `openssl s_client`, `temporal operator cluster health`), not a fix. Confidence framing is a skill-level triage norm, not a documented Temporal property — but it's how this skill avoids prescribing fixes on the strength of ambiguous evidence.
