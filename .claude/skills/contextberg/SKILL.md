---
name: contextberg
description: >
  Use Contextberg's stored information — activity history, AI-generated reports, and Long-Term Memory — via MCP.
  Invoke when the user asks about past work, wants context before starting a task, or needs to update their LTM.
  Requires the Contextberg desktop app to be running (localhost:18080).
---

# contextberg

This skill connects to the Contextberg MCP server to retrieve and update information that Contextberg has collected and generated on the user's local machine.

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

### `get_daily_memory` — AI-interpreted summaries

Use when the raw timeline from `get_activity` is too dense and you want Contextberg's own AI analysis — topic extraction, hourly summaries, and a daily digest already written in natural language.

> "What were the main things I accomplished today?"

`get_activity` gives you the raw evidence. `get_daily_memory` gives you the interpretation.

### `get_weekly_memory` — multi-day patterns

Use when the question spans more than one day. `get_activity` is per-date only; `get_weekly_memory` aggregates daily reports, top apps, and trends across up to 30 days.

> "How was my week? What did I spend most time on?"

### `get_agent_history` — full turn-by-turn deep dive

`get_activity` includes a summary of each agent session (first few turns). `get_agent_history` retrieves **all turns** for a specific session with full `max_turns` / `max_chars` control. Use it only when you need the complete conversation detail.

> "Pull every decision made in the screenest project yesterday — all turns."

Filter by `project` to stay focused. **For a quick briefing, `get_activity` is sufficient.**

Supported sources: Claude Code, Cursor, OpenClaw.

### `read_ltm` — persistent user context

Use before making recommendations. `get_activity` shows what the user did today; `read_ltm` shows who they are — their preferred stack, expertise level, work style, and current priorities.

> Before suggesting a library, call `read_ltm` with `section: "Toolbox & Stack"` to recommend within their existing ecosystem.

Sections: `Work Style`, `Toolbox & Stack`, `Expertise`, `Communication`, `Interests`, `Current Context`.

### `update_ltm` — persisting decisions

When the user makes a significant decision or states a new priority, write it to LTM so it survives across sessions.

```jsonc
{
  "section": "Current Context",
  "content": "Decided to keep existing SQLite schema — migration complexity outweighs benefits. [Priority: High] [Weight: 4]",
  "replace": false
}
```

## Tool hierarchy at a glance

```
get_activity          ← always start here (full daily picture, includes agent sessions)
  ├── get_daily_memory      → need AI summaries instead of raw data
  ├── get_weekly_memory     → need multi-day trends
  ├── get_agent_history     → need ALL turns for a specific session (deep dive)
  └── read_ltm / update_ltm → need persistent user profile & priorities
```

## LTM sections

| Section | What it contains |
|---------|-----------------|
| `Work Style` | How the user prefers to work, pace, habits |
| `Toolbox & Stack` | Languages, frameworks, tools in active use |
| `Expertise` | Domain knowledge and skill levels |
| `Communication` | Preferred interaction style with AI assistants |
| `Interests` | Side interests and learning goals |
| `Current Context` | Active projects, decisions, priorities — changes most often |

## Setup

The Contextberg desktop app must be running. Add to `.claude/settings.local.json`:

```json
{
  "mcpServers": {
    "contextberg": {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "@contextberg/mcp-server"]
    }
  }
}
```
