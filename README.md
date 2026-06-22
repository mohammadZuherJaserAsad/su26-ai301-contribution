# Contribution 1: Implementing ZipStore's `__delitem__` via overwrite

**Contribution Number:** 1
**Student:** Mohammad Zuher Jaser Asad
**Issue:** zarr-developers/zarr-python#828
**Status:** Phase III Complete

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

Python 3.11 was used. No dev container was present; setup was straightforward following the project README. The only minor friction was ensuring `numcodecs` and `pytest` were installed, which `pip install -e ".[dev]"` handled automatically.

### Steps to Reproduce

1. Clone the fork and install in editable mode (see Environment Setup above).
2. Open a Python shell or script in the repo root.

```python
import zarr
import tempfile, os

with tempfile.NamedTemporaryFile(suffix=".zip", delete=False) as f:
    tmp = f.name

store = zarr.storage.ZipStore(tmp, mode="w")
await store._open()

import zarr.core.buffer as buf
from zarr.core.buffer import cpu
value = cpu.Buffer.from_bytes(b"hello")
await store.set("test_key", value)

# Attempt to delete — this raises NotImplementedError
await store.delete("test_key")
```

Observed result: `NotImplementedError` is raised by `ZipStore.delete()`.

Confirm by running the existing test suite: `pytest tests/test_store/test_zip.py -v` — the test in `test_stateful.py` explicitly skips ZipStore for delete with `pytest.skip(reason="ZipStore does not support delete")`.

### Reproduction Evidence

**Branch link:** https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828-1

---

## Solution Approach

### Implementation Plan

**Understand:**

ZipStore cannot natively delete zip members because the Python `zipfile` module does not expose a delete API (this is a known CPython limitation: python/cpython#51067). The maintainer proposed a soft-delete workaround: overwrite the entry with an empty byte string (`b""`), then treat any entry whose content is `b""` as logically deleted across all read/list methods.

**Match:**

The existing `_set()` method already handles writing arbitrary bytes to a zip entry using `zipfile.ZipFile.writestr()`. The `_get()` method reads entries using `self._zf.open(key)`. The `list()` method iterates `self._zf.namelist()`. These are the three methods that need to be updated to implement and respect soft-deletes. A similar pattern is used internally by the `clear()` method which closes and recreates the zip file.

**Plan:**

1. Change `supports_deletes = False` to `supports_deletes = True` in the class definition.
2. Implement `ZipStore.delete(key)`: write `b""` directly to the zip entry inside the lock. Do not raise `NotImplementedError`. No-op if key is absent.
3. Implement `ZipStore.delete_dir(prefix)`: collect all live keys under the prefix, then call `self.delete(key)` on each.
4. Update `ZipStore._get(key, ...)`: after reading the bytes, if the result is `b""`, return `None` (treat as deleted/missing).
5. Update `ZipStore.exists(key)`: after confirming the key is in namelist(), also check that the stored bytes are not `b""`.
6. Update `ZipStore.list()`: use a `seen` set to deduplicate, and filter out any entries whose content is `b""`.
7. Update `ZipStore.list_dir()`: consume the filtered `list()` output for consistency.
8. Remove the `pytest.skip` in `tests/test_store/test_stateful.py` for ZipStore delete tests.
9. Add targeted tests in `tests/test_store/test_zip.py`.

**Implement:**

See branch: https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828-1

**Review:**

Will follow CONTRIBUTING.md guidelines: run `pre-commit run --all-files`, ensure `pytest` passes on new/modified lines, and ensure docstrings are updated.

**Evaluate:**

- `del store["key"]` no longer raises; key is gone from `exists()`, `get()`, and `list()`
- `delete_dir("prefix/")` removes all keys under that prefix
- Deleting a non-existent key is a no-op (no error)
- Existing read/write behavior is unaffected
- Stateful store tests pass without the ZipStore skip

### Root Cause

`ZipStore.delete()` raises `NotImplementedError` because the Python `zipfile` module has no native delete API, and no soft-delete fallback was implemented. The class attribute `supports_deletes = False` explicitly documents this gap.

### Alternative Approaches Considered

- **Rebuild the zip file on every delete**: Correct but extremely expensive for large stores. Rejected due to performance impact.
- **Maintain an in-memory set of deleted keys**: Would work at runtime but deleted keys would reappear after reopening the store. Rejected for lack of persistence.
- **Soft-delete via `b""` (chosen)**: Persistent, low-cost, consistent with the maintainer's own suggestion in the issue. Prior PR #2838 validated this approach.

---

## Implementation Notes

### Week 3 Progress

**Branch:** https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828-1

### Key Changes

| File | Method | Change |
|---|---|---|
| `src/zarr/storage/_zip.py` | `supports_deletes` | Changed `False` → `True` |
| `src/zarr/storage/_zip.py` | `_get()` | Reads all bytes first; returns `None` if content is `b""` |
| `src/zarr/storage/_zip.py` | `delete()` | Implemented: writes `b""` to zip entry; no-op if key absent |
| `src/zarr/storage/_zip.py` | `delete_dir()` | Implemented: collects live keys, calls `delete()` on each |
| `src/zarr/storage/_zip.py` | `exists()` | Returns `False` for soft-deleted (empty) entries |
| `src/zarr/storage/_zip.py` | `list()` | Deduplicates entries and skips soft-deleted keys |
| `src/zarr/storage/_zip.py` | `list_dir()` | Consumes filtered `list()` output |
| `tests/test_store/test_zip.py` | multiple | 7 new test methods + updated `test_api_integration` |
| `tests/test_store/test_stateful.py` | `test_zarr_hierarchy`, `test_zarr_store` | Removed 2 `pytest.skip` blocks |

**Why soft-delete?** The Python `zipfile` module provides no API for removing entries from an archive. The maintainer explicitly recommended this approach in the issue thread. Overwriting with `b""` is persistent (survives store close/reopen), low-cost (one write per delete), and consistent with prior PR attempts.

### Challenges Faced

- **CodeMirror 6 editor access**: GitHub's web editor uses CM6, which differs from CM5. Used `execCommand('insertText')` to set file content programmatically.
- **Duplicate zip entries**: Python's `zipfile` allows multiple entries with the same name. After a soft-delete, the zip contains two entries for the key. Solved with a `seen` set in `list()` to deduplicate, relying on Python's `NameToInfo` dict (which always tracks the last-written entry).
- **Lock re-entrancy**: `delete_dir()` calls `list_prefix()` (which holds the lock) and then `delete()` (also acquires the lock). Works correctly because `threading.RLock` is re-entrant for the same thread.

### Testing Strategy

**New tests in `tests/test_store/test_zip.py`:**
- `test_store_supports_deletes` — asserts `store.supports_deletes is True`
- `test_delete_makes_key_inaccessible` — verifies `exists`, `get`, and `list` all treat deleted key as gone
- `test_delete_nonexistent_key_is_noop` — no exception for deleting a missing key
- `test_delete_does_not_affect_other_keys` — sibling keys are unaffected
- `test_delete_already_deleted_key_is_noop` — double-delete is safe
- `test_delete_dir_removes_all_keys_under_prefix` — prefix removal works
- `test_delete_dir_empty_prefix_is_noop` — no-op on unmatched prefix
- `test_list_excludes_soft_deleted_keys` — `list()` never yields soft-deleted keys
- `test_api_integration` (updated) — removed two `NotImplementedError` assertions

**`tests/test_store/test_stateful.py`:** Removed both `pytest.skip` blocks that excluded ZipStore from delete-based stateful tests.

### Edge Cases Considered

- **Key not in archive**: `delete()` checks `namelist()` before writing; skips if key absent.
- **Double-delete**: Safe — second call finds sentinel already written, overwrites again (idempotent).
- **Duplicate zip entries**: `list()` uses `seen` set so each name is yielded at most once.
- **`list_dir` consistency**: Refactored to consume filtered `list()`, inheriting soft-delete filtering automatically.
- **Read-only stores**: `delete()` and `delete_dir()` call `_check_writable()` at the top.

---

## Pull Request

PR Link: *(to be opened in Phase IV)*

PR Summary: *(to be written in Phase IV)*

---

## Maintainer Feedback Log

| Date | Feedback | Response/Action |
|---|---|---|
| *(Phase IV)* | *(to be filled)* | *(to be filled)* |

---

## Reflection

*(To be completed after Phase IV)*
