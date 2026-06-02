# Security Policy

EngInsights Agent is a Microsoft 365 Declarative Agent — pure JSON manifests,
no hosted backend, no request-time code we own. This document explains the
trust boundaries, where secrets live, and how to report a security issue.

## Trust boundary

The agent runs inside Microsoft 365 Copilot Chat and calls **GitHub's hosted
Remote MCP Server** at `https://api.githubcopilot.com/mcp/` using the **end
user's own GitHub OAuth token**. Every call is performed as the signed-in
user.

This means:

- We cannot see anything the user cannot see on github.com.
- Multi-tenant data leakage is impossible by design — there is no shared
  service-account token that could be misused across users.
- Revoking access is a single click in the user's GitHub settings
  (https://github.com/settings/applications) — no support ticket required.

## OAuth scope minimisation

The bundled GitHub OAuth registration requests only:

- `read:user` — fetch the user's own profile (used by `get_me`).
- `repo` — read pull requests, issues, commits, code, and search results
  across repositories the user already has access to.

No admin, write, delete, packages, or workflow scopes are requested. The
agent's tool surface in [`appPackage/mcp-tools.json`](appPackage/mcp-tools.json)
is intentionally **14 read-only operations** — no PR comment, no merge, no
issue create, no file write. Adding writes is a deliberate, reviewable
change to two JSON files; this is not accidental.

## Where secrets live

| Secret                           | Location                                                                            |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| GitHub OAuth **client_id**       | M365 Plugin Vault (entered once during `Provision`)                                 |
| GitHub OAuth **client_secret**   | M365 Plugin Vault                                                                   |
| Per-user GitHub **access_token** | M365 Plugin Vault, keyed by `(tenant, user, plugin)`                                |
| GitHub OAuth **registration ID** | `env/.env.dev` (`MCP_DA_AUTH_ID_APIGITHUBC`) — a Microsoft-issued GUID, not a secret |

Nothing in the public repository contains a real secret. The `env/.env.dev`
template ships with empty values; the Toolkit `Provision` step writes
non-secret references at install time. Per-user state files
(`env/.env.*.user`) are `.gitignore`d.

## What we deliberately do NOT do

- **No service-account fallback.** If the user is not signed in to GitHub,
  the agent says so — it does not silently fall back to an admin token.
- **No persistent caching of GitHub data.** Every question re-derives the
  answer from a live GitHub call, so stale data cannot leak across users
  or survive a token revocation.
- **No proactive Teams channel posts.** Replies go only to the signed-in
  user's own Copilot Chat.
- **No agent-side LLM endpoint.** The orchestrator is Microsoft 365
  Copilot itself; we provide rules in `instruction.txt`, not a model.

## Manager-pattern privacy framing

When the agent answers manager questions (e.g. *"anyone on my team need
help?"*) it appends a disclaimer: *"Workflow signals only — not performance
evaluations. Use to start conversations, not grade people."* This is a soft
control encoded in [`appPackage/instruction.txt`](appPackage/instruction.txt)
§7.

People-graph data accessed via Work IQ's `People` capability is **org
metadata only** (names, roles, manager, reports). The agent does not read
email content, chat content, document content, or any other personal data
from Microsoft 365.

## Reporting a vulnerability

If you believe you have found a security issue in this repository (a
manifest misconfiguration that exposes data, a scope creep, a leaked
secret, etc.), please report it privately:

- Email **zunyang.chin@centific.com** with subject `[security] EngInsights Agent`, or
- Open a private security advisory at
  https://github.com/zunyangc/enginsights-agent/security/advisories/new

Please do not file a public issue for security reports. We aim to
acknowledge within 72 hours.

## Out of scope

- Vulnerabilities in Microsoft 365 Copilot, the M365 Plugin Vault, or the
  Microsoft Teams app platform — report through MSRC at
  https://msrc.microsoft.com/.
- Vulnerabilities in GitHub's hosted MCP server or the GitHub REST API —
  report through https://bounty.github.com/.
- Vulnerabilities in any third-party MCP server you may swap in instead of
  GitHub's.
