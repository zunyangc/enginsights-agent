# Work IQ Integration

> A short doc judges can read in 60 seconds, mapping how this agent
> satisfies hackathon **Requirement R2 — Microsoft IQ integration**.

## What Work IQ is, in one paragraph

**Work IQ** is Microsoft's intelligence layer behind Microsoft 365
Copilot. It builds memory from the user's working context — emails,
meetings, chats, documents, org graph — and exposes that context to
agents through the official **Skills + Tools** extensibility surface
(declarative agents, plugins, actions, and capabilities like `People`).
When a declarative agent reads `instruction.txt`, resolves a colleague's
identity via the People capability, selects the right tool to call,
formats a response, and remembers the user's prior turns in the same
chat, that orchestration is Work IQ doing its job.

## How this repo uses Work IQ

This repo is a canonical Work IQ consumer. It uses the documented
Skills + Tools + People extension pattern end-to-end:

| Work IQ capability                        | Where it shows up in this repo                                                                                       |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Conversation orchestration**            | Copilot Chat reads `declarativeAgent.json`, treats `instruction.txt` as the system prompt for every user turn         |
| **Tool selection (Skills)**               | The orchestrator picks one of 14 MCP tools declared in `ai-plugin.json` based on `description_for_model`             |
| **Tool invocation (Tools / MCP)**         | Plugin runtime opens a session against the external MCP server and calls `tools/call` on the user's behalf           |
| **People knowledge**                      | `declarativeAgent.json` declares `"capabilities": [{ "name": "People" }]` — manager / reports / peers resolve at runtime |
| **Per-user identity + auth state**        | M365 Plugin Vault stores the user's GitHub OAuth token; Work IQ injects `Authorization: Bearer` on every call         |
| **Conversational memory within a turn**   | The orchestrator chains multiple tool calls inside one user turn (`get_me` → `search_pull_requests` → render)         |
| **Conversational memory across turns**    | The agent remembers the user's GitHub `login` and persona for the rest of the chat without recomputing                |
| **Grounded answer rendering**             | Work IQ produces the final markdown using the tool results plus the formatting templates in `instruction.txt`         |

## Why this is "Work IQ via Skills + Tools" specifically

Microsoft's [IQ Series page][iq-series] lists three options for IQ
integration:

1. **Foundry IQ** — agentic retrieval over enterprise sources.
2. **Work IQ** — the intelligence layer behind M365 Copilot, **extended
   with Skills and Tools**.
3. **Fabric IQ** — semantic ontologies over Fabric data.

This agent is a Work IQ extension. The "Skills" part is the declarative
agent itself (a packaged skill in the Copilot agent list). The "Tools"
part is the MCP plugin (the actions surface). The `People` capability is
Work IQ contributing the org-graph layer. Together they form the
**canonical Microsoft-documented path** for extending Work IQ with new
domain intelligence — exactly what the hackathon asks for.

[iq-series]: https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/iq-series

## Evidence judges can verify

1. The agent loads as a sideloaded entry in
   https://m365.cloud.microsoft/chat (R1 — M365 Copilot Chat).
2. `appPackage/declarativeAgent.json` follows declarative-agent schema
   **v1.7** with `capabilities: [{ "name": "People" }]`.
3. `appPackage/ai-plugin.json` follows plugin-manifest schema **v2.4**.
4. Tools fire against `https://api.githubcopilot.com/mcp/` — visible in
   the demo recording's OAuth handshake.
5. Conversational memory is observable in the demo: the agent answers
   "why is the first one stuck?" without re-asking the user's GitHub
   login. That's Work IQ retaining intra-session context.
6. People-knowledge fan-out is observable in the "Anyone on my team
   need help?" demo: the agent never asks who the team is, it asks Work
   IQ.
