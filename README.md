# Contribution 1: Implementing ZipStore's `__delitem__` via overwrite

**Contribution Number:** 1
**Student:** Mohammad Zuher Jaser Asad
**Issue:** zarr-developers/zarr-python#828
**Status:** Phase IV Complete — PR open, addressing CI failures ahead of review

---

## Why I Chose This Issue

I chose this issue because it is a well-scoped, concrete Python problem that directly touches core data storage logic in zarr-python — a library widely used in scientific computing and AI/ML data pipelines. The fix involves implementing a soft-delete mechanism for ZipStore by overwriting entries with an empty byte string (`b""`) and updating related methods (`__contains__`, `__getitem__`, `keylist()`) to treat those entries as deleted. This matches my Python skills and gives me a clear path to making a real contribution.

I also chose it because two previous contributors attempted it (PRs #1184 and #2838) but both were closed as stale, meaning the issue is still open and the prior work serves as a detailed technical roadmap. The maintainer specifically noted that the v3 ZipStore implementation needs the same fix, which gives me a fresh angle to contribute that hasn't been addressed before.

---

## Understanding the Issue

### Problem Description

`ZipStore.__delitem__` currently raises `NotImplementedError` because the ZIP file format does not natively support deleting members from an archive. This means any Zarr operation that tries to delete a key from a ZipStore — such as consolidating metadata, overwriting a chunk, or running `zarr.consolidate_metadata` — will crash with a `NotImplementedError`.

### Expected Behavior

Calling `del store[key]` on a `ZipStore` should succeed. The key should no longer appear in iteration (`list()`, `list_dir()`) and should not be accessible via `store[key]` or `store.exists(key)`. The soft-delete approach (overwriting with `b""`) means the zip file entry remains on disk but is treated as deleted at the Zarr level.

### Current Behavior

Calling `del store[key]` raises `NotImplementedError`. The `delete_dir` method also raises `NotImplementedError` as soon as any key is found. The class attribute `supports_deletes = False` reflects this limitation.

### Affected Components

- `src/zarr/storage/_zip.py` — primary file: `ZipStore.delete()`, `ZipStore.delete_dir()`, `ZipStore.exists()`, `ZipStore.list()`, `ZipStore._get()`, and the class attribute `supports_deletes`
- `tests/test_store/test_zip.py` — test coverage for delete behavior
- `tests/test_store/test_stateful.py` — previously skipped ZipStore for delete tests; skip removed after fix

---

## Reproduction Process

### Environment Setup

Cloned the fork locally and set up the development environment using the project's standard Python tooling:

```bash
git clone https://github.com/mohammadZuherJaserAsad/zarr-python.git
cd zarr-python
pip install -e ".[dev]"
```

Python 3.11 was used. No dev container was present; setup was straightforward following the project README.

### Steps to Reproduce

```python
import zarr, tempfile, os
from zarr.core.buffer import cpu

with tempfile.NamedTemporaryFile(suffix=".zip", delete=False) as f:
    tmp = f.name

store = zarr.storage.ZipStore(tmp, mode="w")
await store._open()
value = cpu.Buffer.from_bytes(b"hello")
await store.set("test_key", value)
await store.delete("test_key")  # raises NotImplementedError
```

### Reproduction Evidence

**Branch:** https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828-1

---

## Solution Approach

### Root Cause

`ZipStore.delete()` raises `NotImplementedError` because the Python `zipfile` module has no native delete API, and no soft-delete fallback was implemented. The class attribute `supports_deletes = False` explicitly documents this gap.

### Implementation Plan

1. Change `supports_deletes = False` to `supports_deletes = True`
2. Implement `ZipStore.delete(key)`: write `b""` to the zip entry; no-op if key is absent
3. Implement `ZipStore.delete_dir(prefix)`: collect live keys, call `delete()` on each
4. Update `ZipStore._get(key)`: return `None` if content is `b""`
5. Update `ZipStore.exists(key)`: return `False` if content is `b""`
6. Update `ZipStore.list()`: deduplicate and skip sentinel entries
7. Update `ZipStore.list_dir()`: consume filtered `list()` output
8. Remove `pytest.skip` blocks in `test_stateful.py`
9. Add targeted tests in `test_zip.py`

### Alternative Approaches Considered

- **Rebuild zip on every delete**: Correct but O(n) in archive size. Rejected.
- **In-memory deleted-key set**: Doesn't survive store close/reopen. Rejected.
- **Soft-delete via `b""` (chosen)**: Persistent, low-cost, matches maintainer's suggestion.

---

## Implementation Notes

### Key Changes

| File | Method | Change |
|---|---|---|
| `src/zarr/storage/_zip.py` | `supports_deletes` | Changed `False` → `True` |
| `src/zarr/storage/_zip.py` | `_get()` | Returns `None` if content is `b""` |
| `src/zarr/storage/_zip.py` | `delete()` | Writes `b""` to zip entry; no-op if key absent |
| `src/zarr/storage/_zip.py` | `delete_dir()` | Collects live keys, calls `delete()` on each |
| `src/zarr/storage/_zip.py` | `exists()` | Returns `False` for soft-deleted entries |
| `src/zarr/storage/_zip.py` | `list()` | Deduplicates and skips soft-deleted keys |
| `src/zarr/storage/_zip.py` | `list_dir()` | Consumes filtered `list()` output |
| `tests/test_store/test_zip.py` | multiple | 7 new tests + updated `test_api_integration` |
| `tests/test_store/test_stateful.py` | multiple | Removed 2 `pytest.skip` blocks |

### Testing Strategy

New tests: `test_store_supports_deletes`, `test_delete_makes_key_inaccessible`, `test_delete_nonexistent_key_is_noop`, `test_delete_does_not_affect_other_keys`, `test_delete_already_deleted_key_is_noop`, `test_delete_dir_removes_all_keys_under_prefix`, `test_delete_dir_empty_prefix_is_noop`, `test_list_excludes_soft_deleted_keys`.

### Edge Cases Considered

- Key not in archive: `delete()` is a no-op
- Double-delete: idempotent
- Duplicate zip entries: `list()` uses `seen` set
- Read-only stores: `_check_writable()` called at top of write methods

---

## Pull Request

**PR Link:** https://github.com/zarr-developers/zarr-python/pull/4107

**PR Summary:**

Implements soft-delete for `ZipStore` so that `delete()` and `delete_dir()` no longer raise `NotImplementedError`. The ZIP format has no native entry-removal API, so this PR overwrites deleted entries with an empty byte sentinel (`b""`) and filters it out in all read, exists, and list paths. Changes span 3 files: `_zip.py` (core implementation), `test_zip.py` (7 new tests + updated integration test), and `test_stateful.py` (removed 2 skip blocks). 161 additions, 53 deletions.

---

## Maintainer Feedback Log

| Date | Feedback | Response/Action |
|---|---|---|
| 2026-06-27 | PR opened — awaiting review | CI checks running |
| 2026-07-05 | CI now failing: Lint flagged an unused import (F401 in test_stateful.py) and a ruff-format diff in _zip.py. Slow Hypothesis CI found a real bug: repeated writestr() calls to the same zip entry name raise a UserWarning (Duplicate name), and a stateful property test caught a key-count mismatch after delete. Contributor d-v-b tagged maintainer mkitti for review; no review comments yet. | Plan to run prek run --all-files locally to fix the lint issues, then investigate why list()'s dedup doesn't prevent the duplicate-name warning on repeated writes/deletes before pushing a follow-up commit. |

---

## Reflection

This project gave me real hands-on experience contributing to a widely-used open source library. The hardest part was understanding why the ZIP format can't natively delete entries and designing a workaround that's persistent, thread-safe, and doesn't break existing behavior. I learned how to trace through an async Python codebase, write targeted pytest tests, and navigate GitHub's PR workflow from fork to submission. If I did it again I'd set up the local dev environment earlier so I could run tests before committing, rather than relying solely on the CI checks in the PR.
