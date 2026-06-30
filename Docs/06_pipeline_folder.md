# Pipeline Folder

Folder: `lexiredact/pipeline/`

This folder contains the processing engine. The most important file is `orchestrator.py`, which is documented separately in `00_pipeline_orchestrator.md`.

## `orchestrator.py`

Role:

- Chooses `dual`, `preredacted`, or `raw` mode.
- Runs PII detection, redaction, embedding, cache lookup, and vector storage.
- Returns `ProcessingResult` objects.

Core mode switch:

```python
match mode:
    case "dual":
        results = await self._run_dual(chunks)
    case "preredacted":
        results = await self._run_preredacted(chunks)
    case "raw":
        results = await self._run_raw(chunks)
```

Importance:

- This file defines LexiRedact's privacy behavior.
- It controls whether original or sanitized text is embedded.
- It ensures sanitized text is stored in privacy-preserving modes.

## `__init__.py`

This package marker makes `lexiredact.pipeline` importable.

## Subfolders

The pipeline folder is split into focused subfolders:

- `embedder/`: embedding backends and factory.
- `pii/`: detection, engine selection, and redaction.
- `store/`: vector store interfaces and Chroma implementation.

These subfolders keep the orchestrator small enough to coordinate behavior without owning every implementation detail.
