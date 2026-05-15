# Cloud Ops API

The Cloud Ops API provides programmatic management of Temporal Cloud control plane resources, including Namespaces, Users, Service Accounts, API Keys, and others. <!-- docs/cloud/operation-api.mdx:21 --> The Temporal Cloud Terraform Provider, `tcld` CLI, and Web UI all use the Cloud Ops API. <!-- docs/cloud/operation-api.mdx:21 -->

**Stage:** Public Preview. <!-- docs/cloud/operation-api.mdx:17 -->

---

## Endpoints

| Interface | URL | Notes |
|---|---|---|
| HTTP | `saas-api.tmprl.cloud` | Same URL for both HTTP and gRPC <!-- docs/cloud/operation-api.mdx:25 --> |
| gRPC | `saas-api.tmprl.cloud:443` | Port 443 for gRPC connections <!-- docs/cloud/operation-api.mdx:135 --> |

- HTTP API docs: [saas-api.tmprl.cloud/docs/httpapi.html](https://saas-api.tmprl.cloud/docs/httpapi.html#description/introduction) <!-- docs/cloud/operation-api.mdx:21 -->
- gRPC API source: [github.com/temporalio/cloud-api](https://github.com/temporalio/cloud-api/tree/main) <!-- docs/cloud/operation-api.mdx:21 -->
- gRPC docs on Buf: [buf.build/temporalio/cloud-api](https://buf.build/temporalio/cloud-api/docs/main:temporal.api.cloud.cloudservice.v1#temporal.api.cloud.cloudservice.v1.CloudService) <!-- docs/cloud/operation-api.mdx:61 -->

The HTTP API supports the same operations as the gRPC API, but is usable via standard HTTP methods. It does **not** allow interaction with individual Workflows or Activities via HTTP. <!-- docs/cloud/operation-api.mdx:45-49 -->

---

## Prerequisites

- Temporal Cloud user account <!-- docs/cloud/operation-api.mdx:31 -->
- API Key for authentication. Many operations require Admin privileges. <!-- docs/cloud/operation-api.mdx:32, 138 -->

---

## API version header

Always include the `temporal-cloud-api-version` header in every request, specifying the API version identifier. The current API version can be found at [github.com/temporalio/cloud-api/blob/main/VERSION](https://github.com/temporalio/cloud-api/blob/main/VERSION). <!-- docs/cloud/operation-api.mdx:131-133 -->

---

## Go SDK

For Go developers, use the [Go SDK](https://github.com/temporalio/cloud-sdk-go) which provides pre-compiled Go bindings and a more idiomatic interface. <!-- docs/cloud/operation-api.mdx:54, 65 -->

Install:

```go
go get github.com/temporalio/cloud-sdk-go
```
<!-- docs/cloud/operation-api.mdx:71 -->

Import:

```go
import (
    "github.com/temporalio/cloud-sdk-go/client"
)
```
<!-- docs/cloud/operation-api.mdx:76-78 -->

Go samples: [github.com/temporalio/cloud-samples-go](https://github.com/temporalio/cloud-samples-go) <!-- docs/cloud/operation-api.mdx:65 -->

---

## Compiling protobuf (non-Go languages)

For languages other than Go, download the gRPC protobufs from the [Cloud Ops API repository](https://github.com/temporalio/cloud-api/tree/main/temporal/api/cloud) and compile them manually. <!-- docs/cloud/operation-api.mdx:87 -->

Example using Python:

```bash
git clone https://github.com/temporalio/cloud-api.git
cd cloud-api
python -m grpc_tools.protoc -I./ --python_out=./ --grpc_python_out=./ *.proto
```
<!-- docs/cloud/operation-api.mdx:94-107 -->

For operation specifics, refer to `cloudservice/v1/request_response.proto` for gRPC messages and `cloudservice/v1/service.proto` for gRPC services. <!-- docs/cloud/operation-api.mdx:142 -->

---

## Use cases

Common reasons to use the Cloud Ops API: <!-- docs/cloud/operation-api.mdx:36-42 -->

- Provision Namespaces per environment or tenant via pipelines.
- Bootstrap new projects by creating users, assigning roles, and creating Namespaces via custom scripts.
- Rotate service account keys on a schedule with a job.
- Audit and report access across orgs with scheduled HTTP requests.

---

## Rate limits

| Scope | Limit |
|---|---|
| Account-level total | 160 RPS <!-- docs/cloud/operation-api.mdx:152 --> |
| Per user | 40 RPS <!-- docs/cloud/operation-api.mdx:158 --> |
| Per service account | 80 RPS <!-- docs/cloud/operation-api.mdx:162 --> |
| Concurrent async operations | 10 <!-- docs/cloud/operation-api.mdx:166 --> |

Rate limits are enforced across all Temporal Cloud control plane operations (tcld, UI, Cloud Ops API). <!-- docs/cloud/operation-api.mdx:172 -->

Multiple clients used by the same identity (user or service account) share the same rate limit. <!-- docs/cloud/operation-api.mdx:173 -->

Authentication method (SSO, API keys) does not affect rate limiting. <!-- docs/cloud/operation-api.mdx:174 -->

### Requesting limit increases

If your use case requires higher rate limits, submit a support ticket. Provide your current usage patterns, the specific limits you need increased, and a description of your use case. <!-- docs/cloud/operation-api.mdx:179-183 -->

---

## Connection setup

- Connect using the gRPC URL: `saas-api.tmprl.cloud:443`. <!-- docs/cloud/operation-api.mdx:135 -->
- Establish a secure connection. See the example [client setup in Go](https://github.com/temporalio/cloud-samples-go/blob/main/client/temporal/client.go). <!-- docs/cloud/operation-api.mdx:140 -->
