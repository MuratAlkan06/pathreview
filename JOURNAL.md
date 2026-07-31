# Module 3 Journal — Murat Alkan (MuratAlkan06)

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/159

**Issue title:** structlog output is not captured by pytest caplog — log assertions fail suite-wide

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The application logs through structlog, but structlog is never wired into Python's
standard logging system during tests. pytest's `caplog` fixture only captures records
that flow through stdlib `logging`, so tests that assert on log output fail even though
the code under test emits the expected event. I reproduced this locally: running
`tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty`
shows the warning "Empty chunks list provided to BatchEmbeddingProcessor" printed to
stdout while `caplog.text` is empty. A successful fix configures structlog in
`tests/conftest.py` to route events through stdlib logging so caplog assertions pass,
without changing production logging behavior. Today only `test_batch_processor.py`
asserts via caplog (1 failing test out of a 53-failure baseline — the rest belong to
other tracked issues like #158/#148/#149/#150), but the fix makes log assertions
reliable for the entire suite going forward.

**Branch name:** fix/159-structlog-caplog-capture

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/MuratAlkan06/pathreview/commit/36b4af6cde358270434e94473e2a9970031e2926

**Reproduction summary:**
Ran `tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty` and observed the warning "Empty chunks list provided to BatchEmbeddingProcessor" in captured stdout while `caplog.text` stayed empty — confirming structlog's default `PrintLoggerFactory` bypasses stdlib logging. Full-suite baseline: 53 failed / 375 passed, with exactly one failure belonging to this issue.

**PLAN.md link:** https://github.com/MuratAlkan06/pathreview/blob/fix/159-structlog-caplog-capture/PLAN.md

**Walkthrough video (recommended):** https://www.loom.com/share/ffa77d3f137d45bcb8becef73ef54858

**Blockers or open questions:**
Maintainer preference between reusing `core.logging.configure_logging()` in tests (Option A) vs. the test-scoped structlog config I implemented (Option B); `configure_logging()` is never called at API startup — likely a separate issue to file; repo-wide `ruff`/`black`/`mypy` are already failing on `main`, so lint acceptance is scoped to the files this fix touches.
