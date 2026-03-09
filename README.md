# Claw Memory Ultra

Enhanced long-term memory plugin for [OpenClaw](https://github.com/openclaw/openclaw). Replaces the built-in `memory-lancedb` with hybrid retrieval, multi-scope isolation, and a management CLI.

## What's Different from Built-in Memory

| | Built-in `memory-lancedb` | Claw Memory Ultra |
|---|---|---|
| Retrieval | Vector only | Hybrid (Vector + BM25 fusion) |
| Reranking | None | Cross-encoder (Jina, SiliconFlow, Voyage, Pinecone) |
| Scoring | Single-pass | 10-stage pipeline (recency, importance, time decay, MMR, etc.) |
| Scope isolation | None | Multi-scope (`global`, `agent:<id>`, `project:<id>`, custom) |
| Noise filtering | None | Filters refusals, greetings, meta-questions |
| Embedding providers | Limited | Any OpenAI-compatible (OpenAI, Gemini, Jina, Ollama) |
| Management | None | Full CLI (list, search, export, import, migrate) |

## Architecture

```
                        ┌──────────────────────────┐
                        │    index.ts (Entry)       │
                        │  Config · Hooks · Lifecycle│
                        └─────┬──────┬──────┬───────┘
                              │      │      │
                 ┌────────────┤      │      ├────────────┐
                 │            │      │      │            │
           ┌─────▼────┐ ┌────▼───┐ ┌▼──────▼──┐  ┌──────▼────┐
           │ store.ts  │ │embedder│ │retriever  │  │ scopes.ts │
           │ (LanceDB) │ │ .ts    │ │ .ts       │  │ (ACL)     │
           └───────────┘ └────────┘ └─────┬─────┘  └───────────┘
                                          │
                                    ┌─────▼──────────────┐
                                    │ noise-filter.ts     │
                                    │ adaptive-retrieval.ts│
                                    └─────────────────────┘
           ┌───────────┐  ┌────────┐  ┌──────────┐
           │ tools.ts   │  │ cli.ts │  │migrate.ts│
           │ (4 tools)  │  │ (CLI)  │  │          │
           └───────────┘  └────────┘  └──────────┘
```

### How Retrieval Works

```
Query ──→ Vector Search (cosine ANN) ─┐
                                       ├─→ RRF Fusion ──→ Rerank ──→ Scoring Pipeline ──→ Results
Query ──→ BM25 Full-Text Search ──────┘

Scoring Pipeline:
  Recency Boost → Importance Weight → Length Norm → Time Decay → Hard Min Score → MMR Diversity
```

The retriever fetches candidates from both vector and BM25 indexes, fuses them using Reciprocal Rank Fusion, optionally reranks with a cross-encoder, then runs through a multi-stage scoring pipeline that considers recency, importance, document length, time decay, and diversity.

## Installation

```bash
# Clone into your OpenClaw workspace plugins directory
cd ~/.openclaw/workspace
git clone https://github.com/jimmdd/claw-memory-ultra.git plugins/claw-memory-ultra
cd plugins/claw-memory-ultra
npm install
```

Add to your `openclaw.json`:

```json
{
  "plugins": {
    "allow": ["claw-memory-ultra"],
    "load": {
      "paths": ["plugins/claw-memory-ultra"]
    },
    "slots": {
      "memory": "claw-memory-ultra"
    },
    "entries": {
      "claw-memory-ultra": {
        "enabled": true,
        "config": {
          "embedding": {
            "apiKey": "${JINA_API_KEY}",
            "model": "jina-embeddings-v5-text-small",
            "baseURL": "https://api.jina.ai/v1",
            "dimensions": 1024
          }
        }
      }
    }
  }
}
```

Restart the gateway:

```bash
openclaw gateway restart
```

Verify:

```bash
openclaw plugins list
openclaw config get plugins.slots.memory  # should show "claw-memory-ultra"
```

> **Note:** Only one memory plugin can be active at a time. Disable the built-in `memory-lancedb` if you were using it.

## Configuration

### Embedding Providers

Works with any OpenAI-compatible embedding API:

| Provider | Model | Base URL | Dimensions |
|---|---|---|---|
| Jina | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 |
| OpenAI | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 |
| Google Gemini | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 |
| Ollama (local) | `nomic-embed-text` | `http://localhost:11434/v1` | varies |

### Rerank Providers

Optional cross-encoder reranking for better result quality:

| Provider | Config value | Endpoint |
|---|---|---|
| Jina (default) | `jina` | `https://api.jina.ai/v1/rerank` |
| SiliconFlow | `siliconflow` | `https://api.siliconflow.com/v1/rerank` |
| Voyage AI | `voyage` | `https://api.voyageai.com/v1/rerank` |
| Pinecone | `pinecone` | `https://api.pinecone.io/rerank` |

### Full Config Reference

<details>
<summary>All configuration options</summary>

```json
{
  "embedding": {
    "apiKey": "sk-...",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true,
    "chunking": true
  },
  "dbPath": "~/.openclaw/memory/lancedb-pro",
  "autoCapture": true,
  "autoRecall": false,
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "minScore": 0.3,
    "rerank": "cross-encoder",
    "rerankApiKey": "sk-...",
    "rerankModel": "jina-reranker-v3",
    "rerankEndpoint": "https://api.jina.ai/v1/rerank",
    "rerankProvider": "jina",
    "candidatePoolSize": 20,
    "recencyHalfLifeDays": 14,
    "recencyWeight": 0.1,
    "filterNoise": true,
    "lengthNormAnchor": 500,
    "hardMinScore": 0.35,
    "timeDecayHalfLifeDays": 60,
    "reinforcementFactor": 0.5,
    "maxHalfLifeMultiplier": 3
  },
  "enableManagementTools": false,
  "scopes": {
    "default": "global",
    "definitions": {
      "global": { "description": "Shared knowledge" },
      "agent:my-bot": { "description": "Bot-specific memories" }
    },
    "agentAccess": {
      "my-bot": ["global", "agent:my-bot"]
    }
  },
  "sessionMemory": {
    "enabled": false,
    "messageCount": 15
  }
}
```

</details>

### Key Settings

| Setting | Default | Description |
|---|---|---|
| `autoCapture` | `true` | Auto-extract facts/preferences from conversations |
| `autoRecall` | `false` | Auto-inject relevant memories into agent context |
| `retrieval.mode` | `hybrid` | `hybrid` (vector + BM25) or `vector` only |
| `retrieval.rerank` | `cross-encoder` | `cross-encoder`, `lightweight`, or `none` |
| `retrieval.hardMinScore` | `0.35` | Discard results below this score |
| `enableManagementTools` | `false` | Enable `memory_list` and `memory_stats` tools |

## Agent Tools

The plugin registers these tools automatically:

| Tool | Description |
|---|---|
| `memory_store` | Store a memory with category, importance, and scope |
| `memory_recall` | Search memories using hybrid retrieval |
| `memory_forget` | Delete a memory by ID or search query |
| `memory_update` | Update an existing memory in-place |
| `memory_list` | List memories (requires `enableManagementTools`) |
| `memory_stats` | View memory statistics (requires `enableManagementTools`) |

## CLI

```bash
openclaw memory-pro list [--scope global] [--category fact] [--limit 20] [--json]
openclaw memory-pro search "query" [--scope global] [--limit 10] [--json]
openclaw memory-pro stats [--scope global] [--json]
openclaw memory-pro delete <id>
openclaw memory-pro delete-bulk --scope global [--before 2025-01-01] [--dry-run]
openclaw memory-pro export [--scope global] [--output memories.json]
openclaw memory-pro import memories.json [--scope global] [--dry-run]
openclaw memory-pro reembed --source-db /path/to/old-db [--batch-size 32]
openclaw memory-pro migrate check|run|verify [--source /path] [--dry-run]
```

## Database Schema

LanceDB table `memories`:

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Primary key |
| `text` | string | Memory content (FTS indexed) |
| `vector` | float[] | Embedding vector |
| `category` | string | `preference`, `fact`, `decision`, `entity`, `other` |
| `scope` | string | Scope identifier (e.g. `global`, `agent:main`) |
| `importance` | float | 0-1 importance score |
| `timestamp` | int64 | Creation time (ms) |
| `metadata` | string (JSON) | Extended metadata |

## Migrating from Built-in Memory

If you have existing memories in the built-in `memory-lancedb` plugin:

```bash
openclaw memory-pro migrate check    # preview what will be migrated
openclaw memory-pro migrate run      # run the migration
openclaw memory-pro migrate verify   # confirm everything transferred
```

## Dependencies

| Package | Purpose |
|---|---|
| `@lancedb/lancedb` | Vector database (ANN + FTS) |
| `openai` | OpenAI-compatible embedding client |
| `@sinclair/typebox` | JSON Schema type definitions |

## License

MIT
