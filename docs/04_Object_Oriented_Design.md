# Object-Oriented Design In LexiRedact

| Field | Value |
| --- | --- |
| Purpose | Teach OOP and design patterns using this codebase as the concrete example. |
| Audience | Python developers who know classes but want architectural judgment. |
| Related files | [03_Class_Diagram.md], [11_Extending_LexiRedact.md], [15_Architecture_Decisions.md] |

## Design Map

```text
Facade:              LexiredactPipeline
Abstract contracts:  EmbedderBase, VectorStoreBase
Concrete adapters:   SentenceTransformerEmbedder, HuggingFaceEmbedder, ChromaStore
Composition root:    LexiredactPipeline.__init__()
Runtime coordinator: Orchestrator
Data contracts:      Chunk, DetectedEntity, ProcessingResult, Pydantic config models
Factory:             create_embedder()
```

## Encapsulation

Encapsulation means keeping state and implementation details behind a small public interface. LexiRedact uses leading-underscore attributes heavily: `_config`, `_adapter`, `_orchestrator`, `_model`, `_client`, and `_collection`.

```python
self._config = config
self._adapter = ChunkAdapter(config.input_schema)
self._logger = get_logger("pipeline_api")
```

These lines are in `lexiredact/pipeline_api.py`, lines 48-50. The user receives `LexiredactPipeline.ingest()` as the public method, while construction details stay hidden. The alternative would be forcing users to wire every subsystem manually, which would make basic ingestion brittle.

## Abstraction

Abstraction means depending on what something can do, not how it does it. `EmbedderBase` defines `embed_batch`, `query_embed`, and `get_dimension` without importing any ML library (see: `lexiredact/pipeline/embedder/base.py`, lines 33-95). `VectorStoreBase` similarly defines storage behavior without importing Chroma (see: `lexiredact/pipeline/store/base.py`, lines 17-61).

```text
Orchestrator
  -> EmbedderBase
  -> VectorStoreBase
```

The alternative would be `Orchestrator` calling `SentenceTransformer.encode()` and `chromadb.Collection.upsert()` directly. That would make tests and new backends harder.

## Inheritance And Method Overriding

Inheritance is used only where there is a real contract. `SentenceTransformerEmbedder` and `HuggingFaceEmbedder` inherit `EmbedderBase` and override all abstract methods (see: `sentence_transformers.py`, lines 30-111; `huggingface.py`, lines 35-103). `ChromaStore` inherits `VectorStoreBase` and implements `upsert_batch`, `query`, and `count` (see: `pipeline/store/chroma.py`, lines 22-122).

```text
EmbedderBase
  -> SentenceTransformerEmbedder
  -> HuggingFaceEmbedder

VectorStoreBase
  -> ChromaStore
```

The design avoids inheritance for the orchestrator because modes are not separate subclasses; they are private methods selected by config.

## Polymorphism

Polymorphism means the same call works against different concrete types. The orchestrator calls `self._embedder.embed_batch(...)` and `self._store.upsert_batch(...)` without knowing the concrete classes (see: `lexiredact/pipeline/orchestrator.py`, lines 120-125 and 140-148).

| Contract | Concrete options |
| --- | --- |
| `EmbedderBase` | `SentenceTransformerEmbedder`, `HuggingFaceEmbedder`, custom examples |
| `VectorStoreBase` | `ChromaStore`, custom examples |

This is why examples can pass custom classes into `LexiredactPipeline(config, embedder=..., store=...)` (see: `examples/custom_embedder.py`, lines 38-45; `examples/custom_vectorstore.py`, lines 54-61).

## Composition Over Inheritance

Composition means an object owns or receives collaborators. `LexiredactPipeline` composes `ChunkAdapter`, embedder, store, cache, and orchestrator instead of inheriting from them (see: `lexiredact/pipeline_api.py`, lines 48-57). `Orchestrator` composes detector/redactor and aggregates injected embedder/store/cache (see: `pipeline/orchestrator.py`, lines 55-61).

```text
LexiredactPipeline *-- Orchestrator *-- PIIDetector
LexiredactPipeline *-- EmbeddingCache
Orchestrator o-- EmbedderBase
Orchestrator o-- VectorStoreBase
```

Inheritance would be worse here because a pipeline is not a kind of store, detector, or cache.

## Dependency Injection

Constructor injection is used for custom embedders and stores:

```python
def __init__(self, config, embedder=None, store=None):
    _embedder = embedder or create_embedder(config.embedder)
    _store = store or ChromaStore(config.store, _embedder.get_dimension())
```

See `lexiredact/pipeline_api.py`, lines 42-55. This lets tests and examples inject lightweight fakes. Setter injection is not used, which is good because changing core collaborators after construction could break lifecycle assumptions. A service locator is not used, avoiding hidden global dependencies.

## Factory Pattern

`create_embedder(config)` is a factory: it centralizes object creation based on configuration. It checks `config.backend` and returns the matching concrete embedder (see: `lexiredact/pipeline/embedder/registry.py`, lines 28-55).

```text
EmbedderConfig.backend
  -> "sentence_transformers" -> SentenceTransformerEmbedder
  -> "huggingface"           -> HuggingFaceEmbedder
```

The alternative is scattering `if backend == ...` through the pipeline and CLI.

## Adapter Pattern

There are two adapter meanings. `ChunkAdapter` adapts caller dictionaries to internal `Chunk` objects (see: `lexiredact/adapters/chunk_adapter.py`, lines 20-93). `ChromaStore` adapts LexiRedact's vector-store contract to ChromaDB's API (see: `pipeline/store/chroma.py`, lines 53-108).

The adapter pattern isolates external shape changes. If Chroma changes `collection.query()`, only `ChromaStore.query()` should change.

## Facade Pattern

`LexiredactPipeline` is a facade. It hides configuration-driven construction and internal async orchestration behind `ingest()` (see: `lexiredact/pipeline_api.py`, lines 32-90).

```text
User -> LexiredactPipeline.ingest()
     -> adapter + orchestrator + detector + redactor + embedder + cache + store
```

Without the facade, users would need to know too much about internal order.

## Template Method

No strict template-method base class exists. The nearest shape is `Orchestrator.process_batch()`, which provides a skeleton dispatch and delegates to mode-specific private methods (see: `pipeline/orchestrator.py`, lines 63-88). Because there are no subclasses filling hooks, this is not a classic template method.

## Strategy Pattern

Embedding backends are strategies: both implement `EmbedderBase`, and `create_embedder()` selects one from config (see: `pipeline/embedder/base.py`, lines 33-95; `pipeline/embedder/registry.py`, lines 40-48). PII detection is less strategic: `PIIDetector` supports different Presidio NLP engines through config, but there is no `DetectorBase` interface.

## Middleware Pattern

No middleware chain exists. There is no `next()` function, no ordered list, and no registration API. The fixed orchestrator flow replaces middleware today:

```text
process_batch()
  -> _run_dual()
  -> fixed detector/redactor/embedder/store order
```

For a Django/WSGI-style middleware system, a future design would wrap `process_batch(context, next)` before storage.

## Builder And Singleton

No builder pattern is used. Pydantic config models handle structured construction directly. No intentional singleton exists. Logging uses the standard library logger registry, but no LexiRedact class enforces singleton lifecycle (see: `lexiredact/app_logging.py`, lines 16-57).

## SOLID

| Principle | Demonstrated by | Partial gap |
| --- | --- | --- |
| Single Responsibility | `ChunkAdapter` only adapts/validates input (see: `chunk_adapter.py`, lines 20-93). | `Orchestrator` coordinates many concerns; acceptable because it is the workflow owner. |
| Open/Closed | `EmbedderBase` allows new embedders without changing orchestrator. | Adding a store provider through config requires code; no store factory exists. |
| Liskov Substitution | Custom stores can replace `ChromaStore` if they honor `VectorStoreBase`. | Contracts are documented, not runtime-enforced beyond method presence. |
| Interface Segregation | `EmbedderBase` and `VectorStoreBase` are small. | `VectorStoreBase` includes `query()` even though public pipeline does not expose retrieval. |
| Dependency Inversion | Orchestrator depends on `EmbedderBase` and `VectorStoreBase`. | It constructs concrete `PIIDetector` and `PIIRedactor` internally (see: `orchestrator.py`, lines 56-57). |

## DRY, KISS, YAGNI

| Principle | Example | Comment |
| --- | --- | --- |
| DRY | Embedder creation is centralized in `create_embedder()` (see: `registry.py`, lines 28-55). | Avoids repeated backend branching. |
| KISS | Cache failure returns miss/no-op (see: `redis_cache.py`, lines 37-74). | Keeps pipeline behavior simple under Redis failure. |
| YAGNI | No middleware or document chunker exists. | The package focuses on pre-chunked ingestion. |

