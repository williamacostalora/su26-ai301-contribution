# Contribution #2: Fix Python 3.13 Warnings in PyIceberg

**Contribution Number:** 2
**Student:** William Acosta Lora
**Issue:** https://github.com/apache/iceberg-python/issues/2530
**Status:** Phase I — Complete

---

## Why I Chose This Issue

PyIceberg is Apache's Python implementation of the Iceberg table format, widely used in production data lakes and MLOps pipelines. This issue is well-scoped: Python 3.13 support was added in PR #2863 but the underlying sqlite and ray deprecation warnings were suppressed with filters rather than properly resolved. The maintainer (@kevinjqliu) pointed directly at the lines to fix, which means I can start reproducing and implementing immediately rather than spending weeks on archaeology. This is directly relevant to my data pipeline interests and my Ecolab internship work with Snowflake and data infrastructure.

---

## Understanding the Issue

### Problem Description
When running PyIceberg with Python 3.13, sqlite and ray libraries emit deprecation warnings. Rather than fixing the root cause, PR #2863 filtered out these warnings to unblock Python 3.13 support. The underlying issues remain unresolved.

### Expected Behavior
PyIceberg should run cleanly under Python 3.13 with no warning filters needed for sqlite or ray.

### Current Behavior
Warning filters suppress deprecation warnings from sqlite and ray rather than fixing the root cause.

### Affected Components
- Warning filter lines added in PR #2863 (sqlite and ray related)
- Likely `pyiceberg/` source files where the deprecated APIs are called

---

## Reproduction Process
*(To be filled in Phase II)*

---

## Solution Approach
*(To be filled in Phase II)*

---

## Testing Strategy
*(To be filled in Phase III)*

---

## Implementation Notes
*(To be filled in Phase III)*

---

## Pull Request
*(To be filled in Phase IV)*

---

## Learnings & Reflections
*(To be filled in Phase IV)*

---

## Resources Used
- [Issue #2530](https://github.com/apache/iceberg-python/issues/2530)
- [PR #2863 — where warning filters were added](https://github.com/apache/iceberg-python/pull/2863/files#diff-50c86b7ed8ac2cf95bd48334961bf0530cdc77b5a56f852c5c61b89d735fd711R164-R168)
- [PyIceberg CONTRIBUTING.md](https://github.com/apache/iceberg-python/blob/main/CONTRIBUTING.md)
