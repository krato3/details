# Error Handling

| Field | Value |
| --- | --- |
| Purpose | Explain how failures are represented, caught, logged, or propagated. |
| Audience | Developers debugging failures or adding new error paths. |
| Related files | [06_YAML_System.md], [12_Code_Walkthrough.md], [13_FAQs.md] |

## Exception Hierarchy

```text
Exception
  -> LexiredactError
     -> LexiredactConfigError
     -> LexiredactInputError
     -> LexiredactStorageError
     -> LexiredactCacheError
```

All custom exceptions live in `lexiredact/exceptions.py`, lines 11-57. `LexiredactError` stores `message` and optional `context`, and `__str__()` appends context key-values (lines 19-28).

| Exception | Raised when | Propagation |
| --- | --- | --- |
| `LexiredactConfigError` | Bad config source, YAML, validation, unsupported backend/engine. | Propagates to caller/CLI. |
| `LexiredactInputError` | Missing ID/text field or empty text. | Collected by `adapt_batch()` for per-item skip. |
| `LexiredactStorageError` | Chroma init/upsert/query/count failure. | Propagates; storage loss is not silent. |
| `LexiredactCacheError` | Redis init or invalid cached value internally. | Caught inside cache; never surfaces. |

## Try/Except Inventory

| Location | Attempt | Catches | Behavior |
| --- | --- | --- | --- |
| `config/loader.py`, lines 46-52 | Read YAML file | `FileNotFoundError` | Raise `LexiredactConfigError`. |
| `config/loader.py`, lines 53-59 | Parse YAML | `yaml.YAMLError` | Raise `LexiredactConfigError`. |
| `config/loader.py`, lines 72-78 | Pydantic validation | `ValidationError` | Raise `LexiredactConfigError`. |
| `chunk_adapter.py`, lines 80-85 | Adapt one dict | `LexiredactInputError` | Log warning, collect failed item. |
| `redis_cache.py`, lines 42-62 | Redis get/decode | cache/json/any error | Log warning, return `None`. |
| `redis_cache.py`, lines 69-74 | Redis set | any error | Log warning, no-op. |
| `redis_cache.py`, lines 85-98 | Redis client init | any error | Raise `LexiredactCacheError` to caller inside same module. |
| `detector.py`, lines 108-116 | Presidio analyze | any error | Log warning, return empty entity list for that chunk. |
| `engine_factory.py`, lines 53-60 | Import Presidio engine provider | `ImportError` | Raise `LexiredactConfigError`. |
| `engine_factory.py`, lines 84-91 | Create NLP engine | any error | Raise `LexiredactConfigError`. |
| `redactor.py` | No catch | Presidio anonymizer errors | Propagate. |
| `orchestrator.py`, lines 140-151 | Store in dual | `LexiredactStorageError` | Re-raise. |
| `orchestrator.py`, lines 204-217 | Store in preredacted | `LexiredactStorageError` | Re-raise. |
| `orchestrator.py`, lines 272-282 | Store in raw | `LexiredactStorageError` | Re-raise. |
| `store/chroma.py`, lines 33-51 | Chroma init | any error | Raise `LexiredactStorageError`. |
| `store/chroma.py`, lines 64-72 | Chroma upsert | any error | Raise `LexiredactStorageError`. |
| `store/chroma.py`, lines 88-108 | Chroma query | any error | Raise `LexiredactStorageError`. |
| `store/chroma.py`, lines 116-122 | Chroma count | any error | Raise `LexiredactStorageError`. |
| `cli.py`, multiple commands | Command body | `LexiredactError` | Print error and `sys.exit(1)`. |

## Validation Errors

User config is validated at startup by Pydantic and wrapped by `load_config()` (see: `config/loader.py`, lines 72-78). User chunk input is validated in `ChunkAdapter.adapt()` and `Chunk.__post_init__()` (see: `chunk_adapter.py`, lines 39-66; `models/chunk.py`, lines 30-40). Invalid chunks are skipped rather than aborting the whole batch (see: `chunk_adapter.py`, lines 80-93).

## Fallback Behavior

| Component | Trigger | Fallback | Reported? |
| --- | --- | --- | --- |
| Cache get | Redis/decode/cache error | Treat as miss. | Warning log. |
| Cache set | Redis error | No-op. | Warning log. |
| PII single chunk | Presidio analyze error | Empty entity list. | Warning log. |
| HuggingFace device move | `.to(device)` fails | Continue on CPU-ish model state. | Warning log (see: `huggingface.py`, lines 142-151). |

There is no fallback embedder, no fallback vector store, and no retry mechanism.

## Retry Mechanisms

No retry logic exists for Redis, Chroma, Presidio, or model calls. Redis uses 2-second socket timeouts when the client is created (see: `cache/redis_cache.py`, lines 87-92). If production retry is needed, add it behind cache/store abstractions so the orchestrator stays simple.

## Logging And Observability

Logging is namespaced under `lexiredact`. A `NullHandler` is installed at import time so library users do not see warnings unless they configure logging (see: `app_logging.py`, lines 16-17). `configure_logging(level)` adds one stream handler and avoids duplicates (lines 22-44). `get_logger(name)` returns `logging.getLogger(f"lexiredact.{name}")` (lines 47-57).

Useful log points:

| Level | Location | Message purpose |
| --- | --- | --- |
| INFO | `pipeline_api.py`, lines 59-64 | Pipeline ready and selected backend/mode. |
| INFO | `orchestrator.py`, lines 84-87 | Batch completed. |
| INFO | `detector.py`, lines 46-50 and 92-100 | PII detector/analyzer lifecycle. |
| WARNING | `chunk_adapter.py`, lines 83-90 | Invalid input chunks. |
| WARNING | `redis_cache.py`, lines 54-74 | Cache failures. |
| WARNING | `detector.py`, lines 114-116 | Detection failure for one chunk. |

## Error Propagation

```text
Config error -> load_config() -> caller/CLI
Input error -> ChunkAdapter.adapt_batch() -> warning + skipped item
Cache error -> EmbeddingCache -> warning + miss/no-op
Detection error for one chunk -> PIIDetector -> warning + []
Storage error -> ChromaStore -> Orchestrator re-raises -> pipeline.ingest() caller
```

The core design is selective: failures that threaten correctness of persistence propagate; optional acceleration failures do not.

