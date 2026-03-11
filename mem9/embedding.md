# mem9 Embedding

## Overview

mem9 supports two embedding modes. The server detects which mode is active via the `MNEMO_EMBED_AUTO_MODEL` env var and branches SQL queries accordingly.

---

## Mode A — External Embedder (default)

The mnemo-server calls an external OpenAI-compatible API to generate embeddings, then stores them as a `VECTOR(1536)` column in TiDB.

```
User stores memory
  → server calls POST /embeddings (OpenAI / Ollama / LM Studio)
  → receives float[] vector
  → stores in TiDB VECTOR(1536) column
```

**Configuration:**

```bash
# OpenAI
MNEMO_EMBED_API_KEY=sk-...                      # required
MNEMO_EMBED_MODEL=text-embedding-3-small        # default
MNEMO_EMBED_DIMS=1536                           # default

# Ollama (local, no key needed)
MNEMO_EMBED_BASE_URL=http://localhost:11434/v1
MNEMO_EMBED_MODEL=nomic-embed-text
MNEMO_EMBED_DIMS=768
```

If both `MNEMO_EMBED_API_KEY` and `MNEMO_EMBED_BASE_URL` are empty → embedder is `nil` → falls back to keyword-only search (no vector search).

**Key implementation detail:** `encoding_format: "float"` is always set in the embedding request. This is required for Ollama/LM Studio which default to base64 encoding. OpenAI also accepts it, so it's safe universally.

---

## Mode B — TiDB Auto-Embedding (TiDB Cloud Serverless only)

The `embedding` column is a **SQL generated column** that TiDB Cloud evaluates on every INSERT:

```sql
-- schema.sql
embedding VECTOR(1024) GENERATED ALWAYS AS (
  EMBED_TEXT("tidbcloud_free/amazon/titan-embed-text-v2", content)
) STORED
```

```
INSERT INTO memories (content, ...)
  → TiDB Cloud executes EMBED_TEXT() generated column expression
  → TiDB Cloud internally calls Amazon Bedrock API (AWS-managed)
  → Returns float[1024] vector → stored in VECTOR column
```

**Configuration:**

```bash
MNEMO_EMBED_AUTO_MODEL=tidbcloud_free/amazon/titan-embed-text-v2
MNEMO_EMBED_AUTO_DIMS=1024
```

**How `EMBED_TEXT()` works:**
- It is a **TiDB Cloud Serverless-only** SQL extension — not in open-source TiDB, not in MySQL.
- `tidbcloud_free/` prefix = TiDB Cloud provides free access to this model via their AWS Bedrock integration. No model API key needed.
- The model runs on AWS infrastructure (not on the TiDB server process itself).
- On INSERT, TiDB triggers a synchronous AWS Bedrock call before committing the row — inserts are slightly slower.

For queries in auto mode, the server uses `VEC_EMBED_COSINE_DISTANCE(embedding, queryText)` instead of `VEC_COSINE_DISTANCE(embedding, queryVec)` — another TiDB built-in that embeds the query string inside the DB engine on the fly.

---

## Comparison

| | Mode A (External) | Mode B (TiDB Auto) |
|---|---|---|
| Who calls the model | mnemo-server | TiDB Cloud (AWS Bedrock) |
| Model choice | Any OpenAI-compat API | Only TiDB-supported models |
| API key needed | Yes (or local Ollama) | No |
| Cost | You pay for API calls | Free tier available |
| Works on | Any TiDB / MySQL | TiDB Cloud Serverless only |
| Vector dims | 1536 (OpenAI default) | 1024 (Titan Embed) |
| Insert latency | Extra HTTP roundtrip | Extra Bedrock roundtrip |
