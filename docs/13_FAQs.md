# FAQs

| Field | Value |
| --- | --- |
| Purpose | Answer common questions about reading, using, and extending LexiRedact. |
| Audience | New contributors and maintainers. |
| Related files | [02_Request_Flow.md], [07_Error_Handling.md], [12_Code_Walkthrough.md] |

```text
FAQ dependency path:
config -> pipeline_api -> adapter -> orchestrator -> detector/redactor/embedder/cache/store
```

1. **What happens if I modify `lexiredact/__init__.py`?**  
It changes the package-root public API. The file re-exports `load_config`, `LexiredactPipeline`, config, models, exceptions, and logging helpers (see: `lexiredact/__init__.py`, lines 10-41). Removing a name can break user imports even if internal code still works.

2. **Why is `EmbedderBase` abstract?**  
The orchestrator needs embedding behavior without caring about model provider. `EmbedderBase` defines `embed_batch`, `query_embed`, and `get_dimension` (see: `pipeline/embedder/base.py`, lines 33-95).

3. **Is there a `BaseDetector`?**  
No. `PIIDetector` is concrete and constructed directly by `Orchestrator` (see: `pipeline/orchestrator.py`, line 56). Adding pluggable detectors requires a new interface.

4. **Where is YAML loaded?**  
`load_config()` in `lexiredact/config/loader.py` reads the file and parses it with `yaml.safe_load()` (lines 43-55).

5. **What happens if the config path does not exist?**  
The loader catches `FileNotFoundError` and raises `LexiredactConfigError` with path context (see: `config/loader.py`, lines 46-52).

6. **How does startup initialize components?**  
`LexiredactPipeline.__init__()` creates adapter, embedder, Chroma store, cache, and orchestrator in that order (see: `pipeline_api.py`, lines 48-57).

7. **How are embeddings generated?**  
The selected embedder comes from `create_embedder()` based on `EmbedderConfig.backend` (see: `pipeline/embedder/registry.py`, lines 28-55). Built-in backends are sentence-transformers and HuggingFace.

8. **Where is the embedding model initialized?**  
Models are lazy. Sentence-transformers loads in `_ensure_loaded()` (see: `sentence_transformers.py`, lines 117-143). HuggingFace loads in `_ensure_loaded()` (see: `huggingface.py`, lines 109-140).

9. **How does cache lookup happen?**  
Dual mode calls `self._cache.get(t)` for each original text, embeds misses, then calls `self._cache.set()` (see: `orchestrator.py`, lines 111-128).

10. **What happens if Redis is unavailable at startup?**  
Nothing if cache is disabled. Even when enabled, Redis connects lazily on first get/set (see: `cache/redis_cache.py`, lines 32-35 and 81-98).

11. **What happens if Redis fails at request time?**  
`EmbeddingCache.get()` logs a warning and returns `None`; `set()` logs and no-ops (see: `redis_cache.py`, lines 54-74).

12. **How is the Chroma collection created?**  
`ChromaStore.__init__()` calls `chromadb.PersistentClient()` and `get_or_create_collection()` with cosine metadata (see: `pipeline/store/chroma.py`, lines 30-39).

13. **What happens on subsequent Chroma runs?**  
Because `get_or_create_collection()` is used with a persistent directory, the same collection is opened if it exists (see: `chroma.py`, lines 35-39).

14. **Why are adapters used instead of direct vector DB calls?**  
`VectorStoreBase` keeps Chroma details out of the orchestrator (see: `pipeline/store/base.py`, lines 17-61).

15. **Where is async used and why?**  
`Orchestrator` uses async methods and `asyncio.gather()` to overlap redaction and embedding in dual mode (see: `orchestrator.py`, lines 90-135).

16. **What breaks if middleware fails to call `next()`?**  
There is no middleware system, so this failure mode does not exist. A future middleware design would need to enforce return contracts.

17. **Where is configuration validated?**  
Pydantic validates in `LexiredactConfig(**raw)`, called by `load_config()` (see: `config/loader.py`, lines 72-78).

18. **How are documents chunked?**  
They are not. LexiRedact expects pre-chunked dicts. `ChunkAdapter` maps each dict to one `Chunk` (see: `adapters/chunk_adapter.py`, lines 30-66).

19. **How are vectors stored?**  
The orchestrator passes IDs, vectors, and metadata to `store.upsert_batch()` (see: `orchestrator.py`, lines 140-148).

20. **What metadata is stored?**  
At minimum `"text"` is stored. In dual/preredacted modes it is sanitized text plus selected chunk metadata (see: `orchestrator.py`, lines 144-146 and 210-212).

21. **How are document IDs generated?**  
They are not generated. The input `id_field` is converted to `str` (see: `chunk_adapter.py`, line 48).

22. **Is the original text mutated?**  
No. `Chunk.text` stores original text, and redaction returns a new string (see: `models/chunk.py`, lines 20-23; `redactor.py`, lines 38-78).

23. **Why use factories?**  
`create_embedder()` centralizes backend selection and lazy imports (see: `registry.py`, lines 7-19 and 28-55).

24. **What is the lifecycle of one request?**  
Adapt raw dicts, process batch by mode, detect/redact/embed/cache/store, return `ProcessingResult` (see: [02_Request_Flow.md]).

25. **How does dependency injection work?**  
The pipeline constructor accepts optional embedder and store, then passes dependencies into the orchestrator (see: `pipeline_api.py`, lines 42-57).

26. **What if a required config key is missing?**  
Most keys have defaults in `config/schema.py`, lines 14-112. Missing required explicit values are only enforced by validators, such as non-spacy `nlp_model` (lines 46-56).

27. **What if user constructor args are missing?**  
`LexiredactPipeline` requires `config`; Python raises `TypeError` if omitted (see signature in `pipeline_api.py`, lines 42-47).

28. **How are exceptions propagated?**  
Config/storage errors surface; input errors are collected; cache errors are swallowed with warnings (see: [07_Error_Handling.md]).

29. **Why is `adapters/` separate?**  
It isolates user input shape from internal `Chunk` shape (see: `adapters/chunk_adapter.py`, lines 1-6).

30. **Why is a method async if libraries are sync?**  
The orchestrator uses `run_in_executor()` so synchronous model/redaction work can be overlapped (see: `orchestrator.py`, lines 102-121).

31. **What class should I understand first?**  
Start with `LexiredactPipeline`, then `Orchestrator` (see: `pipeline_api.py`, lines 32-90; `orchestrator.py`, lines 38-310).

32. **What file should a contributor read first?**  
Read `pipeline_api.py` for the facade and `orchestrator.py` for runtime behavior.

33. **What happens if PII detection is removed?**  
Dual/preredacted modes would either fail because `entity_lists` is missing or store unredacted text if replaced with empty lists. The privacy guarantee depends on detection before redaction (see: `orchestrator.py`, lines 96-148).

34. **How can I trace a request through logs?**  
Call `configure_logging("DEBUG")`, then watch pipeline ready, detector load, cache warnings, and batch processed logs (see: `app_logging.py`, lines 22-44).

35. **Where should I add a new feature?**  
Use the folder that owns the concern: config in `config/`, embeddings in `pipeline/embedder/`, stores in `pipeline/store/`, PII in `pipeline/pii/`, CLI in `cli.py`.

36. **What coding conventions does the project follow?**  
Type hints, small interfaces, lazy heavy imports, Pydantic config, dataclasses for data, and custom exception wrapping.

37. **Can I call `pipeline.ingest()` from FastAPI async route?**  
Not directly without care. It calls `asyncio.run()` (see: `pipeline_api.py`, line 90), which fails inside a running event loop.

38. **Is there public retrieval?**  
No. Use `embedder.query_embed()` plus `store.query()` or add a facade method (see: `pipeline/store/chroma.py`, lines 74-108).

39. **Are tests current?**  
Showcase tests import `lexiredact`, not `lexiredact` (see: `tests/showcase/conftest.py`, lines 19-20), so they likely need rename cleanup.

40. **What is the most common extension pitfall?**  
Storing original text in privacy modes. Check metadata construction in `orchestrator.py`, lines 144-146 and 210-212.

