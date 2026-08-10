# Nexus Caller-Namespace Limits (Temporal Cloud)

Operator reference for the default 1,000-caller-Namespace ceiling on each Nexus Endpoint's Access Policy in Temporal Cloud: what the limit is, how to inspect the current allowlist, how to add/remove/set entries via `tcld nexus`, and how to raise the ceiling.

Scope: **Temporal Cloud only.** Self-hosted deployments do not have a Cloud-managed Access Policy — self-hosted authorization goes through a custom Authorizer plugin instead.

---

## The limit

**A single Nexus Endpoint can have a maximum of 1,000 caller Namespaces in its Access Policy by default.**  Request further increases beyond the initial 1,000 by opening a support ticket.

The Access Policy is the allowlist of caller Namespaces permitted to use the Endpoint at runtime.  When a caller Workflow executes a Nexus Operation, Temporal Cloud verifies the caller's Namespace is in the Endpoint's allowlist before routing the request to the handler.

**No callers are allowed by default, even if in the same Namespace as the Endpoint target.**  The allowlist is empty at Endpoint create time unless `--allow-namespace` is supplied.

### Adjacent Nexus limits (cross-reference only)

- **100 Nexus Endpoints per Account by default.**  Also raised via support ticket. This is a distinct limit — don't conflate the two 1,000/100 figures.
- Other Nexus limits (rate limits, per-Workflow in-flight Operation ceilings, callback ceilings, handler-request timeout, ScheduleToClose maximum) are indexed at [`docs/cloud/nexus/limits.mdx`](https://docs.temporal.io/cloud/nexus/limits). This file does not re-own them.

---

## Inspect the current allowlist

Read-only. Safe to run without confirmation.

```bash
tcld nexus endpoint allowed-namespace list --name <endpoint-name>
```

Required flag: `--name` (alias `-n`).

To see the Endpoint's full configuration (target Namespace, target Task Queue, description, allowlist):

```bash
tcld nexus endpoint get --name <endpoint-name>
```

---

## Manage the allowlist

The subcommand group is `allowed-namespace` (with a `d`). The `--allow-namespace` singular form (no `d`) is only a flag on `endpoint create` for seeding the initial list.

### Add caller Namespaces

```bash
tcld nexus endpoint allowed-namespace add \
    --name <endpoint-name> \
    --namespace <caller-ns-1> \
    --namespace <caller-ns-2>
```

Required flags: `--name` and `--namespace`. `--namespace` is repeatable. Namespaces already on the list are silently ignored. Aliases: `--name` → `-n`, `--namespace` → `-ns`.

### Remove caller Namespaces

```bash
tcld nexus endpoint allowed-namespace remove \
    --name <endpoint-name> \
    --namespace <caller-ns>
```

Namespaces not currently allowed are silently ignored.

### Replace the full allowlist

Use `set` when you want to declare the exact list rather than diff-apply. This is the byte-precise form and replaces the previous allowlist entirely.

```bash
tcld nexus endpoint allowed-namespace set \
    --name <endpoint-name> \
    --namespace <caller-ns-1> \
    --namespace <caller-ns-2>
```

**Set replaces the full list of allowed namespaces**, dropping any entry not supplied in this call.  Treat `set` as a destructive operation: run `allowed-namespace list` first to capture the current state, diff against your intended list, and confirm with the user before running.

### Seed the allowlist at Endpoint create

`--allow-namespace` (singular, no `d`) is a repeatable flag on `endpoint create` that seeds the initial allowlist. It is not a subcommand.

`tcld nexus endpoint create` is marked experimental.

```bash
tcld nexus endpoint create \
    --name <endpoint-name> \
    --target-namespace <handler-ns.account> \
    --target-task-queue <task-queue> \
    --allow-namespace <caller-ns-1> \
    --allow-namespace <caller-ns-2>
```

`--target-namespace` and `--target-task-queue` are required at create time. `--allow-namespace` is optional and repeatable. Aliases: `--name` → `-n`, `--target-namespace` → `-tns`, `--target-task-queue` → `-ttq`, `--allow-namespace` → `-ans`.

---

## Raise the 1,000 ceiling

There is no CLI flag, config key, or Terraform variable that raises the cap. **Increases require opening a support ticket.**

When the user is approaching or at the limit:

1. Confirm current allowlist size with `allowed-namespace list --name <endpoint-name>`.
2. Identify whether the growth is expected (real fan-in of caller Namespaces) or accidental (stale entries).
3. If growth is expected, direct the user to file a support request via the Temporal Cloud support portal (see the Cloud support page in `docs/cloud/support`).

Do not propose sharding-by-Endpoint or other workaround architectures — the documentation does not prescribe one, and the supported path is the support ticket.

---

## When to hand this off

- **User wants to *write* a Nexus caller Workflow or Service definition:** hand off to `skill-temporal-developer`. This file is for the operator surface only.
- **Nexus Operation is stuck / failing at runtime, not an allowlist-configuration issue:** see [`workflow-stuck.md#pending-nexus-operations`](../triage/workflow-stuck.md#pending-nexus-operations).
- **Provisioning Endpoints (including the allowlist) via Terraform:** see [`cloud-terraform.md#nexus-endpoint-management`](cloud-terraform.md#nexus-endpoint-management). The Terraform resource field is `allowed_caller_namespaces`.
- **Nexus Endpoint CRUD on self-hosted:** see [`self-hosted-admin.md#nexus-endpoint-commands`](self-hosted-admin.md#nexus-endpoint-commands). Self-hosted has no equivalent 1,000-caller cap because the Access Policy is a Cloud-only construct.

---

## Common mistakes

- **Confusing the 1,000-caller-Namespaces-per-Endpoint limit with the 100-Endpoints-per-Account limit.** They sit four lines apart in the same source and are easy to swap.
- **Treating `set` like `add`.** `set` replaces the full allowlist; entries you don't pass are removed. Read the current list first.
- **Using `allow-namespace` (singular, no `d`) as a subcommand name.** The subcommand group is `allowed-namespace`; the singular form is only a `--allow-namespace` flag on `endpoint create`.
- **Assuming same-Namespace callers are permitted by default.** They are not — no callers are allowed until explicitly added, even from the Endpoint's own target Namespace.
- **Attributing the 1,000 limit to self-hosted deployments.** The Access Policy is a Cloud construct.

---

## Resources

- [Temporal Cloud limits — Nexus caller Namespace limits](https://docs.temporal.io/cloud/limits#nexus-endpoint-access-policy-limits)
- [Nexus limits index](https://docs.temporal.io/cloud/nexus/limits)
- [Nexus Security — Runtime access controls](https://docs.temporal.io/nexus/security#runtime-access-controls)
- [Nexus Registry — Configure runtime access controls](https://docs.temporal.io/nexus/registry#configure-runtime-access-controls)
- [`tcld nexus` CLI reference](https://docs.temporal.io/cloud/tcld/nexus)
