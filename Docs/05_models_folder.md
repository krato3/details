# Models Folder

Folder: `lexiredact/models/`

This folder contains the data objects that move through the pipeline and return to users.

## `chunk.py`

This file defines `Chunk`, the internal unit of processing.

```python
@dataclass
class Chunk:
    id: str
    text: str
    metadata: dict[str, Any] = field(default_factory=dict)
```

Validation:

```python
def __post_init__(self) -> None:
    if not self.text.strip():
        raise LexiredactInputError(...)
```

Importance:

- Represents one pre-chunked piece of text.
- Keeps original text immutable during pipeline processing.
- Carries metadata selected by `InputSchemaConfig`.
- Prevents empty text from entering PII detection or embedding.

Where it is used:

- Created by `ChunkAdapter`.
- Passed into `Orchestrator`.
- Read by PII detection, embedding, redaction, and storage logic.

## `result.py`

This file defines public result objects.

### `DetectedEntity`

Represents one PII span found in text.

```python
@dataclass
class DetectedEntity:
    text: str
    entity_type: str
    start: int
    end: int
    score: float
```

Importance:

- Stores the exact PII text and character offsets.
- Connects detection output to redaction input.
- Can be serialized with `to_dict()`.

### `ProcessingResult`

Represents the final outcome for one chunk.

```python
@dataclass
class ProcessingResult:
    chunk_id: str
    sanitized_text: str
    entities_detected: list[DetectedEntity]
    embedding_stored: bool
    latency_ms: float
    cache_hit: bool
    pipeline_mode: str
```

Importance:

- This is what `LexiredactPipeline.ingest()` returns.
- It reports privacy output, performance data, cache behavior, and storage status.
- It provides a stable public contract for CLI output and programmatic use.

Serialization:

```python
def to_dict(self) -> dict[str, Any]:
    return {
        "chunk_id": self.chunk_id,
        "sanitized_text": self.sanitized_text,
        "entities_detected": [e.to_dict() for e in self.entities_detected],
        "latency_ms": round(self.latency_ms, 2),
    }
```

## `__init__.py`

This package marker makes `lexiredact.models` importable.
