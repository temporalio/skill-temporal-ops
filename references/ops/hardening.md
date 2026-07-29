# Hardening: making destructive-operation approval enforceable

The [Destructive operations](../../SKILL.md#destructive-operations) policy in
SKILL.md is guidance an agent follows. It is not a control the environment
enforces. Guidance degrades under pressure — a user saying "just terminate all of
them, I already told you it's fine" is in direct competition with it, and the
policy has no way to win that argument on its own.

If you need destructive commands to be *unable* to run without a human in the
loop, the mechanism has to live outside the model. This file describes three
layers, weakest to strongest. They compose; none of them replaces the others.

Everything here is **opt-in configuration for your own environment**. A skill
cannot install it for you — `.claude/settings.json` and hooks are yours, not the
skill's.

---

## Why command-text matching is a speed bump, not a sandbox

Layers 1 and 2 match the *text* of a shell command, and the same operation has
many spellings.

`tcld` subcommands have single-letter aliases — `namespace` is `n` and `delete` is
`d` <!-- docs/cloud/tcld/namespace.mdx:22,299 --> so `tcld n d -n <ns>` and
`tcld namespace delete -n <ns>` are the same call with no shared substring beyond
`tcld`. Add shell variables, a `$(...)` substitution, a wrapper script, or an
alias defined in the user's own profile, and text matching loses outright.

So: write these rules to **over-trigger**. A rule that stops to ask about a
harmless `list` costs a keystroke. A rule that misses one spelling of
`namespace delete` costs the Namespace. Only layer 3 is spelling-independent.

---

## Layer 1: `deny` rules in settings.json

Cheapest, no scripting. Put these in `.claude/settings.json` (project) or
`~/.claude/settings.json` (all projects). A `deny` rule cannot be overridden by
the model or by an `allow` rule, and it is not subject to any permission mode —
including `bypassPermissions`.

```json
{
  "permissions": {
    "deny": [
      "Bash(tcld namespace delete:*)",
      "Bash(tcld namespace delete-region:*)",
      "Bash(tcld namespace failover:*)",
      "Bash(tcld apikey delete:*)",
      "Bash(tcld user delete:*)",
      "Bash(tcld user-group delete:*)",
      "Bash(tcld service-account delete:*)",
      "Bash(temporal operator namespace delete:*)"
    ]
  }
}
```

This is a hard block, not a prompt: the agent cannot run these at all, and you
run them yourself when you actually mean to. That is the right trade for the
handful of operations with no undo.

Two things to know. These are **prefix** patterns, so every alias spelling above
walks straight past them — layer 2 is what covers those. And because the block is
absolute, keep this list to operations you are content to always run by hand;
anything you want the *option* of approving belongs in layer 2 with `ask`.

Confirm the patterns took effect with `/permissions` — pattern syntax has varied
across Claude Code versions, and a rule that silently fails to parse looks
identical to a rule that is working.

---

## Layer 2: a `PreToolUse` hook

A hook sees the full command string and can apply a regex, so it catches alias
spellings that layer 1 misses, and it can *ask* rather than refuse.

Save as `.claude/hooks/temporal-destructive.sh` and `chmod +x` it:

```bash
#!/usr/bin/env bash
# PreToolUse hook: route destructive temporal/tcld commands to a human.
# Deliberately over-matches. Two tiers: hard block, and prompt.
cmd=$(jq -r '.tool_input.command // ""')

# Not a Temporal CLI call at all.
grep -qE '(^|[;&|[:space:]])(tcld|temporal)[[:space:]]' <<<"$cmd" || exit 0

# Tier 1 — no undo, never automate. Hard block; run these by hand.
never='\b(namespace[[:space:]]+(delete|delete-region|failover)|(apikey|user|user-group|service-account)[[:space:]]+delete)\b'
# Same operations spelled with tcld's single-letter aliases: `tcld n d`, `tcld sa d`.
alias='\btcld[[:space:]]+(n|ak|u|ug|sa)[[:space:]]+(d|dr|f)\b'
if grep -qE "$never|$alias" <<<"$cmd"; then
  echo "Blocked: irreversible Temporal control-plane operation." >&2
  echo "Present it to the user with the Namespace it resolves to; they will run it themselves." >&2
  exit 2
fi

# Tier 2 — fan-out and prompt suppression. Prompt, don't block.
if grep -qE '(^|[[:space:]])(--query|-q|--yes|-y)([[:space:]]|=|$)' <<<"$cmd"; then
  cat <<'JSON'
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask",
"permissionDecisionReason":"Fan-out or prompt-suppressing Temporal command. Confirm the blast-radius count from `temporal workflow count` and the target Namespace before approving."}}
JSON
fi
exit 0
```

Register it:

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

Exit code 2 blocks the call and feeds stderr back to the agent, which is what
turns tier 1 into a useful instruction rather than an opaque failure. Tier 2
returns a permission decision of `ask` instead, so the command reaches the user as
a prompt and runs on approval. Confirm the JSON shape against the hook reference
for your version and verify with `claude --debug` before relying on it; if the
`ask` decision does not apply cleanly, fall back to exiting 2 for tier 2 as well
and run those commands yourself.

The tiering matters more than the regex. Tier 2 has to prompt rather than block,
because `--yes` is exactly what an approved batch needs in order to finish — the
CLI's interactive confirmation requires a terminal, and without one the command
reports `user denied confirmation` and does nothing. Hard-blocking the flag leaves
an agent holding an approved action it cannot perform, and the available
workaround is a loop over single-target `temporal workflow terminate
--workflow-id`, which prompts for nothing, ignores `--rps`, and cannot be stopped
with `temporal batch terminate`. Blocking the controlled form of a fan-out pushes
the work toward the uncontrolled one.

What makes `--yes` worth matching is that it is a reliable signal a fan-out is
about to run, and the flag also suppresses the count the prompt would have
printed. Prompting on it puts that number back in front of a human.

---

## Layer 3: don't give the agent credentials that can do it

The only layer that survives a bypassed harness, a novel command spelling, or a
human running the command by hand in the wrong terminal.

- **Scope the credential.** Run agent sessions under a Service Account whose
  Namespace permissions are read-only, or Namespace Admin only on a scratch
  Namespace. A `tcld namespace delete` that returns `PERMISSION_DENIED` needed no
  regex to stop it. See [cloud-iam.md](cloud-iam.md) for roles and Namespace
  permissions.
- **Turn on delete protection** for Namespaces that should never be deleted, via
  `--enable-delete-protection` at create time. See
  [cloud-namespace-admin.md](cloud-namespace-admin.md#delete-protection). If a
  delete is refused because protection is on, that is a deliberate decision by
  whoever provisioned it — report the block and stop.
- **Isolate the default target.** Both CLIs resolve connection settings from
  `TEMPORAL_*` env vars and config-file profiles, so a command with no explicit
  `--namespace` can land somewhere you did not intend. Give agent sessions a
  profile that points at a dev Namespace, and require an explicit `--namespace`
  for anything else. See
  [cli-conventions.md](cli-conventions.md#connection-and-identity).
- **Keep the audit trail.** Control-plane mutations land in Cloud Audit Logs, so
  you can answer "what did it actually do" after the fact. See
  [cloud-audit-logs.md](cloud-audit-logs.md).

---

## What none of this covers

Data-plane operations do not all leave a trail. Activity Operations
(`temporal activity pause | unpause | reset`) produce no Event History events, so
they leave no audit record beyond the current `describe` output
<!-- docs/encyclopedia/activities/activity-operations.mdx:269-276 --> — a hook
that lets one through is the only record that it happened.

And a batch job drains asynchronously. Once started, the useful question is no
longer "was it approved" but "how far has it gotten":
`temporal batch describe --job-id <id>` answers that and
`temporal batch terminate --job-id <id>` stops the remainder. Executions it
already acted on are not recoverable. See
[cli-conventions.md](cli-conventions.md#inspecting-and-aborting-a-running-job).
