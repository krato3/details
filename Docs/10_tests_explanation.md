# Tests Folder Explanation

Folder: `tests/showcase/`

The current tests are a pytest-based showcase suite. They are not only checking correctness; they also generate a Markdown report named `showcase_report.md` after the run. The suite is organized around three themes: model configuration, input validation, and concurrent ingestion.

## Testing technique used

### 1. Pytest fixtures

`conftest.py` defines shared fixtures used by multiple tests:

```python
@pytest.fixture(scope="session")
def base_config() -> LexiredactConfig:
    return load_config(str(_DEFAULT_CONFIG_PATH))

@pytest.fixture(scope="session")
def sample_chunks() -> list[dict]:
    return [{"id": "showcase_001", "text": "..."}, ...]
```

Technique: fixture-based testing.

Why it matters:

- Avoids repeating setup code.
- Loads the same config for all tests.
- Provides reusable sample chunks for pipeline and concurrency tests.

## 2. Parametrized testing

`test_02_input_validation.py` uses `pytest.mark.parametrize` to test many input cases with one test function:

```python
@pytest.mark.parametrize(
    "case_name,raw,expect_ok",
    VALIDATION_CASES,
    ids=[c[0] for c in VALIDATION_CASES],
)
def test_single_record_validation(base_config, case_name, raw, expect_ok):
    ...
```

Technique: data-driven / parametrized testing.

Why it matters:

- Tests valid and invalid inputs consistently.
- Covers missing ids, missing text, empty text, whitespace-only text, long text, Unicode text, and numeric ids.
- Makes it easy to add more validation cases.

## 3. Unit-style validation tests

The input validation tests focus on `ChunkAdapter` directly:

```python
adapter = ChunkAdapter(base_config.input_schema)
chunks, failed = adapter.adapt_batch([raw])
```

Technique: focused unit-style testing.

Why it matters:

- These tests are fast.
- They do not require ML models, Presidio, Chroma, or Redis.
- They verify one boundary of the system: raw user dictionaries becoming valid internal `Chunk` objects.

These are the strongest tests in the folder because they are focused and deterministic.

## 4. Integration testing

`test_01_model_config.py` loads real components:

```python
embedder = create_embedder(base_config.embedder)
vectors = embedder.embed_batch([text])
```

and:

```python
detector = PIIDetector(base_config.pii)
entities = detector.detect_batch([sample])[0]
```

Technique: integration testing.

Why it matters:

- Verifies the configured embedding backend can load and produce vectors.
- Verifies Presidio can initialize and detect PII entities.
- Confirms config values actually work with real runtime dependencies.

Tradeoff:

- These tests are slower.
- They depend on installed ML libraries and local/downloadable models.
- They can fail because of environment setup, not only because of code bugs.

## 5. Threshold behavior testing

The detector score test checks that returned PII entities respect the configured threshold:

```python
assert all(e.score >= base_config.pii.score_threshold for e in entities)
```

Technique: configuration behavior testing.

Why it matters:

- Ensures low-confidence entities are filtered out.
- Confirms `PIIConfig.score_threshold` affects detection output.

## 6. Concurrency / load smoke testing

`test_03_load_concurrency.py` uses `ThreadPoolExecutor`:

```python
with ThreadPoolExecutor(max_workers=N_CONCURRENT) as pool:
    futures = [pool.submit(worker, i) for i in range(N_CONCURRENT)]
    results = [f.result() for f in as_completed(futures)]
```

Technique: concurrency smoke testing / small load testing.

Why it matters:

- Simulates multiple clients calling `pipeline.ingest()` at the same time.
- Checks that each request returns one result.
- Checks that chunk ids do not cross-contaminate between requests.
- Checks that embeddings are stored successfully under concurrent calls.

Limitation:

- It is not a full stress test.
- It does not prove complete thread safety.
- It does not assert a minimum overlap factor, even though it records one in the report.

## 7. Custom report generation

`conftest.py` implements pytest session hooks:

```python
def pytest_sessionstart(session):
    ...

def pytest_sessionfinish(session, exitstatus):
    runtime_s = time.perf_counter() - _SUITE_START
    _write_report(runtime_s)
```

Technique: test reporting / documentation-driven test output.

Why it matters:

- Produces `showcase_report.md` after tests finish.
- Records objectives, inputs, outputs, checks, metrics, and pass/fail status.
- Makes the test suite useful for demos and project explanation.

## Files in the tests folder

### `conftest.py`

Provides shared fixtures and report generation.

Main contribution:

- Loads config.
- Provides sample chunks.
- Defines `ReportRecord`.
- Writes `showcase_report.md` at the end of the test session.

### `test_01_model_config.py`

Tests model and PII configuration.

Main contribution:

- Checks embedder loading.
- Checks vector output shape and float values.
- Checks Presidio detector loading.
- Checks configured score threshold behavior.

### `test_02_input_validation.py`

Tests input adaptation and validation.

Main contribution:

- Checks valid records are accepted.
- Checks malformed records are rejected.
- Checks partial batch failure isolation.
- Checks custom input schema field names.
- Checks empty batch behavior.

### `test_03_load_concurrency.py`

Tests concurrent ingestion.

Main contribution:

- Runs multiple simultaneous ingest requests.
- Checks each result is isolated by id.
- Checks storage success under concurrent usage.

## Are these tests good?

The suite is useful, but it is a mixed showcase/integration suite rather than a clean fast unit test suite.

Strong parts:

- Input validation coverage is good and deterministic.
- Parametrization is used well.
- The tests check important user-facing behavior.
- The report generation is useful for explaining the project.

Weak parts:

- Model and PII tests rely on heavy external dependencies.
- Concurrency test is a smoke test, not a deep thread-safety proof.
- There are no mocked unit tests for the orchestrator modes.
- There are no direct tests proving that dual/preredacted modes never store original text.

Best classification:

- `test_02_input_validation.py`: good unit-style tests.
- `test_01_model_config.py`: useful integration tests.
- `test_03_load_concurrency.py`: useful concurrency smoke test.

Overall, the tests are good for a showcase and basic validation, but they should be expanded with faster mocked orchestrator tests for stronger CI confidence.