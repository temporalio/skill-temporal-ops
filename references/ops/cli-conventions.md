# Temporal CLI conventions and command index

Cross-cutting conventions for the `temporal` data-plane CLI (`workflow`, `batch`,
`schedule`, `activity`), plus an index that routes each operation to the file
that owns its judgment.

> **Scope.** Data-plane commands are documented across several files (see the
> [command index](#operation--command-index) below); this file is *not* the home
> for any single command. It holds the cross-command rules that don't belong to
> one operation, and points at the owner file for everything else.

## Use `--help` for flags

This file does not reproduce full flag tables — run `temporal <command> --help`
for the exhaustive, version-current flag set:

```bash
temporal <command> --help          # e.g. temporal workflow reset --help
```

It *does* curate the behaviors that are destructive, non-obvious, or buried in
`--help`'s description prose, plus the cross-command rules that belong to no
single command. (When a curated fact is version-volatile, the *behavior* is
stated here and the exact flag spelling/values are left to `--help`.)

## Connection and identity

Every `temporal` command reads the same connection settings, as an environment
variable or a flag (the flag wins):

| Env var | Flag | Purpose |
|---|---|---|
| `TEMPORAL_ADDRESS` | `--address` | Frontend `host:port` (default `localhost:7233`). |
| `TEMPORAL_NAMESPACE` | `--namespace`, `-n` | Namespace (default `default`). |
| `TEMPORAL_API_KEY` | `--api-key` | API-key auth (implies TLS). |

- mTLS uses `TEMPORAL_TLS_CERT` / `TEMPORAL_TLS_KEY` (`--tls-cert-path` /
  `--tls-key-path`). The endpoint form differs by auth method — see
  [../triage/connectivity.md](../triage/connectivity.md).
- `--identity` records who ran a mutating command (default
  `temporal-cli:$USER@$HOST`) and shows up in Event History and audit logs. Set
  it explicitly in shared automation.

Full env-var list: [docs.temporal.io/cli/setup-cli](https://docs.temporal.io/cli/setup-cli).

## Output and formatting

- `--output`, `-o` — `text` (default), `json`, `jsonl`, `none`. Use `json`/`jsonl`
  for scripting and pipe to `jq`; the triage/health runbooks recommend JSON
  output whenever a step fans out over many results.
- `--time-format` — `relative` (default), `iso`, `raw`.
- Payload shorthand: JSON output renders payloads inline by default; pass
  `--no-json-shorthand-payloads` to emit the raw payload envelope instead.

## The `--query` ⇒ batch-job bridge

The one cross-command rule worth memorizing. Passing `--query` (a
[List Filter](workflow-health.md#1-list-filter-fundamentals)) in place of
`--workflow-id` to `temporal workflow cancel | terminate | signal | delete` does
**not** act inline — it starts an asynchronous **batch job** over every matching
Execution. (The `--query` form of `temporal activity reset | unpause` behaves the
same way.)

```bash
temporal workflow terminate \
    --query 'ExecutionStatus="Running" AND WorkflowType="<YourType>"' \
    --reason "<why>" \
    --rps <n> \        # throttle the batch; only valid with --query
    --yes              # skip the confirm prompt; only valid with --query
```

Inspect and manage the resulting jobs with `temporal batch`:

```bash
temporal batch list
temporal batch describe --job-id <JobId>
temporal batch terminate --job-id <JobId> --reason "<why>"   # stops the job, not the workflows it already acted on
```

`--reason`, `--rps`, and `--yes` are accepted only when `--query` is present. For
*which* List Filter to run, see [workflow-health.md](workflow-health.md).

**Destructive single-target commands run with no confirmation.** A single-target
`workflow cancel | terminate | delete | signal` (with `--workflow-id`) executes
immediately — the interactive prompt, and `--yes` to skip it, exist **only** on
the `--query` batch form above. Verify the target before running. And
`workflow delete` in a multi-region (global) Namespace removes the Execution from
**all replicas**; requests to a passive cluster are forwarded to the active one by
default — pass `--grpc-meta xdc-redirection=false` to target a passive cluster.

## Schedule time-spec forms

`temporal schedule create` (and `update`) accept any combination of three spec
flags (run `temporal schedule create --help` for the rest):

- `--interval` — shorthand duration, e.g. `45m`, or `6h/5h` (every 6h, offset 5h).
- `--calendar` — JSON, e.g. `{"dayOfWeek":"Fri","hour":"3","minute":"30"}`.
- `--cron` — Unix cron or robfig (`@daily`, `@every 1h`), e.g. `"30 12 * * Fri"`.

`--overlap-policy` takes one of six values (`Skip`, `BufferOne`, `BufferAll`,
`CancelOther`, `TerminateOther`, `AllowAll`); their semantics and the backfill
workflow live in [../triage/schedule-missed.md](../triage/schedule-missed.md).
Concept page: [docs.temporal.io/schedule](https://docs.temporal.io/schedule).

**Update/delete gotchas** (`--help` buries these in the command description):
`temporal schedule update` **fully replaces** the schedule spec — options you don't
pass reset to defaults, so `describe` first and re-specify everything; `memo` and
search attributes can't be changed after creation. `temporal schedule delete` does
**not** stop already-running Executions — terminate those separately (e.g.
`temporal workflow terminate` by `TemporalScheduledById`).

## Operation → command index

One row per common data-plane operation. The linked file owns the judgment (when
to run it, how to read the output); run `temporal <cmd> --help` for flags.

| Operation | Skeleton | Owner |
|---|---|---|
| Find / list / count unhealthy workflows | `temporal workflow list --query '<filter>'` | [workflow-health.md](workflow-health.md) |
| Inspect one workflow | `temporal workflow describe -w <id>` / `show` / `stack` | [workflow-health.md](workflow-health.md), [../triage/workflow-stuck.md](../triage/workflow-stuck.md) |
| Recover a stuck workflow (signal / cancel / terminate / reset) | `temporal workflow signal\|cancel\|terminate\|reset ...` | [../triage/workflow-stuck.md](../triage/workflow-stuck.md#recovery-commands) |
| Reset past a non-determinism divergence | `temporal workflow reset -w <id> --event-id <n>` | [../triage/non-determinism.md](../triage/non-determinism.md) |
| Bulk cancel / terminate / signal / delete | `--query` form → [batch bridge](#the---query--batch-job-bridge) | this file + [workflow-health.md](workflow-health.md) |
| Pause / unpause / reset a stuck activity | `temporal activity pause\|unpause\|reset ...` | [../triage/workflow-stuck.md](../triage/workflow-stuck.md#temporal-activity-pause--unpause--reset) |
| Complete / fail an activity externally | `temporal activity complete\|fail -a <id> -w <id>` | `temporal activity --help` |
| Schedule CRUD (create / update / toggle / trigger / delete) | `temporal schedule <sub> -s <id> ...` | this file ([spec forms](#schedule-time-spec-forms)) |
| Backfill / diagnose missed schedule actions | `temporal schedule backfill -s <id> ...` | [../triage/schedule-missed.md](../triage/schedule-missed.md) |
