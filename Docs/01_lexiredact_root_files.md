# LexiRedact Root Files

Folder: `lexiredact/`

This folder exposes the public package API and contains cross-cutting support files used by the whole project.

## `__init__.py`

This is the package export file. It lets users import the important classes and functions directly from `lexiredact`.

Important exports:

```python
from lexiredact.config.loader import load_config
from lexiredact.pipeline_api import LexiredactPipeline
from lexiredact.models.result import DetectedEntity, ProcessingResult

__version__ = "0.0.2"
```

Importance:

- Defines the public import surface.
- Hides internal folder structure from users.
- Exposes `LexiredactPipeline`, `load_config`, result models, exceptions, and logging helpers.

Example:

```python
from lexiredact import LexiredactPipeline, load_config

config = load_config("lexiredact_config.yaml")
pipeline = LexiredactPipeline(config)
```

## `pipeline_api.py`

This is the main public API for ingestion. Users normally interact with `LexiredactPipeline`, not the lower-level orchestrator.

Core construction:

```python
_embedder = embedder or create_embedder(config.embedder)
_store = store or ChromaStore(config.store, _embedder.get_dimension())
_cache = EmbeddingCache(config.cache)
self._orchestrator = Orchestrator(config, _embedder, _store, _cache)
```

What it contributes:

- Creates the adapter, embedder, vector store, cache, and orchestrator.
- Allows optional custom embedders and custom stores.
- Converts raw dictionaries into valid internal chunks before processing.

Main method:

```python
def ingest(self, raw_chunks: list[dict[str, Any]]) -> list[ProcessingResult]:
    chunks, failed = self._adapter.adapt_batch(raw_chunks)
    if not chunks:
        return []
    return asyncio.run(self._orchestrator.process_batch(chunks))
```

Importance:

- It is the user-facing entry point.
- It protects the orchestrator from malformed raw input.
- It keeps public usage simple while the internal pipeline stays modular.

## `cli.py`

This file provides the command-line interface using Click.

Commands include:

- `ingest`
- `inspect`
- `validate`
- `export`
- `stats`
- `info`

Ingest command example:

```python
@cli.command()
@click.option("--config", "config_path", required=True)
@click.option("--input", "input_path", required=True)
def ingest(...):
    config = load_config(config_path)
    pipeline = LexiredactPipeline(config)
    results = pipeline.ingest(raw_chunks)
```

Importance:

- Lets the project run without writing Python code.
- Validates config and input files.
- Provides operational commands to inspect and export stored data.

## `app_logging.py`

This file centralizes logging behavior.

Important snippet:

```python
logging.getLogger("lexiredact").addHandler(logging.NullHandler())

def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(f"lexiredact.{name}")
```

Importance:

- Prevents unwanted logging warnings when LexiRedact is used as a library.
- Gives every module a consistent `lexiredact.*` logger namespace.
- Provides `configure_logging()` for CLI and debugging use.

## `exceptions.py`

This file defines the custom exception hierarchy.

Important classes:

```python
class LexiredactError(Exception): ...
class LexiredactConfigError(LexiredactError): ...
class LexiredactInputError(LexiredactError): ...
class LexiredactStorageError(LexiredactError): ...
class LexiredactCacheError(LexiredactError): ...
```

Importance:

- Gives callers predictable error types.
- Adds optional structured context to errors.
- Separates config, input, storage, and cache failures.

Example:

```python
raise LexiredactConfigError(
    "Unknown pipeline mode",
    context={"pipeline_mode": mode},
)
```

## `README.md`

This is package-level documentation. It explains LexiRedact from the package perspective and is useful for understanding intended usage.

Importance:

- Helps users start with the package.
- Complements the root project `README.md`.
- Documents the package at the same level as the source code.
