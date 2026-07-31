# Fix Plan — Issue #159

**Issue:** [structlog output is not captured by pytest caplog — log assertions fail suite-wide](https://github.com/ascherj/pathreview/issues/159)
**Tier:** 1 · **Branch:** `fix/159-structlog-caplog-capture`

## 1. Reproduction

```bash
docker compose up -d && make setup            # one-time
.venv/bin/pytest "tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty" -q --tb=short
```

Observed:

```
E   AssertionError: assert ('Empty chunks list' in '' or False)
E    +  where '' = <_pytest.logging.LogCaptureFixture object>.text
----------------------------- Captured stdout call -----------------------------
2026-07-21 23:09:54 [warning  ] Empty chunks list provided to BatchEmbeddingProcessor
```

The warning **is** emitted — it is printed to stdout — but `caplog.text` is empty. The log
never reaches pytest's capture handler.

Baseline for the whole suite on `main`: **53 failed / 375 passed**. Only 1 of those
failures belongs to this issue; the rest belong to other tracked issues (#158, #148,
#149, #150) and are out of scope here.

## 2. Root cause

`caplog` only sees records that pass through the standard library `logging` module.
The interesting part is *why* structlog isn't reaching it:

- `core/logging.py:11-52` defines `configure_logging()`, and it is already correct for
  this purpose — it sets `logger_factory=structlog.stdlib.LoggerFactory()`, which routes
  structlog through stdlib logging.
- **That function is never called during tests.** The only caller in the repo is
  `scripts/seed_db.py:20`. `api/main.py` does not call it, and no application module
  imports `core.logging`, so importing app code does not configure structlog as a side effect.
- With no configuration applied, structlog falls back to its **default** setup, which uses
  `PrintLoggerFactory` — writing directly to stdout, bypassing stdlib logging entirely.

So the defect is a **missing test-time configuration**, not a broken logging
implementation. That also explains the symptom precisely: the message is visible in
captured stdout while `caplog.text` stays empty.

Emitting code: `ingestion/embeddings/batch_processor.py:39-41`, using a module-level
`structlog.get_logger()` bound at line 7.

## 3. Scope and blast radius

- The only test file asserting on logs today is `tests/unit/test_batch_processor.py`
  (lines 36, 42, 43). One test fails because of this issue.
- `tests/conftest.py` is the only conftest; it has two fixtures and **no autouse fixtures**
  and no logging setup. `pyproject.toml:83-90` sets no logging-related pytest options.
- No test asserts on captured stdout/stderr, and no test uses `structlog.testing.capture_logs`,
  so **no existing test depends on the current print-to-stdout behavior.**
- Production behavior must not change: the fix belongs in test configuration only.

## 4. Options considered

| # | Approach | Trade-off |
|---|---|---|
| A | Autouse fixture in `tests/conftest.py` calling `core.logging.configure_logging()` | Tests exactly what ships, but the dev branch ends in `ConsoleRenderer`, so `record.message` becomes a whole pre-rendered (possibly ANSI-colored) line, making assertions fragile. Also triggers `logging.basicConfig()` as a side effect. |
| B | **Autouse fixture in `tests/conftest.py` that configures structlog for tests with `structlog.stdlib.LoggerFactory` and a `structlog.stdlib.render_to_log_kwargs` final processor** | Routes events into stdlib logging with the event as a clean `record.message`. Minimal, self-contained, production config untouched. **Recommended.** |
| C | Rewrite assertions to use `structlog.testing.capture_logs` | Fixes one test but not the general problem; the issue explicitly asks for caplog to work, and this changes test style repo-wide. |

**Chosen: B.** It resolves the issue as stated (caplog-based assertions work), touches one
test-support file, and leaves runtime logging untouched.

### 4.1 Final fixture design (implemented in `tests/conftest.py`)

```python
@pytest.fixture(autouse=True)
def configure_structlog_for_tests() -> Iterator[None]:
    structlog.configure(
        processors=[structlog.stdlib.render_to_log_kwargs],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=False,
    )
    yield
    structlog.reset_defaults()
```

Decisions and risks baked into that shape:

- **`cache_logger_on_first_use=False` — risk, decided.** `configure_logging()` uses `True` in
  production, and mirroring that here would introduce an order-dependency bug. Modules bind
  their logger at import time (`ingestion/embeddings/batch_processor.py:7`), and
  `structlog.get_logger()` returns a *lazy proxy* that resolves and then caches a concrete
  bound logger on first use. With caching on, the first test that touches a module-level
  logger would freeze whatever configuration was live at that moment, and the
  `reset_defaults()` teardown would leave that cached logger pointing at a stale
  (print-based) factory — so caplog capture would pass or fail depending on collection
  order. `False` re-resolves per call; the cost is irrelevant at test scale. Production
  config is untouched and keeps `True`.
- **`wrapper_class=structlog.stdlib.BoundLogger`** so `logger.warning(...)` dispatches to the
  matching stdlib level method; that is what makes `record.levelname` correct rather than the
  level living only inside the event dict.
- **`render_to_log_kwargs` as the sole processor** keeps `record.message` equal to the raw
  event string and moves bound key/values into `extra`, where stdlib turns them into record
  attributes. No timestamp/level/renderer processors are needed — stdlib supplies those, and
  omitting them is what avoids option A's pre-rendered, possibly ANSI-colored `record.message`.
- **`reset_defaults()` teardown** so the fixture cannot leak test configuration into anything
  that expects structlog defaults.
- **Constraint for future log assertions.** `pyproject.toml` sets no `log_level`/`log_cli`, so
  pytest leaves the root logger at `WARNING`. Assertions on `info`/`debug` events must wrap the
  call in `caplog.at_level(logging.INFO)`. The regression test exercises both paths.

## 5. Acceptance criteria

1. `tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty` passes.
2. Suite-wide, any test asserting via `caplog` captures structlog events; `caplog.records`
   carry the correct level and message.
3. No production module is modified — the diff is limited to test support
   (`tests/conftest.py` plus the new `tests/unit/test_logging_capture.py`).
4. No regression: total pass count increases by exactly the tests this fix targets, and the
   remaining known failures (#158, #148, #149, #150) are unchanged at 52.
5. Lint/format/type checks report **no new findings in the files this change touches**. A
   repo-wide green run is not an achievable bar here: on `main`, `ruff check .` already
   reports 182 findings (71 of them under `tests/`), `black --check .` wants to reformat 52
   files, and `mypy api/ core/ ingestion/ rag/ agent/ safety/ --ignore-missing-imports`
   aborts on a numpy stub because `python_version = "3.11"` is configured while the installed
   stubs use 3.12 `type` syntax (99 pre-existing errors surface when forced to 3.12). All
   three CI checks are therefore red before this change. Pre-commit is scoped to staged
   files, so what matters — and what holds — is that `tests/conftest.py` and
   `tests/unit/test_logging_capture.py` are clean under ruff and black.

## 6. Verification

Run on this branch (`.venv`: Python 3.12, structlog 26.1.0, pytest 9.1.1):

| Check | Result |
|---|---|
| `pytest "…::test_empty_chunks_list_returns_empty" -q` | `1 passed` (previously failing) |
| `pytest tests/unit -q` | `52 failed, 378 passed` vs baseline `53 failed, 375 passed` |
| sorted `FAILED` list vs baseline | only `test_empty_chunks_list_returns_empty` removed; nothing new |
| `pytest tests/unit -q -m unit` | same `52 failed, 378 passed`; new file collected under both invocations |
| `ruff check tests/conftest.py tests/unit/test_logging_capture.py` | `All checks passed!` |
| `black --check` on the same two files | `2 files would be left unchanged` |

`tests/unit/test_logging_capture.py` is the regression guard. It binds a module-level
`structlog.get_logger()` — the same import-time pattern the bug affected — and asserts the
level, the message and a structured attribute on `caplog.records`, plus `caplog.text`. Run
from outside the `tests/` tree, i.e. without the conftest fixture, both of its cases fail with
the original symptom (event visible in captured stdout, `caplog` empty), so they cannot pass
vacuously.

## 7. Open questions for maintainers

- Preference between reusing `configure_logging()` (option A) and a test-specific structlog
  configuration (option B)? I chose B for assertion stability but will follow the project's
  preference.
- Should `configure_logging()` also be wired into `api/main.py` startup? It appears to be
  called nowhere but the seed script, which looks like a separate gap — happy to file a
  follow-up issue rather than widen this PR.
- The `lint`, `format` and `typecheck` CI jobs are already failing on `main` (see §5). That
  predates this branch and is untouched here — worth its own issue rather than folding a
  repo-wide reformat into this fix?

## 8. Out of scope

The other 52 baseline failures (`test_review_service.py` async mocks — #158; skill
extractor — #148; structural chunker — #149; tech detector — #150). This PR stays limited
to log capture in tests.
