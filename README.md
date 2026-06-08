# MSP Security Copilot Agent

> **Microsoft Agents League @ AI Skills Fest 2026 — Enterprise Agents Challenge**

A Microsoft 365 Copilot Declarative Agent that gives MSP (Managed Service Provider) IT teams instant, natural-language access to the security status of all their managed Microsoft 365 customer tenants — directly from Teams chat.

---

## The Problem

MSPs managing dozens of Microsoft 365 customer tenants face a daily challenge: monitoring the security posture of every customer requires logging into a dedicated portal, navigating dashboards, and manually checking findings, scores, and alerts. During incidents, every minute counts.

There is no way to ask a simple question like *"Which of my customers has the most critical security issues right now?"* and get an immediate, data-driven answer.

## The Solution

The **MSP Security Copilot Agent** integrates Microsoft 365 Copilot with a custom MSP Security Portal API. MSP technicians can ask security questions in plain language — in any language — and receive real-time answers drawn directly from live tenant data, without ever leaving Teams.

### Example interactions

| User asks | Agent does |
|-----------|-----------|
| *"What is the security status of all customers?"* | Calls `getSecurityOverview` → summarizes all tenants with scores and critical finding counts |
| *"Show me critical identity findings for Contoso"* | Calls `getTenants` to resolve the name, then `getFindings?severity=Critical&category=Identity` |
| *"Are there suspicious sign-ins at Fabrikam this week?"* | Calls `getSuspiciousSignins` → lists anomalous logins with location, risk level, BEC indicators |
| *"Give me an executive report for Northwind"* | Calls `getReportSummary` → score trend, top risks, resolved findings, recommended actions |

---

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for the full Mermaid diagram.

```
Microsoft 365 Copilot (Teams)
  └─ Declarative Agent (v1.7)
       └─ API Plugin (v2.4, OAuthPluginVault)
            └─ OAuth 2.0 via Microsoft Entra ID (access_as_user)
                 └─ MSP Security Portal REST API (ASP.NET Core 8)
                      └─ Azure SQL ← Background Scanner ← Microsoft Graph (multi-tenant)
```

### How it works

1. The technician asks a security question in Microsoft 365 Copilot (Teams)
2. The **Declarative Agent** determines which API plugin function to call
3. **OAuthPluginVault** initiates an OAuth 2.0 Authorization Code flow with Microsoft Entra ID
4. The user authenticates via SSO — no separate login required
5. Copilot calls the **MSP Security Portal API** with a delegated Bearer token
6. The API validates the token (`AddMicrosoftIdentityWebApi`), queries Azure SQL, and returns JSON
7. Copilot synthesizes the response into a natural language answer

---

## Repository Structure

```
copilot/
  manifest.json         # Teams App Manifest (v1.19)
  declarativeAgent.json # Declarative Agent definition (v1.7)
  plugin.json           # API Plugin manifest (v2.4)
  openapi.json          # OpenAPI 3.0 spec for the MSP Security Portal API
docs/
  architecture.md       # Architecture diagram (Mermaid)
```

> **Note:** The MSP Security Portal backend (ASP.NET Core API, scanner, database) is a production system and remains in a private repository. This repo contains only the Microsoft 365 Copilot integration layer.

---

## Setup Guide

### Prerequisites

- Microsoft 365 tenant with Copilot license
- A deployed MSP Security Portal API (ASP.NET Core 8) with:
  - Azure AD app registration (`access_as_user` scope exposed)
  - Multi-tenant Microsoft Graph consent from customer tenants
- Teams Developer Portal access

### Step 1 — Azure AD App Registration

1. Create (or use existing) an app registration in Azure Portal
2. Under **Expose an API**: add scope `access_as_user`
3. Under **Authentication**: add Web redirect URIs:
   - `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`
   - `https://teams.microsoft.com/api/platform/v1.0/oAuthConsentRedirect`
4. Pre-authorize the Teams Enterprise Token Store (`ab3be6b7-f5df-413d-ac2d-abf1e3fd9c0b`) for the scope
5. Create a client secret

### Step 2 — Teams Developer Portal: OAuth Registration

1. Go to [dev.teams.microsoft.com/tools/oauth](https://dev.teams.microsoft.com/tools/oauth)
2. Create a new **OAuth Client Registration** (not Entra SSO) with:
   - **Client ID**: your Azure AD app ID
   - **Client Secret**: your client secret
   - **Authorization URL**: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize`
   - **Token URL**: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
   - **Refresh URL**: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
   - **Scope**: `api://<APP_ID>/access_as_user offline_access`
   - **Base URL**: `https://your-portal-domain.com`
3. Copy the generated **reference_id**

### Step 3 — Configure the copilot files

Update placeholders in the files:

**`manifest.json`**: Replace `<YOUR_TEAMS_APP_ID>` with a new UUID and your domain

**`plugin.json`**: Replace `<YOUR_OAUTH_CLIENT_REGISTRATION_REFERENCE_ID>` with the reference_id from Step 2

**`openapi.json`**: Replace:
- `your-portal-domain.com` → your API domain
- `<YOUR_TENANT_ID>` → your Azure AD tenant ID
- `<YOUR_APP_ID>` → your Azure AD app ID

Host `openapi.json` publicly at `https://your-portal-domain.com/openapi.json`.

### Step 4 — Package and deploy

```bash
zip -r msp-security-agent.zip manifest.json declarativeAgent.json plugin.json openapi.json color.png outline.png
```

Upload to [Teams Developer Portal](https://dev.teams.microsoft.com) → Apps → Import app.  
Enable "Any Teams app for testing" in the OAuth registration settings.  
Install in Teams and test via Microsoft 365 Copilot.

---

## Technologies Used

- **Microsoft 365 Copilot** — Declarative Agent (manifest v1.19, agent schema v1.7)
- **Teams API Plugin** — schema v2.4, OAuthPluginVault authentication
- **OpenAPI 3.0** — REST API specification
- **Microsoft Entra ID** — OAuth 2.0 Authorization Code flow, delegated permissions
- **ASP.NET Core 8** — Backend REST API with `AddMicrosoftIdentityWebApi` JWT validation
- **Microsoft Graph API** — Multi-tenant security data collection
- **Azure SQL** — Security findings and score history storage
- **Azure VM + Nginx** — Hosting with TLS (Let's Encrypt)

---

## Judging Criteria Alignment

| Criterion | How this project addresses it |
|-----------|-------------------------------|
| **Accuracy & Relevance** | Live API data — no hallucination; all answers sourced directly from real tenant scans |
| **Reasoning & Multi-step Thinking** | Agent resolves tenant names via `getTenants` before calling detail endpoints; chains calls intelligently |
| **Creativity & Originality** | First-of-its-kind MSP security copilot: natural language across a multi-tenant security fleet |
| **User Experience** | Conversation starters, multilingual responses, structured findings sorted by severity |
| **Reliability & Safety** | OAuth 2.0 delegated auth, JWT validation, no sensitive data in repo, production-hardened API |

---

## License

MIT — see [LICENSE](LICENSE)

*Submitted to the Microsoft Agents League @ AI Skills Fest 2026 — Enterprise Agents Challenge*
