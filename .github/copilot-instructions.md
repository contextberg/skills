# Contextberg

Use Contextberg's stored information — activity history, AI-generated reports, and Long-Term Memory — via MCP. Invoke when the user asks about past work, wants context before starting a task, or needs to update their LTM. Requires the Contextberg desktop app running on `localhost:18080`.

## Start here: `get_activity`

**Always call `get_activity` first.** It is the single consolidated view of everything Contextberg recorded for a given day — app usage timeline, keystrokes per region (editor / terminal / AI sidebar), clipboard history, browser visits, and **AI coding agent session history (Claude Code / Cursor / OpenClaw)**. All in one response.

```jsonc
// Default: full activity for today
{ "date": "2026-04-26" }

// Narrow to one app if the response is too large
{ "date": "2026-04-26", "app_filter": "Code" }
```

From this single call you can answer most questions:

| Question | Where to look in the response |
|----------|-------------------------------|
| "What was I working on?" | App timeline — which apps were active and when |
| "How long did I spend coding?" | Keystroke counts in the editor region |
| "What did I copy/paste?" | Clipboard history section |
| "What sites did I visit?" | Browser visits section |
| "What did I work on with my AI coding agent?" | AI Coding Agent Sessions section |

**Only reach for other tools when you need something `get_activity` doesn't cover** — AI-generated summaries, multi-day trends, or Long-Term Memory.

## Drilling deeper: when to use other tools

- `get_daily_memory` — AI-interpreted daily summary (topics, hourly digest). Use when raw timeline is too dense.
- `get_weekly_memory` — multi-day trends across up to 30 days. Use when question spans more than one day.
- `get_agent_history` — full turn-by-turn deep dive for a specific Claude Code / Cursor / OpenClaw session. Filter by `project`.
- `read_ltm` — persistent user profile (Work Style, Toolbox & Stack, Expertise, Communication, Interests, Current Context). Call before making recommendations so suggestions match the user's existing stack.
- `update_ltm` — persist a significant decision or new priority so it survives across sessions.

## Tool hierarchy at a glance

```
get_activity          ← always start here (full daily picture, includes agent sessions)
  ├── get_daily_memory      → need AI summaries instead of raw data
  ├── get_weekly_memory     → need multi-day trends
  ├── get_agent_history     → need ALL turns for a specific session (deep dive)
  └── read_ltm / update_ltm → need persistent user profile & priorities
```

## Setup

The Contextberg desktop app must be running. Configure the MCP server in your Copilot / VS Code MCP settings to launch:

```
npx -y @contextberg/mcp-server
```
