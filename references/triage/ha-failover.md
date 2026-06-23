# HA Failover

Diagnose Temporal Cloud Multi-region / Multi-cloud Namespace failover symptoms: clients not following the CNAME swap, PrivateLink breaking after the active region changes, and failovers that don't appear to have taken effect. This file is Cloud-only — self-hosted HA is out of scope.

Out of scope here (link, don't absorb):
- DNS / TCP reachability in general (before HA is even a hypothesis) → [connectivity.md](connectivity.md) (layers 1-2). The PrivateLink / CNAME facts this file relies on are grounded in `ha-connectivity.mdx` and are also referenced from [connectivity.md § PrivateLink and PSC](connectivity.md#privatelink-and-psc).
- Worker placement architecture decisions → `skill-temporal-deploy` (this file only gives a triage-layer pointer)
- Enabling HA / choosing replica regions / pricing at setup time → `docs/cloud/high-availability/enable.mdx` (setup, not triage)
- `tcld` flag semantics in depth → `skill-temporal-cli` → `references/core/cloud-control-plane.md`

## Table of Contents

- [How Cloud HA routing works](#how-cloud-ha-routing-works)
- [Verify the current active region](#verify-the-current-active-region)
- [Symptom: clients did not follow the failover](#symptom-clients-did-not-follow-the-failover)
- [Symptom: PrivateLink stopped working after failover](#symptom-privatelink-stopped-working-after-failover)
- [Symptom: failover was requested but never happened](#symptom-failover-was-requested-but-never-happened)
- [Symptom: failover triggered but Workflows are rejected during handover](#symptom-failover-triggered-but-workflows-are-rejected-during-handover)
- [Worker placement — triage-layer pointer](#worker-placement--triage-layer-pointer)
- [Known platform limits](#known-platform-limits)
- [RPO / RTO semantics](#rpo--rto-semantics)

## How Cloud HA routing works

A Namespace with High Availability features keeps a primary and a replica in separate isolation domains; on failover Temporal changes which one is active and DNS reroutes the Namespace Endpoint to the new active region. <!-- docs/cloud/high-availability/index.mdx:23-24 --><!-- docs/cloud/high-availability/index.mdx:58 -->

Two DNS names matter during failover:

- **Namespace Endpoint** — `<namespace>.<account>.tmprl.cloud:7233`. Single hostname clients use. It does not change on failover. <!-- docs/cloud/get-started/namespaces.mdx:331 --><!-- docs/cloud/high-availability/ha-connectivity.mdx:24-28 -->
- **Regional Endpoint target** — `<region>.region.tmprl.cloud` (e.g. `aws-us-west-2.region.tmprl.cloud`). The Namespace Endpoint is a CNAME to this regional record, and on failover Temporal updates the CNAME target from the old region to the new one. <!-- docs/cloud/high-availability/ha-connectivity.mdx:26-31 --><!-- docs/cloud/high-availability/ha-connectivity.mdx:44-57 -->

Docs quantify DNS convergence explicitly: Namespace DNS records are configured with a 15 second TTL, and clients should converge to the newly targeted region within, at most, a 30-second delay. <!-- docs/cloud/high-availability/ha-connectivity.mdx:152-154 -->

Temporal Cloud also enforces a maximum connection lifetime of 5 minutes, which gives Workers an opportunity to re-resolve DNS. <!-- docs/cloud/high-availability/failovers/manage.mdx:196-197 -->

## Verify the current active region

Run both checks — the control-plane and DNS views must agree for traffic to flow correctly.

```bash
# Control-plane view
tcld namespace get --namespace <namespace_id>.<account_id>
# docs/cloud/tcld/namespace.mdx:446-468
```

`tcld namespace get` takes only `--namespace` (or `$TEMPORAL_CLOUD_NAMESPACE`) and returns the Namespace record including its regions. <!-- docs/cloud/tcld/namespace.mdx:456-467 --> For the precise field name reported as "active region" or "replica state" in the output, inspect the live response — docs describe the concept (a Namespace's active region is reflected in the target of the Namespace Endpoint's CNAME record) <!-- docs/cloud/high-availability/ha-connectivity.mdx:212 --> but do not spell the JSON schema. <!-- VERIFY: exact JSON field name for active region in `tcld namespace get` output -->

```bash
# DNS view from the client environment
dig +short <namespace>.<account>.tmprl.cloud   # man: dig(1)
nslookup <namespace>.<account>.tmprl.cloud     # man: nslookup(1)
```

The CNAME target encodes the active region (e.g. `aws-us-east-1.region.tmprl.cloud`). <!-- docs/cloud/high-availability/ha-connectivity.mdx:26-31 -->

Failover events are also written to the audit log as `"operation": "FailoverNamespace"` and shown on the Namespace detail page in the Web UI; Temporal emails account admins when a failover happens. <!-- docs/cloud/high-availability/failovers/index.mdx:128-130 --><!-- docs/cloud/high-availability/monitoring.mdx:67-69 -->

**Healthy state after failover:** both views agree, and the CNAME points at the region you expect to be active.

**If they disagree:** the control plane made the change, but DNS resolvers on the client path have cached the old CNAME. Continue to [clients did not follow](#symptom-clients-did-not-follow-the-failover).

## Symptom: clients did not follow the failover

**Symptom shape:** after a failover, workflow starts or worker polls still hit the old region, connections time out, or the control-plane view and client-side DNS disagree about the active region.

**First check:** re-resolve the Namespace Endpoint from the failing environment and compare against `tcld namespace get` (see [Verify the current active region](#verify-the-current-active-region)).

**Things to discriminate:**

1. **DNS resolver caching on the client path.** Docs state that any DNS cache should re-resolve the Namespace record within the 15-second TTL and converge within ~30 seconds. <!-- docs/cloud/high-availability/ha-connectivity.mdx:152-154 --> If a specific resolver on the client path (NodeLocal DNSCache, a local `dnsmasq`, a VM's stub resolver) is holding the old CNAME longer than that, repeated `dig +short` will show the stale target. There's no Temporal-docs-authoritative list of which resolvers honor a 15-second TTL; verify empirically.
2. **Long-lived connection not re-resolving.** Temporal Cloud enforces a maximum connection lifetime of 5 minutes specifically so Workers get an opportunity to re-resolve DNS. <!-- docs/cloud/high-availability/failovers/manage.mdx:196-197 --> If a Worker is wedged on a single connection that outlives this window, restarting the worker pod is the fastest way to force a fresh resolution.
3. **Application-level address caching.** If the caller resolved the hostname to an IP at startup and reused it, a CNAME swap doesn't help. Pass the hostname to the client config, not a pre-resolved IP.
4. **Private DNS override covers only one region.** See [PrivateLink stopped working after failover](#symptom-privatelink-stopped-working-after-failover).

**Verify the fix worked:**

```bash
dig +short <namespace>.<account>.tmprl.cloud   # man: dig(1)
# CNAME target should now match the expected active region from `tcld namespace get`.
```

Then re-run whatever operation was failing.

## Symptom: PrivateLink stopped working after failover

**Symptom shape:** pre-failover the Namespace was reachable via an AWS PrivateLink VPC Endpoint; post-failover DNS resolves the Namespace Endpoint to a public IP that the VPC cannot reach, or to no address at all.

**What it means:** the private DNS override covered only the old active region's `<region>.region.tmprl.cloud` record. After the CNAME flips to the new region, that record has no private hosted zone entry, so the client falls back to public DNS (or dead-ends in a no-egress VPC).

**Fix:** the `region.tmprl.cloud` private hosted zone must cover every region the Namespace can fail over to, mapping each `<region>.region.tmprl.cloud` to the VPC Endpoint in that region's VPC. <!-- docs/cloud/high-availability/ha-connectivity.mdx:67-74 --><!-- docs/cloud/connectivity/aws-connectivity.mdx:217-229 --> Workers must also be able to reach the new region — either by running Workers in both regions, or by linking the VPCs (Transit Gateway / VPC Peering). <!-- docs/cloud/high-availability/ha-connectivity.mdx:78-81 --><!-- docs/cloud/connectivity/aws-connectivity.mdx:227-229 -->

**Related trap — direct VPCE targeting:** the direct-VPCE approach (pointing Workers at the VPC Endpoint DNS name with an SNI override, no per-Namespace DNS record) is explicitly not compatible with HA Namespaces, because HA relies on Temporal's public DNS CNAME records to route traffic to the active region; bypassing DNS means Workers cannot follow the CNAME to the new region. <!-- docs/cloud/connectivity/aws-connectivity.mdx:206-213 -->

## Symptom: failover was requested but never happened

**Symptom shape:** a failover was initiated (Web UI, `tcld`, or Cloud Ops API) but `tcld namespace get` still shows the old active region, the audit log has no matching `FailoverNamespace` entry, and traffic has not shifted.

**Things to discriminate:**

1. **Manual `tcld` invocation.** The documented command is:
   ```bash
   tcld namespace failover \
       --namespace <namespace_id>.<account_id> \
       --region <target_region>
   ```
   <!-- docs/cloud/tcld/namespace.mdx:387-399 -->
   `--namespace` and `--region` are required. <!-- docs/cloud/tcld/namespace.mdx:408-420 --> When using API key authentication with `--api-key`, it must come directly after `tcld` and before `namespace failover`. <!-- docs/cloud/tcld/namespace.mdx:392-399 -->
2. **Target region must be an `Activated` replica.** Docs: `<target_region>` must be a region where the Namespace has a replica that is ready to be failed over to (replica state is `Activated`). <!-- docs/cloud/high-availability/failovers/manage.mdx:58-59 --><!-- docs/cloud/manage-access/permissions-reference.mdx:144 --> If the replica is unhealthy, the Web UI disables "Trigger a failover" to prevent failing over to an unhealthy replica; common causes are data sync issues, replication lag, network issues, and failed health checks. <!-- docs/cloud/high-availability/monitoring.mdx:31-40 -->
3. **Namespace has no replica.** Before failover is possible, the Namespace must have been upgraded with HA features (`tcld namespace add-region` or the Web UI). <!-- docs/cloud/tcld/namespace.mdx:47-90 --><!-- docs/cloud/high-availability/enable.mdx:64-104 -->
4. **Permissions.** The `FailoverNamespaceRegion` Cloud Ops operation requires Namespace Admin. <!-- docs/cloud/manage-access/permissions-reference.mdx:137-144 --> Account Owner and Global Admin automatically have Namespace Admin on all Namespaces. <!-- docs/cloud/manage-access/roles-and-permissions.mdx:17-18 -->
5. **Automatic (Temporal-initiated) failover didn't fire.** Automatic failover is driven by Temporal Cloud's health checks on error rates, latencies, and infrastructure indicators. <!-- docs/cloud/high-availability/failovers/index.mdx:56-76 --> If Temporal-initiated failover is disabled on the Namespace (`tcld namespace update-high-availability --disable-auto-failover=true`), Temporal will not initiate failovers itself — the account team must trigger manually. <!-- docs/cloud/tcld/namespace.mdx:1756-1796 --> When auto-failover is disabled, Temporal Cloud cannot set an RPO/RTO for the Namespace. <!-- docs/cloud/rto-rpo.mdx:55-57 -->
6. **Automatic failback vs. user-initiated failover behavior.** After a user-triggered failover, Temporal will *not* automatically fail back — the user must trigger failback manually. Automatic failback is only available after Temporal-managed failovers. <!-- docs/cloud/high-availability/failovers/manage.mdx:156-162 -->

If the failover legitimately did not execute when requested, escalate to Temporal Cloud support with the Namespace ID and timestamps. Temporal manages retries internally and on-call engineers are paged when a failover workflow cannot complete. <!-- docs/cloud/high-availability/failovers/manage.mdx:130-132 -->

## Symptom: failover triggered but Workflows are rejected during handover

**Symptom shape:** during a graceful (or the graceful phase of hybrid) failover, clients see a brief window where Temporal Cloud returns a "Service unavailable error" and start/signal requests are rejected.

**What it means, per docs:** the failover process is a single hybrid strategy. Temporal Cloud first attempts a *graceful failover* — it pauses traffic, drains in-flight replication, and switches to the replica with no data conflicts. If the graceful attempt does not complete within 10 seconds, Temporal Cloud falls back to a *forced failover*, which immediately activates the replica; any events not yet replicated then undergo conflict resolution once the original region comes back. The attempt does **not** revert — it proceeds to the forced failover. During the switch, Workflow operations are briefly paused and Temporal Cloud returns a retryable "Service unavailable" error to SDKs, which the SDKs retry automatically. <!-- docs/cloud/high-availability/failovers/index.mdx:102-117 -->

**Things to check if the handover window is longer or the error is not retried:**

- Is the caller a Temporal SDK or a raw gRPC client? SDK retries handle this window by design; a raw client must retry the `UNAVAILABLE` status itself. <!-- grpc: UNAVAILABLE -->
- Is replication lag large? A forced failover with significant replication lag has a higher likelihood of rolling back Workflow progress; docs recommend always checking the lag before failing over. <!-- docs/cloud/high-availability/failovers/manage.mdx:31-36 --><!-- docs/cloud/high-availability/monitoring.mdx:55-56 --> Lag is exposed via `temporal_cloud_v1_replication_lag_p50` / `_p95` / `_p99`. <!-- docs/cloud/metrics/openmetrics/metrics-reference.mdx:692-708 -->

## Worker placement — triage-layer pointer

The triage concern is narrow: confirm that Workers are able to reach whichever region is currently active and that they follow the CNAME rather than hard-coding a Regional Endpoint.

- Docs describe two Worker configurations: run Workers in both regions continuously, or establish cross-region connectivity (Transit Gateway / VPC Peering) so a single-region Worker fleet can reach the newly active region. <!-- docs/cloud/high-availability/ha-connectivity.mdx:78-81 --><!-- docs/cloud/connectivity/aws-connectivity.mdx:227-229 -->
- Docs call out explicitly that enabling HA does not require specific Worker configuration; the DNS redirection is invisible to Workers that use the Namespace Endpoint. <!-- docs/cloud/high-availability/failovers/manage.mdx:176-179 -->
- On a regional outage, Workers in that region may fail alongside the primary Namespace; docs recommend a second set of Workers in the replica's region to keep Workflows moving. <!-- docs/cloud/high-availability/failovers/manage.mdx:191-194 -->

Deployment-level pattern selection (cost, latency, operational complexity) is a `skill-temporal-deploy` concern, not triage.

## Known platform limits

- **`sa-east-1` is not available for Multi-region Namespaces.** Currently it's the only Temporal Cloud region on its continent, and the replica must be on the same continent as the primary. <!-- docs/cloud/high-availability/ha-connectivity.mdx:136-140 --><!-- docs/cloud/high-availability/index.mdx:65-66 --><!-- docs/cloud/high-availability/enable.mdx:15 -->
- **GCP Private Service Connect does not support automatic failover via Temporal Cloud DNS.** If you use GCP PSC, you must manually update Workers to point to the active region's PSC endpoint when a failover occurs. <!-- docs/cloud/connectivity/gcp-connectivity.mdx:32-36 -->
- **Direct VPCE targeting (SNI-override pattern) is incompatible with HA Namespaces.** HA depends on the public DNS CNAME to route to the active region; bypassing DNS breaks failover. <!-- docs/cloud/connectivity/aws-connectivity.mdx:206-213 -->
- **Multi-region and Multi-cloud cannot both be enabled on the same Namespace simultaneously.** <!-- docs/cloud/high-availability/index.mdx:69-70 -->
- **Seven-day waiting period after replica removal.** After `tcld namespace delete-region`, re-enabling HA in the same region requires waiting 7 days. <!-- docs/cloud/tcld/namespace.mdx:329-352 --><!-- docs/cloud/high-availability/enable.mdx:115-121 -->

Check `docs/cloud/high-availability/` and `docs/cloud/regions` for the current list when diagnosing — limits move.

## RPO / RTO semantics

Full reference: `docs/cloud/rto-rpo.mdx` (slug `/cloud/rpo-rto`). Summary only:

- HA Namespaces target **sub-1-minute RPO and 20-minute RTO** for cell, regional, and (for Multi-cloud) cloud-wide outages. <!-- docs/cloud/high-availability/index.mdx:94 --><!-- docs/cloud/rto-rpo.mdx:40-49 -->
- These objectives apply only to Namespaces with Temporal-initiated failovers enabled (the default). When Temporal-initiated failovers are disabled, Temporal Cloud cannot set an RPO/RTO because it cannot control when the user triggers failover. <!-- docs/cloud/rto-rpo.mdx:32 --><!-- docs/cloud/rto-rpo.mdx:55-57 -->
- Availability-zone outages are handled by Temporal Cloud's three-AZ replication with zero RPO and near-zero RTO on *all* Namespaces, not just HA ones. <!-- docs/cloud/rto-rpo.mdx:34-37 -->
- After a Temporal-managed failover, Temporal Cloud automatically fails back to the original region once it is healthy. After a user-triggered failover, failback is the user's responsibility. Follow `https://status.temporal.io` for region health. <!-- docs/cloud/high-availability/failovers/manage.mdx:134-162 -->
- For manual-failover guidance (why operators might want to trigger faster than Temporal, how to sequence application-side failover against a Namespace failover), see `docs/cloud/rto-rpo.mdx` §"Minimizing the Recovery Time". <!-- docs/cloud/rto-rpo.mdx:86-113 -->
