# Code Review: CL 7556734 — [Clipboard][Windows] Use async ReadFileNames with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7556734
**Author:** Hewro Hewei (ihewro@chromium.org)
**Reviewer:** Rohan Raja (roraja@microsoft.com)
**Date:** 2026-02-13

---

## Review Summary

| Category    | Status | Notes                                                                                      |
|-------------|--------|--------------------------------------------------------------------------------------------|
| Correctness | ✅      | Logic is sound; follows established `ReadAsync` pattern used by ReadHTML and other methods  |
| Style       | ⚠️      | Minor style nits: missing `&` on function pointer, `CHECK` vs `DCHECK` inconsistency       |
| Security    | ✅      | No new security concerns; clipboard access properly scoped via `ScopedClipboard`            |
| Performance | ✅      | Core goal achieved — blocking Win32 clipboard I/O moved off UI thread to ThreadPool         |
| Testing     | ⚠️      | Good basic coverage; could benefit from multi-file and different format type test cases      |

---

## Detailed Findings

### Issue #1: Missing `&` operator on static method pointer in `BindOnce`
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~429 (new async `ReadFilenames` method)
**Description**: The call `base::BindOnce(ClipboardWin::ReadFilenamesInternal, buffer)` omits the address-of operator `&` before the function name. While both forms are technically valid C++ for static member functions, Chromium convention consistently uses the `&` prefix for all function/method pointers passed to `base::Bind*`. This is also enforced in the [Chromium C++ style guide](https://chromium.googlesource.com/chromium/src/+/main/styleguide/c++/c++.md) and matches other `ReadAsync` call sites (e.g., ReadHTML).
**Suggestion**: Change to:
```cpp
ReadAsync(base::BindOnce(&ClipboardWin::ReadFilenamesInternal, buffer),
          std::move(callback));
```

---

### Issue #2: `DCHECK(result)` upgraded to `CHECK(result)` in sync overload
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~683 (sync `ReadFilenames` method)
**Description**: The original code used `DCHECK(result)` (debug-only assertion) to validate the output pointer. The new code upgrades it to `CHECK(result)` (always-on fatal check). While `CHECK` is defensible here (dereferencing a null `result` would crash anyway), this upgrade is inconsistent with the patch set 11 commit message ("recover original dcheck instead of check") and changes the behavior in release builds. The `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)` in `ReadFilenamesInternal` correctly remains a `DCHECK`, but the null-pointer check was upgraded.
**Suggestion**: Consider reverting to `DCHECK(result)` for consistency with the original code and the stated intent in patch set 11, or explicitly document why the upgrade to `CHECK` is intentional. If `CHECK` is preferred, that's also fine since it's a null-pointer guard before a dereference.

---

### Issue #3: Duplicated `data_dst` comment block
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~424 and ~433
**Description**: The comment `// |data_dst| is not used. It's only passed to be consistent with other // platforms.` now appears twice in close proximity — once above the new async `ReadFilenames` and once above `ReadAvailableTypes`. While each copy applies to its respective method, having them back-to-back can look like accidental duplication.
**Suggestion**: This is acceptable as-is since each comment documents its own method, but consider whether the comment for `ReadAvailableTypes` was already there before this CL. If so, no action needed. If the new method's comment was placed directly above an existing identical comment, consider adding the method name to each for clarity, e.g., `// |data_dst| in ReadFilenames is not used...`.

---

### Issue #4: Test coverage for multiple filenames and alternate clipboard formats
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win_unittest.cc`
**Line**: ~278+
**Description**: The new tests (`ReadFilenamesAsyncReturnsWrittenData` and `ReadFilenamesAsyncEmptyClipboard`) cover the single-file happy path and the empty clipboard case. However, there is no test for:
1. **Multiple filenames** — verifying that all filenames are returned when >1 file is on the clipboard.
2. **Different clipboard format fallbacks** — the `ReadFilenamesInternal` method tries three formats in sequence (`CF_HDROP`, `FileNameW`, `FileNameA`). Only the first (`CF_HDROP` via `WriteFilenames`) is exercised.

These edge cases may be covered by existing sync `ReadFilenames` tests elsewhere, but having async-specific coverage would increase confidence.
**Suggestion**: Consider adding a test that writes multiple filenames and asserts the count and paths are correct. Format fallback testing may require lower-level Win32 clipboard setup and could be deferred.

---

### Issue #5: Sync `ReadFilenames` lost `DCHECK_EQ(buffer, ...)` and `RecordRead` calls
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: ~681-685
**Description**: The sync `ReadFilenames` overload previously had `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)` and `RecordRead(ClipboardFormatMetric::kFilenames)` at its top. These were moved into `ReadFilenamesInternal`, so they're still executed. This is correct since the sync method now delegates to `ReadFilenamesInternal`. However, this means every async call also records the metric on the ThreadPool thread — worth verifying that `RecordRead` is thread-safe.
**Suggestion**: Verify that `RecordRead(ClipboardFormatMetric::kFilenames)` is safe to call from a ThreadPool thread. If it uses atomics or is thread-safe (as is likely), no change needed.

---

## Positive Observations

- **Follows established pattern**: The async implementation follows the exact same `ReadAsync` + `*Internal` static method pattern used by `ReadHTML`, making the codebase consistent and predictable.
- **Clean refactoring**: Extracting `ReadFilenamesInternal` as a static method that returns by value is a clean separation. The sync overload becomes a thin wrapper, and the `ReadAsync` call is minimal.
- **Good use of `emplace_back`**: Consistently replacing `push_back(ui::FileInfo(...))` with `emplace_back(...)` throughout the method is a nice efficiency improvement.
- **NRVO-friendly return**: Returning `std::vector<ui::FileInfo>` by value with a named local variable enables Named Return Value Optimization, avoiding unnecessary copies.
- **Responsive to reviewer feedback**: The author promptly addressed the diff-minimization suggestion from patch set 2, resulting in a much cleaner diff.
- **Tests cover key scenarios**: The added tests verify both the happy path (write then async read) and the empty clipboard case, with proper use of `TestFuture` for async verification.
- **Proper use of `ScopedAllowBlockingForTesting`**: The test correctly allows blocking in the test environment for temporary file creation.

---

## Overall Assessment

**LGTM with minor comments**

This is a well-structured CL that achieves its stated goal of moving blocking Win32 clipboard access off the UI thread. The implementation follows the established `ReadAsync` pattern, the refactoring is clean, and the tests cover the primary use cases.

The issues identified are all minor or suggestions:
- **Issue #1** (`&` on function pointer) is a style nit that should be fixed for consistency.
- **Issue #2** (`CHECK` vs `DCHECK`) should be clarified — either is acceptable but should match the stated intent.
- **Issues #3–5** are suggestions for improvement, not blockers.

The CL has passed multiple dry runs successfully, confirming compilation and test correctness on CI.
