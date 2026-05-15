# Batch and Lifecycle Operations

Bulk and lifecycle CLI commands for managing Workflow Executions, Batch Jobs,
Schedules, and Activity completions.

---

## Workflow Cancel

Canceling records a `WorkflowExecutionCancelRequested` event in the Event
History. The Service schedules a new Command Task, and the Workflow Execution
performs any cleanup work supported by its implementation. <!-- docs/cli/workflow.mdx:52-55 -->

### Single Workflow

```
temporal workflow cancel \
    --workflow-id YourWorkflowId
```
<!-- docs/cli/workflow.mdx:60-61 -->

### Bulk via visibility Query

```
temporal workflow cancel \
    --query YourQuery
```
<!-- docs/cli/workflow.mdx:68-69 -->

### Flags

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:84 -->
| `--run-id`, `-r` | No | string | Run ID. Only use with `--workflow-id`. Cannot use with `--query`. | <!-- docs/cli/workflow.mdx:83 -->
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:80 -->
| `--reason` | No | string | Reason for batch operation. Only use with `--query`. Defaults to user name. | <!-- docs/cli/workflow.mdx:81 -->
| `--rps` | No | float | Limit batch's requests per second. Only allowed if query is present. | <!-- docs/cli/workflow.mdx:82 -->
| `--yes`, `-y` | No | bool | Don't prompt to confirm. Only allowed when `--query` is present. | <!-- docs/cli/workflow.mdx:85 -->
| `--headers` | No | string[] | Workflow headers in `KEY=VALUE` format. | <!-- docs/cli/workflow.mdx:79 -->

---

## Workflow Terminate

Termination records a `WorkflowExecutionTerminated` event as the closing Event
in the Workflow Execution's history. Workflow code cannot see or respond to
terminations. To perform clean-up work in Workflow code, use
`temporal workflow cancel` instead. <!-- docs/cli/workflow.mdx:653-664 -->

### Single Workflow

```
temporal workflow terminate \
    --reason YourReasonForTermination \
    --workflow-id YourWorkflowId
```
<!-- docs/cli/workflow.mdx:646-649 -->

### Bulk via visibility Query

```
temporal workflow terminate \
    --query YourQuery \
    --reason YourReasonForTermination
```
<!-- docs/cli/workflow.mdx:657-660 -->

### Flags

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:677 -->
| `--run-id`, `-r` | No | string | Run ID. Can only be set with `--workflow-id`. Do not use with `--query`. | <!-- docs/cli/workflow.mdx:676 -->
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:673 -->
| `--reason` | No | string | Reason for termination. Defaults to message with the current user's name. | <!-- docs/cli/workflow.mdx:674 -->
| `--rps` | No | float | Limit batch's requests per second. Only allowed if query is present. | <!-- docs/cli/workflow.mdx:675 -->
| `--yes`, `-y` | No | bool | Don't prompt to confirm termination. Can only be used with `--query`. | <!-- docs/cli/workflow.mdx:678 -->

---

## Workflow Delete

Delete a Workflow Execution and its Event History. The removal executes
asynchronously. If the Execution is Running, the Service terminates it before
deletion. <!-- docs/cli/workflow.mdx:108-116 -->

### Single Workflow

```
temporal workflow delete \
    --workflow-id YourWorkflowId
```
<!-- docs/cli/workflow.mdx:110-111 -->

### Bulk via visibility Query

Uses `--query` in place of `--workflow-id`, same as cancel/terminate.

### Flags

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:130 -->
| `--run-id`, `-r` | No | string | Run ID. Only use with `--workflow-id`. Cannot use with `--query`. | <!-- docs/cli/workflow.mdx:129 -->
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/workflow.mdx:126 -->
| `--reason` | No | string | Reason for batch operation. Only use with `--query`. Defaults to user name. | <!-- docs/cli/workflow.mdx:127 -->
| `--rps` | No | float | Limit batch's requests per second. Only allowed if query is present. | <!-- docs/cli/workflow.mdx:128 -->
| `--yes`, `-y` | No | bool | Don't prompt to confirm. Only allowed when `--query` is present. | <!-- docs/cli/workflow.mdx:131 -->
| `--headers` | No | string[] | Workflow headers in `KEY=VALUE` format. | <!-- docs/cli/workflow.mdx:125 -->

---

## Workflow Reset

Reset a Workflow Execution so it can resume from a point in its Event History
without losing its progress up to that point. <!-- docs/cli/workflow.mdx:369-370 -->

### By Event ID

```
temporal workflow reset \
    --workflow-id YourWorkflowId \
    --event-id YourLastEvent
```
<!-- docs/cli/workflow.mdx:373-375 -->

### By reset type

```
temporal workflow reset \
    --workflow-id YourWorkflowId \
    --type LastContinuedAsNew
```
<!-- docs/cli/workflow.mdx:380-382 -->

### Batch resets

For batch resets, limit your resets to `FirstWorkflowTask`, `LastWorkflowTask`,
or `BuildId`. Do not use Workflow IDs, run IDs, or event IDs with this
command. <!-- docs/cli/workflow.mdx:386-387 -->

<!-- RESOLVED (genuinely ambiguous): docs/cli/workflow.mdx has no flags table for
     `temporal workflow reset` itself (only for the `with-workflow-update-options`
     subcommand). The flags exist in CLI help output but are not in the docs
     reference page. Flag details omitted to avoid fabrication. -->

### Subcommand: reset with-workflow-update-options

Run Workflow Update Options atomically after the Workflow is
reset. <!-- docs/cli/workflow.mdx:394-395 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--versioning-override-behavior` | Yes | string-enum | Override the versioning behavior. Accepted values: `pinned`, `auto_upgrade`. | <!-- docs/cli/workflow.mdx:401 -->
| `--versioning-override-build-id` | No | string | Build ID of the version to target (when `pinned`). | <!-- docs/cli/workflow.mdx:402 -->
| `--versioning-override-deployment-name` | No | string | Deployment Name of the version to target (when `pinned`). | <!-- docs/cli/workflow.mdx:403 -->

---

## Batch Job Management (`temporal batch`)

Batch jobs are created implicitly when you pass `--query` to
`temporal workflow cancel`, `temporal workflow terminate`,
`temporal workflow signal`, or `temporal workflow delete`.
The `temporal batch` commands let you inspect and manage those
jobs. <!-- docs/cli/workflow.mdx:73, 80-82 -->

### batch describe

Show the progress of an ongoing batch job: <!-- docs/cli/batch.mdx:26-27 -->

```
temporal batch describe \
    --job-id YourJobId
```
<!-- docs/cli/batch.mdx:31-32 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--job-id` | Yes | string | Batch job ID. | <!-- docs/cli/batch.mdx:39 -->

### batch list

Return a list of batch jobs on the Service or within a single
Namespace: <!-- docs/cli/batch.mdx:42-43 -->

```
temporal batch list \
    --namespace YourNamespace
```
<!-- docs/cli/batch.mdx:47-48 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--limit` | No | int | Maximum number of batch jobs to display. | <!-- docs/cli/batch.mdx:55 -->

### batch terminate

Terminate a batch job. You must provide a reason; the Service stores it as
metadata for the termination event: <!-- docs/cli/batch.mdx:59-61 -->

```
temporal batch terminate \
    --job-id YourJobId \
    --reason YourTerminationReason
```
<!-- docs/cli/batch.mdx:64-66 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--job-id` | Yes | string | Job ID to terminate. | <!-- docs/cli/batch.mdx:73 -->
| `--reason` | Yes | string | Reason for terminating the batch job. | <!-- docs/cli/batch.mdx:74 -->

---

## Schedule Operations (`temporal schedule`)

### schedule create

Create a new Schedule that automatically starts Workflow Executions at
specified times. Supports `--calendar`, `--interval`, and
`--cron`. <!-- docs/cli/schedule.mdx:82-97 -->

```
temporal schedule create \
    --schedule-id "YourScheduleId" \
    --calendar '{"dayOfWeek":"Fri","hour":"3","minute":"30"}' \
    --workflow-id YourBaseWorkflowIdName \
    --task-queue YourTaskQueue \
    --type YourWorkflowType
```
<!-- docs/cli/schedule.mdx:88-93 -->

Timing specifications: <!-- docs/cli/schedule.mdx:96-104 -->

- `--interval` shorthand: e.g. `45m` (every 45 min) or `6h/5h` (every 6h, offset 5h)
- `--calendar` JSON: e.g. `{"dayOfWeek":"Fri","hour":"17","minute":"5"}`
- `--cron` Unix-style: e.g. `"30 12 * * Fri"` or robfig `@daily`/`@every X`

Key flags (selected; see docs for full table): <!-- docs/cli/schedule.mdx:109-143 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. |
| `--type` | Yes | string | Workflow Type name. |
| `--task-queue`, `-t` | Yes | string | Workflow Task queue. |
| `--calendar` | No | string[] | Calendar specification in JSON. |
| `--interval` | No | string[] | Interval duration (e.g. `90m`, `60m/15m`). |
| `--cron` | No | string[] | Calendar specification in cron string format. |
| `--overlap-policy` | No | string-enum | Accepted values: `Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll`. |
| `--catchup-window` | No | duration | Maximum catch-up time for when the Service is unavailable. |
| `--jitter` | No | duration | Max difference in time from the specification. |
| `--time-zone` | No | string | Interpret calendar specs with the `TZ` time zone. |
| `--paused` | No | bool | Pause the Schedule immediately on creation. |
| `--pause-on-failure` | No | bool | Pause schedule after Workflow failures. |
| `--remaining-actions` | No | int | Total allowed actions. Default is zero (unlimited). |
| `--notes` | No | string | Initial notes field value. |
| `--start-time` | No | timestamp | Schedule start time. |
| `--end-time` | No | timestamp | Schedule end time. |
| `--workflow-id`, `-w` | No | string | Workflow ID. If not supplied, the Service generates a unique ID. |
| `--schedule-memo` | No | string[] | Set schedule memo using `KEY="VALUE"` pairs. |
| `--schedule-search-attribute` | No | string[] | Set schedule Search Attributes using `KEY="VALUE"` pairs. |

### schedule describe

Show a Schedule configuration, including information about past, current, and
future Workflow runs: <!-- docs/cli/schedule.mdx:164-167 -->

```
temporal schedule describe \
    --schedule-id YourScheduleId
```
<!-- docs/cli/schedule.mdx:170-171 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. | <!-- docs/cli/schedule.mdx:178 -->

### schedule list

Lists the Schedules hosted by a Namespace: <!-- docs/cli/schedule.mdx:182-183 -->

```
temporal schedule list \
    --namespace YourNamespace
```
<!-- docs/cli/schedule.mdx:185-186 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--long`, `-l` | No | bool | Show detailed information. | <!-- docs/cli/schedule.mdx:193 -->
| `--query`, `-q` | No | string | Filter results using given List Filter. | <!-- docs/cli/schedule.mdx:194 -->
| `--really-long` | No | bool | Show extensive information in non-table form. | <!-- docs/cli/schedule.mdx:195 -->

### schedule toggle (pause/unpause)

Pause or unpause a Schedule: <!-- docs/cli/schedule.mdx:199-200 -->

```
temporal schedule toggle \
    --schedule-id "YourScheduleId" \
    --pause \
    --reason "YourReason"
```
<!-- docs/cli/schedule.mdx:202-205 -->

```
temporal schedule toggle \
    --schedule-id "YourScheduleId" \
    --unpause \
    --reason "YourReason"
```
<!-- docs/cli/schedule.mdx:211-214 -->

The `--reason` text updates the Schedule's `notes` field for operations
communication. It defaults to `"(no reason provided)"` if
omitted. <!-- docs/cli/schedule.mdx:217-219 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. | <!-- docs/cli/schedule.mdx:227 -->
| `--pause` | No | bool | Pause the Schedule. | <!-- docs/cli/schedule.mdx:225 -->
| `--unpause` | No | bool | Unpause the Schedule. | <!-- docs/cli/schedule.mdx:228 -->
| `--reason` | No | string | Reason for pausing or unpausing. | <!-- docs/cli/schedule.mdx:226 -->

### schedule trigger

Trigger a Schedule to run immediately: <!-- docs/cli/schedule.mdx:232-233 -->

```
temporal schedule trigger \
    --schedule-id "YourScheduleId"
```
<!-- docs/cli/schedule.mdx:235-236 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. | <!-- docs/cli/schedule.mdx:244 -->
| `--overlap-policy` | No | string-enum | Accepted values: `Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll`. | <!-- docs/cli/schedule.mdx:243 -->

### schedule update

Full replacement of Schedule configuration. Any options not provided will be
reset to their default values. Re-specify all options, not just the ones you
want to change. Use `temporal schedule describe` before updating. <!-- docs/cli/schedule.mdx:257-261 -->

Schedule memo and search attributes cannot be updated with this command. They
are set only during Schedule creation. <!-- docs/cli/schedule.mdx:263-265 -->

```
temporal schedule update \
    --schedule-id "YourScheduleId" \
    --workflow-type "NewWorkflowType"
```
<!-- docs/cli/schedule.mdx:252-254 -->

Key flags match `schedule create` (same timing, policy, and workflow options)
with `--schedule-id` required. See `docs/cli/schedule.mdx` lines 269-302 for
the full table. <!-- docs/cli/schedule.mdx:269-302 -->

### schedule delete

Deletes a Schedule. Removing a Schedule will not affect Workflow Executions it
started that are still running. To cancel or terminate those, use
`temporal workflow delete` with the `TemporalScheduledById` Search
Attribute. <!-- docs/cli/schedule.mdx:147-157 -->

```
temporal schedule delete \
    --schedule-id YourScheduleId
```
<!-- docs/cli/schedule.mdx:150-151 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. | <!-- docs/cli/schedule.mdx:162 -->

### schedule backfill

Batch-execute actions that would have run during a specified time interval.
Use to fill in runs from when a Schedule was paused, before a Schedule was
created, from the future, or to re-process a previously executed
interval. <!-- docs/cli/schedule.mdx:34-37 -->

```
temporal schedule backfill \
    --schedule-id "YourScheduleId" \
    --start-time "2022-05-01T00:00:00Z" \
    --end-time "2022-05-31T23:59:59Z" \
    --overlap-policy BufferAll
```
<!-- docs/cli/schedule.mdx:46-50 -->

Overlap policies: <!-- docs/cli/schedule.mdx:53-69 -->

- **AllowAll** -- Unlimited concurrent executions. Speeds up backfill.
- **BufferAll** -- Buffer incoming while waiting for running to complete.
- **Skip** -- If previous still running, discard new.
- **BufferOne** -- Like Skip but buffer one to run after previous completes.
- **CancelOther** -- Cancel running and replace with incoming.
- **TerminateOther** -- Terminate running and replace with incoming.

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--schedule-id`, `-s` | Yes | string | Schedule ID. | <!-- docs/cli/schedule.mdx:77 -->
| `--start-time` | Yes | timestamp | Backfill start time. | <!-- docs/cli/schedule.mdx:78 -->
| `--end-time` | Yes | timestamp | Backfill end time. | <!-- docs/cli/schedule.mdx:75 -->
| `--overlap-policy` | No | string-enum | Accepted values: `Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll`. | <!-- docs/cli/schedule.mdx:76 -->

---

## Activity Complete

Complete an Activity, marking it as successfully finished. <!-- docs/cli/activity.mdx:62-63 -->

```
temporal activity complete \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId \
    --result '{"YourResultKey": "YourResultVal"}'
```
<!-- docs/cli/activity.mdx:66-69 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | Yes | string | Activity ID. May be a Workflow-invoked Activity or a Standalone Activity. | <!-- docs/cli/activity.mdx:76 -->
| `--result` | Yes | string | Result JSON to return. | <!-- docs/cli/activity.mdx:77 -->
| `--workflow-id`, `-w` | No | string | Workflow ID. Required for workflow Activities. Omit for Standalone Activities. | <!-- docs/cli/activity.mdx:79 -->
| `--run-id`, `-r` | No | string | Run ID. For workflow Activities, this is the Workflow Run ID. For Standalone Activities, this is the Activity Run ID. | <!-- docs/cli/activity.mdx:78 -->

---

## Activity Fail

Fail an Activity, marking it as having encountered an
error: <!-- docs/cli/activity.mdx:162-163 -->

```
temporal activity fail \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId
```
<!-- docs/cli/activity.mdx:166-167 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | Yes | string | Activity ID. May be a Workflow-invoked Activity or a Standalone Activity. | <!-- docs/cli/activity.mdx:174 -->
| `--detail` | No | string | Failure detail (JSON). Attached as the failure details payload. | <!-- docs/cli/activity.mdx:175 -->
| `--reason` | No | string | Failure reason. Attached as the failure message. | <!-- docs/cli/activity.mdx:176 -->
| `--workflow-id`, `-w` | No | string | Workflow ID. Required for workflow Activities. Omit for Standalone Activities. | <!-- docs/cli/activity.mdx:178 -->
| `--run-id`, `-r` | No | string | Run ID. For workflow Activities, this is the Workflow Run ID. For Standalone Activities, this is the Activity Run ID. | <!-- docs/cli/activity.mdx:177 -->

---

## Activity Pause

Pause an Activity. Not supported for Standalone
Activities. <!-- docs/cli/activity.mdx:200-202 -->

If the Activity is not currently running, it will not be run again until
unpaused. If the Activity is currently running, it will run until the next time
it fails, completes, or times out, at which point the pause kicks
in. <!-- docs/cli/activity.mdx:204-207 -->

If the Activity is on its last retry attempt and fails, the failure is returned
to the caller as if it had not been paused. <!-- docs/cli/activity.mdx:209-210 -->

```
temporal activity pause \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId
```
<!-- docs/cli/activity.mdx:216-218 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | No | string | The Activity ID to pause. Required. | <!-- docs/cli/activity.mdx:228 -->
| `--workflow-id`, `-w` | Yes | string | Workflow ID. | <!-- docs/cli/activity.mdx:232 -->
| `--run-id`, `-r` | No | string | Run ID. | <!-- docs/cli/activity.mdx:231 -->
| `--reason` | No | string | Reason for pausing the Activity. | <!-- docs/cli/activity.mdx:230 -->

---

## Activity Unpause

Re-schedule a previously paused Activity for execution. Not supported for
Standalone Activities. <!-- docs/cli/activity.mdx:382-384 -->

If the Activity is not running and is past its retry timeout, it will be
scheduled immediately. Otherwise, it will be scheduled after its retry timeout
expires. <!-- docs/cli/activity.mdx:386-388 -->

Use `--reset-attempts` to reset the number of previous run attempts to zero.
Use `--reset-heartbeats` to reset the Activity's
heartbeats. <!-- docs/cli/activity.mdx:390-395 -->

### Single Activity

```
temporal activity unpause \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId \
    --reset-attempts \
    --reset-heartbeats
```
<!-- docs/cli/activity.mdx:402-406 -->

### Bulk via visibility Query

```
temporal activity unpause \
    --query 'TemporalPauseInfo IS NOT NULL'
```
<!-- docs/cli/activity.mdx:411-412 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | No | string | Activity ID to unpause. Mutually exclusive with `--query`. Requires `--workflow-id`. | <!-- docs/cli/activity.mdx:420 -->
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/activity.mdx:429 -->
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. Must set either `--workflow-id` or `--query`. (Experimental) | <!-- docs/cli/activity.mdx:423 -->
| `--reset-attempts` | No | bool | Reset the activity attempts. | <!-- docs/cli/activity.mdx:425 -->
| `--reset-heartbeats` | No | bool | Reset the Activity's heartbeats. | <!-- docs/cli/activity.mdx:426 -->
| `--run-id`, `-r` | No | string | Run ID. Only use with `--workflow-id`. Cannot use with `--query`. | <!-- docs/cli/activity.mdx:428 -->
| `--reason` | No | string | Reason for batch operation. Only use with `--query`. Defaults to user name. | <!-- docs/cli/activity.mdx:424 -->
| `--jitter` | No | duration | Activity starts at random time within the specified duration. Only with `--query`. | <!-- docs/cli/activity.mdx:422 -->
| `--rps` | No | float | Limit batch's requests per second. Only allowed if query is present. | <!-- docs/cli/activity.mdx:427 -->
| `--yes`, `-y` | No | bool | Don't prompt to confirm. Only allowed when `--query` is present. | <!-- docs/cli/activity.mdx:430 -->

---

## Activity Reset

Reset an Activity. Not supported for Standalone Activities. This restarts the
Activity as if it were first being scheduled -- resets both the number of
attempts and the activity timeout. <!-- docs/cli/activity.mdx:235-240 -->

If the Activity is already paused, it will be unpaused by default. Provide
`--keep-paused` to prevent this. <!-- docs/cli/activity.mdx:248-252 -->

### Single Activity

```
temporal activity reset \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId \
    --keep-paused \
    --reset-heartbeats
```
<!-- docs/cli/activity.mdx:269-273 -->

### Bulk via visibility Query

```
temporal activity reset \
    --query 'WorkflowType="YourWorkflow"'
```
<!-- docs/cli/activity.mdx:279-280 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | No | string | Activity ID to reset. Mutually exclusive with `--query`. Requires `--workflow-id`. | <!-- docs/cli/activity.mdx:287 -->
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. | <!-- docs/cli/activity.mdx:298 -->
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. Must set either `--workflow-id` or `--query`. (Experimental) | <!-- docs/cli/activity.mdx:291 -->
| `--keep-paused` | No | bool | If the activity was paused, it will stay paused. | <!-- docs/cli/activity.mdx:290 -->
| `--reset-heartbeats` | No | bool | Reset the Activity's heartbeats. | <!-- docs/cli/activity.mdx:294 -->
| `--reset-attempts` | No | bool | Reset the activity attempts. | <!-- docs/cli/activity.mdx:293 -->
| `--restore-original-options` | No | bool | Restore the original options of the activity. | <!-- docs/cli/activity.mdx:295 -->
| `--jitter` | No | duration | Activity resets at random time within the specified duration. Only with `--query`. | <!-- docs/cli/activity.mdx:289 -->
| `--run-id`, `-r` | No | string | Run ID. Only use with `--workflow-id`. Cannot use with `--query`. | <!-- docs/cli/activity.mdx:297 -->
| `--reason` | No | string | Reason for batch operation. Only use with `--query`. Defaults to user name. | <!-- docs/cli/activity.mdx:292 -->
| `--rps` | No | float | Limit batch's requests per second. Only allowed if query is present. | <!-- docs/cli/activity.mdx:296 -->
| `--yes`, `-y` | No | bool | Don't prompt to confirm. Only allowed when `--query` is present. | <!-- docs/cli/activity.mdx:299 -->

---

## Activity Cancel

Request cancellation of a Standalone Activity. Transitions the Activity's run
state to CancelRequested. If the Activity is heartbeating, a cancellation error
will be raised when the next heartbeat response is received; if the Activity
allows this error to propagate, it transitions to canceled
status. <!-- docs/cli/activity.mdx:39-51 -->

```
temporal activity cancel \
    --activity-id YourActivityId
```
<!-- docs/cli/activity.mdx:43-44 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | Yes | string | Activity ID. | <!-- docs/cli/activity.mdx:56 -->
| `--reason` | No | string | Reason for cancellation. | <!-- docs/cli/activity.mdx:57 -->
| `--run-id`, `-r` | No | string | Activity Run ID. If not set, targets the latest run. | <!-- docs/cli/activity.mdx:58 -->

---

## Activity Terminate

Terminate a Standalone Activity. Activity code cannot see or respond to
terminations. <!-- docs/cli/activity.mdx:362-372 -->

```
temporal activity terminate \
    --activity-id YourActivityId \
    --reason YourReason
```
<!-- docs/cli/activity.mdx:366-368 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | Yes | string | Activity ID. | <!-- docs/cli/activity.mdx:377 -->
| `--reason` | No | string | Reason for termination. Defaults to a message with the current user's name. | <!-- docs/cli/activity.mdx:378 -->
| `--run-id`, `-r` | No | string | Activity Run ID. If not set, targets the latest run. | <!-- docs/cli/activity.mdx:379 -->

---

## Activity Update-Options

Update the options of a running Activity that were passed into it from a
Workflow. Updates are incremental, only changing specified options. Not
supported for Standalone Activities. <!-- docs/cli/activity.mdx:432-435 -->

### Single Activity

```
temporal activity update-options \
    --activity-id YourActivityId \
    --workflow-id YourWorkflowId \
    --task-queue NewTaskQueueName
```
<!-- docs/cli/activity.mdx:441-444 -->

### Bulk via visibility Query

```
temporal activity update-options \
    --query 'WorkflowType="YourWorkflow"' \
    --task-queue NewTaskQueueName
```
<!-- docs/cli/activity.mdx:462-464 -->

Key flags: <!-- docs/cli/activity.mdx:469-488 -->

| Flag | Req | Type | Description |
|------|-----|------|-------------|
| `--activity-id`, `-a` | No | string | Activity ID to update. Mutually exclusive with `--query`. Requires `--workflow-id`. |
| `--workflow-id`, `-w` | No | string | Workflow ID. Must set either `--workflow-id` or `--query`. |
| `--query`, `-q` | No | string | SQL-like `QUERY` List Filter. (Experimental for batch activity ops) |
| `--task-queue` | No | string | Name of the task queue for the Activity. |
| `--schedule-to-close-timeout` | No | duration | Max time for Activity Execution, including retries. |
| `--schedule-to-start-timeout` | No | duration | Max time an Activity task can stay in a task queue. |
| `--start-to-close-timeout` | No | duration | Max time for a single Activity attempt. |
| `--heartbeat-timeout` | No | duration | Max time between successful worker heartbeats. |
| `--retry-initial-interval` | No | duration | Interval of the first retry. |
| `--retry-maximum-interval` | No | duration | Maximum interval between retries. |
| `--retry-backoff-coefficient` | No | float | Coefficient for next retry interval. Must be >= 1. |
| `--retry-maximum-attempts` | No | int | Maximum number of attempts. 1 disables retries, 0 means unlimited. |
| `--restore-original-options` | No | bool | Restore the original options of the activity. |
| `--rps` | No | float | Limit batch's requests per second. Only with `--query`. |
| `--yes`, `-y` | No | bool | Don't prompt to confirm. Only with `--query`. |
