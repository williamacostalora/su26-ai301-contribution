# Contribution #1: Better Support for Apple Silicon Chips in CodeCarbon

**Contribution Number:** 1
**Student:** William Acosta Lora
**Issue:** https://github.com/mlco2/codecarbon/issues/758
**Status:** Phase I — Complete

---

## Why I Chose This Issue

CodeCarbon is a Python library that tracks the carbon footprint of machine learning compute — something directly relevant to my work in AI fairness and responsible ML. The issue is straightforward to articulate: Apple Silicon chips from the M2 generation onward are not properly recognized by CodeCarbon's TDP lookup table or detection logic, so users on modern Macs silently fall back to a constant CPU power estimate instead of a real measurement. That means anyone running ML workloads on an M3 or M4 Mac is getting inaccurate emissions data without knowing it.

I chose this issue because the maintainer has already pointed to the exact lines of code that need changing and even provided a partial TDP table in the comments — which means I can dive directly into implementation rather than spending weeks on archaeology. The fix spans two concrete files: adding rows to a CSV data file and updating a short string-matching block in the tracker. This is well within my Python skill set, it requires no special hardware (I can test the detection logic with mocking), it's genuinely unassigned, and it contributes to the accuracy of ML environmental impact reporting — a problem I want to help solve.

---

## Understanding the Issue

### Problem Description
CodeCarbon's CPU power detection only recognizes Apple M1 chips by name. Users on M2, M3, or M4 Macs receive a `"We saw that you have a Apple M3 Pro but we don't know it"` warning and fall back to a constant CPU power estimate, making emissions tracking inaccurate for a large and growing portion of ML practitioners.

### Expected Behavior
CodeCarbon should detect M2, M3, and M4 series chips and look up their approximate TDP values from `cpu_power.csv`, the same way M1 is handled today.

### Current Behavior
The detection block in `emissions_tracker.py` only checks for `"M1"`, `"M2"`, and `"M3"` strings (M4 is missing), and `cpu_power.csv` is missing TDP entries for most Apple Silicon variants beyond M1.

### Affected Components
- `codecarbon/data/cpu_power.csv` — needs new rows for M2/M3/M4 chip variants with TDP values
- `codecarbon/emissions_tracker.py` (or `core/cpu.py`) — detection logic that checks for chip model strings

---

## Reproduction Process
*(To be filled in Phase II)*

---

## Solution Approach
*(To be filled in Phase II)*

---

## Testing Strategy

### Unit Tests
- [ ] Test that M2/M3/M4 chip strings are detected correctly by the detection logic
- [ ] Test that TDP lookup returns correct values for new CSV entries
- [ ] Test that unknown chips still fall back gracefully with a warning

### Integration Tests
- [ ] Full tracker init with a mocked M3/M4 CPU model string resolves to correct TDP

### Manual Testing
*(To be filled in Phase III)*

---

## Implementation Notes
*(Weekly updates in Phase III)*

---

## Pull Request
*(To be filled in Phase IV)*

---

## Learnings & Reflections
*(To be filled in Phase IV)*

---

## Resources Used
- [Issue #758 with maintainer TDP table](https://github.com/mlco2/codecarbon/issues/758)
- [CodeCarbon CONTRIBUTING.md](https://github.com/mlco2/codecarbon/blob/master/CONTRIBUTING.md)
- [emissions_tracker.py chip detection logic](https://github.com/mlco2/codecarbon/blob/master/codecarbon/emissions_tracker.py)
- [cpu_power.csv data file](https://github.com/mlco2/codecarbon/blob/master/codecarbon/data/cpu_power.csv)
