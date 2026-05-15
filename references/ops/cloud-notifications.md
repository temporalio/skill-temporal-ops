# Cloud Notifications

Temporal Cloud sends notifications about system status and important administrative events.

---

## Status page subscriptions

In the event of an incident, Temporal updates the [Temporal Cloud status page](https://status.temporal.io/). Users can subscribe to updates in their preferred mode (e.g. email, Slack, SMS, etc.) by visiting this page. <!-- docs/cloud/notifications.mdx:24-25 -->

---

## Administrative email notifications

Temporal Cloud sends emails to notify users of important administrative events. <!-- docs/cloud/notifications.mdx:29 -->

| Reason for email | Who receives email |
|---|---|
| Certificate Expiring in 15 days | Global Administrator, Namespace Administrator, Account Owner <!-- docs/cloud/notifications.mdx:33 --> |
| Certificate Expiring in 10 days | Global Administrator, Namespace Administrator, Account Owner <!-- docs/cloud/notifications.mdx:34 --> |
| Certificate Expiring in 5 days | Global Administrator, Namespace Administrator, Account Owner <!-- docs/cloud/notifications.mdx:35 --> |
| API Key Expiring in 30 days | Global Administrator, Account Owner, individual user (if API Key has an owner) <!-- docs/cloud/notifications.mdx:36 --> |
| API Key Expiring in 20 days | Global Administrator, Account Owner, individual user (if API Key has an owner) <!-- docs/cloud/notifications.mdx:37 --> |
| API Key Expiring in 10 days | Global Administrator, Account Owner, individual user (if API Key has an owner) <!-- docs/cloud/notifications.mdx:38 --> |
| Sign up credit expiring in 30 days | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:39 --> |
| Sign up credit expiring in 14 days | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:40 --> |
| Sign up credit expiring in 7 days | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:41 --> |
| Sign up credit expiring in 1 day | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:42 --> |
| Sign up credit is 50% consumed | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:43 --> |
| Sign up credit is 90% consumed | Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:44 --> |
| Account plan type changed | Global Administrator, Account Owner, Finance Administrator <!-- docs/cloud/notifications.mdx:45 --> |
| Namespace Failover Completed/Failed | Global Administrator, Namespace Administrator, Account Owner <!-- docs/cloud/notifications.mdx:46 --> |

---

## Quick reference: notification thresholds

| Resource | Notification schedule |
|---|---|
| Certificate expiry | 15, 10, 5 days before expiry <!-- docs/cloud/notifications.mdx:33-35 --> |
| API Key expiry | 30, 20, 10 days before expiry <!-- docs/cloud/notifications.mdx:36-38 --> |
| Sign up credit expiry | 30, 14, 7, 1 day(s) before expiry <!-- docs/cloud/notifications.mdx:39-42 --> |
| Sign up credit consumption | 50%, 90% consumed <!-- docs/cloud/notifications.mdx:43-44 --> |

---

## Email sender

To ensure you receive email notifications, configure your junk-email filters to permit email from `noreply@temporal.io`. <!-- docs/cloud/notifications.mdx:49 -->

---

## Providing feedback

To provide feedback on notifications or request changes, create a support ticket. <!-- docs/cloud/notifications.mdx:51 -->
