# Architecture

## Layered view

```mermaid
flowchart TB
    subgraph U[User]
      A[User in M365 Copilot Chat]
    end
    subgraph M[Microsoft 365 Copilot]
      B[Copilot Chat Orchestrator<br/>Work IQ + Skills/Tools]
      C[Declarative Agent<br/>declarativeAgent.json + instructions.md]
      D[Plugin Runtime<br/>plugin-manifest.json schema v2.4]
      V[(M365 Plugin Vault<br/>holds OAuth client_id + client_secret<br/>+ minted user tokens)]
    end
    subgraph G[GitHub]
      E[Remote MCP Server<br/>api.githubcopilot.com/mcp]
      F[GitHub REST API]
      O[GitHub OAuth Authorization Server]
    end

    A --> B
    B --> C
    C --> D
    D <-->|"Bearer &lt;user token&gt;"| E
    E --> F
    D -.uses.-> V
    V -.first-time login.-> O
    O -.grants per-user token.-> V

    style C fill:#cde,stroke:#369,stroke-width:2px
    style D fill:#cde,stroke:#369,stroke-width:2px
    style E fill:#fce,stroke:#a33,stroke-width:2px
```

**What we own**: the two cyan boxes (the declarative agent + plugin
manifest). Everything else is Microsoft or GitHub infrastructure.

---

## OAuth handshake (per user, first call only)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as Copilot Chat
    participant V as M365 Plugin Vault
    participant G as GitHub OAuth
    participant M as GitHub MCP Server

    U->>C: "What did I do this week?"
    C->>M: tools/call (no token yet)
    M-->>C: HTTP 401 + WWW-Authenticate:<br/>Bearer resource_metadata="…/oauth-protected-resource/mcp/"
    C->>U: "Sign in to GitHub" card
    U->>G: OAuth /authorize<br/>(client_id from vault, scopes: read:user, repo)
    G-->>U: Consent prompt
    U->>G: Approves
    G-->>C: redirect to teams.microsoft.com/api/platform/v1.0/oAuthRedirect<br/>?code=…
    C->>G: POST /token (code + client_secret from vault)
    G-->>C: access_token (per-user)
    C->>V: cache token under (tenant, user, plugin)
    C->>M: tools/call with Authorization: Bearer <user token>
    M->>M: validate token, call GitHub REST as the user
    M-->>C: tool result
    C-->>U: rendered answer
```

Subsequent turns reuse the cached token until it expires; the runtime
silently refreshes (or re-prompts) as needed.

---

## End-to-end request flow (single turn after auth)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant O as Copilot Orchestrator
    participant I as instructions.md (rules)
    participant M as GitHub MCP Server
    participant H as GitHub REST API

    U->>O: "Give me my daily briefing."
    O->>I: read persona rules + tool cheatsheet
    O->>M: tools/call get_me {}
    M->>H: GET /user
    H-->>M: {login: zunyangc, …}
    M-->>O: {login: zunyangc, …}
    O->>M: tools/call search_pull_requests<br/>{q: "is:pr author:@me state:open"}
    M->>H: GET /search/issues?q=…
    H-->>M: {items: [...PRs...]}
    M-->>O: {items: [...]}
    O->>M: tools/call search_pull_requests<br/>{q: "is:pr reviewed-by:@me updated:>2026-05-23"}
    M->>H: GET /search/issues?q=…
    H-->>M: {items: [...reviewed PRs...]}
    M-->>O: {items: [...]}
    O->>I: render with "Daily briefing" template
    O-->>U: 📌 Open PRs · ⚠️ Stuck · 🔍 Reviewed · 👉 Next action
```

---

## File-by-file responsibility

| File                                  | Owns                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------- |
| `appPackage/manifest.json`            | Teams app shell — id, icons, developer info, points at the declarative agent          |
| `appPackage/declarativeAgent.json`    | Conversation starters, agent name/description, points at instructions + plugin       |
| `appPackage/plugin-manifest.json`     | Declares the 14 MCP function names, the `RemoteMCPServer` runtime, and OAuth config  |
| `appPackage/mcp-tools.json`           | The curated tool list (matches the MCP server's `tools/list` shape)                  |
| `appPackage/instructions.md`          | The brain — persona routing, tool selection, output formatting, safety               |

---

## Why MCP instead of OpenAPI

The predecessor (`enginsights-copilot` v0.10.3) used a custom OpenAPI
plugin pointing at a FastAPI service we hosted. That worked, but:

| Concern                  | OpenAPI plugin (v0.10.3)                 | MCP plugin (this repo)                          |
| ------------------------ | ---------------------------------------- | ----------------------------------------------- |
| Hosting                  | Our Azure App Service ($)                | GitHub's `api.githubcopilot.com/mcp/` (free)   |
| Auth                     | GitHub App + tenant-config JSON          | Per-user OAuth (canonical)                      |
| Adding a new endpoint    | Code + deploy + rebuild manifest         | Add the tool name to `mcp-tools.json`           |
| Standard compliance      | Custom REST contract                     | Model Context Protocol (Anthropic-led standard) |
| Hackathon scoring        | Bonus 3/4/5 not earned                   | Bonus 3, 4, 5 all earned                        |
