# Architecture Diagram

## MSP Security Copilot Agent — System Architecture

```mermaid
flowchart TD
    User["👤 MSP Technician\n(Teams Desktop)"]

    subgraph M365["Microsoft 365 Copilot (Work IQ Orchestrator)"]
        Agent["🤖 MSP Security\nDeclarative Agent\n(declarativeAgent.json v1.7)"]
        Plugin["🔌 API Plugin\n(plugin.json v2.4)"]
        Vault["🔐 OAuthPluginVault\n(Teams Developer Portal\nOAuth Registration)"]
    end

    subgraph WorkIQ["Work IQ Knowledge (SharePoint)"]
        SP["📚 Security Runbooks\nhttps://YOUR_TENANT.sharepoint.com\n/sites/SecurityRunbooks"]
    end

    subgraph EntraID["Microsoft Entra ID"]
        OAuth["OAuth 2.0\nAuthorization Code Flow\naccess_as_user scope"]
    end

    subgraph Portal["MSP Security Portal\n(Azure VM · ASP.NET Core 8)"]
        API["REST API\n/api/overview\n/api/tenants\n/api/tenants/{id}/findings\n/api/tenants/{id}/reports/summary\n/api/tenants/{id}/forensic/suspicious-signins"]
        Auth["JWT Validation\nAddMicrosoftIdentityWebApi"]
        DB[("Azure SQL\nDatabase")]
        Scanner["Background Scanner\n(every 5 min per tenant)"]
    end

    subgraph Graph["Microsoft Graph API"]
        Tenants["Customer Tenants\n(multi-tenant consent)"]
    end

    User -->|"Natural language query\ne.g. 'Security status of all customers?'"| Agent
    Agent -->|"Selects function\ne.g. getSecurityOverview"| Plugin
    Plugin -->|"Token request"| Vault
    Vault -->|"OAuth flow"| OAuth
    OAuth -->|"Bearer token\naud: api://APP_ID/access_as_user"| Plugin
    Plugin -->|"GET /api/overview\nAuthorization: Bearer ..."| API
    API --> Auth
    Auth -->|"Validated"| DB
    DB -->|"Tenant security data"| API
    API -->|"JSON response"| Plugin
    Plugin -->|"Structured answer"| Agent
    Agent -->|"Natural language response"| User

    Agent -->|"Knowledge search\n(remediation runbooks)"| SP
    SP -->|"Remediation procedures"| Agent

    Scanner -->|"Reads security config\nvia delegated consent"| Tenants
    Scanner -->|"Stores findings & scores"| DB
```

## Key Technology Components

| Component | Technology |
|-----------|-----------|
| Copilot Agent | Microsoft 365 Copilot Declarative Agent (manifest v1.19, agent v1.7) — Work IQ Orchestrator |
| Work IQ Knowledge | SharePoint `OneDriveAndSharePoint` capability — Security Runbooks grounding |
| API Plugin | Teams API Plugin (schema v2.4, OpenAPI 3.0) |
| Authentication | OAuthPluginVault → Microsoft Entra ID OAuth 2.0 (delegated) |
| Backend API | ASP.NET Core 8, `AddMicrosoftIdentityWebApi` JWT validation |
| Data Store | Azure SQL (Entity Framework Core) |
| Infrastructure | Azure VM, Nginx reverse proxy, Let's Encrypt TLS |
| Security Data | Microsoft Graph API (multi-tenant app consent) |

## Authentication Flow

1. User triggers a function via Copilot chat
2. Teams **OAuthPluginVault** initiates OAuth 2.0 Authorization Code flow
3. User authenticates with **Microsoft Entra ID** (single sign-on)
4. Entra issues a delegated token with `access_as_user` scope
5. Copilot includes `Authorization: Bearer <token>` in every API call
6. API validates token audience (`api://<APP_ID>`) via `AddMicrosoftIdentityWebApi`
7. Request is processed and data returned from Azure SQL
