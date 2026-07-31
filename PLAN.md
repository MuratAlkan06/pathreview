# Fix Plan — Issue #159

**Issue:** [structlog output is not captured by pytest caplog — log assertions fail suite-wide](https://github.com/ascherj/pathreview/issues/159)
**Tier:** 1 · **Branch:** `fix/159-structlog-caplog-capture`

## Solution plan

**Issue:** #159 — structlog output is not captured by pytest caplog — log assertions fail
suite-wide (https://github.com/ascherj/pathreview/issues/159)

### Understand

**Expected behavior.** A test that exercises code emitting a structlog event can assert on it
through pytest's `caplog` fixture: `caplog.text` contains the message, and `caplog.records`
holds a `LogRecord` with the correct level, message and structured fields.

**Actual behavior.** The event is emitted and visible in captured stdout, but `caplog.text` is
empty and `caplog.records` is empty, so every caplog-based log assertion fails.

**Reproduction:**

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

**Root cause.** `caplog` only sees records that pass through the standard library `logging`
module. The interesting part is *why* structlog isn't reaching it:

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

### Map

**Files involved in the defect**

| File | Lines | Role |
|---|---|---|
| `core/logging.py` | 11-52 | `configure_logging()` — already sets `structlog.stdlib.LoggerFactory()` and `cache_logger_on_first_use=True`, ends in `structlog.dev.ConsoleRenderer()` (line 37) and calls `logging.basicConfig()` (line 48). Never invoked from tests or from `api/main.py`; sole caller is `scripts/seed_db.py:20`. |
| `ingestion/embeddings/batch_processor.py` | 7, 40 | Module-level `logger = structlog.get_logger()` bound at import time (line 7); emits `logger.warning("Empty chunks list provided to BatchEmbeddingProcessor")` (line 40) — the event the failing test asserts on. |
| `tests/unit/test_batch_processor.py` | 36, 42-43 | `test_empty_chunks_list_returns_empty(self, processor, caplog)` asserts `"Empty chunks list" in caplog.text` or `record.message` over `caplog.records`. The only caplog assertions in the repo today. |
| `tests/conftest.py` | whole file | The only conftest. Before the fix: two data fixtures, **no autouse fixtures**, no logging setup. |
| `pyproject.toml` | 83-90 | `[tool.pytest.ini_options]` sets `testpaths` and four markers only — **no** `log_level`, `log_cli` or other logging options, so pytest leaves the root logger at `WARNING`. |

**Blast radius**

- No test asserts on captured stdout/stderr, and no test uses `structlog.testing.capture_logs`,
  so **no existing test depends on the current print-to-stdout behavior.**
- Production behavior must not change: the fix belongs in test configuration only.

**Files actually touched by the fix**

| File | Change |
|---|---|
| `tests/conftest.py` | New autouse fixture `configure_structlog_for_tests` (lines 9-34) plus the `structlog` import. |
| `tests/unit/test_logging_capture.py` | New regression test module (46 lines): module-level `structlog.get_logger()` at line 16, `TestStructlogCaplogCapture` with `test_warning_is_captured_with_level_and_message` and `test_bound_values_do_not_leak_into_the_message`. |

No production module is edited.

### Plan

1. **Reproduce and capture a baseline failure list — done.** Run the single failing test to
   confirm the symptom (event in captured stdout, empty `caplog.text`), then run the full unit
   suite on `main` and save the sorted `FAILED` list: **53 failed / 375 passed**, of which
   exactly one belongs to #159.

2. **Evaluate the options — done.**

   | # | Approach | Trade-off |
   |---|---|---|
   | A | Autouse fixture in `tests/conftest.py` calling `core.logging.configure_logging()` | Tests exactly what ships, but the dev branch ends in `ConsoleRenderer`, so `record.message` becomes a whole pre-rendered (possibly ANSI-colored) line, making assertions fragile. Also triggers `logging.basicConfig()` as a side effect. |
   | B | **Autouse fixture in `tests/conftest.py` that configures structlog for tests with `structlog.stdlib.LoggerFactory` and a `structlog.stdlib.render_to_log_kwargs` final processor** | Routes events into stdlib logging with the event as a clean `record.message`. Minimal, self-contained, production config untouched. **Recommended.** |
   | C | Rewrite assertions to use `structlog.testing.capture_logs` | Fixes one test but not the general problem; the issue explicitly asks for caplog to work, and this changes test style repo-wide. |

   **Chosen: B.** It resolves the issue as stated (caplog-based assertions work), touches one
   test-support file, and leaves runtime logging untouched.

3. **Implement the autouse fixture in `tests/conftest.py` — done.**

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

   Decisions baked into that shape:

   - **`cache_logger_on_first_use=False` — deliberate divergence from production**, which uses
     `True`. Mirroring production here would introduce an order-dependency bug; full reasoning
     in *Risks & unknowns*.
   - **`wrapper_class=structlog.stdlib.BoundLogger`** so `logger.warning(...)` dispatches to the
     matching stdlib level method; that is what makes `record.levelname` correct rather than the
     level living only inside the event dict.
   - **`render_to_log_kwargs` as the sole processor** keeps `record.message` equal to the raw
     event string and moves bound key/values into `extra`, where stdlib turns them into record
     attributes. No timestamp/level/renderer processors are needed — stdlib supplies those, and
     omitting them is what avoids option A's pre-rendered, possibly ANSI-colored `record.message`.
   - **`reset_defaults()` teardown** so the fixture cannot leak test configuration into anything
     that expects structlog defaults.

4. **Add regression tests in `tests/unit/test_logging_capture.py` — done.** Bind a module-level
   `structlog.get_logger()` — the same import-time pattern the bug affected — and assert on
   `caplog.records`: level, message, and a structured attribute, plus `caplog.text`. Cover both
   the WARNING path (captured with no extra setup) and an INFO path wrapped in
   `caplog.at_level(logging.INFO)`.

5. **Verify by failure-list diff and scoped lint — done.** Re-run the targeted test, re-run the
   unit suite and diff the sorted `FAILED` list against the baseline (expect exactly one removal
   and no additions), re-run with `-m unit` to confirm the new file is collected under both
   invocations, and run `ruff`/`black` on the two touched files. Results in *Inputs & outputs*.

### Inputs & outputs

**Inputs — what the fix consumes.** Nothing at runtime. The change takes no configuration, no
environment variable and no new dependency; it consumes only pytest's fixture lifecycle. The
autouse fixture runs per test, applies a **test-time-only** structlog configuration, and tears it
down with `structlog.reset_defaults()`. `core/logging.configure_logging()` and every production
module are left exactly as they are.

**Outputs — what becomes observable.**

- Structlog events emitted anywhere under test are routed into stdlib `logging` and therefore
  into pytest's capture handler.
- `caplog.text` contains the event message.
- `caplog.records` carries `LogRecord`s whose `levelname` matches the structlog method called
  (`warning` → `WARNING`) and whose `message` is the clean, unrendered event string.
- Bound key/values (`logger.bind(k=v)` / `logger.warning(msg, k=v)`) land in `extra` and surface
  as **record attributes** (`record.k`), not as text glued into `record.message`.
- Suite totals move from **53 failed / 375 passed** to **52 failed / 378 passed** (one existing
  failure fixed, two new regression tests added).

**Acceptance criteria**

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

**Verification results** (this branch; `.venv`: Python 3.12, structlog 26.1.0, pytest 9.1.1):

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

### Risks & unknowns

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
- **Root logger stays at `WARNING` — constraint for future log assertions.** `pyproject.toml`
  sets no `log_level`/`log_cli` (see *Map*), so pytest leaves the root logger at `WARNING`.
  Assertions on `info`/`debug` events must wrap the call in `caplog.at_level(logging.INFO)`.
  The regression test exercises both paths. Anyone writing a log assertion without this will
  see an empty `caplog` and may mistake it for a recurrence of #159.
- **The fixture is autouse, so it applies to every current and future test**, including the
  integration suite. That is intended — the point is suite-wide caplog reliability — but the
  integration tests require Docker services and were **not** run locally, so the fixture's
  effect there is unverified. Risk is low (it only changes structlog routing and resets after
  each test), but it is an unknown until CI runs the full matrix.
- **`ruff`, `black` and `mypy` are already failing repo-wide on `main`** (numbers in the
  acceptance criteria above). That predates this branch and is untouched here, so lint
  acceptance is deliberately scoped to the two files this fix touches. It also means CI's
  `lint`/`format`/`typecheck` jobs will be red for reasons unrelated to this PR.

**Open questions for maintainers**

- Preference between reusing `configure_logging()` (option A) and a test-specific structlog
  configuration (option B)? I chose B for assertion stability but will follow the project's
  preference.
- Should `configure_logging()` also be wired into `api/main.py` startup? It appears to be
  called nowhere but the seed script, which looks like a separate gap — happy to file a
  follow-up issue rather than widen this PR.
- The `lint`, `format` and `typecheck` CI jobs are already failing on `main`. That predates
  this branch and is untouched here — worth its own issue rather than folding a repo-wide
  reformat into this fix?

### Edge cases

1. **Bound key/values must not leak into the message.** `logger.warning("msg", user_id=7)` must
   produce `record.message == "msg"` with `record.user_id == 7`, not `"msg user_id=7"`. This is
   exactly what `render_to_log_kwargs` guarantees (event → `msg`, remaining keys → `extra`), and
   it is asserted by `test_bound_values_do_not_leak_into_the_message`.
2. **INFO/DEBUG events are dropped without `caplog.at_level`.** No `log_level` is configured in
   `pyproject.toml:83-90`, so the root logger defaults to `WARNING`; an `info`/`debug` event is
   filtered by stdlib before it ever reaches the capture handler. Tests must use
   `caplog.at_level(logging.INFO)` (as `tests/unit/test_logging_capture.py:40` does). Warnings
   and above need no extra setup.
3. **Loggers bound at module import time.** `ingestion/embeddings/batch_processor.py:7` resolves
   its logger before any fixture runs. `cache_logger_on_first_use=False` makes the lazy proxy
   re-resolve on every call, so an import-time logger still picks up the per-test configuration
   regardless of which test touched it first. The regression test reproduces this pattern
   deliberately (module-level `structlog.get_logger()` at line 16).
4. **Configuration leaking between tests.** The fixture's `structlog.reset_defaults()` teardown
   returns structlog to its defaults after each test, so neither the test config nor a cached
   logger survives into the next test. Verified with order-shuffled runs producing identical
   results, and with the `-m unit` invocation collecting a different test subset.
5. **Pre-rendered or ANSI-colored messages.** Reusing the production config (option A) would run
   `structlog.dev.ConsoleRenderer` (`core/logging.py:37`) and hand caplog a fully rendered,
   possibly color-escaped line as `record.message`, breaking substring assertions. Avoided by
   not reusing the production processor chain in tests.

### Out of scope

The other 52 baseline failures (`test_review_service.py` async mocks — #158; skill
extractor — #148; structural chunker — #149; tech detector — #150). This PR stays limited
to log capture in tests.
