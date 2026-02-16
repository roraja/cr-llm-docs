# Code Review: CL 7574178

## [Clipboard][Windows] Use async ReadAvailableTypes with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7574178
**Author:** Hewro Hewei (ihewro@chromium.org)
**Files Changed:** 3 files, +82/-9 lines

---

### Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ✅ | Logic is sound; correctly follows the established `ReadAsync` pattern |
| Style | ⚠️ | Minor: missing blank line between function definitions |
| Security | ✅ | No new security concerns introduced |
| Performance | ✅ | This is a performance improvement — moves blocking clipboard reads off the UI thread |
| Testing | ✅ | Adequate tests for the new async path, both with data and empty clipboard |

---

### Detailed Findings

#### Issue #1: Missing blank line between function definitions
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 431-432
**Description**: There is no blank line between the end of `ReadAsciiText` (async overload) and the start of `ReadAvailableTypes` (async overload). All other function definitions in this file are separated by a blank line, and the Chromium/Google style guide expects blank lines between function definitions.
**Suggestion**: Add a blank line after line 431 (`}`) before the `ReadAvailableTypes` definition.

```cpp
  ReadAsync(base::BindOnce(&ClipboardWin::ReadAsciiTextInternal, buffer),
            std::move(callback));
}
                                    // <-- add blank line here
void ClipboardWin::ReadAvailableTypes(
```

#### Issue #2: Missing comment block before async `ReadAvailableTypes`
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 432
**Description**: The other async read methods (`ReadText`, `ReadAsciiText`, `ReadHTML`, `ReadFilenames`) each have the standard comment `// |data_dst| is not used. It's only passed to be consistent with other // platforms.` before them. The new `ReadAvailableTypes` async overload is missing this comment. While not strictly required (since the comment is somewhat obvious), it breaks the local consistency.
**Suggestion**: Add the same comment block before the new `ReadAvailableTypes` async overload for consistency with the surrounding code, or intentionally omit all of them as a separate cleanup CL.

#### Issue #3: `DCHECK` vs `CHECK` consistency in `IsFormatAvailableInternal`
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 341
**Description**: `IsFormatAvailableInternal` retains `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)` (line 341), while `ReadAvailableTypesInternal` (which calls it) is a static method that will run on a worker thread. The `DCHECK` is fine here since it's just validating the enum value, but note that the sync `ReadAvailableTypes` changed from `DCHECK(types)` to `CHECK(types)` at line 479. The inconsistency between `DCHECK_EQ` and `CHECK` for similar precondition checks is a minor style question — the `DCHECK_EQ` here is acceptable since buffer validation is a programming error, not a runtime safety issue.
**Suggestion**: No change needed, but consider whether the `DCHECK_EQ` in `IsFormatAvailableInternal` should be upgraded to `CHECK_EQ` for consistency with the stricter checking direction of the codebase.

#### Issue #4: `GetClipboardDataWithLimit` called without clipboard opened guard
**Severity**: Minor (pre-existing)
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 503-505
**Description**: In `ReadAvailableTypesInternal`, `GetClipboardDataWithLimit` is called after `clipboard.Acquire(owner_window)` succeeds (line 499), which is correct — `GetClipboardData` requires the clipboard to be open. This is properly guarded. No issue.
**Suggestion**: N/A — this is correct as-is.

#### Issue #5: Thread-safety of `::IsClipboardFormatAvailable` calls
**Severity**: Minor (informational, pre-existing pattern)
**File**: `ui/base/clipboard/clipboard_win.cc`
**Lines**: 339-355, 386-411
**Description**: Both `IsFormatAvailableInternal` and `GetStandardFormatsInternal` are now static and called from `ReadAvailableTypesInternal` which runs on a worker thread (when `kNonBlockingOsClipboardReads` is enabled). These call `::IsClipboardFormatAvailable()`, which is a Win32 API that does **not** require the clipboard to be open and is documented as thread-safe. This is the same pattern used by other existing `Read*Internal` methods for the initial format availability checks, so this is consistent and correct.
**Suggestion**: No change needed.

---

### Positive Observations

- **Follows established pattern well**: The CL correctly follows the `ReadAsync` + static `*Internal` method pattern already established by `ReadTextInternal`, `ReadAsciiTextInternal`, `ReadHTMLInternal`, and `ReadFilenamesInternal`. This makes the code easy to understand and maintain.

- **Good refactoring to static helpers**: Extracting `IsFormatAvailableInternal` and `GetStandardFormatsInternal` as static methods enables their reuse from the worker thread without needing `this`, which is the correct approach since `ReadAsync` dispatches to a thread pool.

- **Correct `owner_window` handling**: The `ReadAvailableTypesInternal` properly receives `owner_window` as a parameter (which will be `nullptr` on the worker thread and `GetClipboardWindow()` on the UI thread), matching the `ReadAsync` contract.

- **DCHECK to CHECK upgrade**: The change from `DCHECK(types)` to `CHECK(types)` in the sync `ReadAvailableTypes` overload (line 479) is appropriate — `CHECK` is preferred for null pointer dereference prevention in Chromium.

- **Adequate test coverage**: Two new test cases cover the happy path (`ReadAvailableTypesAsyncReturnsWrittenData`) and the empty clipboard edge case (`ReadAvailableTypesAsyncEmptyClipboard`). The existing `NoDataChangedNotificationOnRead` test was also correctly extended to verify the async overload doesn't trigger change notifications.

- **Clean return-value-based design**: Converting from out-parameter (`std::vector<std::u16string>*`) to return-value-based (`std::vector<std::u16string>`) in the `Internal` method is cleaner and works well with `PostTaskAndReplyWithResult`.

---

### Overall Assessment

**LGTM with minor comments**

This is a well-structured CL that correctly extends the async clipboard reading pattern to `ReadAvailableTypes`. The implementation is correct, follows the established patterns in the codebase, and has appropriate test coverage. The only issues are cosmetic:

1. **[Minor]** Add a missing blank line between function definitions (line 431-432 in `clipboard_win.cc`).
2. **[Suggestion]** Add the standard `// |data_dst| is not used.` comment for consistency.

No correctness, security, or performance concerns.
