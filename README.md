# EngInsights Agent

A **Microsoft 365 Declarative Agent** that turns GitHub into an engineering
intelligence layer inside Copilot Chat — for individual developers _and_
engineering managers — with **zero hosted backend**.

You ask:

> _"Anyone on my team need help right now?"_

The agent fans out across your direct reports (resolved via Work IQ), checks
their open pull requests on GitHub, and flags anything stuck > 7 days,
abandoned drafts, or review starvation — with PR links you can click. All
through the user's own GitHub OAuth token; nothing leaves the trust
boundary GitHub already enforces.

---

## What's in the box

The entire agent is **5 JSON files + 1 text prompt + 2 PNG icons**.
There is no FastAPI service, no Docker container, no Azure App Service, no
database. Two managed runtimes do the work:

- **Microsoft 365 Copilot Chat** orchestrates the conversation, resolves
  identity via **Work IQ** (Skills + Tools), and chains tool calls.
- **GitHub's hosted Remote MCP Server** at
  `https://api.githubcopilot.com/mcp/` speaks Model Context Protocol and
  proxies into the GitHub REST API using the **end user's own OAuth token**.

Pattern reference: Microsoft Learn's
[Build MCP plugins for Microsoft 365 Copilot][docs-mcp].

[docs-mcp]: https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/build-mcp-plugins

---

## Hackathon scoring — at a glance

| Criterion                                  | How this repo satisfies it                                                                    |
| ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| ✅ R1 — Hosted in M365 Copilot Chat        | Declarative agent, sideloaded as `appPackage.zip` via the Agents Toolkit                       |
| ✅ R2 — Microsoft IQ integration           | **Work IQ** via Skills + Tools — Copilot orchestrator + MCP plugin + `People` capability       |
| ⭐ Bonus 3 — MCP Apps                      | Plugin uses the v2.4 manifest schema over an MCP server (`api.githubcopilot.com/mcp/`)         |
| ⭐ Bonus 4 — External MCP server           | Official `github/github-mcp-server`, hosted by GitHub at `api.githubcopilot.com/mcp/`         |
| ⭐ Bonus 5 — OAuth on the MCP server       | Per-user GitHub OAuth 2.0; client credentials live in the M365 Plugin Vault, never in repo    |

Full self-scored rubric: [`SUBMISSION.md`](SUBMISSION.md).

---

## Repository layout

```
enginsights-agent/
├── README.md                     (this file)
├── LICENSE                       (MIT)
├── SUBMISSION.md                 (hackathon self-evaluation)
├── m365agents.yml                Agents Toolkit lifecycle (provision/publish)
├── .vscode/                      Toolkit launch + MCP-server configs
├── env/
│   └── .env.dev                  Empty template; Toolkit fills it during Provision
├── evals/
│   └── prompts.json              Starter dataset for the M365 Copilot Agent Evaluations CLI
├── appPackage/
│   ├── manifest.json             Teams app manifest v1.27 (registers the declarative agent)
│   ├── declarativeAgent.json     Declarative-agent schema v1.7 (starters, instructions, plugin)
│   ├── ai-plugin.json            Plugin manifest v2.4 (function names + OAuth)
│   ├── mcp-tools.json            Curated subset of GitHub MCP tools (`tools/list` shape)
│   ├── instruction.txt           The agent's brain (~8 KB) — routing, formatting, safety
│   ├── color.png                 192×192 colour icon
│   └── outline.png               32×32 outline icon
└── docs/
    ├── architecture.md           Diagrams + sequence flows
    └── work-iq-integration.md    How Work IQ shows up in this design
```

No code in this repo runs at request-time. Every behaviour is a **declared rule**.

---

## What the agent can do

Six conversation starters (in `declarativeAgent.json`), front-loaded with
the manager patterns:

| # | Starter             | What it does                                                                                   |
| - | ------------------- | ---------------------------------------------------------------------------------------------- |
| 1 | **Need help?**      | Fan out across your team, flag stuck PRs / idle drafts / review starvation                     |
| 2 | **1:1 prep**        | Pick a report → wins (merged), concerns (stuck), in-flight (open) with PR links                |
| 3 | **Monthly digest**  | Per-engineer table + cycle-time / review-load / shipping-theme trend callouts for the window   |
| 4 | **Team snapshot**   | What did the team ship this week on `<repo>`                                                   |
| 5 | **Daily briefing**  | Your own open PRs, stuck items, reviewed this week, single next action                         |
| 6 | **What's stuck?**   | Your open PRs sorted by idle days, with the reason each one is waiting                         |

It also handles ad-hoc questions like _"What did `<login>` ship on
`<repo>` since 1 April?"_ — using `search_pull_requests`,
`pull_request_read`, `search_commits`, etc. The capability surface is
strictly the **14 read-only GitHub MCP tools** declared in
`appPackage/mcp-tools.json`.

Anything those tools cannot read from GitHub (HR roster, build telemetry,
private analytics), the agent does not see. OAuth scopes are the security
boundary.

---

## Try it locally (≈ 5-minute sideload)

### Prerequisites

- A Microsoft 365 tenant with **Copilot Chat** enabled.
- VS Code with the **Microsoft 365 Agents Toolkit** extension (v6.3.x or newer).
- A GitHub account.

### Step 1 — Register a GitHub OAuth App

1. https://github.com/settings/developers → **OAuth Apps** → **New OAuth App**.
2. Fill in:
   - **Application name**: `EngInsights Agent — Dev`
   - **Homepage URL**: `https://github.com/zunyangc/enginsights-agent`
   - **Authorization callback URL**:
     `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`
     (Microsoft's fixed plugin redirect — we don't host it.)
3. Copy the **Client ID**. Generate a **Client Secret** and copy it once.

### Step 2 — Clone and Provision

```bash
git clone https://github.com/zunyangc/enginsights-agent.git
code enginsights-agent
```

In VS Code:

1. Open the **Microsoft 365 Agents Toolkit** side panel → sign in to your
   M365 dev account.
2. Click **Provision**. When prompted, paste your GitHub OAuth Client ID
   and Client Secret. The Toolkit will:
   - create the Teams app shell (writes `TEAMS_APP_ID` into `env/.env.dev`),
   - register your OAuth credentials in the **M365 Plugin Vault** (writes
     a reference id into `env/.env.dev` — the secret itself never touches
     the repo),
   - build `appPackage/build/appPackage.dev.zip`,
   - validate it, and call `teamsApp/extendToM365` to register it in your tenant.

### Step 3 — Open it in Copilot Chat

1. Open https://m365.cloud.microsoft/chat.
2. Click **Agents** → find **EngInsights -dev** in your agent list.
3. On the first message, Copilot will pop a "Sign in to GitHub" card —
   complete the OAuth handshake (scopes: `read:user`, `repo`).
4. Click one of the conversation starters, or ask anything from the
   capability list above.

### Step 4 (optional) — Schedule a recurring prompt

In Copilot Chat, open the agent → ⋮ menu → **Scheduled prompts** → create
one for, e.g., "Daily briefing" at 09:00. Microsoft 365 Copilot runs the
prompt on the schedule and posts the result into your chat history.

Combined with this agent's stateless design, that gives you a daily /
weekly / monthly cadence with **no cache file** — GitHub remains the
source of truth, the agent re-derives the answer each time.

---

## Architecture in one sentence

The Teams **manifest** registers a **declarative agent**; the declarative
agent points at a **plugin manifest**; the plugin manifest says _"call
GitHub's hosted MCP server, authenticated with OAuth pulled from
Microsoft's plugin vault."_ Everything else is rules in `instruction.txt`.

See [`docs/architecture.md`](docs/architecture.md) for sequence diagrams.

---

## Deliberate non-goals

- **No writes.** Read-only tools only. You cannot open a PR, comment, or
  merge through this agent (v1 trust boundary).
- **No persistent team roster.** The agent resolves "your team" via Work
  IQ's `People` capability at the time of the query. Nothing is cached.
- **No proactive Teams channel posts.** Replies go to the user's own
  Copilot Chat. For proactive delivery you'd add a Power Automate flow or
  a Bot Framework bot.
- **No deterministic SQL-style aggregation.** The orchestrator tabulates
  with the LLM, so "PRs idle > 7 days" is a sort/filter the model
  performs against the tool output. Documented behaviour, not a bug.

---

## Authors

Designed and built by **Zun Yang** with **Copilot CLI (Claude Opus 4.7)**
as a paired coding partner. Released under MIT — see [`LICENSE`](LICENSE).
