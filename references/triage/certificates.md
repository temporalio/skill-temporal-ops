# Certificates

Diagnose TLS and x509 failures against Temporal Cloud or a self-hosted frontend. This file covers layer 3 of the [diagnostic ladder](diagnostic-ladder.md).

Prerequisite: before reading this file, rule out layers 1 and 2 (DNS, TCP). TLS errors can masquerade as connectivity errors when a middlebox drops packets mid-handshake, so confirm TCP reachability first via [connectivity.md](connectivity.md#connection-refused). The reverse also happens: when a TLS handshake completes but the cert is invalid, the Go client emits cert-shaped errors that are firmly in layer 3.

Out of scope here (link, don't absorb):
- DNS / TCP / endpoint / firewall → [connectivity.md](connectivity.md) (layers 1-2)
- gRPC `UNAUTHENTICATED` after a successful TLS handshake → [authentication.md](authentication.md) (layer 4)
- API-key auth semantics (API-key connections still ride TLS, so TLS-level issues in this file apply — but the API-key authorization check itself is not a TLS issue) → [authentication.md](authentication.md)
- gRPC `RESOURCE_EXHAUSTED` → [rate-limits.md](rate-limits.md)

## Table of Contents

- [Error-string origin cheat sheet](#error-string-origin-cheat-sheet)
- [Handshake failure](#handshake-failure)
- [Expired or not-yet-valid](#expired-or-not-yet-valid)
- [Unknown authority](#unknown-authority)
- [Hostname mismatch](#hostname-mismatch)
- [Server name override](#server-name-override)
- [Key does not match cert](#key-does-not-match-cert)
- [Accepted client CA set (mTLS Cloud)](#accepted-client-ca-set-mtls-cloud)
- [Certificate requirements (Cloud mTLS)](#certificate-requirements-cloud-mtls)
- [Rotation and expiry notifications](#rotation-and-expiry-notifications)
- [openssl recipes](#openssl-recipes)
- [TLS / cert error reference](#tls--cert-error-reference)

## Error-string origin cheat sheet

Strings in this file come from the Go standard library (client side) or the TLS peer (alert descriptions). Every quoted string below is tagged with its source so you can tell a Go-local complaint from a peer-originated alert.

- **`x509: ...`** — emitted by Go's `crypto/x509` when the client rejects a peer cert locally. Source: `src/crypto/x509/verify.go`. <!-- go: crypto/x509 -->
- **`tls: ...`** — emitted by Go's `crypto/tls` during the handshake (typically a local protocol-level error). Source: `src/crypto/tls/`. <!-- go: crypto/tls -->
- **`remote error: tls: <alert description>`** — a TLS alert received from the *peer*, formatted by Go. The alert description words (`handshake failure`, `bad certificate`, `unknown certificate authority`, `expired certificate`, `internal error`, etc.) come from `src/crypto/tls/alert.go`; the `remote error: tls: ` prefix is added by Go when it surfaces the peer's alert. <!-- go: crypto/tls/alert.go -->
- **`Failed reaching server: last connection error`** — surfaced by the Temporal Cloud connection path; the troubleshooting guide identifies an expired TLS certificate as a common root cause. <!-- docs/troubleshooting/last-connection-error.mdx:15 -->

If you have an error string that doesn't start with `x509:`, `tls:`, or `remote error: tls:`, it is probably not a TLS-layer error — re-check the layer above (connectivity.md) or below (authentication.md).

## Handshake failure

**Symptom shape:**
- Client side: `tls: handshake failure` <!-- go: crypto/tls --> or a gRPC `UNAVAILABLE` <!-- grpc: UNAVAILABLE --> whose wrapped cause starts with `tls:` or `remote error: tls:`.
- Peer-alerted: `remote error: tls: handshake failure` <!-- go: crypto/tls/alert.go --> (the word `handshake failure` is the TLS alert description emitted by the other side).

**What it means:** the two sides did not agree on protocol version, cipher, or client authentication. On its own, `handshake failure` is not specific — it is the generic TLS alert when the peer cannot continue. Use `openssl s_client` (see [openssl recipes](#openssl-recipes)) to get a more specific line such as `certificate has expired`, `unknown certificate authority`, or a hostname-mismatch error.

**First check:** reproduce the handshake from the same host that saw the failure:

```bash
openssl s_client -connect <host>:7233 -servername <host> </dev/null
# -connect host:port: who to connect to         <!-- openssl: s_client -connect -->
# -servername name:  set the SNI server name    <!-- openssl: s_client -servername -->
```

For mTLS, add `-cert` and `-key`:

```bash
openssl s_client -connect <host>:7233 \
  -servername <host> \
  -cert client.pem -key client.key \
  -showcerts </dev/null
# -cert file:       client cert, PEM assumed    <!-- openssl: s_client -cert -->
# -key file:        client private key          <!-- openssl: s_client -key -->
# -showcerts:       show all server certs       <!-- openssl: s_client -showcerts -->
```

(The same `openssl s_client -connect <endpoint> -showcerts -cert ... -key ...` form is used by the Temporal troubleshooting guide <!-- docs/troubleshooting/last-connection-error.mdx:48 -->.)

Interpret by looking at the last few lines of output:

| Output line | Interpretation | Where to go |
|---|---|---|
| `Verify return code: 0 (ok)` and certificate info printed | TLS succeeded | Not a TLS problem. Move to [authentication.md](authentication.md). |
| `Verify return code: 10 (certificate has expired)` or `x509: certificate has expired or is not yet valid` <!-- go: crypto/x509 --> on the client | Cert past validity on server or client chain | [Expired or not-yet-valid](#expired-or-not-yet-valid) |
| `Verify return code: 19 (self signed certificate in certificate chain)` or `Verify return code: 20/21` | Client doesn't trust the server chain | [Unknown authority](#unknown-authority) |
| `tlsv1 alert unknown ca` / `remote error: tls: unknown certificate authority` <!-- go: crypto/tls/alert.go --> | Server rejected the client CA | [Accepted client CA set](#accepted-client-ca-set-mtls-cloud) |
| `tlsv1 alert bad certificate` / `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go --> | Server rejected the client cert (cert-filter mismatch, malformed cert, or wrong cert presented) | [Accepted client CA set](#accepted-client-ca-set-mtls-cloud) and [certificate filters in `docs/cloud/certificates.mdx`](/cloud/certificates#manage-certificate-filters) <!-- docs/cloud/get-started/certificates.mdx:402 --> |
| Handshake opens TCP but closes with no TLS alert bytes | Middlebox dropping / TLS-inspecting proxy | Back off to [connectivity.md → firewall and proxy](connectivity.md#firewall-and-proxy) |

## Expired or not-yet-valid

**Symptom shapes:**
- Go client: `x509: certificate has expired or is not yet valid: ` followed by detail <!-- go: crypto/x509 -->
- Peer alert: `remote error: tls: expired certificate` <!-- go: crypto/tls/alert.go -->
- Temporal Cloud: `Failed reaching server: last connection error` — the troubleshooting guide names an expired TLS certificate as a common root cause. <!-- docs/troubleshooting/last-connection-error.mdx:15 -->
- Workers that were fine yesterday stopped connecting overnight with no deploy.

**What to check first — exact expiry times of each cert in play:**

```bash
# Local cert file
openssl x509 -enddate -noout -in client.pem
# -enddate: print notAfter field    <!-- openssl: x509 -enddate -->
# -noout:   no encoded output       <!-- openssl: x509 -noout -->
# -in file: input file              <!-- openssl: x509 -in -->

# Both notBefore and notAfter
openssl x509 -dates -noout -in client.pem
# -dates: Both Before and After dates <!-- openssl: x509 -dates -->

# Server cert fetched from a live endpoint
openssl s_client -connect <host>:7233 -servername <host> </dev/null 2>/dev/null \
  | openssl x509 -enddate -noout
```

For the accepted-client-ca set on a Cloud Namespace, the troubleshooting guide lists expiry via `tcld namespace accepted-client-ca list` with `jq`:

```bash
tcld namespace accepted-client-ca list \
  --namespace <namespace_id>.<account_id> \
  | jq -r '.[0].notAfter'
# tcld namespace accepted-client-ca list: <!-- docs/cloud/tcld/namespace.mdx:869 -->
# --namespace (-n) modifier:             <!-- docs/cloud/tcld/namespace.mdx:878 -->
# Recipe exactly as written in:          <!-- docs/troubleshooting/last-connection-error.mdx:35-38 -->
```

**Clock skew — the "not yet valid" variant:** the same Go error string covers both "expired" and "not yet valid" (`x509: certificate has expired or is not yet valid: `) <!-- go: crypto/x509 -->. If `date -u` on the client disagrees with a time authority by more than the cert's overlap window (common in containers with no NTP, on appliances with a dead RTC battery, or right after a host boot), a cert that is in fact valid will still fail verification. Verify with `date -u` before regenerating anything.

**Classification:**

- **End-entity (leaf) cert expired, CA still valid:** regenerate the leaf against the existing CA. See [openssl recipes → issue a new leaf with tcld](#issue-a-new-leaf-with-tcld).
- **CA cert expired (so all leaves under it fail):** regenerate CA + leaf, upload the new CA *alongside* the existing one before removing the old one, then distribute new leaves and restart clients. See [Rotation and expiry notifications](#rotation-and-expiry-notifications). The Cloud docs describe this as a "rollover process" that "enables your Namespace to serve both CA certificates for a period of time until traffic to your old CA certificate ceases." <!-- docs/cloud/get-started/certificates.mdx:340-342 -->
- **Temporal Cloud server-side cert expired:** you don't manage the server side on Cloud. Open a support ticket per `docs/troubleshooting/last-connection-error.mdx`. <!-- docs/troubleshooting/last-connection-error.mdx:102 -->
- **Self-hosted server cert expired:** rotate the frontend's server TLS cert on your deployment. No tcld involvement.

:::caution
An expired root CA certificate invalidates all downstream certificates, per `docs/cloud/get-started/certificates.mdx:31`. <!-- docs/cloud/get-started/certificates.mdx:31 -->
:::

## Unknown authority

**Symptom shapes:**
- Go client: `x509: certificate signed by unknown authority` <!-- go: crypto/x509 -->
- Go client (verify chain): `x509: no valid chains built` <!-- go: crypto/x509 --> or `x509: failed to load system roots and no roots provided` <!-- go: crypto/x509 -->
- Peer alert: `remote error: tls: unknown certificate authority` <!-- go: crypto/tls/alert.go -->
- Peer alert: `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go -->

**Two directions this error travels — check which side is complaining before you act:**

1. **Client does not trust server (client-side `x509: ...`).** The client cannot validate the server certificate against any root it knows about. One thing to check is whether the client's trust store is populated — minimal container images often ship without system CAs — and whether the client is being pointed at a non-Temporal endpoint (TLS-inspecting proxy). The Go client does not ship its own root bundle; it relies on the host's trust store or an explicit `--tls-ca-path` <!-- docs/cli/cmd-options.mdx:653 --> / `TEMPORAL_TLS_CA` <!-- docs/cli/index.mdx:273 -->.
2. **Server does not trust client (peer-alert `remote error: tls: unknown certificate authority`).** For mTLS, the Namespace's accepted-client-ca set does not contain the CA that signed the client cert. Fix in [Accepted client CA set](#accepted-client-ca-set-mtls-cloud).

**Verify locally that the client cert chains to the CA you think it chains to:**

```bash
openssl verify -CAfile ca.pem client.pem
# -CAfile file: Certificate Authority file   <!-- openssl: verify -CAfile -->
```

If intermediates exist, supply them via `-untrusted`:

```bash
openssl verify -CAfile root-ca.pem -untrusted intermediate.pem client.pem
# -untrusted file: untrusted certificates file <!-- openssl: verify -untrusted -->
```

## Hostname mismatch

**Symptom shapes:**
- `x509: certificate is valid for <SAN list>, not <requested host>` <!-- go: crypto/x509 -->
- `x509: certificate is not valid for any names, but wanted to match <host>` <!-- go: crypto/x509 -->
- `x509: cannot validate certificate for <host>` <!-- go: crypto/x509 -->
- Go client may also emit: `x509: certificate relies on legacy Common Name field, use SANs instead` <!-- go: crypto/x509 --> when a server cert has no SAN and only a CN.

**What it means:** the hostname the client asked for does not match any Subject Alternative Name (or DNSName) on the server certificate the peer presented.

**First check:** which hostname is the client asking for?

```bash
# Inspect server cert SANs as actually served:
openssl s_client -connect <host>:7233 -servername <host> </dev/null 2>/dev/null \
  | openssl x509 -text -noout \
  | grep -A1 "Subject Alternative Name"
# -text: print certificate in text form  <!-- openssl: x509 -text -->
```

**Common cause on Temporal Cloud:** connecting by VPC-endpoint DNS name (PrivateLink) or GCP PSC IP, or by Regional Endpoint, without overriding the TLS server name. The TLS peer serves the Namespace's certificate, whose SAN is the Namespace Endpoint hostname — the VPC-endpoint hostname is not in it. Fix in the next section.

## Server name override

**Symptom:** TLS handshake fails with hostname-mismatch errors (see [Hostname mismatch](#hostname-mismatch) above) when connecting through a PrivateLink VPC endpoint, a GCP Private Service Connect IP, or a Cloud Regional Endpoint.

**What the Cloud docs say:**

- When a client uses a PrivateLink / PSC endpoint instead of the Namespace Endpoint DNS name, the docs instruct you to "Set TLS configuration to override the TLS server name (e.g., my-namespace.my-account.tmprl.cloud)." <!-- docs/cloud/connectivity/index.mdx:215-218 -->
- When a client uses a Regional Endpoint with mTLS, the docs say: "the Temporal Client must set the `server_name` property to `<namespace endpoint value>` in its request to the value of the Namespace endpoint. This tells the client to expect a different SNI header during the TLS handshake, since the request to the regional endpoint is redirected to the specific Namespace." <!-- docs/cloud/get-started/namespaces.mdx:338 -->
- Temporal recommends configuring private DNS instead, so the Namespace Endpoint hostname resolves to the VPC endpoint directly and no server-name override is needed. <!-- docs/cloud/connectivity/index.mdx:210-213 -->

**How to override on each client — verified forms only:**

`temporal` CLI flag (value: the Namespace Endpoint, e.g. `<namespace>.<account>.tmprl.cloud`):

```bash
temporal workflow list \
  --address vpce-0123456789abcdef-abc.us-east-1.vpce.amazonaws.com:7233 \
  --namespace <namespace>.<account> \
  --tls-cert-path client.pem --tls-key-path client.key \
  --tls-server-name <namespace>.<account>.tmprl.cloud
# --tls-server-name: Overrides the target TLS server name   <!-- docs/cli/cmd-options.mdx:677 -->
# --tls-cert-path:   Path to x509 certificate              <!-- docs/cli/cmd-options.mdx:661 -->
# --tls-key-path:    Path to private certificate key       <!-- docs/cli/cmd-options.mdx:673 -->
```

`temporal` CLI env var equivalent:

```bash
export TEMPORAL_ADDRESS=vpce-0123456789abcdef-abc.us-east-1.vpce.amazonaws.com:7233
export TEMPORAL_TLS_CERT=/path/to/cert.pem
export TEMPORAL_TLS_KEY=/path/to/cert.key
export TEMPORAL_TLS_SERVER_NAME=<namespace>.<account>.tmprl.cloud
# TEMPORAL_TLS_SERVER_NAME: Override for target TLS server name   <!-- docs/cli/index.mdx:277 -->
# TEMPORAL_TLS_CERT:        Path to x509 certificate              <!-- docs/cli/index.mdx:274 -->
# TEMPORAL_TLS_KEY:         Path to private certificate key       <!-- docs/cli/index.mdx:276 -->
# TEMPORAL_ADDRESS:         Host and port for the Temporal Frontend Service  <!-- docs/cli/index.mdx:269 -->

temporal workflow list --namespace <namespace>.<account>
```

The exact form above — `TEMPORAL_ADDRESS=vpce-...:7233` paired with `TEMPORAL_TLS_SERVER_NAME=my-namespace.my-account.tmprl.cloud` — is written out in the Cloud connectivity guide. <!-- docs/cloud/connectivity/index.mdx:234-238 -->

`grpcurl` (useful as a SDK-independent probe) — exactly the form the Cloud docs give:

```bash
grpcurl \
  -servername <namespace>.<account>.tmprl.cloud \
  -cert path/to/cert.pem \
  -key path/to/cert.key \
  vpce-0123456789abcdef-abc.us-east-1.vpce.amazonaws.com:7233 \
  temporal.api.workflowservice.v1.WorkflowService/GetSystemInfo
# Recipe as written in: <!-- docs/cloud/connectivity/index.mdx:243-251 -->
```

**For SDK clients:** the SDK concept is the same — set the TLS `ServerName` (Go / Java / .NET / Python / TypeScript) to the Namespace Endpoint hostname. The exact property name varies by SDK; refer to each SDK's client-connection doc linked from `docs/cloud/certificates#configure-clients-to-use-client-certificates` <!-- docs/cloud/get-started/certificates.mdx:484-489 --> and cross-check against `skill-temporal-developer`. This triage file deliberately does not spell SDK APIs out, to avoid drift. <!-- VERIFY: SDK-specific property names per-release; out of scope here. -->

## Key does not match cert

**Symptom shapes:**
- `tls: failed to find any PEM data in certificate input` <!-- go: crypto/tls --> (the file is empty, wrong format, or the wrong file)
- Go's `tls.LoadX509KeyPair` surfaces a keypair mismatch at client start. The precise string varies across Go versions, so don't pattern-match on it; the *shape* is an error at client dial saying the cert and key don't pair.

**Verify by comparing modulus hashes of the cert and the private key:**

```bash
# RSA
openssl x509 -modulus -noout -in client.pem | shasum -a 256
openssl rsa  -modulus -noout -in client.key | shasum -a 256
# -modulus: print the RSA key modulus    <!-- openssl: x509 -modulus --> <!-- openssl: rsa -modulus -->
# The two digests must be identical.
```

If the modulus hashes disagree, the cert and key file are from different keypairs. Find the key that was generated alongside this cert — commonly in the same directory — or regenerate both together.

(For ECDSA keys, `-modulus` does not apply; compare public keys with `openssl pkey -in client.key -pubout` against `openssl x509 -in client.pem -pubkey -noout`.)

## Accepted client CA set (mTLS Cloud)

On Temporal Cloud, an mTLS Namespace authenticates a client by validating the client cert against the CA set configured on the Namespace. The server-side error when the CA is not trusted is `remote error: tls: unknown certificate authority` <!-- go: crypto/tls/alert.go -->. When a CA *is* trusted but the end-entity cert fails other checks (certificate filter mismatch, malformed cert), the server sends `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go -->.

**List what the Namespace currently accepts:**

```bash
tcld namespace accepted-client-ca list \
  --namespace <namespace_id>
# Command:     <!-- docs/cloud/tcld/namespace.mdx:867 -->
# --namespace: <!-- docs/cloud/tcld/namespace.mdx:878 -->
```

**Add a new CA to the accepted set:**

```bash
tcld namespace accepted-client-ca add \
  --namespace <namespace_id> \
  --ca-certificate-file <path>
# Command:                 <!-- docs/cloud/tcld/namespace.mdx:778 -->
# --ca-certificate-file:   <!-- docs/cloud/tcld/namespace.mdx:850 -->
```

**Remove a CA (by fingerprint, safer than by PEM):**

```bash
tcld namespace accepted-client-ca remove \
  --namespace <namespace_id> \
  --ca-certificate-fingerprint <fingerprint>
# Command:                           <!-- docs/cloud/tcld/namespace.mdx:892 -->
# --ca-certificate-fingerprint (--fp): <!-- docs/cloud/tcld/namespace.mdx:985 -->
```

**Set the whole bundle at once (zero-downtime rollover):**

The Cloud docs describe a concat-old-plus-new-then-set pattern for rolling over CA certs without dropping traffic:

```bash
# 1. Create a file with old + new CA PEM blocks concatenated.
# 2. Run:
tcld namespace accepted-client-ca set \
  --ca-certificate-file <path>
# Command and rollover procedure: <!-- docs/cloud/tcld/namespace.mdx:1002-1039 -->
# Same procedure in:              <!-- docs/cloud/get-started/certificates.mdx:374-400 -->
# 3. Wait for traffic to old CA to drain.
# 4. Create a file with only the new CA and run the set command again.
```

**Verify locally before uploading** that the client cert chains to the CA you're about to upload:

```bash
openssl verify -CAfile new-ca.pem client.pem
# Expected: client.pem: OK
```

If this fails locally, it will fail at the peer too. Fix before touching the Namespace.

## Certificate requirements (Cloud mTLS)

The docs state hard requirements for any CA or leaf cert you upload to a Cloud Namespace. These are where subtle TLS rejections come from when a cert "looks fine" but the peer sends `remote error: tls: bad certificate` anyway. Quoting the requirements as listed at `docs/cloud/get-started/certificates.mdx:62-96`:

**CA certificates:**
- X.509v3. <!-- docs/cloud/get-started/certificates.mdx:69 -->
- Each cert in a bundle is either a root or issued by another cert in the bundle. <!-- docs/cloud/get-started/certificates.mdx:70 -->
- Each cert includes `CA: true`. <!-- docs/cloud/get-started/certificates.mdx:71 -->
- Cannot be a well-known CA (DigiCert, Let's Encrypt, etc.) unless the user also specifies certificate filters. <!-- docs/cloud/get-started/certificates.mdx:72-73 -->
- Signing algorithm must be RSA or ECDSA and must include SHA-256 or stronger. SHA-1 and MD5 cannot be used. <!-- docs/cloud/get-started/certificates.mdx:74-75 -->
- Cannot be generated with a passphrase. <!-- docs/cloud/get-started/certificates.mdx:76 -->
- Bundle ≤ 16 CA certs, ≤ 32 KB before base64 encoding. <!-- docs/cloud/get-started/certificates.mdx:80-81 -->
- In a full end-entity → root chain, each certificate must have a unique Distinguished Name (DN comparison is case-insensitive). <!-- docs/cloud/get-started/certificates.mdx:52-60 -->

**End-entity (leaf) certificates:**
- X.509v3. <!-- docs/cloud/get-started/certificates.mdx:92 -->
- Basic constraints must include `CA: false`. <!-- docs/cloud/get-started/certificates.mdx:93 -->
- Key usage must include Digital Signature. <!-- docs/cloud/get-started/certificates.mdx:94 -->
- Signing algorithm: RSA or ECDSA with SHA-256 or stronger. <!-- docs/cloud/get-started/certificates.mdx:95-96 -->

**Certificate Revocation Lists:** Temporal does not support or check CRLs; customers are expected to keep certificates up to date. <!-- docs/cloud/get-started/certificates.mdx:268-270 -->

**Algorithm choice when generating with tcld:** `tcld gen ca` defaults to ECDSA P-384; `--rsa-algorithm` (alias `--rsa`) switches to a 4096-bit RSA key pair. <!-- docs/cloud/tcld/generate-certificates.mdx:83-94 -->

**Duration caps when generating with tcld:** `tcld gen ca` has a maximum duration of 1 year (`-d 1y`). You must set an end-entity cert to expire before its root CA. <!-- docs/cloud/get-started/certificates.mdx:129-131 -->

## Rotation and expiry notifications

**Notifications.** Temporal Cloud sends email notifications before CA expiry. The notifications doc lists "Certificate Expiring in 15 days" as an admin notification sent to Global Administrators, Namespace Administrators, and Account Owners. <!-- docs/cloud/notifications.mdx:33 --> The certificates guide states: "Temporal Cloud begins sending notifications 15 days before expiration." <!-- docs/cloud/get-started/certificates.mdx:336 -->

**Rollover strategy.** The Cloud docs prescribe a zero-downtime rollover pattern: upload the new CA alongside the existing one, wait for traffic to shift to leaves signed by the new CA, then remove the old CA. The same shape applies whether you use the UI or `tcld namespace accepted-client-ca set`. <!-- docs/cloud/get-started/certificates.mdx:340-400 --><!-- docs/cloud/tcld/namespace.mdx:1011-1039 -->

**Rotation command sequence (issue with tcld, upload with tcld):**

```bash
# Issue a new CA cert (if rotating CA). Default is ECDSA P-384.
tcld generate-certificates certificate-authority-certificate \
  --organization <org> \
  --validity-period 1y \
  --ca-certificate-file new-ca.pem \
  --ca-key-file new-ca.key
# Command (alias: tcld gen ca):  <!-- docs/cloud/tcld/generate-certificates.mdx:24 -->
# --organization (--org):        <!-- docs/cloud/tcld/generate-certificates.mdx:34 -->
# --validity-period (-d):        <!-- docs/cloud/tcld/generate-certificates.mdx:46 -->
# --ca-certificate-file (--ca-cert): <!-- docs/cloud/tcld/generate-certificates.mdx:59 -->
# --ca-key-file (--ca-key):          <!-- docs/cloud/tcld/generate-certificates.mdx:71 -->

# Issue a new end-entity (leaf) cert against a CA
tcld generate-certificates end-entity-certificate \
  --organization <org> \
  --validity-period 364d \
  --ca-certificate-file new-ca.pem \
  --ca-key-file new-ca.key \
  --certificate-file client.pem \
  --key-file client.key
# Command (alias: tcld gen leaf):    <!-- docs/cloud/tcld/generate-certificates.mdx:96 -->
# --certificate-file (--cert):       <!-- docs/cloud/tcld/generate-certificates.mdx:165 -->
# --key-file (--key):                <!-- docs/cloud/tcld/generate-certificates.mdx:177 -->

# Upload concatenated old+new CA bundle to the Namespace, then (after drain) upload new-only bundle.
tcld namespace accepted-client-ca set \
  --namespace <namespace_id> \
  --ca-certificate-file <path-to-bundle>
# Procedure:                              <!-- docs/cloud/tcld/namespace.mdx:1011-1039 -->
# --namespace:                            <!-- docs/cloud/tcld/namespace.mdx:1043 -->
```

**When the CA itself is already expired** (the rollover window was missed), a new CA must be uploaded before any client can reconnect. This is the "cert expired at 3 a.m." shape; see [recipes.md](recipes.md) for the full step-by-step.

## openssl recipes

These are reference-card forms; each flag is cross-linked to the openssl usage output.

### Inspect a local cert

```bash
# Full text
openssl x509 -in cert.pem -noout -text
# -in / -noout / -text        <!-- openssl: x509 -in / -noout / -text -->

# Subject and issuer
openssl x509 -in cert.pem -noout -subject -issuer
# -subject / -issuer          <!-- openssl: x509 -subject / -issuer -->

# Validity dates
openssl x509 -in cert.pem -noout -dates
# -dates: both Before and After  <!-- openssl: x509 -dates -->

# Fingerprint (for tcld --ca-certificate-fingerprint)
openssl x509 -in cert.pem -noout -fingerprint
# -fingerprint                <!-- openssl: x509 -fingerprint -->
```

Note: LibreSSL (the `openssl` shipped with macOS as of LibreSSL 3.x) does not support the upstream OpenSSL `-ext <extname>` filter on `openssl x509`. To see SANs on LibreSSL, use `openssl x509 -text -noout` and read the `X509v3 Subject Alternative Name` section, or use grep as in [Hostname mismatch](#hostname-mismatch). <!-- VERIFY: behavior depends on local openssl build; upstream OpenSSL 1.1.1+ has -ext. -->

### Verify a chain

```bash
# Leaf + root
openssl verify -CAfile ca.pem client.pem

# Leaf + root + intermediate
openssl verify -CAfile root-ca.pem -untrusted intermediate.pem client.pem
# -CAfile / -untrusted        <!-- openssl: verify -CAfile / -untrusted -->
```

### Test a live endpoint

```bash
# Public-internet or mTLS Namespace, inspect the peer's cert chain
openssl s_client -connect <host>:7233 \
  -servername <host> \
  -showcerts </dev/null 2>/dev/null \
  | openssl x509 -text -noout

# Full mTLS handshake (what the Temporal troubleshooting guide uses)
openssl s_client -connect <namespace>.<account>.tmprl.cloud:7233 \
  -showcerts \
  -cert client.pem -key client.key \
  -tls1_2 </dev/null
# Recipe as written in the troubleshooting guide: <!-- docs/troubleshooting/last-connection-error.mdx:48 -->
# -tls1_2: force TLSv1.2 (the troubleshooting guide uses this) <!-- openssl: s_client -tls1_2 -->
```

Reading the output:
- Server cert block(s) appear between `-----BEGIN CERTIFICATE-----` markers under `Certificate chain`.
- The last line normally prints `Verify return code: 0 (ok)` on success; a non-zero code is the OpenSSL verify error, e.g. `10 (certificate has expired)`.
- If the server sent a TLS alert rather than certs, you will see `tlsv1 alert <description>`; the alert description is one of the words listed in `src/crypto/tls/alert.go` (`handshake failure`, `bad certificate`, `unknown certificate authority`, `expired certificate`, etc.). <!-- go: crypto/tls/alert.go -->

### Compare a keypair

```bash
openssl x509 -in cert.pem -modulus -noout | shasum -a 256
openssl rsa  -in key.pem  -modulus -noout | shasum -a 256
# Digests must match for the files to be from the same keypair.
```

### Self-signed cert workflow (self-hosted or temporary)

The troubleshooting guide suggests using `temporal namespace describe` with explicit TLS flags when working with self-signed certs:

```bash
temporal namespace describe \
  --namespace <namespace_id>.<account_id> \
  --address <namespace_grpc_endpoint> \
  --tls-cert-path <path-to-mTLS-pem-file> \
  --tls-key-path <path-to-mTLS-key-file>
# Recipe as written in: <!-- docs/troubleshooting/last-connection-error.mdx:53-61 -->
```

### Issue a new leaf with tcld

```bash
tcld gen leaf \
  --org <org> \
  -d 364d \
  --ca-cert ca.pem --ca-key ca.key \
  --cert client.pem --key client.key
# tcld gen leaf = tcld generate-certificates end-entity-certificate <!-- docs/cloud/tcld/generate-certificates.mdx:96-102 -->
# Aliases listed in:                                               <!-- docs/cloud/tcld/generate-certificates.mdx:102 -->
# Example as written in:                                           <!-- docs/cloud/get-started/certificates.mdx:146-147 -->
```

## TLS / cert error reference

Each row is tagged with the string's origin. When an error doesn't fit any row here, re-read [Error-string origin cheat sheet](#error-string-origin-cheat-sheet) to decide whether it is really a TLS-layer error at all.

| Error text (exact, as emitted) | Origin | One thing to check |
|---|---|---|
| `x509: certificate has expired or is not yet valid: <detail>` <!-- go: crypto/x509 --> | Go x509 | [Expired or not-yet-valid](#expired-or-not-yet-valid); also check `date -u` for clock skew |
| `x509: certificate signed by unknown authority` <!-- go: crypto/x509 --> | Go x509 | Client doesn't trust peer's root. [Unknown authority](#unknown-authority) |
| `x509: certificate is valid for <SAN>, not <host>` <!-- go: crypto/x509 --> | Go x509 | [Server name override](#server-name-override) |
| `x509: cannot validate certificate for <host>` <!-- go: crypto/x509 --> | Go x509 | [Hostname mismatch](#hostname-mismatch) |
| `x509: no valid chains built` <!-- go: crypto/x509 --> | Go x509 | Chain does not reach a trusted root; verify with `openssl verify -CAfile ...` |
| `x509: a root or intermediate certificate is not authorized to sign for this name: <detail>` <!-- go: crypto/x509 --> | Go x509 | Name-constraints extension rejects the leaf |
| `x509: certificate is not authorized to sign other certificates` <!-- go: crypto/x509 --> | Go x509 | Cert with `CA: false` is being used as an issuer |
| `x509: failed to load system roots and no roots provided` <!-- go: crypto/x509 --> | Go x509 | Minimal container missing `ca-certificates`; or `TEMPORAL_TLS_CA` / `--tls-ca-path` not set |
| `x509: certificate relies on legacy Common Name field, use SANs instead` <!-- go: crypto/x509 --> | Go x509 | Peer cert has no SAN, only CN; re-issue with SANs |
| `tls: handshake failure` <!-- go: crypto/tls --> | Go tls (local) | See [Handshake failure](#handshake-failure) and reproduce with `openssl s_client` |
| `remote error: tls: handshake failure` <!-- go: crypto/tls/alert.go --> | peer alert | Peer rejected the handshake; reproduce with `openssl s_client` for the alert description |
| `remote error: tls: bad certificate` <!-- go: crypto/tls/alert.go --> | peer alert | [Accepted client CA set](#accepted-client-ca-set-mtls-cloud); also check certificate filters <!-- docs/cloud/get-started/certificates.mdx:402 --> |
| `remote error: tls: unknown certificate authority` <!-- go: crypto/tls/alert.go --> | peer alert | [Accepted client CA set](#accepted-client-ca-set-mtls-cloud) |
| `remote error: tls: expired certificate` <!-- go: crypto/tls/alert.go --> | peer alert | [Expired or not-yet-valid](#expired-or-not-yet-valid) |
| `remote error: tls: internal error` <!-- go: crypto/tls/alert.go --> | peer alert | Peer-side failure, not a cert problem on this end; retry and, if persistent on Cloud, open a support ticket |
| `Failed reaching server: last connection error` <!-- docs/troubleshooting/last-connection-error.mdx:15 --> | Temporal client diagnostic | Often an expired TLS cert per the troubleshooting guide; run the expiry checks in [Expired or not-yet-valid](#expired-or-not-yet-valid) |

For TLS errors that only appear after the handshake (gRPC `UNAUTHENTICATED`, `PERMISSION_DENIED`), jump to [authentication.md](authentication.md). For `UNAVAILABLE` without a TLS-layer cause, back off to [connectivity.md](connectivity.md).
