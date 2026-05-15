# Missed Schedule Actions

When a Schedule does not start a Workflow Execution at its expected time, the Action was either skipped intentionally (paused, overlap policy, end time reached) or the Temporal Service could not take the Action within the Catchup Window. This guide covers the second case. <!-- docs/troubleshooting/schedule-missed-actions.mdx:21 -->

---

## Alert on missed catchup window

The Temporal Service emits a counter each time it skips a scheduled Action because it could not run it within the configured Catchup Window. Alert on any non-zero value. <!-- docs/troubleshooting/schedule-missed-actions.mdx:24-25 -->

### Temporal Cloud

Alert on `temporal_cloud_v1_schedule_missed_catchup_window_count` grouped by `temporal_namespace`. <!-- docs/troubleshooting/schedule-missed-actions.mdx:29 -->

Example PromQL:

```promql
sum by (temporal_namespace) (
  increase(temporal_cloud_v1_schedule_missed_catchup_window_count[5m])
) > 0
```
<!-- docs/troubleshooting/schedule-missed-actions.mdx:33-36 -->

### Self-hosted

Alert on `schedule_missed_catchup_window` grouped by `namespace`. <!-- docs/troubleshooting/schedule-missed-actions.mdx:41 -->

Example PromQL:

```promql
sum by (namespace) (
  increase(schedule_missed_catchup_window[5m])
) > 0
```
<!-- docs/troubleshooting/schedule-missed-actions.mdx:45-48 -->

The metric is scoped to the Namespace, not to individual Schedules. A non-zero value tells you that at least one Schedule in the Namespace missed an Action, but not which one. <!-- docs/troubleshooting/schedule-missed-actions.mdx:51 -->

---

## Investigate which Schedule missed an Action

### Step 1: List Schedules in the Namespace

```bash
temporal schedule list --namespace <your-namespace>
```
<!-- docs/troubleshooting/schedule-missed-actions.mdx:62-63 -->

`ListSchedules` returns Schedule IDs and summary information. It does not return per-Schedule miss counters; use it only to produce the set of Schedule IDs to inspect. <!-- docs/troubleshooting/schedule-missed-actions.mdx:65 -->

### Step 2: Describe each Schedule

```bash
temporal schedule describe \
  --schedule-id <your-schedule-id> \
  --namespace <your-namespace>
```
<!-- docs/troubleshooting/schedule-missed-actions.mdx:72-74 -->

`DescribeSchedule` returns full Schedule state, including the `info` block with cumulative counters. <!-- docs/troubleshooting/schedule-missed-actions.mdx:77 -->

Relevant fields: <!-- docs/troubleshooting/schedule-missed-actions.mdx:79-87 -->

| Field | Meaning |
|---|---|
| `missedCatchupWindow` | Actions skipped because they could not run within the Catchup Window. Non-zero identifies the Schedule responsible for the alert. |
| `overlapSkipped` | Actions skipped because the previous run was still in progress and the Overlap Policy is `Skip`. |
| `bufferDropped` | Buffered Actions dropped because the buffer was full under `BufferOne` or `BufferAll`. |
| `bufferSize` | Current depth of the Action buffer. |
| `recentActions` | Most recent Action times and results. |
| `runningWorkflows` | Workflow Executions currently running for this Schedule. |

Scripting the fan-out against the JSON output (`temporal schedule describe -o json`) is usually faster than inspecting each Schedule interactively. <!-- docs/troubleshooting/schedule-missed-actions.mdx:88 -->

---

## Interpret the result

### Assess impact

- Compare `recentActions` to the Schedule's Spec to determine how many Actions were skipped and over what time period. <!-- docs/troubleshooting/schedule-missed-actions.mdx:96 -->
- If the Schedule uses the `Skip` Overlap Policy and the preceding run was long-running, the miss may reflect that run exceeding the Catchup Window, not a Service outage. <!-- docs/troubleshooting/schedule-missed-actions.mdx:97 -->
- For business-critical Schedules, Backfill the skipped interval once the underlying cause is resolved. <!-- docs/troubleshooting/schedule-missed-actions.mdx:98 -->

### Common root causes

- **Service or Namespace outage longer than the Catchup Window.** The default Catchup Window is **one year**, so a miss typically means the Schedule is configured with a tighter window (minimum ten seconds) and the outage exceeded it. <!-- docs/troubleshooting/schedule-missed-actions.mdx:102 -->
- **Namespace rate limiting.** If scheduled starts are throttled, Actions can queue past the Catchup Window. Cross-check `temporal_cloud_v1_schedule_rate_limited_count` (Cloud) or `schedule_rate_limited` (self-hosted) in the same time range. <!-- docs/troubleshooting/schedule-missed-actions.mdx:103 -->
- **Buffer overruns under `BufferAll`.** Long-running Workflow Executions under `BufferAll` can push buffered Actions past the Catchup Window. Cross-check `temporal_cloud_v1_schedule_buffer_overruns_count` (Cloud) or `schedule_buffer_overruns` (self-hosted) and examine `bufferSize`. <!-- docs/troubleshooting/schedule-missed-actions.mdx:104 -->

### Remediation

- Widen the Catchup Window if the current value is tighter than the Service's worst-case unavailability. The trade-off is more late Actions during recovery. <!-- docs/troubleshooting/schedule-missed-actions.mdx:108 -->
- Revisit the Overlap Policy if runs routinely exceed the Spec interval. `BufferAll` and `Skip` have different failure modes under sustained delay. <!-- docs/troubleshooting/schedule-missed-actions.mdx:109 -->
- Increase Namespace throughput limits if rate limiting is the contributing factor. <!-- docs/troubleshooting/schedule-missed-actions.mdx:110 -->
- Backfill the missed interval if the skipped Actions need to run. <!-- docs/troubleshooting/schedule-missed-actions.mdx:111 -->

---

## Backfill

Batch-execute actions that would have run during a specified time interval. Use `BufferAll` or `AllowAll` overlap policies for backfills to avoid skipping Workflow Executions. <!-- docs/cli/schedule.mdx:33-40 -->

```bash
temporal schedule backfill \
    --schedule-id "YourScheduleId" \
    --start-time "2022-05-01T00:00:00Z" \
    --end-time "2022-05-31T23:59:59Z" \
    --overlap-policy BufferAll
```
<!-- docs/cli/schedule.mdx:46-51 -->

---

## Overlap policies reference

The overlap policy controls what happens when a new scheduled Action fires while a previous run is still in progress. <!-- docs/cli/schedule.mdx:53-69 -->

| Policy | Behavior |
|---|---|
| `Skip` | If a previous Workflow Execution is still running, discard new Workflow Executions. <!-- docs/cli/schedule.mdx:61-62 --> |
| `BufferOne` | Same as Skip but buffer a single Workflow Execution to run after the previous completes. Discard others. <!-- docs/cli/schedule.mdx:63-65 --> |
| `BufferAll` | Buffer all incoming Workflow Executions while waiting for the running one to complete. <!-- docs/cli/schedule.mdx:59-60 --> |
| `CancelOther` | Cancel the running Workflow Execution and replace it with the incoming new one. <!-- docs/cli/schedule.mdx:66-67 --> |
| `TerminateOther` | Terminate the running Workflow Execution and replace it with the incoming new one. <!-- docs/cli/schedule.mdx:68-69 --> |
| `AllowAll` | Allow unlimited concurrent Workflow Executions. Significantly speeds up backfilling. Ensure running Workflows do not interfere with each other. <!-- docs/cli/schedule.mdx:55-58 --> |

There are exactly 6 overlap policies. <!-- docs/cli/schedule.mdx:76 -->

---

## Sibling skill pointers

- For metrics collection and alerting infrastructure, see the observability skill (planned: `skill-temporal-observability`).
- For Schedule CRUD operations (`temporal schedule create`, `temporal schedule update`, etc.), see `batch-and-lifecycle.md` in the ops reference files.
