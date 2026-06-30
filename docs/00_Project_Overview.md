# LexiRedact Project Overview

| Field | Value |
| --- | --- |
| Purpose | Build a mental model of LexiRedact before reading implementation code. |
| Audience | Mid-level Python developer new to this repository. |
| Related files | [01_Folder_Explanation.md], [02_Request_Flow.md], [03_Class_Diagram.md] |

## Problem Statement

LexiRedact protects retrieval-augmented generation ingestion pipelines from storing personally identifiable information in a vector database. In a normal RAG system, raw documents are chunked, embedded, and persisted. If those chunks contain names, phone numbers, emails, IP addresses, card numbers, or similar sensitive values, the embedding layer and vector store become a long-lived privacy risk. LexiRedact is built around the rule that sanitized text should be stored in normal privacy-preserving modes (see: `lexiredact/pipeline/orchestrator.py`, lines 6-10 and 137-148).

The core risk is not only that a user might retrieve visible PII later. Sensitive strings can affect embedding vectors, metadata, cache keys, logs, and vector database snapshots. LexiRedact reduces that risk by detecting PII spans, producing redacted text, and controlling which text is embedded and which text is stored according to `pipeline_mode` (see: `lexiredact/config/schema.py`, lines 96-112; `lexiredact/pipeline/orchestrator.py`, lines 90-306).

In the current implementation, LexiRedact expects pre-chunked input dictionaries, not raw full documents. There is no built-in document splitter. The canonical public call is synchronous: `pipeline.ingest([...])` (see: `lexiredact/pipeline_api.py`, lines 66-90).

## High-Level Architecture

```text
User code
  |
  | from lexiredact import load_config, LexiredactPipeline
  v
+----------------------+        +-------------------------+
| load_config()        |        | LexiredactPipeline      |
| config/loader.py     |        | pipeline_api.py         |
+----------+-----------+        +-----------+-------------+
           |                                |
           v                                v
+----------------------+        +-------------------------+
| LexiredactConfig     |        | ChunkAdapter            |
| config/schema.py     |        | adapters/chunk_adapter.py|
+----------------------+        +-----------+-------------+
                                            |
                                            v
                                +-------------------------+
                                | Orchestrator            |
                                | pipeline/orchestrator.py|
                                +---+-----------+---------+
                                    |           |
                      +-------------+           +----------------+
                      v                                      v
          +-----------------------+              +-----------------------+
          | PIIDetector           |              | EmbeddingCache         |
          | pipeline/pii/detector.py             | cache/redis_cache.py   |
          +-----------+-----------+              +-----------+-----------+
                      |                                      |
                      v                                      v
          +-----------------------+              +-----------------------+
          | PIIRedactor           |              | EmbedderBase impl      |
          | pipeline/pii/redactor.py             | embedder/*.py          |
          +-----------+-----------+              +-----------+-----------+
                      |                                      |
                      +------------------+-------------------+
                                         v
                             +-----------------------+
                             | VectorStoreBase impl  |
                             | store/chroma.py       |
                             +-----------------------+
```

There is no concrete middleware chain in the repository. The phrase "middleware" in the package description maps to the pipeline facade and orchestrator design, not a `middleware/` package or `next()` chain.

## Core Vocabulary

| Term | Meaning in this repository | Source |
| --- | --- | --- |
| Pipeline | The public facade plus orchestrator that adapts, detects, redacts, embeds, caches, and stores chunks. | `pipeline_api.py`, lines 32-90 |
| Middleware | Marketing/architecture term here; no middleware abstraction exists in code. | no `lexiredact/middleware` folder |
| Adapter | A boundary object that translates caller dicts into `Chunk` objects. | `adapters/chunk_adapter.py`, lines 20-93 |
| Detector | `PIIDetector`, a Presidio wrapper that returns spans without modifying text. | `pipeline/pii/detector.py`, lines 28-133 |
| Anonymiser/redactor | `PIIRedactor`, which replaces detected spans with `<ENTITY_TYPE>`. | `pipeline/pii/redactor.py`, lines 24-78 |
| Embedding | A list of floats returned by an `EmbedderBase` implementation. | `pipeline/embedder/base.py`, lines 33-95 |
| Vector store | A backend satisfying `VectorStoreBase` for upsert, query, and count. | `pipeline/store/base.py`, lines 17-61 |
| Chunk | Internal pre-chunked text unit with `id`, `text`, and `metadata`. | `models/chunk.py`, lines 16-40 |
| Cache | Redis-backed embedding cache keyed by SHA-256 of text. | `cache/redis_cache.py`, lines 22-98 |
| PII entity | A `DetectedEntity` with text, type, offsets, and score. | `models/result.py`, lines 13-38 |
| Redaction strategy | Fixed replacement with `<ENTITY_TYPE>` placeholders. | `pipeline/pii/redactor.py`, lines 65-73 |
| YAML config | YAML or dict converted into Pydantic config models. | `config/loader.py`, lines 31-81 |
| Factory | `create_embedder()` chooses an embedder implementation from config. | `pipeline/embedder/registry.py`, lines 28-55 |

## Main Pipeline In Plain English

1. The caller loads configuration with `load_config()`. The loader accepts a dict, string path, or `Path`, reads YAML when needed, and validates with `LexiredactConfig` (see: `lexiredact/config/loader.py`, lines 31-81).
2. The caller creates `LexiredactPipeline(config)`. The constructor creates a `ChunkAdapter`, embedder, Chroma store, Redis cache wrapper, and `Orchestrator` (see: `lexiredact/pipeline_api.py`, lines 42-57).
3. The caller passes a list of raw chunk dictionaries to `ingest()`. The adapter validates `id` and `text` fields and creates `Chunk` dataclasses (see: `lexiredact/adapters/chunk_adapter.py`, lines 30-66).
4. The orchestrator chooses one of three modes. In `dual`, it detects PII first, then concurrently redacts text and embeds original text, and finally stores only sanitized metadata (see: `lexiredact/pipeline/orchestrator.py`, lines 90-178). In `preredacted`, it embeds sanitized text (lines 180-245). In `raw`, it skips PII and stores original text for baseline evaluation (lines 247-306).
5. The vector store receives IDs, vectors, and metadata. The Chroma implementation uses a persistent local client and upserts into a cosine collection (see: `lexiredact/pipeline/store/chroma.py`, lines 30-72).

Retrieval is only implemented at the vector store level. `ChromaStore.query()` accepts a query vector and returns IDs, metadata, and distances (see: `lexiredact/pipeline/store/chroma.py`, lines 74-108). `EmbedderBase.query_embed()` exists for query embedding (see: `lexiredact/pipeline/embedder/base.py`, lines 64-81), but `LexiredactPipeline` does not expose `retrieve()`.

## Common Usage

```python
from lexiredact import LexiredactPipeline, load_config

config = load_config("lexiredact_config.yaml")
pipeline = LexiredactPipeline(config)
results = pipeline.ingest([{"id": "case-1", "text": "Jane emailed jane@example.com"}])
```

Line by line:

| Line | Explanation |
| --- | --- |
| `from lexiredact import ...` | Uses package-root re-exports declared in `lexiredact/__init__.py`, lines 10-22 and 26-41. |
| `load_config(...)` | Reads YAML, parses with `yaml.safe_load`, and constructs `LexiredactConfig` (see: `config/loader.py`, lines 43-81). |
| `LexiredactPipeline(config)` | Wires adapter, embedder, store, cache, and orchestrator (see: `pipeline_api.py`, lines 48-57). |
| `pipeline.ingest([...])` | Adapts dicts to chunks, skips invalid chunks, then calls `asyncio.run(self._orchestrator.process_batch(chunks))` (see: `pipeline_api.py`, lines 79-90). |

## Design Philosophy

The first principle is boundary isolation. User dictionaries are isolated behind `ChunkAdapter`, embedders behind `EmbedderBase`, vector stores behind `VectorStoreBase`, and config behind Pydantic models (see: `adapters/chunk_adapter.py`, lines 20-93; `pipeline/embedder/base.py`, lines 33-95; `pipeline/store/base.py`, lines 17-61; `config/schema.py`, lines 14-112).

The second principle is fail-open for performance aids and fail-closed for persistence. Cache errors are logged and treated as misses so ingestion can continue (see: `cache/redis_cache.py`, lines 37-74). Storage errors are raised because silently losing writes would be worse than failing the request (see: `pipeline/orchestrator.py`, lines 140-151 and `pipeline/store/base.py`, lines 6-8).

## Call Graph

```text
user code
  -> load_config()
     -> LexiredactConfig(**raw)
  -> LexiredactPipeline.__init__()
     -> ChunkAdapter()
     -> create_embedder()
     -> ChromaStore()
     -> EmbeddingCache()
     -> Orchestrator()
  -> LexiredactPipeline.ingest()
     -> ChunkAdapter.adapt_batch()
     -> Orchestrator.process_batch()
        -> _run_dual() | _run_preredacted() | _run_raw()
```

