# Vidya 🧠

[![CI](https://github.com/AkashaHQ/Vidya/actions/workflows/ci.yml/badge.svg)](https://github.com/AkashaHQ/Vidya/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/AkashaHQ/Vidya/graph/badge.svg)](https://codecov.io/gh/AkashaHQ/Vidya)

**Hybrid Memory for OpenClaw Agents**

> *विद्या (Vidya) — Sanskrit for "knowledge"*

A memory plugin for [OpenClaw](https://github.com/openclaw/openclaw) that gives AI agents persistent long-term memory with hybrid retrieval. Combines LanceDB vector search with BM25 full-text search, Voyage AI embeddings and reranking, multi-scope isolation, and intelligent auto-capture.

## Features

- **Hybrid Search** — Vector similarity + BM25 keyword search fused via Reciprocal Rank Fusion (RRF)
- **Multi-Provider Embeddings** — Voyage AI, OpenAI, or Jina AI embeddings with a single config switch
- **Voyage AI Reranking** — Cross-encoder reranking (`rerank-2.5`) for high-quality retrieval
- **Multi-Scope Isolation** — Separate memory spaces per agent, project, or user with access control
- **Auto-Capture** — Automatically stores important information from conversations using LLM judgment (with heuristic fallback)
- **Auto-Recall** — Injects relevant memories into agent context before each turn
- **Noise Filtering** — Filters out agent denials, meta-questions, and boilerplate
- **MMR Diversity** — Maximal Marginal Relevance deduplication prevents redundant results
- **Post-Processing Pipeline** — Recency boost, importance weighting, length normalization, time decay
- **Daily Backups** — Automatic JSONL exports with 7-day rotation
- **CLI Tools** — List, search, export, import, re-embed, and migrate memories
- **6 Agent Tools** — `memory_recall`, `memory_store`, `memory_forget`, `memory_update`, `memory_stats`, `memory_list`

## Quick Start

> **⚠️ Upgrading from pre-1.0.1:** The plugin id changed from `memory-lancedb-voyage` to `vidya`. Update your `openclaw.json`:
> - Rename `plugins.entries.memory-lancedb-voyage` to `plugins.entries.vidya`
> - Update `plugins.slots.memory` from `"memory-lancedb-voyage"` to `"vidya"`

### 1. Install

**Option A — OpenClaw CLI** (recommended):

```bash
openclaw plugins install @akashahq/vidya
```

This handles downloading, placement in `~/.openclaw/extensions/`, and config scaffolding automatically.

**Option B — git clone** (for development):

```bash
cd ~/.openclaw/extensions
git clone https://github.com/AkashaHQ/Vidya.git vidya
cd vidya
npm install
```

### 2. Configure

Add to your `openclaw.json`:

> **Note:** The `plugins.entries` key (`"vidya"` below) must match the `id` field in the plugin's `openclaw.plugin.json` manifest. If these don't match, OpenClaw won't associate the config with the plugin.

```json
{
  "plugins": {
    "allow": ["vidya"],
    "slots": {
      "memory": "vidya"
    },
    "entries": {
      "vidya": {
        "enabled": true,
        "config": {
          "embedding": {
            "apiKey": "${VOYAGE_API_KEY}",
            "model": "voyage-4-large"
          },
          "autoCapture": true,
          "autoRecall": true,
          "retrieval": {
            "mode": "hybrid",
            "rerank": "cross-encoder"
          }
        }
      }
    }
  }
}
```

> **Note:** `plugins.allow` is OpenClaw's plugin allowlist. Plugins not listed here won't be loaded even if they exist in `extensions/`. Add `"vidya"` to the array alongside any other plugins you're already using.

You can also configure via the OpenClaw gateway API:

```
gateway(action="config.patch", raw=JSON.stringify({
  plugins: {
    slots: { memory: "vidya" },
    allow: ["vidya"],  // add to allowlist
    entries: {
      vidya: {
        enabled: true,
        config: {
          embedding: { apiKey: "${VOYAGE_API_KEY}", model: "voyage-4-large" },
          autoCapture: true,
          autoRecall: true,
          retrieval: { mode: "hybrid", rerank: "cross-encoder" }
        }
      }
    }
  }
}))
```

Or via CLI:

```bash
openclaw config patch '{"plugins":{"slots":{"memory":"vidya"},"entries":{"vidya":{"enabled":true,"config":{"embedding":{"apiKey":"${VOYAGE_API_KEY}","model":"voyage-4-large"}}}}}}'
```

### 3. Set API Key

```bash
# Voyage AI (default)
export VOYAGE_API_KEY="pa-..."

# Or for OpenAI / Jina:
# export OPENAI_API_KEY="sk-..."
# export JINA_API_KEY="jina_..."
```

Get your key at [dash.voyageai.com](https://dash.voyageai.com/), [platform.openai.com](https://platform.openai.com/), or [jina.ai](https://jina.ai/).

### 4. Restart OpenClaw

```bash
openclaw gateway restart
```

See [`config.example.json`](config.example.json) for all available options.

## Configuration

| Section | Key Options | Description |
|---------|-------------|-------------|
| `embedding` | `provider`, `apiKey`, `model`, `dimensions`, `baseUrl` | Embedding provider and model. See [Embedding Providers](#embedding-providers) below |
| `retrieval` | `mode`, `rerank`, `minScore`, `hardMinScore` | `hybrid` (vector+BM25) or `vector` only. Rerank: `cross-encoder`, `lightweight`, or `none` |
| `autoCapture` | `captureLlm`, `captureLlmModel`, `captureLlmUrl`, `captureLlmApiKey` | LLM judges capture-worthiness. Set `captureLlmUrl` for custom endpoint, `captureLlmApiKey` for auth. Falls back to heuristic if LLM unavailable |
| `scopes` | `default`, `definitions`, `agentAccess` | Memory isolation. Define scopes and restrict agent access |
| `sessionMemory` | `enabled`, `messageCount` | Store session summaries on `/new` command |
| `enableManagementTools` | — | Enables `memory_stats` and `memory_list` tools |

## Embedding Providers

Vidya supports three embedding providers. Set `embedding.provider` in your config:

### Voyage AI (default)

```json
{
  "embedding": {
    "provider": "voyage",
    "apiKey": "${VOYAGE_API_KEY}",
    "model": "voyage-4-large"
  }
}
```

Models: `voyage-4-large` (1024d), `voyage-4` (1024d), `voyage-4-lite` (1024d), `voyage-3-large` (1024d), `voyage-3` (1024d), `voyage-3-lite` (512d), `voyage-code-3` (1024d)

Voyage supports task-aware embeddings (`query` vs `document` input types) and cross-encoder reranking via the same API key.

### OpenAI

```json
{
  "embedding": {
    "provider": "openai",
    "apiKey": "${OPENAI_API_KEY}",
    "model": "text-embedding-3-small"
  }
}
```

Models: `text-embedding-3-small` (1536d), `text-embedding-3-large` (3072d), `text-embedding-ada-002` (1536d)

Supports `dimensions` parameter for truncated embeddings (e.g., `"dimensions": 256`). Use `baseUrl` for Azure OpenAI or compatible endpoints.

### Jina

```json
{
  "embedding": {
    "provider": "jina",
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v3"
  }
}
```

Models: `jina-embeddings-v3` (1024d), `jina-embeddings-v2-base-en` (768d)

Jina v3 supports task-aware embeddings (`retrieval.query` / `retrieval.passage`).

> **Note:** Reranking always uses Voyage AI regardless of embedding provider. If you use OpenAI or Jina for embeddings, set `retrieval.rerank` to `"lightweight"` or `"none"` unless you also have a Voyage API key configured.

## Auto-Capture (LLM Judgment)

Vidya can automatically capture important information from conversations. When `captureLlm` is enabled (default), an LLM evaluates each conversation turn to decide what's worth remembering.

**How it works:**
1. Heuristic pre-filter checks for trigger patterns (preferences, decisions, entities, etc.)
2. LLM receives the conversation context and judges capture-worthiness
3. If the LLM says store, it returns refined memory text with category and importance
4. If LLM is unavailable, falls back to the heuristic capture

**LLM endpoint:** By default, Vidya calls the OpenClaw gateway (`localhost:3000/v1/chat/completions`). Set `captureLlmUrl` to use any OpenAI-compatible endpoint.

**Authentication:** External LLM APIs (OpenAI, etc.) require an API key. Set `captureLlmApiKey` to send a `Bearer` token with each request. The field supports `${ENV_VAR}` syntax. If omitted, Vidya falls back to the `OPENCLAW_LLM_API_KEY` environment variable. No auth header is sent when the key is empty (suitable for local gateways).

```json
{
  "captureLlm": true,
  "captureLlmModel": "anthropic/claude-haiku-4-5-20251001",
  "captureLlmUrl": "https://api.openai.com",
  "captureLlmApiKey": "${OPENCLAW_LLM_API_KEY}"
}
```

Set `captureLlm: false` to use heuristic-only capture (no LLM calls, lower quality but zero cost).

## Retrieval Pipeline

```
Query
 ├─ Adaptive Skip Check (greetings, commands, short text → skip)
 ├─ Vector Search (LanceDB ANN, cosine distance)
 └─ BM25 Full-Text Search (LanceDB FTS index)
      │
      ├─ RRF Fusion (weighted combination)
      ├─ Voyage Rerank (cross-encoder, blended 60/40 with fusion score)
      ├─ Recency Boost (exponential decay, configurable half-life)
      ├─ Importance Weighting (per-memory importance score)
      ├─ Length Normalization (penalize overly long entries)
      ├─ Time Decay (gradual score reduction for old memories, floor 0.5x)
      ├─ Hard Min Score Filter (discard low-confidence results)
      ├─ Noise Filter (remove denials, meta-questions, boilerplate)
      └─ MMR Diversity (deduplicate similar results, cosine threshold 0.85)
```

## Agent Tools

| Tool | Description |
|------|-------------|
| `memory_recall` | Search memories with hybrid retrieval. Supports scope/category filters. |
| `memory_store` | Save information with category, importance, and scope. Deduplicates against existing memories. |
| `memory_forget` | Delete by ID or search query. Shows candidates for ambiguous matches. |
| `memory_update` | Update text, importance, or category in-place. Supports ID prefix matching. |
| `memory_stats` | Memory count by scope and category, retrieval config info. *(requires `enableManagementTools`)* |
| `memory_list` | List recent memories with scope/category/offset filters. *(requires `enableManagementTools`)* |

## CLI

```bash
# List memories
openclaw memory list [--scope global] [--category fact] [--limit 20]

# Search
openclaw memory search "your query" [--limit 5]

# Stats
openclaw memory stats

# Delete
openclaw memory delete <memory-id>

# Export to JSONL
openclaw memory export [--output memories.jsonl]

# Import from JSONL
openclaw memory import <file.jsonl>

# Re-embed all memories (after model change)
openclaw memory reembed [--batch-size 10]

# Migrate from legacy DB
openclaw memory migrate <old-db-path>
```

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full module dependency map.

```
index.ts          → Plugin entry, lifecycle hooks, auto-capture/recall
src/config.ts     → Config parser and validation
src/embedder.ts   → Voyage AI embedding (native fetch, no SDK)
src/store.ts      → LanceDB storage (vector + BM25 search)
src/retriever.ts  → Hybrid retrieval, RRF fusion, reranking, post-processing
src/scopes.ts     → Multi-scope access control
src/tools.ts      → Agent tool definitions
src/noise-filter.ts → Low-quality memory filtering
src/adaptive-retrieval.ts → Skip retrieval for trivial queries
src/migrate.ts    → Legacy DB migration
cli.ts            → CLI commands
```

## Benchmarks

Tested with 32 integration tests on production data:

| Metric | Score |
|--------|-------|
| Exact noun recall | 92% |
| Semantic recall | 100% |
| Hybrid recall | 100% |
| Avg retrieval latency | ~400ms |

## Troubleshooting

### Plugin Not Found / Tools Don't Appear

If `memory_store`, `memory_recall`, etc. don't show up after installation:

1. **Extension must be directly under `~/.openclaw/extensions/`** — OpenClaw scans `~/.openclaw/extensions/*.ts` and `~/.openclaw/extensions/*/index.ts`. If the plugin is nested inside a `node_modules/` subfolder, it won't be discovered.

2. **Plugin manifest must exist** — Each extension needs an `openclaw.plugin.json` file at its root with a valid `id` field.

3. **Config key must match manifest `id`** — The key in `plugins.entries.<id>` in your `openclaw.json` must exactly match the `id` field in the plugin's `openclaw.plugin.json`. For Vidya, this is `"vidya"`.

4. **Run the doctor** — Use `openclaw plugins doctor` to diagnose plugin loading issues.

### jiti Cache

OpenClaw loads `.ts` files via [jiti](https://github.com/unjs/jiti). After modifying `.ts` source files, you must clear the transpilation cache:

```bash
rm -rf /tmp/jiti/
openclaw gateway restart
```

Without this, OpenClaw may continue running stale compiled code.

### Vector Dimension Lock

Once a database is created with a specific embedding model, changing models requires a new `dbPath` or re-embedding all memories via `openclaw memory reembed`.

### Plugin Discovery Order

OpenClaw discovers extensions in this order (first match wins):

1. Paths listed in `plugins.load.paths`
2. `<workspace>/.openclaw/extensions/*.ts` and `*/index.ts`
3. `~/.openclaw/extensions/*.ts` and `*/index.ts`
4. Bundled extensions

## Dependencies

- [`@lancedb/lancedb`](https://github.com/lancedb/lancedb) — Vector database (embedded, no server needed)
- [`@sinclair/typebox`](https://github.com/sinclairzx81/typebox) — Tool parameter schemas
- [Voyage AI API](https://voyageai.com/) — Embeddings and reranking (API key required)
- No `openai` package — all API calls use native `fetch()`

## License

[MIT](LICENSE) © 2026 AKASHA-HQ
