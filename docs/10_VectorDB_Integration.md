# Vector DB Integration

| Field | Value |
| --- | --- |
| Purpose | Explain the vector store abstraction, Chroma implementation, and how to add another backend. |
| Audience | Developers adding or debugging vector database integrations. |
| Related files | [03_Class_Diagram.md], [05_Main_Files.md], [11_Extending_LexiRedact.md] |

## Why An Abstraction Exists

Vector databases differ in client setup, collection creation, upsert APIs, metadata filtering, and query response shapes. LexiRedact isolates that behind `VectorStoreBase` so the orchestrator only needs three operations: upsert, query, and count (see: `lexiredact/pipeline/store/base.py`, lines 17-61).

Without this layer, `Orchestrator` would call Chroma directly. Then replacing Chroma with Qdrant would require changing workflow code and risk privacy logic.

## Adapter Interface

```text
VectorStoreBase
  + upsert_batch(ids, vectors, metadatas) -> None
  + query(query_vector, top_k=5, filters=None) -> list[dict]
  + count() -> int
```

Contract details:

| Method | Must do | Invariants |
| --- | --- | --- |
| `upsert_batch` | Persist IDs, vectors, and metadata. | Same ID overwrites previous entry; failures raise `LexiredactStorageError`; metadata should include sanitized `"text"` in privacy modes. |
| `query` | Return nearest neighbors. | Shape must be `[{"id": ..., "metadata": ..., "distance": ...}]`. |
| `count` | Return stored record count. | Failures should raise `LexiredactStorageError`. |

Source: `lexiredact/pipeline/store/base.py`, lines 24-61.

## Chroma Adapter Walkthrough

Constructor:

```text
ChromaStore.__init__(config, embedding_dimension)
  -> chromadb.PersistentClient(path=config.persist_directory)
  -> client.get_or_create_collection(name=config.collection_name,
                                     metadata={"hnsw:space": "cosine"})
```

Source: `lexiredact/pipeline/store/chroma.py`, lines 30-43. Chroma is configured for persistent local storage and cosine similarity.

Upsert:

```text
upsert_batch(ids, vectors, metadatas)
  -> self._collection.upsert(ids=ids, embeddings=vectors, metadatas=metadatas)
```

Source: `chroma.py`, lines 53-72. IDs come from `Chunk.id`; metadata is constructed by the orchestrator. In dual/preredacted mode, `"text"` is sanitized (see: `orchestrator.py`, lines 140-148 and 207-214). In raw mode, original text is intentionally stored (see: `orchestrator.py`, lines 271-279).

Query:

```text
query(query_vector, top_k, filters)
  -> collection.query(query_embeddings=[query_vector],
                      n_results=top_k,
                      include=["metadatas", "distances"],
                      where=filters?)
  -> flatten first query result
```

Source: `chroma.py`, lines 74-108.

Count:

```text
count()
  -> self._collection.count()
```

Source: `chroma.py`, lines 110-122.

Errors: constructor, upsert, query, and count wrap exceptions in `LexiredactStorageError` (see: `chroma.py`, lines 44-51, 68-72, 104-108, 118-122).

## Adding A Qdrant Adapter

Step-by-step:

1. Read `VectorStoreBase` first.
2. Create `lexiredact/pipeline/store/qdrant.py`.
3. Implement all abstract methods.
4. Wrap client errors in `LexiredactStorageError`.
5. Export it from `pipeline/store/__init__.py`.
6. Add config fields or a store factory. Today `LexiredactPipeline` always defaults to `ChromaStore` unless a store object is injected (see: `pipeline_api.py`, lines 52-55).

Skeleton:

```python
from typing import Any

from lexiredact.exceptions import LexiredactStorageError
from lexiredact.pipeline.store.base import VectorStoreBase


class QdrantStore(VectorStoreBase):
    def __init__(self, client, collection_name: str) -> None:
        self._client = client
        self._collection_name = collection_name

    def upsert_batch(
        self,
        ids: list[str],
        vectors: list[list[float]],
        metadatas: list[dict[str, Any]],
    ) -> None:
        try:
            # Convert ids/vectors/metadatas into Qdrant points here.
            ...
        except Exception as exc:
            raise LexiredactStorageError("Qdrant upsert failed", {"error": str(exc)}) from exc

    def query(
        self,
        query_vector: list[float],
        top_k: int = 5,
        filters: dict[str, Any] | None = None,
    ) -> list[dict[str, Any]]:
        try:
            # Return [{"id": id, "metadata": payload, "distance": score_or_distance}]
            ...
        except Exception as exc:
            raise LexiredactStorageError("Qdrant query failed", {"error": str(exc)}) from exc

    def count(self) -> int:
        try:
            ...
        except Exception as exc:
            raise LexiredactStorageError("Qdrant count failed", {"error": str(exc)}) from exc
```

## Configuration Of Adapters

Current `StoreConfig` has `provider`, `collection_name`, and `persist_directory` (see: `lexiredact/config/schema.py`, lines 88-93). However, `provider` is not used by a store factory. `LexiredactPipeline` defaults directly to `ChromaStore` (see: `pipeline_api.py`, lines 52-55).

To make YAML select Qdrant, add a store factory:

```text
create_store(config.store, dimension)
  -> provider == "chroma": ChromaStore(...)
  -> provider == "qdrant": QdrantStore(...)
```

Then replace direct `ChromaStore(...)` construction in `pipeline_api.py`.

## Testing Adapters

There is no abstract adapter test suite in the repository. Existing showcase tests appear stale because they import `lexiredact`, not `lexiredact` (see: `tests/showcase/conftest.py`, lines 19-20). A proper adapter contract test should assert:

| Test | Expected |
| --- | --- |
| Upsert one record | `count()` increases and query can return metadata. |
| Upsert same ID | record is overwritten, not duplicated. |
| Query shape | list of dicts with `id`, `metadata`, `distance`. |
| Failure wrapping | backend exceptions become `LexiredactStorageError`. |
| Privacy metadata | orchestrator passes sanitized `"text"` in privacy modes. |

