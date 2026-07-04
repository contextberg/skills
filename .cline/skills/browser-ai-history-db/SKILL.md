---
name: browser-ai-history-db
description: Locate, inspect, and query ContextBerg Browser AI History SQLite archives. Use when an agent needs to find captured browser AI conversations, understand how conversations/messages/session indexes are stored in SQLite, run SQL searches, inspect generated Markdown/JSON/AgentHistory exports, or troubleshoot the local bridge archive.
---

# Browser AI History DB

Use this skill to work with the local archive created by ContextBerg Browser AI History. Treat the Chrome extension as the capture surface. Treat SQLite and generated files as the durable archive.

## Locate The Archive

Default Windows archive:

```powershell
Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history"
```

Override archive for a shell session:

```powershell
$env:CONTEXTBERG_BROWSER_AI_DIR="C:\path\to\browser-ai-history"
```

Demo archive:

```powershell
C:\Users\mochi\trackq-dev\contextberg-chrome\.tmp\demo-browser-ai-history
```

Important files:

- `browser-ai-history.sqlite`: primary SQLite database.
- `browser-ai-conversations.md`: generated Markdown view for reading/search snippets.
- `browser-ai-conversations.json`: generated JSON bundle.
- `browser-ai-conversations.jsonl`: generated JSONL records.
- `browser-ai-agent-history.json`: ContextBerg agent-history compatible generated bundle.
- `conversations\*.json`: raw per-conversation JSON captures.

## SQLite Schema

Primary tables:

- `conversations`: one row per captured conversation.
- `messages`: one row per message, ordered by `order_index`.
- `conversation_meta`: user-editable project/account/labels/favorite/notes.
- `session_index`: provider sidebar/session links, including sessions that may not be fully captured.

Core columns:

```sql
conversations(
  id TEXT PRIMARY KEY,
  schema TEXT,
  provider TEXT,
  source TEXT,
  capture_method TEXT,
  title TEXT,
  url TEXT,
  conversation_id TEXT,
  captured_at TEXT,
  updated_at TEXT,
  imported_at TEXT,
  page_language TEXT,
  message_count INTEGER,
  raw_json TEXT
);

messages(
  conversation_id TEXT,
  order_index INTEGER,
  message_id TEXT,
  role TEXT,
  body TEXT,
  markdown TEXT,
  created_at TEXT,
  raw_json TEXT
);

conversation_meta(
  conversation_id TEXT PRIMARY KEY,
  project TEXT,
  account_id TEXT,
  account_label TEXT,
  labels_json TEXT,
  is_favorite INTEGER,
  notes TEXT,
  updated_at TEXT
);

session_index(
  id TEXT PRIMARY KEY,
  provider TEXT,
  title TEXT,
  url TEXT,
  conversation_id TEXT,
  first_seen_at TEXT,
  last_seen_at TEXT,
  capture_count INTEGER,
  raw_json TEXT
);
```

## SQL Examples

Start with diagnostics and FTS availability:

```sql
SELECT
  (SELECT COUNT(*) FROM conversations) AS conversation_count,
  (SELECT COUNT(*) FROM messages) AS message_count,
  (SELECT COUNT(*) FROM session_index) AS indexed_session_count,
  (SELECT name FROM sqlite_master WHERE type = 'table' AND name = 'message_search') AS fts_table;
```

Provider counts:

```sql
SELECT provider, COUNT(*) AS count
FROM conversations
GROUP BY provider
ORDER BY provider;
```

Recent conversations with user metadata:

```sql
SELECT c.provider, c.title, c.updated_at, c.message_count,
       meta.project, meta.account_label, meta.labels_json, meta.is_favorite
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

Search message bodies:

```sql
SELECT c.provider, c.title, s.role, substr(s.body, 1, 240) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"pricing"'
ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC
LIMIT 20;
```

Fallback search when FTS5 is unavailable or a query cannot be represented as FTS:

```sql
SELECT c.provider, c.title, m.role, substr(m.body, 1, 240) AS snippet
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE m.body LIKE '%pricing%' ESCAPE '\'
   OR m.markdown LIKE '%pricing%' ESCAPE '\'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 20;
```

Search titles, URLs, tags, project labels, and message bodies together:

```sql
SELECT c.id, c.provider, c.title, c.url, c.updated_at, c.message_count,
       meta.project, meta.account_label, meta.labels_json
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE c.title LIKE '%pricing%' ESCAPE '\'
   OR c.url LIKE '%pricing%' ESCAPE '\'
   OR meta.project LIKE '%pricing%' ESCAPE '\'
   OR meta.account_label LIKE '%pricing%' ESCAPE '\'
   OR meta.labels_json LIKE '%pricing%' ESCAPE '\'
   OR EXISTS (
     SELECT 1
     FROM message_search
     WHERE message_search.conversation_id = c.id
       AND message_search MATCH '"pricing"'
   )
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

Read one full conversation as message rows:

```sql
SELECT m.order_index, m.role, m.markdown
FROM messages m
WHERE m.conversation_id = 'chatgpt:demo-chatgpt-pricing'
ORDER BY m.order_index ASC;
```

Find indexed sessions that were seen but may not have full messages:

```sql
SELECT provider, title, url, last_seen_at, capture_count
FROM session_index
ORDER BY datetime(last_seen_at) DESC
LIMIT 20;
```

Favorite or labeled work:

```sql
SELECT c.provider, c.title, meta.project, meta.labels_json
FROM conversations c
JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE meta.is_favorite = 1
   OR lower(meta.labels_json) LIKE lower('%pricing%')
ORDER BY datetime(c.updated_at) DESC;
```

## Run Queries From PowerShell

Use Node's built-in SQLite when available:

```powershell
$dbPath = Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history\browser-ai-history.sqlite"
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1]); console.log(db.prepare('SELECT provider, COUNT(*) AS count FROM conversations GROUP BY provider ORDER BY provider').all())" $dbPath
```

Search with FTS5 from PowerShell:

```powershell
$dbPath = Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history\browser-ai-history.sqlite"
$term = '"pricing"'
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1]); const rows = db.prepare('SELECT c.provider, c.title, s.role, substr(s.body, 1, 240) AS snippet FROM message_search s JOIN conversations c ON c.id = s.conversation_id WHERE message_search MATCH ? ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC LIMIT 20').all(process.argv[2]); console.log(JSON.stringify(rows, null, 2));" $dbPath $term
```

For the demo archive:

```powershell
$dbPath = "C:\Users\mochi\trackq-dev\contextberg-chrome\.tmp\demo-browser-ai-history\browser-ai-history.sqlite"
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1]); console.log(db.prepare('SELECT title, provider, message_count FROM conversations ORDER BY updated_at DESC LIMIT 10').all())" $dbPath
```

## Local Bridge APIs

When the local bridge is running:

```text
http://127.0.0.1:18765/
http://127.0.0.1:18765/health
http://127.0.0.1:18765/api/browser-ai-diagnostics
http://127.0.0.1:18765/api/browser-ai-conversations?compact=1
http://127.0.0.1:18765/api/browser-ai-sessions
http://127.0.0.1:18765/api/export?format=contextberg&days=30&mode=per-conversation
```

Use another port only when `CONTEXTBERG_BROWSER_AI_PORT` was set.

## Troubleshooting

1. Confirm the archive path first. If `CONTEXTBERG_BROWSER_AI_DIR` is set, it overrides the default.
2. Check `/health` and `/api/browser-ai-diagnostics` before inspecting raw files.
3. If `session_index` has rows but `conversations` does not, the provider sidebar was indexed but sessions were not opened/captured yet.
4. Prefer `message_search` FTS5 for broad text search. Fall back to `messages.body LIKE ...` only if FTS5 is missing or the query fails.
5. If `messages.markdown` is plain text, inspect `messages.raw_json` and provider extraction logic.
6. Prefer SQLite for counts/filtering. Prefer Markdown/JSON files for human-readable review and import checks.
