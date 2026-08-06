# Limiting what an agent can do to your Temporal environment

This skill lets a coding agent run `temporal` and `tcld` commands against whatever
environment your CLIs are pointed at. Some of those commands have no undo:
`tcld namespace delete` removes every Workflow Execution and Task Queue
immediately, `tcld apikey delete` breaks every Worker still presenting the key, and
the `--query` form of `temporal workflow terminate` fans out across every matching
Execution in the Namespace.

The skill instructs the agent to establish a blast radius, propose the command, and
wait for your approval before running anything in that category. That is a
behavior, not a boundary — it depends on the agent following instructions, and it
offers no protection against a mistake you approve because the proposal looked
reasonable.

**It is strongly recommended to manually review every command the agent seeks to perform against your Temporal account and Namespaces.**

If you want a boundary, it has to live in your environment. Three tiers below,
strongest first. They compose, and the first one is worth doing even if you skip
the others.

---

## Tier 1: scope the credentials you hand the agent

You have to authenticate `tcld` and `temporal` before the agent, with or without the skill, can take action against Temporal. Be judicious about which credentials you give access to which accounts and Namespaces. Understand that LLM-based AI agents are non-deterministic and have a documented history of making mistakes.

It's recommended to give an agent session a credential that cannot perform the operations you would not approve. This holds regardless of which agent tool you use.

- **Use a scoped Service Account, not your own login.** Temporal Cloud has
  account-level roles and per-Namespace permissions.
- **Point the default profile somewhere harmless.** Both CLIs resolve connection
  settings from flags, then `TEMPORAL_*` / `TEMPORAL_CLOUD_NAMESPACE` env vars,
  then a config-file profile — so a command with no explicit `--namespace` lands
  wherever the environment says. Give agent sessions a profile targeting a dev
  Namespace and require an explicit `--namespace` for anything else. See
  `references/ops/cli-conventions.md`.
- **Turn on delete protection** for Namespaces that should never be deleted
  (`--enable-delete-protection` at create time, or `tcld namespace lifecycle set`
  afterwards). This one is enforced server-side, independent of any credential.
  See `references/ops/cloud-namespace-admin.md`.
- **Keep the audit trail.** Control-plane mutations land in Cloud Audit Logs, so
  "what did it actually do" is answerable after the fact. See
  `references/ops/cloud-audit-logs.md`.

---

## Tier 2: deny specific commands in your agent tool

Most coding agents can allow or deny shell commands by pattern. Denying the handful
of operations you would never want an agent to run — as opposed to ones you want
the *option* of approving — means those commands simply aren't available, and you
run them yourself when you mean to.

Reasonable candidates: `tcld namespace delete`, `tcld namespace delete-region`,
`tcld namespace failover`, and the identity deletes (`tcld apikey delete`,
`tcld user delete`, `tcld user-group delete`, `tcld service-account delete`).

**Caveat that applies to every tool in this tier:** pattern matching sees command
*text*, and the same operation has many spellings. `tcld` subcommands have
single-letter aliases — `namespace` is `n` and `delete` is `d` — so `tcld n d` is
the same call as `tcld namespace delete` with no shared substring beyond `tcld`.
Add shell variables, a `$(...)` substitution, or a wrapper script and text matching
loses outright. Write these patterns to over-trigger, and treat the tier as a speed
bump rather than a sandbox. Tier 1 is what holds.

<details>
<summary>Example: Claude Code</summary>

In `.claude/settings.json` (project) or `~/.claude/settings.json` (all projects). A
`deny` rule cannot be overridden by the model or by an `allow` rule, and applies in
every permission mode:

```json
{
  "permissions": {
    "deny": [
      "Bash(tcld namespace delete:*)",
      "Bash(tcld namespace delete-region:*)",
      "Bash(tcld namespace failover:*)",
      "Bash(tcld apikey delete:*)",
      "Bash(tcld service-account delete:*)"
    ]
  }
}
```

These are prefix patterns, so alias spellings walk past them — tier 3 is what
covers those. Confirm the rules parsed with `/permissions`; a pattern that silently
fails to parse looks identical to one that is working.

</details>

---

## Tier 3: intercept commands before they execute

If your agent tool supports a pre-execution hook or callback, you can inspect the
full command string, apply a regex that catches alias spellings, and decide per
command whether to refuse it or surface it to you as a prompt.

The design point that matters more than the regex: **use two responses, not one.**

- **Refuse** the operations with no undo. You run those by hand.
- **Prompt** for fan-out operations (`--query`) and for prompt-suppressing flags
  (`--yes`), rather than refusing them.

The reason to prompt rather than refuse on the second group is specific to how
these CLIs work. The `--query` batch form asks `Start batch against approximately N
workflow(s)? y/N`, and that prompt needs a terminal — under an agent there isn't
one, so the command reports `user denied confirmation` and does nothing. `--yes` is
therefore how an *approved* batch actually runs. Refuse it outright and the agent is
left holding an approved action it cannot perform, and the available workaround is a
loop over single-target `temporal workflow terminate --workflow-id`, which prompts
for nothing, ignores `--rps`, and cannot be stopped with `temporal batch terminate`.
Blocking the controlled form of a fan-out pushes the work toward the uncontrolled
one.

Matching `--yes` is still worthwhile: it is a reliable signal that a fan-out is
about to run, and it suppresses the count the prompt would have printed. Prompting
on it puts that number back in front of a human.

<details>
<summary>Example: Claude Code</summary>

Save as `.claude/hooks/temporal-destructive.sh`, `chmod +x`, and register it under
`hooks.PreToolUse` with matcher `Bash`:

```bash
#!/usr/bin/env bash
# Deliberately over-matches. Tier A refuses; tier B prompts.
cmd=$(jq -r '.tool_input.command // ""')
grep -qE '(^|[;&|[:space:]])(tcld|temporal)[[:space:]]' <<<"$cmd" || exit 0

# Tier A — no undo. Exit 2 blocks and feeds the message back to the agent.
never='\b(namespace[[:space:]]+(delete|delete-region|failover)|(apikey|user|user-group|service-account)[[:space:]]+delete)\b'
alias='\btcld[[:space:]]+(n|ak|u|ug|sa)[[:space:]]+(d|dr|f)\b'   # tcld n d, tcld sa d
if grep -qE "$never|$alias" <<<"$cmd"; then
  echo "Blocked: irreversible Temporal control-plane operation. Present it with the target Namespace; the user will run it." >&2
  exit 2
fi

# Tier B — fan-out / prompt suppression. Ask, don't block.
if grep -qE '(^|[[:space:]])(--query|-q|--yes|-y)([[:space:]]|=|$)' <<<"$cmd"; then
  cat <<'JSON'
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask",
"permissionDecisionReason":"Fan-out or prompt-suppressing Temporal command. Confirm the count from `temporal workflow count` and the target Namespace before approving."}}
JSON
fi
exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/temporal-destructive.sh" }]
      }
    ]
  }
}
```

Verify with `claude --debug` before relying on it, and confirm the `ask` decision
shape against the hook reference for your version — if it does not apply cleanly,
fall back to exiting 2 for tier B as well and run those commands yourself.

</details>

---

## After the fact

Not every destructive operation leaves a trail you can act on, and a batch job
drains asynchronously. Once it starts, the useful question is no
longer whether it was approved but how far it has gotten:
`temporal batch describe --job-id <id>` answers that, and
`temporal batch terminate --job-id <id>` stops the remainder. Executions it already
acted on are not recoverable.
