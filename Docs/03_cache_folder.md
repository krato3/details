# Cache Folder

Folder: `lexiredact/cache/`

This folder contains optional embedding cache support.

## `redis_cache.py`

This file defines `EmbeddingCache`, a Redis-backed cache for embeddings.

Purpose:

- Avoid recomputing embeddings for the same text.
- Store vectors in Redis using a hash of the text.
- Fail open if Redis is unavailable.

Key creation:

```python
def _make_key(self, text: str) -> str:
    hash_fragment = hashlib.sha256(text.encode()).hexdigest()[:16]
    return f"{self._config.key_prefix}:emb:{hash_fragment}"
```

Read behavior:

```python
def get(self, text: str) -> list[float] | None:
    if not self._config.enabled:
        return None
    try:
        self._ensure_connected()
        raw = self._client.get(key)
        if raw is None:
            return None
        return json.loads(raw)
    except Exception:
        return None
```

Write behavior:

```python
def set(self, text: str, vector: list[float]) -> None:
    if not self._config.enabled:
        return
    self._client.setex(key, self._config.ttl_seconds, json.dumps(vector))
```

Importance:

- Improves performance for repeated chunks.
- Keeps cache failures from breaking ingestion.
- Is used by `dual` and `raw` modes in the orchestrator.

Important design decision:

Cache failures do not propagate. A Redis failure becomes a cache miss, so the model computes the embedding normally.

## `__init__.py`

This package marker makes `lexiredact.cache` importable.
