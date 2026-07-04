---
name: contextberg-chrome-dev
description: Query and analyze the local ContextBerg Browser AI History SQLite archive created by the Chrome extension. Use when an agent needs to find saved ChatGPT, Claude, Gemini, Perplexity, or Manus conversations, search message text with FTS5, inspect conversation metadata/tags/favorites, compare providers, audit capture quality, or report Chrome extension capture issues discovered from the database.
---

# ContextBerg Browser AI History SQLite

Use this skill to read the local SQLite archive created by ContextBerg Browser AI History. The goal is not to modify the Chrome extension first. Start by querying the database, then answer from saved conversations and report any capture-quality problems that affect the analysis.

## Archive Paths

Default Windows archive:

```powershell
$archiveDir = Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history"
$dbPath = Join-Path $archiveDir "browser-ai-history.sqlite"
```

Demo archive:

```powershell
$dbPath = "C:\Users\mochi\trackq-dev\contextberg-chrome\.tmp\demo-browser-ai-history\browser-ai-history.sqlite"
```

If `CONTEXTBERG_BROWSER_AI_DIR` is set, it overrides the default archive folder.

Generated companion files:

- `browser-ai-history.sqlite`: primary database.
- `browser-ai-history.contextberg.sqlite`: mirror database for ContextBerg desktop integrations.
- `browser-ai-conversations.md`: readable generated view.
- `browser-ai-conversations.json`: full JSON export.
- `browser-ai-conversations.jsonl`: JSONL records.
- `browser-ai-contextberg.json`: ContextBerg-compatible generated bundle.
- `conversations\*.json`: raw per-conversation captures.

## Query Workflow

1. Confirm the archive exists and inspect row counts.
2. Confirm whether `message_search` exists. Prefer FTS5 when available.
3. Search broadly across messages, then open the few relevant conversations in message order.
4. Filter out obvious capture noise before drawing conclusions.
5. Cite conversation title, provider, message role, and snippet in the answer.
6. If useful results are missing because capture quality is poor, report Chrome extension feedback separately.

## Core Schema

```sql
conversations(
  id TEXT PRIMARY KEY,
  provider TEXT,
  title TEXT,
  url TEXT,
  conversation_id TEXT,
  capture_method TEXT,
  captured_at TEXT,
  updated_at TEXT,
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

When available, `message_search` is an FTS5 table over conversation title, URL, provider, message body, and markdown.

## First Diagnostics

Run this before searching:

```powershell
$dbPath = Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history\browser-ai-history.sqlite"
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1], { readOnly: true }); const row = db.prepare(`SELECT (SELECT COUNT(*) FROM conversations) AS conversations, (SELECT COUNT(*) FROM messages) AS messages, (SELECT COUNT(*) FROM session_index) AS sessions, (SELECT COUNT(*) FROM conversation_meta) AS meta, (SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'message_search') AS has_message_search`).get(); console.log(JSON.stringify(row, null, 2));" $dbPath
```

Provider counts:

```sql
SELECT provider, COUNT(*) AS count
FROM conversations
GROUP BY provider
ORDER BY count DESC, provider ASC;
```

Recent conversations:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count,
       meta.project, meta.labels_json, meta.is_favorite
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

## Search With FTS5

Use quoted tokens and `AND` for multi-word searches:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, s.role,
       substr(replace(s.body, char(10), ' '), 1, 240) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"SQLite" AND "export"'
ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC
LIMIT 20;
```

PowerShell version:

```powershell
$dbPath = Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history\browser-ai-history.sqlite"
$term = '"SQLite" AND "export"'
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1], { readOnly: true }); const rows = db.prepare('SELECT c.id, c.provider, c.title, c.updated_at AS updatedAt, s.role, substr(replace(s.body, char(10), '' ''), 1, 240) AS snippet FROM message_search s JOIN conversations c ON c.id = s.conversation_id WHERE message_search MATCH ? ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC LIMIT 20').all(process.argv[2]); console.log(JSON.stringify(rows, null, 2));" $dbPath $term
```

Fallback when FTS5 fails:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, m.role,
       substr(replace(m.body, char(10), ' '), 1, 240) AS snippet
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE m.body LIKE '%SQLite%' ESCAPE '\'
   OR m.markdown LIKE '%SQLite%' ESCAPE '\'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 20;
```

## Open A Conversation

After finding a relevant `conversation_id`, read it in order:

```sql
SELECT m.order_index, m.role, m.markdown
FROM messages m
WHERE m.conversation_id = 'chatgpt:example'
ORDER BY m.order_index ASC;
```

For compact review:

```sql
SELECT m.order_index, m.role,
       substr(replace(m.markdown, char(10), ' '), 1, 500) AS text
FROM messages m
WHERE m.conversation_id = 'chatgpt:example'
ORDER BY m.order_index ASC;
```

## Metadata, Tags, Favorites

Favorite or tagged conversations:

```sql
SELECT c.id, c.provider, c.title, meta.project, meta.account_label,
       meta.labels_json, meta.is_favorite, meta.notes
FROM conversations c
JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE meta.is_favorite = 1
   OR meta.labels_json <> '[]'
ORDER BY datetime(c.updated_at) DESC;
```

Search by project/tag plus message text:

```sql
SELECT DISTINCT c.id, c.provider, c.title, c.updated_at, meta.project, meta.labels_json
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE meta.project LIKE '%work%' ESCAPE '\'
   OR meta.labels_json LIKE '%research%' ESCAPE '\'
   OR EXISTS (
     SELECT 1
     FROM message_search
     WHERE message_search.conversation_id = c.id
       AND message_search MATCH '"pricing"'
   )
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

## Session Index

`session_index` contains seen provider sessions that may not have full messages yet. Use it to find links that need opening/capture.

```sql
SELECT provider, title, url, conversation_id, last_seen_at, capture_count
FROM session_index
WHERE lower(rtrim(url, '/')) NOT LIKE '%/new'
ORDER BY datetime(last_seen_at) DESC
LIMIT 50;
```

If `session_index` has relevant rows but `conversations` has no matching messages, report that the extension indexed the provider sidebar but has not captured the conversation body yet.

## Noise Filters

Real archives can include failed captures. Apply these filters before analysis when results look suspicious:

```sql
AND c.title <> 'Sign in'
AND c.raw_json NOT LIKE '%self.__next_f.push%'
AND c.raw_json NOT LIKE '%Googleで続ける%'
AND c.message_count BETWEEN 1 AND 80
```

Find likely noisy captures:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count,
       length(c.raw_json) AS raw_json_bytes,
       max(length(m.body)) AS max_message_bytes,
       sum(CASE
         WHEN m.body LIKE '%self.__next_f.push%'
           OR m.body LIKE '%Googleで続ける%'
           OR c.title = 'Sign in'
         THEN 1 ELSE 0 END) AS noisy_messages
FROM conversations c
LEFT JOIN messages m ON m.conversation_id = c.id
GROUP BY c.id
HAVING noisy_messages > 0 OR max_message_bytes > 100000 OR c.title = 'Sign in'
ORDER BY noisy_messages DESC, max_message_bytes DESC, datetime(c.updated_at) DESC;
```

Chrome extension feedback to report from DB analysis:

- `title = 'Sign in'`: provider login or landing page was captured as a conversation.
- `raw_json/body LIKE '%self.__next_f.push%'`: page framework payload was captured instead of chat messages.
- Very large single messages: selector likely captured a container or script payload.
- Many `session_index` rows but few `conversations`: sidebar indexing works, but full chat import needs the user to open sessions or provider-specific history capture needs improvement.
- Missing `message_search`: local SQLite build or migration did not create FTS5 search index.

## Answering Pattern

When answering a user request from the archive:

1. State the archive path and counts inspected.
2. List the query terms or filters used.
3. Summarize the top matching conversations by provider/title/date.
4. Use message snippets or ordered messages to answer the request.
5. Separate "answer from history" from "capture/extension feedback".
