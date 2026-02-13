# Code Review: CL 7556734

## [Clipboard][Windows] Use async ReadFileNames with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7556734
**Bug:** [458194647](https://crbug.com/458194647)
**Author:** Hewro Hewei <ihewro@chromium.org>
**Reviewer:** Code Review Agent
**Date:** 2026-02-13 (Patch Set 10)

**Related CLs:**
- [CL 7151578](https://chromium-review.googlesource.com/c/chromium/src/+/7151578): [Clipboard][Windows] Use async ReadHTML with ThreadPool offloading (MERGED — established the async clipboard pattern)
- [CL 7565599](https://chromium-review.googlesource.com/c/chromium/src/+/7565599): [Clipboard][Windows] Simplify ReadAsync template (MERGED — simplified `ReadAsync` to `OnceCallback<Result(HWND)>`)

---

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | `DCHECK_EQ` could be `CHECK_EQ` for consistency; minor parameter-address style nit; overall logic is sound |
| Style | ⚠️ | Duplicate comment block; missing `&` on static function pointer in `base::BindOnce`; good improvement over ReadHTMLInternal pattern |
| Security | ✅ | No new security concerns; clipboard data bounded by existing `GetClipboardDataWithLimit` |
| Performance | ✅ | Core goal achieved — blocking Win32 clipboard reads moved off UI thread via `ReadAsync` + `worker_task_runner_` |
| Testing | ⚠️ | Good content validation and empty clipboard tests added; missing multi-file and error-path coverage |

---

## Detailed Findings

---

#### Issue #1: `DCHECK_EQ` Should Be `CHECK_EQ` for Buffer Validation
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~685 (in `ReadFilenamesInternal`)
**Description**: The original `ReadFilenames` used `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)`. The CL preserves this as a `DCHECK_EQ` in `ReadFilenamesInternal`. However, the parent CL (7151578) review specifically recommended upgrading `DCHECK` to `CHECK` for clipboard invariants (per Dan Clark's review guidance). The sync `ReadFilenames` wrapper already uses `CHECK(result)`. For consistency, the buffer check should also be `CHECK_EQ`.

Additionally, since `ReadFilenamesInternal` is now `static` and can be called from a ThreadPool worker thread (where the calling context may be less controlled), a hard `CHECK` provides stronger defense against misuse.
**Suggestion**:
```cpp
CHECK_EQ(buffer, ClipboardBuffer::kCopyPaste);
```

---

#### Issue #2: Missing `&` Operator on Static Function Pointer in `base::BindOnce`
**Severity**: Minor (Style)
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~428 (in async `ReadFilenames`)
**Description**: The code passes the static member function without the address-of operator:
```cpp
ReadAsync(base::BindOnce(ClipboardWin::ReadFilenamesInternal, buffer),
          std::move(callback));
```
While this compiles for static member functions (the function name decays to a function pointer), Chromium's `base::BindOnce` usage convention typically uses the explicit `&` for function pointers. This is also what `git cl format` / `clang-tidy` expects and makes it visually consistent with non-static member function binding patterns.
**Suggestion**:
```cpp
ReadAsync(base::BindOnce(&ClipboardWin::ReadFilenamesInternal, buffer),
          std::move(callback));
```

---

#### Issue #3: Duplicate Comment Block Before `ReadAvailableTypes`
**Severity**: Minor (Style)
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~424–434
**Description**: The new async `ReadFilenames` method is inserted with its own `// |data_dst| is not used...` comment, immediately followed by the identical comment for `ReadAvailableTypes`:
```cpp
// |data_dst| is not used. It's only passed to be consistent with other
// platforms.
void ClipboardWin::ReadFilenames(...) { ... }

// |data_dst| is not used. It's only passed to be consistent with other
// platforms.
void ClipboardWin::ReadAvailableTypes(...) { ... }
```
While each comment is technically correct for its respective function, the consecutive duplication looks like a copy-paste artifact. Consider varying the wording or consolidating.
**Suggestion**: Keep the comment for `ReadFilenames` but consider slightly differentiating, e.g.:
```cpp
// |data_dst| is unused on Windows; passed for cross-platform consistency.
```
Or simply accept the duplication since each method has its own doc comment — this is a very minor nit.

---

#### Issue #4: `ReadFilenamesInternal` Parameter Order Inconsistency with `ReadHTMLInternal`
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win.h`
**Line**: ~147
**Description**: The two `*Internal` static helpers have inconsistent parameter ordering:
- `ReadHTMLInternal(HWND owner_window, ClipboardBuffer buffer, ...)` — HWND first
- `ReadFilenamesInternal(ClipboardBuffer buffer, HWND owner_window)` — buffer first

The `ReadFilenamesInternal` ordering (buffer first, HWND last) is the **better design** for `base::BindOnce` with the simplified `ReadAsync<Result>(OnceCallback<Result(HWND)>, ...)` template — it allows binding `buffer` and leaving `HWND` as the remaining unbound parameter. `ReadHTMLInternal` requires a lambda wrapper to adapt its HWND-first signature.

This is not a problem to fix in this CL, but it's worth noting:
1. The `ReadFilenamesInternal` pattern is the one future `*Internal` methods should follow.
2. There's already a TODO to refactor `ReadHTMLInternal` to return a struct (eliminating out-params). When that TODO is addressed, the parameter order should also be updated to match `ReadFilenamesInternal`.
**Suggestion**: No change needed in this CL. Consider adding a comment on `ReadFilenamesInternal` indicating this is the preferred pattern, or update the existing TODO on `ReadHTMLInternal`.

---

#### Issue #5: Test Coverage — Missing Multi-File Test Case
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win_unittest.cc`
**Line**: ~288 (after `ReadFilenamesAsyncReturnsWrittenData`)
**Description**: `ReadFilenamesAsyncReturnsWrittenData` tests writing and reading back a single file. The `ReadFilenamesInternal` implementation handles three distinct clipboard formats (CF_HDROP, CF_FileNameW, CF_FileNameA) and the CF_HDROP path specifically iterates over multiple filenames. A test with multiple files via CF_HDROP would exercise the iteration logic.
**Suggestion**: Consider adding a test that writes multiple filenames and verifies all are returned:
```cpp
TEST_F(ClipboardWinTest, ReadFilenamesAsyncMultipleFiles) {
  base::ScopedAllowBlockingForTesting allow_blocking;
  base::ScopedTempDir temp_dir;
  ASSERT_TRUE(temp_dir.CreateUniqueTempDir());
  base::FilePath file1, file2;
  ASSERT_TRUE(base::CreateTemporaryFileInDir(temp_dir.GetPath(), &file1));
  ASSERT_TRUE(base::CreateTemporaryFileInDir(temp_dir.GetPath(), &file2));

  auto* clipboard = Clipboard::GetForCurrentThread();
  {
    ScopedClipboardWriter writer(ClipboardBuffer::kCopyPaste);
    writer.WriteFilenames(ui::FileInfosToURIList(
        {ui::FileInfo(file1, base::FilePath()),
         ui::FileInfo(file2, base::FilePath())}));
  }

  base::test::TestFuture<std::vector<ui::FileInfo>> filenames_future;
  clipboard->ReadFilenames(ClipboardBuffer::kCopyPaste, std::nullopt,
                           filenames_future.GetCallback());
  ASSERT_TRUE(filenames_future.Wait());
  const auto& filenames = filenames_future.Get();
  ASSERT_EQ(2u, filenames.size());
  // Verify both files are present (order may vary)
}
```

---

#### Issue #6: Sync `ReadFilenames` Now Delegates to `ReadFilenamesInternal` — Verify No Behavioral Change
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~681–683
**Description**: The sync `ReadFilenames` now delegates to `ReadFilenamesInternal`:
```cpp
void ClipboardWin::ReadFilenames(ClipboardBuffer buffer,
                                 const DataTransferEndpoint* data_dst,
                                 std::vector<ui::FileInfo>* result) const {
  CHECK(result);
  result->clear();
  *result = ReadFilenamesInternal(buffer, GetClipboardWindow());
}
```
The `CHECK(result)` upgrade from `DCHECK(result)` is a good hardening change. The `result->clear()` call is moved before `ReadFilenamesInternal`, and the assignment via `*result = ...` replaces the previous in-place modification. This is correct but subtly changes the behavior: the old code would append to a pre-cleared vector incrementally, while the new code assigns the entire result at once. Since `result->clear()` is called first either way, the net result is identical.

However, note that the `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)` check that was in the original sync `ReadFilenames` is now only inside `ReadFilenamesInternal`. The sync path goes through `ReadFilenamesInternal`, so the check is still executed — no issue, just confirming.
**Suggestion**: No change needed. The refactoring preserves behavioral parity.

---

#### Issue #7: `ReadFilenamesInternal` Uses `emplace_back` Consistently — Good Modernization
**Severity**: Suggestion (Positive)
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~710, ~724, ~738
**Description**: The original code inconsistently used `push_back(ui::FileInfo(path, base::FilePath()))` in some places and `emplace_back(base::FilePath(filename), base::FilePath())` in others. The CL consistently uses `emplace_back` throughout `ReadFilenamesInternal`, which avoids an unnecessary move/copy. This is a good modernization improvement.

---

#### Issue #8: Test Uses `base::ScopedAllowBlockingForTesting` — Verify It's Necessary
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win_unittest.cc`
**Line**: ~278 (in `ReadFilenamesAsyncReturnsWrittenData`)
**Description**: The test uses `base::ScopedAllowBlockingForTesting allow_blocking;` which is needed because `base::CreateTemporaryFileInDir` performs blocking I/O. This is correct — the blocking allowance is for the test setup (temp file creation), not for the clipboard read itself. The `ReadFilenamesAsyncEmptyClipboard` test correctly omits it since it doesn't create temp files.
**Suggestion**: No change needed. Good practice to scope the blocking allowance appropriately.

---

## Positive Observations

1. **Clean Refactoring Pattern**: `ReadFilenamesInternal` follows the improved pattern established after the `ReadAsync` simplification (CL 7565599) — returns the result directly by value, puts `HWND` as the last parameter for clean `base::BindOnce` integration, and is `static` for safe ThreadPool execution. This is the model for future `*Internal` methods.

2. **Minimal Diff Per Reviewer Feedback**: The author responded to the reviewer's (roraja@) nit about minimizing the diff by renaming the existing implementation to `ReadFilenamesInternal` and keeping it near the original location. This makes the CL easy to review and the git history clean.

3. **Good Test Coverage for New Functionality**: Unlike the initial ReadHTML CL which only had a smoke test, this CL includes both content validation (`ReadFilenamesAsyncReturnsWrittenData`) and empty-state (`ReadFilenamesAsyncEmptyClipboard`) tests from the start.

4. **Behavioral Parity Between Sync and Async Paths**: Both the sync `ReadFilenames` and the async `ReadFilenames` delegate to the same `ReadFilenamesInternal` static helper, ensuring identical behavior regardless of which API surface is used.

5. **Consistent Modernization**: The `push_back` → `emplace_back` cleanup throughout the function is a welcome micro-optimization and style improvement.

6. **Correct `ReadAsync` Integration**: The `base::BindOnce(ReadFilenamesInternal, buffer)` idiom perfectly matches the simplified `ReadAsync<Result>(OnceCallback<Result(HWND)>, ...)` template from CL 7565599. The result type `std::vector<ui::FileInfo>` is directly compatible with `ReadFilenamesCallback`, requiring no intermediate conversion.

7. **Dry Runs Pass**: Multiple dry runs (PS 2, 3, 6, 8) have passed, confirming compilation and test correctness across the CQ bots.

---

## Cross-Reference Notes

### Relationship with CL 7565599 (Simplify ReadAsync)
CL 7565599 simplified the `ReadAsync` template to accept `OnceCallback<Result(HWND)>` and `OnceCallback<void(Result)>`, replacing the previous tuple-based approach. This CL is the **first consumer** of the simplified template for a new data type, and it demonstrates the pattern works cleanly for `std::vector<ui::FileInfo>`.

### Relationship with CL 7151578 (Async ReadHTML)
CL 7151578 introduced the async clipboard pattern. Several issues identified in that review have been addressed organically in this CL:
- **Issue #2 (MayBlock)**: The `worker_task_runner_` was fixed in a prior CL; this CL inherits the fix.
- **Issue #6 (Out-params → return-by-value)**: `ReadFilenamesInternal` returns `std::vector<ui::FileInfo>` directly, not via out-params. This is the pattern the ReadHTMLInternal TODO requests.
- **Issue #9 (Test coverage)**: This CL includes content validation and empty clipboard tests, addressing the test coverage gap noted in the ReadHTML review.

### Outstanding Items from ReadHTML Review Still Applicable:
- **Issue #1 (Typo `kNonBlockingAysncClipboardApi`)**: If the feature flag typo hasn't been fixed yet, it affects this CL's code path as well.
- **Issue #3 (CHECK strictness in `ScopedClipboard::Acquire`)**: The relaxation suggestion still applies to all clipboard reads including filenames.

---

## Overall Assessment

**LGTM with minor comments**

This is a well-executed CL that extends the async clipboard pattern to `ReadFilenames`. It follows the established architecture cleanly, improves upon the original ReadHTML pattern (return-by-value, better parameter ordering), and includes meaningful test coverage. The code has passed multiple dry runs.

### Should Fix:
1. **Upgrade `DCHECK_EQ` to `CHECK_EQ`** in `ReadFilenamesInternal` for consistency with the project's direction (Issue #1).
2. **Add `&` to `base::BindOnce` function pointer** for style consistency (Issue #2).

### Nice to Have:
3. Add a multi-file test case to exercise the CF_HDROP iteration path (Issue #5).
4. Address the duplicate comment before `ReadAvailableTypes` (Issue #3).

### No Action Needed:
- The parameter ordering inconsistency with `ReadHTMLInternal` (Issue #4) is a pre-existing design debt that should be addressed when the `ReadHTMLInternal` TODO is resolved.
- The sync-path refactoring (Issue #6) correctly preserves behavioral parity.

---

*Review completed: 2026-02-13*
