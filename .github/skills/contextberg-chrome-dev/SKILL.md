---
name: contextberg-chrome-dev
description: Query and synthesize the local ContextBerg Browser AI History SQLite archive created by the Chrome extension. Use when an agent needs to answer from saved ChatGPT, Claude, Gemini, Perplexity, or Manus conversations; find prior prompts, answers, decisions, links, or references; summarize a topic across past chats; compare providers; inspect tags, favorites, or projects; or export evidence from saved browser AI history.
---

# ContextBerg Browser AI History SQLite

Use this skill to answer user requests from saved browser AI conversations. The primary job is retrieval and synthesis, not extension debugging.

## Archive Paths

Resolve the archive folder in this order:

1. `CONTEXTBERG_BROWSER_AI_DIR`, if set.
2. Windows: `%LOCALAPPDATA%\ContextBerg\browser-ai-history`
3. macOS: `~/Library/Application Support/ContextBerg/browser-ai-history`
4. Linux: `~/.local/share/contextberg/browser-ai-history`

The main database is `browser-ai-history.sqlite` inside that folder.

Related generated files may exist:

- `browser-ai-conversations.md`: readable archive view.
- `browser-ai-conversations.json`: full JSON export.
- `browser-ai-conversations.jsonl`: JSONL records.
- `browser-ai-contextberg.json`: ContextBerg-compatible bundle.
- `conversations/*.json`: per-conversation captures.

## Command Strategy

Prefer SQL that works in any SQLite client.

macOS:

```bash
DB="${CONTEXTBERG_BROWSER_AI_DIR:-$HOME/Library/Application Support/ContextBerg/browser-ai-history}/browser-ai-history.sqlite"
sqlite3 "$DB" "SELECT COUNT(*) FROM conversations;"
```

Linux:

```bash
DB="${CONTEXTBERG_BROWSER_AI_DIR:-$HOME/.local/share/contextberg/browser-ai-history}/browser-ai-history.sqlite"
sqlite3 "$DB" "SELECT COUNT(*) FROM conversations;"
```

Windows PowerShell:

```powershell
$archiveDir = if ($env:CONTEXTBERG_BROWSER_AI_DIR) { $env:CONTEXTBERG_BROWSER_AI_DIR } else { Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history" }
$DB = Join-Path $archiveDir "browser-ai-history.sqlite"
```

Node 22+ fallback for JSON output or when `sqlite3` is unavailable:

```bash
node --input-type=module - "$DB" <<'NODE'
import { DatabaseSync } from "node:sqlite";
const db = new DatabaseSync(process.argv[2], { readOnly: true });
console.log(db.prepare("SELECT COUNT(*) AS conversations FROM conversations").get());
NODE
```

## Schema

```sql
conversations(id, provider, title, url, conversation_id, capture_method, captured_at, updated_at, message_count, raw_json)
messages(conversation_id, order_index, message_id, role, body, markdown, created_at, raw_json)
conversation_meta(conversation_id, project, account_id, account_label, labels_json, is_favorite, notes, updated_at)
session_index(id, provider, title, url, conversation_id, first_seen_at, last_seen_at, capture_count, raw_json)
```

When available, `message_search` is an FTS5 table over provider, title, URL, message body, and markdown.

## Start Every Task

Inspect counts and FTS availability first:

```sql
SELECT
  (SELECT COUNT(*) FROM conversations) AS conversations,
  (SELECT COUNT(*) FROM messages) AS messages,
  (SELECT COUNT(*) FROM conversation_meta) AS meta,
  (SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'message_search') AS has_message_search;
```

Provider coverage:

```sql
SELECT provider, COUNT(*) AS conversations, SUM(message_count) AS messages, MAX(updated_at) AS latest
FROM conversations
WHERE title <> 'Sign in'
GROUP BY provider
ORDER BY conversations DESC, provider ASC;
```

## User Request Recipes

### Find what I discussed about a topic

Use FTS5 first. Quote important terms and combine them with `AND` for precision.

```sql
SELECT c.id, c.provider, c.title, c.updated_at, s.role,
       substr(replace(COALESCE(s.markdown, s.body), char(10), ' '), 1, 320) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"pricing" AND "launch"'
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC
LIMIT 30;
```

Fallback without FTS5:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, m.role,
       substr(replace(COALESCE(m.markdown, m.body), char(10), ' '), 1, 320) AS snippet
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE (m.body LIKE '%pricing%' OR m.markdown LIKE '%pricing%')
  AND (m.body LIKE '%launch%' OR m.markdown LIKE '%launch%')
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 30;
```

### Summarize a specific conversation

After selecting a conversation id, read messages in order.

```sql
SELECT m.order_index, m.role, COALESCE(m.markdown, m.body) AS text
FROM messages m
WHERE m.conversation_id = 'chatgpt:example'
ORDER BY m.order_index ASC;
```

### Show recent work memory

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count,
       COALESCE(meta.project, '') AS project,
       COALESCE(meta.labels_json, '[]') AS labels,
       COALESCE(meta.is_favorite, 0) AS favorite
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC
LIMIT 25;
```

### Find decisions, TODOs, or next actions

Search for intent words, then open the best matching conversations.

```sql
SELECT c.id, c.provider, c.title, c.updated_at, s.role,
       substr(replace(COALESCE(s.markdown, s.body), char(10), ' '), 1, 360) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"decided" OR "decision" OR "TODO" OR "next" OR "follow-up"'
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC
LIMIT 40;
```

### Find links and citations

```sql
SELECT c.id, c.provider, c.title, c.updated_at, m.role,
       substr(COALESCE(m.markdown, m.body), 1, 900) AS text
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE COALESCE(m.markdown, m.body) LIKE '%http%'
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 40;
```

### Compare providers on a topic

```sql
SELECT c.provider, c.id, c.title, c.updated_at, s.role,
       substr(replace(COALESCE(s.markdown, s.body), char(10), ' '), 1, 300) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"robotics" AND "market"'
  AND c.title <> 'Sign in'
ORDER BY c.provider ASC, datetime(c.updated_at) DESC
LIMIT 50;
```

Group findings by provider in the answer and call out where providers disagree or add unique evidence.

### Use tags, favorites, or projects

```sql
SELECT c.id, c.provider, c.title, c.updated_at,
       meta.project, meta.labels_json, meta.is_favorite, meta.notes
FROM conversations c
JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE meta.is_favorite = 1
   OR meta.project LIKE '%work%'
   OR meta.labels_json LIKE '%research%'
ORDER BY datetime(c.updated_at) DESC
LIMIT 30;
```

Combine metadata with FTS using `EXISTS`:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, meta.project, meta.labels_json
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE (meta.project LIKE '%work%' OR meta.labels_json LIKE '%research%')
  AND EXISTS (
    SELECT 1
    FROM message_search
    WHERE message_search.conversation_id = c.id
      AND message_search MATCH '"pricing"'
  )
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

### Build a timeline for a topic

```sql
SELECT c.updated_at, c.provider, c.title, c.id,
       substr(replace(COALESCE(s.markdown, s.body), char(10), ' '), 1, 260) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"ContextBerg"'
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) ASC, CAST(s.order_index AS INTEGER) ASC
LIMIT 80;
```

## Answering Pattern

When answering from history:

1. State the database path and counts inspected.
2. State the search terms, provider filters, and metadata filters used.
3. List the most relevant conversations by provider, title, date, and id.
4. Open the top conversations and answer from ordered messages, not snippets alone.
5. Separate evidence from interpretation.
6. If results are weak, say what was searched and what was missing.

Useful answer shape:

```text
Inspected: <db path>, <N> conversations, <M> messages.
Searched: <terms/filters>.

Most relevant conversations:
- <provider> / <title> / <date> / <conversation_id>

Answer from saved history:
...

Evidence:
- <provider/title/order_index>: <short snippet or summary>
```

## Optional Troubleshooting

Only use diagnostics when the user asks why history is missing or results look broken. Keep this out of normal analysis.

Check whether seen sessions lack captured conversation bodies:

```sql
SELECT COUNT(*) AS indexed_without_body
FROM session_index s
LEFT JOIN conversations c
  ON c.conversation_id = s.conversation_id
  OR c.url = s.url
WHERE c.id IS NULL
  AND lower(rtrim(s.url, '/')) NOT LIKE '%/new';
```

Check for likely malformed captures:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count
FROM conversations c
LEFT JOIN messages m ON m.conversation_id = c.id
GROUP BY c.id
HAVING c.title = 'Sign in'
   OR max(length(COALESCE(m.body, m.markdown))) > 100000
   OR sum(CASE WHEN COALESCE(m.body, m.markdown) LIKE '%self.__next_f.push%' THEN 1 ELSE 0 END) > 0
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```
