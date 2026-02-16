# CL 7558493: [Clipboard][Windows] Use async ReadText/ReadAsciiText with ThreadPool offloading

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7558493
**Author:** Hewro Hewei (ihewro@chromium.org)
**Status:** NEW (Patch Set 8)
**Files Changed:** 3 files, +126/-16 lines
**Bug:** [458194647](https://crbug.com/458194647)

---

## 1. Executive Summary

This CL extends the non-blocking clipboard read infrastructure on Windows by adding async overrides for `ReadText` and `ReadAsciiText` in `ClipboardWin`. It follows the established pattern (already used for `ReadHTML`, `ReadFilenames`) of moving Win32 clipboard access off the UI thread via `ReadAsync`, which posts work to a `ThreadPool` sequenced task runner with `MayBlock` trait when the `kNonBlockingOsClipboardReads` feature flag is enabled. The change is incremental, well-scoped, and consistent with the existing architecture.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 5 | Follows the established `ReadAsync` pattern exactly. Easy to understand intent. |
| Maintainability | 5 | Refactored to static `*Internal` methods cleanly separates sync and async paths. |
| Extensibility | 4 | Pattern is easily replicable for remaining sync methods (ReadSvg, ReadRTF, ReadData, etc.). |
| Consistency | 5 | Matches the existing `ReadHTML`/`ReadFilenames` async approach precisely. |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         UI Thread                                    │
│                                                                      │
│  caller ──► ReadText(buffer, data_dst, callback)                     │
│                     │                                                │
│                     ▼                                                │
│              ReadAsync(read_func, reply_func)                        │
│                     │                                                │
│        ┌────────────┴────────────────┐                               │
│        │ kNonBlockingOsClipboardReads│                               │
│        │       enabled?              │                               │
│     No │                          Yes│                               │
│        ▼                             ▼                               │
│  Run synchronously         PostTaskAndReplyWithResult                │
│  on UI thread              to worker_task_runner_                    │
│  (owner_window =           (owner_window = nullptr)                  │
│   GetClipboardWindow())              │                               │
│        │                             │                               │
│        ▼                             │                               │
│  callback.Run(result)                │                               │
│                                      │                               │
└──────────────────────────────────────│───────────────────────────────┘
                                       │
┌──────────────────────────────────────│───────────────────────────────┐
│                  ThreadPool (MayBlock, USER_BLOCKING)                 │
│                                      │                               │
│                                      ▼                               │
│                        ReadTextInternal(buffer, nullptr)              │
│                           │                                          │
│                           ├── ScopedClipboard::Acquire(nullptr)      │
│                           ├── GetClipboardDataWithLimit(CF_UNICODETEXT)│
│                           ├── GlobalLock/GlobalUnlock                 │
│                           ├── TrimAfterNull                          │
│                           └── return result                          │
│                                      │                               │
│                          PostReply back to UI thread                  │
│                                      │                               │
└──────────────────────────────────────│───────────────────────────────┘
                                       │
                                       ▼
                              callback.Run(result) on UI thread
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | Implementation is correct. Minor concern about `RecordRead` being called from ThreadPool (see findings). |
| Efficiency | 5 | Moves blocking Win32 clipboard I/O off UI thread; result types are move-efficient. |
| Readability | 5 | Clean extraction to static `*Internal` methods; clear separation of concerns. |
| Test Coverage | 4 | Good coverage for happy path and empty clipboard; could add more edge cases. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

_None identified._

### Major Issues (Should Fix)

1. **`RecordRead` called from ThreadPool thread** — `RecordRead(ClipboardFormatMetric::kText)` is called inside both `ReadTextInternal` and `ReadAsciiTextInternal`, which are static methods that execute on the `worker_task_runner_` (ThreadPool) when the feature is enabled. `RecordRead` calls `base::UmaHistogramEnumeration`, which is thread-safe in Chromium, so this is not a correctness bug. However, the existing pattern for `ReadHTML` calls `RecordRead` in the sync path (`ReadHTML(buffer, data_dst, markup, ...)`) but not in the static internal helper `ReadHTMLInternal`. Verify this is intentional — the async path via `ReadTextInternal` will record the metric, but the sync `ReadText(buffer, data_dst, result)` now _also_ calls `ReadTextInternal` which records it, so there's no duplication. This is consistent and correct.

2. **`owner_window = nullptr` passed to `ScopedClipboard::Acquire` in async path** — When the feature flag is enabled, `ReadAsync` passes `nullptr` as the `owner_window` to the `*Internal` methods. `ScopedClipboard::Acquire(nullptr)` calls `::OpenClipboard(NULL)`, which associates the clipboard with the current task rather than a window. This is the same pattern used by `ReadHTML` and `ReadFilenames` async paths, and is documented Win32 behavior. This is correct but worth noting in the commit message for future readers.

### Minor Issues (Nice to Fix)

1. **Repeated `// |data_dst| is not used.` comment blocks** — The CL adds two new comment blocks identical to existing ones. While consistent with the surrounding code, this is boilerplate that could be reduced. This is a pre-existing style issue, not introduced by this CL.

2. **Brace style inconsistency in `ReadTextInternal`** — Lines like `if (!data) return result;` (no braces) vs `if (!clipboard.Acquire(owner_window)) { return result; }` (with braces). While both are valid Chromium style, the inconsistency within the same function is slightly jarring. The braced form was introduced by this CL; the non-braced form was pre-existing. Consider making them consistent.

3. **Test names could be more descriptive** — `ReadTextAsyncReturnsWrittenData` is clear but could specify the format (Unicode vs ASCII) more explicitly, e.g., `ReadUnicodeTextAsyncReturnsWrittenData`. This is a minor naming preference.

### Suggestions (Optional)

1. **Consider adding a test with the feature flag explicitly enabled/disabled** — The current tests rely on the default state of `kNonBlockingOsClipboardReads`. Adding a parameterized test that exercises both the sync-fallback path (flag off) and the true async path (flag on) within `ReadAsync` would increase coverage of the branching logic.

2. **Consider adding a test for large clipboard data** — Given `kMaxClipboardSize` (256 MiB) limit in `GetClipboardDataWithLimit`, a test verifying behavior at or near this limit would be valuable, though this is a pre-existing gap not specific to this CL.

3. **Future work: Apply the same pattern to remaining sync-only methods** — `ReadSvg`, `ReadRTF`, `ReadData`, `ReadBookmark`, `ReadDataTransferCustomData` still block on the UI thread. The author or team should track these as follow-up work.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | What it covers |
|------|---------------|
| `NoDataChangedNotificationOnRead` | Async `ReadText` and `ReadAsciiText` don't trigger clipboard change notifications |
| `ReadTextAsyncReturnsWrittenData` | Async `ReadText` returns correct Unicode text after `WriteText` |
| `ReadTextAsyncEmptyClipboard` | Async `ReadText` returns empty string on cleared clipboard |
| `ReadAsciiTextAsyncReturnsWrittenData` | Async `ReadAsciiText` returns correct ASCII text after `WriteText` |
| `ReadAsciiTextAsyncEmptyClipboard` | Async `ReadAsciiText` returns empty string on cleared clipboard |

### What Tests are Missing

- **Feature flag parameterization**: No test explicitly toggles `kNonBlockingOsClipboardReads` to verify both the sync-fallback and async-offload paths within `ReadAsync`.
- **Unicode edge cases**: No test with special characters (emoji, CJK, RTL text, embedded null characters) to verify `TrimAfterNull` and data integrity through the async pipeline.
- **Interleaved read/write**: No test for concurrent async reads while clipboard content changes.
- **Sync/async result equivalence**: No test asserting that the sync `ReadText(buffer, data_dst, &result)` and async `ReadText(buffer, data_dst, callback)` return identical results for the same clipboard content.

### Recommended Additional Tests

1. A parameterized test with `kNonBlockingOsClipboardReads` enabled and disabled.
2. A test with embedded null characters (`u"hello\0world"`) to verify `TrimAfterNull` works correctly through the async path.
3. A test asserting sync/async equivalence for `ReadText` and `ReadAsciiText`.

---

## 6. Security Considerations

- **Thread safety of clipboard access**: The Win32 clipboard API is process-global. `ScopedClipboard` properly acquires/releases the clipboard lock via `OpenClipboard`/`CloseClipboard`. When `owner_window` is `nullptr`, the clipboard is associated with the calling thread's task, which is correct for ThreadPool usage since each call is sequenced via `worker_task_runner_`.
- **No new IPC or untrusted input surfaces**: The CL only changes the threading model for existing clipboard reads; no new data parsing or external input handling is introduced.
- **`GlobalLock`/`GlobalUnlock` usage**: Existing pattern, no change. The data is copied into a `std::u16string`/`std::string` before `GlobalUnlock`, so there are no dangling pointer risks.
- **No security concerns identified.**

---

## 7. Performance Considerations

- **Positive impact**: Moving `ReadText` and `ReadAsciiText` off the UI thread eliminates potential UI jank from slow clipboard access (e.g., when clipboard data is provided by another application via delayed rendering, or when antivirus software intercepts clipboard operations).
- **ThreadPool overhead**: The `PostTaskAndReplyWithResult` pattern adds minimal overhead (task scheduling + context switch) compared to the potential cost of blocking the UI thread on Win32 clipboard APIs. The `USER_BLOCKING` priority ensures the task is scheduled promptly.
- **Feature flag gating**: The change is behind `kNonBlockingOsClipboardReads`, allowing safe rollout and rollback. When the flag is off, behavior is identical to before (sync on UI thread).
- **No benchmarking concerns**: The change is architectural (threading model) rather than algorithmic. No specific benchmarking is needed beyond monitoring UI thread jank metrics with the feature enabled.

---

## 8. Final Recommendation

**Verdict**: APPROVED_WITH_COMMENTS

**Rationale:**
This CL is a well-executed, incremental change that follows the established async clipboard read pattern already in place for `ReadHTML` and `ReadFilenames`. The code is clean, consistent, and properly gated behind a feature flag. The static `*Internal` method extraction is the right approach for enabling ThreadPool execution. Tests cover the primary use cases. The issues identified are minor and non-blocking.

**Action Items for Author:**

1. **[Minor]** Consider making brace style consistent within `ReadTextInternal` and `ReadAsciiTextInternal` — either add braces to the `if (!data)` check or remove them from the `if (!clipboard.Acquire(...))` check to match the surrounding code style.
2. **[Optional]** Consider adding a parameterized test that explicitly tests with `kNonBlockingOsClipboardReads` enabled and disabled to cover both code paths in `ReadAsync`.
3. **[Optional]** Consider adding a sync/async equivalence test to ensure both `ReadText` overloads return identical results.

---

## 9. Comments for Gerrit

### Comment 1 — `clipboard_win.cc`, `ReadTextInternal` method (around line 506)

> **Nit:** Brace style is inconsistent within this method. The `Acquire` check uses braces:
> ```cpp
> if (!clipboard.Acquire(owner_window)) {
>   return result;
> }
> ```
> but the data check doesn't:
> ```cpp
> if (!data)
>   return result;
> ```
> Consider making these consistent per Chromium style (both braced or both unbraced for single-statement bodies). Same applies to `ReadAsciiTextInternal`.

### Comment 2 — `clipboard_win_unittest.cc`, new tests (around line 320)

> Nice test coverage! Consider adding a test that explicitly enables/disables `kNonBlockingOsClipboardReads` via `base::test::ScopedFeatureList` to exercise both the sync-fallback and async-offload code paths within `ReadAsync`. This would ensure the branching logic is covered regardless of the default feature state in tests.

### Comment 3 — General (CL-level comment)

> LGTM with minor comments. The implementation cleanly follows the existing `ReadHTML`/`ReadFilenames` async pattern. The static `*Internal` extraction is well-done and makes the sync/async split very clear. Thanks for the incremental approach — this makes review and rollback straightforward.
