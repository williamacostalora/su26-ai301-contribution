# Contribution #1: Better Support for Apple Silicon Chips in CodeCarbon

**Contribution Number:** 1
**Student:** William Acosta Lora
**Issue:** https://github.com/mlco2/codecarbon/issues/758
**Status:** Phase IV — Complete

---

## Why I Chose This Issue

CodeCarbon is a Python library that tracks the carbon footprint of machine learning compute — something directly relevant to my work in AI fairness and responsible ML. The issue is straightforward: Apple Silicon chips from the M2 generation onward are not properly recognized by CodeCarbon's TDP lookup table, so users on modern Macs silently fall back to a constant CPU power estimate instead of a real measurement. That means anyone running ML workloads on an M3 or M4 Mac is getting inaccurate emissions data without knowing it.

I chose this issue because the maintainer already pointed to the exact lines of code that need changing. The fix spans two concrete files: adding rows to a CSV data file and writing tests. This is well within my Python skill set, it requires no special hardware, and it contributes to the accuracy of ML environmental impact reporting.

---

## Understanding the Issue

### Problem Description
CodeCarbon's CPU power detection only has an entry for `Apple M1` in its TDP lookup table (`cpu_power.csv`). Users on M2, M3, or M4 Macs receive a `"We saw that you have a Apple M3 Pro but we don't know it"` warning and fall back to a constant estimate of 4W × number of CPU threads, producing inaccurate emissions data.

### Expected Behavior
CodeCarbon should detect M2, M3, and M4 series chips and look up their real TDP values from `cpu_power.csv`, the same way M1 is handled today.

### Current Behavior
Any Apple Silicon chip beyond M1 triggers the fallback warning. For example, an M3 Pro on an 8-core machine incorrectly reports 32W (4W × 8) instead of the correct ~18W TDP.

### Affected Components
- `codecarbon/data/hardware/cpu_power.csv` — missing TDP entries for M2/M3/M4 variants
- `tests/test_cpu.py` — needed new test cases for the new chip entries

---

## Reproduction Process

### Environment Setup
- macOS (Apple M1), Python 3.13.2
- Installed `uv` via `curl -LsSf https://astral.sh/uv/install.sh | sh`
- Cloned fork and ran `uv sync` and `uv run task pre-commit-install`
- One gotcha: `uv` wasn't on PATH until I ran `source ~/.zshrc`

### Steps to Reproduce
1. Clone the repository and run `uv sync`
2. Run the following to simulate an unrecognized chip:
```python
from unittest.mock import patch
from codecarbon.core.cpu import TDP

with patch('codecarbon.core.cpu.detect_cpu_model', return_value='Apple M3 Pro'):
    t = TDP()
    print('Model:', t.model)
    print('TDP:', t.tdp)
```
3. **Expected:** TDP of 18W (Apple M3 Pro spec)
4. **Actual:** Warning logged, TDP falls back to 32W (4W × 8 threads)

### Reproduction Evidence
- **Branch:** https://github.com/williamacostalora/codecarbon/tree/fix-issue-758
- **Reproduction output:**
[codecarbon WARNING] We saw that you have a Apple M3 Pro but we don't know it. Please contact us.

[codecarbon WARNING] We will use the default power consumption of 4 W per thread for your 8 CPU, so 32W.

Model: Apple M3 Pro

TDP: 32

---

## Solution Approach

### Analysis
The root cause is simply missing rows in `codecarbon/data/hardware/cpu_power.csv`. The `TDP` class in `cpu.py` calls `detect_cpu_model()`, then looks up the result in the CSV via fuzzy string matching. If no match is found, it falls back to a per-thread estimate. M2/M3/M4 chips were never added to the CSV.

### Proposed Solution
Add TDP entries for all current Apple Silicon variants to the CSV. No logic changes needed — the existing lookup and fuzzy matching already works correctly once the data is present.

### Implementation Plan (UMPIRE)

**Understand:** M2/M3/M4 Apple Silicon chips are absent from `cpu_power.csv`, causing CodeCarbon to fall back to a default power estimate for all modern Apple Mac users.

**Match:** The existing `Apple M1,10` entry in `cpu_power.csv` is the pattern to follow. The `TDP._get_matching_cpu()` method handles fuzzy lookup automatically.

**Plan:**
1. Add 12 new rows to `codecarbon/data/hardware/cpu_power.csv` for M2/M3/M4 variants with Apple-spec TDP values
2. Add 3 new test methods to `tests/test_cpu.py` (one per chip generation) verifying correct TDP lookup
3. Run full CPU test suite to confirm no regressions

**Implement:** https://github.com/williamacostalora/codecarbon/tree/fix-issue-758

**Review:** Followed project CONTRIBUTING.md — used `uv`, passed `pre-commit` (autoflake, isort, black, flake8), matched existing test style in `TestTDP` class.

**Evaluate:** 43/43 CPU tests pass including 4 new Apple Silicon tests (M2, M3, M4 chip TDP lookups, plus a fallback test for unrecognized future chips). Verified fix by mocking chip detection for M2, M3 Pro, and M4 Max — all return correct TDP with no warnings.

---

## Testing Strategy

### Unit Tests
- [x] M2 series chips (M2, M2 Pro, M2 Max, M2 Ultra) return correct TDP
- [x] M3 series chips (M3, M3 Pro, M3 Max, M3 Ultra) return correct TDP
- [x] M4 series chips (M4, M4 Pro, M4 Max, M4 Ultra) return correct TDP
- [x] Unrecognized future Apple chip still falls back gracefully (no crash, uses default estimate)
- [x] All 43 tests in `test_cpu.py` pass (41 pre-existing + 4 new... 2 skipped are platform-specific, unrelated)

### Manual Testing
Mocked `detect_cpu_model()` for each new chip variant and confirmed correct TDP returned with no fallback warning logged.
---

## Implementation Notes

### Week 1 Progress
Set up environment with `uv`, reproduced the bug via mocking, added 12 CSV entries and 3 test methods, all tests passing, committed and pushed to `fix-issue-758`.

### Week 2 Progress
- Rebased `fix-issue-758` onto latest `upstream/master` (28 commits behind → caught up cleanly, no conflicts)
- Added edge-case test confirming an unrecognized future chip (e.g. "Apple M5 Pro") still falls back gracefully to the existing default power estimate instead of erroring
- Re-ran full `tests/test_cpu.py` suite after rebase to confirm no regressions
- All pre-commit hooks (autoflake, isort, black, flake8) passing on every commit

### Code Changes
- **Files modified:** `codecarbon/data/hardware/cpu_power.csv`, `tests/test_cpu.py`
- **Key commits:**
  - `fce86829` — feat: add Apple M2/M3/M4 chip TDP values to cpu_power.csv
  - `8a3e3889` — test: verify unknown Apple Silicon chips fall back gracefully
- **Branch:** https://github.com/williamacostalora/codecarbon/tree/fix-issue-758

### Challenges Faced
- Initial `uv` PATH issue after install — resolved with `source ~/.zshrc`
- Forgot to add `upstream` remote, so my fork was 28 commits behind `mlco2/codecarbon:master` by the time I came back to do Phase III. Fixed with `git remote add upstream` + `git rebase upstream/master` — clean rebase, no conflicts since my changes were isolated to two files
- Mock patching gotcha: had to patch `codecarbon.core.cpu.detect_cpu_model` (where it's imported/used) rather than `codecarbon.core.util.detect_cpu_model` (where it's defined) for the mock to take effect

---

## Pull Request

**PR Link:** https://github.com/mlco2/codecarbon/pull/1256

**PR Description:**
What does this PR do?: Adds TDP (Thermal Design Power) entries for Apple M2, M3, and M4 chip variants to `cpu_power.csv`, and adds unit tests covering correct TDP lookup for all new entries plus a fallback test for unrecognized future Apple chips.

Why was this PR needed?: CodeCarbon's CPU power detection only had an entry for `Apple M1` in its TDP lookup table. Users on M2, M3, or M4 Macs received a fallback warning and an inaccurate generic power estimate (4W per CPU thread), producing inaccurate emissions data for a large and growing portion of ML practitioners on Apple Silicon.

What are the relevant issue numbers?: Fixes #758

Does this PR meet the acceptance criteria?:
- [x] Tests added for new behavior
- [x] All tests passing (43/43, 2 pre-existing skips unrelated to this change)
- [x] Follows project style guide (passed autoflake, isort, black, flake8)
- [x] No breaking changes

**Maintainer Feedback:**
- No direct maintainer review received yet after initial submission, but on re-reading the original issue thread I noticed my first-pass TDP values didn't match the sourced table @makoeppel had already posted there (NotebookCheck, CPU Monkey). Proactively corrected this before further review rather than waiting for a maintainer to flag it.
- Pushed a follow-up commit (`7d97afe`) correcting all M2/M3/M4 TDP values to align with the issue's sourced data, and added a "Note on TDP value sourcing" section to the PR description, being transparent about which values are direct sources vs. extrapolated (M3 Ultra, M4 Pro/Max/Ultra had no published source, so these are extrapolated proportionally from M3-generation equivalents).
- Closed a duplicate PR (#1255) I had accidentally opened on the same branch.
- Left a comment on the PR summarizing the correction and inviting feedback on the extrapolated values.

**Status:** Awaiting review
---
## Learnings & Reflections

### Technical Skills Gained
- Practical experience with `uv` for Python dependency management
- Writing and debugging `unittest.mock.patch` correctly — specifically understanding that you must patch where a function is *used* (`codecarbon.core.cpu.detect_cpu_model`), not where it's *defined* (`codecarbon.core.util.detect_cpu_model`)
- Rebasing a long-lived feature branch cleanly onto a fast-moving upstream
- Writing a transparent, well-sourced PR description that documents data provenance, not just code changes

### Challenges Overcome
- Initial `uv` PATH issue after install, resolved by sourcing `~/.zshrc`
- Fork fell 28 commits behind upstream between Phase II and Phase III; resolved by adding the `upstream` remote and rebasing
- Caught a data-quality issue in my own PR (TDP values didn't match the maintainer-sourced table already posted in the issue) before a reviewer had to flag it, and corrected it proactively with a transparent explanation in the PR description

### What I'd Do Differently Next Time
I'd cross-reference all existing issue comments for sourced data *before* writing my first implementation, rather than estimating values and correcting them afterward. The maintainer-sourced TDP table was sitting in the issue thread from the start — reading it more carefully up front would have saved a correction cycle.

## Resources Used
- [Issue #758](https://github.com/mlco2/codecarbon/issues/758)
- [CodeCarbon CONTRIBUTING.md](https://github.com/mlco2/codecarbon/blob/master/CONTRIBUTING.md)
- [cpu_power.csv](https://github.com/mlco2/codecarbon/blob/master/codecarbon/data/hardware/cpu_power.csv)
- [Apple Silicon TDP specs](https://www.apple.com/mac/compare/)
