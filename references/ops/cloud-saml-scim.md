# Cloud SAML and SCIM

SAML enables single sign-on (SSO) by allowing your identity provider to authenticate users into Temporal Cloud. SCIM automatically creates, updates, and removes users and groups in Temporal Cloud based on changes in your identity provider. <!-- docs/cloud/manage-access/index.mdx -->

---

## SAML SSO

SAML 2.0 integration allows you to authenticate users of your Temporal Cloud account using your organization's IdP. This enforces corporate identity policies such as multi-factor authentication (MFA) and password complexity. <!-- docs/cloud/manage-access/saml.mdx -->

SAML is included in the Business, Enterprise, and Mission Critical plans. <!-- docs/cloud/manage-access/saml.mdx -->

### Configuration overview

1. Locate your Temporal Cloud Account Id (5-6 characters after the period in your Namespace Id, e.g., `f45a2`).
2. Configure SAML with your IdP (Microsoft Entra ID or Okta).
3. Share connection information with Temporal and test the connection.

### Entity identifier format

```
urn:auth0:prod-tmprl:ACCOUNT_ID-saml
```

Example:

```
urn:auth0:prod-tmprl:f45a2-saml
```

### Callback URL format

```
https://login.tmprl.cloud/login/callback?connection=ACCOUNT_ID-saml
```

Example:

```
https://login.tmprl.cloud/login/callback?connection=f45a2-saml
```

### Sign on URL format (Entra ID only)

```
https://cloud.temporal.io/login/saml?connection=ACCOUNT_ID-saml
```

### Microsoft Entra ID configuration

1. Sign in to Microsoft Entra ID.
2. **Manage Microsoft Entra ID** > **View** > **Add > Enterprise application**.
3. **Create your own application** > name it (e.g., `temporal-cloud`) > select **Integrate any other application you don't find in the gallery**.
4. **Getting Started** > **Set up single sign on** > **SAML**.
5. In **Basic SAML Configuration**:
   - Set **Identifier (Entity ID)** to the entity identifier above.
   - Set **Reply URL (Assertion Consumer Service URL)** to the callback URL above.
   - Set **Sign on URL** to the sign on URL above.
6. In **Attributes & Claims**:
   - Set **Unique User Identifier (NameID)** to `user.userprincipalname`.
   - Set **NameID format** to `emailAddress`.
   - Ensure **Email** and **Name** are present under Additional claims.
7. Collect for Temporal: download **Certificate (Base64)** and copy **Login URL**.

### Okta configuration

1. Sign in to Okta Admin Console.
2. **Applications** > **Create App Integration** > **SAML 2.0** > **Next**.
3. Name the application (e.g., `temporal-cloud`).
4. In **Configure SAML**:
   - Set **Single sign on URL** to the callback URL above.
   - Set **Audience URI (SP Entity ID)** to the entity identifier above.
   - Set **Name ID format** to `EmailAddress`.
   - Set **Attribute Statements**: `email` and `name`.
5. In the **Feedback** section, select **Finish**.
6. On the application page > **Sign On** tab > **View SAML setup instructions**. Copy IdP settings and download the active certificate.

### Finish SAML configuration (customer ticket)

Create a support ticket with:

- The sign-in URL from your application
- The X.509 SAML sign-in certificate (PEM or Base64 are both acceptable on the ticket; ocld accepts `--x509-cert` / `--x509-cert-file`, or `--saml-metadata-file` for SAML metadata XML)
- One or more IdP domains to map to the SAML connection

The IdP domain is generally the same as your email domain. Multiple IdP domains can be provided.

After Temporal confirms configuration is complete, go to the Cloud login page, enter your email, choose **Enterprise identity**, then click **Continue**. Do **not** use **Continue with Google** or **Continue with Microsoft** for SAML SSO.

### Temporal-side SAML (Auth0 + ocld)

Ops configure SAML on the Temporal side via Auth0 and `ocld`. Typical flow:

1. **Create the Auth0 SAML connection** (connection name must be `ACCOUNT_ID-saml`):

   ```bash
   ct ocld <env> account create-saml-connection \
     -a <ACCOUNT_ID> \
     --sign-in-url '<idp-sso-url>' \
     --x509-cert '<base64-x509>' \
     --idp-domain '<example.com>'
   ```

   Alternatives to `--x509-cert` / `-c`: `--x509-cert-file` / `-cf` (path to cert) or `--saml-metadata-file` / `-smf` (SAML metadata XML). `--idp-domain` / `-d` is required and may be repeated for multiple domains.

2. **Enable SAML Organization** (`--enable-saml-organization`) on the account so domain-based enterprise login routes to that connection.
3. **Optional — SAML-only enforcement** (separate from enabling SAML):

   | Setting | Effect |
   |---------|--------|
   | SAML enabled only | Enterprise SAML works; email+password+MFA and social login (Google/Microsoft) remain available |
   | `EnableSAMLOnlyConnection` / `--enable-saml-only-connection` | **Only** SAML login is allowed — blocks email+password+MFA and social login |

   Enabling SAML alone does **not** block other auth methods. Set SAML-only explicitly when the customer requires IdP-only access.

4. **Delete** a connection when decommissioning:

   ```bash
   ct ocld <env> account delete-saml-connection -a <ACCOUNT_ID>
   ```

5. Support can also use the **configure-saml** UI for assisted setup when appropriate.

---

## SCIM user provisioning

SCIM lets you integrate your identity provider with Temporal Cloud to automate user provisioning and access. Changes in the IdP are reflected in Temporal Cloud: <!-- docs/cloud/manage-access/scim.mdx -->

- User creation / onboarding
- User deletion / offboarding
- User membership in groups

SCIM requires SAML. Pricing:

- **Business:** SCIM is a paid add-on (+$500/mo)
- **Enterprise / Mission Critical:** SCIM included

### Supported IdP vendors

<!-- docs/cloud/manage-access/scim.mdx -->
- Okta
- Microsoft Entra ID (Azure AD)
- Google Workspace
- OneLogin
- CyberArk
- JumpCloud
- PingFederate
- Any SCIM 2.0-compliant provider

### Prerequisites

1. Configure SAML SSO first.
2. Identify your organization's IdP administrator and specify their contact details in the support ticket (invite that admin for Directory Sync only — they do not need broad Cloud admin rights for SCIM setup).
3. Submit a support ticket to enable SCIM.

### Cloud-managed vs SCIM-managed lifecycle

| Subject | Who manages create/delete | Who manages group membership | Who assigns Temporal roles |
|---------|---------------------------|------------------------------|----------------------------|
| **Cloud-managed users** | Cloud UI/API invite and delete, until `DisableUserLifecycleManagement` | Cloud (or SCIM if later synced into groups) | Cloud UI / tcld / Terraform |
| **SCIM-managed users** | IdP only — **cannot** delete via Cloud UI/API; offboard in the IdP | IdP only | Roles still assigned in Cloud (directly or via synced groups) |
| **SCIM-synced groups** | IdP creates/updates/deletes groups | IdP only | Assign roles in Cloud **after** sync (UI / tcld / Terraform). IdP does **not** map Temporal roles |

Until user lifecycle management is disabled, you can still invite and remove **Cloud-managed** users outside of SCIM. SCIM users remain IdP-owned for delete/offboard.

**Ops escape hatch:** `ct ocld <env> user descim` clears WorkOS SCIM linkage on a user (not customer-facing). Use when support must untangle a stuck SCIM identity.

### Okta onboarding flow

1. Temporal Support enables the SCIM integration on your account. Enabling integration automatically emails a configuration link to the Okta administrator.
2. The Okta administrator opens the link, which leads to step-by-step configuration instructions.
3. Once configured, Temporal Cloud begins receiving SCIM messages and automatically onboards/offboards users and groups.

### Key behaviors

- User and group change events are applied within **10 minutes** of being made in the IdP.
- User lifecycle management with SCIM also allows user roles to be derived from group membership (roles assigned on the group in Cloud).
- Once a group has been synced in Temporal Cloud, assign roles to the group via the **Cloud UI**, **tcld**, or **Terraform**. See [User Group Management](https://github.com/temporalio/tcld?tab=readme-ov-file#user-group-management).

### Temporal-side SCIM (WorkOS + ocld)

Ops enable and tune Directory Sync via WorkOS:

```bash
ct ocld <env> account wos configure -a <ACCOUNT_ID> --enabled=true --cadence=5m
```

Notes:

- Invite the IdP admin for **Directory Sync** setup only (setup link / bearer token flow).
- Disabling SCIM does **not** remove already-synced users or groups.
- Lost bearer token: reset the directory and resend the setup link — do not expect the old token to keep working.

---

## Access model context

Access to Temporal Cloud is governed by role-based access control (RBAC). Each access principal has one account-level role and optionally one or more Namespace-level permissions. <!-- docs/cloud/manage-access/index.mdx -->

Access principals:

- **Users** — Individual user accounts
- **User Groups** — Groups for simplified access management
- **Service Accounts** — Automated access
- **Custom Roles** — Customer-defined permission sets assignable to principals

SAML and SCIM are identity integration features, not access principals.

Multiple accounts can coexist on the same email domain, each with its own SAML configuration tied to its unique Account ID. However, each email address can only be associated with a single Temporal Cloud account.

### Troubleshooting

- **Lost MFA access:** Click **Try another method** on the MFA screen. Enter your recovery code or receive a verification code via email. Then remove the authenticator via **My Profile** > **Password and Authentication** > **Authenticator App** > **Remove method** (not "reset").
- **Password reset:** If logged in: **My Profile** > **Password and Authentication** > **Reset Password**. If not logged in: enter email, click **Continue**, then **Forgot password**.
- **Email domain changes:** If your organization changed its email domain, create a support ticket with your previous and new email addresses and your Account Id.

### Debug / pitfalls

| Symptom / trap | Cause / fix |
|----------------|-------------|
| Social login (Google/Microsoft) still visible after SAML | SAML ≠ SAML-only. Enable `EnableSAMLOnlyConnection` / `--enable-saml-only-connection` to block non-SAML methods |
| Wrong login path | Prefer Enterprise identity → Continue, or `/login/saml?connection=ACCOUNT_ID-saml`. Do not confuse with generic `?connection=ACCOUNT_ID-saml` on the wrong entrypoint |
| SuperLogin | Internal support login path — not a customer SAML fix; avoid conflating with customer SSO troubleshooting |
| Dead zone | Misconfigured connection/domain mapping leaves users unable to complete either social or SAML login — verify connection name `ACCOUNT_ID-saml`, IdP domains, and cert |
| Stale permissions after role/group change | Expect delay from poll + bundle + cache; SCIM events apply within ~10 minutes, then Cloud auth caches may lag further |
