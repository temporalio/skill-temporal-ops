# Replay a Workflow Execution locally

This file is about the *tooling* that reproduces a recorded Workflow Execution in a debugger. For what replay divergence *means* as a concept — why the Worker Task fails, how the server classifies the cause, and how to remediate — see [non-determinism.md](non-determinism.md).

A Replay is "the method by which a Workflow Execution resumes making progress. During a Replay the Commands that are generated are checked against an existing Event History." <!-- docs/encyclopedia/workflow/workflow-execution/workflow-execution.mdx:71 --> Running a recorded history through a local replayer against your Worker source tree is the canonical way to reproduce a non-determinism error under a debugger, verify a fix, and pin a CI regression test.

The **SDK replayer** is the primary tool and the one to reach for in almost every case: it is documented for every supported SDK, runs headless, attaches to any debugger, and drops into CI as a regression guard. The **VS Code extension** is a TypeScript-only convenience wrapper around the same replayer — covered briefly at the end.

Out of scope (link, don't absorb):

- What non-determinism means, how to identify it in an Event History, and remediation options → [non-determinism.md](non-determinism.md)
- Worker not polling the Task Queue at all — nothing to replay, fix the Worker first → [worker-health.md](worker-health.md)
- Can't reach the server to fetch the history → [connectivity.md](connectivity.md), [authentication.md](authentication.md)
- The bottom-up layer model for routing between files → [diagnostic-ladder.md](diagnostic-ladder.md)

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1 — Export the Event History](#step-1--export-the-event-history)
- [Step 2 — Run the SDK replayer (all supported SDKs)](#step-2--run-the-sdk-replayer-all-supported-sdks)
- [TEMPORAL_DEBUG: suppress the deadlock detector while stepping](#temporal_debug-suppress-the-deadlock-detector-while-stepping)
- [Interpreting a replay that diverges](#interpreting-a-replay-that-diverges)
- [Interpreting a replay that succeeds](#interpreting-a-replay-that-succeeds)
- [The VS Code extension (TypeScript only, interactive)](#the-vs-code-extension-typescript-only-interactive)

## Prerequisites

- The Workflow source tree at the commit that was deployed when the recorded Workflow Execution ran. Replaying current `main` against an older recording will produce divergence *for a different reason than the bug you're triaging*.
- The SDK installed and importable in that workspace (the replayer is part of the SDK, not a standalone binary).
- A way to fetch the Event History of the run — either the `temporal` CLI or the SDK client's history-fetch API.

## Step 1 — Export the Event History

Every replayer takes the same input: a JSON Event History. Export it with:

```bash
temporal workflow show \
    --workflow-id YourWorkflowId \
    --run-id YourRunId \
    --output json > history.json
```

`--output` accepts `text, json, jsonl, none` (default `text`); replay requires `json`. <!-- docs/cli/command-reference/workflow.mdx:903 --> `--run-id` is optional; if omitted, the CLI targets the most recent run of the given Workflow ID. <!-- docs/cli/command-reference/workflow.mdx:445 -->

The docs document the replayer handoff explicitly: "When using JSON output (`--output json`), you may pass the results to an SDK to perform a replay." <!-- docs/cli/command-reference/workflow.mdx:429-430 -->

If the run id is unknown, find it via `temporal workflow describe --workflow-id YourWorkflowId` (see [workflow-stuck.md](workflow-stuck.md)) or the Web UI. The testing-suite pages note histories can also be obtained "from the Web UI or the Temporal CLI." <!-- docs/develop/typescript/best-practices/testing-suite.mdx:531 --><!-- docs/develop/python/best-practices/testing-suite.mdx:213 -->

For Cloud or any non-default target, add the usual connection flags (`--address`, `--namespace`, and either `--api-key` or the mTLS `--tls-*` flags). See [authentication.md](authentication.md).

## Step 2 — Run the SDK replayer (all supported SDKs)

Each SDK's testing-suite page documents a replayer. Names and signatures below are transcribed from those pages; they are what the docs state, not what the runtime type system exports at any given version. All run headless — attach your IDE's debugger of choice to the test process, and use the bulk variant to pin a CI regression test.

### Go

```go
replayer := worker.NewWorkflowReplayer()
replayer.RegisterWorkflow(YourWorkflow)
err := replayer.ReplayWorkflowHistory(nil, hist)
```

<!-- docs/develop/go/best-practices/testing-suite.mdx:639-641 --> Use `worker.WorkflowReplayer` to "replay an existing Workflow Execution from its Event History to replicate errors." <!-- docs/develop/go/best-practices/testing-suite.mdx:595 --> "If a noticeably different code path was followed or some code caused a deadlock, it will be returned in the error code." <!-- docs/develop/go/best-practices/testing-suite.mdx:646 -->

Attach Delve / your IDE debugger to the test process that calls `ReplayWorkflowHistory`. Set `TEMPORAL_DEBUG=true` while stepping (see below).

### Python

```python
replayer = Replayer(workflows=[YourWorkflow])
await replayer.replay_workflow(WorkflowHistory.from_json(history_json_str))
```

<!-- docs/develop/python/best-practices/testing-suite.mdx:204-205 --> For bulk replay, use `replayer.replay_workflows(histories)`. <!-- docs/develop/python/best-practices/testing-suite.mdx:198 --> "If any replay fails, the code raises an exception." <!-- docs/develop/python/best-practices/testing-suite.mdx:190 --> The docs note a history-encoding pitfall: "the data can be protobuf-encoded (`bytes`). The `Replayer`, however, often works with decoded histories (like a `dict`)." <!-- docs/develop/python/best-practices/testing-suite.mdx:216-218 -->

Attach pdb / your IDE debugger to the test process.

### TypeScript

```ts
const history = JSON.parse(await fs.promises.readFile('./history.json', 'utf8'));
await Worker.runReplayHistory(
  { workflowsPath: require.resolve('./your/workflows') },
  history,
);
```

<!-- docs/develop/typescript/best-practices/testing-suite.mdx:533-541 --> For bulk replay, use `Worker.runReplayHistories`. <!-- docs/develop/typescript/best-practices/testing-suite.mdx:570 -->

Two error classes are documented: "When an Event History is replayed and non-determinism is detected (that is, the Workflow code is incompatible with the History), `DeterminismViolationError` is thrown. If replay fails for any other reason, `ReplayError` is thrown." <!-- docs/develop/typescript/best-practices/testing-suite.mdx:528-529 -->

### Java

```java
File file = new File("history.json");
WorkflowReplayer.replayWorkflowExecution(file, MyWorkflow.class);
```

<!-- docs/develop/java/best-practices/testing-suite.mdx:759-760 --> Use `WorkflowReplayer` from the `temporal-testing` package. <!-- docs/develop/java/best-practices/testing-suite.mdx:722 --> For bulk replay, `WorkflowReplayer.replayWorkflowExecutions`. <!-- docs/develop/java/best-practices/testing-suite.mdx:752-753 --> "In both examples, if Event History is non-deterministic, an error is thrown. You can choose to wait until all histories have been replayed with `replayWorkflowExecutions` by setting the `failFast` argument to `false`." <!-- docs/develop/java/best-practices/testing-suite.mdx:763-764 -->

### .NET

```csharp
var replayer = new WorkflowReplayer(
    new WorkflowReplayerOptions().AddWorkflow<MyWorkflow>());
await replayer.ReplayWorkflowAsync(
    WorkflowHistory.FromJson("my-workflow-id", historyJson));
```

<!-- docs/develop/dotnet/best-practices/testing-suite.mdx:280-282 --> For bulk replay, iterate `replayer.ReplayWorkflowsAsync(...)` and check each `result.ReplayFailure`. <!-- docs/develop/dotnet/best-practices/testing-suite.mdx:302-306 -->

### Ruby

```ruby
replayer = Temporalio::Worker::WorkflowReplayer.new(workflows: [MyWorkflow])
replayer.replay_workflow(history)
```

<!-- docs/develop/ruby/best-practices/testing-suite.mdx:242-245 --> For bulk replay, pass `client.list_workflows(...)` to `replayer.replay_workflows(...)`; set `raise_on_replay_failure: true`, or inspect each `result.replay_failure`. <!-- docs/develop/ruby/best-practices/testing-suite.mdx:259-262 -->

### PHP

The replayer is `\Temporal\Testing\Replay\WorkflowReplayer`. <!-- docs/develop/php/best-practices/testing-suite.mdx:202 --> Replay from a running server with `replayFromServer(...)`, <!-- docs/develop/php/best-practices/testing-suite.mdx:223 --> from an exported JSON file with `replayFromJSON(...)`, <!-- docs/develop/php/best-practices/testing-suite.mdx:236 --> or from an in-memory history with `replayHistory($history)`. <!-- docs/develop/php/best-practices/testing-suite.mdx:250 --> A non-deterministic replay throws `\Temporal\Testing\Replay\Exception\ReplayerException`. <!-- docs/develop/php/best-practices/testing-suite.mdx:227 -->

## TEMPORAL_DEBUG: suppress the deadlock detector while stepping

Two SDKs explicitly document a debug-mode env var. Without it, pausing on a breakpoint for more than a second can cause the Worker's deadlock detector to fail the Workflow Task *during your debugging session*:

- **Go.** "The Temporal Go SDK includes deadlock detection which fails a Workflow Task in case the code blocks over a second without relinquishing execution control. Because of this you can often encounter a `PanicError: Potential deadlock detected` while stepping through Workflow Definitions during debugging. To alleviate this issue, you can set the `TEMPORAL_DEBUG` environment variable to `true` before debugging your Workflow Definition." <!-- docs/develop/go/best-practices/debugging.mdx:28-31, 315-318 -->
- **Java.** "The Temporal Java SDK includes deadlock detection which fails a Workflow Task in case the code blocks over a second without relinquishing execution control. Because of this you can often encounter the `PotentialDeadlockException` Exception while stepping through Workflow code during debugging. To alleviate this issue, you can set the `TEMPORAL_DEBUG` environment variable to true before debugging your Workflow code." <!-- docs/develop/java/best-practices/debugging.mdx:28-31 -->

Both pages add the same warning: "Make sure to set `TEMPORAL_DEBUG` to true only during debugging." <!-- docs/develop/go/best-practices/debugging.mdx:35, 322 --><!-- docs/develop/java/best-practices/debugging.mdx:31 -->

Python and TypeScript debugging pages do not document a `TEMPORAL_DEBUG` env var in the sources consulted. <!-- docs/develop/python/best-practices/debugging.mdx --><!-- docs/develop/typescript/best-practices/debugging.mdx --><!-- VERIFY: whether the Python and TypeScript SDKs honor a `TEMPORAL_DEBUG` env var is not stated on their debugging pages. If your debugger sessions are being killed by a deadlock detector in those SDKs, check the SDK's source for an equivalent flag. -->

## Interpreting a replay that diverges

The documented behavior when replay detects non-determinism:

- **TypeScript**: throws `DeterminismViolationError`; any other replay failure throws `ReplayError`. <!-- docs/develop/typescript/best-practices/testing-suite.mdx:528-529 -->
- **Go**: the replayer returns an error from `ReplayWorkflowHistory`; the docs describe the condition as "cause the Workflow to fail with a nondeterminism error" without pinning a public type name. <!-- docs/develop/go/workflows/versioning.mdx:70 --><!-- VERIFY: exact Go error type is not spelled out in the docs snapshot. -->
- **Java**: `WorkflowReplayer.replayWorkflowExecution` throws; the versioning doc describes the condition as "This would cause the Workflow to fail with a nondeterminism error." <!-- docs/develop/java/workflows/versioning.mdx:71 --><!-- VERIFY: exact Java exception class is not stated in the docs snapshot. -->
- **Python**: `Replayer.replay_workflow` raises; "If any replay fails, the code raises an exception." <!-- docs/develop/python/best-practices/testing-suite.mdx:190 -->

Under a debugger — whether the VS Code extension or a native IDE attach on the SDK replayer — the process halts where the SDK throws. The stack frame where execution stops is *the Worker machinery that detected the mismatch*, not the Workflow line that emitted the bad Command. To locate the offending Workflow line, compare:

- The last Command the code was about to emit (the frame just below the SDK entry in the stack), and
- The next non-bookkeeping Event in the recorded history (see [non-determinism.md §Identifying ND from the Event History](non-determinism.md#identifying-nd-from-the-event-history)).

The divergence is the mismatch between those two. The encyclopedia frames this as: "If a generated Command doesn't match what it needs to in the existing Event History, then the Workflow Execution returns a non-deterministic error." <!-- docs/encyclopedia/workflow/workflow-definition.mdx:231 -->

For remediation paths (Worker Versioning, per-SDK patching, reset past the divergence), return to [non-determinism.md §Remediation](non-determinism.md#remediation-worker-versioning-preferred).

## Interpreting a replay that succeeds

If replay succeeds locally against the checked-out source, but production Workers keep failing with the same Workflow Execution, the deployed Worker code is different from the local checkout. Find the deployed build (via Worker Versioning metadata if used, or your deploy system) and reproduce from that commit. See [non-determinism.md §Reproducing ND locally via replay](non-determinism.md#reproducing-nd-locally-via-replay).

A local-replay success on the *current* source with production still failing is also the signal that Worker Versioning would have prevented this class of incident. See [non-determinism.md §Remediation: Worker Versioning (preferred)](non-determinism.md#remediation-worker-versioning-preferred).

## The VS Code extension (TypeScript only, interactive)

Temporal publishes a VS Code extension that wraps the TypeScript SDK's replayer in the VS Code debugger UI — one-click "open history → set breakpoints → step through." For Workflows authored in any other SDK, skip it and use the SDK replayer above under your IDE's native debugger; the observability is the same (a stack trace at the point the SDK detected divergence) and the replayer APIs are first-party and doc-backed.

Claims in this section come from the extension's marketplace listing; the Temporal docs themselves do not document the extension. <!-- https://marketplace.visualstudio.com/items?itemName=temporal-technologies.temporalio, consulted 2026-04-21 --><!-- VERIFY: local /Users/joe/sap/documentation/docs/ was Grep'd for "vscode" and "VS Code" with zero hits on 2026-04-21. -->

- **Install:** search the VS Code Marketplace for "Temporal" and install the extension published by **Temporal Technologies Inc.** (`temporal-technologies.temporalio`). As of the listing consulted, it debugs **TypeScript workflows only**. <!-- marketplace -->
- **Configure:** the replayer is driven by a TypeScript entrypoint that calls `startDebugReplayer` with a `workflowsPath`; the extension reads the path from the `temporal.replayerEntrypoint` setting (default `src/debug-replayer.ts`). <!-- marketplace --><!-- VERIFY: copy the exact `startDebugReplayer` import path and option-object shape from the extension README at the version you install. -->
- **Run:** **Temporal: Open Panel** from the Command Palette → enter a Workflow ID (fetched from the configured server, default `localhost:7233`) or pick a history JSON file → **Start** → set breakpoints in Workflow source or on history events, then step through. For Cloud, supply the Cloud gRPC endpoint plus the credentials in [authentication.md](authentication.md) and [certificates.md](certificates.md). <!-- marketplace -->

For anything the marketplace page does not spell out (a supported-SDK matrix beyond TypeScript, the full command list, `launch.json` recipes, OS requirements, mTLS-vs-API-key support), consult the extension's README for the version you installed — the citations here are a point-in-time snapshot. <!-- VERIFY: these are not documented on the marketplace page at the snapshot date. -->
