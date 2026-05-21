# Codec Server

An HTTP service you host that lets the Temporal CLI and Web UI display
decoded payloads without the Temporal Service ever seeing plaintext.
Works with Temporal Cloud and self-hosted deployments.

---

## 1. What a Codec Server is

A Codec Server is an HTTP server, operated by you, that shares your Payload Codec logic with your Workers. The SDK Worker encodes outgoing payloads; the Codec Server's `/decode` endpoint reverses that so humans reading the Web UI or CLI output see decoded values. <!-- docs/production-deployment/data-encryption.mdx:52-54 -->

The Codec Server is independent of the Temporal Service. Payloads on the Temporal Service (Cloud or self-hosted) remain encoded; decoding happens client-side on demand when the CLI or Web UI calls your Codec Server. <!-- docs/production-deployment/data-encryption.mdx:57 -->

"Payload Codec" is a Data Converter component that does byte-to-byte transformation (encryption, compression, schema rewriting) between the Payload Converter and the wire. <!-- docs/production-deployment/data-encryption.mdx:22-23 -->

---

## 2. HTTP contract

The Web UI and CLI issue `POST` requests to `/decode` (and optionally `/encode`) on your Codec Server. <!-- docs/production-deployment/data-encryption.mdx:74-77 -->

### Request headers

| Header | Value | Notes |
|--------|-------|-------|
| `Content-Type` | `application/json` | Required. <!-- docs/production-deployment/data-encryption.mdx:96 --> |
| `X-Namespace` | `<namespace>` (e.g. `myapp-dev.acctid123`) | Custom header; must be in the CORS allow-list for the UI. <!-- docs/production-deployment/data-encryption.mdx:98 --> |
| `Authorization` | `<credentials>` | Optional; present only when auth is enabled. <!-- docs/production-deployment/data-encryption.mdx:100 --> |

### Request body

The Temporal Payload envelope, with base64-encoded field values by default: <!-- docs/production-deployment/data-encryption.mdx:107-119 -->

```json
{
  "payloads": [{
    "metadata": {
      "encoding": "<base64EncodedEncodingHint>"
    },
    "data": "<encryptedPayloadData>"
  }]
}
```

The response uses the same envelope, with `data` transformed (decoded). <!-- docs/production-deployment/data-encryption.mdx:188-212 -->

---

## 3. Hosting pointers

Any HTTP server in any language works. Per-language reference implementations: <!-- docs/production-deployment/data-encryption.mdx:83-88 -->

- [Go](https://github.com/temporalio/samples-go/tree/main/codec-server)
- [Java](https://github.com/temporalio/sdk-java/tree/master/temporal-remote-data-encoder)
- [Python](https://github.com/temporalio/samples-python/blob/main/encryption/codec_server.py)
- [TypeScript](https://github.com/temporalio/samples-typescript/blob/main/encryption/src/codec-server.ts)
- [.NET](https://github.com/temporalio/samples-dotnet/blob/main/src/Encryption/CodecServer/Program.cs)

Share codec logic with your Workers' Payload Codec so encode/decode stay symmetric. <!-- docs/production-deployment/data-encryption.mdx:66 -->

Expect multiple requests per Workflow Execution from the UI. Provision capacity accordingly. <!-- docs/production-deployment/data-encryption.mdx:61 -->

Restrict access (VPN, auth, HTTPS). Authorization from the Web UI requires HTTPS. <!-- docs/production-deployment/data-encryption.mdx:62 -->

---

## 4. CLI wiring

Three global flags wire the `temporal` CLI to a Codec Server: <!-- docs/cli/env.mdx:122-124 -->

| Flag | Purpose |
|------|---------|
| `--codec-endpoint` | Remote Codec Server endpoint (base URL; the CLI appends `/decode`). <!-- docs/cli/env.mdx:123 --> |
| `--codec-auth` | Authorization header for Codec Server requests. <!-- docs/cli/env.mdx:122 --> |
| `--codec-header` | HTTP headers for requests to codec server. Format `KEY=VALUE`. Repeatable. <!-- docs/cli/env.mdx:124 --> |

Two matching environment variables: <!-- docs/cli/index.mdx:270-271 -->

| Env var | Maps to |
|---------|---------|
| `TEMPORAL_CODEC_ENDPOINT` | `--codec-endpoint` <!-- docs/cli/index.mdx:271 --> |
| `TEMPORAL_CODEC_AUTH` | `--codec-auth` <!-- docs/cli/index.mdx:270 --> |

There is no documented `TEMPORAL_CODEC_HEADER` env var. Custom headers are set via the `--codec-header` flag only. <!-- docs/cli/index.mdx:267-279 -->

### Per-invocation example

```bash
temporal --codec-endpoint "http://localhost:8888" \
    --namespace "yourNamespace" \
    workflow show \
        --workflow-id "yourWorkflow" \
        --run-id "<yourRunId>" \
        --output "table"
```
<!-- docs/production-deployment/data-encryption.mdx:320-321 -->

With auth:

```bash
temporal workflow show \
    --workflow-id converters_workflowID \
    --codec-endpoint 'http://localhost:8081/{namespace}' \
    --codec-auth 'auth-header'
```
<!-- docs/production-deployment/data-encryption.mdx:329-333 -->

### Storing codec settings in a stored environment

Codec flags are persistable via `temporal env set`: <!-- docs/production-deployment/data-encryption.mdx:312-314 -->

```bash
temporal env set --codec-endpoint "http://localhost:8888"
```

Explicit key/value form with a named environment:

```bash
temporal env set --env prod --key codec-endpoint --value "https://codec.internal.example"
temporal env set --env prod --key codec-auth     --value "Bearer <token>"
```
<!-- docs/cli/env.mdx:87-95 -->

Precedence: CLI flag > `TEMPORAL_*` env var > stored environment value. <!-- docs/cli/env.mdx:117-149 -->

---

## 5. Web UI wiring

Three distinct places to configure a Codec Server for the Web UI.

### Dev server -- `--ui-codec-endpoint`

`temporal server start-dev` accepts `--ui-codec-endpoint` (string, UI remote codec HTTP endpoint). <!-- docs/cli/server.mdx:81 -->

```bash
temporal server start-dev --ui-codec-endpoint http://localhost:8888
```

This is distinct from the CLI-side `--codec-endpoint`:

- `--codec-endpoint` tells the `temporal` CLI where to decode payloads it prints to stdout.
- `--ui-codec-endpoint` tells the dev server's embedded Web UI where the browser should decode payloads.

### Self-hosted production Web UI -- config file

For a self-hosted Temporal Service with a dedicated UI Server, the codec endpoint lives in the UI server config: <!-- docs/production-deployment/data-encryption.mdx:299-304 -->

```yaml
codec:
    endpoint: {{ default .Env.TEMPORAL_CODEC_ENDPOINT "{namespace}"}}
```

### Namespace-level (Cloud or self-hosted) -- Web UI or `tcld`

Cloud and self-hosted UIs expose a per-Namespace "Codec Server" section under Namespace settings where an admin sets an endpoint, optional "Pass access token", and optional "Include cross-origin credentials". <!-- docs/production-deployment/data-encryption.mdx:266-272 -->

From `tcld`: <!-- docs/cloud/tcld/namespace.mdx:1683-1688 -->

```bash
tcld namespace update-codec-server \
    --namespace <namespace_id> \
    --endpoint <https_url> \
    --pass-access-token true \
    --include-credentials true
```

| Flag | Alias | Required | Description |
|------|-------|----------|-------------|
| `--namespace`, `-n` | `-n` | Yes | Namespace hosted on Temporal Cloud. <!-- docs/cloud/tcld/namespace.mdx:1695-1696 --> |
| `--endpoint`, `-e` | `-e` | Yes | HTTPS endpoint to decode payloads. Must be a valid HTTPS URL. <!-- docs/cloud/tcld/namespace.mdx:1709-1713 --> |
| `--pass-access-token` | `--pat` | No | Pass a user access token with each request. Default `false`. <!-- docs/cloud/tcld/namespace.mdx:1726-1728 --> |
| `--include-credentials` | `--ic` | No | Include cross-origin credentials. Default `false`. <!-- docs/cloud/tcld/namespace.mdx:1741-1743 --> |

### Browser-level override

In both Cloud and self-hosted UIs, an individual user can override the Namespace-level endpoint via the "Configure Codec Server" control on the Workflows page. <!-- docs/production-deployment/data-encryption.mdx:277-298 -->

Two override modes in Cloud:

- "Use Namespace-level settings, where available. Otherwise, use my browser setting."
- "Use my browser setting and ignore Namespace-level setting."

---

## 6. Authorization

Auth is the Codec Server's responsibility. Temporal's UI and CLI forward credentials. HTTPS is required for the Web UI to perform authorized requests. <!-- docs/production-deployment/data-encryption.mdx:155 -->

**From the CLI:** `--codec-auth <value>` sets the `Authorization` header on every Codec Server request. <!-- docs/cli/cmd-options.mdx:173 --> Persist it via `temporal env set --key codec-auth --value "..."` or `TEMPORAL_CODEC_AUTH=`. Use `--codec-header KEY=VALUE` (repeatable) for additional custom headers. <!-- docs/cli/env.mdx:124 -->

**From Temporal Cloud Web UI:** enabling "Pass access token" attaches a JWT in an `Authorization` header. Verify the token against `https://login.tmprl.cloud/.well-known/jwks.json`. The token includes the requesting user's email for authorization decisions. <!-- docs/production-deployment/data-encryption.mdx:159-171 -->

**From a self-hosted Web UI:** auth must be explicitly configured in the Web UI server config. Once enabled, the UI forwards access tokens the same way. <!-- docs/production-deployment/data-encryption.mdx:173-178 -->

**Network-only auth pattern:** restricting ingress to the Codec Server (VPN, private network, localhost-only) is a legitimate pattern. The Web UI can reach a `localhost`-only Codec Server from the browser running alongside it. <!-- docs/production-deployment/data-encryption.mdx:148-149 -->

---

## 7. CORS

CORS is required for Web UI usage (the browser is the client). It is not required for CLI usage. <!-- docs/production-deployment/data-encryption.mdx:124-125 -->

Minimum response headers your Codec Server must set: <!-- docs/production-deployment/data-encryption.mdx:129-131 -->

- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods`
- `Access-Control-Allow-Headers`

Example for Temporal Cloud Web UI at `https://cloud.temporal.io`: <!-- docs/production-deployment/data-encryption.mdx:133-137 -->

- `Access-Control-Allow-Origin: https://cloud.temporal.io`
- `Access-Control-Allow-Methods: POST, GET, OPTIONS`
- `Access-Control-Allow-Headers: X-Namespace, Content-Type`

If authorization is enabled, add `Authorization` to `Access-Control-Allow-Headers`. <!-- docs/production-deployment/data-encryption.mdx:140 -->

If "Include cross-origin credentials" is enabled, your CORS response also needs to permit credentials. <!-- docs/production-deployment/data-encryption.mdx:124 -->

---

## 8. Common gotchas

1. **Browser reachability is not service reachability.** The Web UI calls the Codec Server from the browser, not from the Temporal Service. A `localhost:8888` endpoint works for a browser on the same machine but is unreachable from any other device viewing the UI. <!-- docs/production-deployment/data-encryption.mdx:260-263 -->

2. **Local Network Access prompts.** Chrome may block or prompt for Web UI access to a Codec Server on a private network. Allow the "Local network" site permission for the UI host to unblock. <!-- docs/production-deployment/data-encryption.mdx:260-263 -->

3. **HTTPS required for UI auth.** The Web UI will not attach access tokens unless the Codec Server is HTTPS. `tcld namespace update-codec-server` enforces HTTPS for the registered endpoint. Plain-HTTP Codec Servers are only useful in unauthenticated/local-dev scenarios. <!-- docs/production-deployment/data-encryption.mdx:155 --> <!-- docs/cloud/tcld/namespace.mdx:1713 -->

4. **`--codec-endpoint` (CLI) vs `--ui-codec-endpoint` (dev server).** The former points the CLI client; the latter configures the Web UI served by `temporal server start-dev`. They are set in different places and address different clients. <!-- docs/cli/server.mdx:81 --> <!-- docs/cli/env.mdx:123 -->

5. **`--codec-endpoint` (CLI) vs `--endpoint` (`tcld namespace update-codec-server`).** They configure the same HTTP server from opposite sides: the CLI flag tells `temporal` where to send decode requests; the `tcld` flag records the endpoint on the Cloud Namespace so the Cloud UI knows where to send them. <!-- docs/cli/env.mdx:123 --> <!-- docs/cloud/tcld/namespace.mdx:1709 -->

6. **`TEMPORAL_CODEC_HEADER` does not exist.** Custom codec-server headers are set via repeated `--codec-header KEY=VALUE` flags; there is no documented env-var form. <!-- docs/cli/index.mdx:267-279 -->

7. **Latency.** Every payload display request in the UI goes through your Codec Server. Expect additional latency on Workflow/Event pages. <!-- docs/production-deployment/data-encryption.mdx:64 -->
