---
name: contextberg-chrome-dev
description: Query and analyze the local ContextBerg Browser AI History SQLite archive created by the Chrome extension. Use when an agent needs to answer from saved ChatGPT, Claude, Gemini, Perplexity, or Manus conversations; search messages with FTS5; inspect tags, favorites, projects, or provider history; compare past answers; find links or prompts; audit capture quality; or report Chrome extension capture issues discovered from the database.
---

# ContextBerg Browser AI History SQLite

Use this skill to answer user requests from the local SQLite archive created by ContextBerg Browser AI History. Query first, then answer from saved conversations. Report capture-quality problems separately from the answer.

## Archive Paths

Resolve the archive folder in this order:

1. `CONTEXTBERG_BROWSER_AI_DIR`, if set.
2. Windows: `%LOCALAPPDATA%\ContextBerg\browser-ai-history`
3. macOS: `~/Library/Application Support/ContextBerg/browser-ai-history`
4. Linux: `~/.local/share/contextberg/browser-ai-history`

The primary database is `browser-ai-history.sqlite` inside that folder.

Generated companion files may also exist:

- `browser-ai-history.contextberg.sqlite`: mirror database for ContextBerg desktop integrations.
- `browser-ai-conversations.md`: readable generated view.
- `browser-ai-conversations.json`: full JSON export.
- `browser-ai-conversations.jsonl`: JSONL records.
- `browser-ai-contextberg.json`: ContextBerg-compatible generated bundle.
- `conversations/*.json`: raw per-conversation captures.

## Command Strategy

Prefer SQL snippets that work in any SQLite client. For shell commands:

- Use `sqlite3 "$DB" "SQL..."` on macOS/Linux when `sqlite3` is available.
- Use Node 22+ `node:sqlite` when `sqlite3` is missing or JSON output is useful.
- On Windows PowerShell, pass the DB path as a quoted argument; do not hardcode user-specific paths unless the user gave one.

Cross-platform DB path examples:

```bash
# macOS
DB="${CONTEXTBERG_BROWSER_AI_DIR:-$HOME/Library/Application Support/ContextBerg/browser-ai-history}/browser-ai-history.sqlite"

# Linux
DB="${CONTEXTBERG_BROWSER_AI_DIR:-$HOME/.local/share/contextberg/browser-ai-history}/browser-ai-history.sqlite"
```

```powershell
# Windows PowerShell
$archiveDir = if ($env:CONTEXTBERG_BROWSER_AI_DIR) { $env:CONTEXTBERG_BROWSER_AI_DIR } else { Join-Path $env:LOCALAPPDATA "ContextBerg\browser-ai-history" }
$DB = Join-Path $archiveDir "browser-ai-history.sqlite"
```

Node JSON runner:

```bash
node --input-type=module - "$DB" <<'NODE'
import { DatabaseSync } from "node:sqlite";
const db = new DatabaseSync(process.argv[2], { readOnly: true });
const rows = db.prepare("SELECT provider, COUNT(*) AS count FROM conversations GROUP BY provider ORDER BY count DESC").all();
console.log(JSON.stringify(rows, null, 2));
NODE
```

```powershell
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync(process.argv[1], { readOnly: true }); const rows = db.prepare('SELECT provider, COUNT(*) AS count FROM conversations GROUP BY provider ORDER BY count DESC').all(); console.log(JSON.stringify(rows, null, 2));" $DB
```

## Core Schema

```sql
conversations(id, provider, title, url, conversation_id, capture_method, captured_at, updated_at, message_count, raw_json)
messages(conversation_id, order_index, message_id, role, body, markdown, created_at, raw_json)
conversation_meta(conversation_id, project, account_id, account_label, labels_json, is_favorite, notes, updated_at)
session_index(id, provider, title, url, conversation_id, first_seen_at, last_seen_at, capture_count, raw_json)
```

When available, `message_search` is an FTS5 table over conversation title, URL, provider, message body, and markdown.

## First Diagnostics

Always start with counts and FTS availability:

```sql
SELECT
  (SELECT COUNT(*) FROM conversations) AS conversations,
  (SELECT COUNT(*) FROM messages) AS messages,
  (SELECT COUNT(*) FROM session_index) AS sessions,
  (SELECT COUNT(*) FROM conversation_meta) AS meta,
  (SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'message_search') AS has_message_search;
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
WHERE c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

## Common Requests / Fast Commands

### "最近何を話していた？"

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count,
       COALESCE(meta.labels_json, '[]') AS labels,
       COALESCE(meta.is_favorite, 0) AS favorite
FROM conversations c
LEFT JOIN conversation_meta meta ON meta.conversation_id = c.id
WHERE c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC
LIMIT 20;
```

### "Xについて過去に何を相談した？"

Use FTS5 first. Quote terms and combine with `AND` for precision.

```sql
SELECT c.id, c.provider, c.title, c.updated_at, s.role,
       substr(replace(s.body, char(10), ' '), 1, 280) AS snippet
FROM message_search s
JOIN conversations c ON c.id = s.conversation_id
WHERE message_search MATCH '"pricing" AND "launch"'
  AND c.title <> 'Sign in'
  AND c.raw_json NOT LIKE '%self.__next_f.push%'
ORDER BY datetime(c.updated_at) DESC, CAST(s.order_index AS INTEGER) ASC
LIMIT 30;
```

Fallback when FTS5 is unavailable:

```sql
SELECT c.id, c.provider, c.title, c.updated_at, m.role,
       substr(replace(COALESCE(m.markdown, m.body), char(10), ' '), 1, 280) AS snippet
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE (m.body LIKE '%pricing%' OR m.markdown LIKE '%pricing%')
  AND (m.body LIKE '%launch%' OR m.markdown LIKE '%launch%')
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 30;
```

### "この会話全文を読んで要約して"

```sql
SELECT m.order_index, m.role, COALESCE(m.markdown, m.body) AS text
FROM messages m
WHERE m.conversation_id = 'chatgpt:example'
ORDER BY m.order_index ASC;
```

Compact preview:

```sql
SELECT m.order_index, m.role,
       substr(replace(COALESCE(m.markdown, m.body), char(10), ' '), 1, 600) AS text
FROM messages m
WHERE m.conversation_id = 'chatgpt:example'
ORDER BY m.order_index ASC;
```

### "ChatGPT/Claude/Geminiごとに傾向を見たい"

```sql
SELECT provider,
       COUNT(*) AS conversations,
       SUM(message_count) AS messages,
       MAX(updated_at) AS latest
FROM conversations
WHERE title <> 'Sign in'
GROUP BY provider
ORDER BY conversations DESC, provider ASC;
```

Recent conversations from one provider:

```sql
SELECT id, title, updated_at, message_count, url
FROM conversations
WHERE provider = 'chatgpt'
  AND title <> 'Sign in'
ORDER BY datetime(updated_at) DESC
LIMIT 20;
```

### "タグ/お気に入り/プロジェクト別に探して"

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

Combine metadata with message search using `EXISTS`:

```sql
SELECT DISTINCT c.id, c.provider, c.title, c.updated_at, meta.project, meta.labels_json
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

### "リンクや参照元を探して"

```sql
SELECT c.id, c.provider, c.title, m.role,
       substr(COALESCE(m.markdown, m.body), 1, 700) AS text
FROM messages m
JOIN conversations c ON c.id = m.conversation_id
WHERE COALESCE(m.markdown, m.body) LIKE '%http%'
  AND c.title <> 'Sign in'
ORDER BY datetime(c.updated_at) DESC, m.order_index ASC
LIMIT 40;
```

### "未キャプチャの履歴がある？"

`session_index` contains provider sessions that were seen, even when full messages are not captured yet.

```sql
SELECT s.provider, s.title, s.url, s.conversation_id, s.last_seen_at, s.capture_count
FROM session_index s
LEFT JOIN conversations c
  ON c.conversation_id = s.conversation_id
  OR c.url = s.url
WHERE c.id IS NULL
  AND lower(rtrim(s.url, '/')) NOT LIKE '%/new'
ORDER BY datetime(s.last_seen_at) DESC
LIMIT 50;
```

If this returns rows, say the extension indexed provider history/sidebar links but has not captured those conversation bodies.

### "検索品質/キャプチャ品質を監査して"

```sql
SELECT c.id, c.provider, c.title, c.updated_at, c.message_count,
       length(c.raw_json) AS raw_json_bytes,
       max(length(COALESCE(m.body, m.markdown))) AS max_message_bytes,
       sum(CASE
         WHEN COALESCE(m.body, m.markdown) LIKE '%self.__next_f.push%'
           OR COALESCE(m.body, m.markdown) LIKE '%Google縺ｧ邯壹￠繧%'
           OR c.title = 'Sign in'
         THEN 1 ELSE 0 END) AS noisy_messages
FROM conversations c
LEFT JOIN messages m ON m.conversation_id = c.id
GROUP BY c.id
HAVING noisy_messages > 0 OR max_message_bytes > 100000 OR c.title = 'Sign in'
ORDER BY noisy_messages DESC, max_message_bytes DESC, datetime(c.updated_at) DESC;
```

Report likely extension feedback as:

- `title = 'Sign in'`: login or landing page was captured as a conversation.
- `self.__next_f.push`: framework payload was captured instead of chat messages.
- Very large single messages: selector likely captured a container/script payload.
- Many `session_index` rows but few `conversations`: provider history was indexed, but full chat import still requires opening sessions or better provider-specific capture.
- Missing `message_search`: FTS5 index/migration did not run.

### "エクスポート/生成ファイルはどこ？"

List generated files:

```bash
ls -lh "$(dirname "$DB")"
```

```powershell
Get-ChildItem -LiteralPath (Split-Path -Parent $DB)
```

For the local bridge, check `http://127.0.0.1:18765/health` and export from `http://127.0.0.1:18765/`.

## Answering Pattern

When the user asks from history:

1. State the archive path and row counts inspected.
2. State the query terms, metadata filters, and provider filters used.
3. List the most relevant conversations by provider, title, date, and id.
4. Open the top conversations and answer from ordered messages, not just snippets.
5. Separate "what the history says" from "capture/extension feedback".
6. If results are weak, say whether the database lacks the conversation or capture quality is the blocker.

Useful response shape:

```text
Inspected: <db path>, <N> conversations, <M> messages.
Searched: <terms/filters>.
Most relevant:
- <provider> / <title> / <date> / <conversation_id>

Answer from saved history:
...

Capture notes:
...
```
