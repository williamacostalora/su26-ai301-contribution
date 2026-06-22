# Contribution #1: Better Support for Apple Silicon Chips in CodeCarbon

**Contribution Number:** 1
**Student:** William Acosta Lora
**Issue:** https://github.com/mlco2/codecarbon/issues/758
**Status:** Phase III — Complete

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

## Resources Used
- [Issue #758](https://github.com/mlco2/codecarbon/issues/758)
- [CodeCarbon CONTRIBUTING.md](https://github.com/mlco2/codecarbon/blob/master/CONTRIBUTING.md)
- [cpu_power.csv](https://github.com/mlco2/codecarbon/blob/master/codecarbon/data/hardware/cpu_power.csv)
- [Apple Silicon TDP specs](https://www.apple.com/mac/compare/)
