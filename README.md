# Contribution 1: Implementing ZipStore's `__delitem__` via overwrite

**Contribution Number:** 1
**Student:** Mohammad Zuher Jaser Asad
**Issue:** zarr-developers/zarr-python#828
**Status:** Phase IV — PR open, core CI (Slow Hypothesis) now passing after redesign; awaiting maintainer's full review

---

## Why I Chose This Issue

I chose this issue because it is a well-scoped, concrete Python problem that directly touches core data storage logic in zarr-python — a library widely used in scientific computing and AI/ML data pipelines. The fix involves implementing a soft-delete mechanism for ZipStore and updating related methods (`exists()`, `_get()`, `list()`) to treat deleted entries as absent. This matches my Python skills and gives me a clear path to making a real contribution.

I also chose it because two previous contributors attempted it (PRs #1184 and #2838) but both were closed as stale, meaning the issue is still open and the prior work serves as a detailed technical roadmap. The maintainer specifically noted that the v3 ZipStore implementation needs the same fix, which gives me a fresh angle to contribute that hasn't been addressed before.

---

## Understanding the Issue

### Problem Description

`ZipStore.__delitem__` currently raises `NotImplementedError` because the ZIP file format does not natively support deleting members from an archive. This means any Zarr operation that tries to delete a key from a ZipStore — such as consolidating metadata, overwriting a chunk, or running `zarr.consolidate_metadata` — will crash with a `NotImplementedError`.

### Expected Behavior

Calling `del store[key]` on a `ZipStore` should succeed. The key should no longer appear in iteration (`list()`, `list_dir()`) and should not be accessible via `store[key]` or `store.exists(key)`. The soft-delete approach means the zip file entry remains on disk but is treated as deleted at the Zarr level.

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
2. Implement `ZipStore.delete(key)`: append an empty entry flagged as deleted; no-op if key is absent
3. Implement `ZipStore.delete_dir(prefix)`: collect live keys, call `delete()` on each
4. Update `ZipStore._get(key)`, `exists(key)`, `list()`, `list_dir()` to treat flagged entries as absent
5. Remove `pytest.skip` blocks in `test_stateful.py`
6. Add targeted tests in `test_zip.py`

### Alternative Approaches Considered

- **Rebuild zip on every delete**: Correct but O(n) in archive size. Rejected.
- **In-memory deleted-key set**: Doesn't survive store close/reopen (a real regression for actual ZipStore usage, where the archive is commonly reopened in a later process). Rejected.
- **Soft-delete via a fixed sentinel *value* (`b""`, later a long "unique" byte string)**: This was my original approach and matched the maintainer's own suggestion in the issue. It is fundamentally unsafe: since Zarr payload bytes are arbitrary, a legitimate value can always equal whatever sentinel constant is chosen. Slow Hypothesis CI proved this twice — first against `b""`, then again against a 45-byte "unique" marker string, by generating a falsifying example that set a key's value to the literal sentinel bytes. Rejected after two failed CI runs.
- **Soft-delete via `ZipInfo.comment` marker (chosen)**: Flags a deleted entry using the ZIP central-directory *comment* field instead of the entry's *data*. This field is metadata the store never exposes through `set()`/`get()`, so it structurally cannot collide with any byte string a caller stores as data — verified empirically against the exact byte value a legitimate write could produce. It is also persisted to disk (survives store close/reopen), unlike the in-memory approach, and is cheaper to check than reading and decompressing full entry content on every `list()`/`exists()` call.

---

## Implementation Notes

### Key Changes

| File | Method | Change |
|---|---|---|
| `src/zarr/storage/_zip.py` | `supports_deletes` | Changed `False` → `True` |
| `src/zarr/storage/_zip.py` | `_is_soft_deleted()` | New helper: checks `ZipInfo.comment` against the soft-delete marker |
| `src/zarr/storage/_zip.py` | `_get()` | Returns `None` if `_is_soft_deleted()` |
| `src/zarr/storage/_zip.py` | `delete()` | Appends an empty entry with `ZipInfo.comment` set to the soft-delete marker; no-op if key absent |
| `src/zarr/storage/_zip.py` | `delete_dir()` | Collects live keys, calls `delete()` on each |
| `src/zarr/storage/_zip.py` | `exists()` | Returns `False` for soft-deleted entries |
| `src/zarr/storage/_zip.py` | `list()` | Deduplicates and skips soft-deleted keys via `_is_soft_deleted()` |
| `src/zarr/storage/_zip.py` | `list_dir()` | Consumes filtered `list()` output |
| `tests/test_store/test_zip.py` | multiple | 7 new tests + updated `test_api_integration` |
| `tests/test_store/test_stateful.py` | multiple | Removed 2 `pytest.skip` blocks; added `filterwarnings` marks for the expected duplicate-name warning |

### Testing Strategy

New tests: `test_store_supports_deletes`, `test_delete_makes_key_inaccessible`, `test_delete_nonexistent_key_is_noop`, `test_delete_does_not_affect_other_keys`, `test_delete_already_deleted_key_is_noop`, `test_delete_dir_removes_all_keys_under_prefix`, `test_delete_dir_empty_prefix_is_noop`, `test_list_excludes_soft_deleted_keys`.

Local verification for the comment-field redesign: wrote a standalone `zipfile` script that (1) soft-deletes an entry, (2) writes a second entry whose *value* is byte-for-byte identical to the soft-delete marker, (3) closes and reopens the archive fresh (simulating a new process), and confirmed `getinfo().comment` correctly distinguishes the two — the deleted entry reads as deleted, the entry whose data happens to equal the marker string still reads as live.

### Edge Cases Considered

- Key not in archive: `delete()` is a no-op
- Double-delete: idempotent
- Duplicate zip entries: `list()` uses a `seen` set
- Read-only stores: `_check_writable()` called at top of write methods
- A legitimate value that is byte-for-byte identical to the deletion marker: no longer possible to misclassify, since the marker lives in entry metadata (`ZipInfo.comment`), never in entry data

---

## Pull Request

**PR Link:** https://github.com/zarr-developers/zarr-python/pull/4107

**PR Summary:**

Implements soft-delete for `ZipStore` so that `delete()` and `delete_dir()` no longer raise `NotImplementedError`. The ZIP format has no native entry-removal API, so deleted entries are flagged via the `ZipInfo.comment` field on an appended empty entry, and that flag is checked in all read, exists, and list paths. Changes span 3 files: `_zip.py` (core implementation), `test_zip.py` (7 new tests + updated integration test), and `test_stateful.py` (removed 2 skip blocks, added `filterwarnings` marks).

---

## Maintainer Feedback Log

| Date | Feedback | Response/Action |
|---|---|---|
| 2026-06-27 | PR opened — awaiting review | CI checks running |
| 2026-07-05 | CI now failing: Lint flagged an unused import (F401 in test_stateful.py) and a ruff-format diff in _zip.py. Slow Hypothesis CI found a real bug: repeated writestr() calls to the same zip entry name raise a UserWarning (Duplicate name), and a stateful property test caught a key-count mismatch after delete. Contributor d-v-b tagged maintainer mkitti for review; no review comments yet. | Fixed lint issues, changed the soft-delete sentinel from `b""` to a long "unique" byte string, and suppressed the expected duplicate-name warning inside `delete()`. Pushed as commits 4db600c / 2b784fe. |
| 2026-07-09 | Slow Hypothesis CI failed again on the pushed fix — this time on `test_zarr_store[zip]`, with Hypothesis reporting a falsifying example that set a key's value to the *exact* sentinel bytes I had just introduced, reproducing the same class of bug against the new constant. Still no maintainer review comments; PR remains unreviewed. | Diagnosed this as a structural flaw, not a tuning problem: any fixed sentinel *value* can always collide with a legitimate value, no matter how "unique" it looks, because Zarr payload bytes are arbitrary. Redesigned `delete()` to flag entries via the `ZipInfo.comment` metadata field instead of entry content — metadata the store never exposes through `set()`/`get()`, so it cannot collide with caller data by construction. Verified the fix locally with a standalone `zipfile` reproduction script before pushing as commit 81af773; CI re-running now. |
| 2026-07-20 | Slow Hypothesis CI now passes on the ZipInfo.comment redesign -- the collision bug from the last two attempts is resolved. Maintainer mkitti left a first comment ("Soft delete is the best we can here I think. I have not looked closely at the implementation yet.") but has not yet done a full review. Three CI checks still fail on the minimal-deps matrix (py3.12-min_deps, py3.14 deps=minimal) -- the same test_array.py issue noted previously, unrelated to files this PR touches. Merge remains blocked pending a required review; branch is out-of-date with main but cleanly mergeable. | No code changes needed for the now-passing Hypothesis check. Continuing to treat the minimal-deps test_array.py failure as out of scope for this PR. Waiting on mkitti (or another maintainer) for a substantive review; will update branch from main if requested. |

---

## Reflection

This project gave me real hands-on experience contributing to a widely-used open source library. The hardest part was discovering that my first two fixes for the same bug class only made the collision less *likely*, not impossible — Hypothesis's property-based testing kept finding a falsifying example no matter how I tweaked the sentinel value, which forced me to recognize this was a design problem, not a tuning problem, and pushed me toward storing the deletion flag as metadata instead of data. I learned how to trace through an async Python codebase, read Hypothesis failure output (including "Falsifying example" replay traces), write targeted pytest tests, and navigate GitHub's PR workflow from fork to submission. If I did it again I'd write a quick standalone reproduction script for edge cases like this *before* choosing an implementation strategy, rather than discovering the flaw through CI after the fact.
