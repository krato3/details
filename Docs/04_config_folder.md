# Config Folder

Folder: `lexiredact/config/`

This folder defines and loads LexiRedact configuration.

## `schema.py`

This file contains Pydantic configuration models.

Main root model:

```python
class LexiredactConfig(BaseModel):
    pipeline_mode: Literal["raw", "preredacted", "dual"] = "dual"
    input_schema: InputSchemaConfig = InputSchemaConfig()
    pii: PIIConfig = PIIConfig()
    embedder: EmbedderConfig = EmbedderConfig()
    cache: CacheConfig = CacheConfig()
    store: StoreConfig = StoreConfig()
```

Importance:

- Defines all valid config fields.
- Rejects unknown YAML keys with `extra="forbid"`.
- Ensures pipeline mode is one of `raw`, `preredacted`, or `dual`.

Important config classes:

`InputSchemaConfig`:

```python
class InputSchemaConfig(BaseModel):
    text_field: str = "text"
    id_field: str = "id"
    metadata_fields: list[str] = []
```

This controls how raw user dictionaries are read by `ChunkAdapter`.

`PIIConfig`:

```python
class PIIConfig(BaseModel):
    entities: list[str] = ["PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER", ...]
    language: str = "en"
    nlp_engine: Literal["spacy", "transformers", "stanza"] = "spacy"
    score_threshold: float = 0.7
```

This controls Presidio detection.

`EmbedderConfig`:

```python
class EmbedderConfig(BaseModel):
    backend: Literal["sentence_transformers", "huggingface"] = "sentence_transformers"
    model_name: str = "intfloat/e5-small-v2"
    document_prefix: str = "passage: "
    query_prefix: str = "query: "
```

This controls embedding backend, model, prefixes, batch size, and dimension.

`StoreConfig`:

```python
class StoreConfig(BaseModel):
    provider: str = "chroma"
    collection_name: str = "lexiredact"
    persist_directory: str = "./chroma_db"
```

This controls vector database storage.

## `loader.py`

This file defines `load_config()`.

Purpose:

- Accept a config dictionary, string path, or `Path`.
- Read YAML files.
- Validate through `LexiredactConfig`.
- Convert Pydantic validation errors into `LexiredactConfigError`.

Important snippet:

```python
def load_config(source: dict[str, Any] | str | Path) -> LexiredactConfig:
    match source:
        case dict():
            raw = source
        case str() | Path():
            text = path.read_text(encoding="utf-8")
            loaded = yaml.safe_load(text)
            raw = loaded

    return LexiredactConfig(**raw)
```

Importance:

- It is the config entry point used by both Python users and the CLI.
- It prevents invalid runtime configuration from reaching the pipeline.
- It gives readable errors when YAML or validation fails.

## `__init__.py`

This package marker makes `lexiredact.config` importable.
