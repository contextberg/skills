# Contextberg Skills

> **NEW · MCP support for Claude Code, Cursor, and OpenClaw.**
> Every context, connected. Every agent, empowered.

This repository is the **official skill / rules / instructions distribution** for [Contextberg](https://contextberg.com) — drop it into your project and your AI coding agent will know how to read your activity history, daily reports, and Long-Term Memory through the Contextberg MCP server.

---

## What is Contextberg?

Contextberg is a **local memory app for your AI agents**. It watches your screens, agent transcripts, and browser in the background — so Claude Code, Cursor, OpenClaw, and other agents can just remember.

| Pillar | What it does |
|---|---|
| **01 · Context** | Records screens, inputs, browser activity, and agent transcripts — delivered to your agent via MCP. No config files, just connect. |
| **02 · Memory** | Distills your daily work into three layers — *activity* (raw timeline), *daily* (per-day digest), and *long-term* (your tools and patterns). |
| **03 · Local** | Runs entirely on your machine. Pair with LM Studio for a fully offline pipeline — no data ever leaves your PC. |
| **04 · Remember** | Auto-summarizes what you were doing before you stepped away, so you pick up right where you left off. Then dig deeper in chat. |

[**→ Download for Windows**](https://contextberg.com)

---

## What is this repo?

The Contextberg desktop app exposes its memory via a local **MCP server** at `http://localhost:18080`. Agents *can* talk to it directly, but they work much better when they know **which tool to call when** — `get_activity` first, then drill into `get_daily_memory`, `get_weekly_memory`, `get_agent_history`, `read_ltm`, or `update_ltm`.

This repo packages that guidance as a **skill** in every popular agent format, so any of these agents can use Contextberg with a single drop-in. Two layouts are provided side by side:

**1. Unified `<tool>/skills/contextberg/SKILL.md` layout** (drop-in for tools that follow the new convention):

| Agent | Path |
|---|---|
| Generic / `.agent` aware | [`.agent/skills/contextberg/SKILL.md`](.agent/skills/contextberg/SKILL.md) |
| Generic / `.agents` aware | [`.agents/skills/contextberg/SKILL.md`](.agents/skills/contextberg/SKILL.md) |
| Claude Code | [`.claude/skills/contextberg/SKILL.md`](.claude/skills/contextberg/SKILL.md) |
| Cline | [`.cline/skills/contextberg/SKILL.md`](.cline/skills/contextberg/SKILL.md) |
| Codex | [`.codex/skills/contextberg/SKILL.md`](.codex/skills/contextberg/SKILL.md) |
| Cursor | [`.cursor/skills/contextberg/SKILL.md`](.cursor/skills/contextberg/SKILL.md) |
| Gemini CLI | [`.gemini/skills/contextberg/SKILL.md`](.gemini/skills/contextberg/SKILL.md) |
| GitHub (Copilot / Models) | [`.github/skills/contextberg/SKILL.md`](.github/skills/contextberg/SKILL.md) |
| OpenCode | [`.opencode/skills/contextberg/SKILL.md`](.opencode/skills/contextberg/SKILL.md) |

**2. Native per-tool rule formats** (for tools that read their own rule file shape):

| Agent | Path |
|---|---|
| Cursor (MDC rules) | [`.cursor/rules/contextberg.mdc`](.cursor/rules/contextberg.mdc) |
| GitHub Copilot (instructions) | [`.github/copilot-instructions.md`](.github/copilot-instructions.md) |
| Windsurf (rules) | [`.windsurf/rules/contextberg.md`](.windsurf/rules/contextberg.md) |
| Cline (legacy rules) | [`.clinerules/contextberg.md`](.clinerules/contextberg.md) |
| Continue (rules) | [`.continue/rules/contextberg.md`](.continue/rules/contextberg.md) |
| OpenClaw / Aider / any AGENTS.md-aware agent | [`AGENTS.md`](AGENTS.md) |

All formats carry the same canonical content (kept in sync) — pick the one(s) your agent reads.

---

## Install

### Option A — Copy just the format you need

Copy the file or directory for your agent (see table above) into your project root. That's it.

### Option B — Clone alongside your project

```bash
git clone https://github.com/contextberg/contextberg-skills .contextberg-skills
# then symlink or copy the formats you want into your project
```

### MCP server setup

The skill assumes the Contextberg desktop app is running locally. Add the MCP server to your agent's config. Example (Claude Code, `.claude/settings.local.json`):

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

For Cursor, OpenClaw, and other MCP-aware agents, point them at the same `npx @contextberg/mcp-server` command — exact location depends on the agent.

---

## What the skill teaches your agent

```
get_activity          ← always start here (full daily picture, includes agent sessions)
  ├── get_daily_memory      → AI-written daily summary
  ├── get_weekly_memory     → multi-day patterns / trends
  ├── get_agent_history     → all turns of one Claude Code / Cursor / OpenClaw session
  └── read_ltm / update_ltm → who the user is — preferred stack, work style, current priorities
```

`get_activity` returns a single consolidated view: app timeline, keystrokes per region (editor / terminal / AI sidebar), clipboard history, browser visits, and agent session summaries. Most "what was I doing?" questions can be answered from this one call — the rest of the tools exist for AI summaries, multi-day trends, full-turn deep dives, and the persistent user profile.

---

## Contributing

Open an issue or PR if your agent uses a format that isn't listed here. Keep all formats in sync — the canonical text lives in [`AGENTS.md`](AGENTS.md); changes there should be mirrored to the per-agent files.

## License

MIT
