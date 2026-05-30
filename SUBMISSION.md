# Hackathon Submission — Self-Scored Rubric

**Submission**: EngInsights Agent
**Repository**: https://github.com/zunyangc/enginsights-agent
**Author**: Zun Yang Chin (`zunyangc`)
**Demo video**: _to be filled at submission time_
**Microsoft Learn username**: _to be filled at submission time_

---

## Core Requirements

### ✅ R1 — Hosted in Microsoft 365 Copilot Chat

**Evidence**:
- `appPackage/manifest.json` is a Teams app manifest v1.20 with a
  `copilotAgents.declarativeAgents[0]` block — the official packaging for
  M365 Copilot Chat agents.
- `appPackage/declarativeAgent.json` follows declarative-agent schema v1.2.
- Sideloads cleanly into https://m365.cloud.microsoft/chat via Agents →
  Upload custom agent.
- Demo video shows the agent running inside Copilot Chat with the agent
  rail tile and conversation starters.

### ✅ R2 — Microsoft IQ integration (Work IQ via Skills + Tools)

**Evidence**:
- Full mapping in [`docs/work-iq-integration.md`](docs/work-iq-integration.md).
- The agent uses the **canonical** Microsoft-documented extension pattern
  for Work IQ — declarative agents (Skills) calling tools (Tools) via the
  plugin manifest.
- `instructions.md` exercises Work IQ memory: persona detection persists,
  the user's `login` is remembered across turns, and the orchestrator
  chains multiple tool calls (`get_me` → search → render) without
  user prompting.
- The plugin manifest's `description_for_model` is a Skills-style
  description that Work IQ uses to decide when to invoke this plugin.

---

## Bonus Criteria

### ⭐ Bonus 3 — MCP Apps (Higher Rating)

**Evidence**:
- The plugin packages a full MCP-server experience into the M365 Copilot
  extensibility surface using the **v2.4 plugin manifest schema** with
  `runtime.type = RemoteMCPServer` — Microsoft's first-party MCP App
  packaging format.
- 14 MCP tools are surfaced as Copilot functions in
  [`appPackage/plugin-manifest.json`](appPackage/plugin-manifest.json),
  with their schemas in [`appPackage/mcp-tools.json`](appPackage/mcp-tools.json)
  (matching the MCP `tools/list` shape).
- The result is a complete, sideloadable enterprise MCP App — anyone in
  the tenant can install it and use it without further infrastructure.
- Followed Microsoft's
  [Build MCP plugins for M365 Copilot][docs-mcp] guidance throughout.

[docs-mcp]: https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/build-mcp-plugins

### ⭐ Bonus 4 — External MCP server integration (Optional)

**Evidence**:
- Integrated with `github/github-mcp-server` running on GitHub's hosted
  endpoint `https://api.githubcopilot.com/mcp/`.
- Verified live with `curl` — see captured handshake transcript below.
- The MCP server is fully external (not built by us, not hosted by us);
  this proves the agent can compose with arbitrary third-party MCP
  servers, not just a custom one we wrote.

**Read operations exercised**: `get_me`, `search_pull_requests`,
`list_pull_requests`, `pull_request_read`, `search_issues`, `list_issues`,
`issue_read`, `list_commits`, `get_commit`, `search_commits`,
`search_code`, `search_repositories`, `search_users`, `get_file_contents`.

**Write operations**: deliberately excluded in v1 to minimise the trust
boundary. Adding them (e.g. `create_pull_request`, `add_issue_comment`)
is a one-line change to `mcp-tools.json` and `plugin-manifest.json`.

**Verified handshake** (run from the repo author's machine):

```
$ curl -X POST https://api.githubcopilot.com/mcp/
HTTP/2 401
www-authenticate: Bearer error="invalid_request",
  resource_metadata="https://api.githubcopilot.com/.well-known/oauth-protected-resource/mcp/"

$ curl -X POST https://api.githubcopilot.com/mcp/ \
    -H "Authorization: Bearer <user token>" \
    -d '{"jsonrpc":"2.0","id":1,"method":"initialize",...}'
HTTP/2 200
mcp-session-id: 26b6651f-74bf-40e0-b984-3667030d056f
{"result":{"serverInfo":{"name":"github-mcp-server",
  "version":"github-mcp-server/remote-3ae183d1ec75a2bc1ce714aca999ec13d237e771",...}}}
```

This is the exact MCP-spec OAuth-protected-resource discovery handshake.

### ⭐ Bonus 5 — OAuth Security for the MCP server (Optional)

**Evidence**:
- The plugin manifest declares
  `authentication.type = "OAuthPluginVault"` with a `reference_id`
  pointing at the M365 Plugin Vault entry created during Provision.
- This is Microsoft's recommended pattern: client_id and client_secret
  never appear in the repository or the shipped manifest. They live in
  the tenant-local vault, set once during the Agents Toolkit Provision
  step.
- Per-user tokens are minted at first chat via the GitHub OAuth
  Authorization Code flow with PKCE (Microsoft owns the redirect at
  `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`).
- Token refresh is handled by the Microsoft 365 plugin runtime — we do
  not implement it ourselves.
- Error handling: if `get_me` returns 401, `instructions.md` directs the
  agent to ask the user to re-authorize rather than retry blindly.

**Scope minimisation**: the OAuth App requests **only** `read:user` and
`repo` — no admin, no write_packages, no delete — matching the read-only
tool surface.

**Trust boundary diagram**: see
[`docs/architecture.md`](docs/architecture.md) "OAuth handshake" sequence.

---

## Why this submission is non-trivial despite being declarative

| Concern                                         | Decision                                                                        |
| ----------------------------------------------- | ------------------------------------------------------------------------------- |
| **Multi-tenant identity** without leaking data | OAuth per-user — every call runs as the user; no tenant config to misconfigure  |
| **Hallucination risk** of LLM-rendered tables  | Strict templates in `instructions.md` + "never invent data" rule + URL citation  |
| **Manager outputs being weaponised**           | Mandatory disclaimer framing as *workflow signals*, not performance evaluation  |
| **Privacy of "show me alice's PRs"**           | Explicit rule: only fetch what the OAuth scope returns; 403 → say so plainly    |
| **Tool surface drift** as GitHub updates MCP   | `mcp-tools.json` is the manifest's contract — refresh via `tools/list` curl     |
| **No code = less control**                     | Trade-off accepted; in exchange we get ~zero infra, zero hosting cost, zero PEM |

---

## Files reviewers should look at, in order

1. [`README.md`](README.md) — the elevator pitch + sideload steps.
2. [`docs/architecture.md`](docs/architecture.md) — diagrams.
3. [`appPackage/instructions.md`](appPackage/instructions.md) — the brain.
4. [`appPackage/plugin-manifest.json`](appPackage/plugin-manifest.json) — the MCP + OAuth declaration.
5. [`docs/work-iq-integration.md`](docs/work-iq-integration.md) — R2 mapping.
