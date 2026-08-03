# Temporal Cloud Capacity Modes

Quick-reference for Temporal Cloud capacity configuration: On-Demand vs Provisioned modes, APS/RPS/OPS definitions, TRUs, CLI commands, default limits, throttling, and APS management best practices.

---

## APS, RPS, and OPS

These three measures apply at different layers. Do not conflate them.

| Measure | Full Name | Scope | What It Measures |
|---------|-----------|-------|------------------|
| **APS** | Actions Per Second | Temporal Cloud Namespace | Rate of billable Actions (starting/signaling Workflows, scheduling Activities, etc.) <!-- docs/cloud/capacity-modes.mdx:50-53 --> |
| **RPS** | Requests Per Second | Temporal Service (Cloud and self-hosted) | Rate of gRPC requests to the Temporal Service <!-- docs/cloud/capacity-modes.mdx:55-57 --> |
| **OPS** | Operations Per Second | Temporal Cloud | Anything a user does directly, or Temporal does on behalf of the user, that produces load on Temporal Server <!-- docs/cloud/capacity-modes.mdx:59-60 --> |

APS is the higher-level, primary limit for Namespaces. RPS and OPS are lower-level measures to control and balance request rates at the service level. <!-- docs/cloud/capacity-modes.mdx:63-64 -->

---

## What Counts as an Action?

An Action is any billable operation within Temporal Cloud. Key categories: <!-- docs/evaluate/temporal-cloud/actions.mdx:24-26 -->

- **Workflow**: Starting, resetting, Continue-As-New, Child Workflow start, Search Attribute upsert <!-- docs/evaluate/temporal-cloud/actions.mdx:67-84 -->
- **Activity**: Starting, retrying, Heartbeating (only if the heartbeat reaches the Server) <!-- docs/evaluate/temporal-cloud/actions.mdx:109-119 -->
- **Timer**: Timer started (including implicit SDK timers from timeouts) <!-- docs/evaluate/temporal-cloud/actions.mdx:137-138 -->
- **Signal**: Every Signal sent (from Client or Workflow); one Action for Signal-With-Start regardless of whether the Workflow starts <!-- docs/evaluate/temporal-cloud/actions.mdx:145-148 -->
- **Query**: Every Query received by a Worker (`__temporal_workflow_metadata` excluded) <!-- docs/evaluate/temporal-cloud/actions.mdx:158-161 -->
- **Update**: Every accepted or rejected Update <!-- docs/evaluate/temporal-cloud/actions.mdx:170-173 -->
- **Schedule**: Each Schedule execution accrues 3 Actions (2 for Schedule start + 1 for the target Workflow start) <!-- docs/evaluate/temporal-cloud/actions.mdx:183-188 -->
- **Nexus**: Scheduling or canceling a Nexus Operation each counts as 1 Action on the caller Namespace <!-- docs/evaluate/temporal-cloud/actions.mdx:200-207 -->

Actions during Workflow Replay do **not** count. <!-- docs/evaluate/temporal-cloud/actions.mdx:44-46 -->

Actions excluded from APS calculations: Export, Capacity-related Actions. <!-- docs/cloud/capacity-modes.mdx:94-98 -->

---

## On-Demand Capacity

Default mode. Namespace capacity scales automatically based on trailing usage. <!-- docs/cloud/capacity-modes.mdx:100-102 -->

### Default limits

| Measure | Default Limit |
|---------|---------------|
| APS     | 500           |
| RPS     | 2,000         |
| OPS     | 4,000         |

<!-- docs/cloud/capacity-modes.mdx:105-107 -->

The limit never falls below the default value. <!-- docs/cloud/capacity-modes.mdx:119 -->

### Auto-scaling formula

Limit = the **greater** of: <!-- docs/cloud/capacity-modes.mdx:123-129 -->

1. Default limit (500 APS)
2. The **lesser** of:
   - 4 x APS Mean (over the past 7 days)
   - 2 x APS P90 (over the past 7 days)

<!-- docs/cloud/capacity-modes.mdx:110 -->

**Example**: If your average APS over 7 days is 200 and your P90 is 500: <!-- docs/cloud/capacity-modes.mdx:123-129 -->

- 4 x 200 = 800
- 2 x 500 = 1,000
- Lesser of those = 800
- Greater of 800 vs default 500 = **800 APS limit**

Under On-Demand you are only charged for the Actions you use. <!-- docs/cloud/capacity-modes.mdx:121 -->

---

## Provisioned Capacity

Lets you manually control Namespace limits by requesting Temporal Resource Units (TRUs). <!-- docs/cloud/capacity-modes.mdx:134-136 -->

### Per-TRU rates

| Measure | Per TRU |
|---------|---------|
| APS     | 500     |
| RPS     | 1,500   |
| OPS     | 4,000   |

<!-- docs/cloud/capacity-modes.mdx:138-140 -->

### Valid TRU counts

**2, 3, 4, 6, 8, 10, 12** -- subject to regional availability. <!-- docs/cloud/capacity-modes.mdx:142, 149 -->

TRUs can be adjusted hourly. <!-- docs/cloud/capacity-modes.mdx:142 -->

When TRUs are requested, Temporal aims to provision the additional capacity within two minutes. <!-- docs/cloud/capacity-modes.mdx:150 -->

For requests in excess of 4 TRUs in regions outside of the US, submit a support ticket to ensure capacity availability. <!-- docs/cloud/capacity-modes.mdx:153-156 -->

### When to use Provisioned Capacity

- Planned events (promotions, load testing, migrations) <!-- docs/cloud/capacity-modes.mdx:166-172 -->
- Unplanned events / usage spikes
- Known but sudden system spikes
- Load testing
- Migrating workloads

When switching back to On-Demand mode, your APS limit resets to the running average from the last 7 days. If Temporal Support has set a custom limit for your namespace, this limit is persisted across capacity mode changes. <!-- docs/best-practices/managing-aps-limits.mdx:208-212 -->

---

## Setting Capacity Modes

Capacity modes can be set and adjusted by **Global Admin** and **Namespace Admin**. <!-- docs/cloud/capacity-modes.mdx:179 -->

### CLI

Update capacity:

```
tcld namespace capacity update \
  --namespace <namespace_name> \
  --capacity-mode <on_demand|provisioned> \
  [--capacity-value <tru value>] \
  [--request-id <request_id>] \
  [--resource-version <resource-version>]
```

<!-- docs/cloud/tcld/namespace.mdx#update -->

- `--capacity-mode` (`--cm`): `on_demand` for automatic scaling, `provisioned` for fixed allocation. <!-- docs/cloud/capacity-modes.mdx:212-213 -->
- `--capacity-value` (`--cv`): throughput value in TRUs. Required and must be greater than 0 when `--capacity-mode` is `provisioned`; ignored for `on_demand`. <!-- docs/cloud/tcld/namespace.mdx#update -->
- `--request-id`: optional; server assigns one if not specified. <!-- docs/cloud/capacity-modes.mdx:218 -->
- `--resource-version`: optional; CLI uses the latest version if not set. <!-- docs/cloud/capacity-modes.mdx:219 -->

Get current capacity (alias `g`):

```
tcld namespace capacity get \
  --namespace <namespace_name>
```

<!-- docs/cloud/tcld/namespace.mdx#get -->

- `--namespace` (`-n`): required. <!-- docs/cloud/tcld/namespace.mdx#get -->

If using API key authentication with `--api-key`, add it directly after `tcld` and before `capacity update`. <!-- docs/cloud/capacity-modes.mdx:221 -->

`capacity update` changes both the bill and the throughput ceiling, so propose it
rather than running it: read the current setting with `capacity get` first, and put
the before-and-after mode and TRU count in front of the user. The direction that
causes an incident is downward — lowering TRUs, or switching `provisioned` →
`on_demand` on a Namespace that was provisioned precisely because auto-scaling
could not keep up, throttles production traffic with `RESOURCE_EXHAUSTED` rather
than failing the command. See [Throttling Behavior](#throttling-behavior) and
[../triage/rate-limits.md](../triage/rate-limits.md).

### UI

Navigate to the Namespace page in Temporal Cloud UI (`https://cloud.temporal.io/namespaces/<Namespace ID>`), click **Manage Capacity**, then select On-Demand or Provisioned and configure TRUs via the slider. <!-- docs/cloud/capacity-modes.mdx:183-198 -->

### API

Call the `UpdateNamespace` API after Namespace creation and define the desired capacity state as part of the capacity spec. <!-- docs/cloud/capacity-modes.mdx:225-226 -->

---

## Throttling Behavior

When your Action rate exceeds your APS (or RPS/OPS) limit, Temporal Cloud throttles requests. <!-- docs/cloud/capacity-modes.mdx:70-71 -->

1. **Priority-based**: Low-priority operations throttled first; higher-priority operations (`StartWorkflowExecution`, `SignalWorkflowExecution`, `UpdateWorkflowExecution`) continue when possible. <!-- docs/evaluate/temporal-cloud/limits.mdx:95 -->
2. **Not instantaneous**: Usage may briefly exceed your limit before throttling takes effect. <!-- docs/evaluate/temporal-cloud/limits.mdx:96 -->
3. **`ResourceExhausted` errors**: Server returns a `ResourceExhausted` gRPC error; SDK clients automatically retry based on the default gRPC retry policy. <!-- docs/evaluate/temporal-cloud/limits.mdx:97 -->
4. **Potential failure**: If throttling persists beyond the SDK's retry limit, client calls fail -- work **can** be lost if you do not handle these failures. <!-- docs/evaluate/temporal-cloud/limits.mdx:98 -->

> For diagnosis of `RESOURCE_EXHAUSTED` errors in triage context, see `../triage/rate-limits.md`.

**Best practices for handling throttling**: <!-- docs/evaluate/temporal-cloud/limits.mdx:100-103 -->

- Log any failed `StartWorkflowExecution`, `SignalWorkflowExecution`, or `UpdateWorkflowExecution` calls (including payloads) so you can retry or backfill later.
- Set up Cloud metrics (`temporal_cloud_v0_resource_exhausted_errors`) to alert when throttling occurs. <!-- docs/best-practices/managing-aps-limits.mdx:237 -->
- Alert at 70-80% utilization to give time to react. <!-- docs/best-practices/managing-aps-limits.mdx:238 -->

---

## Other Namespace-Level Limits

| Limit | Default | Notes |
|-------|---------|-------|
| Namespaces per account | 10 (auto-increases) | <!-- docs/evaluate/temporal-cloud/limits.mdx:49-51 --> |
| Schedules RPS | 10 per second | Use jitter to avoid thundering herd <!-- docs/evaluate/temporal-cloud/limits.mdx:108 --> |
| Visibility API | 30 calls per second | Not configurable <!-- docs/evaluate/temporal-cloud/limits.mdx:120 --> |
| Certificates | 32 KB or 16 certificates (whichever first) | <!-- docs/evaluate/temporal-cloud/limits.mdx:140 --> |
| Concurrent Task pollers | 20,000 Activity + 20,000 Workflow Task | Per Namespace <!-- docs/evaluate/temporal-cloud/limits.mdx:144 --> |
| Retention period | 30 days default, configurable 1-90 days | <!-- docs/evaluate/temporal-cloud/limits.mdx:152-169 --> |
| Batch jobs | 1 concurrent per Namespace, max 50 Executions/sec | <!-- docs/evaluate/temporal-cloud/limits.mdx:172-174 --> |

---

## APS Management Best Practices

### Common reasons for hitting APS limits

1. **Bursty traffic**: Calendar-driven spikes, event-driven surges, recovery thundering herds, timer storms, retry storms. <!-- docs/best-practices/managing-aps-limits.mdx:78-89 -->
2. **Cascading Workflows and fan-out**: Parent Workflows spawning many Child Workflows; each child's full action lifecycle counts against the Namespace APS. <!-- docs/best-practices/managing-aps-limits.mdx:103-115 -->
3. **Human-in-the-loop at scale**: Long-running Workflows with frequent Queries from UIs for state polling. <!-- docs/best-practices/managing-aps-limits.mdx:122-129 -->
4. **Many small Activities**: 1,000 single-record Activities vs 10 batched Activities -- each Activity adds Action overhead. <!-- docs/best-practices/managing-aps-limits.mdx:153-168 -->
5. **Multiple use cases in one Namespace**: APS limit is per Namespace, so multiple workloads compound. <!-- docs/best-practices/managing-aps-limits.mdx:170-176 -->

### Mitigation strategies

- **Stagger and jitter**: Use Schedule jitter and Start Delay to smooth batch starts. <!-- docs/best-practices/managing-aps-limits.mdx:98-99 -->
- **Batch Activities**: Combine multiple external calls in a single Activity; process data in chunks. <!-- docs/best-practices/managing-aps-limits.mdx:166-168 -->
- **Reduce fan-out depth**: Evaluate whether Child Workflows are necessary; limit fan-out size; flatten deeply nested hierarchies. <!-- docs/best-practices/managing-aps-limits.mdx:118-120 -->
- **Push state, don't poll**: Avoid polling patterns where UIs constantly Query Workflow state; push state changes to a database that UIs read. <!-- docs/best-practices/managing-aps-limits.mdx:132 -->
- **Use longer monitoring intervals**: Check SLAs every 30 minutes instead of every 1 minute; consolidate Timers. <!-- docs/best-practices/managing-aps-limits.mdx:145-148 -->
- **Separate Namespaces per use case**: Plan for one set of Namespaces (per environment) per use case. <!-- docs/best-practices/managing-aps-limits.mdx:176-178 -->
- **Provision TRUs for known spikes**: Pre-provision before planned events, deprovision after. <!-- docs/best-practices/managing-aps-limits.mdx:195-202 -->

### Automation for TRU scaling

- Use the Cloud Ops API, Terraform Provider, or `tcld` CLI to programmatically scale capacity. <!-- docs/best-practices/managing-aps-limits.mdx:221 -->
- Set utilization thresholds (e.g., scale up at 70-80% of limit). <!-- docs/best-practices/managing-aps-limits.mdx:222 -->
- Schedule capacity changes with Temporal Schedules or Workflows. <!-- docs/best-practices/managing-aps-limits.mdx:223 -->
- React to upstream leading indicators (queue depth, campaign start) to trigger capacity changes proactively. <!-- docs/best-practices/managing-aps-limits.mdx:224 -->

### Monitoring

- Track `temporal_cloud_v0_resource_exhausted_errors` to detect throttling events. <!-- docs/best-practices/managing-aps-limits.mdx:237 -->
- Alert at 70-80% utilization. <!-- docs/best-practices/managing-aps-limits.mdx:238 -->
- Analyze historical patterns to decide between reactive TRU provisioning and proactive automation. <!-- docs/best-practices/managing-aps-limits.mdx:239 -->
- For Provisioned Namespaces, on-demand envelope metrics show what limits would be under On-Demand mode. <!-- docs/cloud/capacity-modes.mdx:91-92 -->

---

## Quick Reference

| Question | Answer |
|----------|--------|
| Default APS (On-Demand) | 500 <!-- docs/cloud/capacity-modes.mdx:107 --> |
| Default RPS (On-Demand) | 2,000 <!-- docs/cloud/capacity-modes.mdx:107 --> |
| Default OPS (On-Demand) | 4,000 <!-- docs/cloud/capacity-modes.mdx:107 --> |
| APS per TRU | 500 <!-- docs/cloud/capacity-modes.mdx:140 --> |
| RPS per TRU | 1,500 <!-- docs/cloud/capacity-modes.mdx:140 --> |
| OPS per TRU | 4,000 <!-- docs/cloud/capacity-modes.mdx:140 --> |
| Valid TRU counts | 2, 3, 4, 6, 8, 10, 12 <!-- docs/cloud/capacity-modes.mdx:142, 149 --> |
| TRU provisioning time | Within 2 minutes <!-- docs/cloud/capacity-modes.mdx:150 --> |
| On-Demand scaling window | Past 7 days <!-- docs/cloud/capacity-modes.mdx:110 --> |
| On-Demand formula | lesser of 4 x APS Mean or 2 x APS P90 <!-- docs/cloud/capacity-modes.mdx:110 --> |
| Who can change capacity | Global Admin, Namespace Admin <!-- docs/cloud/capacity-modes.mdx:179 --> |
| CLI commands | `tcld namespace capacity get`, `tcld namespace capacity update` <!-- docs/cloud/tcld/namespace.mdx#get --> |
| Throttling error | `ResourceExhausted` gRPC error <!-- docs/evaluate/temporal-cloud/limits.mdx:97 --> |
