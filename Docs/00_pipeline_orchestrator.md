# LexiRedact Pipeline Orchestrator

Most important file: `lexiredact/pipeline/orchestrator.py`

This file is the core of LexiRedact. It decides how every input chunk moves through PII detection, redaction, embedding, caching, and vector storage. The public API prepares the data, but the orchestrator is where the pipeline behavior actually happens.

## Why this file matters

`Orchestrator` connects these components:

- `PIIDetector` detects PII spans in original text.
- `PIIRedactor` replaces detected spans with placeholders.
- `EmbedderBase` creates vector embeddings.
- `EmbeddingCache` avoids repeated embedding work.
- `VectorStoreBase` persists vectors and metadata.

The class is initialized with already-created dependencies:

```python
class Orchestrator:
    def __init__(
        self,
        config: LexiredactConfig,
        embedder: EmbedderBase,
        store: VectorStoreBase,
        cache: EmbeddingCache,
    ) -> None:
        self._config = config
        self._detector = PIIDetector(config.pii)
        self._redactor = PIIRedactor()
        self._embedder = embedder
        self._store = store
        self._cache = cache
```

This makes the orchestrator the coordination layer. It does not know the exact embedding backend or vector database implementation; it only relies on their interfaces.

## Main routing function

`process_batch()` chooses the pipeline mode from configuration:

```python
async def process_batch(self, chunks: list[Chunk]) -> list[ProcessingResult]:
    mode = self._config.pipeline_mode

    match mode:
        case "dual":
            results = await self._run_dual(chunks)
        case "preredacted":
            results = await self._run_preredacted(chunks)
        case "raw":
            results = await self._run_raw(chunks)
        case _:
            raise LexiredactConfigError(...)

    return results
```

This is the central switch for LexiRedact's behavior.

## Mode 1: `dual`

Dual mode is the default privacy-preserving performance path.

Flow:

1. Detect PII in the original text.
2. Run redaction and original-text embedding concurrently.
3. Store only sanitized text as vector metadata.

Important snippet:

```python
sanitized_texts, (vectors, cache_hits) = await asyncio.gather(
    redact_all(), embed_all()
)

self._store.upsert_batch(
    ids=[c.id for c in chunks],
    vectors=vectors,
    metadatas=[
        {"text": sanitized_texts[i], **chunks[i].metadata}
        for i in range(len(chunks))
    ],
)
```

The key idea is that embeddings are created from original text for retrieval quality, but the text stored in the vector DB is sanitized. Original text is not written to the store in this mode.

## Mode 2: `preredacted`

Preredacted mode is stricter and sequential.

Flow:

1. Detect PII.
2. Redact text.
3. Embed sanitized text.
4. Store sanitized text.

Snippet:

```python
sanitized_texts = [
    self._redactor.redact(chunk.text, entities)
    for chunk, entities in zip(chunks, entity_lists)
]

vectors = await loop.run_in_executor(
    None, lambda: self._embedder.embed_batch(sanitized_texts)
)
```

This mode never embeds the original text. That can reduce retrieval quality compared with dual mode, but it is the most conservative path because PII is removed before embedding.

## Mode 3: `raw`

Raw mode is mostly useful as an evaluation baseline.

Flow:

1. Skip PII detection.
2. Embed original text.
3. Store original text.

Snippet:

```python
self._store.upsert_batch(
    ids=[c.id for c in chunks],
    vectors=vectors,
    metadatas=[{"text": chunk.text, **chunk.metadata} for chunk in chunks],
)
```

This mode intentionally stores original text. It should not be used where privacy protection is required.

## Cache behavior

Dual and raw modes use the embedding cache because they embed original text:

```python
cached_vectors = [self._cache.get(t) for t in texts]
miss_indices = [i for i, v in enumerate(cached_vectors) if v is None]
```

If a vector is missing from cache, the embedder computes it and the cache stores it for future use.

Preredacted mode does not use the cache because sanitized text changes after redaction and may not match the original-text cache key.

## Error behavior

Storage errors are not swallowed:

```python
try:
    self._store.upsert_batch(...)
except LexiredactStorageError:
    embedding_stored = False
    raise
```

This is important because silent storage failure would mean vectors were not persisted even though the caller might think ingestion succeeded.

## Output

Every mode returns a list of `ProcessingResult` objects. These include:

- chunk id
- sanitized text
- detected entities
- whether embedding was stored
- latency
- cache hit status
- pipeline mode
- per-stage latency breakdown

Example result creation:

```python
ProcessingResult(
    chunk_id=chunks[i].id,
    sanitized_text=sanitized_texts[i],
    entities_detected=entity_lists[i],
    embedding_stored=embedding_stored,
    latency_ms=latency_ms,
    cache_hit=cache_hits[i],
    pipeline_mode="dual",
)
```

## Summary

`orchestrator.py` is the most important LexiRedact file because it defines the actual privacy and retrieval behavior. The rest of the package provides adapters, models, config, detection, redaction, embedding, caching, and storage, but this file decides how they are combined.
