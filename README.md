# Contribution #1: Better Support for Apple Silicon Chips in CodeCarbon

**Contribution Number:** 1
**Student:** William Acosta Lora
**Issue:** https://github.com/mlco2/codecarbon/issues/758
**Status:** Phase II — Complete

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

**Evaluate:** 42/42 CPU tests pass including 3 new Apple Silicon tests. Verified fix by mocking chip detection for M2, M3 Pro, and M4 Max — all return correct TDP with no warnings.

---

## Testing Strategy

### Unit Tests
- [x] Test that M2 series chips return correct TDP values
- [x] Test that M3 series chips return correct TDP values  
- [x] Test that M4 series chips return correct TDP values
- [x] All 42 existing CPU tests still pass

### Manual Testing
Mocked `detect_cpu_model` to return each chip variant — confirmed correct TDP returned with no fallback warning.

---

## Implementation Notes

### Week 1 Progress
Set up environment with `uv`, reproduced the bug via mocking, added 12 CSV entries and 3 test methods, all tests passing, committed and pushed to `fix-issue-758`.

### Code Changes
- **Files modified:** `codecarbon/data/hardware/cpu_power.csv`, `tests/test_cpu.py`
- **Key commit:** fce86829

---

## Pull Request
*(To be opened in Phase III)*

---

## Learnings & Reflections
*(To be filled in Phase IV)*

---

## Resources Used
- [Issue #758](https://github.com/mlco2/codecarbon/issues/758)
- [CodeCarbon CONTRIBUTING.md](https://github.com/mlco2/codecarbon/blob/master/CONTRIBUTING.md)
- [cpu_power.csv](https://github.com/mlco2/codecarbon/blob/master/codecarbon/data/hardware/cpu_power.csv)
- [Apple Silicon TDP specs](https://www.apple.com/mac/compare/)
