# Code Review: CL 7578053 — [clipboard][Windows] Make ReadPng non-blocking and refactor internals

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7578053
**Author:** Hewro Hewei <ihewro@chromium.org>
**Files changed:** 3 files, +78/−15 lines

---

## Review Summary

| Category    | Status | Notes                                                                 |
|-------------|--------|-----------------------------------------------------------------------|
| Correctness | ⚠️     | Async path does synchronous PNG encoding; tests don't exercise async  |
| Style       | ✅     | Consistent with existing patterns, good naming                        |
| Security    | ✅     | No new security concerns                                              |
| Performance | ⚠️     | Bitmap encoding blocks worker thread in async path                    |
| Testing     | ⚠️     | Tests don't enable `kNonBlockingOsClipboardReads`; async path untested |

---

## Detailed Findings

#### Issue #1: Tests don't exercise the async code path
**Severity**: Major
**File**: clipboard_win_unittest.cc
**Line**: 404–428
**Description**: The `ClipboardWinTest` fixture does not enable `features::kNonBlockingOsClipboardReads`. Both new tests (`ReadPngAsyncReturnsWrittenData` and `ReadPngAsyncEmptyClipboard`) therefore only exercise the **synchronous fallback** path in `ReadPng` (lines 733–748 of clipboard_win.cc). The actual new async code path—`ReadAsync` dispatching `ReadPngInternal` to the worker thread—is completely untested.

While this is consistent with the existing pattern for other `ReadXxxAsync` tests (ReadText, ReadAsciiText, etc.), the core purpose of this CL is to add the async path, so it should be tested.

**Suggestion**: Add a parameterized test fixture or a separate test class with `base::test::ScopedFeatureList` that enables `kNonBlockingOsClipboardReads`, similar to:
```cpp
class ClipboardWinAsyncTest : public ClipboardWinTest {
 public:
  ClipboardWinAsyncTest() {
    feature_list_.InitAndEnableFeature(features::kNonBlockingOsClipboardReads);
  }
 private:
  base::test::ScopedFeatureList feature_list_;
};
```
Then duplicate (or parameterize) the ReadPng tests for both sync and async modes.

---

#### Issue #2: Behavioral difference in bitmap fallback between sync and async paths
**Severity**: Minor
**File**: clipboard_win.cc
**Lines**: 728–748 (sync path) vs 1095–1111 (ReadPngInternal, async path)
**Description**: When there is no PNG data on the clipboard and the code falls back to reading a bitmap:

- **Sync path** (lines 743–748): `EncodeBitmapToPng` is posted to the **thread pool** via `PostTaskAndReplyWithResult`, allowing the UI thread to return immediately.
- **Async path** (`ReadPngInternal`, lines 1107–1110): `EncodeBitmapToPng` runs **synchronously** on the worker thread.

This means in the async path, the serialized `worker_task_runner_` is blocked for the duration of PNG encoding, which could delay other queued clipboard reads. This is likely acceptable since the worker thread is dedicated to clipboard I/O, and the encoding is done off the UI thread, but it's a subtle behavioral difference worth documenting.

**Suggestion**: Add a brief comment in `ReadPngInternal` explaining that synchronous encoding is acceptable here because the function runs on the worker thread, unlike the sync fallback path which posts encoding separately.

---

#### Issue #3: Test names suggest async but don't test async
**Severity**: Suggestion
**File**: clipboard_win_unittest.cc
**Lines**: 404, 419
**Description**: The test names `ReadPngAsyncReturnsWrittenData` and `ReadPngAsyncEmptyClipboard` contain "Async" which implies they test asynchronous behavior. However, without the feature flag enabled, they only test the synchronous code path. This could be misleading to future developers.

**Suggestion**: Either enable the feature flag (see Issue #1), or rename the tests to match what they actually verify (e.g., `ReadPngReturnsWrittenData`, `ReadPngEmptyClipboard`). Better yet, test both modes.

---

#### Issue #4: `data_dst` parameter is unused and passed through multiple layers
**Severity**: Suggestion
**File**: clipboard_win.cc / clipboard_win.h
**Lines**: 1094–1098 (ReadPngInternal signature), 729 (BindOnce capture)
**Description**: `ReadPngInternal` takes `const std::optional<DataTransferEndpoint>& data_dst` but never uses it (as noted in the comment). This parameter is captured by value in `base::BindOnce` at line 729, creating an unnecessary copy of `std::optional<DataTransferEndpoint>` into the bind state. While this is consistent with other `*Internal` methods (e.g., `ReadTextInternal`), it adds overhead for an unused parameter.

**Suggestion**: This is a pre-existing pattern and not specific to this CL, but worth noting. No action needed now—the comment "is kept as it may be used in the future" is reasonable.

---

#### Issue #5: Redundant feature flag check
**Severity**: Suggestion
**File**: clipboard_win.cc
**Lines**: 728 and 1081
**Description**: `ReadPng` checks `kNonBlockingOsClipboardReads` at line 728 before calling `ReadAsync`, and `ReadAsync` checks the same flag again at line 1081. When the flag is enabled, the inner check is redundant. This is by design (other callers like `ReadText` rely on `ReadAsync` to handle both cases), but it means the flag is evaluated twice in the ReadPng async path.

**Suggestion**: No change needed—this is intentional and consistent with how `ReadAsync` serves as a general-purpose routing function. Just noting for awareness.

---

## Positive Observations

- **Good refactoring of internal methods to static**: Making `ReadPngInternal`, `ReadPngTypeDataInternal`, and `ReadBitmapInternal` static is the correct approach for functions that need to run on a worker thread without access to `this`. This is consistent with how `ReadTextInternal`, `ReadAsciiTextInternal`, etc. are already implemented.

- **Clean separation of concerns**: Extracting `ReadPngTypeDataInternal` (raw PNG clipboard data) from `ReadPngInternal` (full PNG reading with bitmap fallback) is a good decomposition. The sync fallback path can call the lower-level functions directly, while the async path uses the composite `ReadPngInternal`.

- **Template generalization is well-done**: The change from `<typename Result>` to `<typename TaskReturnType, typename ReplyArgType = TaskReturnType>` correctly handles the type mismatch between `ReadPngInternal`'s return type (`std::vector<uint8_t>`) and `ReadPngCallback`'s parameter type (`const std::vector<uint8_t>&`). The default template argument (`ReplyArgType = TaskReturnType`) ensures backward compatibility with existing callers.

- **Consistent coding style**: The CL follows the existing patterns in `clipboard_win.cc` for async read operations. Brace style fixes (adding braces to single-line `if` blocks) are a nice cleanup.

- **Tests follow existing conventions**: The test structure mirrors the existing `ReadTextAsync*`, `ReadAsciiTextAsync*`, etc. tests.

---

## Overall Assessment

**LGTM with minor comments**

The CL correctly gates `ReadPng` behind `kNonBlockingOsClipboardReads` and refactors the internal methods to support running on a worker thread. The template generalization is well-motivated and correctly implemented. The primary concern is that the new async code path is not tested when the feature flag is actually enabled (Issue #1). This is consistent with existing tests in the file, but since this CL is specifically adding async support, testing it would strengthen confidence. Issues #2 and #3 are minor suggestions for improved clarity.
