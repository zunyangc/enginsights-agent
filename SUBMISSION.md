# Hackathon Submission — Self-Scored Rubric

**Submission**: EngInsights Agent  
**Repository**: https://github.com/zunyangc/enginsights-agent  
**Author**: Zun Yang (`zunyangc`), paired with Copilot CLI (Claude Opus 4.7)  
**Demo video**: https://youtu.be/vWAyDhlOpow  
**Microsoft Learn username**: _to be filled at submission time_

---

## Core requirements

### ✅ R1 — Hosted in Microsoft 365 Copilot Chat

- `appPackage/manifest.json` is a Teams app manifest **v1.27** with a
  `copilotAgents.declarativeAgents[0]` block — the official packaging for
  M365 Copilot Chat agents.
- `appPackage/declarativeAgent.json` follows declarative-agent schema **v1.7**.
- Sideloads cleanly into https://m365.cloud.microsoft/chat via the
  Microsoft 365 Agents Toolkit `Provision` lifecycle
  (`teamsApp/zipAppPackage` → `teamsApp/validateAppPackage` →
  `teamsApp/extendToM365`).
- The demo video shows the agent running inside Copilot Chat with the
  agent tile and all six conversation starters.

### ✅ R2 — Microsoft IQ integration (Work IQ via Skills + Tools + People)

- Full mapping in [`docs/work-iq-integration.md`](docs/work-iq-integration.md).
- The agent uses the canonical Microsoft-documented extension pattern for
  Work IQ: **declarative agent (Skills) + plugin manifest (Tools) +
  `People` capability**.
- `declarativeAgent.json` declares `"capabilities": [ { "name": "People" } ]`,
  giving the orchestrator access to the user's org-graph entities
  (manager, reports, peers).
- `instruction.txt` exercises Work IQ memory: the user's GitHub `login`
  is remembered across turns; the orchestrator chains multiple tool
  calls per question; team membership is resolved per-query through the
  `People` capability rather than from a stored roster.
- The plugin manifest's `description_for_human` /
  `description_for_model` are Skills-style descriptions Work IQ uses to
  decide when to invoke this plugin.

---

## Bonus criteria

### ⭐ Bonus 3 — MCP Apps (higher rating)

- The plugin packages a complete MCP-server experience into the M365
  Copilot extensibility surface using the **v2.4 plugin manifest schema**.
- 14 GitHub MCP tools are surfaced as Copilot functions in
  [`appPackage/ai-plugin.json`](appPackage/ai-plugin.json), with their
  full JSON schemas in
  [`appPackage/mcp-tools.json`](appPackage/mcp-tools.json) (matching the
  MCP `tools/list` shape).
- The result is a complete, sideloadable enterprise MCP App — any user
  in the tenant can install it and use it without further infrastructure.
- Followed Microsoft's [Build MCP plugins for M365 Copilot][docs-mcp]
  guidance throughout.

[docs-mcp]: https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/build-mcp-plugins

### ⭐ Bonus 4 — External MCP server integration

- Integrated with `github/github-mcp-server` running on GitHub's hosted
  endpoint `https://api.githubcopilot.com/mcp/`.
- Verified live during development — the MCP discovery handshake (401 +
  `WWW-Authenticate` pointing at GitHub's authorization server) is what
  triggers the plugin runtime's OAuth flow.
- The MCP server is fully external (not built by us, not hosted by us);
  this proves the agent composes with arbitrary third-party MCP servers,
  not just one we wrote.

**Read operations exercised**: `get_me`, `search_pull_requests`,
`list_pull_requests`, `pull_request_read`, `search_issues`, `list_issues`,
`issue_read`, `list_commits`, `get_commit`, `search_commits`,
`search_code`, `search_repositories`, `search_users`, `get_file_contents`.

**Write operations**: deliberately excluded in v1 to minimise the trust
boundary. Enabling them is a one-line change to `mcp-tools.json` and
`ai-plugin.json`.

### ⭐ Bonus 5 — OAuth security for the MCP server

- `m365agents.yml` registers an `oauth/register` step that creates a
  reference in the **M365 Plugin Vault** keyed by `apigithubc`. The
  client_id / client_secret are entered once during Provision and never
  appear in the repository or the shipped manifest.
- `ai-plugin.json` references that vault entry via
  `"reference_id": "${{MCP_DA_AUTH_ID_APIGITHUBC}}"` — the literal id is
  injected at build time from `env/.env.dev` (Toolkit-managed).
- Per-user tokens are minted at first chat via the GitHub OAuth
  Authorization Code flow; Microsoft owns the redirect at
  `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`.
- Token refresh is handled by the M365 plugin runtime — we do not
  implement it ourselves.
- Error handling: if `get_me` returns 401, `instruction.txt` directs the
  agent to ask the user to re-authorise rather than retry blindly.

**Scope minimisation**: the OAuth App requests only `read:user` + `repo`
— no admin, no write_packages, no delete — matching the read-only tool
surface.

---

## Why this submission is non-trivial despite being declarative

| Concern                                          | Decision                                                                                       |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **Multi-tenant identity** without leaking data   | OAuth per user — every call runs as the user; no tenant config to misconfigure                 |
| **Hallucination risk** for tabulated results     | Strict templates in `instruction.txt` + "never invent data" rule + GitHub URL citation         |
| **Manager outputs being weaponised**             | Mandatory disclaimer framing as *workflow signals*, never as performance evaluation            |
| **Privacy** of "show me `<login>`'s PRs"         | Only fetch what the OAuth scope returns; 403 → say so plainly, never guess                     |
| **Tool-surface drift** as GitHub updates the MCP | `mcp-tools.json` is the manifest's contract — refresh via `tools/list` against the live server |
| **Stale answers** between turns                  | Agent is stateless — every question re-derives from GitHub; no cache to drift                  |

---

## Files reviewers should look at, in order

1. [`README.md`](README.md) — elevator pitch + sideload steps.
2. [`docs/architecture.md`](docs/architecture.md) — diagrams.
3. [`appPackage/instruction.txt`](appPackage/instruction.txt) — the brain.
4. [`appPackage/ai-plugin.json`](appPackage/ai-plugin.json) — MCP + OAuth declaration.
5. [`appPackage/declarativeAgent.json`](appPackage/declarativeAgent.json) — starters + `People` capability.
6. [`docs/work-iq-integration.md`](docs/work-iq-integration.md) — R2 mapping.
