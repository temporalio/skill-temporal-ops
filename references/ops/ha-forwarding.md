# HA forwarding

A Temporal Cloud Namespace with High Availability features has one active replica and one passive replica; by default, any request that reaches the passive replica is transparently forwarded to the active region and the response returned to the caller.  The `disablePassivePollerForwarding` Namespace setting controls this behavior for **Worker poll traffic only** — Client requests such as Start Workflow, Signal, and Query keep forwarding regardless of the setting.

Canonical docs page: `https://docs.temporal.io/cloud/high-availability/enable#change-forwarding-behavior`.

## Scope of the setting

- With `disablePassivePollerForwarding` enabled, Worker polls that reach a passive replica are not forwarded, and those Workers do not execute Workflows or Activities.
- Workers connected to such a passive replica receive a `NamespaceNotActive` error on poll requests, stay connected, and start executing Workflows and Activities if the replica becomes active.
- Same-region replicas are not affected by this setting.
- A request can reach the passive replica through the passive region's Regional Endpoint, through a PrivateLink or Private Service Connect endpoint in the passive region, or transiently through the Namespace Endpoint during a failover.

## APIs that keep forwarding regardless

The following interface-level contract lists the APIs that are forwarded from the passive replica to the active region whether or not `disablePassivePollerForwarding` is enabled, with responses returned to the Client.  The docs cite [`selectedAPIsForwardingRedirectionPolicyWhitelistedAPIs`](https://github.com/temporalio/temporal/blob/main/common/rpc/interceptor/dc_redirection_policy.go#L59-L71) in `common/rpc/interceptor/dc_redirection_policy.go` as the origin of this whitelist.

| API group | Forwarded operations |
| --- | --- |
| Workflow | Start Workflow, Signal-with-Start, Signal, Cancel, Terminate, Delete, Query |
| Standalone Activity | Start, Cancel, Terminate, Delete, Pause, Unpause, Reset, Update Activity Options |
| Standalone Nexus Operation | Start, Cancel, Terminate, Delete |

A Client that reaches the passive region keeps working while the setting is enabled: it can still start Workflows, send Signals, and run Queries successfully, and the resulting Workflow and Activity Tasks are processed by the active region's Workers.

## How to change the setting

### With the `temporal cloud` CLI

Use the [`temporal cloud namespace ha update`](https://docs.temporal.io/cli/command-reference/cloud/namespace#ha-update) command with `--passive-poller-forwarding {enabled,disabled}`.

```bash
temporal cloud namespace ha update \
    --namespace <namespace>.<account> \
    --passive-poller-forwarding disabled
```

Set the flag to `enabled` to restore forwarding.

> [!NOTE]
> `tcld namespace update-high-availability` does not carry this flag; it only takes `--disable-auto-failover`. Use the `temporal cloud namespace ha update` command instead.

### With the Cloud Ops API

The setting lives at `.spec.highAvailability.disablePassivePollerForwarding` on the Namespace spec. The field uses **disable-polarity**: set `true` to disable forwarding, or `false` to restore the default.  Because `UpdateNamespace` replaces the entire spec, fetch the current spec, merge the new value, and post it back with the current `resourceVersion`; preserve any other High Availability fields (such as `disableManagedFailover`) with a `jq` merge.  For the full curl + `jq` recipe, see `https://docs.temporal.io/cloud/high-availability/enable#set-forwarding-curl`.

## How to read the current value

- CLI: [`temporal cloud namespace ha get`](https://docs.temporal.io/cli/command-reference/cloud/namespace#ha-get) shows the active region, whether managed failover is enabled, and whether passive poller forwarding is enabled.
- Cloud Ops API: read `.namespace.spec.highAvailability.disablePassivePollerForwarding` from `GET /cloud/namespaces/<ns>`. A result of `null` means the field has never been set, which is equivalent to `false` — proto3 JSON omits default-`false` values from responses, so forwarding is enabled.

## When you would disable it

The documented motivation is the Active/Hot-Passive pattern: a full Worker fleet in the passive region connects through Regional or VPC Endpoints and stays on standby until failover, at which point it begins processing immediately without a cold start. See `https://docs.temporal.io/cloud/high-availability/architecture-patterns#active-hot`.

## Common mistakes

- Assuming the setting stops Signals, Start Workflow, or Query requests from crossing regions — those APIs are always forwarded per the table above.
- Reaching for `tcld namespace update-high-availability` for this flag — that command sets automatic-failover only (`--disable-auto-failover`).
- Inverting polarity in the Ops API JSON: `disablePassivePollerForwarding: false` re-enables forwarding, it does not disable it.
- Assuming the setting has any effect on Same-region replicas — they are not affected.
- Assuming a passive-region Client will fail when the setting is enabled — it does not; the Client's request is forwarded to the active region, and the active region's Workers process the resulting tasks.

## Related references

- `cloud-namespace-admin.md` — HA lifecycle commands (`ha region add`, `ha failover`, `ha update --auto-failover`).
- `../triage/ha-failover.md` — symptom triage when something looks wrong after toggling forwarding or failing over.
