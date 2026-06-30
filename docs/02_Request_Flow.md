# LexiRedact Request Flow

| Field | Value |
| --- | --- |
| Purpose | Trace runtime execution for ingestion, retrieval-level primitives, config loading, and cache behavior. |
| Audience | Developers debugging or extending request execution. |
| Related files | [00_Project_Overview.md], [03_Class_Diagram.md], [12_Code_Walkthrough.md] |

## Important API Reality

The requested example `await pipeline.ingest(document="...")` is not implemented. The actual API is synchronous and accepts a list of already chunked dictionaries:

```python
from lexiredact import LexiredactPipeline, load_config

config = load_config("lexiredact_config.yaml")
pipeline = LexiredactPipeline(config)
result = pipeline.ingest([{"id": "doc-1", "text": "..."}])
```

This is visible in `LexiredactPipeline.ingest(self, raw_chunks: list[dict[str, Any]])` (see: `lexiredact/pipeline_api.py`, lines 66-90). There is no public `retrieve()` method on `LexiredactPipeline`; retrieval primitives exist as `EmbedderBase.query_embed()` and `VectorStoreBase.query()` (see: `lexiredact/pipeline/embedder/base.py`, lines 64-81; `lexiredact/pipeline/store/base.py`, lines 40-52).

## Part A: Ingestion Flow

```text
Latency hot path:

load_config()
  -> LexiredactConfig validation
LexiredactPipeline.__init__()
  -> create_embedder()
  -> embedder.get_dimension()          [may load model if dimension is None]
  -> ChromaStore.__init__()            [opens/creates persistent collection]
  -> Orchestrator.__init__()
LexiredactPipeline.ingest()
  -> ChunkAdapter.adapt_batch()
  -> asyncio.run(Orchestrator.process_batch())
     -> _run_dual() | _run_preredacted() | _run_raw()
        -> PIIDetector.detect_batch()  [critical path in privacy modes]
        -> cache.get()                 [cache branch]
        -> embedder.embed_batch()      [critical path]
        -> PIIRedactor.redact()        [critical path in privacy modes]
        -> store.upsert_batch()        [critical path]
```

### Step 1: Load Config

`load_config()` receives either a dict, string path, or `Path` (see: `lexiredact/config/loader.py`, lines 31-43). If the input is a path, it reads UTF-8 text, parses YAML with `yaml.safe_load`, requires a top-level mapping, and constructs `LexiredactConfig` (lines 44-81).

Failure behavior:

| Failure | Raised as | Source |
| --- | --- | --- |
| Missing file | `LexiredactConfigError` | `config/loader.py`, lines 46-52 |
| YAML parse error | `LexiredactConfigError` | `config/loader.py`, lines 53-59 |
| Top-level YAML is not mapping | `LexiredactConfigError` | `config/loader.py`, lines 60-64 |
| Pydantic validation error | `LexiredactConfigError` | `config/loader.py`, lines 72-78 |

### Step 2: Construct Pipeline

```python
pipeline = LexiredactPipeline(config)
```

The constructor stores config, creates `ChunkAdapter`, gets a logger, creates an embedder through `create_embedder()`, creates `ChromaStore`, creates `EmbeddingCache`, and passes all dependencies into `Orchestrator` (see: `lexiredact/pipeline_api.py`, lines 42-57).

Side effects:

| Component | Side effect |
| --- | --- |
| `create_embedder()` | Creates object only; concrete models are lazy (see: `pipeline/embedder/registry.py`, lines 28-55). |
| `_embedder.get_dimension()` | May load model when `config.embedder.dimension` is `None` (see: `sentence_transformers.py`, lines 98-111). |
| `ChromaStore()` | Creates/opens persistent Chroma collection (see: `pipeline/store/chroma.py`, lines 30-43). |
| `EmbeddingCache()` | Does not connect to Redis yet (see: `cache/redis_cache.py`, lines 32-35). |
| `Orchestrator()` | Creates `PIIDetector` and `PIIRedactor` (see: `pipeline/orchestrator.py`, lines 48-61). |

### Step 3: Adapt Input

```python
results = pipeline.ingest([{"id": "doc-1", "text": "..."}])
```

`LexiredactPipeline.ingest()` calls `self._adapter.adapt_batch(raw_chunks)` (see: `lexiredact/pipeline_api.py`, line 79). `adapt_batch()` loops through raw dicts, calls `adapt()`, collects successful `Chunk` objects, and collects per-item failures without raising them (see: `lexiredact/adapters/chunk_adapter.py`, lines 68-93).

Validation:

| Check | Source | Failure |
| --- | --- | --- |
| ID field exists | `chunk_adapter.py`, lines 39-46 | `LexiredactInputError` collected by `adapt_batch()` |
| Text field exists | `chunk_adapter.py`, lines 50-58 | `LexiredactInputError` collected by `adapt_batch()` |
| Text not blank | `models/chunk.py`, lines 30-40 | `LexiredactInputError` collected by `adapt_batch()` |

If all chunks fail, ingestion logs a warning and returns `[]` (see: `lexiredact/pipeline_api.py`, lines 84-86).

### Step 4: Enter Async Orchestrator

The public method is synchronous but internal orchestration is async:

```python
return asyncio.run(self._orchestrator.process_batch(chunks))
```

This line is in `lexiredact/pipeline_api.py`, line 90. It means calling `pipeline.ingest()` from inside an already-running event loop can raise Python's standard `RuntimeError`. The codebase does not expose an async public alternative.

### Step 5: Mode Dispatch

`Orchestrator.process_batch()` reads `self._config.pipeline_mode` and dispatches to `_run_dual`, `_run_preredacted`, or `_run_raw` (see: `lexiredact/pipeline/orchestrator.py`, lines 63-88).

| Mode | Behavior | Source |
| --- | --- | --- |
| `dual` | Detect PII, concurrently redact and embed original, store sanitized text. | `orchestrator.py`, lines 90-178 |
| `preredacted` | Detect, redact, embed sanitized, store sanitized. | `orchestrator.py`, lines 180-245 |
| `raw` | Embed original, store original. | `orchestrator.py`, lines 247-306 |

### Step 6: Dual Mode Detailed Path

Plain-language summary: dual mode tries to preserve retrieval quality by embedding original text while still storing sanitized text. The original text is not written to Chroma metadata in this mode (see: `lexiredact/pipeline/orchestrator.py`, lines 137-148).

```text
_run_dual(chunks)
  -> PIIDetector.detect_batch(chunks)
     -> PIIDetector._ensure_loaded()
        -> build_nlp_engine(config.pii)
        -> AnalyzerEngine(...)
     -> PIIDetector._detect_single(chunk)
  -> asyncio.gather(redact_all(), embed_all())
     -> redact_all()
        -> run_in_executor(...)
           -> PIIRedactor.redact(text, entities)
     -> embed_all()
        -> EmbeddingCache.get(text)
        -> run_in_executor(...)
           -> EmbedderBase.embed_batch(miss_texts)
        -> EmbeddingCache.set(text, vector)
  -> VectorStoreBase.upsert_batch(ids, vectors, metadatas)
  -> ProcessingResult(...)
```

The detector builds its Presidio analyzer lazily on first detection (see: `pipeline/pii/detector.py`, lines 79-100). Each chunk is analyzed with configured entities and language, then filtered by score and entity list (lines 108-133). Detection errors for one chunk are logged and produce an empty entity list, not a failed request (lines 114-116).

Redaction is executed in an executor through the local `redact_all()` coroutine (see: `orchestrator.py`, lines 101-109). Embedding also runs in an executor after cache checks (lines 111-128). Both are awaited through `asyncio.gather()` (lines 130-135).

Storage receives IDs, vectors, and metadata where `"text"` is the sanitized text (see: `orchestrator.py`, lines 140-148). If storage raises `LexiredactStorageError`, the orchestrator re-raises it (lines 149-151).

### Step 7: Return Value

Each successful chunk produces a `ProcessingResult` containing chunk ID, sanitized text, detected entities, storage flag, latency, cache hit, mode, and stage latencies (see: `models/result.py`, lines 41-88). Dual mode constructs these results at `orchestrator.py`, lines 162-178.

## Part B: Retrieval Flow

There is no implemented public retrieval facade. The available retrieval primitive flow is:

```text
caller
  -> embedder.query_embed([query])
  -> store.query(query_vector, top_k, filters)
  -> list[{"id": str, "metadata": dict, "distance": float}]
```

`EmbedderBase.query_embed()` is part of the contract because query prefixes may differ from document prefixes (see: `pipeline/embedder/base.py`, lines 64-81). `SentenceTransformerEmbedder.query_embed()` applies `config.query_prefix` and encodes text (see: `pipeline/embedder/sentence_transformers.py`, lines 78-96). `ChromaStore.query()` builds Chroma query args, optionally adds `where` filters, calls collection query, and reshapes Chroma's nested result into a list of dictionaries (see: `pipeline/store/chroma.py`, lines 74-108).

Failure behavior: Chroma query errors are wrapped in `LexiredactStorageError` (see: `pipeline/store/chroma.py`, lines 104-108).

## Part C: Configuration Loading Sub-Flow

```text
load_config(source)
  -> match source
     -> dict: use directly
     -> str|Path: Path(source).read_text()
        -> yaml.safe_load(text)
        -> require dict
     -> other: LexiredactConfigError
  -> LexiredactConfig(**raw)
     -> nested Pydantic models
     -> PIIConfig._resolve_nlp_model()
  -> return config
```

Defaults live in `config/schema.py`, lines 14-112. Unknown keys are rejected because every config model uses `ConfigDict(extra="forbid")` (see: lines 14-15, 22-23, 59-60, 79-80, 88-89, 105).

Configuration precedence is simple:

| Source | Precedence |
| --- | --- |
| Pydantic defaults | Lowest |
| YAML/dict values passed to `load_config()` | Override defaults |
| CLI overrides | Highest inside CLI only, via `model_copy(update=...)` (see: `cli.py`, lines 94-103) |

No environment variable is used by `load_config()`.

## Part D: Middleware Execution Sub-Flow

No middleware chain is implemented. There are no middleware classes, no registration, no ordering list, and no `next()` callback. The closest equivalent is the orchestrator's fixed mode dispatch:

```text
process_batch()
  -> _run_dual()
  -> _run_preredacted()
  -> _run_raw()
```

If a future middleware system is added, it should wrap `Orchestrator.process_batch()` or the public facade without making store/cache/embedder implementations import application-level middleware.

## Part E: Cache Interaction Sub-Flow

```text
embed_all()
  -> texts = [chunk.text ...]
  -> cached_vectors = [cache.get(text) ...]
  -> miss_indices = indexes where vector is None
  +-- all hit:
  |     -> vectors = cached_vectors
  |     -> cache_hits = [True, ...]
  +-- some miss:
        -> embedder.embed_batch(miss_texts)
        -> cache.set(text, vector) for each miss
        -> vectors = cached + new
```

Dual mode cache code is in `orchestrator.py`, lines 111-128. Raw mode has the same branch in lines 253-269. Preredacted mode does not use the cache and sets `cache_hit=False` because the implementation comments that sanitized text has a different hash and cache is not useful (see: `orchestrator.py`, lines 197-201 and 235).

Cache internals:

| Operation | Source | Behavior |
| --- | --- | --- |
| Disabled get | `redis_cache.py`, lines 37-40 | returns `None` |
| Key creation | `redis_cache.py`, lines 76-79 | `{prefix}:emb:{sha256(text)[:16]}` |
| Lazy Redis connect | `redis_cache.py`, lines 81-98 | creates Redis client on first use |
| JSON decode failure | `redis_cache.py`, lines 57-59 | warning then miss |
| Redis failure | `redis_cache.py`, lines 60-62 and 73-74 | warning then miss/no-op |

## Request-Scoped Dependency Graph

```text
ingest request
  pipeline_api
    -> adapters
       -> models
    -> pipeline/orchestrator
       -> pipeline/pii
       -> pipeline/embedder
       -> cache
       -> pipeline/store
       -> models
```

