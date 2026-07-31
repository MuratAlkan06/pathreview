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

## 5. Acceptance criteria

1. `tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty` passes.
2. Suite-wide, any test asserting via `caplog` captures structlog events; `caplog.records`
   carry the correct level and message.
3. No production module is modified — the diff is limited to test support (`tests/conftest.py`).
4. No regression: total pass count increases by exactly the tests this fix targets, and the
   remaining known failures (#158, #148, #149, #150) are unchanged at 52.
5. `make lint` / `make typecheck` pass; pre-commit hooks pass.

## 6. Verification

- Before/after run of the single repro test.
- Before/after run of `.venv/bin/pytest tests/unit -q`, comparing the failure list (not just
  counts) to confirm nothing new breaks.
- A new test asserting that a structlog event is captured by `caplog` with the expected
  level, so the behavior is protected against regression.

## 7. Open questions for maintainers

- Preference between reusing `configure_logging()` (option A) and a test-specific structlog
  configuration (option B)? I chose B for assertion stability but will follow the project's
  preference.
- Should `configure_logging()` also be wired into `api/main.py` startup? It appears to be
  called nowhere but the seed script, which looks like a separate gap — happy to file a
  follow-up issue rather than widen this PR.

## 8. Out of scope

The other 52 baseline failures (`test_review_service.py` async mocks — #158; skill
extractor — #148; structural chunker — #149; tech detector — #150). This PR stays limited
to log capture in tests.
