# Architecture Decisions

| Field | Value |
| --- | --- |
| Purpose | Explain major design choices, trade-offs, exclusions, and future evolution. |
| Audience | Maintainers planning refactors or defending architecture. |
| Related files | [04_Object_Oriented_Design.md], [05_Main_Files.md], [11_Extending_LexiRedact.md] |

## Decision Log

```text
Stable contracts:
  - LexiredactPipeline constructor and ingest()
  - load_config()
  - EmbedderBase
  - VectorStoreBase
  - ProcessingResult.to_dict()
```

### Decision: Use a facade for user interaction

Context: Users should not wire adapters, embedders, stores, caches, and orchestrators manually.

Options considered: expose all components directly; provide a single facade; build a full dependency container.

Decision rationale: `LexiredactPipeline` gives a small API while keeping injection hooks for embedder/store (see: `pipeline_api.py`, lines 42-57).

Trade-offs accepted: the facade currently hardcodes `ChromaStore` default and synchronous `ingest()`.

Consequences: easy quickstart, but async web integration needs additional API work.

### Decision: Use YAML/Pydantic configuration

Context: Runtime behavior should be declarative and validated early.

Options considered: plain dicts, environment variables, YAML with Pydantic.

Decision rationale: Pydantic captures defaults, nested structure, forbidden extras, and validation (see: `config/schema.py`, lines 14-112).

Trade-offs accepted: Pydantic is a core dependency and v2-specific.

Consequences: adding config keys is straightforward but requires schema updates.

### Decision: Use vector store adapter abstraction

Context: Vector DB APIs differ and should not leak into workflow code.

Options considered: direct Chroma calls everywhere; abstract base class; protocol.

Decision rationale: `VectorStoreBase` is small and explicit (see: `pipeline/store/base.py`, lines 17-61).

Trade-offs accepted: no provider factory yet, so YAML `store.provider` is underused.

Consequences: custom stores can be injected, but config-selected stores need a refactor.

### Decision: Use Redis as optional cache

Context: Embedding repeated text can be expensive.

Options considered: no cache, in-memory cache, Redis cache.

Decision rationale: Redis gives TTL and process-independent reuse. The cache is transparent and fail-open (see: `cache/redis_cache.py`, lines 37-74).

Trade-offs accepted: cache keys omit model configuration.

Consequences: deployments must manage `key_prefix` when changing embedding models.

### Decision: Keep PII detection and redaction separate

Context: Dual mode needs detected spans before it can redact, but embedding can run alongside redaction.

Options considered: one combined detector-redactor class; separate detector/redactor; middleware chain.

Decision rationale: separate classes allow detection first, then concurrent redaction and embedding (see: `pipeline/pii/redactor.py`, lines 8-10; `orchestrator.py`, lines 101-135).

Trade-offs accepted: detector/redactor are concrete, not abstract strategies.

Consequences: current design is simple but less pluggable for non-Presidio detection.

### Decision: Use an async internal orchestrator

Context: Redaction and embedding can overlap, but libraries are synchronous.

Options considered: fully synchronous pipeline; async orchestration with executor; fully async provider APIs.

Decision rationale: `asyncio.gather()` plus `run_in_executor()` gives overlap without replacing libraries (see: `orchestrator.py`, lines 101-135).

Trade-offs accepted: public `ingest()` uses `asyncio.run()`, which is awkward inside existing event loops.

Consequences: future web/server use should add `async_ingest()`.

### Decision: Use factory-based embedder creation

Context: Embedding backends are configurable.

Options considered: direct constructor in pipeline; registry factory; plugin system.

Decision rationale: `create_embedder()` centralizes backend branching and lazy imports (see: `registry.py`, lines 28-55).

Trade-offs accepted: only embedders have a factory; stores and caches do not.

Consequences: adding embedding backends is cleaner than adding store/cache backends.

## What Was Explicitly Not Built

| Capability | Why excluded | What would change |
| --- | --- | --- |
| Document chunking | Current API assumes pre-chunked input. YAGNI for core privacy middleware. | Add splitter before `ChunkAdapter` or support `document=` facade. |
| Public retrieval facade | Store/query primitives exist, but no high-level API. | Add `LexiredactPipeline.retrieve()` using `query_embed()` and `store.query()`. |
| Middleware chain | Fixed pipeline modes are simpler. | Add context object and ordered middleware contract. |
| Multi-store factory | Injection covers custom stores for now. | Add `create_store()` and adapter-specific config. |
| Retry/backoff | Simpler failure semantics. | Add retry policies in cache/store layers. |

## Known Technical Debt

| Area | Problem | Better design |
| --- | --- | --- |
| Public async API | `ingest()` calls `asyncio.run()` (see: `pipeline_api.py`, line 90). | Add `async_ingest()` and have sync wrapper call it only when safe. |
| Store provider config | `StoreConfig.provider` is not used for construction. | Add store factory. |
| PII pluggability | Orchestrator constructs concrete detector/redactor. | Add `DetectorBase` and `RedactorBase` injection. |
| Cache key versioning | Key excludes model/config. | Include model name, prefixes, dimension, normalization in key namespace. |
| Tests after rename | Tests import `lexiredact`. | Rename imports and add focused unit tests. |
| Root API inconsistency | `Chunk` doc says internal but root exports it. | Decide whether `Chunk` is public and update docs/API. |

## Future Evolution

For streaming documents: add a chunk generator and make orchestrator process iterable batches without holding all vectors in memory.

For multi-tenancy: add tenant ID to config/request context, metadata filters, cache prefixes, and collection naming.

For 100x more documents: move from local Chroma to a server vector DB, add explicit connection management, retries, batching limits, and observability metrics.

For new PII detector classes: introduce detector/redactor base contracts and factories, then pass them through the pipeline constructor like embedders and stores.

## Breaking Change Analysis

```text
High stability:
  load_config()
  LexiredactPipeline.__init__()
  LexiredactPipeline.ingest()
  EmbedderBase
  VectorStoreBase
  ProcessingResult.to_dict()

Internal but dangerous:
  Orchestrator._run_dual()
  PIIRedactor.redact()
  ChromaStore.upsert_batch()
```

Changing stable contracts requires versioning and migration notes. Changing internal dangerous methods requires privacy regression tests.

