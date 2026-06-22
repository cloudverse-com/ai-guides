# Exposing a Line-of-Business API as MCP via Azure API Management

**Architecture Design &amp; Setup Guide**
_Secured with Microsoft Entra ID &middot; Consumed by Microsoft Foundry agents and Copilot Studio_

> **Version:** 0.1 (draft) &middot; **Status:** For review

---

## Contents

- [1. Purpose and scope](#1-purpose-and-scope)
- [2. Selected architecture and decisions](#2-selected-architecture-and-decisions)
- [3. Solution architecture](#3-solution-architecture)
- [4. Microsoft Entra identity design](#4-microsoft-entra-identity-design)
  - [4.1 Resource app registration (the MCP API)](#41-resource-app-registration-the-mcp-api)
  - [4.2 Client app registration (for the agent platforms)](#42-client-app-registration-for-the-agent-platforms)
  - [4.3 Group-to-role assignment](#43-group-to-role-assignment)
  - [4.4 Token flow](#44-token-flow)
- [5. Azure API Management design](#5-azure-api-management-design)
  - [5.1 The two MCP servers](#51-the-two-mcp-servers)
  - [5.2 Inbound authorization policy](#52-inbound-authorization-policy)
  - [5.3 Backend authentication (APIM → AWS)](#53-backend-authentication-apim--aws)
  - [5.4 Operational caveats for MCP servers](#54-operational-caveats-for-mcp-servers)
- [6. Consuming the MCP servers](#6-consuming-the-mcp-servers)
  - [6.1 Microsoft Foundry agent](#61-microsoft-foundry-agent)
  - [6.2 Microsoft Copilot Studio](#62-microsoft-copilot-studio)
- [7. Step-by-step setup guide](#7-step-by-step-setup-guide)
- [8. Security considerations](#8-security-considerations)
- [9. Validation and testing](#9-validation-and-testing)
- [10. Assumptions and open items](#10-assumptions-and-open-items)

---

## 1. Purpose and scope

This document describes a production architecture for exposing an existing line-of-business (LOB) REST API as Model Context Protocol (MCP) servers, so the API can be invoked as tools by Microsoft Foundry agents and Microsoft Copilot Studio agents. The LOB API is hosted on AWS, exposes a mix of read-only and read/write operations, and is secured to its own callers with HTTP Basic authentication.

The design uses Azure API Management (APIM) as the AI gateway. APIM natively exposes a managed REST API as a remote MCP server and applies policy at the gateway: it validates the caller's Microsoft Entra token, authorizes the call against an application role, and authenticates separately to the AWS backend. Access is split into a read-only MCP server and a read/write MCP server, each gated by Entra group membership.

**In scope:** Entra app registrations and app roles, group-to-role mapping, the two MCP servers in APIM, the inbound authorization and outbound backend-authentication policies, and the consumer-side configuration in Foundry and Copilot Studio.
**Out of scope:** changes to the LOB API itself, network connectivity between APIM and AWS, and APIM instance provisioning (assumed to exist on a tier that supports MCP servers).

---

## 2. Selected architecture and decisions

The following decisions were confirmed and drive the rest of this document.

| Decision area | Choice | Rationale |
|---|---|---|
| **Caller identity** | OAuth identity passthrough (delegated / OBO) | Agents always run interactively. Passthrough is the only model that carries the signed-in user's group membership to APIM, satisfying per-user read vs read/write access. |
| **Authorization model** | App roles assigned to Entra security groups | Yields a clean `roles` claim, avoids the group-overage problem, and uses readable role names instead of group GUIDs in policy. |
| **MCP surface** | Two MCP servers: read-only and read/write | Keeps tool surfaces clean and lets each server enforce a single required role. |
| **Backend auth (APIM → AWS)** | HTTP Basic via `authentication-basic` policy | Matches how the LOB team secures the API; credentials stored as Key Vault-backed named values. |
| **Consumers** | Foundry agents + Copilot Studio | Both support manual OAuth configuration against an Entra-protected MCP server. |

---

## 3. Solution architecture

There are three independent trust boundaries. Conflating them is the most common source of confusion, so they are called out explicitly. The OAuth fields configured in Foundry or Copilot Studio govern only the first boundary; the backend credential is internal to APIM.

![Token flow and the three trust boundaries](architecture.png)

*Figure 1. Token flow and the three trust boundaries.*

### Trust boundaries

- **Boundary 1 &mdash; client to APIM (Entra OAuth).** The signed-in user authenticates to Entra; the agent presents the resulting access token (audience = the MCP server's app registration, with a `roles` claim) to APIM as a Bearer token.
- **Boundary 2 &mdash; token validation inside APIM.** APIM validates the token against Entra and asserts the required application role for the specific MCP server before forwarding anything.
- **Boundary 3 &mdash; APIM to AWS (Basic auth).** APIM authenticates to the LOB API with a service-account username and password, injected from Key Vault. The agent never sees these credentials.

> [!NOTE]
> APIM is **not** an OAuth authorization server. It does not host `/authorize` or `/token` endpoints; it only validates tokens issued by Entra. Every OAuth URL entered on the consumer side therefore points at the Entra tenant, not at APIM.

---

## 4. Microsoft Entra identity design

Two app registrations are used. This is the standard resource-server plus client pattern. The LOB application is **neither** of them &mdash; it is not represented in Entra at all, because its authentication is handled entirely by APIM at boundary 3.

### 4.1 Resource app registration (the MCP API)

Represents the APIM-published MCP endpoint and is the audience of the access token APIM validates.

- **Application ID URI:** `api://lob-mcp` (example).
- **App roles** (under *App roles*): `Mcp.Read` and `Mcp.ReadWrite`, both with allowed member types set to Users/Groups.
- **Expose an API:** publish a delegated scope (for example `api://lob-mcp/Mcp.Invoke`) so the client can request a token for this resource. Roles carry the read vs write decision; the scope establishes the audience.

### 4.2 Client app registration (for the agent platforms)

Drives the OAuth authorization-code flow on behalf of the user. Its client ID and secret are entered into the Foundry / Copilot Studio OAuth configuration.

- **Redirect URIs:** added after the consumer generates them (Foundry and Copilot Studio each produce a redirect URL during MCP tool setup &mdash; see Section 6).
- **API permissions:** delegated permission to the resource app's exposed scope (`api://lob-mcp/Mcp.Invoke`), plus `offline_access` for refresh tokens.
- **Client secret:** created and stored securely; required by the passthrough configuration unless the app uses certificate-based credentials.

### 4.3 Group-to-role assignment

Two existing Entra security groups are mapped to the two app roles on the resource app registration. Assignment is done on the resource app's enterprise application (*Users and groups* blade).

| Entra security group | App role assigned | Effective access |
|---|---|---|
| `LOB-MCP-ReadOnly` | `Mcp.Read` | Read-only MCP server only |
| `LOB-MCP-ReadWrite` | `Mcp.ReadWrite` | Read/write MCP server (and, if desired, read-only) |

**Why app roles rather than raw group claims:** when a user belongs to more than roughly 200 groups, Entra omits the groups claim and emits an overage indicator, forcing a Microsoft Graph lookup at validation time. App roles assigned to groups surface as a stable `roles` claim with human-readable values, so APIM policy stays simple and fast.

### 4.4 Token flow

1. The user interacts with the agent and is prompted to sign in (first use generates a consent link).
2. Entra authenticates the user and issues an access token with `aud` = the resource app and a `roles` claim reflecting the user's group-derived role(s).
3. The agent presents the token to the APIM MCP endpoint as a Bearer token.
4. APIM validates the token and asserts the role required by that MCP server (`Mcp.Read` or `Mcp.ReadWrite`).
5. On success, APIM injects the backend Basic credential and forwards the call to the AWS LOB API.

> [!IMPORTANT]
> **Scopes vs roles &mdash; a common snag.** The Scopes field on the consumer side (Section 6) takes the delegated scope `Mcp.Invoke`, **not** an app role. App roles (`Mcp.Read` / `Mcp.ReadWrite`) **cannot** be requested through the scope parameter &mdash; they are assigned to groups and arrive automatically in the token's `roles` claim. So every consumer, read-only and read/write alike, sends the same scope value (`api://lob-mcp/Mcp.Invoke`); the `roles` claim, populated from the user's group membership, is what each MCP server checks to allow or deny the call. The scope gets the caller in the door (audience + consent); the role decides what they can do once inside.

---

## 5. Azure API Management design

### 5.1 The two MCP servers

Import the LOB API into APIM (OpenAPI or manual operations), then create two MCP servers under **APIs → MCP Servers → Create MCP server → Expose an API as an MCP server**. Select operations as tools per server:

- **Read-only MCP server:** select only the safe read operations (typically GET).
- **Read/write MCP server:** select the operations that mutate state (plus reads if convenient).

Each MCP server has its own policy scope. APIM currently supports MCP *tools* (not resources or prompts) for servers exposed from managed REST APIs, and MCP servers are not supported in workspaces.

### 5.2 Inbound authorization policy

Applied to each MCP server. It validates the Entra token, pins the audience to the resource app, and requires the role for that server. Example for the **read/write** MCP server (swap the role value to `Mcp.Read` on the read-only server):

```xml
<inbound>
  <base />
  <validate-azure-ad-token tenant-id="{{aad-tenant-id}}"
      header-name="Authorization"
      failed-validation-httpcode="401"
      failed-validation-error-message="Unauthorized.">
    <client-application-ids>
      <application-id>{{client-app-id}}</application-id>
    </client-application-ids>
    <audiences>
      <audience>api://lob-mcp</audience>
    </audiences>
    <required-claims>
      <claim name="roles" match="any">
        <value>Mcp.ReadWrite</value>
      </claim>
    </required-claims>
  </validate-azure-ad-token>
</inbound>
```

### 5.3 Backend authentication (APIM → AWS)

The Basic credential for the LOB API is injected outbound. Use named values backed by Key Vault for the username and password rather than literals in policy:

```xml
<backend>
  <base />
  <authentication-basic username="{{lob-api-username}}"
                        password="{{lob-api-password}}" />
</backend>
```

Create the named values `lob-api-username` and `lob-api-password` in **APIM → Named values**, each referencing a secret in Azure Key Vault. The `authentication-basic` policy sets the Authorization header to the Base64 of `user:password` for the backend call.

### 5.4 Operational caveats for MCP servers

- Do not read `context.Response.Body` in MCP server policies &mdash; it forces response buffering and breaks the streaming the MCP protocol requires.
- If Application Insights / Azure Monitor diagnostic logging is enabled at the global (all-APIs) scope, set the Frontend Response payload-bytes-to-log to `0` to avoid interfering with MCP servers; log payloads selectively per API instead.
- External MCP clients must support MCP version `2025-06-18` or later when connecting to APIM-governed servers; this is relevant if you later front existing external MCP servers as well.

---

## 6. Consuming the MCP servers

### 6.1 Microsoft Foundry agent

In the agent's Playground, expand **Tools → Add → Model Context Protocol (MCP) → Create**, and configure an Entra-based connection using **OAuth Identity Passthrough**. Field mapping:

| Field | Value |
|---|---|
| Name | A unique label, e.g. `lob-mcp-readwrite` |
| Remote MCP Server endpoint | The APIM MCP server URL (`.../<mcp-server>/mcp` or `/sse`) |
| Authentication | OAuth Identity Passthrough |
| Client ID / Client secret | From the client app registration (Section 4.2) |
| Auth URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` |
| Token URL / Refresh URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` |
| Scopes | `api://lob-mcp/Mcp.Invoke offline_access` |

Select **Connect**; Foundry returns a redirect URL &mdash; add it to the client app registration's Redirect URIs, then Save the tool on the agent. On first use the agent prompts the user to sign in and consent, then calls the server with the user's token. Repeat for the read-only MCP server as a second tool. Users need at least the Foundry User role on the project, and their Entra tenant must match the project's tenant (cross-tenant passthrough is not supported).

### 6.2 Microsoft Copilot Studio

In the agent, open **Tools → Add a tool → Model Context Protocol**. In the wizard's Authentication section choose **OAuth 2.0 → Manual**, then Create. Copy the generated Redirect URL into the client app registration. Provide the same Entra authorization/token endpoints, the client ID and secret, and the resource scope. Copilot Studio registers a matching custom connector (visible under Custom connectors in the Power Platform portal); if your server advertises OAuth dynamic client registration, the wizard's **Dynamic discovery** option can configure this automatically instead of manual entry.

---

## 7. Step-by-step setup guide

End-to-end order of operations. Steps 1&ndash;6 are one-time platform setup; 7&ndash;9 are per-consumer.

1. Register the resource app (`api://lob-mcp`). Define app roles `Mcp.Read` and `Mcp.ReadWrite`; publish the delegated scope under *Expose an API*.
2. Register the client app. Grant delegated permission to the resource scope plus `offline_access`; create a client secret.
3. On the resource app's enterprise application, assign `LOB-MCP-ReadOnly` → `Mcp.Read` and `LOB-MCP-ReadWrite` → `Mcp.ReadWrite`.
4. In Key Vault, store the LOB service-account username and password; create matching named values in APIM.
5. Import the LOB API into APIM. Create two MCP servers (read-only and read/write) selecting the appropriate operations as tools.
6. Apply the inbound `validate-azure-ad-token` policy (with the correct required role per server) and the backend `authentication-basic` policy to each MCP server.
7. In Foundry, add each MCP server as an MCP tool using OAuth identity passthrough; copy the redirect URL back into the client app registration.
8. In Copilot Studio, add each MCP server via the MCP wizard (OAuth 2.0 → Manual); copy its redirect URL into the client app registration.
9. Validate end to end (Section 9), including negative tests for role enforcement.

---

## 8. Security considerations

- **Least privilege at the backend.** The Basic service account should hold only the permissions the exposed operations need; the read-only server should ideally map to a read-only backend identity if the LOB team can provide one.
- **Audience pinning.** Always assert the audience (`api://lob-mcp`) in policy so tokens minted for other resources cannot be replayed against the MCP endpoint.
- **Secret hygiene.** Keep the LOB credential and the client secret in Key Vault; rotate on a schedule. Prefer certificate credentials for the client app where policy allows.
- **Separate roles, separate servers.** Because each MCP server requires a distinct role, a read-only user presenting a valid token is still rejected by the read/write server at boundary 2.
- **Tenant boundary.** OAuth identity passthrough requires the user and the Foundry project to share a tenant; plan accordingly for any external collaborators.
- **Tool approval.** Consider requiring per-call approval on write tools in the agent configuration for an added human checkpoint on mutations.

---

## 9. Validation and testing

- **Plumbing first.** Before enabling OAuth, smoke-test each MCP server with a subscription key (header `Ocp-Apim-Subscription-Key`) to confirm tool discovery and backend connectivity.
- **Token validation.** Use an MCP inspector or the Foundry playground to confirm sign-in, consent, and tool invocation under the user's identity.
- **Positive tests.** A `LOB-MCP-ReadWrite` member can invoke write tools on the read/write server; a `LOB-MCP-ReadOnly` member can invoke read tools on the read-only server.
- **Negative tests.** A read-only user is rejected (401/403) by the read/write server; a token with no role is rejected by both.
- **Refresh.** Confirm long sessions survive token expiry (validates that `offline_access` is present and refresh works).

---

## 10. Assumptions and open items

- The APIM instance is on a tier that supports MCP servers and has network reachability to the AWS-hosted LOB API.
- The LOB API accepts a single shared service account over HTTP Basic; if it instead issues its own session token from a login endpoint, boundary 3 changes to a token-exchange step in policy (not required for the confirmed Basic-header case).
- Foundry OAuth identity passthrough has had model-specific rough edges reported in the playground; prototype the passthrough path early against the model(s) the agents will use.
- Group names (`LOB-MCP-ReadOnly` / `LOB-MCP-ReadWrite`) and the `api://lob-mcp` identifier are placeholders to be finalized with the platform team.

---

_End of document &mdash; draft for review._
