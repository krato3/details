# Main Files And Criticality

| Field | Value |
| --- | --- |
| Purpose | Rank high-leverage files and explain what breaks if they change. |
| Audience | Contributors deciding where to start or how risky a change is. |
| Related files | [01_Folder_Explanation.md], [03_Class_Diagram.md], [15_Architecture_Decisions.md] |

## Criticality Ranking

```text
1.  lexiredact/pipeline_api.py
2.  lexiredact/pipeline/orchestrator.py
3.  lexiredact/config/schema.py
4.  lexiredact/config/loader.py
5.  lexiredact/pipeline/store/base.py
6.  lexiredact/pipeline/embedder/base.py
7.  lexiredact/pipeline/pii/detector.py
8.  lexiredact/pipeline/pii/redactor.py
9.  lexiredact/pipeline/store/chroma.py
10. lexiredact/cache/redis_cache.py
11. lexiredact/adapters/chunk_adapter.py
12. lexiredact/pipeline/embedder/registry.py
13. lexiredact/pipeline/embedder/sentence_transformers.py
14. lexiredact/pipeline/embedder/huggingface.py
15. lexiredact/models/result.py
16. lexiredact/models/chunk.py
17. lexiredact/exceptions.py
18. lexiredact/app_logging.py
19. lexiredact/cli.py
20. __init__.py files
```

## 1. `lexiredact/pipeline_api.py`

Why it exists: it is the public facade that makes the system usable without manual wiring (see: lines 32-40).

Responsibilities:

1. Store the root config.
2. Create `ChunkAdapter`.
3. Create or accept an embedder.
4. Create or accept a vector store.
5. Create `EmbeddingCache`.
6. Create `Orchestrator`.
7. Adapt raw dicts and start async processing.

Inputs: `LexiredactConfig`, optional `EmbedderBase`, optional `VectorStoreBase`, and `raw_chunks` (see: lines 42-90).

Outputs: `list[ProcessingResult]` from `ingest()`.

Internal dependencies: adapters, cache, config, models, embedder registry/base, orchestrator, store base/concrete (see: lines 20-29).

External dependencies: standard `asyncio`.

Breaking impact: changing the constructor or `ingest()` signature breaks the main user API and every example using `LexiredactPipeline` (see: `examples/quickstart.py`, lines 39-59).

## 2. `lexiredact/pipeline/orchestrator.py`

Why it exists: it centralizes runtime policy for raw, preredacted, and dual processing.

Responsibilities:

1. Dispatch by `pipeline_mode`.
2. Run PII detection.
3. Run redaction.
4. Run embedding and cache checks.
5. Store vectors and metadata.
6. Build `ProcessingResult`.
7. Measure latency.

Inputs: `LexiredactConfig`, `EmbedderBase`, `VectorStoreBase`, `EmbeddingCache`, and `list[Chunk]` (see: lines 48-63).

Outputs: `list[ProcessingResult]`.

Internal dependencies: cache, config, exceptions, models, embedder base, PII classes, store base (lines 24-33).

External dependencies: standard `asyncio` and `time`.

Breaking impact: this file controls whether original text reaches storage. A wrong edit can create a privacy regression, especially around metadata construction in dual/preredacted modes (see: lines 140-148 and 207-214).

## 3. `lexiredact/config/schema.py`

Why it exists: the runtime is declarative; this file defines all validated config knobs.

Responsibilities:

1. Define defaults.
2. Forbid unknown keys.
3. Restrict enum values with `Literal`.
4. Resolve `PIIConfig.nlp_model` from legacy `spacy_model`.

Inputs: raw dict values from loader.

Outputs: Pydantic config objects.

Internal dependencies: none.

External dependencies: Pydantic.

Breaking impact: changing defaults changes runtime behavior for every config that omits those keys. Changing `Literal` values can make existing YAML invalid (see: lines 37, 63, 107).

## 4. `lexiredact/config/loader.py`

Why it exists: it normalizes dict/path/YAML input into one `LexiredactConfig`.

Responsibilities:

1. Accept supported config sources.
2. Read YAML files.
3. Parse YAML safely.
4. Wrap validation failures in `LexiredactConfigError`.

Breaking impact: a loader change affects package root `load_config()` and all CLI commands (see: `lexiredact/__init__.py`, line 10; `lexiredact/cli.py`, line 92).

## 5. `lexiredact/pipeline/store/base.py`

Why it exists: vector DBs must be replaceable.

Responsibilities: define `upsert_batch`, `query`, and `count` contracts (see: lines 24-61).

Breaking impact: changing this interface breaks all custom stores and `ChromaStore`.

## 6. `lexiredact/pipeline/embedder/base.py`

Why it exists: embedding models must be replaceable.

Responsibilities: define document embedding, query embedding, and dimension contracts (see: lines 48-95).

Breaking impact: changing this interface breaks all built-in and custom embedders.

## 7. `lexiredact/pipeline/pii/detector.py`

Why it exists: PII detection is separate from redaction so the orchestrator can decide mode policy.

Breaking impact: bad edits affect privacy recall. Returning spans with wrong offsets breaks redaction (see: lines 124-131).

## 8. `lexiredact/pipeline/pii/redactor.py`

Why it exists: detected spans must be transformed into sanitized text.

Breaking impact: changing operator construction can leak original text or corrupt replacements (see: lines 65-73).

## 9. `lexiredact/pipeline/store/chroma.py`

Why it exists: it adapts LexiRedact storage calls to ChromaDB.

Breaking impact: metadata or ID changes alter persisted data compatibility. Query result shape changes break callers using `VectorStoreBase.query()` (see: lines 96-103).

## 10. `lexiredact/cache/redis_cache.py`

Why it exists: cache optional embedding reuse without making Redis mandatory.

Breaking impact: changing fail-open behavior can make Redis outages fail ingestion. Key changes invalidate old cache entries (see: lines 76-79).

## Files Ranked 11 And Below

`adapters/chunk_adapter.py` is the only per-chunk input validation point. Changing it affects which records are accepted or skipped.

`pipeline/embedder/registry.py` is the embedder factory. Adding a backend normally starts here (see: lines 40-55).

`pipeline/embedder/sentence_transformers.py` is the primary embedding implementation. It controls prefixes, batch size, and lazy model loading.

`pipeline/embedder/huggingface.py` is the secondary embedding implementation. Its pooling and truncation choices affect vector meaning.

`models/result.py` defines result serialization. API consumers may depend on `to_dict()` keys (see: lines 73-88).

`models/chunk.py` defines the internal unit of work. It enforces non-empty text (see: lines 30-40).

`exceptions.py` defines user-facing error types. Changing names or inheritance breaks exception handling.

`app_logging.py` controls library logging behavior. It intentionally installs a `NullHandler` (see: lines 16-17).

`cli.py` exposes command-line workflows. It is less central than the library facade but useful for operational inspection.

`__init__.py` files document and re-export interfaces. The root `__init__.py` is public API; subpackage initializers are mostly convenience/documentation.

## Criticality Heatmap

| File | Criticality | Safe to modify? | Interface stable? | Risk if changed |
| --- | --- | --- | --- | --- |
| `pipeline_api.py` | High | No | Yes | Breaks public API. |
| `orchestrator.py` | High | No | Internal but critical | Privacy/storage regressions. |
| `config/schema.py` | High | Carefully | Yes | Existing YAML may fail. |
| `config/loader.py` | High | Carefully | Yes | Config startup breaks. |
| `store/base.py` | High | No | Yes | Custom stores break. |
| `embedder/base.py` | High | No | Yes | Custom embedders break. |
| `pii/detector.py` | High | Carefully | Internal | Missed PII or false positives. |
| `pii/redactor.py` | High | Carefully | Internal | PII leakage. |
| `store/chroma.py` | Medium | Carefully | Mostly | Persistence/query issues. |
| `cache/redis_cache.py` | Medium | Yes with tests | Internal | Cache outages may become fatal. |
| `chunk_adapter.py` | Medium | Yes with tests | Internal | Input compatibility changes. |
| `registry.py` | Medium | Yes | Internal | Backend selection breaks. |
| `sentence_transformers.py` | Medium | Yes with integration tests | Internal | Embedding quality changes. |
| `huggingface.py` | Medium | Yes with integration tests | Internal | Vector dimensions/quality change. |
| `models/result.py` | Medium | Carefully | Yes | Consumers' result parsing breaks. |
| `models/chunk.py` | Low | Yes | Internal | Validation behavior changes. |
| `exceptions.py` | Medium | Carefully | Yes | Error handling breaks. |
| `app_logging.py` | Low | Yes | Public helpers | Logging duplication/silence. |
| `cli.py` | Low | Yes | CLI surface | Commands break. |

