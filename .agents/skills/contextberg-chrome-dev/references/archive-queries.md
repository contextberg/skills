# Archive views and query recipes

Prefer these standard SQLite views when present. They use no custom functions and reflect new captures immediately. They are not a write API.

## View contract

| View | Identity and purpose |
| --- | --- |
| `agent_archive_status_v1` | Total/sample/non-sample counts, latest archive update, count of messages with recorded timestamps, FTS availability |
| `agent_conversations_v1` | One row per `archive_id`; provider/title, original `source_url`, derived provider session ID/canonical URL, metadata, timestamps and quality |
| `agent_messages_v1` | One row per `(archive_id, message_index)`; role, body, markdown, `message_text`, original message time and status |
| `agent_sessions_v1` | One row per `index_id`; observations and canonical-URL-based capture match count/status; `archive_id` is NULL unless exactly one match |

Important meanings:

- `legacy_local_id`: existing `conversations.conversation_id`, often a local hash. Do not join it to `session_index.conversation_id`.
- `provider_session_id`: parsed only from recognized provider session URLs; NULL for home, temporary, WEB placeholders, shared URLs and unsupported layouts.
- `canonical_session_url`: query/fragment-free identity, with ChatGPT project routes normalized to `/c/<id>`. Gemini account routes are preserved. Unknown formats remain NULL; they are not declared broken. Original navigation uses `source_url` after validation.
- `source_link_status`: `session_url`, `shared_url`, `unavailable` or `unverified`. Syntax recognition is not an account-access check.
- `record_kind`: `demo` from explicit markers; `sample_candidate` from historical `/demo-` URLs; otherwise `capture`. `is_sample` includes both sample classes. Classification does not change records.
- `capture_integrity`: `empty`, `count_mismatch`, `counts_match`. None proves semantic completeness; `content_completeness` remains `unverified`.
- `message_count` is counted from stored rows, `declared_message_count` is the original stored count. `timestamped_message_count` counts parseable recorded message timestamps.
- `has_metadata` distinguishes missing metadata from false/empty values. Project and notes remain NULL if absent; tags default to `[]` for querying.

## Scope and provider coverage

```sql
SELECT * FROM agent_archive_status_v1;
SELECT provider, COUNT(*) AS saved_records, SUM(message_count) AS stored_messages,
       MAX(archive_updated_at) AS latest_archive_update_at
FROM agent_conversations_v1 WHERE is_sample = 0
GROUP BY provider ORDER BY saved_records DESC;
```

## Hybrid text search

Example: Japanese substring `動画`, plus exact-token FTS matches. Adapt the terms to the request. This query requires the FTS table; omit the FTS arm if it is absent. `UNION` removes overlapping message hits.

```sql
WITH hits AS (
  SELECT conversation_id AS archive_id, CAST(order_index AS INTEGER) AS message_index
  FROM message_search WHERE message_search MATCH '"動画"'
  UNION
  SELECT archive_id, message_index FROM agent_messages_v1
  WHERE instr(COALESCE(body, ''), '動画') > 0
     OR instr(COALESCE(markdown, ''), '動画') > 0
), matched AS (
SELECT m.archive_id, m.message_index, c.provider, c.title, m.role,
       substr(m.message_text, 1, 320) AS snippet, c.source_url,
       c.content_completeness, c.archive_updated_at,
       ROW_NUMBER() OVER (PARTITION BY m.archive_id ORDER BY m.message_index) AS hit_in_conversation
FROM hits h JOIN agent_messages_v1 m
  ON m.archive_id = h.archive_id AND m.message_index = h.message_index
JOIN agent_conversations_v1 c ON c.archive_id = m.archive_id
WHERE c.is_sample = 0
)
SELECT * FROM matched WHERE hit_in_conversation <= 3
ORDER BY archive_updated_at DESC, archive_id, message_index LIMIT 40;
```

To find title/notes/project-only matches (even if message search has no hits):

```sql
SELECT archive_id, provider, title, source_url, project, notes
FROM agent_conversations_v1
WHERE is_sample = 0 AND (
  instr(COALESCE(title, ''), '動画') > 0 OR instr(COALESCE(notes, ''), '動画') > 0
  OR instr(COALESCE(project, ''), '動画') > 0 OR instr(labels_json, '動画') > 0
)
ORDER BY archive_updated_at DESC LIMIT 30;
```

For decisions/TODOs search Japanese words such as `決めた`, `採用`, `次に`, `やること` and relevant English terms. Read the user response to distinguish approval from an assistant suggestion. For tags/favorites, filter `project`, `is_favorite`, or the normalized `conversation_labels` table using `cl.conversation_id = c.archive_id`.

## Read an ordered conversation

Bind `:archive_id` to the returned local archive ID. Use `message_index > :last_read_index` for pagination; do not order FTS's untyped `order_index` lexicographically.

```sql
SELECT message_index, role, message_text, message_created_at, message_time_status
FROM agent_messages_v1 WHERE archive_id = :archive_id
ORDER BY message_index ASC;
```

## Usage and quality

```sql
SELECT provider, role, COUNT(*) AS stored_messages
FROM agent_messages_v1 WHERE is_sample = 0 GROUP BY provider, role;
SELECT source_link_status, capture_integrity, COUNT(*) AS saved_records
FROM agent_conversations_v1 WHERE is_sample = 0
GROUP BY source_link_status, capture_integrity;
SELECT index_id, provider, title, archive_match_count, archive_match_status
FROM agent_sessions_v1 WHERE archive_match_status <> 'unique' LIMIT 30;
```

A timeline ordered by `archive_updated_at` is an **archive update timeline**. To present original conversation chronology, first check message timestamp availability. No timestamp means no precise usage date can be inferred.

## Compatibility with archives without views

No migration is required to retrieve older history. Inspect table columns if needed. The correct raw joins are:

```sql
SELECT c.id AS archive_id, c.provider, c.title, m.order_index AS message_index,
       m.role, COALESCE(NULLIF(m.markdown, ''), m.body, '') AS message_text
FROM conversations c JOIN messages m ON m.conversation_id = c.id
WHERE c.source IS NOT 'demo-seed' AND c.capture_method IS NOT 'demo'
  AND COALESCE(c.url, '') NOT LIKE '%/demo-%'
  AND (instr(COALESCE(m.body, ''), '動画') > 0
       OR instr(COALESCE(m.markdown, ''), '動画') > 0)
ORDER BY c.updated_at DESC, c.id, m.order_index ASC LIMIT 40;
```

Also exclude `metadata.demo = true` when present in valid `raw_json`; explain any heuristic sample exclusions. For exact FTS matches, join `message_search.conversation_id = conversations.id`, and match the message index using `CAST(message_search.order_index AS INTEGER)`.

Never join `conversations.conversation_id` to `messages.conversation_id`: the latter references `conversations.id`. For old session-index diagnostics, prefer exact provider + URL; do not match titles or assume index IDs and local hash IDs are interchangeable. Unknown/no-match is not proof of a lost capture.
