# Serverless HA

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

AWS Lambda support for Serverless Workers is in Public Preview; GCP Cloud Run support is in Pre-release.  Serverless Workers *can* be used with a Namespace that has Multi-region or Multi-cloud Replication, but they do **not** follow a Namespace failover automatically. When the Namespace endpoint re-routes to the new active region, the Worker Controller Instance (WCI) keeps invoking Workers in the compute provider's originally configured region. Without manual intervention, workloads that failed over will continue to be executed by Workers in the old region — and if that region is degraded, your workloads are degraded with it.

This file explains why the constraint exists and what an operator must do after a failover to keep processing Workflows.

## Table of contents

- [The constraint in one sentence](#the-constraint-in-one-sentence)
- [Why this happens](#why-this-happens)
- [Diagnostic signal — did a failover happen?](#diagnostic-signal--did-a-failover-happen)
- [Manual remediation](#manual-remediation)
  - [AWS Lambda](#aws-lambda)
  - [GCP Cloud Run](#gcp-cloud-run)
- [Prevention: dual-region compute-provider pre-provisioning](#prevention-dual-region-compute-provider-pre-provisioning)
- [Hand-off pointers](#hand-off-pointers)

## The constraint in one sentence

> On failover of a Namespace with Multi-region or Multi-cloud Replication, the WCI keeps invoking Workers in the original region unless you manually repoint the compute provider. Compute provider configuration, such as a Lambda ARN or a Cloud Run Worker Pool, is scoped to a single region.

The canonical statement in the HA docs is equivalent: "each compute provider configuration (for example, a Lambda ARN) is scoped to a single region. The Worker Controller Instance (WCI) has no mechanism to detect a failover or redirect invocations to a Worker in the new active region. To keep processing Workflows after a failover, you must manually reconfigure the Worker Deployment Version's compute provider to point at a Worker in the new region."

See the full docs entry at [https://docs.temporal.io/cloud/high-availability#serverless-workers](https://docs.temporal.io/cloud/high-availability#serverless-workers) and the Constraints table at [https://docs.temporal.io/serverless-workers#constraints](https://docs.temporal.io/serverless-workers#constraints).

## Why this happens

Two independent facts combine to produce the constraint:

1. **Compute provider configuration is per-Worker-Deployment-Version and pins a region.** The compute provider "is set on a Worker Deployment Version and specifies the provider type, the invocation target, and the credentials Temporal needs to trigger the invocation."  For AWS Lambda, the invocation target is a Lambda function ARN, which embeds the AWS region.  For GCP Cloud Run, the target is a Worker Pool whose region is set explicitly by `--gcp-cloud-run-region`.

2. **The WCI has no failover-detection or cross-region redirect.** "One WCI Workflow runs per Worker Deployment Version that has a compute provider configured. The WCI runs in the same Namespace as your Worker Deployment."  The WCI's role is to scale Workers up and down in response to Task Queue conditions by triggering the configured compute provider.  It has no branch that swaps compute provider region on failover; the HA docs are explicit that it "has no mechanism to detect a failover or redirect invocations to a Worker in the new active region."

Contrast with long-lived Workers: for a long-lived fleet, "the DNS redirection orchestrated by Temporal ensures that your existing Workers continue to poll the Namespace without interruption. Temporal Cloud forwards their requests from the passive replica to the active region and the responses back, so Workers keep running through a failover."  That mechanism operates at the *Client-to-Namespace* layer: a Worker process resolves the Namespace Endpoint and re-resolves when a connection cycles.  Serverless Workers do not have a persistent poller in your infrastructure that could re-resolve DNS; each invocation is created *by Temporal* against the compute provider you configured, and that configuration is regional.

For the DNS-based failover mechanics that long-lived Workers rely on, see [`references/triage/ha-failover.md`](./ha-failover.md). Do not re-derive that here.

## Diagnostic signal — did a failover happen?

You are usually called in because Workflows are stalled, latency is up, or a customer reported an outage. Before doing anything provider-specific, confirm that a Namespace failover actually occurred and note the new active region.

- **Web UI.** "After any failover, whether triggered by you or by Temporal, an event appears in both the [Temporal Cloud Web UI](https://cloud.temporal.io/namespaces) (on the Namespace detail page) and in your audit logs."  Open the Namespace detail page and inspect the failover list.
- **Audit log signal.** "The audit log entry uses the `\"operation\": \"FailoverNamespace\"` event."  Temporal Cloud also emails admins on every failover.
- **What long-lived Workers do vs. Serverless Workers.** Long-lived Workers follow the Namespace Endpoint DNS change and, once existing connections cycle (Temporal Cloud enforces a maximum connection lifetime of 5 minutes), re-resolve to the new active region.  Serverless Workers do not — the WCI keeps calling the compute provider you configured until you repoint it. If your fleet is mixed, expect long-lived Workers to recover on their own within a few minutes while Serverless-only Task Queues stay wedged until you act. Note also that the Namespace Endpoint DNS change itself "can take a few minutes to fully propagate to all Clients and Workers."

If a `FailoverNamespace` event has fired and the new active region is not the same region as your compute provider's Lambda ARN or Cloud Run Worker Pool, you are in the constraint. Proceed to remediation.

## Manual remediation

`tcld namespace failover --namespace <namespace_id>.<account_id> --region <target_region>` fails over the *Namespace*.  It does not modify any Worker Deployment Version's compute provider. The documented remediation is to create a new Worker Deployment Version pointing at a Worker in the new active region, then set that Version current.

The compute provider is configured at Version creation time via `temporal worker deployment create-version` with provider-specific flags. Changing the target region means creating a *new* Version — the two provider flag surfaces are described below. After creating the new Version, promote it with `temporal worker deployment set-current-version`; "without this step, tasks on the Task Queue will not route to the version, and Temporal will not invoke the Lambda function."  The equivalent guidance holds for Cloud Run: without setting the version current, "Tasks on the Task Queue will not route to the version, and the WCI will not start any instances."

### AWS Lambda

**Prerequisite (AWS side).** A Lambda function must already exist in the new active AWS region, with an IAM trust configuration that lets Temporal assume the invocation role. The full AWS setup is out of scope for this file — hand off to [https://docs.temporal.io/production-deployment/worker-deployments/serverless-workers/aws-lambda](https://docs.temporal.io/production-deployment/worker-deployments/serverless-workers/aws-lambda) for the deployment guide.

**Create a new Worker Deployment Version pointing at the new region's Lambda ARN.** The Lambda ARN embeds the region, so pointing at a new-region function is the entire point of the remediation.

```bash
temporal worker deployment create-version \
  --namespace <YOUR_NAMESPACE> \
  --deployment-name my-app \
  --build-id build-1-<new-region> \
  --aws-lambda-function-arn <NEW_REGION_LAMBDA_FUNCTION_ARN> \
  --aws-lambda-assume-role-arn <INVOCATION_ROLE_ARN> \
  --aws-lambda-assume-role-external-id <EXTERNAL_ID>
```

Flags in play (all transcribed from the AWS Lambda doc):

- `--deployment-name` — Worker Deployment name. Must match `DeploymentName` in your Worker code.
- `--build-id` — Worker Deployment Version build ID. Must match `BuildID` in your Worker code.
- `--aws-lambda-function-arn` — Qualified versioned ARN of the Lambda function Temporal invokes for this version (for example, `function:my-worker:5`).
- `--aws-lambda-assume-role-arn` — IAM role Temporal assumes to invoke the function.
- `--aws-lambda-assume-role-external-id` — External ID configured in the IAM role trust policy.

**Set the new Version current.**

```bash
temporal worker deployment set-current-version \
  --deployment-name my-app \
  --build-id build-1-<new-region>
```

### GCP Cloud Run

**Prerequisite (GCP side).** A Cloud Run Worker Pool must exist in the new active region, in a project Temporal's invoker service account can impersonate. The Cloud Run setup — Worker Pool creation, invoker IAM, Terraform — is out of scope here; hand off to [https://docs.temporal.io/production-deployment/worker-deployments/serverless-workers/cloud-run](https://docs.temporal.io/production-deployment/worker-deployments/serverless-workers/cloud-run).

**Create a new Worker Deployment Version with `--gcp-cloud-run-region` set to the new active region and the new pool's name/project/service-account.**

```bash
temporal worker deployment create-version \
  --namespace <YOUR_NAMESPACE> \
  --deployment-name my-app \
  --build-id build-1-<new-region> \
  --gcp-cloud-run-project <YOUR_GCP_PROJECT> \
  --gcp-cloud-run-region <NEW_REGION> \
  --gcp-cloud-run-worker-pool <NEW_REGION_WORKER_POOL> \
  --gcp-cloud-run-service-account <INVOKER_SERVICE_ACCOUNT>
```

Flags in play (all transcribed from the Cloud Run doc):

- `--deployment-name` — Worker Deployment name. Must match `deployment_name` in your Worker code.
- `--build-id` — Worker Deployment Version build ID. Must match `build_id` in your Worker code.
- `--gcp-cloud-run-project` — GCP project ID that contains the Worker Pool.
- `--gcp-cloud-run-region` — Region of the Worker Pool.
- `--gcp-cloud-run-worker-pool` — Name of the Worker Pool.
- `--gcp-cloud-run-service-account` — The invoker service account Temporal impersonates to read and scale the pool.

**Set the new Version current.**

```bash
temporal worker deployment set-current-version \
  --namespace <YOUR_NAMESPACE> \
  --deployment-name my-app \
  --build-id build-1-<new-region>
```

The Cloud Run `set-current-version` command asks for confirmation because it changes which version new Tasks route to; pass `--yes` to skip the prompt in an incident.

## Prevention: dual-region compute-provider pre-provisioning

The docs establish that a compute provider is per-Version and per-region, and that the mutation verbs available are `create-version` and `set-current-version`. From that alone, the operator-facing implication is clear: if you keep a Worker Deployment Version already configured for each region *in advance*, then failover-time work reduces to a single `set-current-version` call against the pre-existing new-region Version. You skip the AWS-side or GCP-side provisioning under time pressure.

This means, for each Worker Deployment:

- Publish the underlying compute artifact (Lambda function, Cloud Run Worker Pool) in every region you might fail over to, before you need it.
- Pre-create one Worker Deployment Version per region using `temporal worker deployment create-version` with that region's provider flags. Use distinct `--build-id` values so each Version is addressable.
- On failover, promote the Version whose compute provider is in the new active region with `temporal worker deployment set-current-version --deployment-name ... --build-id ...`.

The docs stop short of prescribing a specific automation pattern (Terraform module, runbook script, or CI hook) for this pre-provisioning. Do not invent one on the operator's behalf. The Temporal Cloud Terraform provider does not support triggering failovers  — that fact is documented for the Namespace-failover path, not the Worker-Deployment-Version path — so treat any Terraform-provider claim about `create-version`/`set-current-version` as something to look up in the CLI reference rather than assert from this file.

## Hand-off pointers

- **Worker placement architecture (Active-Passive vs. Active-Active, latency/cost trade-offs across regions):** see the [`skill-temporal-deploy`](https://docs.temporal.io/cloud/high-availability/architecture-patterns) skill's deployment-pattern references and [https://docs.temporal.io/cloud/high-availability/architecture-patterns](https://docs.temporal.io/cloud/high-availability/architecture-patterns).
- **`tcld` and `temporal` flag semantics in depth (all flags on `namespace failover`, `worker deployment create-version`, `worker deployment set-current-version`):** hand off to the `skill-temporal-cli` skill; that skill owns exhaustive CLI reference and stays authoritative when flag surfaces change.
- **Serverless Worker SDK code (registering Workflows/Activities, using the serverless Worker package for your language SDK, Worker Versioning `AutoUpgrade` vs `Pinned` behavior):** hand off to the `skill-temporal-developer` skill.
- **Long-lived Worker failover behavior (DNS/Namespace Endpoint propagation, the 5-minute connection lifetime, Regional Endpoint escape hatch):** see [`references/triage/ha-failover.md`](./ha-failover.md) in this skill.
- **How to trigger the Namespace failover itself (`tcld`, Web UI, Cloud Ops API):** [https://docs.temporal.io/cloud/high-availability/failovers/manage#trigger-failover](https://docs.temporal.io/cloud/high-availability/failovers/manage#trigger-failover). This file assumes the failover already happened.
