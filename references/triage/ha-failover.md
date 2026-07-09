# HA Failover

Diagnose Temporal Cloud Multi-region / Multi-cloud Namespace failover symptoms: clients not following the CNAME swap, PrivateLink breaking after the active region changes, and failovers that don't appear to have taken effect. Cloud-only — self-hosted HA is out of scope.

Citations point at stable doc anchors (slug + heading), e.g. `/cloud/high-availability/ha-connectivity#namespace-endpoint-recommended`. They are public URL fragments, not line numbers — durable across doc edits.

Out of scope here (link, don't absorb):
- DNS / TCP reachability in general (before HA is even a hypothesis) → [connectivity.md](connectivity.md) (layers 1-2), which also carries the PrivateLink / CNAME facts in [§ PrivateLink and PSC](connectivity.md#privatelink-and-psc).
- Worker placement architecture decisions (cost, latency, pattern selection) → `skill-temporal-deploy`. This file gives only a triage-layer pointer.
- Enabling HA / choosing replica regions / pricing at setup time → `/cloud/high-availability/enable`.
- `tcld` flag semantics in depth → `skill-temporal-cli` → `references/core/cloud-control-plane.md`.

## Table of Contents

- [Start here: establish ground truth](#start-here-establish-ground-truth)
- [How Cloud HA routing works (minimum needed for triage)](#how-cloud-ha-routing-works-minimum-needed-for-triage)
- [Symptom: clients did not follow the failover](#symptom-clients-did-not-follow-the-failover)
- [Symptom: PrivateLink / PSC stopped working after failover](#symptom-privatelink--psc-stopped-working-after-failover)
- [Symptom: failover was requested but never happened](#symptom-failover-was-requested-but-never-happened)
- [Symptom: Workflows are rejected during handover](#symptom-workflows-are-rejected-during-handover)
- [Worker placement — triage-layer pointer](#worker-placement--triage-layer-pointer)
- [RPO / RTO reference](#rpo--rto-reference)

## Start here: establish ground truth

Every failover symptom funnels through one question: **what does the control plane think is active, and what does the client's DNS resolve to?** Get both before forming any hypothesis — most "failover is broken" reports are a disagreement between these two views.

**Control-plane view:**

```bash
tcld namespace get --namespace <namespace_id>.<account_id>
```

Returns the Namespace record including its regions. <!-- /cloud/tcld/namespace#get --> For the exact field reporting the active region or replica state, read the live response — the active region is reflected in the target of the Namespace Endpoint's CNAME record. <!-- /cloud/high-availability/ha-connectivity#how-namespace-dns-records-work-with-privatelink --> <!-- VERIFY: exact JSON field name for active region in `tcld namespace get` output -->

**DNS view, from the failing environment:**

```bash
dig +short <namespace>.<account>.tmprl.cloud   # man: dig(1)
nslookup <namespace>.<account>.tmprl.cloud     # man: nslookup(1)
```

The CNAME target encodes the active region, e.g. `aws-us-east-1.region.tmprl.cloud`. <!-- /cloud/high-availability/ha-connectivity#how-endpoints-route-on-failover -->

**Corroborating signals:** failovers are written to the audit log as `"operation": "FailoverNamespace"`, shown on the Namespace detail page in the Web UI, and emailed to account admins. <!-- /cloud/high-availability/failovers#post-failover-events --> <!-- /cloud/high-availability/monitoring#failover-audit-log -->

| Observation | Reading | Go to |
|---|---|---|
| Both views agree, CNAME points at the expected region | Healthy. The failover itself is fine. | If traffic still fails, the problem is below HA → [connectivity.md](connectivity.md) |
| Control plane moved, client DNS still shows old region | Resolver on the client path cached the old CNAME | [clients did not follow](#symptom-clients-did-not-follow-the-failover) |
| Control plane still shows old region, no `FailoverNamespace` audit entry | The failover never executed | [failover was requested but never happened](#symptom-failover-was-requested-but-never-happened) |
| DNS resolves to a public IP a private VPC can't reach | Private DNS doesn't cover the new region | [PrivateLink / PSC stopped working](#symptom-privatelink--psc-stopped-working-after-failover) |

## How Cloud HA routing works (minimum needed for triage)

An HA Namespace keeps a primary and a replica in separate isolation domains. On failover, Temporal changes which is active and updates DNS so the Namespace Endpoint routes to the new active region. <!-- /cloud/high-availability/failovers --> Two DNS names drive every symptom below:

| Name | Form | Behavior on failover |
|---|---|---|
| **Namespace Endpoint** (what clients use) | `<namespace>.<account>.tmprl.cloud:7233` | Hostname never changes; it is a CNAME whose target Temporal repoints. <!-- /cloud/high-availability/ha-connectivity#namespace-endpoint-recommended --> |
| **Regional record** (CNAME target) | `<cloud>-<region>.region.tmprl.cloud` (e.g. `aws-us-west-2.region.tmprl.cloud`) | Temporal updates the CNAME from the old region to the new one. <!-- /cloud/high-availability/ha-connectivity#how-endpoints-route-on-failover --> |

Two timing facts the triage logic depends on:

- Namespace DNS records carry a **15-second TTL**; clients converge to the new region within ~30 seconds (about 2× the TTL), assuming their resolver honors the TTL. <!-- /cloud/high-availability/ha-connectivity#namespace-endpoint-recommended -->
- Temporal Cloud enforces a **5-minute maximum connection lifetime**, forcing long-lived connections to re-resolve DNS. <!-- /cloud/high-availability/failovers/manage#worker -->

## Symptom: clients did not follow the failover

**Symptom:** after a failover, workflow starts or worker polls still hit the old region, connections time out, or the control-plane and client-side DNS views disagree about the active region.

**Discriminate** (re-resolve from the failing environment and compare against `tcld namespace get` first):

1. **DNS resolver caching on the client path.** A cache should re-resolve within the 15-second TTL and converge within ~30 seconds. <!-- /cloud/high-availability/ha-connectivity#namespace-endpoint-recommended --> A resolver that holds the old CNAME longer (NodeLocal DNSCache, a local `dnsmasq`, a VM stub resolver) shows a stale target on repeated `dig +short`. No doc-authoritative list of which resolvers honor the TTL exists — verify empirically.
2. **Long-lived connection not re-resolving.** The 5-minute connection cap exists to force re-resolution. <!-- /cloud/high-availability/failovers/manage#worker --> A worker wedged on a connection that outlives the window won't move; restarting the worker pod forces fresh resolution.
3. **Application-level address caching.** A caller that resolved the hostname to an IP at startup and reused it won't follow a CNAME swap. Pass the hostname to the client config, never a pre-resolved IP.
4. **GCP Private Service Connect.** PSC has no DNS-based automatic failover — workers must be manually repointed to the new region's PSC endpoint. <!-- /cloud/connectivity/gcp-connectivity#high-availability-and-private-service-connect --> See [PrivateLink / PSC stopped working](#symptom-privatelink--psc-stopped-working-after-failover).
5. **Private DNS override covers only one region.** Same section.

**Fix:** clear/await the offending cache, restart wedged workers, or repoint PSC workers per the discriminator that matched.

**Verify:**

```bash
dig +short <namespace>.<account>.tmprl.cloud   # man: dig(1)
# CNAME target should now match the active region from `tcld namespace get`.
```

Then re-run the operation that was failing.

## Symptom: PrivateLink / PSC stopped working after failover

**Symptom:** pre-failover the Namespace was reachable via a private VPC Endpoint; post-failover DNS resolves the Namespace Endpoint to a public IP the VPC can't reach, or to nothing.

**Discriminate:**

1. **Private DNS covered only the old region.** The private hosted zone overrode only the old active region's `<cloud>-<region>.region.tmprl.cloud` record. After the CNAME flips, the new region has no private entry, so the client falls back to public DNS or dead-ends in a no-egress VPC. This is the common cause.
2. **Workers can't reach the new region.** Even with DNS fixed, a single-region worker fleet needs a network path to the now-active region.
3. **Direct-VPCE targeting without a worker in every region.** Direct VPCE works with HA, but it doesn't follow the CNAME — each worker reaches Temporal only through its own region's VPC Endpoint (same `ServerName` override, a different VPCE address per region). If you deployed a worker + VPCE in just one region, cross-region forwarding keeps it working while that region is passive, but a full outage of that region leaves no path. <!-- /cloud/connectivity/aws-connectivity#direct-vpce -->
4. **GCP PSC.** No automatic DNS failover; workers must be manually repointed. <!-- /cloud/connectivity/gcp-connectivity#high-availability-and-private-service-connect -->

**Fix:** the `region.tmprl.cloud` private hosted zone must map **every** region the Namespace can fail over to, each `<cloud>-<region>.region.tmprl.cloud` → that region's VPC Endpoint. <!-- /cloud/high-availability/ha-connectivity#how-to-set-up-the-dns-override --> <!-- /cloud/connectivity/aws-connectivity#configure-private-dns-for-namespaces-with-high-availability --> Give workers a path to the new region — run workers in both regions, or link the VPCs (Transit Gateway / VPC Peering). For direct-VPCE, deploy a worker plus VPC Endpoint in each region (same `ServerName`, different VPCE per region). For GCP PSC, repoint workers to the active region's PSC endpoint manually on failover.

**Verify:** from inside the client VPC, `dig +short` the Namespace Endpoint and confirm it resolves to the new region's VPC Endpoint, then `nc -zvw10 <vpce-host> 7233`.

## Symptom: failover was requested but never happened

**Symptom:** a failover was initiated (Web UI, `tcld`, or Cloud Ops API) but `tcld namespace get` still shows the old active region, the audit log has no `FailoverNamespace` entry, and traffic hasn't shifted.

**Discriminate:**

1. **Manual `tcld` invocation was malformed.** The command is:
   ```bash
   tcld namespace failover \
       --namespace <namespace_id>.<account_id> \
       --region <target_region>
   ```
   <!-- /cloud/tcld/namespace#failover --> <!-- /cloud/high-availability/failovers/manage#trigger-failover --> `--namespace` and `--region` are required. With API-key auth, `--api-key` must come immediately after `tcld`, before `namespace failover`.
2. **Target region isn't an `Activated` replica.** The target must be a region holding a replica that is ready to be failed over to (state `Activated`). <!-- /cloud/high-availability/failovers/manage#trigger-failover --> An unhealthy replica makes the Web UI disable "Trigger a failover"; common causes are data-sync issues, replication lag, network issues, and failed health checks. <!-- /cloud/high-availability/monitoring#replication-status -->
3. **The Namespace can't support this failover (constraint).** See the constraints table below — a missing replica, region eligibility, or a replication-type conflict can make the failover impossible to request.
4. **Permissions.** `FailoverNamespaceRegion` requires Namespace Admin. <!-- /cloud/manage-access/permissions-reference --> Account Owner and Global Admin hold Namespace Admin on all Namespaces. <!-- /cloud/manage-access/roles-and-permissions -->
5. **Automatic Failover didn't fire.** Automatic Failover is driven by Temporal Cloud health checks on error rates, latencies, and infrastructure indicators. <!-- /cloud/high-availability/failovers#conditions-that-trigger-an-automatic-failover --> If it's disabled (`tcld namespace update-high-availability --disable-auto-failover=true`), Temporal won't initiate failovers — the user must trigger manually, and the published Temporal Cloud RTO does not apply. <!-- /cloud/high-availability/enable#automatic-failovers --> <!-- /cloud/tcld/namespace#update-high-availability -->
6. **Expecting an automatic failback that won't come.** After a user-triggered failover Temporal does *not* fail back automatically; the user must trigger it. Automatic failback only follows an Automatic Failover. <!-- /cloud/high-availability/failovers/manage#failbacks -->

**Constraints that can block a failover from being possible:**

| Constraint | Effect | Source |
|---|---|---|
| Namespace has no replica | Must be upgraded with HA (`tcld namespace add-region` or Web UI) before any failover | <!-- /cloud/high-availability/enable#upgrade --> <!-- /cloud/tcld/namespace#add-region --> |
| Replica must be on the same continent as the primary; `sa-east-1` is the only region on its continent, so it has no eligible Multi-region replica | No replica region to fail over to | <!-- /cloud/high-availability/ha-connectivity#available-regions-privatelink-endpoints-and-dns-record-overrides --> <!-- /cloud/high-availability#high-availability-features --> |
| Multi-region and Multi-cloud can't both be enabled on one Namespace | Limits which replica topologies exist to fail over to | <!-- /cloud/high-availability#high-availability-features --> |
| 7-day wait after `tcld namespace delete-region` before re-enabling HA in that region | A just-removed region can't be re-added as a failover target yet | <!-- /cloud/high-availability/enable#changing --> |

**Fix:** correct the matched discriminator (command form, target state, permissions, or auto-failover setting), or resolve the blocking constraint.

**Verify — and an important guarantee:** once a failover request returns an async operation ID, the failover **is guaranteed to complete** — there is no case where an accepted failover silently fails to execute. Temporal retries the failover Workflow internally and pages on-call to force it through on any internal error. <!-- /cloud/high-availability/failovers/manage#trigger-failover --> So if you received an operation ID but traffic hasn't shifted, the failover succeeded — look to DNS propagation (the [clients did not follow](#symptom-clients-did-not-follow-the-failover) path), not the failover request.

## Symptom: Workflows are rejected during handover

**Symptom:** during the failover, clients see a brief window of retryable "Service unavailable" errors and start/signal requests are rejected.

**What it is (expected behavior):** the failover is a single hybrid strategy. Temporal first attempts a *graceful failover* — pause traffic, drain in-flight replication, switch with no data conflicts. If that doesn't complete within 10 seconds, it falls back to a *forced failover* that immediately activates the replica; unreplicated events undergo conflict resolution when the original region returns. It **proceeds to the forced failover — it does not revert.** Operations pause briefly and SDKs receive a retryable "Service unavailable" error they retry automatically. <!-- /cloud/high-availability/failovers#failover-process -->

**Discriminate** (only if the window is unusually long or the error isn't retried):

1. **Raw gRPC client, not an SDK.** SDK retries cover this window by design; a raw client must retry `UNAVAILABLE` itself. <!-- grpc: UNAVAILABLE -->
2. **Large replication lag.** A forced failover with significant lag is more likely to roll back Workflow progress; always check lag before failing over. <!-- /cloud/high-availability/failovers/manage#trigger-failover --> <!-- /cloud/high-availability/monitoring#monitoring-replication --> Lag is exposed as `temporal_cloud_v1_replication_lag_p50` / `_p95` / `_p99`. <!-- /cloud/metrics/openmetrics/metrics-reference#replication-metrics -->

**Fix:** ensure callers use a Temporal SDK (or add `UNAVAILABLE` retries to raw clients); check and wait out replication lag before manual failovers.

## Worker placement — triage-layer pointer

The triage concern is narrow: confirm workers can reach whichever region is currently active and that they follow the CNAME rather than hard-coding a Regional Endpoint.

- Enabling HA requires no special worker configuration; the DNS redirection is invisible to workers using the Namespace Endpoint. <!-- /cloud/high-availability/failovers/manage#worker -->
- Two supported configurations: run workers in both regions continuously, or establish cross-region connectivity (Transit Gateway / VPC Peering) so a single-region fleet can reach the newly active region. <!-- /cloud/high-availability/ha-connectivity#how-to-set-up-the-dns-override -->
- In a full regional outage, workers in that region may fail alongside the primary; a second fleet in the replica's region keeps Workflows moving. <!-- /cloud/high-availability/failovers/manage#worker -->

Pattern selection (cost, latency, operational complexity) is a `skill-temporal-deploy` concern, not triage.

## RPO / RTO reference

Full reference: `/cloud/rpo-rto`. Triage-relevant facts only:

| Fact | Value | Source |
|---|---|---|
| HA target (cell / regional / Multi-cloud cloud-wide outages) | sub-1-minute RPO, 20-minute RTO | <!-- /cloud/high-availability#service-levels-and-recovery-objectives --> <!-- /cloud/rpo-rto#rto-and-rpo-summary --> |
| AZ outages (all Namespaces, not just HA) | zero RPO, near-zero RTO via 3-AZ replication | <!-- /cloud/rpo-rto#availability-zone-outage --> |
| Automatic Failover disabled | published RTO does not apply (Temporal can't control when the user triggers) | <!-- /cloud/high-availability/enable#automatic-failovers --> <!-- /cloud/rpo-rto#rto-and-rpo-summary --> |
| Failback responsibility | automatic after an Automatic Failover; user's responsibility after a user-triggered failover | <!-- /cloud/high-availability/failovers/manage#failbacks --> |

For manual-failover sequencing (why an operator might trigger faster than Temporal, how to order application-side vs Namespace failover), see `/cloud/rpo-rto` § "Tips for a lower Recovery Time". Region health: `https://status.temporal.io`.
