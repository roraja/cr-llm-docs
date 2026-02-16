# CL Review Summary — CL 7579673

**CL:** [Async Clipboard API: Convert sync mojom IPC to async](https://chromium-review.googlesource.com/c/chromium/src/+/7579673)
**Author:** Rohan Raja (roraja@microsoft.com)
**Status:** NEW | **Branch:** main | **Files Changed:** 15 | **Lines:** +443/−25

---

## 1. Executive Summary

This CL converts 8 `ClipboardHost` mojom read methods from synchronous (`[Sync]`) to asynchronous IPC, addressing a long-standing issue (Bug 474131935) where the renderer main thread blocks during clipboard reads via the Async Clipboard API. To maintain backward compatibility for synchronous callers (DataTransfer/paste events), `[Sync] SyncRead*` legacy variants are introduced. The CL also converts two key Blink-side consumers (`ClipboardPngReader::Read()` and `ClipboardPromise::HandleReadTextWithPermission()`) to use async callbacks.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | Clear separation between async and sync paths via naming convention (`Read*` vs `SyncRead*`). Comments in mojom explain the purpose of each variant. |
| Maintainability | 4 | Dual methods increase surface area but each `SyncRead*` simply delegates to its `Read*` counterpart, minimizing divergence risk. |
| Extensibility | 3 | Adding new read methods now requires adding both variants. Could benefit from a macro or template pattern to reduce boilerplate in the future. |
| Consistency | 4 | Consistent naming pattern (`SyncRead*`) applied uniformly across all 8 methods. All mock implementations follow the same delegation pattern. |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    Renderer Process                      │
│                                                          │
│  ┌─────────────────┐    ┌──────────────────────────┐     │
│  │  DataTransfer /  │    │  Async Clipboard API     │     │
│  │  Paste Events    │    │  (navigator.clipboard.*)  │     │
│  └────────┬─────────┘    └────────────┬─────────────┘     │
│           │                           │                   │
│           │ [Sync IPC]                │ [Async IPC]       │
│           ▼                           ▼                   │
│  ┌────────────────┐      ┌────────────────────────┐      │
│  │ SystemClipboard │      │ ClipboardPromise /      │      │
│  │ SyncRead*()     │      │ ClipboardPngReader      │      │
│  │ (blocking)      │      │ Read*() + callback      │      │
│  └────────┬────────┘      └────────────┬───────────┘      │
│           │                            │                  │
└───────────┼────────────────────────────┼──────────────────┘
            │                            │
     ┌──────▼──────────────────┐  ┌──────▼──────────────────┐
     │  SyncRead*(buffer)      │  │  Read*(buffer)          │
     │  [Sync] mojom IPC       │  │  async mojom IPC        │
     │  (blocks renderer)      │  │  (non-blocking)         │
     └──────────┬──────────────┘  └──────────┬──────────────┘
                │                             │
     ┌──────────▼─────────────────────────────▼─────────────┐
     │                 Browser Process                      │
     │           ClipboardHostImpl                          │
     │  SyncRead*() → delegates to Read*()                  │
     │  Read*()   → performs actual clipboard read           │
     └──────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | All sync callers correctly migrated to `SyncRead*` variants. Async callers correctly use callbacks. One concern about missing async `ReadPlainText` in the diff (see Key Findings). |
| Efficiency | 5 | Eliminates blocking IPC for the Async Clipboard API path—a significant performance improvement, especially for delayed clipboard writes (e.g., Excel). |
| Readability | 4 | Clean callback patterns using `BindOnce` + `WrapPersistent`. `SyncRead*` delegation is straightforward. |
| Test Coverage | 4 | Good tests for `SyncReadText`, `SyncReadHtml`, `SyncReadPng`, `SyncReadAvailableCustomAndStandardFormats`, and async `ReadPng`. Tests cover normal, empty, and unbound-host cases. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

- **None identified.** The CL passes CQ (dry run completed successfully).

### Major Issues (Should Fix)

1. **Missing async `ReadPlainText` overload in the diff.** The `clipboard_promise.cc` calls `ReadPlainText(buffer, callback)` but the async overload for `SystemClipboard::ReadPlainText` is not shown in the diff. This appears to have been added in a prior CL or is already present — confirmed it exists in the current codebase at `system_clipboard.cc:129`. However, the async `ReadPlainText` delegates to `clipboard_->ReadText()` (the now-async variant), meaning the `ReadText` callback for the **sync** `ReadPlainText(buffer)` (line 118) calls `clipboard_->SyncReadText()` while the async overload calls `clipboard_->ReadText()`. This is correct but warrants a comment in `system_clipboard.cc` explaining why the two overloads route to different mojom methods.

2. **`ReadAvailableCustomAndStandardFormats` in `clipboard_promise.cc` is already async-capable** (line 367), calling with a `BindOnce` callback. Since `[Sync]` was removed from `ReadAvailableCustomAndStandardFormats` in the mojom, this is now truly async. However, the `SystemClipboard` wrapper for this method should be verified to ensure it doesn't have a synchronous call path that expects blocking behavior. The existing code appears to already use async patterns here, so this is fine, but should be double-checked.

### Minor Issues (Nice to Fix)

1. **Boilerplate duplication in mock implementations.** Three separate mock classes (`MockClipboardHost` in `content/test/`, `mock_clipboard_host` in `blink/renderer/core/testing/`, and `ClipboardHostImpl`) all have identical `SyncRead*` delegation stubs. Consider a macro or mixin to reduce 43-line copy-paste blocks.

2. **Comment style inconsistency.** Some mojom comments say "Async version — used by the Async Clipboard API" while the original `ReadSvg` (line 147) has no such annotation, even though it was already async. Minor but could be cleaned up for consistency.

3. **Two `Change-Id` lines in the commit message** (lines `Ie216c41c...` and `Ifab4848b...`). The first appears to be a stale `Change-Id` from an earlier iteration. Only one `Change-Id` should be present; Gerrit uses the last one, but the duplicate is confusing.

### Suggestions (Optional)

1. **Consider converting more Async Clipboard API paths to async.** Currently only `ReadPng` and `ReadText` have been converted on the Blink consumer side. Methods like `ReadHtml`, `ReadRtf`, `ReadFiles`, `ReadAvailableTypes` in `ClipboardPromise` and `ClipboardReader` subclasses could also benefit from async conversion in follow-up CLs.

2. **Add a TODO comment** for eventually removing `SyncRead*` variants once all callers are migrated to async, to track technical debt.

3. **Consider using `base::test::TestFuture` consistently** in the new unit tests for the async paths, as done in `Bug474131935_SyncReadTextReturnsCorrectData`. Some tests use raw callbacks while others use `TestFuture`.

---

## 5. Test Coverage Analysis

### What Tests Exist
- **`clipboard_host_impl_unittest.cc`**: 4 new tests covering `SyncReadText`, `SyncReadHtml`, `SyncReadPng`, and `SyncReadAvailableCustomAndStandardFormats`. Tests verify data integrity by comparing sync vs async results.
- **`system_clipboard_test.cc`**: 3 new tests covering async `ReadPng` (normal case, unbound host, empty clipboard). Tests verify callback invocation, data size, and PNG decode correctness.
- Existing tests updated to use `SyncRead*` variants where they previously used `Read*` (2 test modifications in `clipboard_host_impl_unittest.cc`).

### What Tests Are Missing
- **No tests for async `ReadPlainText` overload** in `system_clipboard_test.cc`.
- **No tests for `ClipboardPromise::OnReadTextComplete`** callback being invoked correctly, including the early-return when `GetExecutionContext()` is null.
- **No tests for `ClipboardPngReader::OnRead`** callback path.
- **No tests for `SyncReadRtf`, `SyncReadFiles`, `SyncReadDataTransferCustomData`** equivalence — only 4 of 8 `SyncRead*` methods are tested.

### Recommended Additional Tests
1. Add `system_clipboard_test.cc` test for async `ReadPlainText(buffer, callback)`.
2. Add tests for `ClipboardPromise::OnReadTextComplete` with a null execution context (verifies no crash/UAF).
3. Add equivalence tests for the remaining 4 `SyncRead*` methods (`SyncReadRtf`, `SyncReadFiles`, `SyncReadDataTransferCustomData`, `SyncReadAvailableTypes`).
4. Add a web platform test or integration test verifying that `navigator.clipboard.readText()` resolves correctly end-to-end with the async path.

---

## 6. Security Considerations

- **No new security concerns introduced.** The `SyncRead*` methods delegate directly to the existing `Read*` methods, which already perform all security checks (permission validation, sandboxing, etc.) in `ClipboardHostImpl`.
- **The async path maintains the same security model.** Permission checks in `ClipboardPromise` occur before the async clipboard read is initiated, so the permission check → read ordering is preserved.
- **`ReadFiles` security notes** (must only be used with user activation) remain intact since the sync path (`SyncReadFiles`) delegates to the same `ReadFiles` implementation with the same guards.
- **Recommendation:** Verify that the async callback paths cannot be exploited via use-after-free if the execution context is destroyed between initiating the async read and receiving the callback. The `GetExecutionContext()` null check in `OnReadTextComplete` is correct, but similar guards should be present in all callback paths (they appear to be via `WrapPersistent`).

---

## 7. Performance Considerations

- **Significant performance improvement.** Removing `[Sync]` from 8 mojom methods eliminates renderer main-thread blocking during clipboard reads via the Async Clipboard API. This directly addresses the reported issue where delayed clipboard writes (e.g., from Excel) could freeze the browser for seconds.
- **No performance regression for sync callers.** `SyncRead*` variants retain `[Sync]` behavior, so DataTransfer/paste event paths are unaffected.
- **Benchmarking recommendations:**
  1. Measure `navigator.clipboard.read()` and `navigator.clipboard.readText()` latency before and after this CL, particularly with applications that use delayed clipboard rendering (Excel, LibreOffice).
  2. Verify no regression in paste event (`Ctrl+V`) performance by running clipboard-related Telemetry benchmarks.
  3. Monitor `Blink.Clipboard.Read.NumberOfFormats` UMA histogram for any changes in behavior.

---

## 8. Final Recommendation

**Verdict**: APPROVED_WITH_COMMENTS

**Rationale:**
This CL is a well-structured, focused change that solves a real user-facing performance problem (renderer freezing during async clipboard reads). The approach of introducing `SyncRead*` legacy variants while making the original `Read*` methods async is a clean, incremental strategy that avoids breaking existing synchronous callers. The CL passes CQ, includes meaningful tests, and follows Chromium conventions. The issues identified are minor (boilerplate duplication, commit message cleanup, additional test coverage) and don't block landing.

**Action Items for Author:**
1. Remove the duplicate `Change-Id` line from the commit message.
2. Add a brief comment in `system_clipboard.cc` explaining that the sync `ReadPlainText(buffer)` routes to `SyncReadText` while the async overload routes to `ReadText`.
3. Consider adding tests for async `ReadPlainText` overload and the remaining untested `SyncRead*` methods in a follow-up CL.
4. Add a TODO comment for eventually removing `SyncRead*` variants once all callers migrate to async.

---

## 9. Comments for Gerrit

### Comment 1 — `third_party/blink/public/mojom/clipboard/clipboard.mojom` (general)

> **Nit:** The dual-method approach (Read* async + SyncRead* sync) is clean and well-commented. Consider adding a file-level TODO to track eventual removal of SyncRead* variants once all callers are migrated to async. This helps track the intended end-state.

### Comment 2 — `third_party/blink/renderer/core/clipboard/system_clipboard.cc` (line ~118–136)

> **Suggestion:** The sync `ReadPlainText(buffer)` calls `clipboard_->SyncReadText()` while the async `ReadPlainText(buffer, callback)` calls `clipboard_->ReadText()`. Both are correct, but a brief comment explaining this routing would help future readers understand why two overloads go to different mojom methods.

### Comment 3 — `content/browser/renderer_host/clipboard_host_impl_unittest.cc` (new tests)

> **Good tests!** The SyncRead* equivalence tests are valuable. Consider adding similar tests for `SyncReadRtf`, `SyncReadFiles`, `SyncReadDataTransferCustomData`, and `SyncReadAvailableTypes` for completeness, either here or in a follow-up.

### Comment 4 — `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc` (line ~490)

> **LGTM.** The `OnReadTextComplete` callback correctly checks `GetExecutionContext()` before resolving, which handles the case where the context is destroyed during the async read. Clean pattern.

### Comment 5 — `third_party/blink/renderer/modules/clipboard/clipboard_reader.cc` (line ~45–60)

> **LGTM.** The refactor of `ClipboardPngReader::Read()` to use an async callback + `OnRead()` is clean. The `private:` placement is correct since `OnRead` is an internal callback.

### Comment 6 — Commit Message

> **Nit:** There are two `Change-Id` lines in the commit message. Please remove the stale `Change-Id: Ie216c41c3fe8619cfa29119aa4b6d6195fa0c3c6` line.
