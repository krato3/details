# Pipeline PII Folder

Folder: `lexiredact/pipeline/pii/`

This folder detects and redacts personally identifiable information.

## `detector.py`

Defines `PIIDetector`, a wrapper around Presidio AnalyzerEngine.

Batch detection:

```python
def detect_batch(self, chunks: list[Chunk]) -> list[list[DetectedEntity]]:
    self._ensure_loaded()
    results = []
    for chunk in sub_batch:
        results.append(self._detect_single(chunk))
    return results
```

Single chunk detection:

```python
raw_results = self._analyzer.analyze(
    text=chunk.text,
    entities=self._config.entities,
    language=self._config.language,
)
```

Importance:

- Detects PII without modifying text.
- Returns spans as `DetectedEntity` objects.
- Filters results by configured score threshold and entity list.
- Keeps detection separate from redaction, which enables dual-mode parallelism.

## `engine_factory.py`

Builds the Presidio NLP engine from `PIIConfig`.

Engine selection:

```python
if engine_name == "spacy":
    nlp_configuration = {
        "nlp_engine_name": "spacy",
        "models": [{"lang_code": language, "model_name": model_name}],
    }
elif engine_name == "transformers":
    nlp_configuration = {
        "nlp_engine_name": "transformers",
        "models": [{"lang_code": language, "model_name": model_name}],
    }
```

Importance:

- Keeps Presidio engine setup out of `PIIDetector`.
- Supports `spacy`, `transformers`, and `stanza`.
- Converts setup failures into `LexiredactConfigError`.

## `redactor.py`

Defines `PIIRedactor`, which replaces detected PII spans with placeholders.

Core redaction:

```python
operators = {
    e.entity_type: OperatorConfig("replace", {"new_value": f"<{e.entity_type}>"})
    for e in entities
}

result = self._anonymizer.anonymize(
    text=text,
    analyzer_results=recognizer_results,
    operators=operators,
)
```

Importance:

- Does not detect PII itself.
- Uses spans from `PIIDetector`.
- Replaces sensitive values with labels like `<PERSON>` and `<EMAIL_ADDRESS>`.
- Is stateless per call, so it can run concurrently in dual mode.

## `__init__.py`

This package marker makes `lexiredact.pipeline.pii` importable.
