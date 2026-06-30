# Extending LexiRedact

| Field | Value |
| --- | --- |
| Purpose | Provide practical recipes for adding new capabilities without breaking existing behavior. |
| Audience | Contributors implementing new components. |
| Related files | [03_Class_Diagram.md], [04_Object_Oriented_Design.md], [10_VectorDB_Integration.md] |

## Extension Map

```text
New input shape        -> adapters/chunk_adapter.py or new adapter
New PII detector       -> new interface needed, then orchestrator injection
New redaction strategy -> pipeline/pii/redactor.py or new redactor interface
New embedder           -> implement EmbedderBase + registry
New vector store       -> implement VectorStoreBase + inject or store factory
New cache backend      -> add cache interface first
New config key         -> config/schema.py + propagation
New CLI command        -> cli.py
```

## New Middleware

There is no middleware system today. To add one, first define a request context and callable contract.

Recommended design:

```text
Middleware
  -> receives context and next callable
  -> may inspect/modify context
  -> must return next(context) result or an intentional replacement
```

Files to read first: `pipeline_api.py` and `pipeline/orchestrator.py`. Best insertion point: around `Orchestrator.process_batch()` so lower-level stores and embedders remain unaware.

Common mistake: putting middleware imports in `pipeline/store` or `pipeline/embedder`. Infrastructure should not know application workflow policy.

## New PII Detector

Current code has `PIIDetector` as a concrete class, not `BaseDetector`. To add a true detector strategy:

1. Create `pipeline/pii/base.py` with a `detect_batch(chunks)` abstract method.
2. Make `PIIDetector` implement it.
3. Change `Orchestrator.__init__()` to accept optional detector injection instead of constructing `PIIDetector(config.pii)` directly (see current construction at `orchestrator.py`, line 56).
4. Add config/factory support if YAML selection is needed.

Minimal detector shape:

```python
class RegexDetector:
    def detect_batch(self, chunks):
        return [[DetectedEntity(text="...", entity_type="EMAIL_ADDRESS", start=0, end=3, score=1.0)]]
```

Test: pass chunks with known spans and assert returned offsets align with original text.

## New Anonymiser

Current redaction is hardcoded to `OperatorConfig("replace", {"new_value": f"<{e.entity_type}>"})` (see: `pipeline/pii/redactor.py`, lines 65-68).

To add strategies:

1. Read `PIIRedactor.redact()` (lines 38-78).
2. Add config such as `pii.redaction_strategy`.
3. Change operator construction based on strategy.
4. Keep the method pure: input text in, new sanitized string out.

Example deterministic hash strategy:

```python
new_value = f"<{e.entity_type}:{hashlib.sha256(e.text.encode()).hexdigest()[:8]}>"
```

Test: assert original PII substring is absent and replacement is stable.

## New Embedding Model

Read first: `pipeline/embedder/base.py`, `registry.py`, and the two concrete embedders.

Implement:

```python
from lexiredact.pipeline.embedder.base import EmbedderBase

class MyEmbedder(EmbedderBase):
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        return [...]

    def query_embed(self, texts: list[str]) -> list[list[float]]:
        return [...]

    def get_dimension(self) -> int:
        return 768
```

Register: add a `Literal` value to `EmbedderConfig.backend` (see: `config/schema.py`, line 63), then add a branch to `create_embedder()` (see: `registry.py`, lines 40-55).

Test: assert output length equals input length, values are floats, dimension matches `get_dimension()`, and query/document prefixes are handled correctly.

## New Vector DB Adapter

Read [10_VectorDB_Integration.md]. Implement `VectorStoreBase`, wrap backend failures in `LexiredactStorageError`, and either inject the store into `LexiredactPipeline(config, store=...)` or add a store factory.

Minimal test fake is shown in `examples/custom_vectorstore.py`, lines 14-43.

## New Cache Backend

Current code has concrete `EmbeddingCache` with no base interface. To add DynamoDB or filesystem cache cleanly:

1. Create `cache/base.py` with `get(text)` and `set(text, vector)`.
2. Make `EmbeddingCache` implement it.
3. Type orchestrator constructor against the base interface.
4. Add a cache factory in `pipeline_api.py`.

Behavior contract: cache failures should remain non-fatal unless a new config explicitly says otherwise.

## New Configuration Option

1. Add a field to the relevant config model in `config/schema.py`.
2. Decide default and whether unknown/invalid values should fail startup.
3. Propagate via constructor injection.
4. Update `lexiredact_config.yaml`.
5. Update CLI `info` if users should see it (see: `cli.py`, lines 451-477).

## New CLI Command

Read `lexiredact/cli.py`. Add a function decorated with `@cli.command()` like existing commands (see: `cli.py`, lines 63-84 and 168-175). Load config with `load_config(config_path)`, catch `LexiredactError`, and return nonzero through `sys.exit(1)` like existing commands (see: lines 161-163).

## Common Mistakes

| Mistake | Why it hurts |
| --- | --- |
| Calling package root from low-level modules | Increases circular import risk. |
| Adding Chroma-specific logic to orchestrator | Breaks vector store abstraction. |
| Storing original text in dual/preredacted metadata | Creates privacy regression. |
| Forgetting query prefixes | Reduces retrieval quality for e5-style models. |
| Reusing cache prefix after embedding config change | Can serve stale vectors. |
| Assuming public `retrieve()` exists | It does not; use store/query primitives or add facade method. |
| Calling `pipeline.ingest()` inside an event loop | It uses `asyncio.run()` and can fail. |
| Updating tests without fixing `lexiredact` imports | Current showcase tests are stale after rename. |

