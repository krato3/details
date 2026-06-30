# Interview Q&A

| Field | Value |
| --- | --- |
| Purpose | Prepare a developer to discuss LexiRedact architecture in interviews or code reviews. |
| Audience | Engineers explaining or defending the design. |
| Related files | [04_Object_Oriented_Design.md], [10_VectorDB_Integration.md], [15_Architecture_Decisions.md] |

## Architecture

1. **What is LexiRedact's core architecture?**  
It is a facade plus orchestrator architecture. `LexiredactPipeline` wires components and exposes `ingest()` (see: `pipeline_api.py`, lines 32-90); `Orchestrator` owns runtime mode policy (see: `orchestrator.py`, lines 38-310).

2. **Why split facade and orchestrator?**  
The facade handles user-facing API and dependency construction, while the orchestrator handles per-request workflow. This keeps public API concerns separate from processing policy.

3. **What is the most important privacy boundary?**  
Metadata construction before storage. Dual and preredacted modes store `sanitized_texts[i]`, not original text (see: `orchestrator.py`, lines 144-146 and 210-212).

4. **What would change for raw documents?**  
A chunking layer would need to run before `ChunkAdapter` or replace its input expectations. Today each input dict is already one chunk (see: `chunk_adapter.py`, lines 30-66).

5. **Why is config declarative?**  
Pydantic models make defaults and validation explicit (see: `config/schema.py`, lines 14-112). It avoids scattering defaults across constructors.

6. **Where is the composition root?**  
`LexiredactPipeline.__init__()` creates or receives all major collaborators (see: `pipeline_api.py`, lines 48-57).

7. **Does the architecture support retrieval?**  
Only at primitive level. `query_embed()` and `store.query()` exist, but no public pipeline `retrieve()` exists.

8. **What is the biggest architectural gap?**  
No detector/cache/store factories beyond embedder. Store provider config exists but is not used to choose a store (see: `config/schema.py`, line 91; `pipeline_api.py`, lines 52-55).

## Design Patterns

9. **Where is the facade pattern used?**  
`LexiredactPipeline` is the facade (see: `pipeline_api.py`, lines 32-90).

10. **Where is the adapter pattern used?**  
`ChunkAdapter` adapts external dicts; `ChromaStore` adapts Chroma APIs to `VectorStoreBase`.

11. **Where is factory used?**  
`create_embedder()` selects concrete embedder by config (see: `registry.py`, lines 28-55).

12. **Where is strategy used?**  
Embedding backends are strategies through `EmbedderBase` (see: `embedder/base.py`, lines 33-95).

13. **Is there middleware?**  
No. The orchestrator has fixed flow; there is no chain or `next()`.

14. **Is there template method?**  
Not strictly. `process_batch()` dispatches to private mode methods but no subclass hooks exist.

15. **Why use abstract base classes?**  
They encode contracts for embedders and vector stores, keeping infrastructure replaceable.

16. **What pattern is used for Chroma?**  
Adapter plus dependency inversion: `ChromaStore` implements `VectorStoreBase` (see: `store/chroma.py`, line 22).

## OOP

17. **Where is encapsulation visible?**  
Underscore attributes like `_model`, `_client`, `_orchestrator` hide implementation state.

18. **Inheritance or composition?**  
Inheritance is limited to true contracts (`EmbedderBase`, `VectorStoreBase`). Runtime workflow uses composition.

19. **Why does orchestrator create detector/redactor internally?**  
It simplifies construction, but partially violates dependency inversion. It is a pragmatic gap (see: `orchestrator.py`, lines 56-57).

20. **Which classes are stable extension contracts?**  
`EmbedderBase` and `VectorStoreBase`.

21. **Why are private methods used in orchestrator?**  
Mode methods are internal implementation details; callers should use `process_batch()` only.

22. **Are dataclasses used well?**  
Yes for simple data carriers: `Chunk`, `DetectedEntity`, `ProcessingResult` (see: `models/*.py`).

## Python-Specific

23. **How is `abc.ABC` used?**  
`EmbedderBase` and `VectorStoreBase` inherit `ABC` and use `@abstractmethod`.

24. **How is pattern matching used?**  
`load_config()` uses `match source` (see: `config/loader.py`, lines 39-70); orchestrator matches `pipeline_mode` (see: `orchestrator.py`, lines 71-82).

25. **How are dataclasses used?**  
For result and chunk models (see: `models/result.py`, lines 13 and 41; `models/chunk.py`, line 16).

26. **Are protocols used?**  
No. ABCs are used instead.

27. **Are properties used?**  
No significant `@property` usage exists in core package.

28. **Are context managers used?**  
CLI uses `with open(...)` for input JSON (see: `cli.py`, lines 105-106). Core loader uses `Path.read_text()`.

## Async And Concurrency

29. **What is async?**  
Only orchestrator methods are async; public API is sync.

30. **Why use `run_in_executor()`?**  
Embedding and redaction libraries are synchronous, so executor avoids blocking the event loop (see: `orchestrator.py`, lines 102-121).

31. **What concurrency exists in dual mode?**  
Redaction and embedding overlap via `asyncio.gather()` (see: `orchestrator.py`, lines 130-135).

32. **Is shared state protected?**  
No explicit locks protect lazy `_model`, `_analyzer`, or `_client`.

33. **What async issue affects web apps?**  
`pipeline.ingest()` calls `asyncio.run()` and can fail inside an existing event loop.

## Performance And Caching

34. **Why cache embeddings?**  
To avoid recomputing vectors for identical text. Cache code is in `redis_cache.py`, lines 37-74.

35. **What is the cache key?**  
`{prefix}:emb:{sha256(text)[:16]}` (see: `redis_cache.py`, lines 76-79).

36. **What is the cache invalidation strategy?**  
TTL plus manual `key_prefix` changes. The key does not include model config.

37. **What is the main latency bottleneck?**  
PII detection, embedding, and storage. The orchestrator tracks stage latency (see: `orchestrator.py`, lines 157-175).

38. **What happens if cache fails?**  
Warning and miss/no-op; ingestion continues (see: `redis_cache.py`, lines 54-74).

## Vector DB And Embeddings

39. **What is a vector here?**  
A Python `list[float]` returned by `EmbedderBase` implementations (see: `embedder/base.py`, lines 41-45).

40. **How does Chroma similarity search work here?**  
Collection is created with cosine HNSW metadata and queried with one embedding (see: `store/chroma.py`, lines 36-39 and 88-96).

41. **Why Chroma?**  
It gives persistent local storage without a separate server.

42. **Why does dimension matter?**  
The store receives embedding dimension during construction (see: `pipeline_api.py`, lines 52-55). Mismatched vectors can break backend storage.

43. **How are query embeddings different?**  
They use `query_prefix`, important for e5 models (see: `embedder/base.py`, lines 64-72).

## PII Detection

44. **How is PII detected?**  
Presidio `AnalyzerEngine.analyze()` runs on each chunk text (see: `detector.py`, lines 108-113).

45. **What entity types are supported?**  
Whatever `PIIConfig.entities` lists; defaults are in `config/schema.py`, lines 25-33.

46. **What about false positives?**  
They are controlled partly by `score_threshold` filtering (see: `detector.py`, lines 119-123).

47. **What happens when detection fails?**  
The chunk gets empty entities and processing continues (see: `detector.py`, lines 114-116).

## Error Handling And Observability

48. **What errors are custom?**  
`LexiredactError` and config/input/storage/cache subclasses (see: `exceptions.py`, lines 11-57).

49. **Are storage errors swallowed?**  
No. Orchestrator re-raises `LexiredactStorageError` (see: `orchestrator.py`, lines 149-151).

50. **How do you trace logs?**  
Call `configure_logging("DEBUG")` (see: `app_logging.py`, lines 22-44).

51. **Is there retry logic?**  
No explicit retry/backoff exists.

## Scalability And Trade-Offs

52. **What happens at 10x load?**  
Model inference and Chroma writes dominate. Shared lazy state and default executor sizing should be revisited.

53. **What needs to change for production scale?**  
Async-safe public API, explicit worker pools, store factory, retry policies, and better cache key versioning.

54. **What changes for multi-tenancy?**  
Tenant-aware metadata, collection naming, cache prefixes, and access controls.

55. **What is the main accepted trade-off?**  
The code favors a small, understandable local pipeline over full production abstractions like middleware, streaming, and provider registries.

