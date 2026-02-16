# Code Review: CL 7558493 — [Clipboard][Windows] Use async ReadText/ReadAsciiText with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7558493
**Author:** Hewro Hewei (ihewro@chromium.org)
**Files Changed:** 3 files, +126/-16 lines

## Review Summary

| Category    | Status | Notes                                                                 |
|-------------|--------|-----------------------------------------------------------------------|
| Correctness | ⚠️     | `RecordRead` called on worker thread — likely safe but worth verifying; minor inconsistency in how `GlobalLock` failure is handled |
| Style       | ✅     | Follows established patterns in the file; consistent with `ReadFilenamesInternal`/`ReadHTMLInternal` |
| Security    | ✅     | No new attack surface; clipboard data size limits preserved via `GetClipboardDataWithLimit` |
| Performance | ✅     | This is the purpose of the CL — moves blocking Win32 clipboard access off the UI thread |
| Testing     | ⚠️     | Good coverage of happy path and empty clipboard; missing some edge cases |

## Detailed Findings

#### Issue #1: `RecordRead` (UMA histogram) called on worker thread
**Severity**: Minor
**File**: clipboard_win.cc
**Lines**: 500, 536
**Description**: `ReadTextInternal` and `ReadAsciiTextInternal` are declared `static` and will be invoked on the `worker_task_runner_` thread pool when `kNonBlockingOsClipboardReads` is enabled. Both call `RecordRead(ClipboardFormatMetric::kText)` which invokes `base::UmaHistogramEnumeration`. While `UmaHistogramEnumeration` is documented as thread-safe, this is a behavioral change from before when the metric was always recorded on the UI thread. The existing `ReadFilenamesInternal` (line 733) and `ReadHTMLInternal` (line 576) already follow this same pattern, so this is consistent — but worth noting for awareness.
**Suggestion**: No change needed if this matches intentional project convention; confirm that metrics recorded on the worker thread behave identically (e.g., no sequence-checking issues).

#### Issue #2: No null-check on `GlobalLock` return value
**Severity**: Minor
**File**: clipboard_win.cc
**Lines**: 514, 549
**Description**: `::GlobalLock(data)` can return `nullptr` if the lock fails (e.g., if the memory has been freed by another process between `GetClipboardData` and `GlobalLock`). In `ReadTextInternal` (line 514) and `ReadAsciiTextInternal` (line 549), the return value of `GlobalLock` is passed directly to `result.assign()` without a null-check. If `GlobalLock` returns null, this would pass a null pointer to `std::u16string::assign(const char16_t*, size_t)`, which is undefined behavior.

Note: this is a pre-existing issue — the old code had the same problem. However, since this code is being refactored, it would be a good opportunity to add a guard.
**Suggestion**: Add a null-check:
```cpp
void* locked = ::GlobalLock(data);
if (!locked)
  return result;
result.assign(static_cast<const char16_t*>(locked),
              ::GlobalSize(data) / sizeof(char16_t));
::GlobalUnlock(data);
```

#### Issue #3: Missing test for async ReadText with non-ASCII / Unicode content
**Severity**: Minor
**File**: clipboard_win_unittest.cc
**Description**: The tests only verify ASCII content (`"text_test"`). Since `ReadTextInternal` reads `CF_UNICODETEXT` and constructs a `std::u16string`, it would be valuable to test with multi-byte / surrogate pair content (e.g., emoji, CJK characters) to ensure the `GlobalSize / sizeof(char16_t)` calculation and `TrimAfterNull` work correctly in the async path.
**Suggestion**: Add a test case with Unicode content:
```cpp
TEST_F(ClipboardWinTest, ReadTextAsyncReturnsUnicodeData) {
  auto* clipboard = Clipboard::GetForCurrentThread();
  {
    ScopedClipboardWriter writer(ClipboardBuffer::kCopyPaste);
    writer.WriteText(u"日本語テスト 🎉");
  }

  base::test::TestFuture<std::u16string> text_future;
  clipboard->ReadText(ClipboardBuffer::kCopyPaste, std::nullopt,
                      text_future.GetCallback());
  ASSERT_TRUE(text_future.Wait());
  EXPECT_EQ(text_future.Get(), u"日本語テスト 🎉");
}
```

#### Issue #4: Sync wrappers now do unnecessary work (minor redundancy)
**Severity**: Suggestion
**File**: clipboard_win.cc
**Lines**: 487-493, 523-529
**Description**: The sync `ReadText(...)` and `ReadAsciiText(...)` methods now call `result->clear()` and then immediately assign `*result = ReadTextInternal(...)`. The `clear()` is redundant since `ReadTextInternal` returns a new string that overwrites `*result` entirely via assignment. This is a very minor style nit.
**Suggestion**: The `clear()` could be removed, but this is cosmetic and not blocking.

#### Issue #5: Missing test for the feature-flag-off (synchronous) path of the async API
**Severity**: Minor
**File**: clipboard_win_unittest.cc
**Description**: The `ReadAsync` template has two code paths: one when `kNonBlockingOsClipboardReads` is enabled (posts to worker thread) and one when disabled (runs synchronously). The new tests appear to run in the default configuration. It would be good to explicitly test both paths using `base::test::ScopedFeatureList` to ensure the async override works correctly regardless of feature flag state.
**Suggestion**: Add a parameterized or duplicated test that explicitly enables/disables `kNonBlockingOsClipboardReads`.

#### Issue #6: Inconsistent bracing style in early returns
**Severity**: Suggestion
**File**: clipboard_win.cc
**Lines**: 506-508, 511-512
**Description**: The code inconsistently uses braces for single-statement early returns. Lines 506-508 use braces (`if (!clipboard.Acquire(owner_window)) { return result; }`), while lines 511-512 do not (`if (!data) return result;`). The Chromium style guide states that braces are optional for single-statement bodies but should be consistent within a function.
**Suggestion**: Use braces consistently throughout each function, matching whichever convention is more prevalent in the file. The existing code in the file (e.g., `ReadHTMLInternal`) also has this inconsistency, so this is not blocking.

## Positive Observations

- **Clean pattern reuse**: The CL follows the exact same `ReadAsync` + `*Internal` static method pattern established by `ReadHTMLInternal`, `ReadFilenamesInternal`, and other async methods in the same file. This makes the code easy to understand.
- **Proper parameter binding**: `base::BindOnce(&ClipboardWin::ReadTextInternal, buffer)` correctly partially binds the `buffer` parameter, leaving `HWND` to be provided by `ReadAsync` — consistent with `ReadFilenamesInternal`.
- **Good test coverage**: Four new test cases cover both happy-path and empty-clipboard scenarios for both `ReadText` and `ReadAsciiText` async paths. The `NoDataChangedNotificationOnRead` test is also updated to include the new async overrides.
- **Minimal and focused change**: The refactoring cleanly separates the I/O logic into static `*Internal` methods while keeping the existing sync API working through delegation. The diff is small and well-scoped.
- **Correct thread safety model**: The `*Internal` methods are `static` with no `this` pointer dependency, taking `HWND` as a parameter — correctly allowing them to run on the worker thread without accessing UI-thread-bound state.

## Overall Assessment

**LGTM with minor comments**

The CL is well-structured and follows established patterns in the codebase. The core approach of extracting static `*Internal` methods and wiring them through `ReadAsync` is correct and consistent with existing async clipboard methods (HTML, Filenames). The main items to consider are:

1. **(Minor)** Consider adding a null-check on `GlobalLock` return value — pre-existing issue but good to fix while refactoring.
2. **(Minor)** Consider adding a Unicode content test case.
3. **(Minor)** Consider testing with the feature flag both on and off.

None of these are blocking; the CL can land as-is without functional issues.
