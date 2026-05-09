---
trigger: model_decision
description: Use Contextberg's stored information — activity history, AI-generated reports, and Long-Term Memory — via MCP. Invoke when the user asks about past work, wants context before starting a task, or needs to update their LTM. Requires the Contextberg desktop app running on localhost:18080.
---

# Contextberg

This rule connects Windsurf to the Contextberg MCP server to retrieve and update information that Contextberg has collected and generated on the user's local machine.

## Start here: `get_activity`

**Always call `get_activity` first.** It is the single consolidated view of everything Contextberg recorded for a given day — app usage timeline, keystrokes per region (editor / terminal / AI sidebar), clipboard history, browser visits, and **AI coding agent session history (Claude Code / Cursor / OpenClaw)**. All in one response.

```jsonc
{ "date": "2026-04-26" }
{ "date": "2026-04-26", "app_filter": "Code" }
```

| Question | Where to look |
|----------|---------------|
| "What was I working on?" | App timeline |
| "How long did I spend coding?" | Editor-region keystroke counts |
| "What did I copy/paste?" | Clipboard history section |
| "What sites did I visit?" | Browser visits section |
| "What did I do with my AI coding agent?" | AI Coding Agent Sessions section |

## Drilling deeper

- `get_daily_memory` — AI-written daily digest. Use when raw timeline is too dense.
- `get_weekly_memory` — multi-day trends across up to 30 days.
- `get_agent_history` — all turns of a specific Claude Code / Cursor / OpenClaw session. Filter by `project`.
- `read_ltm` — persistent user profile. Call before recommendations. Sections: `Work Style`, `Toolbox & Stack`, `Expertise`, `Communication`, `Interests`, `Current Context`.
- `update_ltm` — persist a significant decision or new priority so it survives across sessions.

```
get_activity          ← always start here
  ├── get_daily_memory      → AI summaries instead of raw data
  ├── get_weekly_memory     → multi-day trends
  ├── get_agent_history     → all turns of a specific session
  └── read_ltm / update_ltm → persistent user profile & priorities
```

## Setup

The Contextberg desktop app must be running. Add the MCP server in Windsurf's MCP settings to launch `npx -y @contextberg/mcp-server`.
