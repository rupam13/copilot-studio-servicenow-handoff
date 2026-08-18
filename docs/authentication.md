# Authentication

## Principle

Two distinct auth paths, chosen by what the call does:

| Call type | Auth | Why |
|---|---|---|
| User-scoped reads (my tickets, my requests) | OAuth on-behalf-of the user | ServiceNow ACLs must apply to the actual user |
| System writes (create incident, log deflection) | OAuth client credentials, integration user | Consistent attribution, least privilege |

Using a single service account for everything is the most common shortcut here, and it destroys per-user entitlement. Any user could then read any ticket through the bot.

---

## 1. Register the OAuth application in ServiceNow

**System OAuth > Application Registry > New > Create an OAuth API endpoint for external clients**

| Field | Value |
|---|---|
| Name | `Copilot Studio Integration` |
| Client ID | auto-generated |
| Client Secret | auto-generated, store in **Azure Key Vault** |
| Refresh token lifespan | 8,640,000 s (100 days) |
| Access token lifespan | 1,800 s (30 min) |

> The client secret goes to Key Vault immediately. It must never appear in a flow definition, a solution export, or this repository.

---

## 2. Create the integration user

**User Administration > Users > New**

| Setting | Value |
|---|---|
| User ID | `copilot.integration` |
| Web service access only | **true** |
| Internal Integration User | **true** |
| Password | random, stored in Key Vault, never used interactively |

### Roles — least privilege

| Role | Why |
|---|---|
| `itil` | Create and update incidents |
| `sn_incident_write` | Scoped incident write |
| `knowledge` | Read published KB articles |
| Custom `u_bot_deflection_write` | Write to the deflection table only |

**Do not grant `admin`.** An integration user with admin rights turns a leaked token into full instance compromise. If a call fails with 403, fix the ACL, do not add a role.

---

## 3. Configure the custom connector in Power Platform

| Setting | Value |
|---|---|
| Authentication type | OAuth 2.0 |
| Identity provider | Generic OAuth 2 |
| Client ID | `<CLIENT_ID>` |
| Client secret | reference the Key Vault secret |
| Authorization URL | `https://<SN_INSTANCE>.service-now.com/oauth_auth.do` |
| Token URL | `https://<SN_INSTANCE>.service-now.com/oauth_token.do` |
| Refresh URL | `https://<SN_INSTANCE>.service-now.com/oauth_token.do` |
| Scope | `useraccount` |

---

## 4. Copilot Studio agent authentication

**Settings > Security > Authentication > Authenticate with Microsoft Entra ID**

This is what makes `System.User.PrincipalName` trustworthy. Without agent authentication that variable is unreliable, and every caller-attribution guarantee in this repo collapses.

| Setting | Value |
|---|---|
| Service provider | Microsoft Entra ID |
| Require users to sign in | **Yes** |
| Token exchange URL | configured for on-behalf-of flow |

---

## 5. DLP policy

The custom ServiceNow connector must sit in the **Business** data group alongside SharePoint, Teams, and Outlook. If it lands in the same policy group as connectors marked Non-Business, the flow will fail at runtime with an opaque policy error that is genuinely painful to diagnose.

---

## Secret Rotation

| Secret | Rotation | Owner |
|---|---|---|
| ServiceNow client secret | 90 days | Platform team |
| Integration user password | 180 days | Platform team |
| Refresh token | Auto, on use | Runtime |

Set a Key Vault expiry notification at 14 days. An expired client secret fails **silently in the background** — the agent keeps answering questions and simply stops creating tickets, which can go unnoticed for days.
