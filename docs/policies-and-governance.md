# Policies & Governance for the GitHub MCP Server

The GitHub MCP Server lets AI host applications securely access GitHub repositories, issues, pull requests, and other resources through GitHub's public APIs using the Model Context Protocol (MCP). It acts as a bridge between the host application's language model and GitHub, so the model can call curated GitHub tools and perform tasks on the user's behalf. This document explains the policies and controls organizations can use to manage MCP access in both first- and third-party host apps.

## How the GitHub MCP Server Works

The GitHub MCP Server provides access to GitHub resources and capabilities through a standardized protocol, with flexible deployment and authentication options tailored to different use cases. It supports two deployment modes, both built on the same underlying codebase.

### 1. Local GitHub MCP Server
* **Runs:** Locally alongside your IDE or application
* **Authentication & Controls:** Requires Personal Access Tokens (PATs). Users must generate and configure a PAT to connect. Managed via [PAT policies](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#restricting-access-by-personal-access-tokens).
  * Can optionally use GitHub App installation tokens when embedded in a GitHub App-based desktop tool (rare)

### 2. Remote GitHub MCP Server
* **Runs:** As a hosted service accessed over the internet
* **Authentication & Controls:** (determined by the chosen authentication method)
  * **GitHub App Installation Tokens:** Uses a signed JWT to request installation access tokens (OAuth 2.0–inspired but not Authorization Code flow). Provides granular control via [installation](https://docs.github.com/en/apps/using-github-apps/installing-a-github-app-from-a-third-party#requirements-to-install-a-github-app), [permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app) and [repository access controls](https://docs.github.com/en/apps/using-github-apps/reviewing-and-modifying-installed-github-apps#modifying-repository-access).
  * **OAuth App Authorization Code Flow:** Uses the standard OAuth 2.0 Authorization Code flow. Controlled via [OAuth App access policies](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions)
  * **Personal Access Tokens (PATs):** Managed via [PAT policies](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#restricting-access-by-personal-access-tokens).
  * **SAML SSO enforcement:** Applies when using OAuth Apps or PATs to access resources in organizations with SAML SSO enabled. Acts as an overlay control.

### Security Principles for Both Modes
* **Authentication:** Required for all operations, no anonymous access
* **Authorization:** Access enforced by GitHub's native permission model
* **Communication:** All data transmitted over HTTPS with optional SSE for real-time updates
* **Rate Limiting:** Subject to GitHub API rate limits based on authentication method
* **Token Storage:** Tokens should be stored securely using platform-appropriate credential storage
* **Audit Trail:** All operations are logged in GitHub's audit log when available

For integration architecture and implementation details, see the [Host Integration Guide](https://github.com/github/github-mcp-server/blob/main/docs/host-integration.md).

## Where It's Used

The GitHub MCP server can be accessed in various environments (referred to as "host" applications):
* **First-party Code Editors:** GitHub Copilot in VS Code, Visual Studio, JetBrains, Eclipse, and Xcode with integrated MCP support.
* **Third-party Code Editors** – Editors outside the GitHub ecosystem, such as Cursor, Windsurf, and Cline, that support connecting to MCP servers.
* **AI Chat Applications:** Tools like Claude Desktop and other AI assistants that connect to MCP servers to fetch GitHub context or execute write actions.

## What It Can Access

The MCP server accesses GitHub resources based on the permissions granted through the chosen authentication method (PAT, OAuth, or GitHub App). These may include:
* Repository contents (files, branches, commits)
* Issues and pull requests
* Organization and team metadata
* User profile information
* Actions workflow runs, logs, and statuses
* Security and vulnerability endpoints (if explicitly granted)

Access is always constrained by GitHub's public API permission model and the authenticated user's privileges.

## Control Mechanisms

### 1. Copilot Editors (first-party) → Editor Preview Policy

* **Policy:** Editor Preview Features
* **Location:** Enterprise/Org → Policies
* **What it controls:** When disabled, prevents Copilot editors from using the Remote GitHub MCP Server through OAuth connections.
* **What it does NOT affect:**
  * Local GitHub MCP server
  * Community-authored MCP servers using GitHub's public APIs
  * Remote GitHub MCP server configured with PAT
  * Third-party IDEs or host apps (Claude, Windsurf, Cursor, etc.)
* **When it expires:** This policy will no longer apply once both the Remote GitHub MCP Server and a given editor's MCP support reach general availability (GA).

> **Note:** Third-party hosts are governed separately by OAuth App or GitHub App policies.

### 2. Third-Party Host Apps (e.g., Cursor, Claude, Windsurf) → OAuth App or GitHub App Controls

#### a. OAuth App Access Policies
* **Control Mechanism:** OAuth App access restrictions
* **Location:** Org → Settings → Third-party Access → OAuth app policy
* **How it works:**
  * Organization admins must approve OAuth App requests before host apps can access organization data
  * Only applies when the host registers an OAuth App AND the user connects via OAuth 2.0 flow

#### b. GitHub App Installation
* **Control Mechanism:** GitHub App installation and permissions
* **Location:** Org → Settings → Third-party Access → GitHub Apps
* **What it controls:** Organization admins must install the app, select repositories, and grant scopes before the app can access organization-owned data or resources through the Remote GitHub Server.
* **How it works:**
  * Organization admins must install the app, specify repositories, and set scopes
  * Only applies when the host registers a GitHub App AND the user authenticates through that flow

> **Note:** The authentication methods available depend on what your host application supports. While PATs work with any remote MCP-compatible host, OAuth and GitHub App authentication are only available if the host has registered an app with GitHub. Check your host application's documentation or support for more info.

### 3. PAT Access from Any Host → PAT Restrictions

* **Types:** Fine-grained PATs (recommended) and Classic tokens (legacy)
* **Location:**
  * User level: [Personal Settings → Developer Settings → Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
  * Enterprise/Organization level: [Enterprise/Organization → Settings → Personal Access Tokens](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization) (to control PAT creation/access policies)
* **What it controls:** Applies to all host apps and both local & remote GitHub MCP servers when users authenticate via PAT.
* **How it works:** Access limited to the repositories and scopes selected on the token.
* **Limitations:** PATs do not adhere to OAuth App policies and GitHub App installation controls. They are user-scoped and not recommended for production automation.
* **Organization controls:**
  * Classic PATs: Can be completely disabled organization-wide
  * Fine-grained PATs: Cannot be disabled but require explicit approval for organization access

> **Recommendation:** We recommend using fine-grained PATs over classic tokens. Classic tokens have broader scopes and can be disabled in organization settings.

### 4. SAML SSO Enforcement (overlay control)

* **Location:** Enterprise/Organization → SSO settings
* **What it controls:** OAuth tokens and PATs must map to a recent SAML login to access SAML-protected organization data.
* **How it works:** Applies to ALL host apps when using OAuth or PATs.

> **Exception:** Does NOT apply to GitHub App installation tokens (these are installation-scoped, not user-scoped)

### Resources
* [Approving updated permissions for a GitHub App](https://docs.github.com/en/apps/using-github-apps/approving-updated-permissions-for-a-github-app)Installing a GitHub App from GitHub Marketplace for your organizations
* [About OAuth app access restrictions](https://docs.github.com/en/apps/using-github-apps/installing-a-github-app-from-github-marketplace-for-your-organizations)
* [Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions)
* [Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)

## Current Limitations

While the GitHub MCP Server provides dynamic tooling and capabilities, the following enterprise governance features are not yet available:

### Single Enterprise/Organization-Level Toggle

Enterprises and organizations cannot currently disable GitHub MCP Server access globally for all users with a single switch. To achieve this, administrators must use a combination of existing controls:

Organizations must use the combination of:
* **First-party Copilot Editors (VS Code, Visual Studio, JetBrains, Eclipse):**
  * Use the Editor Preview Features policy (while MCP remains in preview)
* **Third-party Host Applications:**
  * Configure OAuth app restrictions
  * Manage GitHub App installations
* **PAT Access in All Host Applications:**
  * Implement fine-grained PAT policies (applies to both remote and local deployments)

### MCP-Specific Audit Logging

While GitHub's standard audit logs capture OAuth authorization and API activity, MCP-specific events are not separately categorized. This means organizations currently lack:

* Real-time visibility into active MCP connections
* Dedicated MCP usage monitoring dashboards
* Tool-specific audit trails for compliance reporting
* Granular insights into which MCP operations are being performed

**Current state:** MCP activities appear as standard API calls in existing audit logs.

## Security Best Practices

### For Organizations

**GitHub App Management**
* Review [GitHub App installations](https://docs.github.com/en/apps/using-github-apps/reviewing-and-modifying-installed-github-apps) regularly
* Audit permissions and repository access
* Monitor installation events in audit logs
* Document approved GitHub Apps and their business purposes

**OAuth App Governance**
* Manage [OAuth App access policies](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions)
* Establish review processes for approved applications
* Monitor which third-party applications are requesting access
* Maintain an allowlist of approved OAuth applications

**Token Management**
* Mandate fine-grained Personal Access Tokens over classic tokens
* Establish token expiration policies (90 days maximum recommended)
* Implement automated token rotation reminders
* Review and enforce [PAT restrictions](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization) at the appropriate level

### For Developers and Users

**Authentication Security**
* Prioritize OAuth 2.0 flows over long-lived tokens
* Store tokens securely using platform-appropriate credential management
* Never commit tokens to source control
* Rotate tokens periodically (every 90 days recommended)

**Scope Minimization**
* Request only the minimum required scopes for your use case
* Regularly review and revoke unused token permissions
* Use repository-specific access instead of organization-wide access
* Document why each permission is needed for your integration

## Resources

**MCP:**
* [Model Context Protocol Specification](https://modelcontextprotocol.io/specification/2025-03-26)
* [Model Context Protocol Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization)

**GitHub Governance & Controls:**
* [OAuth App Access Restrictions](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions)
* [Fine-grained Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
* [GitHub App Permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
* [PAT Policies](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization)
---

**Questions or Feedback?**

Open an [issue in the github-mcp-server repository](https://github.com/github/github-mcp-server/issues) with the label "policies & governance" attached.

This document reflects GitHub MCP Server policies as of July 2025. Policies and capabilities continue to evolve based on customer feedback and security best practices.
