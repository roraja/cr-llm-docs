# Code Review: CL 7578233

## [Clipboard][Windows] Use async ReadSvg/ReadRTF/ReadDataTransferCustomData/ReadData with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7578233
**Author:** Hewro Hewei <ihewro@chromium.org>

---

### Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Potential thread-safety concern with `RecordRead` from thread pool; typo in CL title; failing test |
| Style | ⚠️ | Minor: repetitive `// |data_dst| is not used` comments could be consolidated; missing `result->clear()` in sync `ReadData` |
| Security | ✅ | No new security concerns; existing patterns followed |
| Performance | ✅ | Good: offloading blocking clipboard reads to thread pool is the right approach |
| Testing | ⚠️ | Tests added but `ReadDataAsyncReturnsWrittenData` is failing on trybots; missing async-specific tests |

---

### Detailed Findings

#### Issue #1: CL title has typo "RadRTF" → "ReadRTF"
**Severity**: Minor
**File**: N/A (commit message)
**Description**: The CL title says "RadRTF" instead of "ReadRTF".
**Suggestion**: Fix the commit message before landing:
```
[Clipboard][Windows] Use async ReadSvg/ReadRTF/ReadDataTransferCustomData/ReadData with ThreadPool offloading
```

---

#### Issue #2: Test failure — `ClipboardWinTest.ReadDataAsyncReturnsWrittenData`
**Severity**: Critical
**File**: `ui/base/clipboard/clipboard_win_unittest.cc`
**Line**: 540–554
**Description**: The CQ run reports this test is failing in `interactive_ui_tests` on `win-rel`. The test writes 4 bytes (`{'d', 'a', 't', 'a'}`) using `WriteRawDataForTest` with `PlainTextType()` and then reads back with async `ReadData`. The comparison `EXPECT_EQ(data_future.Get(), "data")` likely fails because `GlobalSize()` may return a size larger than what was written (global memory allocations are rounded up, and `PlainTextType` clipboard data may include a null terminator added by the OS). The result string may contain trailing null bytes or extra padding.

**Suggestion**: Either:
1. Use a custom/non-standard `ClipboardFormatType` for the test (not `PlainTextType()`) so the OS doesn't interfere with the data, or
2. Compare only the expected prefix and check the result starts with `"data"`, or
3. Use `WriteData` + `ClipboardFormatType` that doesn't undergo OS transformations.

This is the most likely root cause of the test failure. `GlobalSize()` returns the actual allocated size of the memory block, which is typically rounded up. For `PlainTextType`, the OS may also add a null terminator when storing clipboard data.

---

#### Issue #3: `RecordRead` called from thread pool thread
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 711, 741, 804, 941
**Description**: `RecordRead()` calls `base::UmaHistogramEnumeration()`, which is thread-safe. However, calling it from the thread pool means metrics are recorded on a background thread, which is consistent with the existing pattern (e.g., `ReadTextInternal` at line 568, `ReadAsciiTextInternal` at line 604 already do this). This is not a bug but worth noting for consistency awareness.
**Suggestion**: No change needed — this follows the existing pattern.

---

#### Issue #4: Sync `ReadData` no longer calls `result->clear()` before assignment
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 931–936
**Description**: The sync `ReadData(const ClipboardFormatType&, const DataTransferEndpoint*, std::string*)` no longer explicitly calls `result->clear()` before assignment. While the `*result = ReadDataInternal(...)` assignment fully replaces the value (so this is functionally correct), the other sync read methods (`ReadSvg`, `ReadRTF`, `ReadDataTransferCustomData`) all explicitly call `result->clear()` before the assignment for defensive clarity.
**Suggestion**: Add `result->clear();` before the assignment for consistency with the pattern used in the other sync methods:
```cpp
void ClipboardWin::ReadData(const ClipboardFormatType& format,
                            const DataTransferEndpoint* data_dst,
                            std::string* result) const {
  CHECK(result);
  result->clear();
  *result = ReadDataInternal(format, GetClipboardWindow());
}
```

---

#### Issue #5: `ReadDataTransferCustomDataInternal` captures `type` by copy — acceptable but worth noting
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard_win.cc`
**Line**: 488–489
**Description**: In the async `ReadDataTransferCustomData`, `type` (a `const std::u16string&`) is bound into `BindOnce`. `base::BindOnce` will copy the string when binding it, which is correct behavior for use from a thread pool. This is fine.
**Suggestion**: No change needed.

---

#### Issue #6: No test coverage for the async-when-feature-enabled path specifically
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard_win_unittest.cc`
**Description**: The new tests exercise the async API signatures, but the `ReadAsync` method falls through to synchronous execution when `kNonBlockingOsClipboardReads` is disabled (which is the default). The tests don't enable this feature flag, so they only test the synchronous code path of `ReadAsync`. The actual thread pool offloading path is not specifically tested.

There are existing tests (e.g., `ReadTextDoNotTriggerDataChanged`) that also don't enable the feature flag for the new async overloads added, but those just test that `data_changed_count()` stays at 0. For the write+read correctness tests (`ReadSvgAsyncReturnsWrittenData`, etc.), they'd benefit from a parameterized version that also enables `kNonBlockingOsClipboardReads`.
**Suggestion**: Consider adding a test fixture variant or parameterized tests that enable `kNonBlockingOsClipboardReads` to exercise the actual async path:
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

---

#### Issue #7: Commit message is too sparse
**Severity**: Suggestion
**File**: N/A (commit message)
**Description**: The commit message only contains the title and Change-Id. It lacks a description of:
- Why these methods need async overrides
- How this relates to the broader `kNonBlockingOsClipboardReads` feature
- Any references to tracking bugs
**Suggestion**: Add a Bug: line and a brief description explaining the motivation (e.g., reducing jank from blocking clipboard reads on the UI thread).

---

### Positive Observations

- **Consistent pattern**: The new async overloads follow the exact same `ReadAsync(BindOnce(&Internal), callback)` pattern established by `ReadText`, `ReadAsciiText`, `ReadAvailableTypes`, `ReadHTML`, `ReadFilenames`, and `ReadPng`. This makes the code easy to review and understand.
- **Good refactoring**: Extracting the implementation into static `*Internal` methods that take an `HWND` parameter is clean and allows the same code to be used from both synchronous and asynchronous paths.
- **Defensive checks**: `CHECK(result)` is added in the sync overloads, which is a good defensive practice.
- **Return-value semantics**: The `*Internal` methods return by value instead of using out-params, which is cleaner and avoids potential null-pointer issues.
- **Brace consistency**: Early returns are now consistently braced (`if (...) { return result; }`), aligning with modern Chromium style.
- **Test coverage**: Both positive (write-then-read) and negative (empty clipboard) cases are tested for each new async method.

---

### Overall Assessment

**Needs changes before approval**

The CL is well-structured and follows established patterns, but has one critical issue:

1. **Critical**: The `ReadDataAsyncReturnsWrittenData` test is failing on the CQ. This is likely due to `GlobalSize()` returning a size larger than the written data when using `PlainTextType()`. This must be fixed before the CL can land.

2. **Minor**: Add `result->clear()` in the sync `ReadData` for consistency.

3. **Minor**: The commit message needs a bug reference and description.

4. **Suggestion**: Consider testing with `kNonBlockingOsClipboardReads` enabled to actually exercise the async thread pool path.
