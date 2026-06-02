# Architecture

## Layered view

```mermaid
flowchart TB
    subgraph U[User]
      A[User in M365 Copilot Chat]
    end
    subgraph M[Microsoft 365 Copilot]
      B[Copilot Chat Orchestrator<br/>Work IQ + Skills/Tools + People]
      C[Declarative Agent<br/>declarativeAgent.json + instruction.txt]
      D[Plugin Runtime<br/>ai-plugin.json schema v2.4]
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

## Manager fan-out flow (single turn: "Anyone on my team need help?")

```mermaid
sequenceDiagram
    autonumber
    participant U as User (manager)
    participant O as Copilot Orchestrator
    participant P as Work IQ (People)
    participant I as instruction.txt (rules)
    participant M as GitHub MCP Server

    U->>O: "Anyone on my team need help right now?"
    O->>I: read 'Help-signals scan' pattern
    O->>P: resolve direct reports of @me
    P-->>O: [report_1, report_2, report_3, …]
    loop for each report
        O->>M: tools/call search_pull_requests<br/>{q: "is:pr author:<report> state:open"}
        M-->>O: open PRs for this engineer
    end
    O->>O: apply heuristics<br/>(idle > 7 days, draft no commits 14d, review-starvation ≥3/0)
    O-->>U: per-engineer signal list with PR links<br/>+ "workflow signals, not performance evaluation"
```

This is the core manager-seamless pattern: the agent never stores who
the team is, never caches PR state — it asks Work IQ at query time and
asks GitHub through the user's own token. The result is fresh by
construction.

---

## File-by-file responsibility

| File                                  | Owns                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------- |
| `appPackage/manifest.json`            | Teams app shell — id, icons, developer info, points at the declarative agent          |
| `appPackage/declarativeAgent.json`    | Conversation starters, agent name/description, `People` capability, points at plugin  |
| `appPackage/ai-plugin.json`           | Declares 14 MCP function names, the runtime endpoint, and OAuth-vault reference       |
| `appPackage/mcp-tools.json`           | The curated tool list (matches the MCP server's `tools/list` shape)                   |
| `appPackage/instruction.txt`          | The brain — persona routing, tool selection, output formatting, manager safety rules  |
| `m365agents.yml`                      | Provision/publish lifecycle steps (`teamsApp/*`, `oauth/register`)                    |

---

## Why MCP instead of OpenAPI

A predecessor repo (`enginsights-copilot`) used a custom OpenAPI plugin
pointing at a FastAPI service we hosted. That worked, but:

| Concern                  | OpenAPI plugin (predecessor)             | MCP plugin (this repo)                          |
| ------------------------ | ---------------------------------------- | ----------------------------------------------- |
| Hosting                  | Our Azure App Service ($)                | GitHub's `api.githubcopilot.com/mcp/` (free)    |
| Auth                     | GitHub App + tenant-config JSON          | Per-user OAuth (canonical)                      |
| Adding a new endpoint    | Code + deploy + rebuild manifest         | Add the tool name to `mcp-tools.json`           |
| Standard compliance      | Custom REST contract                     | Model Context Protocol (Anthropic-led standard) |
| Agent Academy capability surface | Bypasses MCP Apps + External MCP + OAuth-on-MCP | Demonstrates MCP Apps + External MCP + OAuth-on-MCP (the Special Ops focus areas) |
