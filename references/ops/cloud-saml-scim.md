# Cloud SAML and SCIM

SAML enables single sign-on (SSO) by allowing your identity provider to authenticate users into Temporal Cloud. SCIM automatically creates, updates, and removes users and groups in Temporal Cloud based on changes in your identity provider. <!-- docs/cloud/manage-access/index.mdx:24-26 -->

---

## SAML SSO

SAML 2.0 integration allows you to authenticate users of your Temporal Cloud account using your organization's IdP. This enforces corporate identity policies such as multi-factor authentication (MFA) and password complexity. <!-- docs/cloud/saml.mdx:21-24 -->

SAML is included in the Business, Enterprise, and Mission Critical plans. <!-- docs/cloud/saml.mdx:26-27 -->

### Configuration overview

1. Locate your Temporal Cloud Account Id (5-6 characters after the period in your Namespace Id, e.g., `f45a2`). <!-- docs/cloud/saml.mdx:31-34 -->
2. Configure SAML with your IdP (Microsoft Entra ID or Okta). <!-- docs/cloud/saml.mdx:36-38 -->
3. Share connection information with Temporal and test the connection. <!-- docs/cloud/saml.mdx:39 -->

### Entity identifier format

```
urn:auth0:prod-tmprl:ACCOUNT_ID-saml
```
<!-- docs/cloud/saml.mdx:61 -->

Example:

```
urn:auth0:prod-tmprl:f45a2-saml
```
<!-- docs/cloud/saml.mdx:67 -->

### Callback URL format

```
https://login.tmprl.cloud/login/callback?connection=ACCOUNT_ID-saml
```
<!-- docs/cloud/saml.mdx:74 -->

Example:

```
https://login.tmprl.cloud/login/callback?connection=f45a2-saml
```
<!-- docs/cloud/saml.mdx:80 -->

### Sign on URL format (Entra ID only)

```
https://cloud.temporal.io/login/saml?connection=ACCOUNT_ID-saml
```
<!-- docs/cloud/saml.mdx:86 -->

### Microsoft Entra ID configuration

1. Sign in to Microsoft Entra ID. <!-- docs/cloud/saml.mdx:48 -->
2. **Manage Microsoft Entra ID** > **View** > **Add > Enterprise application**. <!-- docs/cloud/saml.mdx:49-50 -->
3. **Create your own application** > name it (e.g., `temporal-cloud`) > select **Integrate any other application you don't find in the gallery**. <!-- docs/cloud/saml.mdx:52-53 -->
4. **Getting Started** > **Set up single sign on** > **SAML**. <!-- docs/cloud/saml.mdx:56-57 -->
5. In **Basic SAML Configuration**: <!-- docs/cloud/saml.mdx:58-81 -->
   - Set **Identifier (Entity ID)** to the entity identifier above.
   - Set **Reply URL (Assertion Consumer Service URL)** to the callback URL above.
   - Set **Sign on URL** to the sign on URL above.
6. In **Attributes & Claims**: <!-- docs/cloud/saml.mdx:96-101 -->
   - Set **Unique User Identifier (NameID)** to `user.userprincipalname`.
   - Set **NameID format** to `emailAddress`.
   - Ensure **Email** and **Name** are present under Additional claims.
7. Collect for Temporal: download **Certificate (Base64)** and copy **Login URL**. <!-- docs/cloud/saml.mdx:104-106 -->

### Okta configuration

1. Sign in to Okta Admin Console. <!-- docs/cloud/saml.mdx:114 -->
2. **Applications** > **Create App Integration** > **SAML 2.0** > **Next**. <!-- docs/cloud/saml.mdx:116-117 -->
3. Name the application (e.g., `temporal-cloud`). <!-- docs/cloud/saml.mdx:118-119 -->
4. In **Configure SAML**: <!-- docs/cloud/saml.mdx:120-148 -->
   - Set **Single sign on URL** to the callback URL above.
   - Set **Audience URI (SP Entity ID)** to the entity identifier above.
   - Set **Name ID format** to `EmailAddress`.
   - Set **Attribute Statements**: `email` and `name`.
5. In the **Feedback** section, select **Finish**. <!-- docs/cloud/saml.mdx:149 -->
6. On the application page > **Sign On** tab > **View SAML setup instructions**. Copy IdP settings and download the active certificate. <!-- docs/cloud/saml.mdx:151-155 -->

### Finish SAML configuration

Create a support ticket with: <!-- docs/cloud/saml.mdx:163 -->

- The sign-in URL from your application
- The X.509 SAML sign-in certificate in PEM format
- One or more IdP domains to map to the SAML connection

The IdP domain is generally the same as your email domain. Multiple IdP domains can be provided. <!-- docs/cloud/saml.mdx:169-170 -->

After Temporal confirms configuration is complete, log in with your email and click **Continue** to be directed to your IdP. <!-- docs/cloud/saml.mdx:172-173 -->

---

## SCIM user provisioning

SCIM lets you integrate your identity provider with Temporal Cloud to automate user provisioning and access. Changes in the IdP are automatically reflected in Temporal Cloud: <!-- docs/cloud/scim.mdx:24-28 -->

- User creation / onboarding
- User deletion / offboarding
- User membership in groups

SCIM groups can be mapped to Temporal Cloud roles and permissions. <!-- docs/cloud/scim.mdx:30 -->

SCIM is a paid feature. <!-- docs/cloud/scim.mdx:34 -->

### Supported IdP vendors

<!-- docs/cloud/scim.mdx:41-48 -->
- Okta
- Microsoft Entra ID (Azure AD)
- Google Workspace
- OneLogin
- CyberArk
- JumpCloud
- PingFederate
- Any SCIM 2.0-compliant provider

### Prerequisites

1. Configure SAML SSO first. <!-- docs/cloud/scim.mdx:54 -->
2. Identify your organization's IdP administrator and specify their contact details in the support ticket. <!-- docs/cloud/scim.mdx:55-56 -->
3. Submit a support ticket to enable SCIM. <!-- docs/cloud/scim.mdx:58 -->

When SCIM is enabled, you can still add and remove users outside of SCIM using the Temporal Cloud interface, until you disable user lifecycle management. You can always change a user's or group's Account Role from the Temporal Cloud interface. <!-- docs/cloud/scim.mdx:62-63 -->

### Okta onboarding flow

1. Temporal Support enables the SCIM integration on your account. Enabling integration automatically emails a configuration link to the Okta administrator. <!-- docs/cloud/scim.mdx:69-71 -->
2. The Okta administrator opens the link, which leads to step-by-step configuration instructions. <!-- docs/cloud/scim.mdx:72-73 -->
3. Once configured, Temporal Cloud begins receiving SCIM messages and automatically onboards/offboards users and groups. <!-- docs/cloud/scim.mdx:74 -->

### Key behaviors

- User and group change events are applied within **10 minutes** of being made in the IdP. <!-- docs/cloud/scim.mdx:78 -->
- User lifecycle management with SCIM also allows user roles to be derived from group membership. <!-- docs/cloud/scim.mdx:79 -->
- Once a group has been synced in Temporal Cloud, use `tcld` to assign roles to the group. See [User Group Management](https://github.com/temporalio/tcld?tab=readme-ov-file#user-group-management). <!-- docs/cloud/scim.mdx:80-81 -->

---

## Access model context

Access to Temporal Cloud is governed by role-based access control (RBAC). Each access principal (user, user group, or service account) has one account-level role and optionally one or more Namespace-level permissions. <!-- docs/cloud/manage-access/index.mdx:19-21 -->

Access principals: <!-- docs/cloud/manage-access/index.mdx:50-54 -->

- **Users** - Individual user accounts
- **User Groups** - Groups for simplified access management
- **Service Accounts** - Automated access

Multiple accounts can coexist on the same email domain, each with its own SAML configuration tied to its unique Account ID. However, each email address can only be associated with a single Temporal Cloud account. <!-- docs/cloud/manage-access/index.mdx:40-44 -->

### Troubleshooting

- **Lost MFA access:** Click **Try another method** on the MFA screen. Enter your recovery code or receive a verification code via email. Then reset the authenticator app via **My Profile** > **Password and Authentication**. <!-- docs/cloud/manage-access/index.mdx:77-85 -->
- **Password reset:** If logged in: **My Profile** > **Password and Authentication** > **Reset Password**. If not logged in: enter email, click **Continue**, then **Forgot password**. <!-- docs/cloud/manage-access/index.mdx:89-93 -->
- **Email domain changes:** If your organization changed its email domain, create a support ticket with your previous and new email addresses and your Account Id. <!-- docs/cloud/manage-access/index.mdx:105-109 -->
