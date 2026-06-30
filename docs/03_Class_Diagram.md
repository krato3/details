# LexiRedact Class Diagram

| Field | Value |
| --- | --- |
| Purpose | Reference every major class, its lifecycle, collaborators, and extension role. |
| Audience | Developers modifying class behavior or adding implementations. |
| Related files | [02_Request_Flow.md], [04_Object_Oriented_Design.md], [11_Extending_LexiRedact.md] |

## UML-Style Overview

```text
+--------------------+       *-- +----------------+
| LexiredactPipeline |-----------| ChunkAdapter   |
| +ingest()          |           | +adapt_batch() |
+---------+----------+           +----------------+
          | *--
          v
+--------------------+       *-- +-------------+
| Orchestrator       |-----------| PIIDetector |
| +process_batch()   |           | +detect_batch()
| -_run_dual()       |           +-------------+
| -_run_preredacted()|       *-- +-------------+
| -_run_raw()        |-----------| PIIRedactor |
+---------+----------+           | +redact()   |
          |                      +-------------+
          | o-- EmbedderBase <--- SentenceTransformerEmbedder
          |                 <--- HuggingFaceEmbedder
          |
          | o-- VectorStoreBase <--- ChromaStore
          |
          *-- EmbeddingCache

Config models:
LexiredactConfig *-- InputSchemaConfig
LexiredactConfig *-- PIIConfig
LexiredactConfig *-- EmbedderConfig
LexiredactConfig *-- CacheConfig
LexiredactConfig *-- StoreConfig

Data:
Chunk
DetectedEntity
ProcessingResult
```

Legend: `*--` composition, `o--` dependency supplied by constructor, `<---` inheritance.

## `LexiredactPipeline`

Plain-language summary: this is the facade. It hides component construction and gives callers one ingestion method. It exists so users do not need to manually create adapters, caches, stores, and the orchestrator (see: `lexiredact/pipeline_api.py`, lines 32-40).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: LexiredactConfig, embedder: EmbedderBase | None = None, store: VectorStoreBase | None = None)` (see: `pipeline_api.py`, lines 42-47). |
| Public methods | `ingest(raw_chunks: list[dict[str, Any]]) -> list[ProcessingResult]` (lines 66-90). |
| Private methods | None. |
| Key attributes | `_config`, `_adapter`, `_logger`, `_orchestrator` (lines 48-57). |
| Inheritance | None. |
| Composition | Creates `ChunkAdapter`, default embedder, `ChromaStore`, `EmbeddingCache`, and `Orchestrator` (lines 49-57). |
| Lifecycle | Created once by user; holds reusable loaded/lazy components across ingestion calls. |
| Collaborators | Config, adapter, embedder, vector store, cache, orchestrator. |
| Extension point | Constructor injection for custom embedder and store (lines 45-46). |
| Thread safety | Holds mutable shared collaborators; no explicit locks. Concurrent calls may share embedder/store/cache. |

Call tree:

```text
LexiredactPipeline.ingest()
  -> ChunkAdapter.adapt_batch()
  -> asyncio.run(Orchestrator.process_batch())
```

## `ChunkAdapter`

Plain-language summary: this class is the input boundary. It converts arbitrary caller dictionaries into canonical `Chunk` objects using configurable field names instead of hardcoded keys (see: `lexiredact/adapters/chunk_adapter.py`, lines 1-6).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: InputSchemaConfig)` stores `_config` (lines 27-28). |
| Public methods | `adapt(raw)` validates one dict and returns `Chunk`; `adapt_batch(raws)` returns `(successful_chunks, failed_items)` (lines 30-93). |
| Private methods | None. |
| Key attributes | `_config`, set at construction. |
| Inheritance | None. |
| Composition | Creates `Chunk` instances. |
| Lifecycle | Created by `LexiredactPipeline`; lives as long as pipeline. |
| Collaborators | `InputSchemaConfig`, `Chunk`, `LexiredactInputError`, logger. |
| Extension point | Replace or subclass only if input shape changes; public facade currently hardcodes this adapter. |

## `Orchestrator`

Plain-language summary: the orchestrator is the core runtime policy engine. It decides which mode runs, coordinates PII detection, redaction, embedding, cache lookup, and storage, and constructs `ProcessingResult` objects (see: `lexiredact/pipeline/orchestrator.py`, lines 38-46).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config, embedder, store, cache)` (lines 48-54). |
| Public methods | `async process_batch(chunks) -> list[ProcessingResult]` (lines 63-88). |
| Private methods | `_run_dual`, `_run_preredacted`, `_run_raw`, `_per_chunk_latency` (lines 90-310). |
| Key attributes | `_config`, `_detector`, `_redactor`, `_embedder`, `_store`, `_cache`, `_logger` (lines 55-61). |
| Inheritance | None. |
| Composition | Owns `PIIDetector` and `PIIRedactor`; receives embedder/store/cache by dependency injection. |
| Lifecycle | Created once per pipeline; reuses detector, redactor, embedder, store, and cache. |
| Collaborators | PII, embedder, cache, vector store, models. |
| Extension point | Add new pipeline modes here, but this is high-risk because it controls storage privacy. |
| Thread safety | Contains mutable collaborators and uses default executor; no request context is stored on `self`. |

Private method rationale: mode-specific methods are private because callers should not choose partial execution paths. Only `process_batch()` should dispatch based on validated config.

## `PIIDetector`

Plain-language summary: `PIIDetector` wraps Presidio analysis and translates Presidio results into LexiRedact's `DetectedEntity` dataclass. It detects spans only; it never changes text (see: `lexiredact/pipeline/pii/detector.py`, lines 4-6).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: PIIConfig)` stores config and sets `_analyzer=None` for lazy load (lines 43-50). |
| Public methods | `detect_batch(chunks) -> list[list[DetectedEntity]]` (lines 52-77). |
| Private methods | `_ensure_loaded()` and `_detect_single()` (lines 79-133). |
| Key attributes | `_config`, `_analyzer`. |
| Inheritance | None. |
| Composition | Creates Presidio `AnalyzerEngine` lazily through `build_nlp_engine()`. |
| Lifecycle | Created in `Orchestrator.__init__`; analyzer loads on first detection and then remains in memory. |
| Collaborators | `PIIConfig`, `Chunk`, `DetectedEntity`, Presidio. |
| Extension point | Current strategy is concrete, not interface-based; a new detector would require orchestrator changes. |
| Thread safety | `_analyzer` lazy initialization is not protected by a lock. |

## `PIIRedactor`

Plain-language summary: `PIIRedactor` replaces detected entity spans with placeholders such as `<PERSON>`. It does not detect PII itself; it trusts entities passed from `PIIDetector` (see: `lexiredact/pipeline/pii/redactor.py`, lines 8-10).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__()` creates Presidio `AnonymizerEngine` (lines 32-36). |
| Public methods | `redact(text, entities) -> str` (lines 38-78). |
| Private methods | None. |
| Key attributes | `_anonymizer`. |
| Inheritance | None. |
| Composition | Owns Presidio anonymizer. |
| Lifecycle | Created once per orchestrator. |
| Collaborators | `DetectedEntity`, Presidio `RecognizerResult`, `OperatorConfig`. |
| Extension point | Replacement strategy is hardcoded in `operators` dict (lines 65-68). |
| Thread safety | Module docstring states it holds no per-call state (lines 12-13). |

## `EmbedderBase`

Plain-language summary: this abstract base class is the contract every embedding backend must satisfy. It prevents the orchestrator from caring whether vectors come from sentence-transformers, raw HuggingFace, or a custom implementation (see: `lexiredact/pipeline/embedder/base.py`, lines 33-45).

| Aspect | Detail |
| --- | --- |
| Constructor | None. |
| Public methods | Abstract `embed_batch`, `query_embed`, `get_dimension` (lines 48-95). |
| Private methods | None. |
| Inheritance | Inherits `ABC`. |
| Composition | None. |
| Lifecycle | Concrete instances are created by factory or caller injection. |
| Collaborators | Orchestrator and stores depend on this contract. |
| Extension point | Primary extension point for new embedding providers. |

## `SentenceTransformerEmbedder`

Plain-language summary: this concrete embedder uses the `sentence-transformers` library with lazy model loading, configurable prefixes, batching, normalization, and dimension detection (see: `lexiredact/pipeline/embedder/sentence_transformers.py`, lines 30-43).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: EmbedderConfig)` sets `_config`, `_model=None`, `_dimension`, `_logger` (lines 45-50). |
| Public methods | `embed_batch`, `query_embed`, `get_dimension` (lines 56-111). |
| Private methods | `_ensure_loaded`, `_apply_prefix`, `_encode` (lines 117-172). |
| Key attributes | `_config`, `_model`, `_dimension`, `_logger`. |
| Inheritance | Extends `EmbedderBase`. |
| Composition | Owns a lazy `SentenceTransformer` model. |
| Lifecycle | Created at pipeline construction; model loads on first embed or dimension lookup. |
| Collaborators | `EmbedderConfig`, sentence-transformers. |
| Thread safety | Lazy `_model` assignment is unguarded; concurrent first calls could race. |

## `HuggingFaceEmbedder`

Plain-language summary: this backend uses raw HuggingFace `AutoTokenizer` and `AutoModel` and mean-pools hidden states. It exists for models not packaged as sentence-transformers (see: `lexiredact/pipeline/embedder/huggingface.py`, lines 35-45).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: EmbedderConfig)` sets `_config`, `_model`, `_tokenizer`, `_dimension`, `_logger` (lines 47-52). |
| Public methods | `embed_batch`, `query_embed`, `get_dimension` (lines 58-103). |
| Private methods | `_ensure_loaded`, `_apply_prefix`, `_encode`, `_mean_pool` (lines 109-213). |
| Key attributes | `_model`, `_tokenizer`, `_dimension`. |
| Inheritance | Extends `EmbedderBase`. |
| Composition | Owns lazy tokenizer and model. |
| Lifecycle | Created at pipeline construction; heavy libraries and model load on first use. |
| Collaborators | transformers, torch. |
| Thread safety | Lazy initialization is unguarded. |

## `EmbeddingCache`

Plain-language summary: this optional Redis cache is transparent to the pipeline. A cache failure must never break ingestion; it degrades to a cache miss (see: `lexiredact/cache/redis_cache.py`, lines 4-8).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: CacheConfig)` stores config and `_client=None` (lines 32-35). |
| Public methods | `get(text)`, `set(text, vector)` (lines 37-74). |
| Private methods | `_make_key`, `_ensure_connected` (lines 76-98). |
| Key attributes | `_config`, `_client`. |
| Inheritance | None. |
| Composition | Owns lazy Redis client. |
| Lifecycle | Created at pipeline construction; connects only if enabled and used. |
| Collaborators | Redis, JSON, SHA-256. |
| Thread safety | Shared `_client` lazy assignment is unguarded. |

## `VectorStoreBase`

Plain-language summary: this abstract class is the storage contract. It lets the orchestrator write vectors without depending on Chroma-specific APIs (see: `lexiredact/pipeline/store/base.py`, lines 17-22).

| Aspect | Detail |
| --- | --- |
| Constructor | None. |
| Public methods | Abstract `upsert_batch`, `query`, `count` (lines 24-61). |
| Private methods | None. |
| Inheritance | Inherits `ABC`. |
| Extension point | Primary extension point for Qdrant, Pinecone, Weaviate, or custom stores. |

## `ChromaStore`

Plain-language summary: this concrete store uses a local persistent ChromaDB collection with cosine distance. It wraps Chroma exceptions in `LexiredactStorageError` (see: `lexiredact/pipeline/store/chroma.py`, lines 22-122).

| Aspect | Detail |
| --- | --- |
| Constructor | `__init__(config: StoreConfig, embedding_dimension: int)` (lines 30-51). |
| Public methods | `upsert_batch`, `query`, `count` (lines 53-122). |
| Private methods | None. |
| Key attributes | `_config`, `_embedding_dimension`, `_client`, `_collection`. |
| Inheritance | Extends `VectorStoreBase`. |
| Composition | Owns Chroma persistent client and collection. |
| Lifecycle | Created at pipeline construction; Chroma collection exists across process runs via `persist_directory`. |
| Collaborators | ChromaDB, `StoreConfig`, `LexiredactStorageError`. |
| Thread safety | Depends on Chroma client behavior; no wrapper locks. |

## Config And Data Classes

| Class | Type | Purpose | Source |
| --- | --- | --- | --- |
| `InputSchemaConfig` | Pydantic model | Field mapping for raw chunks. | `config/schema.py`, lines 14-19 |
| `PIIConfig` | Pydantic model | PII engine, entities, language, score, batch size. | `config/schema.py`, lines 22-56 |
| `EmbedderConfig` | Pydantic model | Embedder backend, model, batch, device, prefixes, dimension. | `config/schema.py`, lines 59-76 |
| `CacheConfig` | Pydantic model | Redis cache enablement and TTL. | `config/schema.py`, lines 79-85 |
| `StoreConfig` | Pydantic model | Vector store provider, collection, persistence path. | `config/schema.py`, lines 88-93 |
| `LexiredactConfig` | Pydantic model | Root config object. | `config/schema.py`, lines 96-112 |
| `Chunk` | Dataclass | Internal text unit. | `models/chunk.py`, lines 16-40 |
| `DetectedEntity` | Dataclass | PII span. | `models/result.py`, lines 13-38 |
| `ProcessingResult` | Dataclass | Per-chunk result. | `models/result.py`, lines 41-88 |

