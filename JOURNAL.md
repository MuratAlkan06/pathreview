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

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implementation is done: the autouse fixture in `tests/conftest.py` (PLAN.md
sub-task "implement Option B") and the two regression tests in
`tests/unit/test_logging_capture.py` are committed (973191f). Verified
locally against the baseline: 53 failed / 375 passed on `main` → 52 failed /
378 passed with the fix — a sorted failure-list diff shows exactly the one
target failure removed and no new ones.

**Next steps:**
Create the clean PR branch (`fix/159-caplog-structlog-config`) by
cherry-picking only the code commit so course artifacts stay out of the
upstream PR, write the PR description against the repo template, run the
self-review checklist, and submit the PR.

**Blockers:**
None blocking. Open question for reviewers: maintainer preference between
reusing `core.logging.configure_logging()` in tests (Option A) vs. the
test-scoped structlog config I implemented (Option B) — noted in the PR.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/740

**Branch:** `fix/159-structlog-caplog-capture` (working branch; PR submitted
from the clean single-commit branch `fix/159-caplog-structlog-config`)

**What you built:**
A test-scoped autouse fixture in `tests/conftest.py` that configures
structlog with `structlog.stdlib.LoggerFactory` and `render_to_log_kwargs`
(with `cache_logger_on_first_use=False`) so log events flow through stdlib
`logging`, where pytest's `caplog` can capture them. Production logging is
untouched; the fixture resets structlog defaults after each test.

**Tests added or updated:**
`tests/unit/test_logging_capture.py` — two regression tests: one asserts a
warning's message and level land on the captured record, one asserts bound
key/values (e.g. `batch_size=100`) arrive as record attributes instead of
leaking into the message. Both fail with the original symptom if run without
the fixture, so they guard against regressions.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
*(scoped to the files this PR touches — repo-wide `make check` and 52
unit-test failures already fail on upstream `main`, tracked in
#158/#148/#149/#150, and are unchanged by this PR)*

**Draft PR feedback received from:** none before submission — @ayc325
commented after the PR went up; documented and answered in Week 10.

## Week 10 — Iteration & reflection

### Reviewer feedback

**Feedback received:** [x] Yes  [ ] No — still awaiting review

**Summary of feedback:**
@ayc325 commented on PR #740 suggesting (1) splitting the change into more
commits mapped to PLAN.md subtasks, and (2) that PLAN.md and JOURNAL.md are
missing from the PR branch. No maintainer review, inline comments, or CI
feedback yet.

**How you responded:**
Replied on the PR explaining both points: PLAN.md/JOURNAL.md are course
artifacts intentionally kept out of the upstream PR — they live on my fork's
working branch, which I linked — and the PR is a single atomic commit because
the conftest fixture and its regression tests are inseparable (any split
leaves an intermediate commit with failing tests). Offered to restructure if
a maintainer prefers. No code changes were warranted.

---

### Reflection

**What was harder than you expected?**
Verification, not the fix. The fix is an 8-line autouse fixture; proving it
correct took most of the effort because `main` already had 53 failing tests,
so "the suite is green" was unavailable as evidence. I diffed sorted failure
lists before/after to show exactly one failure disappeared (the target test)
and zero new ones appeared. Two mechanism details also surprised me:
`cache_logger_on_first_use` had to be `False` because modules bind loggers at
import time — a cached logger would freeze the old stdout factory and
silently ignore my fixture — and caplog's root logger sits at WARNING by
default, so my INFO-level test needed `caplog.at_level(logging.INFO)` or the
event was dropped before caplog's handler ever saw it.

**What did you learn about working in a large codebase?**
The repo already contained the correct solution — `core/logging.py` has a
`configure_logging()` that wires structlog to stdlib properly — it just was
never called during tests (or at API startup). The job wasn't inventing a
fix; it was understanding why existing infrastructure never reached the
failure point and adding the smallest test-scoped bridge. Scope discipline
mattered just as much: 52 of the 53 baseline failures belonged to other
tracked issues (#158/#148/#149/#150) and repo-wide lint/type checks were
already failing on `main`, so "done" had to be defined per-file and
per-issue, not repo-wide. And conventions — conventional commits, the PR
template, branch naming, keeping course files out of upstream — carried as
much weight as the code itself.

**How did AI tools help — and where did they fall short?**
Most useful: root-cause tracing (following structlog's default
`PrintLoggerFactory` straight to stdout to explain the empty `caplog.text`),
generating the three-option analysis, and enforcing verification discipline
(baseline capture, sorted failure-list diffs, non-vacuity checks proving the
new tests fail without the fixture). It fell short in two ways: agent runs
died mid-task when my internet dropped and I had to reconstruct state
manually from the working tree; and it can't answer judgment calls that
belong to maintainers — Option A vs. Option B is a preference question I had
to leave as an explicit reviewer note in the PR rather than have AI decide.

**What would you do differently if you started over?**
Create the clean PR branch on day one and develop there, cherry-picking
course artifacts outward — instead of untangling them at the end. I'd also
match CI's Python 3.11 in my venv instead of running 3.12 and hoping the
difference never mattered, and I'd file the "`configure_logging()` is never
called at API startup" discovery as its own upstream issue immediately
rather than parking it in the PR notes.

**What are you most proud of from this module?**
The verification record: a sorted failure-list diff showing exactly one line
removed between 53F/375P and 52F/378P, plus proof the new regression tests
fail with the original symptom when run outside conftest's reach. Anyone
reviewing the PR can re-run every step.
