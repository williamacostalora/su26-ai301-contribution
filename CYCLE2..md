# Contribution #2: Fix Python 3.13 Warnings in PyIceberg

**Contribution Number:** 2
**Student:** William Acosta Lora
**Issue:** https://github.com/apache/iceberg-python/issues/2530
**Fork:** https://github.com/williamacostalora/iceberg-python
**Branch:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530
**Status:** Phase IV — Complete | PR Open, Awaiting Review

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
Three warning filters in `pyproject.toml` suppress legitimate Python 3.13 resource warnings rather than fixing the underlying leak:

"ignore:unclosed database in <sqlite3.Connection object*:ResourceWarning",
"ignore:unclosed file:ResourceWarning",
"ignore:subprocess.*is still running:ResourceWarning",


### Affected Components
- `pyproject.toml` lines 175-177 — the three warning filters to remove
- `tests/catalog/test_sql.py` — `catalog_memory`, `catalog_sqlite`, `alchemy_engine` fixtures and 4 inline test functions missing `catalog.close()`
- `tests/catalog/test_bigquery_metastore.py` — `BigQueryMetastoreCatalog` instances not closed after use
- `tests/table/test_pyarrow.py` — `InMemoryCatalog` fixture not closed
- `tests/table/test_inspect.py` — `InMemoryCatalog` fixture not closed
- `tests/table/test_datafusion.py` — catalog fixture not closed
- `tests/table/test_upsert.py` — catalog fixture not closed
- `tests/conftest.py` — `catalog` and `catalog_with_warehouse` fixtures missing `catalog.close()`
- `pyiceberg/catalog/sql.py` line 777 — `close()` method already correctly implemented, just never called by fixtures

### Acceptance Criteria
- [x] Warning filter for sqlite (`unclosed database`) removed from `pyproject.toml`
- [ ] Warning filters for ray (`unclosed file`, `subprocess still running`) removed — pending CI verification with ray installed
- [x] All 17 CI checks passing with zero ResourceWarnings
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
- `uv` was not on PATH after install — resolved by running `source ~/.zshrc` and adding `export PATH="$HOME/.local/bin:$PATH"` to `~/.zshrc` permanently
- `ray` is an optional dependency not installed locally — scoped local reproduction to sqlite warnings only; ray fix requires CI verification
- `ruff` and `pre-commit` not installed by default — resolved with `uv add --dev ruff pre-commit`
- CI failed on first push due to ruff formatting — resolved by running `uv run ruff format .` locally before committing
- Initial fix only caught connections in `tests/catalog/test_sql.py` and `tests/conftest.py` — CI revealed unclosed connections in 4 additional test files (`test_bigquery_metastore.py`, `test_pyarrow.py`, `test_inspect.py`, `test_datafusion.py`)

### Steps to Reproduce
1. Clone the repository: `git clone https://github.com/apache/iceberg-python.git && cd iceberg-python`
2. Install dev dependencies: `uv sync --extra dev`
3. Temporarily comment out the sqlite warning filter on line 175 of `pyproject.toml`
4. Run the SQL catalog test suite with ResourceWarnings promoted to errors: `uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -v`
5. Observe `ResourceWarning: unclosed database in <sqlite3.Connection object at 0x...>` emitted from SQLAlchemy's `langhelpers.py` during fixture teardown

### Expected vs. Actual

**Expected:**

All tests PASSED — 0 ResourceWarnings emitted


**Actual (without warning filters):**

ResourceWarning: unclosed database in <sqlite3.Connection object at 0x10a3e17b0>
FAILED tests/catalog/test_sql.py::test_creation_when_no_tables_exist
FAILED tests/catalog/test_bigquery_metastore.py::test_create_table_with_database_location
ResourceWarning: unclosed database in <sqlite3.Connection object at 0x7f9ad2354400>


**After fix:**

17/17 CI checks passing — zero ResourceWarnings across all Python versions (3.10–3.14)


### Root Cause
Numerous test fixtures and inline test functions across 6 test files create `SqlCatalog` or catalog-backed instances using SQLAlchemy but never call `catalog.close()` during teardown. The `close()` method in `pyiceberg/catalog/sql.py` (line 777) calls `self.engine.dispose()` which properly closes all SQLAlchemy connection pool connections — but it was never invoked. Python 3.13 tightened resource leak detection and now emits `ResourceWarning` for these unclosed connections that earlier versions ignored silently.

### Investigative Depth
Used `git log` to trace when the warning filters were introduced — PR #2863 (`infra: Add Python 3.13 support`) merged January 5, 2026, commit `dea8078`. Maintainer @kevinjqliu confirmed in [this comment](https://github.com/apache/iceberg-python/issues/2530#issuecomment-3712112040): *"we had to filter out the warnings. Would be great to resolve the underlying issues."*

The analogous fix pattern already exists in the codebase — `TestSqlCatalogClose` class in `tests/catalog/test_sql.py` (line 213) explicitly tests that `catalog.close()` properly disposes the engine. The fix applies this same pattern across all affected test files.

---

## Solution Approach

### Implementation Plan (UMPIRE)

**Understand:**
Two categories of `ResourceWarning` suppressed by filters in `pyproject.toml`:
1. **sqlite:** Numerous test fixtures across 6 test files never call `catalog.close()` or `engine.dispose()` during teardown
2. **ray:** Subprocess/file handle cleanup race condition between `ray.shutdown()` and Python 3.13's GC

**Match:**
The `close()` method already exists in `pyiceberg/catalog/sql.py` (line 777). `TestSqlCatalogClose` in `test_sql.py` (line 213) even tests it. The fix pattern is: `yield catalog → catalog.destroy_tables() → catalog.close()` for fixtures, or `try/finally` with `catalog.close()` for inline catalog creation.

**Plan:**
1. Add `catalog.close()` / `engine.dispose()` to all affected locations across 6 test files
2. Remove sqlite warning filter from `pyproject.toml`
3. Add regression test to `TestSqlCatalogClose`
4. Add docstrings explaining the Python 3.13 connection lifecycle requirement
5. Run ruff formatting and pre-commit hooks before pushing

**Implement:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530

**Review:**
- Confirmed `catalog.close()` in fixture teardown does not affect tests that reuse the catalog across the module
- All pre-commit hooks pass: trim whitespace, ruff, mypy, pydocstyle, codespell, uv-lock
- 17/17 CI checks passing across Python 3.10, 3.11, 3.12, 3.13, 3.14

**Evaluate:**
`uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -v` → 13/13 passing, zero ResourceWarnings. Full CI: 17/17 passing.

---

## Testing Strategy

### Unit Tests
- [x] `catalog_memory` fixture — `catalog.close()` added to teardown
- [x] `catalog_sqlite` fixture — `catalog.close()` added to teardown
- [x] `alchemy_engine` fixture — converted to `yield` with `engine.dispose()` teardown
- [x] 4 inline catalog creation tests in `test_sql.py` — wrapped in `try/finally` with `catalog.close()`
- [x] `tests/conftest.py` `catalog` and `catalog_with_warehouse` fixtures — `catalog.close()` added
- [x] `tests/catalog/test_bigquery_metastore.py` — all catalog instances properly closed
- [x] `tests/table/test_pyarrow.py` — `InMemoryCatalog` fixture closed
- [x] `tests/table/test_inspect.py` — `InMemoryCatalog` fixture closed
- [x] `tests/table/test_datafusion.py` — catalog fixture closed
- [x] `tests/table/test_upsert.py` — catalog fixture closed
- [x] NEW: `test_catalog_fixture_closes_connections` — regression test verifying no ResourceWarning after `close()`
- [x] 17/17 CI checks passing across all Python versions
- [ ] Ray warning fix — requires CI environment with ray installed

### Manual Testing
Reproduced the warning by removing the sqlite filter from `pyproject.toml` and running with `-W error::ResourceWarning`. Confirmed all tests pass after applying fixes with zero ResourceWarnings about unclosed database connections.

---

## Implementation Notes

### Week 1 Progress
- Set up environment, forked repo, created `fix-issue-2530` branch
- Located warning filters in `pyproject.toml` (lines 175-177)
- Traced root cause to test fixtures missing `catalog.close()` / `engine.dispose()` calls
- Applied initial sqlite fixes in `tests/catalog/test_sql.py` and `tests/conftest.py`
- Removed sqlite `ResourceWarning` filter from `pyproject.toml`
- 13/13 SQL catalog tests passing with `-W error::ResourceWarning`

### Week 2 Progress
- Added regression test `test_catalog_fixture_closes_connections` to `TestSqlCatalogClose`
- Added docstrings to fixtures explaining Python 3.13 connection lifecycle requirement
- Added clarifying comments to `pyproject.toml` distinguishing completed sqlite fix from pending ray fix
- Enhanced `SqlCatalog.close()` docstring with Python 3.13 guidance
- Documented duck typing pattern (`hasattr(cat, "close")`) in `tests/conftest.py`
- Fixed CI ruff formatting failure — ran `uv run ruff format .` and committed (`9cf77b0`)
- Opened draft PR #3688 against `apache/iceberg-python` main

### Week 3 Progress
- CI revealed unclosed connections in 4 additional test files not caught locally
- Fixed `tests/catalog/test_bigquery_metastore.py` — commit `f555e51`
- Fixed `tests/table/test_pyarrow.py` — commit `f555e51`
- Fixed `tests/table/test_inspect.py` — commit `20d352c`
- Fixed `tests/table/test_datafusion.py` and `tests/table/test_upsert.py` — commit `9b5680a`
- All 17 CI checks now passing — PR marked ready for review
- @mentioned @Fokko and @kevinjqliu on the PR

### Code Changes
- **Files modified:** `tests/catalog/test_sql.py`, `tests/catalog/test_bigquery_metastore.py`, `tests/table/test_pyarrow.py`, `tests/table/test_inspect.py`, `tests/table/test_datafusion.py`, `tests/table/test_upsert.py`, `tests/conftest.py`, `pyiceberg/catalog/sql.py`, `pyproject.toml`
- **Key commits:**
  - `83f789d` — fix: close SQLAlchemy connections in test fixtures for Python 3.13
  - `01f3b56` — test: add regression test for Python 3.13 ResourceWarning fix
  - `69c5a95` — docs: add docstrings to catalog fixtures explaining Python 3.13 fix
  - `7281f05` — docs: clarify ray warning filter and sqlite connection fix in pyproject.toml
  - `c8db549` — docs: enhance close() method docstring with Python 3.13 guidance
  - `5f0fe3c` — chore: document duck typing pattern in catalog fixtures
  - `9cf77b0` — style: apply ruff formatting to pass CI lint checks
  - `f555e51` — fix: close leaked InMemoryCatalog in test_pyarrow.py catalog fixture
  - `20d352c` — fix: close leaked InMemoryCatalog in test_inspect.py catalog fixture
  - `9b5680a` — fix: close leaked catalogs in test_datafusion.py and test_upsert.py fixtures
- **Branch:** https://github.com/williamacostalora/iceberg-python/tree/fix-issue-2530

### Challenges Faced
- `uv` not on PATH after install — resolved with `source ~/.zshrc`
- `ray` not installed locally — scoped to sqlite fixes; ray filters remain pending CI
- CI failed on first push due to ruff formatting — resolved with `uv run ruff format .`
- Initial fix only caught connections in 2 files — CI revealed 4 additional affected test files requiring fixes across `test_bigquery_metastore.py`, `test_pyarrow.py`, `test_inspect.py`, `test_datafusion.py`, and `test_upsert.py`

### Testing Notes
- **Automated:** `uv run python -m pytest tests/catalog/test_sql.py -W error::ResourceWarning -q` → 13 passed, 0 warnings
- **New test:** `TestSqlCatalogClose::test_catalog_fixture_closes_connections` — regression test using `warnings.catch_warnings()` and `gc.collect()` to verify no ResourceWarning after `catalog.close()`
- **CI:** 17/17 checks passing across Python 3.10, 3.11, 3.12, 3.13, 3.14
- **Ray:** Cannot test locally — requires CI with ray installed

---

## Pull Request

**PR Link:** https://github.com/apache/iceberg-python/pull/3688

**PR Description:**
Closes #2530. Fixes unclosed SQLAlchemy database connections in test fixtures causing `ResourceWarning` under Python 3.13. Numerous locations across 6 test files created catalog instances without calling `catalog.close()` during teardown. Removed the sqlite `ResourceWarning` filter from `pyproject.toml` — no longer needed. Ray-related filters left in place pending CI verification.

**Before/After Evidence:**
Before (without warning filters):

FAILED tests/catalog/test_sql.py::test_creation_when_no_tables_exist
ResourceWarning: unclosed database in <sqlite3.Connection object at 0x10a3e17b0>
FAILED tests/catalog/test_bigquery_metastore.py::test_create_table_with_database_location
ResourceWarning: unclosed database in <sqlite3.Connection object at 0x7f9ad2354400>

After fix:

17/17 CI checks passing — zero ResourceWarnings across Python 3.10–3.14


**Maintainer Feedback:**
- **July 20, 2026 (CI failure — self-identified):** CI failed on Python 3.13 due to ruff formatting. Fixed by running `uv run ruff format .` locally — commit `9cf77b0`.
- **July 26, 2026 (CI failure — self-identified):** CI failed on Python 3.13 due to unclosed connections in `test_bigquery_metastore.py`, `test_pyarrow.py`, `test_inspect.py`, `test_datafusion.py` — files outside initial local scope. Fixed across all remaining files — commits `f555e51`, `20d352c`, `9b5680a`.
- **July 26, 2026:** All 17 CI checks passing. PR marked ready for review. @mentioned @Fokko and @kevinjqliu.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained
- Deep understanding of SQLAlchemy connection pool lifecycle — specifically how `engine.dispose()` closes all pooled connections and why Python 3.13's stricter GC catches leaks that earlier versions ignored
- Learned the difference between suppressing a warning (masking a bug) vs. fixing the root cause — a distinction that matters in production data pipeline code where connection leaks cause real performance issues
- Practical experience with pytest fixture teardown patterns using `yield` — understanding that code after `yield` runs as cleanup regardless of test outcome
- Learned how to use `warnings.catch_warnings()` and `gc.collect()` together in regression tests to reliably catch resource leaks
- Set up and used `uv`, `ruff`, and `pre-commit` in a large Apache project with complex CI/CD across 5 Python versions
- Understood duck typing patterns in Python (`hasattr(cat, "close")`) as a way to handle optional cleanup methods across different catalog implementations (REST, Glue, SQL catalogs all have different interfaces)

### Challenges Overcome
- `ray` is an optional dependency not installable locally — had to scope the contribution to the sqlite fix only and explicitly document why the ray filters remain. This required communicating clearly in the PR description so maintainers understand the intentional descoping
- CI failed on first push due to ruff formatting — learned to always run `uv run ruff format .` locally before pushing to an Apache project, and to install `pre-commit` hooks early in setup
- The initial fix only resolved connections in 2 files — CI revealed 4 additional affected test files. This taught me to run the full test suite with strict warning flags across ALL test files locally, not just the ones I edited

### What I'd Do Differently Next Time
Install and run `pre-commit` hooks at the very beginning of setup — before writing a single line of code — so formatting issues are caught at commit time rather than after pushing to CI. I would also run the full test suite with `-W error::ResourceWarning` across ALL test directories before opening the PR, rather than just the files I directly modified. This would have caught the additional 4 failing test files locally instead of through CI failures.

### Teachable Insight for Future Contributors
When a project suppresses warnings instead of fixing them, the suppression comment is your treasure map. Find where the filter was added (`git log -S "ignore:unclosed"` or `git blame pyproject.toml`), read the PR that added it, read the maintainer's comment explaining why it was done as a workaround — that comment will almost always tell you exactly what the real fix should be. In this case @kevinjqliu's comment said "would be great to resolve the underlying issues" and linked to the exact lines. The maintainer did half the work already; you just had to follow the breadcrumbs. Also: always run your fix against the ENTIRE test suite with strict warning flags, not just the files you touched — resource leaks in a codebase tend to cluster by pattern, and you'll find more than you expected.

---

## Resources Used
- [Issue #2530](https://github.com/apache/iceberg-python/issues/2530)
- [PR #2863 — warning filters added here](https://github.com/apache/iceberg-python/pull/2863/files#diff-50c86b7ed8ac2cf95bd48334961bf0530cdc77b5a56f852c5c61b89d735fd711R164-R168)
- [Maintainer comment from @kevinjqliu pointing at exact lines](https://github.com/apache/iceberg-python/issues/2530#issuecomment-3712112040)
- [PyIceberg CONTRIBUTING.md](https://github.com/apache/iceberg-python/blob/main/CONTRIBUTING.md)
- [SQLAlchemy engine.dispose() docs](https://docs.sqlalchemy.org/en/14/core/connections.html#engine-disposal)
- [Python 3.13 ResourceWarning docs](https://docs.python.org/3/library/warnings.html#warning-categories)
- [pytest fixture teardown docs](https://docs.pytest.org/en/stable/how-to/fixtures.html#yield-fixtures-recommended)
