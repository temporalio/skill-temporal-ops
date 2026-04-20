# Temporal Triage Skill

A skill for diagnosing Temporal failures — stuck workflows, non-determinism errors, connectivity problems, certificate expirations, worker health issues, rate limits, and HA failover trouble.

> [!WARNING]
> This Skill is currently in Public Preview, and will continue to evolve and improve.
> We would love to hear your feedback - positive or negative - over in the [Community Slack](https://t.mp/slack), in the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY).

## Installation

### As a Claude Code Plugin

1. Run `/plugin marketplace add temporalio/agent-skills`
2. Run `/plugin` to open the plugin manager
3. Select **Marketplaces**
4. Choose `temporal-marketplace` from the list
5. Select **Enable auto-update** or **Disable auto-update**
6. Run `/plugin install temporal-triage@temporalio-agent-skills`
7. Restart Claude Code

### Via `npx skills` — supports all major coding agents

1. `npx skills add temporalio/skill-temporal-triage`
2. Follow prompts

### Via manually cloning the skill repo

1. `mkdir -p ~/.claude/skills && git clone https://github.com/temporalio/skill-temporal-triage ~/.claude/skills/temporal-triage`

Appropriately adjust the installation directory based on your coding agent.

## What this skill covers

Diagnostic flow for common Temporal failures:

- **Stuck workflows** — Event-History-driven triage that identifies the last event, classifies the cause, and proposes a fix
- **Non-determinism errors** — detection, local replay reproduction, and fix patterns
- **Connectivity failures** — can't connect, TLS handshake errors, endpoint mismatches, DNS issues
- **Certificate problems** — x509 errors, expiry, chain verification, openssl diagnostics
- **Authentication failures** — `tcld login` failures, API key expiry, scope/role mismatches
- **Worker health** — no pollers, task-queue backlog, worker registration, build-id mismatches
- **Rate limiting** — `RESOURCE_EXHAUSTED`, APS limits, retry-after patterns
- **HA failover** — multi-region endpoint, DNS staleness, failover verification
- **Ambiguous runtime errors** — `context deadline exceeded`, `workflow is busy`
- **Replay debugging** — using the VS Code debugger extension to step through workflow history

## What this skill does NOT cover

- **CLI commands and flags** — use `skill-temporal-cli`
- **Writing workflows/activities** — use `skill-temporal-developer`
- **Worker performance tuning, scaling, capacity planning** — use `skill-temporal-deploy`
- **SDK-specific ergonomics** — use `skill-temporal-developer`
- **Metrics interpretation and dashboards** — use `skill-temporal-observability`
