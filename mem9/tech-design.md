# mem9 Tech

## 1. Two-Phase LLM Ingest Pipeline (`service/ingest.go`)

**When does ingest trigger?**

Ingest fires once per **agent run** — not per user message, not when the application process shuts down. An agent run is one complete cycle: user sends a message → agent thinks/calls tools → agent responds. The `agent_end` hook fires at the end of that cycle, which triggers ingest on the messages from that run.

For a long-running bot (e.g. Telegram), the process never ends but ingest still fires after every individual exchange. Short factless exchanges ("what time is it?") return `{"facts": []}` in Phase 1 and nothing is stored.

**The two phases:**

**Phase 1 — Fact Extraction**
Calls an LLM (default: `gpt-4o-mini`) with a structured prompt to extract atomic facts from user messages only (assistant/system messages are ignored). Returns `{"facts": ["fact1", "fact2"]}`.

**Phase 2 — Reconcile**
For each extracted fact, searches existing memories (vector + keyword). Then sends ALL facts + ALL retrieved memories to the LLM in a **single batch call**, asking it to decide `ADD / UPDATE / DELETE / NOOP` for each fact.

This is a mem0-style architecture — memories are LLM-curated atomic facts with automatic deduplication, not raw conversation dumps.

## 2. Integer ID Remapping to Prevent LLM Hallucination (`ingest.go:402-408`)

Real UUIDs are never sent to the LLM. Before the reconcile call, existing memories are remapped:

```go
refs[i] = memoryRef{IntID: i, Text: m.Content}
idMap[i] = m.ID  // maps int → real UUID
```

The LLM only sees `{"id": 0, "text": "..."}`. After the LLM responds with integer IDs, they are resolved back to real UUIDs. This prevents the LLM from inventing fake UUIDs that don't exist in the DB.

## 3. Reciprocal Rank Fusion (RRF) for Hybrid Search (`service/memory.go:120-128`)

Vector search results and FTS/keyword results are merged using RRF with k=60:

```go
scores[m.ID] += 1.0 / (rrfK + float64(rank+1))  // rrfK = 60
```

After RRF merging, a type-weight pass applies a **1.5× score boost** to `pinned` memories (user-explicit) over `insight` type.

## 4. `ArchiveAndCreate` Transactional Update (`repository/tidb/memory.go:153-200`)

Updating a memory never mutates the existing row. It atomically:
1. Archives the old row (`state = 'archived'`, sets `superseded_by = newID`)
2. Inserts a brand new row with updated content

This creates a full immutable history chain — any memory version can be traced via `superseded_by`. Implemented as a single DB transaction to prevent partial state.

## 5. `atomic.Bool` FTS Cold-Start Probe (`repository/tidb/memory.go:21-31`)

FTS availability is tracked with `sync/atomic.Bool`. During server cold start, the FTS index may not be ready yet. The code degrades gracefully to LIKE-based keyword search and upgrades transparently once FTS is confirmed available — no restart required.

## 6. TiDB VECTOR INDEX Query Constraint (`repository/tidb/memory.go:361-365`)

`VEC_COSINE_DISTANCE` must appear **identically** in both `SELECT` and `ORDER BY` for TiDB to use the ANN vector index:

```sql
SELECT ..., VEC_COSINE_DISTANCE(embedding, ?) AS distance
FROM memories
WHERE embedding IS NOT NULL
ORDER BY VEC_COSINE_DISTANCE(embedding, ?)  -- must match SELECT exactly
LIMIT ?
```

If they differ even slightly, TiDB falls back to a brute-force full scan. The `embedding IS NOT NULL` condition in `WHERE` is also mandatory for TiDB vector index usage.

## 7. FTS SQL Injection Mitigation via `ftsSafeLiteral` (`repository/tidb/memory.go:458-462`)

TiDB's `FTS_MATCH_WORD()` function rejects parameterized `?` placeholders (Error 1235 — "match against a non-constant string"). The query term must be inlined as a SQL string literal:

```go
func ftsSafeLiteral(s string) string {
    s = strings.ReplaceAll(s, `\`, `\\`)
    s = strings.ReplaceAll(s, `'`, `''`)
    return s
}
// Used as: fts_match_word('` + safeQ + `', content)
```

Manual escaping of `\` and `'` per MySQL string literal rules prevents SQL injection — a real constraint forced by TiDB's FTS implementation.

## 8. Pinned Memory Guards in Reconciliation (`ingest.go:539-576`)

The LLM reconciler is never allowed to auto-modify user-pinned memories:
- `UPDATE` on a pinned memory → silently converted to `ADD` (new insight created instead)
- `DELETE` on a pinned memory → skipped with a warning

This prevents the automated pipeline from overwriting memories the user explicitly stored.
