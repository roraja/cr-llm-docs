# CL 7578233 Review Summary

**CL**: [7578233](https://chromium-review.googlesource.com/c/chromium/src/+/7578233)  
**Title**: [Clipboard][Windows] Use async ReadSvg/ReadRTF/ReadDataTransferCustomData/ReadData with ThreadPool offloading  
**Author**: Hewro Hewei (ihewro@chromium.org)  
**Status**: NEW (Dry run failed)  
**Files Changed**: 3 files, +271/-24 lines

---

## 1. Executive Summary

This CL continues the series of clipboard async refactoring CLs for Windows, converting four more synchronous clipboard read operations (`ReadSvg`, `ReadRTF`, `ReadDataTransferCustomData`, `ReadData`) to support async execution via ThreadPool offloading using the existing `ReadAsync` template pattern. The refactoring extracts static `*Internal` methods from the sync implementations and wires them into callback-based async overrides, following the same pattern established in prior CLs for `ReadText`, `ReadAsciiText`, `ReadFilenames`, `ReadPng`, `ReadHTML`, and `ReadAvailableTypes`. The CL also adds comprehensive unit tests for each newly async-ified method.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | Clean separation of sync wrappers and static `*Internal` methods. Consistent with established patterns from prior CLs. |
| Maintainability | 4 | Each method follows the same refactoring template, making future changes predictable. Sync methods now delegate to Internal methods, reducing duplication. |
| Extensibility | 5 | The `ReadAsync` template pattern makes it trivial to add more async read methods. New Internal methods can be reused by other callers (e.g., `ReadSvgInternal` calls `ReadDataInternal`). |
| Consistency | 4 | Follows the exact pattern of prior CLs in the series. Minor inconsistency: `ReadPng` has a different inline feature-flag check pattern vs. these methods which always delegate to `ReadAsync` (the newer/cleaner pattern). |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Clipboard (base class)                │
│  virtual ReadSvg/ReadRTF/ReadData/ReadCustomData        │
│  (callback-based async API)                             │
└──────────────────────┬──────────────────────────────────┘
                       │ override
┌──────────────────────▼──────────────────────────────────┐
│                    ClipboardWin                          │
│                                                         │
│  ReadSvg(callback) ──► ReadAsync(ReadSvgInternal, cb)   │
│  ReadRTF(callback) ──► ReadAsync(ReadRTFInternal, cb)   │
│  ReadData(callback) ─► ReadAsync(ReadDataInternal, cb)  │
│  ReadDataTransferCustomData(cb)                         │
│    ──► ReadAsync(ReadDataTransferCustomDataInternal, cb) │
│                                                         │
│  ReadSvg(sync)  ──► ReadSvgInternal(owner_window)       │
│  ReadRTF(sync)  ──► ReadRTFInternal(owner_window)       │
│  ReadData(sync) ──► ReadDataInternal(owner_window)      │
│  ReadDataTransferCustomData(sync)                       │
│    ──► ReadDataTransferCustomDataInternal(owner_window)  │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│               ReadAsync<T, R> template                  │
│                                                         │
│  if (!kNonBlockingOsClipboardReads) {                   │
│    run read_func synchronously on UI thread              │
│  } else {                                               │
│    PostTaskAndReplyWithResult on worker_task_runner_     │
│    (ThreadPool, owner_window=nullptr)                   │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | The core refactoring is correct, but the `ReadDataAsyncReturnsWrittenData` test is **failing on trybots** — see Critical Issues. Also, `ReadSvgInternal` calls `ReadDataInternal` which double-records UMA metrics (`kSvg` + `kData`), inheriting a pre-existing behavior quirk. |
| Efficiency | 4 | ThreadPool offloading avoids blocking the UI thread for clipboard I/O. Return-by-value from Internal methods is efficient with RVO/NRVO. |
| Readability | 4 | Clean, consistent pattern. Comment blocks ("data_dst is not used") are duplicated per-method but match existing style. |
| Test Coverage | 4 | Good coverage with positive and empty-clipboard tests for all four methods. Async notification-non-interference tests also added. Missing edge cases noted below. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Test Failure: `ReadDataAsyncReturnsWrittenData`** — The CL's dry run failed on `win-rel` with this test. The test writes raw data `{'d', 'a', 't', 'a'}` using `WriteRawDataForTest(PlainTextType(), ...)` and expects to read back exactly `"data"`. However, `ReadDataInternal` uses `GlobalSize()` which returns the *allocated* size of the global memory block, which may include a null terminator or padding beyond the 4 bytes written. The returned `std::string` will therefore be `"data\0"` (or longer), causing the `EXPECT_EQ` to fail. The old sync `ReadData` had the same behavior, but previously the test didn't exist. **Fix**: Either (a) trim trailing nulls in `ReadDataInternal`, (b) change the test to use `EXPECT_TRUE(base::StartsWith(...))` or check `data_future.Get().substr(0, 4)`, or (c) use a format type whose write path guarantees exact-size allocation.

2. **Typo in CL subject**: "RadRTF" should be "ReadRTF" in the commit message title.

### Major Issues (Should Fix)

1. **Double UMA recording for SVG/RTF reads** — `ReadSvgInternal` records `ClipboardFormatMetric::kSvg` and then calls `ReadDataInternal` which records `ClipboardFormatMetric::kData`. Similarly, `ReadRTFInternal` records `kRtf` then `kData`. This was the same behavior before this CL (the old sync `ReadSvg` called the sync `ReadData`), but now that both are static Internal methods, this is a good opportunity to create a private `ReadDataRawInternal` that skips the `RecordRead(kData)` call, to avoid inflating the `Clipboard.Read.kData` histogram with reads that are actually SVG/RTF reads. This is a pre-existing issue but worth fixing in this CL since the code is being refactored.

2. **`FeatureList::IsEnabled` called from ThreadPool** — `ReadSvgInternal` calls `base::FeatureList::IsEnabled(features::kUseUtf8EncodingForSvgImage)` at line 716. When run on the ThreadPool via `ReadAsync`, `FeatureList::IsEnabled` is generally thread-safe, but if the `FeatureList` is torn down during shutdown, this could be problematic. Other Internal methods in this CL don't use `FeatureList`. Verify that `FeatureList` outlives any pending ThreadPool tasks, or move the feature check to the call site before dispatching to the ThreadPool.

### Minor Issues (Nice to Fix)

1. **Missing `result->clear()` in sync `ReadData`** — The sync `ReadData(format, data_dst, result)` does `CHECK(result); *result = ReadDataInternal(...)` but doesn't call `result->clear()` first. While assignment replaces the entire content, other sync methods (`ReadSvg`, `ReadRTF`, `ReadDataTransferCustomData`) explicitly call `result->clear()` before assignment for consistency.

2. **`locked_data` scope in `ReadDataTransferCustomDataInternal`** — The `ScopedHGlobal` at line 819 locks global memory, but its scope extends beyond where `hdata` is needed. A tighter scope block (like the one in `ReadFilenamesInternal`) would be slightly cleaner, though not functionally wrong.

3. **Commit message lacks bug reference** — There is no `Bug:` line in the commit message. The related tracking bug (if any, e.g., crbug.com/443355 for async clipboard migration) should be referenced.

### Suggestions (Optional)

1. **Consider adding `TrimAfterNull` to `ReadDataInternal`** — Currently `ReadDataInternal` returns raw data including any null padding from `GlobalSize()`. Since many callers (SVG, RTF) trim after null anyway, and the raw bytes could contain trailing garbage from heap allocation, consider trimming in `ReadDataInternal` or at minimum documenting this behavior.

2. **Consider consolidating the `ReadPng` async pattern** — `ReadPng` (line 759) has a slightly different pattern where it checks `kNonBlockingOsClipboardReads` inline before calling `ReadAsync`, while the new methods in this CL always call `ReadAsync` which handles the flag internally. The newer pattern is cleaner and could be applied to `ReadPng` in a follow-up CL for consistency.

3. **Use `std::move(maybe_result).value()` consistently** — In `ReadDataTransferCustomDataInternal` line 823, `std::move(maybe_result.value())` is used. The more canonical C++ pattern is `std::move(*maybe_result)` which avoids an unnecessary `.value()` call. The old code used `std::move(*maybe_result)` — the change to `.value()` is functionally equivalent but a minor style regression.

---

## 5. Test Coverage Analysis

### Existing Tests
- **`NoDataChangedNotificationOnRead`**: Extended to verify async ReadSvg, ReadRTF, ReadDataTransferCustomData, and ReadData don't trigger clipboard change notifications. ✅
- **`ReadSvgAsyncReturnsWrittenData`**: Writes SVG, reads async, verifies content. ✅
- **`ReadSvgAsyncEmptyClipboard`**: Reads SVG from empty clipboard, expects empty. ✅
- **`ReadRTFAsyncReturnsWrittenData`**: Writes RTF, reads async, verifies content. ✅
- **`ReadRTFAsyncEmptyClipboard`**: Reads RTF from empty clipboard, expects empty. ✅
- **`ReadDataTransferCustomDataAsyncReturnsWrittenData`**: Writes custom data via pickle, reads async, verifies content. ✅
- **`ReadDataTransferCustomDataAsyncEmptyClipboard`**: Reads custom data from empty clipboard, expects empty. ✅
- **`ReadDataAsyncReturnsWrittenData`**: Writes raw data, reads async, verifies content. ❌ **FAILING**
- **`ReadDataAsyncEmptyClipboard`**: Reads data from empty clipboard, expects empty. ✅

### Missing Tests
1. **Async read with `kNonBlockingOsClipboardReads` enabled** — All tests run with the default feature state. Tests should explicitly enable this feature flag (using `base::test::ScopedFeatureList`) to verify the ThreadPool path works correctly.
2. **Concurrent async reads** — No test verifies that multiple async reads of different formats can be dispatched simultaneously without interference.
3. **SVG encoding variants** — `ReadSvgInternal` has two code paths depending on `kUseUtf8EncodingForSvgImage`. Tests should cover both paths.
4. **RTF encoding normalization** — `ReadRTFInternal` does encoding detection and normalization. A test with non-UTF-8 RTF content would exercise this path.
5. **Large clipboard data** — No test verifies behavior when clipboard data exceeds `kMaxClipboardSize`.

### Recommended Additional Tests
- Add a parameterized test fixture that runs all async tests with `kNonBlockingOsClipboardReads` both enabled and disabled.
- Add a test for `ReadSvg` with `kUseUtf8EncodingForSvgImage` enabled and disabled.
- Fix `ReadDataAsyncReturnsWrittenData` to account for `GlobalSize()` padding behavior.

---

## 6. Security Considerations

- **Thread safety of static methods**: The new `*Internal` static methods access Win32 clipboard APIs (`GetClipboardData`, `GlobalLock`, `GlobalSize`, `GlobalUnlock`) from either the UI thread or ThreadPool. When on the ThreadPool, `owner_window` is `nullptr`. The Win32 clipboard requires `OpenClipboard`/`CloseClipboard` to be called on the same thread (managed by `ScopedClipboard`), which is correctly handled. **No issues found.**

- **UMA histogram recording from ThreadPool**: `RecordRead` calls `base::UmaHistogramEnumeration` which is thread-safe. **No issues found.**

- **Input validation**: `ReadDataTransferCustomDataInternal` receives a `std::u16string type` parameter from user-provided data. This is passed to `ReadCustomDataForType` which uses it as a lookup key. The type string is not used in any unsafe operation. **No issues found.**

- **Memory handling**: `GlobalLock`/`GlobalUnlock` pairs are correctly matched. `ScopedHGlobal` in `ReadDataTransferCustomDataInternal` provides RAII cleanup. **No issues found.**

---

## 7. Performance Considerations

- **Positive**: Moving clipboard reads off the UI thread reduces jank, especially for large clipboard content (images, long text). This is the primary purpose of the CL series.

- **ThreadPool overhead**: For small clipboard reads, the ThreadPool dispatch overhead (task posting, thread hop, reply) may be more expensive than a direct synchronous read. This is mitigated by the feature flag `kNonBlockingOsClipboardReads` which allows falling back to synchronous reads.

- **Double clipboard open/close for SVG and RTF**: `ReadSvgInternal` calls `ReadDataInternal`, which opens and closes the clipboard. If `ReadSvgInternal` needed to do additional clipboard operations, this would mean multiple open/close cycles. Currently this is not an issue since SVG only does the one data read, but it's worth noting for future extensibility.

- **Benchmarking recommendations**: Measure clipboard read latency on Windows with `kNonBlockingOsClipboardReads` enabled vs. disabled using `chrome://tracing` or perfetto traces. Focus on paste operations with large clipboard content.

---

## 8. Final Recommendation

**Verdict**: NEEDS_WORK

**Rationale**:
The CL is a well-structured, incremental step in the clipboard async migration series. The architecture follows established patterns correctly, and the code quality is generally high. However, the CL has a **confirmed test failure** (`ReadDataAsyncReturnsWrittenData` on `win-rel`) that must be addressed before landing. The test failure is likely caused by `GlobalSize()` returning a larger allocation size than the 4 bytes written, resulting in the read string containing extra bytes beyond `"data"`. Additionally, there is a typo in the commit message ("RadRTF" → "ReadRTF"), and the commit message lacks a bug reference.

**Action Items for Author**:
1. **[Must Fix]** Fix `ReadDataAsyncReturnsWrittenData` test failure — investigate why `GlobalSize()` returns more bytes than written and adjust the test expectation or the write mechanism accordingly.
2. **[Must Fix]** Fix typo in commit message: "RadRTF" → "ReadRTF".
3. **[Should Fix]** Add a `Bug:` line to the commit message referencing the tracking bug for async clipboard migration.
4. **[Should Fix]** Consider adding `result->clear()` in the sync `ReadData` wrapper for consistency with other sync methods.
5. **[Nice to Fix]** Add tests with `kNonBlockingOsClipboardReads` feature flag explicitly enabled to cover the ThreadPool code path.
6. **[Nice to Fix]** Revert `.value()` to `*maybe_result` in `ReadDataTransferCustomDataInternal` to match prior style.

---

## 9. Comments for Gerrit

### Comment 1: `clipboard_win_unittest.cc`, `ReadDataAsyncReturnsWrittenData` test

**File**: `ui/base/clipboard/clipboard_win_unittest.cc`  
**Line**: ~553 (the `EXPECT_EQ(data_future.Get(), "data")` line)

> This test is failing on `win-rel` trybots. The likely cause is that `WriteRawDataForTest` writes 4 bytes (`{'d', 'a', 't', 'a'}`) to the clipboard, but `ReadDataInternal` uses `GlobalSize()` to determine the size of data to read. `GlobalSize()` returns the *allocated* size of the global memory block, which may be larger than the written data (Windows allocates in granular blocks and may include null terminator padding). This means `data_future.Get()` could be `"data\0..."` rather than `"data"`.
>
> Consider either:
> 1. Using `EXPECT_TRUE(base::StartsWith(data_future.Get(), "data"))` or checking a prefix
> 2. Writing data with a format that guarantees exact-size round-tripping
> 3. Trimming trailing nulls in the result

### Comment 2: Commit message typo

**File**: (commit message)

> Typo: "RadRTF" should be "ReadRTF" in the subject line.

### Comment 3: `clipboard_win.cc`, double UMA recording

**File**: `ui/base/clipboard/clipboard_win.cc`  
**Line**: ~711 and ~741

> Nit: `ReadSvgInternal` records `kSvg` then calls `ReadDataInternal` which records `kData`. Similarly `ReadRTFInternal` records `kRtf` then `kData`. This means SVG/RTF reads inflate the `Clipboard.Read.kData` count. This is pre-existing behavior, but since you're refactoring these methods, consider creating a lower-level helper that skips the `RecordRead(kData)` call, or passing a flag to skip recording. Not blocking.

### Comment 4: Missing `result->clear()` in sync ReadData

**File**: `ui/base/clipboard/clipboard_win.cc`  
**Line**: ~933

> Nit: The sync `ReadData` does `CHECK(result); *result = ReadDataInternal(...)` without calling `result->clear()` first. While assignment replaces the content entirely, the other sync wrappers (`ReadSvg`, `ReadRTF`, `ReadDataTransferCustomData`) all call `result->clear()` explicitly before assignment. Consider adding it for consistency.

### Comment 5: Missing test coverage for ThreadPool path

**File**: `ui/base/clipboard/clipboard_win_unittest.cc`

> Could you add tests that explicitly enable `kNonBlockingOsClipboardReads` via `ScopedFeatureList`? The current tests run with the feature disabled (default), so only the synchronous fallback path in `ReadAsync` is exercised. The ThreadPool path is the main purpose of this CL and should be tested.

### Comment 6: Missing Bug reference

**File**: (commit message)

> Please add a `Bug:` line referencing the tracking bug for the async clipboard migration work (e.g., crbug.com/443355 or the specific sub-bug for Windows).
