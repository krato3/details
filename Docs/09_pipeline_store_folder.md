# Pipeline Store Folder

Folder: `lexiredact/pipeline/store/`

This folder handles vector database storage and retrieval.

## `base.py`

Defines the abstract vector store interface.

```python
class VectorStoreBase(ABC):
    @abstractmethod
    def upsert_batch(
        self,
        ids: list[str],
        vectors: list[list[float]],
        metadatas: list[dict[str, Any]],
    ) -> None:
        ...

    @abstractmethod
    def query(self, query_vector: list[float], top_k: int = 5, filters=None) -> list[dict[str, Any]]:
        ...

    @abstractmethod
    def count(self) -> int:
        ...
```

Importance:

- Lets LexiRedact support different vector stores.
- Defines the failure contract: storage failures must raise `LexiredactStorageError`.
- Allows the orchestrator to store vectors without knowing the database details.

## `chroma.py`

Implements `VectorStoreBase` using ChromaDB.

Initialization:

```python
self._client = chromadb.PersistentClient(path=config.persist_directory)
self._collection = self._client.get_or_create_collection(
    name=config.collection_name,
    metadata={"hnsw:space": "cosine"},
)
```

Upsert:

```python
self._collection.upsert(
    ids=ids,
    embeddings=vectors,
    metadatas=metadatas,
)
```

Query:

```python
raw = self._collection.query(
    query_embeddings=[query_vector],
    n_results=top_k,
    include=["metadatas", "distances"],
)
```

Importance:

- Provides the default persistent vector database.
- Uses local storage through `PersistentClient`.
- Stores metadata alongside embeddings.
- Wraps database failures in `LexiredactStorageError`.

Privacy note:

In `dual` and `preredacted` modes, the orchestrator passes sanitized text to `ChromaStore`. `ChromaStore` stores whatever metadata it receives, so privacy is enforced mainly by the orchestrator before storage.

## `__init__.py`

This package marker makes `lexiredact.pipeline.store` importable.
