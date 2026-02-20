# CL 7578053: [clipboard][Windows] Make ReadPng non-blocking and refactor internals

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7578053
**Author:** Hewro Hewei (ihewro@chromium.org)
**Status:** NEW (awaiting review)
**Files Changed:** 3 files, +78/−15 lines

---

## 1. Executive Summary

This CL makes `ClipboardWin::ReadPng` non-blocking by gating it behind the `kNonBlockingOsClipboardReads` feature flag and routing it through the existing `ReadAsync` infrastructure. It refactors the PNG reading logic by extracting `ReadPngInternal` (a static method that consolidates PNG-then-bitmap-fallback logic) and `ReadPngTypeDataInternal` / `ReadBitmapInternal` (now taking an explicit `HWND` parameter), and also generalizes the `ReadAsync` template to support differing task-return and reply-argument types. Two new unit tests verify async round-trip behavior for PNG data.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | Clear separation of async gating vs. internal logic. The dual template parameters on `ReadAsync` add slight cognitive load but are well-documented. |
| Maintainability | 4 | Follows the established pattern used by `ReadText`, `ReadAsciiText`, `ReadHTML`, `ReadFilenames`, and `ReadAvailableTypes`. Easy to reason about future changes. |
| Extensibility | 5 | The generalized `ReadAsync<TaskReturnType, ReplyArgType>` template makes it trivial to async-ify additional clipboard read methods in the future. |
| Consistency | 4 | Consistent with how other Read* methods were migrated. Minor inconsistency: `ReadPng` still has an inline feature-flag check + synchronous fallback path, while other methods (e.g. `ReadText`) delegate entirely to `ReadAsync` which handles both paths internally. |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    ClipboardWin::ReadPng                 │
│                                                         │
│  ┌─ kNonBlockingOsClipboardReads ENABLED ─────────────┐ │
│  │  ReadAsync(ReadPngInternal, callback)               │ │
│  │    ├─ worker_task_runner_->PostTaskAndReplyWithResult│ │
│  │    │    └─ ReadPngInternal(buffer, data_dst, nullptr)│ │
│  │    │         ├─ ReadPngTypeDataInternal(buffer, null)│ │
│  │    │         └─ ReadBitmapInternal(buffer, null)     │ │
│  │    │              └─ EncodeBitmapToPng (synchronous) │ │
│  │    └─ reply on caller sequence with result           │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌─ kNonBlockingOsClipboardReads DISABLED ────────────┐ │
│  │  Synchronous path (legacy):                         │ │
│  │  ReadPngTypeDataInternal(buffer, GetClipboardWindow) │ │
│  │  └─ if empty: ReadBitmapInternal → EncodeBitmapToPng│ │
│  │     (bitmap encoding posted to ThreadPool)           │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | Logic is correct. See "Key Findings" for a potential blocking concern with `EncodeBitmapToPng` in the async path. |
| Efficiency | 3 | In the async path, `EncodeBitmapToPng` runs synchronously inside `ReadPngInternal` on the worker thread—this is acceptable since the worker thread is already off the UI thread, but it's a change from the legacy path where encoding was posted as a separate ThreadPool task. |
| Readability | 4 | Clean, well-structured code. The `ReadPngInternal` consolidation makes the intent clear. |
| Test Coverage | 3 | Two new tests added (`ReadPngAsyncReturnsWrittenData`, `ReadPngAsyncEmptyClipboard`). Missing: explicit test with feature flag toggling, negative/edge cases (see below). |

---

## 4. Key Findings

### Critical Issues (Must Fix)

- **None identified.** The CL is well-structured and follows established patterns.

### Major Issues (Should Fix)

1. **`EncodeBitmapToPng` blocking behavior difference between async and sync paths.**
   - In the **legacy (sync) path** (lines 743–747): when PNG data is unavailable and bitmap fallback is used, `EncodeBitmapToPng` is posted to a ThreadPool task, keeping the calling thread unblocked during encoding.
   - In the **new async path** via `ReadPngInternal` (lines 1107–1110): `EncodeBitmapToPng` runs **synchronously** on the worker thread. While this doesn't block the UI thread (since `ReadPngInternal` runs on `worker_task_runner_`), it means the single sequenced worker thread is blocked during potentially expensive PNG encoding. This could delay other queued clipboard reads.
   - **Recommendation:** Consider whether this is acceptable given that the worker runner is `SequencedTaskRunner`. If multiple clipboard reads are queued, a large bitmap encode could stall them. Document the design decision or consider posting the encode separately.

2. **Redundant feature-flag check in `ReadPng`.**
   - `ReadPng` (line 728) checks `kNonBlockingOsClipboardReads` before calling `ReadAsync`, but `ReadAsync` (line 1081) also checks the same feature flag internally to decide between sync and async execution. Other methods like `ReadText` (line 426) simply call `ReadAsync` without the outer guard, relying on `ReadAsync`'s internal check.
   - The outer check + early return in `ReadPng` means the synchronous fallback code below it (lines 733–747) is dead code when the feature is enabled. This creates two separate code paths that do similar things—`ReadPngInternal` (async path) and the inline code (sync path)—which is a maintenance burden.
   - **Recommendation:** Remove the outer feature-flag check and the inline sync fallback in `ReadPng`, and simply call `ReadAsync(ReadPngInternal, callback)` unconditionally, matching the pattern of `ReadText`, `ReadAsciiText`, etc. This would also eliminate the duplicated PNG-reading logic.

### Minor Issues (Nice to Fix)

1. **`RecordRead(ClipboardFormatMetric::kPng)` is called in `ReadPngInternal` (line 1099) for the async path, and separately at line 733 for the sync path.** If the redundant sync path is removed per the recommendation above, this becomes moot. Otherwise, verify that metrics recording is consistent and happens exactly once per read regardless of path.

2. **`ReadPngInternal` calls `EncodeBitmapToPng` (not `EncodeBitmapToPngAcceptJank`)** (line 1109). The non-`AcceptJank` variant is correct here since it runs off the UI thread, but this should be documented to prevent future confusion with the `AcceptJank` variant used elsewhere (line 1024).

### Suggestions (Optional)

1. **Consider adding a comment in `ReadAsync`** explaining why the template now has two type parameters (`TaskReturnType`, `ReplyArgType`) instead of one—specifically to support cases where the callback takes a `const&` to the result (like `ReadPngCallback` which takes `const std::vector<uint8_t>&`).

2. **Consider a `DCHECK_EQ(buffer, ClipboardBuffer::kCopyPaste)` in `ReadPngInternal`** to match the assertion pattern in `ReadPngTypeDataInternal` and `ReadBitmapInternal`.

---

## 5. Test Coverage Analysis

### Existing Tests
- `ReadPngAsyncReturnsWrittenData` (line 404): Writes a bitmap, reads as PNG via async path, verifies non-empty result.
- `ReadPngAsyncEmptyClipboard` (line 419): Clears clipboard, reads PNG via async path, verifies empty result.
- `InvalidBitmapDoesNotCrash` (line 198): Tests that invalid bitmap data doesn't crash during `ReadPng`.
- `NoDataChangedNotificationOnRead` (line 95): Existing test that covers `ReadPng` in the notification-check path.

### Missing Tests
- **Feature flag toggling test:** No test explicitly enables/disables `kNonBlockingOsClipboardReads` to verify both paths work correctly. Other read methods also lack this, so this may be a broader gap, but since this CL introduces a new feature-gated path it would be valuable.
- **PNG format prioritization test:** No test writes both PNG and bitmap data to the clipboard and verifies that the PNG path is preferred over bitmap fallback.
- **Large bitmap performance test:** No test validates behavior with large bitmaps where `EncodeBitmapToPng` might take significant time.

### Recommended Additional Tests
1. A parameterized test (or two separate tests) that explicitly toggles `kNonBlockingOsClipboardReads` on/off via `base::test::ScopedFeatureList` and verifies `ReadPng` produces correct results in both modes.
2. A test that writes PNG-formatted data directly (not via bitmap) and confirms the PNG-first path returns it without bitmap fallback.

---

## 6. Security Considerations

- **No new security concerns.** The CL does not introduce new IPC, parsing, or untrusted data handling beyond what already exists.
- Clipboard data is read from the system clipboard using existing Win32 APIs (`GetClipboardData`, `GlobalLock`/`GlobalUnlock`) with established size limits (`GetClipboardDataWithLimit`).
- The `owner_window` parameter is correctly set to `nullptr` in the async path (since worker threads don't own windows), which is the documented/intended behavior for non-blocking reads.

---

## 7. Performance Considerations

- **Positive impact:** Moving `ReadPng` to the async path removes a potential UI-thread stall when reading clipboard bitmap data and encoding it to PNG. This is the primary goal of the CL and aligns with the `kNonBlockingOsClipboardReads` feature's purpose.
- **Potential concern:** As noted in Major Issue #1, `EncodeBitmapToPng` now runs synchronously on the sequenced worker thread in the async path. For very large bitmaps, this could block other queued clipboard reads. The legacy path mitigated this by posting the encode to an independent ThreadPool task.
- **Benchmarking recommendation:** Profile `ReadPng` with large bitmaps (e.g., 4K screenshots) to measure worker-thread stall duration and determine if it impacts perceived clipboard read latency for other concurrent reads.

---

## 8. Final Recommendation

**Verdict**: APPROVED_WITH_COMMENTS

**Rationale:**
The CL correctly implements the intended behavior of making `ReadPng` non-blocking, following the well-established pattern used by other clipboard read methods. The code is clean, well-structured, and includes appropriate unit tests. The two major issues identified—the redundant dual code paths and the blocking `EncodeBitmapToPng` on the worker thread—are not correctness bugs but rather maintainability and potential performance concerns that should be addressed before or shortly after landing.

**Action Items for Author:**
1. **Simplify `ReadPng` to unconditionally delegate to `ReadAsync(ReadPngInternal, callback)`**, removing the outer feature-flag check and the inline synchronous fallback. This eliminates duplicated logic and matches the pattern of `ReadText`, `ReadAsciiText`, `ReadAvailableTypes`, `ReadHTML`, and `ReadFilenames`.
2. **Consider the performance implication** of `EncodeBitmapToPng` running synchronously on the sequenced worker thread. Either document that this is acceptable (since the worker thread is already off UI) or consider posting the encode step separately to avoid blocking other queued reads.
3. **Add a test** with explicit `ScopedFeatureList` toggling of `kNonBlockingOsClipboardReads` to cover both paths.

---

## 9. Comments for Gerrit

### Comment 1: `ui/base/clipboard/clipboard_win.cc` — ReadPng method (line 728)

> **[Suggestion]** The outer `kNonBlockingOsClipboardReads` check + early return creates a dual code path: `ReadPngInternal` (used when the feature is enabled) and the inline synchronous logic below (used when disabled). Other Read* methods (e.g., `ReadText`, `ReadAsciiText`, `ReadFilenames`) simply call `ReadAsync(...)` unconditionally and let `ReadAsync` handle both the sync/async decision internally.
>
> Consider simplifying to:
> ```cpp
> void ClipboardWin::ReadPng(ClipboardBuffer buffer,
>                            const std::optional<DataTransferEndpoint>& data_dst,
>                            ReadPngCallback callback) const {
>   ReadAsync(base::BindOnce(&ClipboardWin::ReadPngInternal, buffer, data_dst),
>             std::move(callback));
> }
> ```
> This removes ~15 lines of duplicated logic and makes `ReadPng` consistent with the other methods.

### Comment 2: `ui/base/clipboard/clipboard_win.cc` — ReadPngInternal (line 1107-1109)

> **[Nit/Discussion]** In the async path, `EncodeBitmapToPng` runs synchronously on the `worker_task_runner_`. In the legacy sync path (line 744-747), it was posted to a separate ThreadPool task to avoid blocking. Since `worker_task_runner_` is a `SequencedTaskRunner`, a large bitmap encode here could delay other queued clipboard reads. Is this the intended tradeoff? If so, consider adding a brief comment explaining the choice.

### Comment 3: `ui/base/clipboard/clipboard_win_unittest.cc` — New tests (lines 404-428)

> **[Suggestion]** These tests exercise `ReadPng` via the async API but don't explicitly toggle the `kNonBlockingOsClipboardReads` feature flag. Consider adding `base::test::ScopedFeatureList` to ensure the async path is explicitly exercised, similar to how you might want to verify the sync fallback still works when the feature is disabled.

### Comment 4: `ui/base/clipboard/clipboard_win.h` — ReadAsync template (line 140)

> **[Nit]** Consider adding a brief comment on *why* `ReadAsync` now has two template parameters (`TaskReturnType` and `ReplyArgType`), e.g., to support callbacks that take `const T&` rather than `T` (such as `ReadPngCallback`).
