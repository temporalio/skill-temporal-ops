# Cloud Terraform Provider

The Terraform Temporal Cloud provider allows you to use Terraform to manage resources for Temporal Cloud. It uses the Cloud Ops API. <!-- docs/cloud/terraform-provider.mdx:17-18 -->

Once a resource is managed by Terraform, you should only use Terraform to manage that resource. <!-- docs/cloud/terraform-provider.mdx:23 -->

---

## Prerequisites

- Terraform CLI <!-- docs/cloud/terraform-provider.mdx:45 -->
- An API Key for authentication <!-- docs/cloud/terraform-provider.mdx:46 -->

---

## Setup

Set the `TEMPORAL_CLOUD_API_KEY` environment variable: <!-- docs/cloud/terraform-provider.mdx:71 -->

```bash
export TEMPORAL_CLOUD_API_KEY=<your-secret-key>
```
<!-- docs/cloud/terraform-provider.mdx:70-71 -->

Or pass it directly in the provider block:

```hcl
provider "temporalcloud" { api_key = "my-temporalcloud-api-key" }
```
<!-- docs/cloud/terraform-provider.mdx:101 -->

Required provider configuration:

```hcl
terraform {
  required_providers {
    temporalcloud = {
      source = "temporalio/temporalcloud"
    }
  }
}

provider "temporalcloud" {

}
```
<!-- docs/cloud/terraform-provider.mdx:119-131 -->

---

## Namespace management

Resource: `temporalcloud_namespace` <!-- docs/cloud/terraform-provider.mdx:132 -->

Required identity: Account Owner, Global Admin, or Developer Account Role. <!-- docs/cloud/terraform-provider.mdx:110-111 -->

### Create

```hcl
resource "temporalcloud_namespace" "namespace" {
  name               = "terraform"
  regions            = ["aws-us-east-1"]
  accepted_client_ca = base64encode(file("ca.pem"))
  retention_days     = 14
}
```
<!-- docs/cloud/terraform-provider.mdx:132-137 -->

Key fields: `name`, `regions`, `accepted_client_ca`, `retention_days`. <!-- docs/cloud/terraform-provider.mdx:132-137 -->

For API Key auth, use `api_key_auth = true` instead of `accepted_client_ca`. <!-- docs/cloud/terraform-provider.mdx:314 -->

### Update

Terraform automatically recognizes changes in `.tf` files and applies them. For example, changing `retention_days` triggers an update. <!-- docs/cloud/terraform-provider.mdx:195-198 -->

### Delete

Remove the `temporalcloud_namespace` resource and all dependent resource configurations from your Terraform files and run `terraform apply`. <!-- docs/cloud/terraform-provider.mdx:239-241 -->

Use the `prevent_destroy` argument to prevent accidental deletion. <!-- docs/cloud/terraform-provider.mdx:250-253 -->

### Import

```bash
terraform import temporalcloud_namespace.terraform namespaceid.acctid
```
<!-- docs/cloud/terraform-provider.mdx:271 -->

The Namespace ID is in the format `namespaceid.acctid`, available at the top of the Namespace page in the Cloud UI. <!-- docs/cloud/terraform-provider.mdx:268 -->

---

## Nexus Endpoint management

Resource: `temporalcloud_nexus_endpoint` <!-- docs/cloud/terraform-provider.mdx:344 -->

Required identity: Developer role (or higher) and Namespace Admin permission on the Endpoint's target Namespace. <!-- docs/cloud/terraform-provider.mdx:288-289 -->

### Create

```hcl
resource "temporalcloud_nexus_endpoint" "nexus_endpoint" {
  name        = "terraform-nexus-endpoint"
  description = "my-service"
  worker_target = {
    namespace_id = temporalcloud_namespace.target_namespace.id
    task_queue   = "terraform-task-queue"
  }
  allowed_caller_namespaces = [
    temporalcloud_namespace.caller_namespace.id,
  ]
}
```
<!-- docs/cloud/terraform-provider.mdx:344-364 (simplified) -->

Key fields: `name`, `description`, `worker_target` (namespace_id, task_queue), `allowed_caller_namespaces`. <!-- docs/cloud/terraform-provider.mdx:344-364 -->

### Import

```bash
terraform import temporalcloud_nexus_endpoint <your-nexus-endpoint-ID>
```
<!-- docs/cloud/terraform-provider.mdx:515 -->

---

## User management

Resource: `temporalcloud_user` <!-- docs/cloud/terraform-provider.mdx:591 -->

### Limitations

- Terraform cannot create, update, or delete the Account Owner role. You can import an Account Owner, but not manage the role itself. <!-- docs/cloud/terraform-provider.mdx:550-551 -->
- Namespace access must be managed from the User resource, not from the Namespace resource. This also applies to Service Accounts. <!-- docs/cloud/terraform-provider.mdx:553-554 -->
- Account Owners and Global Admins automatically gain access to all Namespaces; you cannot specify Namespace access for these roles. <!-- docs/cloud/terraform-provider.mdx:555-556 -->
- Manage a specific user in one and only one `.tf` file to avoid overwriting permissions. <!-- docs/cloud/terraform-provider.mdx:557-558 -->
- To import a user, you need the User ID (currently not available in the Cloud UI). Fetch it using `tcld user list`. <!-- docs/cloud/terraform-provider.mdx:559-560 -->

### Create

```hcl
resource "temporalcloud_user" "global_admin" {
  email          = "admin@example.com"
  account_access = "Admin"
}

resource "temporalcloud_user" "namespace_admin" {
  email          = "developer@example.com"
  account_access = "Developer"

  namespace_accesses = [{
    namespace_id = temporalcloud_namespace.namespace.id
    permission   = "Write"
  }]
}
```
<!-- docs/cloud/terraform-provider.mdx:591-605 -->

### Import

```bash
terraform import temporalcloud_user.user 72360058153949edb2f1d47019c1e85f
```
<!-- docs/cloud/terraform-provider.mdx:698 -->

---

## Service Account management

Resource: `temporalcloud_service_account` <!-- docs/cloud/terraform-provider.mdx:751 -->

The process is similar to User management. Service Accounts use a `name` instead of `email`. <!-- docs/cloud/terraform-provider.mdx:708-710 -->

Same limitations as User management apply. <!-- docs/cloud/terraform-provider.mdx:712 -->

---

## API Key management

Resource: `temporalcloud_apikey` <!-- docs/cloud/terraform-provider.mdx:756 -->

### Create

```hcl
resource "temporalcloud_apikey" "global_apikey" {
  display_name = "admin"
  owner_type   = "service-account"
  owner_id     = temporalcloud_service_account.global_service_account.id
  expiry_time  = "2024-11-01T00:00:00Z"
  disabled     = false
}
```
<!-- docs/cloud/terraform-provider.mdx:756-762 -->

To access the API Key token, create an output:

```hcl
output "apikey_token" {
  value     = temporalcloud_apikey.global_apikey.token
  sensitive = true
}
```
<!-- docs/cloud/terraform-provider.mdx:773-776 -->

Retrieve the token:

```bash
terraform output -json apikey_token
```
<!-- docs/cloud/terraform-provider.mdx:813 -->

### Update

You can only edit an API Key's name or description field. Updating does not generate a new secure token. <!-- docs/cloud/terraform-provider.mdx:831-832 -->

### Import

API keys **cannot** be imported into Terraform. Once created, the API Key secret is not stored and cannot be retrieved. Create a new API Key using Terraform directly instead. <!-- docs/cloud/terraform-provider.mdx:842-845 -->

---

## Data sources

The provider supports two data sources: <!-- docs/cloud/terraform-provider.mdx:848-849 -->

### Regions

```hcl
data "temporalcloud_regions" "regions" {}

output "regions" {
  value = data.temporalcloud_regions.regions.regions
}
```
<!-- docs/cloud/terraform-provider.mdx:861-866 -->

### Namespaces

The `temporalcloud_namespaces` data source provides access to available Namespaces in the account. <!-- docs/cloud/terraform-provider.mdx:848-849 -->

---

## Resources

- Terraform Registry: [registry.terraform.io/providers/temporalio/temporalcloud/latest](https://registry.terraform.io/providers/temporalio/temporalcloud/latest) <!-- docs/cloud/terraform-provider.mdx:29 -->
- GitHub repository: [github.com/temporalio/terraform-provider-temporalcloud](https://github.com/temporalio/terraform-provider-temporalcloud/tree/main) <!-- docs/cloud/terraform-provider.mdx:33 -->
