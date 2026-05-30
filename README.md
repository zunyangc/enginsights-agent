# EngInsights Agent

> A declarative Microsoft 365 Copilot Chat agent for engineering intelligence,
> powered by GitHub's hosted MCP server with per-user OAuth — **zero
> executable code, zero hosted backend.**

---

## What this is (and why it's small)

This repository ships a **Microsoft 365 Declarative Agent** that lives in
Copilot Chat. A user can ask "what did I do this week?" or "which of my
pull requests are stuck?" and get a grounded, GitHub-cited answer.

**The entire agent is 4 JSON files + 1 markdown prompt + 2 PNG icons.**
There is no FastAPI service, no Docker container, no Azure App Service, no
database. The hard work is delegated to two Microsoft- and GitHub-owned
runtimes:

- **Microsoft 365 Copilot Chat** orchestrates the conversation and routes
  intents to "tools" using its built-in IQ (Work IQ via Skills + Tools).
- **GitHub's hosted Remote MCP Server** (`https://api.githubcopilot.com/mcp/`)
  speaks the Model Context Protocol and proxies into the GitHub REST API
  using the **end user's own OAuth token**.

This is the same architecture pattern Microsoft documents in
["Build Model Context Protocol (MCP) plugins for Microsoft 365 Copilot"][docs-mcp].
We are not inventing infrastructure — we are composing it.

[docs-mcp]: https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/build-mcp-plugins

---

## Hackathon scoring — at a glance

| Criterion                                  | How this repo satisfies it                                                                    |
| ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| ✅ R1 — Hosted in M365 Copilot Chat        | Declarative agent, sideloaded as `appPackage.zip`                                              |
| ✅ R2 — Microsoft IQ integration           | **Work IQ** via Skills + Tools — Copilot Chat's orchestrator + MCP plugin                      |
| ⭐ Bonus 3 — MCP Apps                      | The plugin packages an MCP-server experience into the Copilot extensibility surface           |
| ⭐ Bonus 4 — External MCP server           | Official `github/github-mcp-server`, hosted by GitHub at `api.githubcopilot.com/mcp/`         |
| ⭐ Bonus 5 — OAuth on the MCP server       | Per-user GitHub OAuth 2.0 with the M365 Plugin Vault holding the client credentials           |

Full self-scored rubric: see [`SUBMISSION.md`](SUBMISSION.md).

---

## Repository layout

```
enginsights-agent/
├── README.md                       (this file)
├── LICENSE                         (MIT)
├── SUBMISSION.md                   (hackathon self-evaluation)
├── appPackage/                     (the entire deployable agent)
│   ├── manifest.json               Teams app manifest v1.20
│   ├── declarativeAgent.json       Declarative agent schema v1.2
│   ├── plugin-manifest.json        Plugin manifest schema v2.4 (RemoteMCPServer + OAuth)
│   ├── mcp-tools.json              Curated subset of GitHub MCP tools (tools/list shape)
│   ├── instructions.md             The agent's brain — persona routing, output formatting, safety
│   ├── color.png                   192×192 colour icon
│   ├── outline.png                 32×32 outline icon
│   └── build/
│       └── appPackage.zip          ← sideload this into Copilot Chat
├── docs/
│   ├── architecture.md             Diagrams + sequence flows
│   └── work-iq-integration.md      How Work IQ shows up in this design
└── demo/
    └── video-script.md             5-minute hackathon demo script
```

---

## How a single user-turn flows end-to-end

```
1. User in Copilot Chat:  "What did I do this week?"
                ↓
2. Copilot Chat orchestrator (Work IQ) reads instructions.md,
   matches the developer persona, selects the get_me + search_pull_requests tools.
                ↓
3. Plugin runtime calls https://api.githubcopilot.com/mcp/
   Authorization: Bearer <user's GitHub OAuth token from Plugin Vault>
   { jsonrpc:"2.0", method:"tools/call", params:{ name:"get_me" } }
                ↓
4. GitHub Remote MCP Server validates the OAuth token, calls GitHub REST API
   under the user's scopes, returns { login, name, html_url, … }.
                ↓
5. Orchestrator computes "updated:>YYYY-MM-DD" for the last 7 days,
   calls search_pull_requests with "is:pr author:@me updated:>…".
                ↓
6. Orchestrator renders the answer using the "Daily briefing" template
   from instructions.md — bullets, idle days, github.com URLs, one next action.
```

No code in this repo runs at request-time. Every behaviour above is a
**declared rule**.

---

## Try it locally (5-minute sideload)

### Prerequisites

- A Microsoft 365 tenant with **Copilot Chat** (free or paid Copilot license).
- VS Code with the **Microsoft 365 Agents Toolkit** extension (v6.3.x+).
  (`aka.ms/M365AgentsToolkit`)
- A GitHub account.

### Step 1 — Register a GitHub OAuth App

1. Go to https://github.com/settings/developers → **OAuth Apps** → **New OAuth App**.
2. Fill in:
   - **Application name**: `EngInsights Agent — Dev`
   - **Homepage URL**: `https://github.com/zunyangc/enginsights-agent`
   - **Authorization callback URL**: `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`
     (This is Microsoft's fixed OAuth redirect for plugin runtimes — we do
     not host it.)
3. After "Register application", note the **Client ID** and click
   **Generate a new client secret** → copy it once and stash it locally
   (you won't see it again).

### Step 2 — Provision via Microsoft 365 Agents Toolkit

1. `git clone https://github.com/zunyangc/enginsights-agent && code enginsights-agent`
2. Open the Agents Toolkit side panel → **Provision**.
3. When prompted, paste your GitHub OAuth Client ID and Client Secret.
   Toolkit uploads them to the **M365 Plugin Vault**, generates a vault
   `reference_id`, and substitutes it into `plugin-manifest.json` in
   place of `${{OAUTH_REGISTRATION_ID}}`.
4. Toolkit then packages everything into `appPackage/build/appPackage.zip`.

### Step 3 — Sideload into Copilot Chat

1. Open https://m365.cloud.microsoft/chat.
2. Click **Agents** → **+ Add an agent** → **Upload custom agent**.
3. Drop `appPackage/build/appPackage.zip`.
4. Pick **EngInsights Agent** from the agent rail.
5. On your first message, Copilot will pop a "Sign in to GitHub" card —
   complete the OAuth handshake (GitHub will ask you to grant `read:user`
   and `repo` to your OAuth App).
6. Try one of the conversation starters, or ask "what did I do this week?"

---

## What it can do (capability surface)

The agent is bounded by the 14 GitHub MCP tools we exposed in
[`appPackage/mcp-tools.json`](appPackage/mcp-tools.json):

- `get_me` — identity discovery
- `search_pull_requests`, `list_pull_requests`, `pull_request_read`
- `search_issues`, `list_issues`, `issue_read`
- `search_commits`, `list_commits`, `get_commit`
- `search_repositories`, `search_users`
- `search_code`, `get_file_contents`

Anything those tools can read from GitHub (within the signed-in user's
permissions), the agent can answer.

Anything those tools can **not** read — e.g. private analytics, team
roster from your HR system, build telemetry — the agent does not see.
This is on purpose: OAuth scopes are the security boundary.

---

## What it can't do (deliberate non-goals)

- **No writes.** The plugin only exposes read-only tools. You can't open
  a PR, comment, or merge through this agent. This is to keep the trust
  boundary minimal for v1.
- **No persistent team roster.** Unlike the original v0.10.3 prototype,
  this agent has no `TeamConfig` storage. Managers either name the team
  members in the prompt ("alice, bob, carol on octocat/hello-world") or
  use GitHub's own teams via the GitHub MCP server's read endpoints.
- **No deterministic numerical aggregation.** The LLM tabulates results,
  so "PRs idle > 7 days" is a sort + filter the orchestrator performs.
  This is documented behaviour, not a bug.

---

## Why this beats a hand-rolled FastAPI backend

A predecessor repo (`enginsights-copilot` v0.10.3, also by the author)
implemented the same product with ~2,800 LOC of FastAPI, a custom
OpenAPI-described plugin, GitHub App PEM-based authentication, Azure App
Service + ACR, a B1 Linux plan, an Azure Files share for tenant config,
and a daily PEM refresh story. It works. It's also a lot of code, a lot
of cloud bill, and a lot of attack surface.

This rewrite achieves the same outcome with:

- Zero deployed services (Microsoft + GitHub run the runtimes).
- Zero secrets in repo (Plugin Vault holds the client_secret; user tokens
  are minted at chat-time).
- Zero per-tenant state (the OAuth user *is* the identity).
- A surface area judges can read end-to-end in 10 minutes.

The trade-off is that the LLM does more of the rendering work, so output
quality lives in `instructions.md` instead of Python code.

---

## License

MIT — see [`LICENSE`](LICENSE).
