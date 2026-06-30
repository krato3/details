# YAML Configuration System

| Field | Value |
| --- | --- |
| Purpose | Explain how declarative YAML controls runtime behavior. |
| Audience | Developers changing configuration or adding config keys. |
| Related files | [02_Request_Flow.md], [07_Error_Handling.md], [11_Extending_LexiRedact.md] |

## Annotated Schema

```yaml
pipeline_mode: dual        # "raw", "preredacted", or "dual"; default "dual"

input_schema:
  text_field: text         # str, default "text"
  id_field: id             # str, default "id"
  metadata_fields: []      # list[str], default []

pii:
  entities:                # list[str], default common Presidio entities
    - PERSON
    - EMAIL_ADDRESS
  language: en             # str, default "en"
  nlp_engine: spacy        # "spacy", "transformers", "stanza"; default "spacy"
  nlp_model: en_core_web_lg # str; if empty and spacy, falls back to spacy_model
  spacy_model: en_core_web_lg # deprecated alias
  score_threshold: 0.7     # float, default 0.7
  batch_size: 16           # int, default 16

embedder:
  backend: sentence_transformers # "sentence_transformers" or "huggingface"
  model_name: intfloat/e5-small-v2
  batch_size: 32
  device: cpu
  normalize_embeddings: true
  document_prefix: "passage: "
  query_prefix: "query: "
  dimension: null          # int or null; null may trigger model load for dimension

cache:
  enabled: false
  redis_url: redis://localhost:6379
  ttl_seconds: 86400
  key_prefix: vs

store:
  provider: chroma
  collection_name: lexiredact
  persist_directory: ./chroma_db
```

The schema is implemented in Pydantic models (see: `lexiredact/config/schema.py`, lines 14-112). Every model sets `extra="forbid"`, so unknown YAML keys fail validation (lines 14-15, 22-23, 59-60, 79-80, 88-89, 105).

## Loading Sequence

```text
load_config(source)
  -> dict: use source
  -> str|Path: read file text
     -> yaml.safe_load(text)
     -> require top-level dict
  -> LexiredactConfig(**raw)
  -> return typed config
```

The loader accepts a dict, string path, or `Path` (see: `lexiredact/config/loader.py`, lines 31-43). YAML files are read with `Path.read_text(encoding="utf-8")` (lines 44-48) and parsed with `yaml.safe_load()` (lines 53-55).

## Parsing And Validation

Validation is Pydantic v2. `LexiredactConfig(**raw)` constructs nested models and raises `ValidationError` on invalid types, unknown keys, or invalid literals. The loader catches that and raises `LexiredactConfigError` with a formatted message (see: `config/loader.py`, lines 72-78).

Special validation exists for `PIIConfig.nlp_model`: if it is empty and engine is `spacy`, it is filled from `spacy_model`; if engine is not `spacy`, the user must set it explicitly (see: `config/schema.py`, lines 46-56).

## Propagation

```text
LexiredactConfig
  -> LexiredactPipeline
     -> ChunkAdapter(config.input_schema)
     -> create_embedder(config.embedder)
     -> ChromaStore(config.store, dimension)
     -> EmbeddingCache(config.cache)
     -> Orchestrator(config, embedder, store, cache)
        -> PIIDetector(config.pii)
```

Propagation is constructor injection, not a global singleton (see: `lexiredact/pipeline_api.py`, lines 48-57; `lexiredact/pipeline/orchestrator.py`, lines 48-61).

## Configuration Precedence

| Rank | Source | Where |
| --- | --- | --- |
| 1 | Pydantic defaults | `config/schema.py`, lines 14-112 |
| 2 | Values passed to `load_config()` as dict/YAML | `config/loader.py`, lines 31-81 |
| 3 | CLI overrides | `cli.py`, lines 94-103 |

There is no environment-variable merge logic in `load_config()`. Tests reference `lexiredact_CONFIG_PATH`, but those tests appear stale and outside the core package.

## Runtime Changes

If a user modifies YAML and creates a new pipeline, the new config applies. If they edit the YAML file after `LexiredactPipeline(config)` already exists, nothing changes because components keep the config object and derived collaborators in memory (see: `pipeline_api.py`, lines 48-57).

## Adding A New Key

1. Add the field and default to the correct Pydantic model in `config/schema.py`.
2. Add validation with a model validator if the value interacts with other keys.
3. Pass the value through the relevant constructor from `pipeline_api.py` or `orchestrator.py`.
4. Update `lexiredact_config.yaml`.
5. Add tests or example coverage.
6. Update CLI display/overrides if users need command-line visibility.

Call graph:

```text
YAML key
  -> LexiredactConfig nested model
  -> pipeline constructor
  -> component config attribute
  -> runtime method
```

