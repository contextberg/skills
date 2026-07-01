---
name: contextberg-chrome-dev
description: Develop, debug, and validate the ContextBerg Browser AI History Chrome extension and local bridge. Use when working in contextberg-chrome on manifest, side panel, popup, content scripts, background service worker, local bridge, export UI, SQLite storage, provider import behavior, or extension E2E checks.
---

# ContextBerg Chrome Dev

Use this skill when changing `contextberg-chrome`, the Chrome extension that captures browser AI chats into a local SQLite archive and exposes the archive through the Chrome side panel and local bridge.

## Repository Map

- `manifest.json`: MV3 permissions, side panel declaration, extension surfaces, and content script registration.
- `src/background.js`: service worker, local bridge posting, imports, side panel routing, and extension commands.
- `src/content/content.js`: provider DOM capture, visible-history import, auto-save, and the in-page ContextBerg ribbon.
- `src/sidepanel/`: Chrome side panel archive UI.
- `src/popup/`: popup controls for opening the side panel, bridge, and export flows.
- `src/options/`: local settings such as bridge URL and default project/account metadata.
- `scripts/local-bridge.mjs`: local archive page and export modal served at `http://127.0.0.1:18765/`.
- `scripts/browser-ai-store.mjs`: SQLite schema, search, metadata updates, export formats, and generated archive views.
- `scripts/validate-extension.mjs`: manifest and file validation used by `npm test`.

## Local Paths

- Workspace: `C:\Users\mochi\trackq-dev\contextberg-chrome`
- Archive folder: `%LOCALAPPDATA%\ContextBerg\browser-ai-history`
- SQLite database: `%LOCALAPPDATA%\ContextBerg\browser-ai-history\browser-ai-history.sqlite`
- Local bridge: `http://127.0.0.1:18765/`
- Demo archive: `C:\Users\mochi\trackq-dev\contextberg-chrome\.tmp\demo-browser-ai-history`

## Development Workflow

1. Read the relevant extension and bridge files before editing.
2. Keep side panel and local bridge labels, tag editing, bookmarks, export wording, and archive card behavior consistent.
3. For capture/import behavior, inspect both `src/content/content.js` and `src/background.js`; the browser page usually extracts data while the service worker persists or forwards it.
4. For archive/search/export behavior, inspect `scripts/browser-ai-store.mjs` and `scripts/local-bridge.mjs` together.
5. Restart the local bridge after changing `scripts/local-bridge.mjs`; the running Node process will not pick up edits automatically.
6. Reload the Chrome extension at `chrome://extensions` and reload active AI provider tabs after changing extension code.

## Validation

Run the smallest relevant checks, then broaden when shared behavior changed.

```powershell
node --check src\background.js
node --check src\content\content.js
node --check src\sidepanel\sidepanel.js
node --check scripts\local-bridge.mjs
node --check scripts\browser-ai-store.mjs
npm test
npm run diagnose:chrome
```

For local bridge UI changes, start the bridge and verify the page manually or with browser automation:

```powershell
npm run bridge
```

If the port is already in use, stop only the existing ContextBerg bridge process:

```powershell
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 18765
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -match 'scripts/local-bridge\.mjs' }
```

## E2E Notes

- Side panel behavior depends on Chrome extension APIs, so test it in Chrome after reloading the unpacked extension.
- Local bridge pages can be tested through `http://127.0.0.1:18765/` without extension APIs.
- When direct extension automation is unavailable, use a browser test harness with mocked `chrome.*` APIs only for layout and DOM behavior, then verify the real side panel in Chrome.
- The side panel should keep archive search, bookmarked state, tag editing, selected conversation messages, and source-chat navigation aligned with the local bridge archive UI.
