# Adapters Folder

Folder: `lexiredact/adapters/`

This folder converts user input into LexiRedact's internal data shape.

## `chunk_adapter.py`

This file defines `ChunkAdapter`.

Purpose:

- Read raw dictionaries from users.
- Validate required fields.
- Convert valid input into `Chunk` objects.
- Collect per-item errors during batch adaptation.

Core method:

```python
def adapt(self, raw: dict[str, Any]) -> Chunk:
    cfg = self._config

    if cfg.id_field not in raw:
        raise LexiredactInputError(...)

    if cfg.text_field not in raw:
        raise LexiredactInputError(...)

    metadata = {
        key: raw[key] for key in cfg.metadata_fields if key in raw
    }

    return Chunk(id=chunk_id, text=text, metadata=metadata)
```

Importance:

- This is the boundary between untrusted user input and trusted internal pipeline objects.
- It keeps field names configurable through `InputSchemaConfig`.
- It prevents the pipeline from dealing with missing ids, missing text, or empty text.

Batch behavior:

```python
def adapt_batch(self, raws: list[dict[str, Any]]) -> tuple[list[Chunk], list[dict[str, Any]]]:
    successful_chunks = []
    failed_items = []

    for index, raw in enumerate(raws):
        try:
            successful_chunks.append(self.adapt(raw))
        except LexiredactInputError as exc:
            failed_items.append({"index": index, "error": str(exc), "raw": raw})
```

This allows partial success. Bad chunks are skipped, while good chunks continue through the pipeline.

## `__init__.py`

This package marker makes `lexiredact.adapters` importable.

Importance:

- Marks the folder as a Python package.
- Supports imports from other parts of the project.
