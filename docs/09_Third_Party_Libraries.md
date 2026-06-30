# Third-Party Libraries

| Field | Value |
| --- | --- |
| Purpose | Explain every declared external dependency and replacement considerations. |
| Audience | Developers changing dependencies or deploying the package. |
| Related files | [06_YAML_System.md], [08_Optimization.md], [10_VectorDB_Integration.md] |

## Dependency Overview

```text
Core:        pydantic, pyyaml
PII extra:   presidio-analyzer, presidio-anonymizer, spacy
Embed extra: sentence-transformers, transformers/torch implied by huggingface backend
Store extra: chromadb
Cache extra: redis
CLI extra:   click
Frontend:    streamlit, pandas, plotly, psutil, streamlit-elements, streamlit-aggrid, streamlit-shadcn-ui
Dev:         pytest, pytest-asyncio, ruff, mypy
```

Dependencies are declared in `pyproject.toml`, lines 28-93.

## Configuration And Serialization

### `pydantic`

What it does: validates and structures config models.

Why selected: Pydantic v2 gives typed nested models, defaults, literals, and clear validation errors with little custom code. It is used in `lexiredact/config/schema.py`, lines 11-112.

Advantages: strong validation, default handling, `model_copy()` used by CLI overrides (see: `lexiredact/cli.py`, lines 98-103). Risks: version-specific v2 APIs; replacing it would require rewriting schema validation and error formatting.

Replacement difficulty: medium. `dataclasses` plus manual validation could replace it but would be more verbose.

### `pyyaml`

What it does: parses YAML config files.

Why selected: simple safe YAML parsing through `yaml.safe_load()` (see: `lexiredact/config/loader.py`, lines 53-55). It is lighter than `ruamel.yaml` because the code does not preserve comments or formatting.

Risks: YAML type coercion surprises. Replacement difficulty: low if switching to TOML/JSON, medium if preserving YAML compatibility.

## NLP/PII Detection

### `presidio-analyzer`

What it does: detects PII entities.

Where imported: lazy in `PIIDetector._ensure_loaded()` and `build_nlp_engine()` (see: `pipeline/pii/detector.py`, lines 88-99; `pipeline/pii/engine_factory.py`, lines 53-60).

Why selected: it provides ready-made recognizers and NLP engine integration. Replacement difficulty: high because entity result shape and confidence filtering would need a new adapter.

### `presidio-anonymizer`

What it does: anonymizes text using detected spans.

Where imported: `PIIRedactor.__init__()` and `redact()` (see: `pipeline/pii/redactor.py`, lines 32-35 and 51-73).

Why selected: pairs naturally with Presidio analyzer result concepts. Replacement difficulty: medium; span replacement could be hand-written but overlap rules are subtle.

### `spacy`

What it does: NLP model backend for Presidio.

Where used: config default `PIIConfig.nlp_engine="spacy"` and `spacy_model` alias (see: `config/schema.py`, lines 36-43). Presidio loads it through `NlpEngineProvider`.

Risks: model downloads and version compatibility. Replacement: transformers or stanza engines are supported by config factory (see: `engine_factory.py`, lines 62-82).

## Embedding

### `sentence-transformers`

What it does: encodes text into embeddings.

Where imported: lazy inside `SentenceTransformerEmbedder._ensure_loaded()` (see: `pipeline/embedder/sentence_transformers.py`, line 127).

Why selected: broad model support and simple `.encode()` API. Advantages: local/offline embeddings, no per-call API cost. Risks: large model downloads, torch stack complexity. Replacement difficulty: medium because `EmbedderBase` limits changes to a new implementation.

### `transformers` and `torch`

What they do: raw HuggingFace model loading and tensor inference.

Where imported: lazy in `HuggingFaceEmbedder._ensure_loaded()` and `_encode()` (see: `huggingface.py`, lines 114-122 and 162-190).

Why selected: fallback for models not available as sentence-transformers. Risks: bigger dependency surface and manual pooling choices. Replacement difficulty: medium.

## Vector DB

### `chromadb`

What it does: local persistent vector storage and similarity search.

Where imported: `ChromaStore.__init__()` (see: `pipeline/store/chroma.py`, lines 33-39).

Why selected: local persistent storage with no separate server. Advantages: easy quickstart and demos. Risks: local disk persistence may not fit multi-tenant production workloads. Replacement difficulty: medium because `VectorStoreBase` isolates most changes.

## Caching

### `redis`

What it does: stores embeddings by key with TTL.

Where imported: lazy in `EmbeddingCache._ensure_connected()` (see: `cache/redis_cache.py`, lines 85-92).

Why selected: common low-latency cache with TTL. Risks: cache key invalidation is manual via prefix. Replacement difficulty: low to medium if a new cache class preserves `get(text)` and `set(text, vector)`.

## CLI

### `click`

What it does: command-line parsing.

Where imported: `lexiredact/cli.py`, line 23.

Why selected: simple decorators and options. Replacement difficulty: medium because every CLI command is decorator-based.

## Frontend And Evaluation Extras

`streamlit`, `pandas`, `plotly`, `psutil`, `streamlit-elements`, `streamlit-aggrid`, and `streamlit-shadcn-ui` support dashboard/frontend code, not the core ingestion path. `pytest`, `pytest-asyncio`, `ruff`, and `mypy` are dev dependencies for tests, linting, and typing.

## Replacement Matrix

| Category | Current | Possible replacement | Difficulty |
| --- | --- | --- | --- |
| Config validation | Pydantic | dataclasses + manual validation | Medium |
| YAML | PyYAML | TOML, JSON, ruamel.yaml | Low/Medium |
| PII | Presidio | spaCy custom NER, regex engine | High |
| Embedding | sentence-transformers | OpenAI, Cohere, Instructor, custom HF | Medium |
| Vector DB | Chroma | Qdrant, Pinecone, Weaviate, FAISS | Medium |
| Cache | Redis | in-memory, DynamoDB, filesystem | Low/Medium |

