# Pipeline Embedder Folder

Folder: `lexiredact/pipeline/embedder/`

This folder handles vector embedding backends.

## `base.py`

Defines the abstract interface all embedders must implement.

```python
class EmbedderBase(ABC):
    @abstractmethod
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        ...

    @abstractmethod
    def query_embed(self, texts: list[str]) -> list[list[float]]:
        ...

    @abstractmethod
    def get_dimension(self) -> int:
        ...
```

Importance:

- Lets the orchestrator work with any embedding implementation.
- Defines the contract for document embeddings and query embeddings.
- Keeps heavyweight ML imports out of the base interface.

## `registry.py`

Creates the correct embedder based on config.

```python
def create_embedder(config: EmbedderConfig) -> EmbedderBase:
    if config.backend == "sentence_transformers":
        return SentenceTransformerEmbedder(config)
    elif config.backend == "huggingface":
        return HuggingFaceEmbedder(config)
```

Importance:

- Centralizes backend selection.
- Uses lazy imports so unused ML libraries are not imported.
- Makes `LexiredactPipeline` simpler.

## `sentence_transformers.py`

Implements embedding with the `sentence-transformers` library.

Document embedding:

```python
def embed_batch(self, texts: list[str]) -> list[list[float]]:
    self._ensure_loaded()
    prefixed = self._apply_prefix(texts, self._config.document_prefix)
    return self._encode(prefixed)
```

Query embedding:

```python
def query_embed(self, texts: list[str]) -> list[list[float]]:
    self._ensure_loaded()
    prefixed = self._apply_prefix(texts, self._config.query_prefix)
    return self._encode(prefixed)
```

Importance:

- Default embedding backend.
- Supports E5-style prefixes through config.
- Lazy-loads the model only when needed.
- Converts model output into plain `list[list[float]]`.

## `huggingface.py`

Implements embedding using raw HuggingFace `AutoModel` and `AutoTokenizer`.

Important behavior:

```python
self._tokenizer = AutoTokenizer.from_pretrained(self._config.model_name)
self._model = AutoModel.from_pretrained(self._config.model_name)
self._dimension = self._model.config.hidden_size
```

Mean pooling:

```python
pooled = torch.sum(token_embeddings * input_mask_expanded, 1) / torch.clamp(
    input_mask_expanded.sum(1), min=1e-9
)
```

Importance:

- Supports encoder models that are not packaged as sentence-transformers.
- Gives direct control over tokenization and pooling.
- Uses the same `EmbedderBase` interface as the default backend.

## `default.py`

Backward compatibility shim.

```python
from lexiredact.pipeline.embedder.sentence_transformers import (
    SentenceTransformerEmbedder as DefaultEmbedder,
)
```

Importance:

- Keeps old imports working.
- Points old `DefaultEmbedder` usage to the current implementation.

## `__init__.py`

This package marker makes `lexiredact.pipeline.embedder` importable.
