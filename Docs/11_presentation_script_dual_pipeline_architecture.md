# LexiRedact Presentation Script: Dual Pipeline, Architecture, and Core Package

Audience: project review / technical presentation  
Presentation date: July 1, 2026

## 1. Opening

Good morning/afternoon everyone.

Today I am going to explain the architecture of my project, LexiRedact, with special focus on the dual pipeline, because that is the core idea of the system.

LexiRedact is a privacy-preserving RAG ingestion middleware. In simple words, it sits between raw text data and a vector database. Its job is to process documents or text chunks, detect personally identifiable information, redact sensitive text before storage, generate embeddings, and store safe metadata in a vector database.

The main problem LexiRedact solves is this:

In a normal RAG pipeline, raw text is embedded and stored in a vector database. But if the text contains names, phone numbers, emails, locations, credit card numbers, or other sensitive information, that sensitive content may end up inside the vector database metadata. This is risky because vector databases are often queried, inspected, exported, or connected to downstream applications.

LexiRedact reduces that risk by separating two concerns:

- Retrieval quality
- Privacy-safe storage

The dual pipeline is designed around that separation.

## 2. High-Level Project Architecture

At a high level, the project has these main parts:

```text
LexiRedactCodebase/
|-- lexiredact/        Core Python package
|-- frontend/          Streamlit user interface
|-- backend/           Monitoring helpers for frontend
|-- eval/              Evaluation datasets, metrics, and reports
|-- tests/             Pytest showcase tests
|-- docs/              Project documentation
|-- Details/           Explanation files and presentation notes
|-- lexiredact_config.yaml
```

The most important folder is:

```text
lexiredact/
```

This is the core package. It contains the actual pipeline logic.

The frontend and backend folders support the dashboard experience, but the core privacy-preserving processing happens inside the `lexiredact` package.

## 3. What The Core Package Does

The core package performs the following steps:

1. Load and validate configuration.
2. Accept raw input chunks.
3. Convert raw dictionaries into internal `Chunk` objects.
4. Detect PII entities.
5. Redact sensitive spans.
6. Generate embeddings.
7. Optionally use Redis cache for repeated embeddings.
8. Store vectors and sanitized metadata in ChromaDB.
9. Return structured processing results.

The public user-facing flow looks like this:

```python
from lexiredact import LexiredactPipeline, load_config

config = load_config("lexiredact_config.yaml")
pipeline = LexiredactPipeline(config)
results = pipeline.ingest([
    {"id": "c1", "text": "John Smith emailed john@example.com"}
])
```

A user does not need to call the detector, redactor, embedder, or vector store manually. The package hides that complexity behind `LexiredactPipeline`.

## 4. Core Package Folder Structure

Inside `lexiredact/`, the important folders and files are:

```text
lexiredact/
|-- __init__.py
|-- pipeline_api.py
|-- cli.py
|-- app_logging.py
|-- exceptions.py
|-- adapters/
|-- cache/
|-- config/
|-- models/
|-- pipeline/
```

Each part has a clear responsibility.

### `pipeline_api.py`

This is the public API entry point. It defines `LexiredactPipeline`.

It creates the main components:

```python
_embedder = embedder or create_embedder(config.embedder)
_store = store or ChromaStore(config.store, _embedder.get_dimension())
_cache = EmbeddingCache(config.cache)
self._orchestrator = Orchestrator(config, _embedder, _store, _cache)
```

This means the public API is responsible for assembling the pipeline.

### `config/`

The config folder defines configuration models using Pydantic.

The root config is:

```python
class LexiredactConfig(BaseModel):
    pipeline_mode: Literal["raw", "preredacted", "dual"] = "dual"
    input_schema: InputSchemaConfig = InputSchemaConfig()
    pii: PIIConfig = PIIConfig()
    embedder: EmbedderConfig = EmbedderConfig()
    cache: CacheConfig = CacheConfig()
    store: StoreConfig = StoreConfig()
```

This config controls the complete behavior of the system.

### `adapters/`

The adapter converts raw user dictionaries into internal `Chunk` objects.

For example, raw input:

```python
{"id": "c1", "text": "John called 555-1234"}
```

becomes:

```python
Chunk(id="c1", text="John called 555-1234", metadata={})
```

This keeps the rest of the pipeline clean because it only deals with validated `Chunk` objects.

### `models/`

The models folder contains data objects:

- `Chunk`: internal input unit
- `DetectedEntity`: one detected PII span
- `ProcessingResult`: final output for each processed chunk

### `pipeline/`

This is the most important folder. It contains:

```text
pipeline/
|-- orchestrator.py
|-- embedder/
|-- pii/
|-- store/
```

The orchestrator coordinates the full pipeline.

## 5. Most Important File: `orchestrator.py`

The most important file in the project is:

```text
lexiredact/pipeline/orchestrator.py
```

This file decides how the pipeline runs.

The main class is:

```python
class Orchestrator:
    def __init__(self, config, embedder, store, cache):
        self._config = config
        self._detector = PIIDetector(config.pii)
        self._redactor = PIIRedactor()
        self._embedder = embedder
        self._store = store
        self._cache = cache
```

This class connects all major components:

- PII detector
- PII redactor
- Embedder
- Cache
- Vector store

The main method is:

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
```

This method routes the input batch to one of three pipeline modes.

## 6. Three Pipeline Modes

LexiRedact supports three modes:

```text
raw
preredacted
dual
```

Each mode exists for a different reason.

## 7. Raw Mode

Raw mode is the simplest mode.

Flow:

```text
Original text -> Embed original text -> Store original text
```

In raw mode:

- No PII detection happens.
- No redaction happens.
- Original text is embedded.
- Original text is stored.

This mode is useful as a baseline for evaluation, but it is not privacy preserving.

Code idea:

```python
texts = [chunk.text for chunk in chunks]
vectors = self._embedder.embed_batch(texts)

self._store.upsert_batch(
    ids=[c.id for c in chunks],
    vectors=vectors,
    metadatas=[{"text": chunk.text, **chunk.metadata} for chunk in chunks],
)
```

The important point is that raw mode stores original text. So for privacy-sensitive data, this mode should not be used.

## 8. Preredacted Mode

Preredacted mode is the most conservative privacy mode.

Flow:

```text
Original text -> Detect PII -> Redact text -> Embed sanitized text -> Store sanitized text
```

In this mode, the text is redacted before embedding.

That means the embedder never receives original PII-containing text.

Code idea:

```python
entity_lists = self._detector.detect_batch(chunks)

sanitized_texts = [
    self._redactor.redact(chunk.text, entities)
    for chunk, entities in zip(chunks, entity_lists)
]

vectors = self._embedder.embed_batch(sanitized_texts)
```

This gives strong privacy protection, but it may reduce retrieval quality.

Why?

Because replacing real names, emails, locations, and other details with placeholders can remove useful semantic information from the embedding.

For example:

```text
Original:  John Smith works in the cardiology department.
Redacted:  <PERSON> works in the cardiology department.
```

The redacted version is safer, but the embedding may lose some detail.

## 9. Dual Mode: The Main Innovation

Dual mode is the central architecture of LexiRedact.

Flow:

```text
                  +--> Redact original text --------+
                  |                                  |
Original text -> Detect PII                          +--> Store sanitized text
                  |                                  |
                  +--> Embed original text ----------+
```

In dual mode:

1. The system detects PII in the original text.
2. Then two operations run in parallel:
   - Redaction
   - Embedding
3. The embedding is created from original text.
4. The stored metadata is sanitized text.

So the vector is based on the original text, but the text stored in the vector database is redacted.

This is the key idea:

```text
Use original text for embedding quality.
Use sanitized text for storage privacy.
```

## 10. Why Dual Mode Is Useful

Dual mode tries to balance two competing goals.

Goal 1: Good retrieval quality

Embeddings work best when they receive complete original text. Original text has more semantic detail.

Goal 2: Privacy-safe storage

Vector DB metadata should not contain raw PII. If someone queries or exports the vector DB, they should see redacted text, not sensitive information.

Dual mode gives both:

- Original text goes to the embedder.
- Sanitized text goes to storage.
- Original text does not get stored as metadata.

## 11. Dual Mode Step-by-Step

Now I will explain the dual mode in detail.

### Step 1: PII Detection

The first step is PII detection:

```python
entity_lists = self._detector.detect_batch(chunks)
```

The detector uses Presidio AnalyzerEngine internally.

For each chunk, it returns a list of detected entities.

Example input:

```text
John Smith emailed john@example.com from Mumbai.
```

Detected entities might be:

```text
PERSON: John Smith
EMAIL_ADDRESS: john@example.com
LOCATION: Mumbai
```

These are represented as `DetectedEntity` objects:

```python
DetectedEntity(
    text="john@example.com",
    entity_type="EMAIL_ADDRESS",
    start=19,
    end=35,
    score=0.98,
)
```

### Step 2: Parallel Redaction and Embedding

After detection, dual mode starts two tasks at the same time.

The code uses `asyncio.gather()`:

```python
sanitized_texts, (vectors, cache_hits) = await asyncio.gather(
    redact_all(), embed_all()
)
```

This is important because redaction and embedding are independent after detection.

The redactor needs:

- original text
- detected entity spans

The embedder needs:

- original text

They do not depend on each other, so they can run concurrently.

### Step 3: Redaction Path

The redaction path creates sanitized text.

```python
self._redactor.redact(chunk.text, entities)
```

Example:

```text
Before: John Smith emailed john@example.com.
After:  <PERSON> emailed <EMAIL_ADDRESS>.
```

The redactor uses Presidio AnonymizerEngine and replaces each detected entity with a placeholder.

### Step 4: Embedding Path

The embedding path creates vectors from original text.

```python
texts = [chunk.text for chunk in chunks]
vectors = self._embedder.embed_batch(texts)
```

This preserves retrieval quality because the embedder sees the full original semantic content.

If cache is enabled, the pipeline first checks Redis:

```python
cached_vectors = [self._cache.get(t) for t in texts]
miss_indices = [i for i, v in enumerate(cached_vectors) if v is None]
```

If the vector is already cached, the model does not recompute it.

### Step 5: Storage

The final step stores vectors with sanitized metadata:

```python
self._store.upsert_batch(
    ids=[c.id for c in chunks],
    vectors=vectors,
    metadatas=[
        {"text": sanitized_texts[i], **chunks[i].metadata}
        for i in range(len(chunks))
    ],
)
```

This line is very important.

The vector comes from original text, but the metadata text is sanitized.

So ChromaDB receives:

```text
id: c1
vector: embedding(original text)
metadata.text: sanitized text
```

It does not receive:

```text
metadata.text: original text
```

That is the privacy protection in dual mode.

## 12. Dual Mode Architecture Diagram

You can explain the architecture like this:

```text
                    Raw Input Dicts
                          |
                          v
                   ChunkAdapter
                          |
                          v
                    Chunk Objects
                          |
                          v
                    Orchestrator
                          |
                          v
                   PII Detection
                          |
          +---------------+---------------+
          |                               |
          v                               v
      Redaction                     Embedding
          |                               |
          v                               v
   Sanitized Text                  Vector Embedding
          |                               |
          +---------------+---------------+
                          |
                          v
                    ChromaStore
                          |
                          v
        Vector + Sanitized Metadata Stored
```

The most important message in this diagram is:

```text
Original text is used for embedding.
Sanitized text is used for storage.
```

## 13. Component Responsibilities

### `LexiredactPipeline`

Located in:

```text
lexiredact/pipeline_api.py
```

Responsibility:

- Public entry point
- Builds dependencies
- Accepts raw chunks
- Calls the orchestrator

Speaker explanation:

`LexiredactPipeline` is the simple interface for users. It hides the internal pipeline complexity. Users call `ingest()`, and the pipeline handles adaptation, orchestration, embedding, redaction, and storage.

### `ChunkAdapter`

Located in:

```text
lexiredact/adapters/chunk_adapter.py
```

Responsibility:

- Validate raw input
- Convert raw dictionaries to `Chunk` objects
- Collect invalid records without crashing the whole batch

Speaker explanation:

This is the input safety layer. It ensures the pipeline receives clean and predictable input.

### `PIIDetector`

Located in:

```text
lexiredact/pipeline/pii/detector.py
```

Responsibility:

- Use Presidio to detect PII
- Return detected spans
- Does not modify text

Speaker explanation:

The detector only identifies sensitive spans. It does not redact anything. This separation is important because dual mode needs detection output for redaction while embedding can still happen from the original text.

### `PIIRedactor`

Located in:

```text
lexiredact/pipeline/pii/redactor.py
```

Responsibility:

- Replace detected PII spans with placeholders

Example:

```text
John Smith -> <PERSON>
john@example.com -> <EMAIL_ADDRESS>
```

Speaker explanation:

The redactor makes the text safe for storage.

### `EmbedderBase` and Embedders

Located in:

```text
lexiredact/pipeline/embedder/
```

Responsibility:

- Convert text into vector embeddings
- Support different embedding backends

The abstract interface is:

```python
class EmbedderBase:
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        ...

    def query_embed(self, texts: list[str]) -> list[list[float]]:
        ...

    def get_dimension(self) -> int:
        ...
```

Speaker explanation:

This abstraction means the orchestrator does not care whether embeddings come from sentence-transformers or HuggingFace. Any embedder can be used as long as it follows the interface.

### `EmbeddingCache`

Located in:

```text
lexiredact/cache/redis_cache.py
```

Responsibility:

- Cache embeddings in Redis
- Avoid repeated model computation
- Fail silently as cache miss if Redis is unavailable

Speaker explanation:

The cache improves performance but is not required for correctness. If Redis fails, the pipeline still runs by computing embeddings normally.

### `VectorStoreBase` and `ChromaStore`

Located in:

```text
lexiredact/pipeline/store/
```

Responsibility:

- Store vectors and metadata
- Query stored vectors
- Count stored documents

Default implementation:

```text
ChromaStore
```

Speaker explanation:

The store abstraction allows LexiRedact to use Chroma now and potentially support other vector databases later.

## 14. Configuration-Driven Design

LexiRedact is controlled by YAML config.

Example:

```yaml
pipeline_mode: dual

pii:
  entities:
    - PERSON
    - EMAIL_ADDRESS
    - PHONE_NUMBER
  language: en
  nlp_engine: spacy
  score_threshold: 0.7

embedder:
  backend: sentence_transformers
  model_name: intfloat/e5-small-v2
  document_prefix: "passage: "
  query_prefix: "query: "

store:
  provider: chroma
  collection_name: lexiredact
  persist_directory: ./chroma_db
```

This is useful because the user can change behavior without changing code.

For example:

- Change `pipeline_mode` from `dual` to `preredacted`.
- Change embedding backend.
- Change PII entities.
- Change Chroma collection name.
- Enable or disable cache.

## 15. Error Handling Design

LexiRedact uses custom exceptions:

```text
LexiredactError
|-- LexiredactConfigError
|-- LexiredactInputError
|-- LexiredactStorageError
|-- LexiredactCacheError
```

The important design decisions are:

- Config errors stop startup.
- Input errors are collected per chunk where possible.
- Storage errors are not swallowed.
- Cache errors do not stop the pipeline.

Why?

Because storage failure means data was not saved, so it must be visible to the caller. But cache failure only means performance is lower, so the pipeline can continue.

## 16. Testing Overview

The project uses pytest tests in:

```text
tests/showcase/
```

The tests cover:

- Model configuration
- PII detector loading
- Input validation
- Partial batch failure handling
- Custom schema fields
- Concurrent ingestion

The main testing techniques are:

- pytest fixtures
- parametrized tests
- integration tests
- concurrency smoke tests
- custom Markdown report generation

Current verification result:

```text
17 passed
```

This means the showcase tests are currently working.

## 17. How To Explain The Dual Pipeline In One Minute

If I need to explain the dual pipeline quickly, I would say:

The dual pipeline is the main privacy-preserving architecture of LexiRedact. First, the system detects PII in the original text. After detection, it runs two operations in parallel: redaction and embedding. The embedding is created from the original text so retrieval quality remains high. At the same time, the redactor creates a sanitized version of the text by replacing PII with placeholders. Finally, LexiRedact stores the vector together with only the sanitized text in ChromaDB. So the vector database gets useful embeddings but does not store raw sensitive text as metadata.

## 18. Strengths Of The Architecture

The main strengths are:

1. Privacy-aware storage

Sensitive text is redacted before storage in dual and preredacted modes.

2. Better retrieval quality

Dual mode embeds original text, so semantic quality is better than embedding heavily redacted text.

3. Modular design

Each component has one job:

- adapter validates input
- detector finds PII
- redactor sanitizes text
- embedder creates vectors
- store persists vectors
- orchestrator coordinates everything

4. Configurable behavior

The pipeline mode, PII engine, embedding backend, cache, and store are controlled through config.

5. Extensibility

Because embedders and stores use base interfaces, new backends can be added with less change to the core pipeline.

## 19. Limitations And Honest Discussion

It is also important to mention limitations.

1. Embedding original text may still encode sensitive information indirectly.

In dual mode, original text is not stored as metadata, but the embedding is still created from original text. Embeddings are not plain text, but privacy risk is not zero.

2. Detection quality depends on Presidio and the selected NLP model.

If the detector misses PII, that PII may remain in sanitized text.

3. Raw mode is not privacy preserving.

Raw mode exists mainly for evaluation comparison.

4. External dependencies matter.

The project depends on libraries like Presidio, sentence-transformers, ChromaDB, and optionally Redis.

This shows that the system is privacy-aware, but it still depends on correct configuration and good PII detection.

## 20. Closing

To conclude, LexiRedact is designed as a privacy-preserving ingestion layer for RAG systems.

The core package is organized around clean separation of responsibilities. The public API accepts raw chunks, the adapter validates them, the orchestrator controls the pipeline, the PII modules detect and redact sensitive data, the embedder creates vectors, and the store persists them.

The most important architectural feature is the dual pipeline:

```text
Original text for embedding quality.
Sanitized text for storage privacy.
```

That is the central idea of the project.

Thank you.

## 21. Possible Questions And Answers

### Q1. Why do we need a dual pipeline?

Because embedding sanitized text can reduce retrieval quality, while storing original text creates privacy risk. Dual mode balances both by embedding original text but storing sanitized text.

### Q2. Does dual mode store original text anywhere?

In dual mode, the vector store metadata receives sanitized text. The orchestrator passes sanitized text into `upsert_batch()`. Original text is used for embedding but is not stored as metadata.

### Q3. What is the difference between dual and preredacted mode?

In preredacted mode, text is redacted before embedding. In dual mode, original text is embedded while sanitized text is stored.

### Q4. Which file handles the full pipeline?

`lexiredact/pipeline/orchestrator.py` handles the core pipeline routing and execution.

### Q5. Which file should a user interact with?

Users should interact with `LexiredactPipeline` from `lexiredact/pipeline_api.py`.

### Q6. What vector database is used?

The default vector database is ChromaDB through `ChromaStore`.

### Q7. What detects PII?

PII detection is handled by Presidio through `PIIDetector`.

### Q8. Is the project configurable?

Yes. Pipeline mode, PII settings, embedding backend, cache, and vector store settings are controlled through `lexiredact_config.yaml`.

### Q9. What happens if Redis cache fails?

The cache failure is treated as a cache miss. The pipeline continues and computes embeddings normally.

### Q10. What happens if vector storage fails?

Storage errors are raised as `LexiredactStorageError`. They are not silently ignored because failed storage means ingestion did not complete correctly.