# Cloud Billing

Temporal Cloud provides billing and costs information for your account. Use this information to assess spending patterns, inspect your credit ledger, check invoice histories, update payment details, and manage your current plan. <!-- docs/cloud/billing-and-usage/index.mdx:26-28 -->

---

## Tools for measuring usage and billing

| Tool | What it provides | Who can view |
|---|---|---|
| **Billing Center** | Summary invoices, credits, plan management, account deletion | Account Owners, Finance Admin <!-- docs/cloud/billing-and-usage/index.mdx:38-39 --> |
| **Billing API** | Namespace-level cost attribution down to hourly granularity, enriched with Tags and Projects; FOCUS-friendly CSV format | Account Owners, Finance Admin <!-- docs/cloud/billing-and-usage/index.mdx:41-42 --> |
| **Usage Dashboards** | Aggregate Actions on a Namespace level with Action categories | Account Owners, Finance Admin, Global Admin (account level); Namespace access holders (namespace level) <!-- docs/cloud/billing-and-usage/index.mdx:45-46 --> |
| **Actions in Event History** | Highlights Actions in a given Event History via the Cloud UI (some Actions are not measured in Workflow histories) | Account Owners, Global Admin, Namespace Admin, Developers, Read-Only <!-- docs/cloud/billing-and-usage/index.mdx:48-49 --> |
| **Actions Metrics** | High-cardinality billable action metric with labels for Category, Action Type, Workflow Type, Namespace (minute granularity) | Metrics Read-Only service account role <!-- docs/cloud/billing-and-usage/index.mdx:51-52 --> |

---

## Billing Center

Access: Account Owners and Finance Admins. <!-- docs/cloud/billing-and-usage/index.mdx:38-39 -->

### Current balance

Shows the balance for the current billing cycle and the date it was last updated. This balance adjusts with use. <!-- docs/cloud/billing-and-usage/billing.mdx:28-29 -->

Billing cycles normally begin on the first of the month (UTC). The minimum plan fee for your first month is prorated based on your sign-up date. <!-- docs/cloud/billing-and-usage/billing.mdx:33-35 -->

### Recent bill

Displays the previous bill amount. If the account pays through Stripe, a **Pay Now** button appears. Auto-payment accounts do not need to manually pay. <!-- docs/cloud/billing-and-usage/billing.mdx:40-47 -->

### Invoices table

| Column | Description |
|---|---|
| Date (UTC) | Date range covered by the invoice <!-- docs/cloud/billing-and-usage/billing.mdx:58 --> |
| Type | Type of invoice (e.g., credit purchase, cloud usage) <!-- docs/cloud/billing-and-usage/billing.mdx:59 --> |
| Status | Current status (e.g., paid, pending) <!-- docs/cloud/billing-and-usage/billing.mdx:60 --> |
| Credit Granted | Total credits added to the account <!-- docs/cloud/billing-and-usage/billing.mdx:61 --> |
| Credit Purchase Amount | Amount paid for purchasing credits <!-- docs/cloud/billing-and-usage/billing.mdx:62 --> |
| Credit Usage | Credits used during the billing cycle <!-- docs/cloud/billing-and-usage/billing.mdx:63 --> |
| Subtotal | Total amount before adjustments <!-- docs/cloud/billing-and-usage/billing.mdx:64 --> |
| Balance Due | Amount to pay after applying credits <!-- docs/cloud/billing-and-usage/billing.mdx:65 --> |

Invoices prior to the current calendar month can be downloaded. The current billing period invoice is not finalized and cannot be downloaded. <!-- docs/cloud/billing-and-usage/billing.mdx:69-74 -->

### Credits table

| Column | Description |
|---|---|
| Effective At (UTC) | Date when the credit grant became effective <!-- docs/cloud/billing-and-usage/billing.mdx:81 --> |
| Type | Whether the transaction was a deduction, expiry, or grant <!-- docs/cloud/billing-and-usage/billing.mdx:82 --> |
| Amount | Credit amount granted, deducted, or expired <!-- docs/cloud/billing-and-usage/billing.mdx:83 --> |
| Credits Remaining | Remaining credit available <!-- docs/cloud/billing-and-usage/billing.mdx:84 --> |

### Cost by Namespace

Account Owners and Finance Admins can see a cost column on the Usage page, enabling per-Namespace cost monitoring. <!-- docs/cloud/billing-and-usage/billing.mdx:96-98 -->

> **Being replaced.** The [Billing API](#billing-api) will replace the Cost by Namespace UI. It provides the same information on a Namespace basis down to hourly granularity, enriched with Tags and Projects. <!-- docs/cloud/billing-and-usage/billing.mdx:90-92 -->

Namespace cost details are not available for "last 90 days" or "last 120 days". Cost breakdowns distribute the total usage cost to namespaces proportionally based on metered usage. The proration reflects your effective price, factoring in included Actions/Storage and tiered pricing rates in your plan. <!-- docs/cloud/billing-and-usage/billing.mdx:104-108 -->

### Plans

Account Owners and Finance Admins can view plan information, pricing details, entitlements, available plans, and Pay-as-You-Go pricing rates. On a standard agreement they can also upgrade and downgrade between available plans. <!-- docs/cloud/billing-and-usage/billing.mdx:113-119 -->

- Upgrades are processed immediately with pro-rated billing. Monthly entitlements reflect the full volume of the upgrade plan for that billing month. After an upgrade, a downgrade cannot be processed until the following billing period. <!-- docs/cloud/billing-and-usage/billing.mdx:123-125 -->
- Downgrades are processed immediately. Billing and entitlements are backdated to the beginning of the billing period. <!-- docs/cloud/billing-and-usage/billing.mdx:127 -->

### Account cancellation

- **Accounts managed by sales team:** Submit a support ticket. <!-- docs/cloud/billing-and-usage/billing.mdx:133-134 -->
- **Self-signup accounts:** Account owners can delete their accounts on the Billing page, under the **Plan** tab. Permanently deleted accounts immediately cease billing and are scheduled for full deletion within 72 hours. Account Data and Active Storage are permanently deleted. Retained Storage is deleted per its configured retention period. <!-- docs/cloud/billing-and-usage/billing.mdx:136-142 -->

---

## Usage Dashboards

Actions usage is tracked across an account in the usage dashboard and is visible to Account Owners, Finance Admin, and Global Admin. Per-namespace usage is visible on the Namespace pages to those with access. <!-- docs/cloud/billing-and-usage/actions-usage.mdx:28-30 -->

### Actions in Workflows

When viewing an Event History, events that represent a Billable Action are annotated with the number consumed by the event in the **Billable Actions** column. These Actions are summarized at the top of the workflow. <!-- docs/cloud/billing-and-usage/actions-usage.mdx:36-37 -->

This estimate is useful for projecting cost. Example: 20 Actions per run, 100 runs/day, 30 days = 60,000 Billable Actions per month. <!-- docs/cloud/billing-and-usage/actions-usage.mdx:49-52 -->

> **Treat the estimate as an estimate.** The Billable Action estimate is an **experimental feature** and only measures Billable Actions that exist within Workflow event histories. If billable events exist outside of event history, the actual Actions count could be higher. Workflows with the `TemporalNamespaceDivision` Search Attribute set may not have accurate estimates. <!-- docs/cloud/billing-and-usage/actions-usage.mdx:56-57, 67-69 -->

Excluded from the Billable Actions estimate: <!-- docs/cloud/billing-and-usage/actions-usage.mdx:57-65 -->

- Query
- Activity Heartbeats
- Rejected Update Workflow Executions
- Export
- Schedule
- Replicated Actions in Namespace replication

---

## Billing API

<!-- GA. Do not re-add a "Public Preview" label from the docs: billing.mdx:90 and billing-api.mdx:42 still say public preview, but that is stale upstream. -->

The Billing API is part of the Cloud Operations API. It provides Namespace-level cost attribution through on-demand billing reports in CSV format, for ingestion into FinOps tooling and cloud cost management platforms. <!-- docs/cloud/billing-and-usage/billing-api.mdx:19-22 -->

Reports contain: <!-- docs/cloud/billing-and-usage/billing-api.mdx:30-34 -->

- Accurate Namespace-level cost attribution
- Hourly, daily, and monthly granularities
- A FOCUS-friendly data format

### Report generation flow

Report generation is **asynchronous**. <!-- docs/cloud/billing-and-usage/billing-api.mdx:38 -->

1. Create a billing report using `CreateBillingReport`. The response includes a `billing_report_id` and `async_operation_id`. <!-- docs/cloud/billing-and-usage/billing-api.mdx:119 -->
2. Poll `GetBillingReport` using the `billing_report_id`. <!-- docs/cloud/billing-and-usage/billing-api.mdx:120 -->
3. When the report state becomes `BILLING_REPORT_STATE_GENERATED`, retrieve the download URL. <!-- docs/cloud/billing-and-usage/billing-api.mdx:121 -->
4. Download the report before the URL expires. <!-- docs/cloud/billing-and-usage/billing-api.mdx:122 -->

Key identifiers: <!-- docs/cloud/billing-and-usage/billing-api.mdx:124-129 -->

| Identifier | Purpose |
|---|---|
| `billing_report_id` | Identifies the billing report; used to retrieve metadata and download URLs |
| `async_operation_id` | Identifies the background operation responsible for generating the report |

The async operation follows the standard Cloud Operations async model. See [cloud-ops-api.md](cloud-ops-api.md). <!-- docs/cloud/billing-and-usage/billing-api.mdx:131 -->

### Allowed date ranges

Date ranges must use billing-month boundaries (MM/YYYY). Requests may include the current billing month. Finalized reports include usage up to `current_time` - 24 hours (rounded down to the granularity level). <!-- docs/cloud/billing-and-usage/billing-api.mdx:49-51 -->

Data range limits by granularity: <!-- docs/cloud/billing-and-usage/billing-api.mdx:42-45 -->

| Granularity | Available range |
|---|---|
| Hourly | Current billing month + previous billing month |
| Daily | Current billing month + previous two billing months |
| Monthly | Current billing month + previous eleven billing months |

### Rate limits and concurrency

Within a single account, only one billing report is generated at a time. Additional requests are accepted but queued. <!-- docs/cloud/billing-and-usage/billing-api.mdx:59-62 -->

Report generation time varies and is not guaranteed. Factors include the size of the requested date range and overall platform load. <!-- docs/cloud/billing-and-usage/billing-api.mdx:66 -->

### Best practices

- Provide an idempotency key (`async_operation_id`) when retrying requests. <!-- docs/cloud/billing-and-usage/billing-api.mdx:71 -->
- Poll `GetBillingReport` using exponential backoff. <!-- docs/cloud/billing-and-usage/billing-api.mdx:73 -->
- Download reports immediately after generation (URLs expire). <!-- docs/cloud/billing-and-usage/billing-api.mdx:75 -->
- Avoid frequent generation of large overlapping ranges in the current billing period. <!-- docs/cloud/billing-and-usage/billing-api.mdx:77 -->

### Report schema (27 columns)

Each row represents a charge record. <!-- docs/cloud/billing-and-usage/billing-api.mdx:83 -->

| Column Name | Description | Example |
|---|---|---|
| `BillingAccountID` | Temporal Cloud account ID | `a2dd6` <!-- docs/cloud/billing-and-usage/billing-api.mdx:87 --> |
| `BillingAccountName` | Temporal Cloud account name | `temporal` <!-- docs/cloud/billing-and-usage/billing-api.mdx:88 --> |
| `BillingCurrency` | The currency an account is billed in | `USD (cents)` <!-- docs/cloud/billing-and-usage/billing-api.mdx:89 --> |
| `BillingPeriodEnd` | Exclusive end bound of a billing period | `2024-02-01T00:00:00Z` <!-- docs/cloud/billing-and-usage/billing-api.mdx:90 --> |
| `BillingPeriodStart` | Inclusive start bound of a billing period | `2024-01-01T00:00:00Z` <!-- docs/cloud/billing-and-usage/billing-api.mdx:91 --> |
| `ChargeCategory` | Highest-level classification based on how it is billed | `Usage` <!-- docs/cloud/billing-and-usage/billing-api.mdx:92 --> |
| `ChargeDescription` | Self-contained summary of the charge's purpose | `Actions - Tier 1` <!-- docs/cloud/billing-and-usage/billing-api.mdx:93 --> |
| `ChargeFrequency` | How often a charge occurs | `Usage-Based` <!-- docs/cloud/billing-and-usage/billing-api.mdx:94 --> |
| `ChargePeriodEnd` | Time period end for the charge (correlates to data granularity) | `2025-10-01T01:00:00.000Z` <!-- docs/cloud/billing-and-usage/billing-api.mdx:95 --> |
| `ChargePeriodStart` | Time period start for the charge (correlates to data granularity) | `2025-10-01T00:00:00.000Z` <!-- docs/cloud/billing-and-usage/billing-api.mdx:96 --> |
| `ContractedCost` | Cost calculated by multiplying `ContractedUnitPrice` and `PricingQuantity` | `100.00` <!-- docs/cloud/billing-and-usage/billing-api.mdx:97 --> |
| `ContractedUnitPrice` | Agreed-upon unit price for a single pricing unit, inclusive of negotiated discounts | `10.00` <!-- docs/cloud/billing-and-usage/billing-api.mdx:98 --> |
| `InvoiceID` | ID of the invoice for this billing period | `in_XXXXXXXXXXXXXXXXXXXX` <!-- docs/cloud/billing-and-usage/billing-api.mdx:99 --> |
| `InvoiceIssuer` | Entity responsible for issuing payable invoices | `stripe` <!-- docs/cloud/billing-and-usage/billing-api.mdx:100 --> |
| `PricingQuantity` | Volume of a given SKU used or purchased | `10.00` <!-- docs/cloud/billing-and-usage/billing-api.mdx:101 --> |
| `PricingUnit` | Measurement unit for `PricingQuantity` | `1 Million Actions` <!-- docs/cloud/billing-and-usage/billing-api.mdx:102 --> |
| `Provider` | Provider of purchased resources or services | `Temporal Technologies` <!-- docs/cloud/billing-and-usage/billing-api.mdx:103 --> |
| `Publisher` | Publisher of purchased resources or services | `Temporal Technologies` <!-- docs/cloud/billing-and-usage/billing-api.mdx:104 --> |
| `ResourceID` | Namespace name + Temporal Cloud account ID | `production.a2dd6` <!-- docs/cloud/billing-and-usage/billing-api.mdx:105 --> |
| `ResourceName` | Namespace name + Temporal Cloud account ID | `production.a2dd6` <!-- docs/cloud/billing-and-usage/billing-api.mdx:106 --> |
| `ResourceType` | Type of resource the charge applies to | `Namespace` <!-- docs/cloud/billing-and-usage/billing-api.mdx:107 --> |
| `ServiceCategory` | Highest-level classification based on core function | `Temporal Cloud` <!-- docs/cloud/billing-and-usage/billing-api.mdx:108 --> |
| `ServiceName` | Offering that can be purchased from a provider | `Temporal Cloud` <!-- docs/cloud/billing-and-usage/billing-api.mdx:109 --> |
| `ServiceSubcategory` | Secondary classification based on core function | `Actions` <!-- docs/cloud/billing-and-usage/billing-api.mdx:110 --> |
| `SKUID` | Unique identifier for a specific SKU | `essentials-actions` <!-- docs/cloud/billing-and-usage/billing-api.mdx:111 --> |
| `SKUMeter` | Functionality being metered by a particular SKU | `Actions` <!-- docs/cloud/billing-and-usage/billing-api.mdx:112 --> |
| `Tags` | Provider and customer defined tags associated with resources | `{"$tmprl_project":["project-id"],"namespace-tag-key":["namespace-tag-value"]}` <!-- docs/cloud/billing-and-usage/billing-api.mdx:113 --> |

**Key schema notes:**

- `ResourceID` is `namespace_name.account_id` (e.g., `production.a2dd6`), not just the namespace name. <!-- docs/cloud/billing-and-usage/billing-api.mdx:105 -->
- `BillingCurrency` values are in cents (e.g., `USD (cents)`). <!-- docs/cloud/billing-and-usage/billing-api.mdx:89 -->
- The cost column is `ContractedCost`, not `Cost` or `TotalCost`. <!-- docs/cloud/billing-and-usage/billing-api.mdx:97 -->
