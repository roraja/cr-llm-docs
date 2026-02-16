# CL Review Summary: [Clipboard][Windows] Use async ReadAvailableTypes with ThreadPool offloading

**CL Number:** 7574178  
**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7574178  
**Author:** Hewro Hewei (ihewro@chromium.org)  
**Status:** NEW  
**Files Changed:** 3 files, +82/-9 lines  
**Patch Set Reviewed:** 10

---

## 1. Executive Summary

This CL extends the async clipboard read pattern on Windows to `ReadAvailableTypes`, moving the blocking Win32 clipboard API calls (`IsClipboardFormatAvailable`, `GetClipboardData`) off the main thread onto a `ThreadPool` sequenced task runner with `MayBlock` traits. It follows the established pattern already in place for `ReadText`, `ReadAsciiText`, `ReadHTML`, `ReadFilenames`, and `ReadPng` by extracting static `*Internal` helper methods that accept an `HWND` parameter and delegating through the existing `ReadAsync` template.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | Clean separation of async wrapper and static internal logic. Follows existing patterns well. |
| Maintainability | 4 | Consistent with how other read methods were made async. Low cognitive overhead. |
| Extensibility | 4 | The `ReadAsync` template makes it trivial to add more async reads in the future. |
| Consistency | 5 | Perfectly mirrors the pattern used for `ReadText`, `ReadAsciiText`, `ReadHTML`, `ReadFilenames`. |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Calling Code                             │
│  clipboard->ReadAvailableTypes(buffer, data_dst, callback)      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│            ClipboardWin::ReadAvailableTypes (async)              │
│  Delegates to ReadAsync(BindOnce(ReadAvailableTypesInternal))   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │  ReadAsync<Result>       │
              │                         │
     Feature OFF?              Feature ON?
     (kNonBlockingOsClipboard) (kNonBlockingOsClipboard)
              │                         │
              ▼                         ▼
   ┌──────────────────┐   ┌──────────────────────────────┐
   │ Run synchronously │   │ PostTaskAndReplyWithResult   │
   │ on caller thread  │   │ on worker_task_runner_       │
   │ HWND=clipboard    │   │ (MayBlock, USER_BLOCKING)    │
   │    window         │   │ HWND=nullptr                 │
   └──────────┬────────┘   └──────────────┬───────────────┘
              │                           │
              ▼                           ▼
   ┌──────────────────────────────────────────────────────┐
   │  ReadAvailableTypesInternal(buffer, owner_window)    │
   │  [static]                                            │
   │                                                      │
   │  1. GetStandardFormatsInternal(buffer) [static]      │
   │     └─ ::IsClipboardFormatAvailable(...)             │
   │  2. IsFormatAvailableInternal(CustomType) [static]   │
   │  3. ScopedClipboard::Acquire(owner_window)           │
   │  4. GetClipboardDataWithLimit(...)                   │
   │  5. ReadCustomDataTypes(...)                         │
   │                                                      │
   │  Returns: std::vector<std::u16string>                │
   └──────────────────────────┬───────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────┐
   │  Callback invoked with result on original sequence   │
   └──────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | Logic is preserved from the synchronous version. The refactoring to static methods is sound. One minor concern about `GetClipboardWindow()` behavior in the sync path (see findings). |
| Efficiency | 4 | Moves blocking Win32 calls off the main thread. ThreadPool task runner with `MayBlock` is appropriate. |
| Readability | 4 | Clean, follows established patterns. Good use of static methods for thread-safe extraction. |
| Test Coverage | 3 | Two new async-specific tests plus an update to existing no-notification test. Missing edge cases (see below). |

---

## 4. Key Findings

### Critical Issues (Must Fix)

*None identified.* The CL is mechanically sound and follows the established async clipboard pattern.

### Major Issues (Should Fix)

1. **Missing blank line before `ReadAvailableTypes` async override (style):**
   In `clipboard_win.cc`, the `ReadAvailableTypes` async method (lines 432-438) is missing the standard comment block `// |data_dst| is not used...` and a blank line separator after the preceding `ReadAsciiText` method. All other async read methods have this comment. This should be added for consistency:
   ```cpp
   // |data_dst| is not used. It's only passed to be consistent with other
   // platforms.
   void ClipboardWin::ReadAvailableTypes(
   ```

2. **Sync `ReadAvailableTypes` changed `DCHECK` to `CHECK`:**
   The sync `ReadAvailableTypes` (line 479) changed `DCHECK(types)` to `CHECK(types)`. While `CHECK` is generally preferred in new code, this change is undocumented in the commit message and is a behavioral difference (crash in release vs. only in debug). This should either be noted in the commit message or left as `DCHECK` for this refactoring-only CL to minimize behavioral changes. However, `CHECK` is the modern Chromium preference, so this is acceptable if intentional.

### Minor Issues (Nice to Fix)

1. **`GetStandardFormatsInternal` and `IsFormatAvailableInternal` don't use `owner_window`:**
   Both `GetStandardFormatsInternal` and `IsFormatAvailableInternal` are static methods that call Win32 APIs (`::IsClipboardFormatAvailable`) which do **not** require clipboard ownership. This design is correct—they don't need `HWND`—but it means they can be called from any thread without coordination concerns. Consider adding a brief comment to `GetStandardFormatsInternal` and `IsFormatAvailableInternal` noting they are thread-safe since they only call `::IsClipboardFormatAvailable`.

2. **Commit message is terse:**
   The commit message ("Moving the blocking work onto a ThreadPool sequenced task runner (MayBlock).") doesn't mention which specific method is being made async (`ReadAvailableTypes`). Adding specificity (e.g., "Implement async ReadAvailableTypes override for ClipboardWin, moving blocking Win32 clipboard calls onto a ThreadPool sequenced task runner") would improve the git log.

### Suggestions (Optional)

1. **Consider using `base::ranges::contains` in tests:**
   The test `ReadAvailableTypesAsyncReturnsWrittenData` uses `std::find(types.begin(), types.end(), ...)`. Consider using `base::Contains(types, kMimeTypePlainText16)` for readability, which is idiomatic in Chromium.
   ```cpp
   // Instead of:
   EXPECT_NE(std::find(types.begin(), types.end(), kMimeTypePlainText16), types.end());
   // Consider:
   EXPECT_TRUE(base::Contains(types, kMimeTypePlainText16));
   ```

2. **Consider adding a test with custom data types:**
   The current tests cover standard text types and empty clipboard but don't test the `DataTransferCustomType` code path in `ReadAvailableTypesInternal`. A test that writes custom data and verifies the async path returns it would improve coverage.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | Coverage |
|------|----------|
| `NoDataChangedNotificationOnRead` (updated) | Verifies async `ReadAvailableTypes` doesn't trigger clipboard change notifications |
| `ReadAvailableTypesAsyncReturnsWrittenData` (new) | Writes plain text, reads async, verifies `text/plain` is in results |
| `ReadAvailableTypesAsyncEmptyClipboard` (new) | Clears clipboard, reads async, verifies empty result |

### Missing Tests

1. **Custom data types path:** No test exercises the `DataTransferCustomType` code path through the async API. When custom MIME types are written via `ScopedClipboardWriter::WriteData`, the async `ReadAvailableTypes` should return those custom types.

2. **Multiple format types:** No test writes HTML + text + image and verifies all standard formats are returned via async.

3. **Feature flag off path:** No explicit test verifies the synchronous fallback when `kNonBlockingOsClipboardReads` is disabled, though this is implicitly covered by the sync `ReadAvailableTypes` tests.

4. **Concurrent access:** No test verifies behavior when clipboard contents change between the async post and the callback. (This is likely out of scope for this CL.)

### Recommended Additional Tests

- Add a test with `ScopedClipboardWriter::WriteData` writing a custom type, then verifying it appears in `ReadAvailableTypes` async results.
- Add a test writing multiple formats (text + HTML) and checking both appear.

---

## 6. Security Considerations

- **No new security concerns.** The CL moves existing Win32 API calls to a thread pool but does not introduce any new data flows, IPC, or user-facing input handling.
- The `HWND=nullptr` passed when running on the thread pool worker (for `ScopedClipboard::Acquire`) is the established pattern and is safe for read operations on Windows — `OpenClipboard(NULL)` associates the clipboard with the calling task.
- The existing `GetClipboardDataWithLimit` function provides bounds checking on clipboard data, and this CL continues to use it.

---

## 7. Performance Considerations

- **Positive impact:** Offloading `ReadAvailableTypes` to the thread pool removes a potential UI-thread blocking operation. Win32 clipboard APIs (`OpenClipboard`, `GetClipboardData`, `IsClipboardFormatAvailable`) can block if another application holds the clipboard lock. This is consistent with the project's goal of non-blocking OS clipboard reads.
- **Feature-gated:** The async behavior is gated behind `kNonBlockingOsClipboardReads`, so there is zero risk to the synchronous path.
- **No extra allocations:** The refactoring returns `std::vector<std::u16string>` by value, which benefits from move semantics. No unnecessary copies introduced.
- **Benchmarking:** No specific benchmarking is needed for this CL since it follows the identical pattern of other already-shipped async clipboard methods. The overhead is one thread pool post + reply per call, which is negligible compared to the Win32 blocking time saved.

---

## 8. Final Recommendation

**Verdict**: APPROVED_WITH_COMMENTS

**Rationale:**
This is a clean, well-structured CL that follows the established async clipboard read pattern on Windows. The code is correct, the refactoring is minimal and mechanical, and the feature is properly gated behind a feature flag. The CL has passed dry runs. The issues identified are minor (style consistency, commit message clarity, and test coverage gaps) and none are blocking.

**Action Items for Author:**

1. Add the standard `// |data_dst| is not used...` comment before the async `ReadAvailableTypes` override in `clipboard_win.cc` for consistency with other async read methods.
2. Consider adding specificity to the commit message about which method was made async.
3. Consider using `base::Contains` instead of `std::find` in the test for readability.
4. *(Optional)* Consider adding a test for the custom data types path through `ReadAvailableTypesInternal`.

---

## 9. Comments for Gerrit

### Patchset-level comment:

> LGTM with minor nits.
>
> Clean implementation following the established async pattern. The refactoring to static `*Internal` methods is well-done and consistent with `ReadTextInternal`, `ReadAsciiTextInternal`, etc. A few minor suggestions below.

### File: `ui/base/clipboard/clipboard_win.cc`

**Line 432** (before `void ClipboardWin::ReadAvailableTypes` async override):
> Nit: Missing the `// |data_dst| is not used...` comment that all the other async read methods have. Please add for consistency:
> ```cpp
> // |data_dst| is not used. It's only passed to be consistent with other
> // platforms.
> ```

**Line 479** (`CHECK(types)` in sync `ReadAvailableTypes`):
> Note: This was changed from `DCHECK` to `CHECK`. The change itself is fine (and aligns with modern Chromium style), but it's a behavioral change beyond the stated scope. Please either mention this in the commit message or revert to `DCHECK` to keep this CL purely about the async refactoring.

### File: `ui/base/clipboard/clipboard_win_unittest.cc`

**Line 389** (`std::find` usage in `ReadAvailableTypesAsyncReturnsWrittenData`):
> Nit: Consider `EXPECT_TRUE(base::Contains(types, kMimeTypePlainText16))` for readability.

**Line 377-401** (new tests):
> The new tests look good. Consider adding a test that writes custom data types (via `ScopedClipboardWriter::WriteData`) to exercise the `DataTransferCustomType` path in `ReadAvailableTypesInternal`, which is the more complex branch.
