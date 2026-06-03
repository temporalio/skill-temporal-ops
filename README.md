# Temporal Ops Skill

A skill for operating and diagnosing Temporal environments — namespace administration, capacity management, IAM, certificate rotation, workflow health queries, batch operations, plus bottom-up diagnosis of stuck workflows, non-determinism errors, connectivity problems, certificate expirations, worker health issues, and rate limits.

Applies to both **Temporal Cloud** (`tcld` commands) and **self-hosted** (`temporal operator` commands) deployments. Data-plane operations (`temporal workflow`, `temporal batch`, `temporal schedule`) work on both.

> [!WARNING]
> This skill is not yet released. It is being shared internally to gather feedback before a public release.
> Please send feedback — positive or negative — to the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY) on the Temporal Slack.

## Installation

While this skill is unreleased, install it by cloning the repo directly:

```bash
mkdir -p ~/.claude/skills && git clone https://github.com/temporalio/skill-temporal-ops ~/.claude/skills/temporal-ops
```

Adjust the installation directory based on your coding agent. Plugin-marketplace and `npx skills` installation will be documented here once the skill is released.

## What this skill covers

### Operations

- **Cloud namespace admin** — create, get, list, delete namespaces; add/remove regions; failover; HA config; retention, tags, codec-server, connectivity rules, search attributes, accepted-client-ca, certificate filters, export
- **Cloud capacity** — On-Demand vs Provisioned modes, APS/RPS/OPS, TRUs, capacity updates, throttling, APS management
- **Cloud IAM** — API key lifecycle, user management, user groups, service accounts, roles, namespace permissions
- **Cloud certificates** — mTLS cert generation, CA upload, certificate filters, rotation, auth method switching
- **Cloud export & connectivity** — Workflow History Export (S3/GCS), PrivateLink/PSC, connectivity rules, Cloud Ops API
- **Self-hosted admin** — cluster health, namespace CRUD, search attributes, Nexus endpoints via `temporal operator`
- **Workflow health queries** — List Filter queries to find stuck/hung workflows, task-queue poller status, workflow counts
- **Batch & lifecycle** — cancel, terminate, reset workflows; batch operations; schedules; external activity completion

### Diagnosis

- **Stuck workflows** — Event-History-driven triage that identifies the last event, classifies the cause, and proposes a fix
- **Non-determinism errors** — detection, local replay reproduction, and fix patterns
- **Connectivity failures** — can't connect, TLS handshake errors, endpoint mismatches, DNS issues
- **Certificate problems** — x509 errors, expiry, chain verification, openssl diagnostics
- **Authentication failures** — `UNAUTHENTICATED` vs `PERMISSION_DENIED`, mTLS identity mapping; Cloud adds `tcld login`, API key expiry, and Cloud role/namespace-permission mismatches
- **Worker health** — no pollers, task-queue backlog, worker registration, build-id mismatches
- **Rate limiting** — `RESOURCE_EXHAUSTED`, Cloud APS/RPS/OPS limits, self-hosted `frontend.rps` / `frontend.namespaceRPS` dynamic config, retry-after patterns
- **HA failover** (Temporal Cloud) — multi-region endpoint, DNS staleness, failover verification
- **Ambiguous runtime errors** — `context deadline exceeded`, `workflow is busy`
- **Replay debugging** — using the VS Code debugger extension to step through workflow history

## What this skill does NOT cover

- **Writing workflows/activities** — use `skill-temporal-developer`
- **Worker performance tuning, scaling, capacity planning** — use `skill-temporal-workertuning`
- **SDK-specific ergonomics** — use `skill-temporal-developer`
