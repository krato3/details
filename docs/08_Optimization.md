# Optimization And Runtime Performance

| Field | Value |
| --- | --- |
| Purpose | Explain performance-related implementation choices and trade-offs. |
| Audience | Developers tuning throughput, latency, or memory usage. |
| Related files | [02_Request_Flow.md], [03_Class_Diagram.md], [12_Code_Walkthrough.md] |

## Hot Path Diagram

```text
ingest()
  -> adapt_batch()             cheap CPU
  -> process_batch()
     -> detect_batch()         hot path in privacy modes, model/NLP CPU
     -> cache.get()            cheap if disabled, network if enabled
     -> embed_batch()          hot path, ML CPU/GPU
     -> redact()               medium cost, CPU
     -> upsert_batch()         hot path, disk/DB I/O
```

## async/await

`Orchestrator.process_batch()` and its private mode methods are async (see: `lexiredact/pipeline/orchestrator.py`, lines 63, 90, 180, 247). The public facade is synchronous and calls `asyncio.run()` (see: `lexiredact/pipeline_api.py`, line 90).

Why: the orchestrator uses `asyncio.gather()` in dual mode to overlap redaction and embedding (see: `orchestrator.py`, lines 130-135). The expensive embedder and redactor calls are synchronous, so they are pushed into the executor with `loop.run_in_executor()` (lines 102-121).

Trade-off: this lets synchronous libraries be used without rewriting them, but `pipeline.ingest()` cannot be safely called from an already-running event loop.

## Concurrent Execution

Dual mode executes `redact_all()` and `embed_all()` concurrently:

```text
_run_dual()
  -> detect_batch()
  -> asyncio.gather(
       redact_all() -> run_in_executor(redactor.redact)
       embed_all()  -> cache + run_in_executor(embedder.embed_batch)
     )
  -> store.upsert_batch()
```

Source: `lexiredact/pipeline/orchestrator.py`, lines 101-135.

Benefit: redaction and embedding can overlap after detection. Trade-off: concurrency is coarse-grained and uses the default executor, so thread pool sizing is not explicit.

## Batch Processing

Batching appears in three places:

| Location | Behavior |
| --- | --- |
| `ChunkAdapter.adapt_batch()` | Adapts many raw dicts and collects per-item failures (see: `chunk_adapter.py`, lines 68-93). |
| `PIIDetector.detect_batch()` | Processes chunks in configured sub-batches but still analyzes each chunk individually (see: `detector.py`, lines 52-77). |
| Embedder `_encode()` | Encodes texts in sub-batches of `config.batch_size` (see: `sentence_transformers.py`, lines 152-172; `huggingface.py`, lines 160-195). |

Trade-off: batching improves model throughput, but errors inside model batch calls are not individually isolated.

## Caching

`EmbeddingCache` caches embeddings by text. The key is `{key_prefix}:emb:{sha256(text)[:16]}` (see: `lexiredact/cache/redis_cache.py`, lines 76-79). Values are JSON-serialized vectors with configured TTL via `setex()` (lines 64-72).

Cache hit path:

```text
cache.get(text) -> JSON list -> use vector -> cache_hit=True
```

Cache miss path:

```text
cache.get(text) -> None
  -> embedder.embed_batch(miss_texts)
  -> cache.set(text, vector)
```

Source: `orchestrator.py`, lines 111-128 and 253-269.

Trade-off: text-hash caching is simple, but the key does not include model name, prefix, normalization, or dimension. Changing embedding config without changing `key_prefix` can reuse stale vectors.

## Redis Integration

Redis is optional and lazy. No Redis connection is attempted when cache is disabled (see: `redis_cache.py`, lines 37-40 and 64-67). When enabled, `_ensure_connected()` imports Redis and creates a client with response decoding and 2-second socket timeouts (lines 81-92).

Data structure: simple Redis string per embedding key. Serialization: JSON list of floats.

Trade-off: simple strings are portable and inspectable, but large vectors as JSON are bigger than binary formats.

## Vector Reuse

Vector reuse is exactly the cache hit behavior. Dual and raw modes reuse embeddings for identical original text; preredacted mode bypasses the cache and marks `cache_hit=False` (see: `orchestrator.py`, line 235).

## Lazy Loading

| Component | Lazy behavior | Source |
| --- | --- | --- |
| `SentenceTransformerEmbedder` | Model loads on first embed/query or dimension if unknown. | `sentence_transformers.py`, lines 117-143 |
| `HuggingFaceEmbedder` | Tokenizer/model load on first use. | `huggingface.py`, lines 109-140 |
| `PIIDetector` | Presidio analyzer loads on first detection. | `detector.py`, lines 79-100 |
| `EmbeddingCache` | Redis client connects on first get/set. | `redis_cache.py`, lines 81-98 |

Trade-off: startup can be fast if dimensions are configured, but the first request may pay a large cold-start cost.

## Techniques Not Present

| Technique | Present? | Note |
| --- | --- | --- |
| Memoisation | No | No pure-function memo caches besides Redis. |
| Streaming | No | Inputs are full pre-chunked lists. |
| Memory-managed chunking | No | No document splitter or streaming parser exists. |
| Connection pooling | Partial | Redis/Chroma clients may pool internally; LexiRedact does not manage pools. |
| Retry/backoff | No | Failures are handled or propagated directly. |
| Explicit locks | No | Lazy shared state is unguarded. |

## Memory Management

Persistent in memory across requests: loaded NLP analyzer, loaded embedding model, Redis client, Chroma client/collection, pipeline config. Transient per request: `Chunk` list, detected entities, sanitized texts, vectors, result objects.

The largest transient object is usually `vectors`, especially for large batches. The code processes embedding sub-batches inside the embedder, but the orchestrator still holds all vectors before calling `upsert_batch()` (see: `orchestrator.py`, lines 132-148).

## Thread Safety And Shared State

Shared mutable state exists in `_model`, `_tokenizer`, `_analyzer`, `_client`, and Chroma collection attributes. No `asyncio.Lock` or `threading.Lock` is used. Because public `ingest()` is synchronous, typical single-threaded use is fine, but external concurrent calls against one pipeline instance should be tested carefully.

