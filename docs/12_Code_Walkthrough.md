# Code Walkthrough

| Field | Value |
| --- | --- |
| Purpose | Walk through concrete ingestion, retrieval primitives, and edge cases. |
| Audience | Developers who learn best by following one request line by line. |
| Related files | [02_Request_Flow.md], [07_Error_Handling.md], [13_FAQs.md] |

## Part A: Complete Ingestion Walkthrough

Example input:

```python
raw = [
    {
        "id": "case-001",
        "text": "John Smith emailed john@example.com. Call him at 212-555-0199.",
    }
]
```

State transitions:

```text
raw dict
  -> Chunk(id="case-001", text=original, metadata={})
  -> DetectedEntity spans:
       PERSON John Smith
       EMAIL_ADDRESS john@example.com
       PHONE_NUMBER 212-555-0199
  -> sanitized text:
       <PERSON> emailed <EMAIL_ADDRESS>. Call him at <PHONE_NUMBER>.
  -> embedding vector:
       [float, float, ...]
  -> stored metadata:
       {"text": sanitized text}
```

Call graph:

```text
LexiredactPipeline.ingest(raw)
  -> ChunkAdapter.adapt_batch(raw)
     -> ChunkAdapter.adapt(raw[0])
        -> Chunk.__post_init__()
  -> Orchestrator.process_batch([chunk])
     -> Orchestrator._run_dual([chunk])
        -> PIIDetector.detect_batch([chunk])
           -> PIIDetector._ensure_loaded()
           -> PIIDetector._detect_single(chunk)
        -> asyncio.gather(redact_all(), embed_all())
           -> PIIRedactor.redact(chunk.text, entities)
           -> EmbeddingCache.get(chunk.text)
           -> EmbedderBase.embed_batch([chunk.text])
           -> EmbeddingCache.set(chunk.text, vector)
        -> ChromaStore.upsert_batch(ids, vectors, metadatas)
        -> ProcessingResult(...)
```

Line-by-line execution:

1. `LexiredactPipeline.ingest()` receives a list of dicts and calls `adapt_batch()` (see: `lexiredact/pipeline_api.py`, lines 66-90). It logs skipped invalid chunks and returns early if none are valid.
2. `ChunkAdapter.adapt()` checks the configured ID field, converts the ID to `str`, checks the text field, collects configured metadata fields, and returns `Chunk` (see: `lexiredact/adapters/chunk_adapter.py`, lines 37-66).
3. `Chunk.__post_init__()` rejects blank text (see: `lexiredact/models/chunk.py`, lines 30-40). The original text is not mutated.
4. `asyncio.run()` enters `Orchestrator.process_batch()` (see: `pipeline_api.py`, line 90).
5. `process_batch()` dispatches by `pipeline_mode` (see: `lexiredact/pipeline/orchestrator.py`, lines 68-88). With the sample YAML, mode is `dual`.
6. `_run_dual()` first calls `PIIDetector.detect_batch()` (see: `orchestrator.py`, lines 96-99).
7. `PIIDetector._ensure_loaded()` imports Presidio, builds the NLP engine, and creates `AnalyzerEngine` (see: `pipeline/pii/detector.py`, lines 79-100).
8. `_detect_single()` calls `self._analyzer.analyze(text=chunk.text, entities=..., language=...)`, filters by score and entity type, and creates `DetectedEntity` objects with original offsets (see: `detector.py`, lines 108-133).
9. `_run_dual()` starts `redact_all()` and `embed_all()` concurrently with `asyncio.gather()` (see: `orchestrator.py`, lines 101-135).
10. `PIIRedactor.redact()` returns the original text unchanged if entities are empty; otherwise it builds Presidio recognizer results and replacement operators (see: `pipeline/pii/redactor.py`, lines 38-78).
11. `embed_all()` checks cache by original text and embeds only misses (see: `orchestrator.py`, lines 111-128).
12. `ChromaStore.upsert_batch()` writes vectors and sanitized metadata (see: `orchestrator.py`, lines 137-148; `pipeline/store/chroma.py`, lines 53-72).
13. `_run_dual()` returns one `ProcessingResult` (see: `orchestrator.py`, lines 162-178).

Expected logs when logging is configured:

```text
INFO lexiredact.pipeline_api: LexiredactPipeline ready...
INFO lexiredact.pipeline.pii.detector: PIIDetector initialized...
INFO lexiredact.pipeline.pii.detector: Loading Presidio AnalyzerEngine...
INFO lexiredact.pipeline.pii.detector: Presidio AnalyzerEngine ready.
INFO lexiredact.pipeline.orchestrator: Processed 1 chunks in ...ms (mode=dual)
```

Debugging tips: add logs after `entity_lists` in `orchestrator.py`, line 98, after `sanitized_texts` in line 132, and before `upsert_batch()` in line 141. Never log full original text in production if privacy matters.

## Part B: Complete Retrieval Walkthrough

There is no `pipeline.retrieve(query="...")`. The implemented primitive flow is:

```python
query_vector = embedder.query_embed(["phone number for John"])[0]
results = store.query(query_vector, top_k=5)
```

Call graph:

```text
caller
  -> SentenceTransformerEmbedder.query_embed([query])
     -> _ensure_loaded()
     -> _apply_prefix(texts, config.query_prefix)
     -> _encode(prefixed)
  -> ChromaStore.query(query_vector, top_k, filters)
     -> collection.query(...)
     -> flatten ids/metadatas/distances
```

`query_embed()` applies the query prefix for asymmetric models (see: `lexiredact/pipeline/embedder/sentence_transformers.py`, lines 78-96). `ChromaStore.query()` returns dictionaries with `id`, `metadata`, and `distance` (see: `lexiredact/pipeline/store/chroma.py`, lines 88-103).

Debugging tips: inspect vector length against `embedder.get_dimension()` before querying. If results are poor, verify document and query prefixes in config.

## Part C: Edge Cases

### No Detectable PII

```text
raw text: "The quarterly report was approved."
entities: []
redacted text: same as raw text
```

`PIIRedactor.redact()` short-circuits and returns the original text when `entities` is empty (see: `pipeline/pii/redactor.py`, lines 48-49). In dual/preredacted mode, that unchanged text is stored because there was nothing to redact.

### Cache Hit

```text
EmbeddingCache.get(text) -> vector
miss_indices = []
embedder.embed_batch() is not called
cache_hit=True
```

This branch is in `orchestrator.py`, lines 111-128. In raw mode the equivalent branch is lines 253-269.

### Invalid Chunk

```text
{"id": "x", "text": "   "}
  -> Chunk.__post_init__()
  -> LexiredactInputError
  -> adapt_batch() warning + failed item
  -> pipeline skips it
```

Source: `models/chunk.py`, lines 30-40; `chunk_adapter.py`, lines 80-93; `pipeline_api.py`, lines 79-86.

