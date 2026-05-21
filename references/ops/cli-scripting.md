# CLI Scripting

Driving `temporal` and `tcld` from shell scripts: output formats, pagination,
log-channel separation, exit codes, `jq` patterns, environment variables,
stored environments, and operational gotchas.

---

## 1. Output formatting

### `--output` enum

Every `temporal` command accepts `--output` (`-o`) as a global flag: <!-- docs/cli/env.mdx:138 -->

| Value | Description |
|-------|-------------|
| `text` (default) | Human-readable tables/text. <!-- docs/cli/env.mdx:138 --> |
| `json` | Structured JSON blob. <!-- docs/cli/env.mdx:138 --> |
| `jsonl` | JSON-lines (one JSON value per line). <!-- docs/cli/env.mdx:138 --> |
| `none` | Suppress non-logging data output. <!-- docs/cli/env.mdx:138 --> |

```bash
temporal workflow list --output json --limit 50
temporal workflow show --workflow-id greet-1 --output json > history.json
temporal workflow describe --workflow-id greet-1 --output jsonl
```

### `--time-format`

| Value | Description |
|-------|-------------|
| `relative` (default) | "2 minutes ago"-style human-friendly text. <!-- docs/cli/env.mdx:140 --> |
| `iso` | ISO 8601 / RFC 3339 timestamps -- parseable. <!-- docs/cli/env.mdx:140 --> |
| `raw` | Raw underlying representation. <!-- docs/cli/env.mdx:140 --> |

Scripts should use `iso`:

```bash
temporal workflow list --output json --time-format iso --limit 100
```

### `--no-json-shorthand-payloads`

By default, `--output json` substitutes a shorthand representation for Payload fields. Pass `--no-json-shorthand-payloads` to get the wire-format Payload envelope (with `metadata.encoding` and base64 `data`): <!-- docs/cli/env.mdx:137 -->

```bash
temporal workflow show \
    --workflow-id YourWorkflowId \
    --output json \
    --no-json-shorthand-payloads > history-raw.json
```

Use cases: SDK replay, forwarding to a tool that decodes with a Payload Codec.

---

## 2. Pagination

| Flag | Purpose |
|------|---------|
| `--limit` | Maximum number of Workflow Executions to display. <!-- docs/cli/workflow.mdx:303 --> |
| `--page-size` | Maximum number of Workflow Executions to fetch at a time from the server. <!-- docs/cli/workflow.mdx:304 --> |

`--limit` caps total rows printed; `--page-size` tunes the per-RPC fetch chunk. There is no documented token-paging flag to resume from a specific cursor.

```bash
# Bounded list
temporal workflow list \
    --query 'ExecutionStatus="Failed"' \
    --output json --time-format iso --limit 1000

# Streamed list, one JSON object per line
temporal workflow list \
    --query 'WorkflowType="LegacyWorkflow"' \
    --output jsonl --time-format iso --limit 10000 | ...
```

For larger result sets, prefer `--output jsonl` and raise `--limit`.

---

## 3. Log channel separation

`--log-format` and `--log-level` control the CLI's own log output, not the data that `--output` prints. Logs go to stderr; structured output goes to stdout. <!-- docs/cli/env.mdx:134-135 -->

| Flag | Accepted values | Default |
|------|-----------------|---------|
| `--log-format` | `text`, `json` <!-- docs/cli/env.mdx:134 --> | `text` |
| `--log-level` | `debug`, `info`, `warn`, `error`, `never` <!-- docs/cli/env.mdx:135 --> | `never` for most commands; `warn` for `server start-dev` |

```bash
# Machine-readable logs alongside machine-readable data
temporal workflow list \
    --output json --time-format iso \
    --log-format json --log-level warn \
    --limit 500 > workflows.json 2> workflows.log
```

`--color` (values: `always`, `never`, `auto`; default `auto`) controls ANSI colour. Scripts can leave it on `auto` (suppressed when stdout is not a TTY) or force `never`. <!-- docs/cli/env.mdx:125 -->

---

## 4. Exit codes

The docs do not enumerate specific exit-code numbers. Convention: zero on success, non-zero on failure; failure diagnostics land on stderr.

`temporal workflow execute` blocks until the run terminates and streams progress. A non-zero exit indicates the workflow did not complete successfully (failed, cancelled, terminated, or timed out). <!-- docs/cli/workflow.mdx:159-204 -->

Scripts that need the failure reason should parse stderr or re-fetch the execution with `temporal workflow describe --output json`.

---

## 5. Piping into `jq`

The CLI does not publish a formal JSON schema. Treat the following patterns as shape-generic:

```bash
# Slurp a JSON list into jq
temporal workflow list --output json --time-format iso --limit 500 | jq '.'

# Stream JSON-lines through jq one record at a time
temporal workflow list --output jsonl --time-format iso --limit 5000 \
  | jq -c 'select(.someField == "someValue")'

# Grab a field from a single describe
temporal workflow describe \
    --workflow-id YourWorkflowId \
    --output json \
  | jq '.'

# Send raw payload envelopes to a downstream decoder
temporal workflow show \
    --workflow-id YourWorkflowId \
    --output json --no-json-shorthand-payloads \
  | jq '.'
```

**Anti-pattern:** hard-coding field names like `.execution.workflowId` without verifying against the actual output of your installed CLI. The docs do not freeze these names. Run the command once, inspect with `jq 'keys'` / `jq '.[0] | keys'`, then write the selector. <!-- docs/cli/workflow.mdx:898 -->

---

## 6. `tcld` JSON output

`tcld` has one documented global modifier: `--auto_confirm` (env var `AUTO_CONFIRM`, default `false`). <!-- docs/cloud/tcld/index.mdx:37-41 -->

There is no `--output`, `--format`, or `-o` flag. `tcld` emits JSON to stdout by design. Pipe directly into `jq`:

```bash
tcld namespace get --namespace payments.abc123 | jq '.'
tcld namespace list | jq '.'
```

Route errors by capturing stderr separately:

```bash
if ! RESULT=$(tcld namespace get --namespace payments.abc123 2>/tmp/tcld.err); then
  echo "tcld failed:" >&2
  cat /tmp/tcld.err >&2
  exit 1
fi
echo "$RESULT" | jq '.'
```

---

## 7. Environment variables

Full `TEMPORAL_*` env-var to flag mapping: <!-- docs/cli/index.mdx:267-279 -->

| Variable | Maps to | Purpose |
|----------|---------|---------|
| `TEMPORAL_ADDRESS` | `--address` | Host:port for the Temporal Frontend Service. <!-- docs/cli/index.mdx:269 --> |
| `TEMPORAL_NAMESPACE` | `--namespace` | Default `"default"`. <!-- docs/cli/index.mdx:272 --> |
| `TEMPORAL_API_KEY` | `--api-key` | API key for request. Picked up by both `temporal` and `tcld`. <!-- docs/cli/index.mdx:278 --> |
| `TEMPORAL_TLS_CA` | `--tls-ca-path` | Path to server CA certificate. <!-- docs/cli/index.mdx:273 --> |
| `TEMPORAL_TLS_CERT` | `--tls-cert-path` | Path to x509 client certificate. <!-- docs/cli/index.mdx:274 --> |
| `TEMPORAL_TLS_KEY` | `--tls-key-path` | Path to private certificate key. <!-- docs/cli/index.mdx:276 --> |
| `TEMPORAL_TLS_SERVER_NAME` | `--tls-server-name` | SNI override for target TLS server name. <!-- docs/cli/index.mdx:277 --> |
| `TEMPORAL_TLS_DISABLE_HOST_VERIFICATION` | `--tls-disable-host-verification` | Default `false`. <!-- docs/cli/index.mdx:275 --> |
| `TEMPORAL_CODEC_ENDPOINT` | `--codec-endpoint` | Remote codec server endpoint. <!-- docs/cli/index.mdx:271 --> |
| `TEMPORAL_CODEC_AUTH` | `--codec-auth` | Authorization header for codec server. <!-- docs/cli/index.mdx:270 --> |
| `TEMPORAL_ENV` | `--env` | Active environment name (stored environment to load from). <!-- docs/cli/env.mdx:130 --> |

HTTP(S) proxy support uses stock gRPC vars: `HTTPS_PROXY` and `NO_PROXY`. <!-- docs/cli/index.mdx:303-322 -->

---

## 8. Precedence rules

Values can be supplied three ways (highest precedence first): <!-- docs/cli/env.mdx:117-149 -->

1. **CLI flags** (`--address`, `--namespace`, `--api-key`, etc.)
2. **`TEMPORAL_*` environment variables** in the shell.
3. **Stored environment** loaded with `--env <name>` (env var `TEMPORAL_ENV`), stored in `$HOME/.config/temporalio/temporal.yaml`. <!-- docs/cli/env.mdx:82 -->

---

## 9. Stored environments (`temporal env`)

`temporal env` stores named environments in `$HOME/.config/temporalio/temporal.yaml`. The flag that selects an environment is `--env <name>` (env var `TEMPORAL_ENV`), default `default`. <!-- docs/cli/env.mdx:82 --> <!-- docs/cli/env.mdx:130 -->

### Create and populate

```bash
# Dotted shorthand, positional key.value
temporal env set prod.address "production.f45a2.tmprl.cloud:7233"
temporal env set prod.namespace "production.f45a2"
temporal env set prod.tls-cert-path /temporal/certs/prod.pem
temporal env set prod.tls-key-path /temporal/certs/prod.key

# Explicit flags
temporal env set --env prod --key api-key --value "<key>"
```
<!-- docs/cli/env.mdx:87-95 -->

### Inspect, use, and clean up

```bash
# List environments
temporal env list

# Show one environment
temporal env get --env prod

# Show a single property
temporal env get --env prod --key tls-cert-path

# Run a command using that environment
temporal workflow list --env prod

# Delete a single key
temporal env delete --env prod --key tls-key-path

# Delete the whole environment
temporal env delete --env prod
```
<!-- docs/cli/env.mdx:27-78 -->

Keys accepted by `temporal env set` are flag names without the `--` prefix (`address`, `namespace`, `api-key`, `tls-cert-path`, `codec-endpoint`, `codec-auth`, etc.). <!-- docs/cli/env.mdx:100-104 -->

---

## 10. Smoke-test procedure

One command confirms address + namespace + TLS + auth are all coherent:

```bash
temporal workflow list --env <name> --limit 1
```

If it returns a result (even an empty list), the connection works. <!-- docs/cli/workflow.mdx:280-286 -->

---

## 11. Operational gotchas

1. **`temporal operator namespace` (self-hosted) vs `tcld namespace` (Cloud) are not interchangeable.** `operator namespace create` against a Cloud frontend fails because Cloud namespace creation is not exposed there. `tcld namespace create` against a self-hosted cluster has nowhere to land. <!-- docs/cli/operator.mdx:155-168 --> <!-- docs/cloud/tcld/namespace.mdx:21-45 -->

2. **`tctl` is deprecated -- use `temporal` or `tcld`.** The `tctl` reference carries an explicit deprecation admonition. Older docs and scripts still reference it; use the current CLIs for new work. <!-- docs/cli/index.mdx:27-32 -->

3. **`temporal env` stored environments are per-binary (not shared with `tcld` or SDKs).** `tcld` uses its own browser-OAuth token or `TEMPORAL_API_KEY`; it does not read `$HOME/.config/temporalio/temporal.yaml`. <!-- docs/cli/env.mdx:82 --> <!-- docs/cloud/tcld/index.mdx:35-42 -->

4. **The flag is `--env`, not `--profile`.** `--profile` is an unrelated, experimental TOML-config flag. Scripts that use `--profile` to select a stored environment pick up nothing and silently fall back to `default`. <!-- docs/cli/env.mdx:130 --> <!-- docs/cli/env.mdx:139 -->

5. **`tcld` has no `--output` flag (it always emits JSON).** `tcld` has exactly one global modifier: `--auto_confirm`. Passing `--output json` to `tcld` produces an "unknown flag" error. <!-- docs/cloud/tcld/index.mdx:35-42 -->

6. **`--output` enum values may disagree between doc pages -- trust per-command help.** `cmd-options.mdx` lists `table, json, card` but every per-command Global Flags table lists `text, json, jsonl, none`. The per-command tables match the shipping CLI. <!-- docs/cli/cmd-options.mdx:449-450 --> <!-- docs/cli/env.mdx:138 -->

7. **`tcld namespace export`: S3 config in `tcld` docs, GCS in separate cloud doc.** The `s3` sink group is in `docs/cloud/tcld/namespace.mdx`; the `gcs` sink group is in `docs/cloud/gcp-export-gcs.mdx`. <!-- docs/cloud/tcld/namespace.mdx:480-749 -->

8. **`tcld` reads `TEMPORAL_API_KEY`, not `TEMPORAL_CLOUD_API_KEY`.** `TEMPORAL_CLOUD_API_KEY` is a Terraform-provider variable; setting it for `tcld` has no effect. <!-- docs/cloud/tcld/index.mdx:35-42 -->

9. **mTLS env-var names have no `_PATH` suffix.** The documented variables are `TEMPORAL_TLS_CA`, `TEMPORAL_TLS_CERT`, `TEMPORAL_TLS_KEY`. Variants like `TEMPORAL_TLS_CERT_PATH` do not exist, even though the flags are `--tls-cert-path`, `--tls-key-path`, `--tls-ca-path`. <!-- docs/cli/index.mdx:273-276 -->

10. **Always verify subcommand flags against `--help` (docs can lag the actual CLI).** Both CLIs add and rename flags between releases. Run `temporal <subcommand> --help` or `tcld <subcommand> --help` for ground truth. <!-- docs/cli/env.mdx:112-150 -->
