# Temporal Triage Skill

A skill for diagnosing Temporal failures — stuck workflows, non-determinism errors, connectivity problems, certificate expirations, worker health issues, and rate limits.

Applies to both **Temporal Cloud** and **self-hosted** deployments. The diagnostic discipline (bottom-up layered triage) and most reference files (connectivity, certificates, worker health, stuck workflows, non-determinism, replay, rate limits, runtime errors) cover both. A small number of items are Cloud-only.

> [!WARNING]
> This skill is not yet released. It is being shared internally to gather feedback before a public release.
> Please send feedback — positive or negative — to the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY) on the Temporal Slack.

## Installation

While this skill is unreleased, install it by cloning the repo directly:

```bash
mkdir -p ~/.claude/skills && git clone https://github.com/temporalio/skill-temporal-triage ~/.claude/skills/temporal-triage
```

Adjust the installation directory based on your coding agent. Plugin-marketplace and `npx skills` installation will be documented here once the skill is released.

## What this skill covers

Diagnostic flow for common Temporal failures:

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

- **CLI commands and flags** — use `skill-temporal-cli`
- **Writing workflows/activities** — use `skill-temporal-developer`
- **Worker performance tuning, scaling, capacity planning** — use `skill-temporal-deploy`
- **SDK-specific ergonomics** — use `skill-temporal-developer`
- **Metrics interpretation and dashboards** — use `skill-temporal-observability`
