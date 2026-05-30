# Demo Video Script — EngInsights Agent (5:00)

Target length **5:00**, hard cap **5:30**. Recorded in 1080p, agent rail
fully visible, browser zoom 110%.

---

## Beat 1 — Hook (0:00 – 0:25)

**Visual**: Title card → split-screen showing two file trees side-by-side:
old `enginsights-copilot` repo (~80 files, mostly Python) on the left,
new `enginsights-agent` repo (7 files in `appPackage/` + 4 docs) on the right.

**Voice over**:
> "Two months ago I built an engineering-intelligence agent for Microsoft
> 365 Copilot Chat. It worked — but it took 2,800 lines of FastAPI, an
> Azure App Service, a GitHub App with a rotating PEM, and a custom
> OpenAPI plugin. Today I rebuilt the same product with zero hosted code
> — just a declarative agent talking to GitHub's hosted MCP server over
> per-user OAuth. Here's how, in five minutes."

---

## Beat 2 — Show the receipts (0:25 – 1:10)

**Visual**: VS Code with `appPackage/` open. Quickly cycle through:
- `manifest.json` (highlight `copilotAgents.declarativeAgents`)
- `declarativeAgent.json` (highlight `actions[0].file`)
- `plugin-manifest.json` (highlight `runtime.type: "RemoteMCPServer"` and
  `auth.type: "OAuthPluginVault"`)
- `instructions.md` (scroll the table of contents)

**Voice over**:
> "The whole agent is four JSON files plus one markdown prompt. The
> Teams manifest registers a declarative agent. The declarative agent
> points at a plugin manifest. The plugin manifest says: 'When the user
> asks something, call GitHub's hosted MCP server, authenticated with
> OAuth pulled from Microsoft's plugin vault.' That's the entire stack."

---

## Beat 3 — Sideload (1:10 – 1:40)

**Visual**: Switch to https://m365.cloud.microsoft/chat. Agents → Add an
agent → Upload → drop `appPackage/build/appPackage.zip`. Agent rail
shows **EngInsights Agent**.

**Voice over**:
> "I sideload the package once. The first time I open a chat, Copilot
> pops a 'Sign in to GitHub' card. This is the OAuth handshake doing
> exactly what the MCP spec defines — the server returned a 401 with a
> WWW-Authenticate header pointing at GitHub's authorization server, and
> the plugin runtime followed the trail."

---

## Beat 4 — OAuth handshake (1:40 – 2:10)

**Visual**: Click the "Sign in" card. GitHub OAuth consent page appears
showing the scopes `read:user` + `repo`. Click Authorize. Returned to
Copilot Chat.

**Voice over**:
> "I'm granting my own GitHub identity, with read-only repo scope. The
> consent is recorded on github.com under my OAuth App authorizations,
> and my token lives in the M365 Plugin Vault for this tenant — not in
> a database I have to operate, not in a key vault I have to rotate."

---

## Beat 5 — Developer query (2:10 – 3:00)

**Visual**: Type "Give me my daily engineering briefing." into the chat.
Watch the agent stream:
- 📌 Open PRs section with idle days + GitHub URLs
- ⚠️ Stuck PRs section flagging anything > 7 days
- 🔍 Reviewed this week count
- 👉 "Your single most important next action: ..."

**Voice over**:
> "First query — daily briefing. Under the hood, the agent called
> `get_me` once to identify me, then `search_pull_requests` with
> 'author:@me state:open', then again with 'reviewed-by:@me updated:>'
> seven days ago. Three MCP calls. The bullets, the idle calculation,
> the 'next action' synthesis — all of that is in instructions.md as
> a *prompt*. No Python."

---

## Beat 6 — Drill-down + memory (3:00 – 3:40)

**Visual**: Type a follow-up: "Why is the first one stuck?" Agent calls
`pull_request_read`, returns 2-3 sentences citing the specific review
state, mergeable status, and suggested action.

**Voice over**:
> "Follow-up question — note I didn't re-state my GitHub login. The
> orchestrator remembered. That's Work IQ retaining intra-session
> context for me. The agent calls `pull_request_read` on the specific
> PR, looks at reviews and merge state, and gives me one concrete next
> action — 'ping reviewer X' or 'resolve the conflict on file Y'."

---

## Beat 7 — Discovery (3:40 – 4:10)

**Visual**: Type "Find me the official repository for kiota." Agent
returns `microsoft/kiota` with description, stars, language, URL.

**Voice over**:
> "And the agent works as a general-purpose GitHub assistant too. 'Find
> me kiota.' That's `search_repositories` under the hood. No team-config
> setup needed — every user just signs in with their own identity, and
> they're off."

---

## Beat 8 — Hackathon scoring summary (4:10 – 4:50)

**Visual**: Cut to `SUBMISSION.md` rubric table on screen. Highlight each row.

**Voice over**:
> "R1 — hosted in M365 Copilot Chat: yes, sideloaded. R2 — Microsoft IQ:
> Work IQ via the canonical Skills-and-Tools extension pattern. Bonus 3
> — MCP Apps: this *is* a packaged MCP App, using Microsoft's v2.4
> plugin schema with the RemoteMCPServer runtime. Bonus 4 — external MCP
> server: GitHub's hosted one. Bonus 5 — OAuth on MCP: per-user GitHub
> OAuth 2.0 with the vault holding the client secret. Five for five."

---

## Beat 9 — Why less is more (4:50 – 5:00)

**Visual**: Hero image — the architecture diagram from README.md. Title
card: "EngInsights Agent · github.com/zunyangc/enginsights-agent · MIT".

**Voice over**:
> "Less code, less infra, less risk, more leverage. Thanks for watching."

---

## Cheatsheet — exact prompts to type, in order

1. _(After sideload, before first message)_ Click the "Sign in to GitHub" card.
2. `Give me my daily engineering briefing.`
3. `Why is the first one stuck?`
4. `Find me the official repository for kiota.`

## Cheatsheet — exact files to open in VS Code

1. `appPackage/manifest.json` — scroll to `copilotAgents`.
2. `appPackage/declarativeAgent.json` — scroll to `actions`.
3. `appPackage/plugin-manifest.json` — scroll to `runtimes[0]`.
4. `appPackage/instructions.md` — scroll table of contents.
5. `SUBMISSION.md` — scroll to the rubric table.

## Recording tips

- Use OBS or QuickTime; **disable webcam overlay** for the demo (judges
  want to see the agent, not your face).
- Mute notifications, close all browser tabs except Copilot Chat and one
  VS Code window.
- Run a dry-run end-to-end **once** before recording — token refresh is
  silent, but the first OAuth call adds 5-10 seconds.
- Keep the cursor visible; if the agent streams slowly, pause at each
  section header rather than reading along.
