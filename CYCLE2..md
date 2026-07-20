# Contribution #2: Fix Python 3.13 Warnings in PyIceberg

**Contribution Number:** 2
**Student:** William Acosta Lora
**Issue:** https://github.com/apache/iceberg-python/issues/2530
**Fork:** https://github.com/williamacostalora/iceberg-python
**Branch:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530
**Status:** Phase III — Complete | Draft PR open, awaiting CI

---

## Why I Chose This Issue

PyIceberg is Apache's Python implementation of the Iceberg table format, used in production data lakes and MLOps pipelines at companies like Netflix, Apple, and LinkedIn. I chose this issue because it sits at the intersection of two things I care about: data pipeline reliability and Python ecosystem hygiene. The problem is concrete — Python 3.13 support was added in PR #2863 but the underlying sqlite and ray `ResourceWarning`s were suppressed with filters in `pyproject.toml` rather than fixed at the source, meaning every developer running the test suite on Python 3.13 sees noise that masks real resource leaks. My specific learning goal is to understand how SQLAlchemy connection pools manage sqlite connections under the hood, and how Python 3.13 tightened resource management detection compared to earlier versions. This connects directly to my data pipeline work at Ecolab with Snowflake and production data infrastructure, where connection lifecycle management is critical.

---

## Understanding the Issue

### Problem Description
When PyIceberg's test suite runs under Python 3.13, two categories of `ResourceWarning` are emitted: (1) sqlite unclosed database connections left open by SQLAlchemy's connection pool after test teardown, and (2) ray unclosed file handles and subprocesses during ray shutdown. Rather than fixing these at the source, PR #2863 added three warning filters in `pyproject.toml` to suppress them. Python 3.13 introduced stricter resource leak detection that catches connections that earlier versions silently ignored — the suppression filters are masking real bugs, not resolving them.

### Expected Behavior
PyIceberg's test suite should run cleanly under Python 3.13 with zero `ResourceWarning` emissions from sqlite or ray. The three warning filters added in PR #2863 should be fully removable without any tests failing.

### Current Behavior
Three warning filters in `pyproject.toml` (lines 175-177) suppress legitimate Python 3.13 resource warnings rather than fixing the underlying leak:
"ignore:unclosed database in <sqlite3.Connection object*:ResourceWarning",
"ignore:unclosed file:ResourceWarning",
"ignore:subprocess.*is still running:ResourceWarning",

### Affected Components
- `pyproject.toml` lines 175-177 — the three warning filters to remove
- `tests/catalog/test_sql.py` lines 50-59 (`catalog_memory` fixture) and lines 62-72 (`catalog_sqlite` fixture) — neither called `catalog.close()` during teardown, leaving SQLAlchemy engine connection pools open
- `pyiceberg/catalog/sql.py` line 777 — `close()` method already correctly implemented (calls `self.engine.dispose()`), just never called by fixtures
- `tests/conftest.py` lines 3143-3165 — `catalog` and `catalog_with_warehouse` fixtures also missing `catalog.close()` teardown
- Warning filter lines added in PR #2863: https://github.com/apache/iceberg-python/pull/2863/files#diff-50c86b7ed8ac2cf95bd48334961bf0530cdc77b5a56f852c5c61b89d735fd711R164-R168

### Acceptance Criteria
- [x] Warning filter for sqlite (`unclosed database`) removed from `pyproject.toml`
- [ ] Warning filters for ray (`unclosed file`, `subprocess still running`) removed from `pyproject.toml` — pending CI verification with ray installed
- [x] Full SQL catalog test suite passes under Python 3.13 with `-W error::ResourceWarning` (13/13)
- [x] New regression test added to prevent future regressions
- [x] No new warning suppression filters introduced as workarounds

---

## Reproduction Process

### Environment Setup
I used the README/CONTRIBUTING.md instructions approach to set up my local development environment:
1. Forked `apache/iceberg-python` to `github.com/williamacostalora/iceberg-python`
2. Cloned: `git clone https://github.com/williamacostalora/iceberg-python.git`
3. Created working branch: `git checkout -b fix-issue-2530`
4. Installed dependencies: `uv sync --extra dev`

**Challenges encountered and resolved:**
- `uv` was not on PATH after install — resolved by running `source ~/.zshrc` to reload shell config and adding `export PATH="$HOME/.local/bin:$PATH"` to `~/.zshrc` permanently
- `ray` is an optional dependency and not installed locally by default — scoped local reproduction to the sqlite warnings only and noted the ray fix as requiring CI verification
- `ruff` and `pre-commit` not installed by default — resolved with `uv add --dev ruff pre-commit` before running lint checks
- CI failed on first PR push due to ruff formatting issues — resolved by running `uv run ruff format .` locally and committing the formatting fix

### Steps to Reproduce
1. Clone the repository: `git clone https://github.com/apache/iceberg-python.git && cd iceberg-python`
2. Install dev dependencies: `uv sync --extra dev`
3. Temporarily comment out the sqlite warning filter on line 175 of `pyproject.toml`
4. Run the SQL catalog test suite with ResourceWarnings promoted to errors: `uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -v`
5. Observe `ResourceWarning: unclosed database in <sqlite3.Connection object at 0x...>` emitted from SQLAlchemy's `langhelpers.py` during fixture teardown

### Expected vs. Actual

**Expected:**
tests/catalog/test_sql.py - all tests PASSED
0 ResourceWarnings emitted

**Actual (without warning filters):**
ResourceWarning: unclosed database in <sqlite3.Connection object at 0x10a3e17b0>
Exception ignored in: <sqlite3.Connection object at 0x10a3e17b0>
pytest.PytestUnraisableExceptionWarning raised
FAILED tests/catalog/test_sql.py::test_creation_when_no_tables_exist
FAILED tests/catalog/test_sql.py::TestSqlCatalogClose::test_sql_catalog_context_manager
2 failed, 10 passed

**After fix:**
13 passed, 0 ResourceWarnings with -W error::ResourceWarning

### Root Cause
**sqlite:** Nine test fixtures and inline test functions across `tests/catalog/test_sql.py` and `tests/conftest.py` create `SqlCatalog` instances backed by SQLAlchemy but never call `catalog.close()` during teardown. The `close()` method in `pyiceberg/catalog/sql.py` (line 777) calls `self.engine.dispose()` which properly closes all SQLAlchemy connection pool connections — but it was never invoked. Python 3.13 tightened resource leak detection and now emits `ResourceWarning` for these unclosed connections that earlier versions ignored silently.

**ray:** The `ray_session` fixture in `tests/conftest.py` (line 3091) calls `ray.shutdown()` but ray's internal subprocess and file handle cleanup may not fully complete before Python 3.13's GC runs. Requires CI environment with ray installed to verify and fix.

### Investigative Depth
Used `git log` to trace when the warning filters were introduced — PR #2863 (`infra: Add Python 3.13 support`) merged January 5, 2026, commit `dea8078`, added all three warning filters as acknowledged workarounds. Maintainer @kevinjqliu confirmed in [this comment](https://github.com/apache/iceberg-python/issues/2530#issuecomment-3712112040): *"we had to filter out the warnings. Would be great to resolve the underlying issues."*

The analogous fix pattern already exists in the codebase — `TestSqlCatalogClose` class in `tests/catalog/test_sql.py` (line 213) explicitly tests that `catalog.close()` properly disposes the engine. The fix applies this same pattern to all fixtures that set up catalog instances.

---

## Solution Approach

### Implementation Plan (UMPIRE)

**Understand:**
Two categories of `ResourceWarning` are suppressed by filters in `pyproject.toml` rather than fixed at the source:
1. **sqlite:** 9 test fixtures/functions across `tests/catalog/test_sql.py` and `tests/conftest.py` never call `catalog.close()` or `engine.dispose()` during teardown
2. **ray:** Subprocess/file handle cleanup race condition between `ray.shutdown()` and Python 3.13's GC

**Match:**
The `close()` method already exists and is correctly implemented in `pyiceberg/catalog/sql.py` (line 777). The `TestSqlCatalogClose` class in `test_sql.py` (line 213) even tests it explicitly. The fix pattern is: `yield catalog → catalog.destroy_tables() → catalog.close()`.

**Plan:**
1. Add `catalog.close()` / `engine.dispose()` to all 9 affected locations in `tests/catalog/test_sql.py` and `tests/conftest.py`
2. Remove sqlite warning filter from `pyproject.toml`
3. Add regression test to `TestSqlCatalogClose`
4. Add docstrings and comments explaining the Python 3.13 connection lifecycle requirement
5. Run ruff formatting and pre-commit hooks before pushing

**Implement:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530

**Review:**
- Confirmed `catalog.close()` in fixture teardown does not affect tests that reuse the catalog across the module
- Confirmed removing the sqlite filter does not break any other tests
- All pre-commit hooks pass: trim whitespace, ruff, mypy, pydocstyle, codespell, uv-lock

**Evaluate:**
`uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -v` → 13/13 passing, zero ResourceWarnings. Ray fix pending CI verification.

---

## Testing Strategy

### Unit Tests
- [x] `catalog_memory` fixture — `catalog.close()` added to teardown
- [x] `catalog_sqlite` fixture — `catalog.close()` added to teardown
- [x] `alchemy_engine` fixture — converted to `yield` with `engine.dispose()` teardown
- [x] `test_creation_with_echo_parameter` — all catalogs wrapped in `try/finally` with `catalog.close()`
- [x] `test_creation_with_pool_pre_ping_parameter` — all catalogs properly closed
- [x] `test_creation_from_impl` — catalog closed in `finally`
- [x] `test_creation_when_no_tables_exist` — `try/finally` with `catalog.close()`
- [x] `test_creation_when_one_tables_exists` — `try/finally` with `catalog.close()`
- [x] `test_creation_when_all_tables_exists` — `try/finally` with `catalog.close()`
- [x] `tests/conftest.py` `catalog` fixture — `catalog.close()` added
- [x] `tests/conftest.py` `catalog_with_warehouse` fixture — `catalog.close()` added
- [x] NEW: `test_catalog_fixture_closes_connections` — regression test verifying no ResourceWarning after close()
- [x] 13/13 SQL catalog tests pass with `-W error::ResourceWarning`
- [ ] Ray warning fix — requires CI environment with ray installed

### Manual Testing
Reproduced the warning by removing the sqlite filter from `pyproject.toml` and running with `-W error::ResourceWarning`. Confirmed all 13 tests pass after applying fixes with zero ResourceWarnings about unclosed database connections.

---

## Implementation Notes

### Week 1 Progress
- Set up environment, forked repo, created `fix-issue-2530` branch
- Located warning filters in `pyproject.toml` (lines 175-177)
- Traced root cause to test fixtures missing `catalog.close()` / `engine.dispose()` calls across 9 locations
- Applied all sqlite fixes — 13/13 tests passing with `-W error::ResourceWarning`
- Removed sqlite `ResourceWarning` filter from `pyproject.toml`

### Week 2 Progress
- Added regression test `test_catalog_fixture_closes_connections` to `TestSqlCatalogClose`
- Added docstrings to `catalog_memory` and `catalog_sqlite` fixtures explaining Python 3.13 connection lifecycle
- Added clarifying comments to `pyproject.toml` distinguishing completed sqlite fix from pending ray fix
- Enhanced `SqlCatalog.close()` docstring in `pyiceberg/catalog/sql.py` with Python 3.13 guidance
- Documented duck typing pattern (`hasattr(cat, "close")`) in `tests/conftest.py` fixtures
- Fixed CI ruff formatting failure — ran `uv run ruff format .` locally and committed formatting fix
- Opened draft PR #3688 against `apache/iceberg-python` main

### Code Changes
- **Files modified:** `tests/catalog/test_sql.py`, `tests/conftest.py`, `pyiceberg/catalog/sql.py`, `pyproject.toml`
- **Key commits (7 total):**
  - `83f789d` — fix: close SQLAlchemy connections in test fixtures for Python 3.13
  - `01f3b56` — test: add regression test for Python 3.13 ResourceWarning fix
  - `69c5a95` — docs: add docstrings to catalog fixtures explaining Python 3.13 fix
  - `7281f05` — docs: clarify ray warning filter and sqlite connection fix in pyproject.toml
  - `c8db549` — docs: enhance close() method docstring with Python 3.13 guidance
  - `5f0fe3c` — chore: document duck typing pattern in catalog fixtures
  - `9cf77b0` — style: apply ruff formatting to pass CI lint checks
- **Branch:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530

### Challenges Faced
- `uv` not on PATH after install — resolved with `source ~/.zshrc`
- `ray` not installed locally — scoped to sqlite fixes; ray filters remain pending CI
- Initial fix only caught 2 of 9 unclosed connection sources — required deeper investigation
- CI failed on first push due to ruff formatting — resolved by installing ruff via `uv add --dev ruff` and running `uv run ruff format .`
- `pre-commit` not installed by default — resolved with `uv add --dev pre-commit` and `uv run pre-commit install`

### Testing Notes
- **Automated:** `uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -q` → 13 passed, 0 warnings
- **New test:** `TestSqlCatalogClose::test_catalog_fixture_closes_connections` — regression test using `warnings.catch_warnings()` and `gc.collect()` to verify no ResourceWarning after `catalog.close()`
- **Manual:** Removed sqlite filter from `pyproject.toml`, ran full suite with `-W error::ResourceWarning` — confirmed zero leaks
- **Ray:** Cannot test locally — requires CI with ray installed

---

## Pull Request

**PR Link:** https://github.com/apache/iceberg-python/pull/3688 (Draft)

**PR Description:**

*Rationale:* Python 3.13 introduced stricter resource leak detection that flags unclosed sqlite3 connections left open by SQLAlchemy connection pools after test teardown. PR #2863 added warning filters to suppress these rather than fixing the root cause. This PR resolves the underlying issue and removes the sqlite filter.

*What are the relevant issue numbers?:* Closes #2530

*Are these changes tested?:* Yes — 13/13 SQL catalog tests pass with `-W error::ResourceWarning`. Added `test_catalog_fixture_closes_connections` regression test.

*Are there any user-facing changes?:* No — all changes in test files and `pyproject.toml`. No public API changes.

**Maintainer Feedback:**
- No feedback yet — PR converted to draft while CI reruns after ruff formatting fix

**Status:** Draft — CI checks running, will mark ready for review once green

---

## Learnings & Reflections
*(To be filled in Phase IV)*

---

## Resources Used
- [Issue #2530](https://github.com/apache/iceberg-python/issues/2530)
- [PR #2863 — warning filters added here](https://github.com/apache/iceberg-python/pull/2863/files#diff-50c86b7ed8ac2cf95bd48334961bf0530cdc77b5a56f852c5c61b89d735fd711R164-R168)
- [Maintainer comment from @kevinjqliu pointing at exact lines](https://github.com/apache/iceberg-python/issues/2530#issuecomment-3712112040)
- [PyIceberg CONTRIBUTING.md](https://github.com/apache/iceberg-python/blob/main/CONTRIBUTING.md)
- [SQLAlchemy engine.dispose() docs](https://docs.sqlalchemy.org/en/14/core/connections.html#engine-disposal)
- [Python 3.13 ResourceWarning docs](https://docs.python.org/3/library/warnings.html#warning-categories)
- [pytest fixture teardown docs](https://docs.pytest.org/en/stable/how-to/fixtures.html#yield-fixtures-recommended)
