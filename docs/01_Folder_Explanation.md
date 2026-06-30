# LexiRedact Folder Explanation

| Field | Value |
| --- | --- |
| Purpose | Explain where every responsibility lives and how folders depend on each other. |
| Audience | Developers navigating the repository for the first time. |
| Related files | [00_Project_Overview.md], [05_Main_Files.md], [15_Architecture_Decisions.md] |

## Repository Map

```text
.
|-- lexiredact/       core library package
|-- examples/         runnable usage examples
|-- tests/            showcase tests, currently stale under old lexiredact imports
|-- backend/          dashboard/system monitoring support
|-- frontend/         Streamlit UI support
|-- eval/             evaluation datasets, metrics, and reports
|-- chroma_db/        local Chroma persistence data
|-- docs/             this handbook
|-- pyproject.toml    package metadata and dependencies
|-- lexiredact_config.yaml sample runtime config
```

## `lexiredact/`

Purpose: The installable library package and public API.

| File | Contains |
| --- | --- |
| `__init__.py` | Package-root re-exports: config, pipeline facade, models, exceptions, logging helpers. |
| `pipeline_api.py` | `LexiredactPipeline`, the public ingestion facade. |
| `exceptions.py` | Custom exception hierarchy. |
| `app_logging.py` | Library logging configuration helpers. |
| `cli.py` | Click-based command-line interface. |
| `README.md` | Package-level README text. |

Public interface: `__init__.py` exports `LexiredactPipeline`, `load_config`, `LexiredactConfig`, `ProcessingResult`, `DetectedEntity`, `Chunk`, custom exceptions, `configure_logging`, and `get_logger` (see: `lexiredact/__init__.py`, lines 10-41).

Dependencies: imports `config`, `pipeline_api`, `exceptions`, `app_logging`, and `models` for re-export. `pipeline_api.py` imports almost every core subsystem because it wires the facade (see: `lexiredact/pipeline_api.py`, lines 20-29).

Dependents: examples, CLI, and users import from this folder. Internal modules should generally avoid importing the package root to reduce circular import risk; `cli.py` does import from `lexiredact` because it is an outer adapter (see: `lexiredact/cli.py`, line 25).

Forbidden dependencies: lower-level folders such as `models`, `config`, and `pipeline/store` should not import from `lexiredact/__init__.py`; doing so would pull the facade and concrete infrastructure into foundational layers.

## `lexiredact/config/`

Purpose: Define and load runtime configuration.

| File | Contains |
| --- | --- |
| `__init__.py` | Folder description only; no re-exports. |
| `schema.py` | Pydantic models: `InputSchemaConfig`, `PIIConfig`, `EmbedderConfig`, `CacheConfig`, `StoreConfig`, `LexiredactConfig`. |
| `loader.py` | `load_config()` and validation error formatting. |

Public interface: the package root re-exports `load_config` and `LexiredactConfig` (see: `lexiredact/__init__.py`, lines 10-12).

Dependencies: `loader.py` imports `yaml`, Pydantic `ValidationError`, config schema, exceptions, and logging (see: `lexiredact/config/loader.py`, lines 10-20). `schema.py` imports Pydantic and `Literal` (see: `lexiredact/config/schema.py`, lines 9-12).

Dependents: adapters use `InputSchemaConfig`; cache uses `CacheConfig`; PII uses `PIIConfig`; embedders use `EmbedderConfig`; store uses `StoreConfig`; facade/orchestrator use `LexiredactConfig`.

Forbidden dependencies: `config/` must not import concrete pipeline implementations, Chroma, Redis, or Presidio. Configuration must remain declarative and cheap to import.

## `lexiredact/adapters/`

Purpose: Convert caller-supplied dictionaries into internal `Chunk` objects.

| File | Contains |
| --- | --- |
| `__init__.py` | Folder description only. |
| `chunk_adapter.py` | `ChunkAdapter` with `adapt()` and `adapt_batch()`. |

Public interface: no root export for `ChunkAdapter`; it is used by `LexiredactPipeline` (see: `lexiredact/pipeline_api.py`, line 49).

Dependencies: imports `InputSchemaConfig`, `LexiredactInputError`, logging, and `Chunk` (see: `lexiredact/adapters/chunk_adapter.py`, lines 12-15).

Dependents: `pipeline_api.py` uses it before calling the orchestrator.

Forbidden dependencies: adapters should not import PII, embedder, cache, or vector store code. Input adaptation must stay separate from processing.

## `lexiredact/models/`

Purpose: Define data objects that move through or out of the pipeline.

| File | Contains |
| --- | --- |
| `__init__.py` | Folder description only. |
| `chunk.py` | Internal `Chunk` dataclass with empty-text validation. |
| `result.py` | Public `DetectedEntity` and `ProcessingResult` dataclasses. |

Public interface: `Chunk`, `DetectedEntity`, and `ProcessingResult` are exported from the package root (see: `lexiredact/__init__.py`, lines 21-32). Note that `chunk.py` says `Chunk` is not public, but `__init__.py` exports it; this is a small public API inconsistency.

Dependencies: `chunk.py` imports `LexiredactInputError`; `result.py` imports dataclasses and `Any` (see: `models/chunk.py`, lines 10-13; `models/result.py`, lines 7-10).

Dependents: adapter creates `Chunk`; detector reads `Chunk`; redactor receives `DetectedEntity`; orchestrator returns `ProcessingResult`.

Forbidden dependencies: models should never import pipeline, cache, store, or config. They are foundational contracts.

## `lexiredact/pipeline/`

Purpose: Coordinate the core privacy-preserving ingestion flow.

| File | Contains |
| --- | --- |
| `__init__.py` | Folder description only. |
| `orchestrator.py` | `Orchestrator`, the runtime coordinator for modes `dual`, `preredacted`, and `raw`. |
| `embedder/` | Embedding interfaces and implementations. |
| `pii/` | PII detection and redaction. |
| `store/` | Vector database abstraction and Chroma implementation. |

Public interface: the orchestrator is internal; users enter through `LexiredactPipeline`.

Dependencies: orchestrator imports cache, config, exceptions, logging, models, embedder base, PII classes, and vector store base (see: `pipeline/orchestrator.py`, lines 21-33).

Dependents: `pipeline_api.py` constructs and calls the orchestrator.

Forbidden dependencies: the orchestrator can depend on subsystems, but concrete lower-level subsystems should not import it. That prevents processing policy from leaking into infrastructure.

## `lexiredact/pipeline/embedder/`

Purpose: Hide embedding model choices behind a stable contract.

| File | Contains |
| --- | --- |
| `__init__.py` | Re-exports `EmbedderBase`, concrete embedders, and `create_embedder`. |
| `base.py` | Abstract `EmbedderBase`. |
| `registry.py` | `create_embedder()` factory. |
| `sentence_transformers.py` | Lazy `SentenceTransformerEmbedder`. |
| `huggingface.py` | Lazy `HuggingFaceEmbedder` using transformers and mean pooling. |
| `default.py` | Backward-compatible alias `DefaultEmbedder`. |

Public interface: `EmbedderBase`, `SentenceTransformerEmbedder`, `HuggingFaceEmbedder`, and `create_embedder` are exported by `embedder/__init__.py` (see: lines 27-37).

Dependencies: base has only `abc`; concrete classes import config and logging and defer heavy ML imports until load time (see: `sentence_transformers.py`, lines 117-134; `huggingface.py`, lines 109-130).

Dependents: registry, pipeline facade, orchestrator, examples, CLI inspect/stats commands.

Forbidden dependencies: embedders should not import PII, cache, or vector stores. They transform text into vectors only.

## `lexiredact/pipeline/pii/`

Purpose: Detect and redact PII while keeping detection separate from replacement.

| File | Contains |
| --- | --- |
| `__init__.py` | Re-exports `PIIDetector` and `PIIRedactor`. |
| `detector.py` | Lazy Presidio `AnalyzerEngine` wrapper. |
| `engine_factory.py` | Presidio NLP engine factory. |
| `redactor.py` | Presidio anonymizer wrapper. |

Public interface: `PIIDetector` and `PIIRedactor` via `pii/__init__.py` (see: lines 12-15).

Dependencies: detector imports `PIIConfig`, logging, `Chunk`, and `DetectedEntity`; factory imports `PIIConfig`; redactor imports logging and `DetectedEntity`.

Dependents: orchestrator owns one detector and one redactor (see: `pipeline/orchestrator.py`, lines 56-57).

Forbidden dependencies: PII code should not import store or cache. Detection/redaction must not know how vectors are persisted.

## `lexiredact/pipeline/store/`

Purpose: Isolate vector database details behind `VectorStoreBase`.

| File | Contains |
| --- | --- |
| `__init__.py` | Re-exports `VectorStoreBase` and `ChromaStore`. |
| `base.py` | Abstract vector store contract. |
| `chroma.py` | ChromaDB persistent local implementation. |

Public interface: `VectorStoreBase` and `ChromaStore` (see: `pipeline/store/__init__.py`, lines 20-23).

Dependencies: base imports only `abc` and `Any`; Chroma imports store config, storage exception, logging, and base class (see: `pipeline/store/chroma.py`, lines 12-17).

Dependents: pipeline facade creates `ChromaStore`; orchestrator receives `VectorStoreBase`; CLI uses Chroma directly for inspection.

Forbidden dependencies: store adapters must never import PII or orchestrator. A database driver should not know application privacy policy; it should only store the metadata it is handed.

## `lexiredact/cache/`

Purpose: Provide optional Redis embedding cache.

| File | Contains |
| --- | --- |
| `__init__.py` | Re-exports `EmbeddingCache`. |
| `redis_cache.py` | `EmbeddingCache`, SHA-256 keying, lazy Redis client, JSON serialization. |

Public interface: `EmbeddingCache` via `cache/__init__.py` (see: lines 9-11).

Dependencies: imports `hashlib`, `json`, `CacheConfig`, `LexiredactCacheError`, and logging; imports Redis lazily inside `_ensure_connected()` (see: `cache/redis_cache.py`, lines 12-18 and 81-98).

Dependents: pipeline facade creates it; orchestrator calls `get()` and `set()`.

Forbidden dependencies: cache should not import embedders or vector stores. It caches vectors but does not generate or persist them.

## Other Top-Level Folders

| Folder | Purpose | Notes |
| --- | --- | --- |
| `examples/` | Usage recipes for custom embedders, custom stores, Redis config, quickstart. | These examples use the real public API. |
| `tests/` | Showcase tests and report generation. | Tests currently import `lexiredact`, not `lexiredact`, so they appear stale after rename. |
| `backend/` | Monitoring helpers. | Outside core library. |
| `frontend/` | Streamlit UI components/pages/services. | Outside core library. |
| `eval/` | Evaluation datasets, runners, metrics, reports. | Outside core library. |
| `chroma_db/` | Chroma persistence files. | Runtime data, not source. |

## Folder-Level Dependency Graph

```text
lexiredact/__init__
  -> config
  -> pipeline_api
  -> exceptions
  -> app_logging
  -> models

pipeline_api
  -> adapters
  -> cache
  -> config
  -> models
  -> pipeline/embedder
  -> pipeline/store
  -> pipeline/orchestrator

pipeline/orchestrator
  -> cache
  -> config
  -> exceptions
  -> models
  -> pipeline/embedder/base
  -> pipeline/pii
  -> pipeline/store/base

adapters -> config, exceptions, app_logging, models
cache -> config, exceptions, app_logging, redis(lazy)
pipeline/pii -> config, app_logging, models, presidio(lazy)
pipeline/embedder -> config, app_logging, ML libraries(lazy)
pipeline/store -> config, exceptions, app_logging, chromadb(lazy in constructor)
models -> exceptions
config -> exceptions, app_logging, pydantic, yaml
```

## Import Hierarchy And Circular Risk

```text
Lowest level: exceptions, app_logging
Next: config/schema, models
Next: adapters, cache, embedder/base, store/base
Next: concrete PII/embedder/store implementations
Next: orchestrator
Top: pipeline_api, cli, package root
```

The main circular import risk is importing the package root from internal modules. Most modules avoid that. The CLI is allowed to import root symbols because it is an outer entry point (see: `lexiredact/cli.py`, line 25).

