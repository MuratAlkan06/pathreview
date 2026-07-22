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
