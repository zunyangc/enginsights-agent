# Demo Video Script — EngInsights Agent (≈ 5:00)

Target length **5:00**, hard cap **5:30**. Recorded in 1080p, agent
visible, browser zoom 110%.

---

## Beat 1 — Hook (0:00 – 0:25)

**Visual**: Title card → split-screen showing two file trees side-by-side.
On the left, the predecessor `enginsights-copilot` repo (~80 files,
mostly Python). On the right, this `enginsights-agent` repo (8 files in
`appPackage/` + a handful of docs).

**Voice over**:
> "Two months ago I built an engineering-intelligence agent for Microsoft
> 365 Copilot Chat. It worked — but it took ~2,800 lines of FastAPI, an
> Azure App Service, a GitHub App with a rotating PEM, and a custom
> OpenAPI plugin. Today I rebuilt the same product with zero hosted code
> — a declarative agent talking to GitHub's hosted MCP server over
> per-user OAuth. Here's how, in five minutes."

---

## Beat 2 — Show the receipts (0:25 – 1:10)

**Visual**: VS Code with `appPackage/` open. Quickly cycle through:
- `manifest.json` (highlight `copilotAgents.declarativeAgents`)
- `declarativeAgent.json` (highlight `actions[0].file` and
  `capabilities[0].name = "People"`)
- `ai-plugin.json` (highlight `runtimes[0].spec.url` and
  `auth.type: "OAuthPluginVault"`)
- `instruction.txt` (scroll through the section headers)

**Voice over**:
> "The whole agent is five JSON files plus one text prompt. The Teams
> manifest registers a declarative agent. The declarative agent points
> at a plugin manifest and declares the People capability. The plugin
> manifest says 'call GitHub's hosted MCP server, authenticated with
> OAuth pulled from Microsoft's plugin vault.' That's the entire stack."

---

## Beat 3 — Sideload via Provision (1:10 – 1:40)

**Visual**: Microsoft 365 Agents Toolkit side panel → click **Provision**.
Watch the green checks land: `teamsApp/create`, `oauth/register`,
`teamsApp/zipAppPackage`, `teamsApp/validateAppPackage`,
`teamsApp/extendToM365`. Switch to https://m365.cloud.microsoft/chat.
The agent appears in the agent list.

**Voice over**:
> "One Provision step pushes everything. The Toolkit creates the Teams
> app, registers my GitHub OAuth credentials in the M365 Plugin Vault,
> packages the zip, validates it, and extends it into Copilot. No code
> deploys, no Azure resources, no container image."

---

## Beat 4 — OAuth handshake (1:40 – 2:10)

**Visual**: First message to the agent. Copilot pops the "Sign in to
GitHub" card. Click it → GitHub consent page showing scopes `read:user`
+ `repo`. Approve. Returned to Copilot Chat.

**Voice over**:
> "I'm granting my own GitHub identity, with read-only repo scope. The
> consent is recorded under my OAuth App authorisations on github.com,
> and my token lives in the M365 Plugin Vault for this tenant — not in
> a database I have to operate, not in a key vault I have to rotate."

---

## Beat 5 — Developer query (2:10 – 2:50)

**Visual**: Click the **Daily briefing** starter. Watch the agent stream:
- 📌 Open PRs with idle days + GitHub URLs
- ⚠️ Stuck PRs flagging anything > 7 days
- 🔍 Reviewed this week count
- 👉 "Your single most important next action: …"

**Voice over**:
> "Daily briefing. Under the hood the agent calls `get_me` to identify
> me, then `search_pull_requests` with `author:@me state:open`, then
> again with `reviewed-by:@me updated:>` seven days ago. Three MCP
> calls. The bullets, the idle calculation, the next-action synthesis
> — all of that is in `instruction.txt` as a prompt. No Python."

---

## Beat 6 — Manager fan-out (2:50 – 3:50)  ⭐ hero beat

**Visual**: Click the **Need help?** starter. The agent:
1. resolves direct reports via Work IQ's People capability,
2. fans out `search_pull_requests` per report,
3. renders per-engineer signals (stuck PRs, idle drafts, review
   starvation), each with a PR link.

**Voice over**:
> "Manager mode. The agent never asks me who's on my team — it asks
> Work IQ's People capability. Then it fans out across each report,
> applies the heuristics from `instruction.txt`, and gives me workflow
> signals — not performance evaluations. Note the closing line, it's a
> hard rule in the prompt."

---

## Beat 7 — Monthly digest + scheduling (3:50 – 4:30)

**Visual**: Click the **Monthly digest** starter. The agent returns a
per-engineer table with cycle-time / review-load / shipping themes.
Then open the agent's ⋮ menu → **Scheduled prompts** → show that the
same prompt can run on a daily/weekly/monthly schedule.

**Voice over**:
> "Monthly digest gives me a per-engineer table with cycle-time,
> review-load, and shipping trends. And because the agent is stateless,
> I can schedule this prompt — daily, weekly, monthly. Copilot runs it
> on the cadence and posts the result into chat. No cache file, no
> drift — GitHub stays the source of truth, the agent re-derives the
> answer each time."

---

## Beat 8 — Hackathon scoring summary (4:30 – 4:50)

**Visual**: Cut to `SUBMISSION.md` rubric table on screen. Highlight
each row briefly.

**Voice over**:
> "R1 — hosted in M365 Copilot Chat: yes. R2 — Microsoft IQ: Work IQ
> via Skills, Tools, and the People capability. Bonus 3 — MCP App
> packaged with the v2.4 plugin schema. Bonus 4 — external MCP server,
> GitHub's hosted one. Bonus 5 — OAuth on MCP, per-user GitHub OAuth
> 2.0 with the vault holding the client secret. Five for five."

---

## Beat 9 — Close (4:50 – 5:00)

**Visual**: Hero shot — the layered architecture diagram from
`docs/architecture.md`. Title card:
`github.com/zunyangc/enginsights-agent · MIT`.

**Voice over**:
> "Less code, less infrastructure, less risk, more leverage. Thanks for
> watching."

---

## Cheatsheet — exact starters to click, in order

1. _(After Provision, before first message)_ Click the "Sign in to GitHub" card.
2. **Daily briefing**
3. **Need help?**
4. **Monthly digest**

## Cheatsheet — exact files to open in VS Code

1. `appPackage/manifest.json` — scroll to `copilotAgents`.
2. `appPackage/declarativeAgent.json` — scroll to `actions` and `capabilities`.
3. `appPackage/ai-plugin.json` — scroll to `runtimes[0]` and `auth`.
4. `appPackage/instruction.txt` — scroll the section headers.
5. `SUBMISSION.md` — scroll to the rubric table.

## Recording tips

- OBS or QuickTime; disable the webcam overlay (judges want to see the agent).
- Mute notifications. Close all browser tabs except Copilot Chat + one VS Code window.
- Run a dry pass end-to-end **once** before recording — the first OAuth
  call adds 5–10 seconds, refresh after that is silent.
- Keep the cursor visible; if the agent streams slowly, pause at each
  section header rather than narrating along.
