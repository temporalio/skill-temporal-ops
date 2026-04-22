# Replay with VS Code

This file is about the *tooling* that reproduces a recorded Workflow Execution in a debugger. For what replay divergence *means* as a concept — why the Worker Task fails, how the server classifies the cause, and how to remediate — see [non-determinism.md](non-determinism.md).

Replay is "the method by which a Workflow Execution resumes making progress. During a Replay the Commands that are generated are checked against an existing Event History." <!-- docs/encyclopedia/workflow/workflow-execution/workflow-execution.mdx:69 --> Running a recorded history through a local replayer against your Worker source tree is the canonical way to reproduce a non-determinism error under a debugger, verify a fix, and pin a CI regression test.

Out of scope (link, don't absorb):

- What non-determinism means, how to identify it in an Event History, and remediation options → [non-determinism.md](non-determinism.md)
- Worker not polling the Task Queue at all — nothing to replay, fix the Worker first → [worker-health.md](worker-health.md)
- Can't reach the server to fetch the history → [connectivity.md](connectivity.md), [authentication.md](authentication.md)
- The bottom-up layer model for routing between files → [diagnostic-ladder.md](diagnostic-ladder.md)

## Table of Contents

- [Two replay tools: SDK replayer and VS Code extension](#two-replay-tools-sdk-replayer-and-vs-code-extension)
- [Prerequisites](#prerequisites)
- [Step 1 — Export the Event History](#step-1--export-the-event-history)
- [Step 2a — SDK replayer (all supported SDKs, CI-friendly)](#step-2a--sdk-replayer-all-supported-sdks-ci-friendly)
- [Step 2b — VS Code extension (TypeScript only, interactive)](#step-2b--vs-code-extension-typescript-only-interactive)
- [TEMPORAL_DEBUG: suppress the deadlock detector while stepping](#temporal_debug-suppress-the-deadlock-detector-while-stepping)
- [Interpreting a replay that diverges](#interpreting-a-replay-that-diverges)
- [Interpreting a replay that succeeds](#interpreting-a-replay-that-succeeds)
- [Limitations of the VS Code extension](#limitations-of-the-vs-code-extension)

## Two replay tools: SDK replayer and VS Code extension

There are two complementary ways to replay a recorded history against Worker source code:

1. **SDK replayer APIs** — a call you make from your test or application code. Available in Go, Python, TypeScript, and Java (and surfaced in each SDK's *testing-suite* docs). Runs headless; attach your IDE's debugger of choice. Suitable for CI as a regression guard.
2. **Temporal's VS Code extension** — a wrapper around the TypeScript SDK's replayer that integrates with VS Code's debugger UI. One-click "open history → set breakpoints → step through." As of the marketplace listing consulted, the extension **debugs TypeScript workflows only**. <!-- https://marketplace.visualstudio.com/items?itemName=temporal-technologies.temporalio -->

For Go / Python / Java, use the SDK replayer and attach your IDE's debugger (Delve / pdb / IntelliJ) to the test process that drives `ReplayWorkflowHistory` / `Replayer.replay_workflow` / `WorkflowReplayer.replayWorkflowExecution`. There is no first-party VS Code extension for these SDKs in the sources consulted. <!-- VERIFY: if the extension has added support for additional SDKs since 2026-04, update this file. -->

## Prerequisites

- The Workflow source tree at the commit that was deployed when the recorded Workflow Execution ran. Replaying current `main` against an older recording will produce divergence *for a different reason than the bug you're triaging*.
- The SDK installed and importable in that workspace (the replayer is part of the SDK, not a standalone binary).
- A way to fetch the Event History of the run — either the `temporal` CLI or the SDK client's history-fetch API.
- For the VS Code extension: VS Code, and a TypeScript Workflow codebase. <!-- https://marketplace.visualstudio.com/items?itemName=temporal-technologies.temporalio -->

## Step 1 — Export the Event History

Both tools take the same input: a JSON Event History. Export it with:

```bash
temporal workflow show \
    --workflow-id YourWorkflowId \
    --run-id YourRunId \
    --output json > history.json
```

`--output` accepts `table, json, card`; replay requires `json`. <!-- docs/cli/cmd-options.mdx:450 --> `--run-id` is optional (not required). <!-- docs/cli/workflow.mdx:439 --> If omitted, standard CLI behavior is to target the most recent run of the given Workflow ID; the run id can also be obtained via `temporal workflow describe` (see [workflow-stuck.md](workflow-stuck.md)) or the Web UI.

The docs explicitly document the replayer handoff: "When using JSON output (`--output json`), you may pass the results to an SDK to perform a replay." <!-- docs/cli/workflow.mdx:424-425 -->

If the run id is unknown, find it via `temporal workflow describe --workflow-id YourWorkflowId` (see [workflow-stuck.md](workflow-stuck.md)) or via the Web UI. The TypeScript testing-suite page notes that histories can also be obtained "from the [Web UI](/web-ui) or the [Temporal CLI](/cli/workflow#show)." <!-- docs/develop/typescript/best-practices/testing-suite.mdx:536 --> Python's testing-suite page makes the same point: "If the Workflow History is exported by [Temporal Web UI](/web-ui) or through [Temporal CLI](/cli), you can pass the JSON file history object as a JSON string…" <!-- docs/develop/python/best-practices/testing-suite.mdx:214-216 -->

For Cloud or any non-default target, add the usual connection flags (`--address`, `--namespace`, and either `--api-key` or the mTLS `--tls-*` flags). See [authentication.md](authentication.md).

## Step 2a — SDK replayer (all supported SDKs, CI-friendly)

Each SDK's testing-suite page documents a replayer. Names and signatures below are transcribed from those pages; they are what the docs state, not what the runtime type system exports at any given version.

### Go

```go
replayer := worker.NewWorkflowReplayer()
replayer.RegisterWorkflow(YourWorkflow)
err := replayer.ReplayWorkflowHistory(nil, hist)
```

<!-- docs/develop/go/best-practices/testing-suite.mdx:477-479 --> Use `worker.WorkflowReplayer` to "replay an existing Workflow Execution from its Event History to replicate errors." <!-- docs/develop/go/best-practices/testing-suite.mdx:433 --> "If a noticeably different code path was followed or some code caused a deadlock, it will be returned in the error code." <!-- docs/develop/go/best-practices/testing-suite.mdx:484 -->

Attach Delve / your IDE debugger to the test process that calls `ReplayWorkflowHistory`. Set `TEMPORAL_DEBUG=true` while stepping (see below).

### Python

```python
replayer = Replayer(workflows=[YourWorkflow])
await replayer.replay_workflow(WorkflowHistory.from_json(history_json_str))
```

<!-- docs/develop/python/best-practices/testing-suite.mdx:206-208 --> For bulk replay, use `replayer.replay_workflows(histories)`. <!-- docs/develop/python/best-practices/testing-suite.mdx:198-202 --> "If any replay fails, the code raises an exception." <!-- docs/develop/python/best-practices/testing-suite.mdx:193 --> The docs note a pitfall with history encoding: "when fetching event histories directly from the server or exporting them, be aware that the data can be protobuf-encoded (`bytes`). The `Replayer`, however, often works with decoded histories (like a `dict`)." <!-- docs/develop/python/best-practices/testing-suite.mdx:219-221 -->

Attach pdb / your IDE debugger to the test process.

### TypeScript

```ts
const history = JSON.parse(await fs.promises.readFile('./history.json', 'utf8'));
await Worker.runReplayHistory(
  { workflowsPath: require.resolve('./your/workflows') },
  history,
);
```

<!-- docs/develop/typescript/best-practices/testing-suite.mdx:538-547 --> For bulk replay, use `Worker.runReplayHistories`. <!-- docs/develop/typescript/best-practices/testing-suite.mdx:565, 575 -->

Two error classes are documented: "When an Event History is replayed and non-determinism is detected (that is, the Workflow code is incompatible with the History), `DeterminismViolationError` is thrown. If replay fails for any other reason, `ReplayError` is thrown." <!-- docs/develop/typescript/best-practices/testing-suite.mdx:533-534 -->

### Java

```java
File file = new File("history.json");
WorkflowReplayer.replayWorkflowExecution(file, MyWorkflow.class);
```

<!-- docs/develop/java/best-practices/testing-suite.mdx:595-597 --> Use `WorkflowReplayer` from the `temporal-testing` package. <!-- docs/develop/java/best-practices/testing-suite.mdx:559 --> For bulk replay, `WorkflowReplayer.replayWorkflowExecutions`. <!-- docs/develop/java/best-practices/testing-suite.mdx:589 --> "In both examples, if Event History is non-deterministic, an error is thrown. You can choose to wait until all histories have been replayed with `replayWorkflowExecutions` by setting the `failFast` argument to `false`." <!-- docs/develop/java/best-practices/testing-suite.mdx:600-601 -->

### .NET, Ruby, PHP

Per-SDK replayer APIs are documented under each SDK's testing-suite page in the `docs/develop/<sdk>/best-practices/` tree. Transcribe the exact API from the relevant page for the SDK version in use. <!-- VERIFY: exact names for .NET / Ruby / PHP replayers not transcribed in this file. -->

## Step 2b — VS Code extension (TypeScript only, interactive)

Claims in this section come from the marketplace listing of the official extension. <!-- https://marketplace.visualstudio.com/items?itemName=temporal-technologies.temporalio, consulted 2026-04-21 --> The Temporal docs themselves do not document this extension at the time of writing. <!-- VERIFY: local /Users/joe/sap/documentation/docs/ was Grep'd for "vscode" and "VS Code" with zero hits on 2026-04-21. -->

### Install

Search the VS Code Marketplace for "Temporal" and install the extension published by **Temporal Technologies Inc.** (marketplace ID `temporal-technologies.temporalio`). <!-- marketplace -->

### Configure the entrypoint

The extension's replayer is driven by a TypeScript entrypoint file that calls `startDebugReplayer` with a `workflowsPath`. <!-- marketplace --> The extension reads the path to this file from the `temporal.replayerEntrypoint` VS Code setting, which defaults to `src/debug-replayer.ts`. <!-- marketplace -->

<!-- VERIFY: the exact `startDebugReplayer` import path and option object shape should be copied from the extension README (github.com/temporalio/vscode-debugger-extension per the marketplace) at the version you install. This file does not fabricate the signature. -->

### Start a session

The marketplace listing documents the following flow: <!-- marketplace -->

1. Run the **Temporal: Open Panel** command from the Command Palette (`Cmd/Ctrl-Shift-P`).
2. In the panel, enter a Workflow ID (the extension fetches its history from the configured server) or pick a history JSON file on disk.
3. Click **Start**.
4. Set breakpoints in your Workflow source *or* on history events, then step through the replay.

### Server connection

The extension connects to a Temporal server to fetch history by Workflow ID. The marketplace listing notes a default of `localhost:7233` and exposes server address, TLS certificate, and TLS key options in the extension's SETTINGS tab. <!-- marketplace --> For Cloud, supply the Cloud gRPC endpoint plus the same credentials documented in [authentication.md](authentication.md) and [certificates.md](certificates.md).

If you only need to replay a history file and not fetch by ID, the server connection is not required — pick the history JSON file in step 2 above.

<!-- VERIFY: whether the extension supports API-key auth (in addition to mTLS) is not documented on the marketplace page at the snapshot date. Check the extension's README for the installed version. -->

## TEMPORAL_DEBUG: suppress the deadlock detector while stepping

Two SDKs explicitly document a debug-mode env var. Without it, pausing on a breakpoint for more than a second can cause the Worker's deadlock detector to fail the Workflow Task *during your debugging session*:

- **Go.** "The Temporal Go SDK includes deadlock detection which fails a Workflow Task in case the code blocks over a second without relinquishing execution control. Because of this you can often encounter a `PanicError: Potential deadlock detected` while stepping through Workflow Definitions during debugging. To alleviate this issue, you can set the `TEMPORAL_DEBUG` environment variable to `true` before debugging your Workflow Definition." <!-- docs/develop/go/best-practices/debugging.mdx:28-32, 315-318 -->
- **Java.** "The Temporal Java SDK includes deadlock detection which fails a Workflow Task in case the code blocks over a second without relinquishing execution control. Because of this you can often encounter the `PotentialDeadlockException` Exception while stepping through Workflow code during debugging. To alleviate this issue, you can set the `TEMPORAL_DEBUG` environment variable to true before debugging your Workflow code." <!-- docs/develop/java/best-practices/debugging.mdx:28-31 -->

Both pages add the same warning: "Make sure to set `TEMPORAL_DEBUG` to true only during debugging." <!-- docs/develop/go/best-practices/debugging.mdx:35, 322 --><!-- docs/develop/java/best-practices/debugging.mdx:31 -->

Python and TypeScript debugging pages do not document a `TEMPORAL_DEBUG` env var in the sources consulted. <!-- docs/develop/python/best-practices/debugging.mdx --><!-- docs/develop/typescript/best-practices/debugging.mdx --><!-- VERIFY: whether Python and TypeScript SDKs honor a `TEMPORAL_DEBUG` env var is not stated on their debugging pages. If your debugger sessions are being killed by a deadlock detector in those SDKs, check the SDK's source for an equivalent flag. -->

## Interpreting a replay that diverges

The documented behavior when replay detects non-determinism:

- **TypeScript**: throws `DeterminismViolationError`; any other replay failure throws `ReplayError`. <!-- docs/develop/typescript/best-practices/testing-suite.mdx:533-534 -->
- **Go**: the replayer returns an error from `ReplayWorkflowHistory`; the docs describe the condition as "cause the Workflow to fail with a nondeterminism error" without pinning a public type name. <!-- docs/develop/go/workflows/versioning.mdx:70 --><!-- VERIFY: exact Go error type is not spelled out in the docs snapshot. -->
- **Java**: `WorkflowReplayer.replayWorkflowExecution` throws; the versioning doc describes the condition as "This would cause the Workflow to fail with a nondeterminism error." <!-- docs/develop/java/workflows/versioning.mdx:71 --><!-- VERIFY: exact Java exception class is not stated in the docs snapshot. -->
- **Python**: `Replayer.replay_workflow` raises; "If any replay fails, the code raises an exception." <!-- docs/develop/python/best-practices/testing-suite.mdx:193 -->

Under a debugger — whether VS Code's extension or a native IDE attach on the SDK replayer — the process halts where the SDK throws. The stack frame where execution stops is *the Worker machinery that detected the mismatch*, not the Workflow line that emitted the bad Command. To locate the offending Workflow line, compare:

- The last Command the code was about to emit (the frame just below the SDK entry in the stack), and
- The next non-bookkeeping Event in the recorded history (see [non-determinism.md §Identifying ND from the Event History](non-determinism.md#identifying-nd-from-the-event-history)).

The divergence is the mismatch between those two. The encyclopedia frames this as: "If a generated Command doesn't match what it needs to in the existing Event History, then the Workflow Execution returns a non-deterministic error." <!-- docs/encyclopedia/workflow/workflow-definition.mdx:196 -->

For remediation paths (Worker Versioning, per-SDK patching, reset past the divergence), return to [non-determinism.md §Remediation](non-determinism.md#remediation-worker-versioning-preferred).

## Interpreting a replay that succeeds

If replay succeeds locally against the checked-out source, but production Workers keep failing with the same Workflow Execution, the deployed Worker code is different from the local checkout. Find the deployed build (via Worker Versioning metadata if used, or your deploy system) and reproduce from that commit. See [non-determinism.md §Reproducing ND locally via replay](non-determinism.md#reproducing-nd-locally-via-replay).

A local-replay success on the *current* source with production still failing is also the signal that Worker Versioning would have prevented this class of incident. See [non-determinism.md §Remediation: Worker Versioning (preferred)](non-determinism.md#remediation-worker-versioning-preferred).

## Limitations of the VS Code extension

What the marketplace listing does *not* document, and therefore this file does not claim:

- A supported-SDK matrix beyond TypeScript. <!-- VERIFY -->
- A documented set of extension commands beyond **Temporal: Open Panel**. <!-- VERIFY: other commands may exist; this file lists only the one the marketplace page names. -->
- `launch.json` recipes. The extension starts its own debug session via the panel; whether (and how) `launch.json` entries are supported alongside it is not spelled out on the marketplace page. <!-- VERIFY -->
- OS requirements. Not stated on the marketplace page. <!-- VERIFY -->
- Auth-method matrix (mTLS vs. API key). Only "client certificate and key options" are mentioned in the SETTINGS tab per the marketplace page. <!-- VERIFY -->

When in doubt, consult the extension's README (linked from the marketplace listing) for the version you have installed. The docs/marketplace citations in this file are a point-in-time snapshot.

For workflows authored in Go, Python, or Java, skip the extension and use the SDK replayer under your IDE's native debugger. The observability is the same — a stack trace at the point the SDK detected divergence — and the replayer APIs are first-party and doc-backed. <!-- docs/develop/go/best-practices/testing-suite.mdx:477-479 --><!-- docs/develop/python/best-practices/testing-suite.mdx:198-208 --><!-- docs/develop/java/best-practices/testing-suite.mdx:589, 597 -->
