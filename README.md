# Contribution 1: Implementing ZipStore's `__delitem__` via overwrite

**Contribution Number:** 1
**Student:** Mohammad Zuher Jaser Asad
**Issue:** zarr-developers/zarr-python#828
**Status:** Phase II Complete

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
- - `tests/test_store/test_zip.py` — test coverage for delete behavior
  - - `tests/test_store/test_stateful.py` — currently skips ZipStore for delete tests; skip should be removed after fix
   
    - ---

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
    2. 2. Open a Python shell or script in the repo root.
       3. 3. Run the following:
         
          4. ```python
             import zarr
             import tempfile, os

             with tempfile.NamedTemporaryFile(suffix=".zip", delete=False) as f:
                 tmp = f.name

             store = zarr.storage.ZipStore(tmp, mode="w")
             await store._open()  # or use sync context

             # Write a key
             import zarr.core.buffer as buf
             from zarr.core.buffer import cpu
             value = cpu.Buffer.from_bytes(b"hello")
             await store.set("test_key", value)

             # Attempt to delete — this raises NotImplementedError
             await store.delete("test_key")
             ```

             4. **Observed result:** `NotImplementedError` is raised by `ZipStore.delete()`.
             5. 5. Confirm by running the existing test suite: `pytest tests/test_store/test_zip.py -v` — the test in `test_stateful.py` explicitly skips ZipStore for delete with `pytest.skip(reason="ZipStore does not support delete")`.
               
                6. ### Reproduction Evidence
               
                7. Branch link: https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828
               
                8. ---
               
                9. ## Solution Approach
               
                10. ### Implementation Plan
               
                11. **Understand:**
                12. `ZipStore` cannot natively delete zip members because the Python `zipfile` module does not expose a delete API (this is a known CPython limitation: python/cpython#51067). The maintainer proposed a soft-delete workaround: overwrite the entry with an empty byte string (`b""`), then treat any entry whose content is `b""` as logically deleted across all read/list methods.
               
                13. **Match:**
                14. The existing `_set()` method already handles writing arbitrary bytes to a zip entry using `zipfile.ZipFile.writestr()`. The `_get()` method reads entries using `self._zf.open(key)`. The `list()` method iterates `self._zf.namelist()`. These are the three methods that need to be updated to implement and respect soft-deletes. A similar pattern is used internally by the `clear()` method which closes and recreates the zip file.
               
                15. **Plan:**
                16. 1. Change `supports_deletes = False` to `supports_deletes = True` in the class definition.
                    2. 2. Implement `ZipStore.delete(key)`: call `self._set(key, Buffer.from_bytes(b""))` inside the lock, overwriting the entry with an empty byte string. Do not raise `NotImplementedError`.
                       3. 3. Implement `ZipStore.delete_dir(prefix)`: iterate all keys with the prefix and call `self.delete(key)` on each.
                          4. 4. Update `ZipStore._get(key, ...)`: after reading the bytes, if the result is `b""`, return `None` (treat as deleted/missing).
                             5. 5. Update `ZipStore.exists(key)`: after confirming the key is in `namelist()`, also check that the stored bytes are not `b""`.
                                6. 6. Update `ZipStore.list()`: filter out any entries whose content is `b""` (soft-deleted keys should not appear in listings).
                                   7. 7. Remove the `pytest.skip` in `tests/test_store/test_stateful.py` for ZipStore delete tests.
                                      8. 8. Add targeted tests in `tests/test_store/test_zip.py` verifying: delete makes key inaccessible, delete on missing key is a no-op, delete_dir removes all keys under a prefix, and list/exists/get all respect the soft-delete.
                                        
                                         9. **Implement:**
                                         10. See branch: https://github.com/mohammadZuherJaserAsad/zarr-python/tree/fix-issue-828
                                        
                                         11. **Review:**
                                         12. Will follow `CONTRIBUTING.md` guidelines: run `pre-commit run --all-files`, ensure `pytest` passes with 100% coverage on new/modified lines, add a changelog entry under `changes/`, and ensure docstrings are updated.
                                        
                                         13. **Evaluate:**
                                         14. Tests to confirm the fix works:
                                         15. - `del store["key"]` no longer raises; key is gone from `exists()`, `get()`, and `list()`
                                             - - `delete_dir("prefix/")` removes all keys under that prefix
                                               - - Deleting a non-existent key is a no-op (no error)
                                                 - - Existing read/write behavior is unaffected
                                                   - - Stateful store tests pass without the ZipStore skip
                                                    
                                                     - ### Root Cause
                                                    
                                                     - `ZipStore.delete()` raises `NotImplementedError` because the Python `zipfile` module has no native delete API, and no soft-delete fallback was implemented. The class attribute `supports_deletes = False` explicitly documents this gap.
                                                    
                                                     - ### Planned Fix
                                                    
                                                     - Implement soft-delete by overwriting zip entries with `b""` and filtering those entries out in `_get()`, `exists()`, and `list()`.
                                                    
                                                     - ### Alternative Approaches Considered
                                                    
                                                     - - **Rebuild the zip file on every delete** (copy all non-deleted entries to a new zip): Correct but extremely expensive for large stores. Rejected due to performance impact.
                                                       - - **Maintain an in-memory set of deleted keys**: Would work at runtime but deleted keys would reappear after reopening the store. Rejected for lack of persistence.
                                                       - **Soft-delete via `b""`** (chosen): Persistent, low-cost, consistent with the maintainer's own suggestion in the issue. Prior PR #2838 validated this approach.
                                                      
                                                       - ---

                                                       ## Implementation Notes

                                                       *(To be completed in Phase III)*

                                                       ### Key Changes

                                                       [Summary of what you changed and why]

                                                       ### Testing Strategy

                                                       [How did you test your changes?]

                                                       ### Edge Cases Considered

                                                       [What edge cases did you think about?]

                                                       ---

                                                       ## Pull Request

                                                       PR Link: [Link to your pull request]

                                                       PR Summary: [Brief description of what your PR does]

                                                       ---

                                                       ## Maintainer Feedback Log

                                                       | Date | Feedback | Response/Action |
                                                       |------|----------|-----------------|
                                                       | [Date] | [Feedback received] | [How you responded] |

                                                       ---

                                                       ## Reflection

                                                       [After completing the contribution, reflect on what you learned, what was harder than expected, and what you would do differently]
