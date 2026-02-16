# Code Review: 474131935

## Review Summary
The fix correctly converts 8 `ClipboardHost` mojom read methods from `[Sync]` to async, adds `[Sync] SyncRead*` legacy variants for backward-compatible synchronous callers (DataTransfer/paste events), updates `SystemClipboard` sync callers to route through `SyncRead*`, and converts `ClipboardPngReader::Read()` and `ClipboardPromise::HandleReadTextWithPermission()` to use async callbacks. The implementation is minimal, follows existing async patterns already established by `ClipboardTextReader`/`ClipboardHtmlReader`/`ClipboardSvgReader`, and correctly handles edge cases (unbound host, destroyed execution context). No critical issues found.

## HLD: Top 5 Approaches

### 1. Convert Mojom Read Methods to Async (⭐ RECOMMENDED — Implemented)
Remove `[Sync]` from `ClipboardHost` read methods (`ReadText`, `ReadHtml`, `ReadPng`, etc.) in `clipboard.mojom` for the Async Clipboard API path. Add `[Sync] SyncRead*` legacy variants for DataTransfer/paste callers. Update `SystemClipboard` sync callers to use `SyncRead*` and async callers to use the now-async `Read*` methods. This is the minimal, surgical approach — it only changes the IPC layer and the two renderer-side callsites that are on the Async Clipboard API path. The browser-side logic is completely shared between sync and async.

**Pros**: Minimal code duplication, no browser-side changes to clipboard logic, backward-compatible, no risk of reentrancy.
**Cons**: Browser UI thread still blocks on OS clipboard (future Phase 2 work). Adds 8 `SyncRead*` methods to the mojom interface.

### 2. Move OS Clipboard Access to Background Thread in Browser Process
Keep `[Sync]` on all mojom methods but move `ui::Clipboard` reads in `ClipboardHostImpl` to `base::ThreadPool`. The renderer still blocks (sync IPC wait), but the browser UI thread is freed.

**Pros**: Unblocks browser UI thread from OS clipboard delays.
**Cons**: Does NOT fix the renderer freeze (the core bug). `ui::Clipboard` has thread affinity (`GetForCurrentThread()`, Windows `::OpenClipboard()` requires the same thread that opened it). Major refactoring needed with uncertain safety guarantees.

### 3. Add Timeout to Sync Mojom IPC Calls
Add a timeout parameter to the sync calls so the renderer unblocks after N seconds. Return empty data on timeout.

**Pros**: Simple to implement, no new methods needed.
**Cons**: Does NOT fix root cause — users still freeze up to the timeout. Clipboard data may be silently lost if timeout is too short. Hard to choose a correct timeout value across platforms and clipboard sources.

### 4. Introduce a New Async Mojom Interface (`AsyncClipboardHost`)
Create a separate mojom interface with only async methods, bound via a separate mojo remote. The Async Clipboard API path would use `AsyncClipboardHost`, while DataTransfer keeps the existing `ClipboardHost`.

**Pros**: Clean separation of concerns, no naming ambiguity.
**Cons**: Significant code duplication on the browser side (two implementations), over-engineered for this problem scope. Both interfaces would need the same security checks, clipboard locking, and data sanitization.

### 5. Use Mojo Associated Interface with `[NoInterrupt]` and Async Pattern
Remove `[Sync]` from all methods and use `base::RunLoop` in the renderer for legacy synchronous callers that need to block until the response arrives.

**Pros**: Only async methods in the interface.
**Cons**: Extremely dangerous — nested `RunLoop` in the renderer causes reentrancy issues (other mojo messages, JavaScript timers, etc. can fire during the nested loop). Explicitly discouraged in Chromium codebase. Can lead to hard-to-debug crashes.

## Checklist Results

### Correctness ✅
- [x] Fix addresses root cause: The root cause is that `[Sync]` mojom calls block the renderer main thread during `navigator.clipboard.read()` / `navigator.clipboard.readText()`. The fix removes `[Sync]` from the mojom methods used by the Async Clipboard API, making them truly asynchronous at the IPC layer.
- [x] No logic errors: `SyncRead*` methods delegate directly to `Read*` on the browser side — identical behavior for sync callers.
- [x] Edge cases handled: Unbound clipboard host returns empty data via callback. Destroyed execution context is checked in `OnReadTextComplete`. Empty clipboard data produces an empty `BigBuffer`.
- [x] Error handling correct: The async `ReadPng` overload invokes the callback with an empty `BigBuffer()` when the host is unbound, matching the pattern of the sync version.

### Crash Safety ✅
- [x] No null dereferences: `GetExecutionContext()` null check in `OnReadTextComplete` prevents use of a destroyed context. `clipboard_.is_bound()` checks prevent calling into an unbound mojo remote.
- [x] No use-after-free: `WrapPersistent(this)` prevents garbage collection of `ClipboardPromise` and `ClipboardPngReader` while the async callback is pending, matching existing patterns (e.g., `ClipboardTextReader`, `ClipboardSvgReader`).
- [x] No invalid iterators: No iterator usage introduced.
- [x] Bounds checked: No array access introduced.

### Memory Safety ✅
- [x] Smart pointers used correctly: `WrapPersistent` for GC-traced Blink objects, `std::move(callback)` for one-shot callbacks — both correct patterns.
- [x] No memory leaks: Callbacks are guaranteed to be invoked (mojo guarantees callback invocation or connection error). `WrapPersistent` prevents premature GC but allows collection after callback runs.
- [x] Object lifetimes correct: `ClipboardPromise` and `ClipboardPngReader` are prevented from GC during async operations via `WrapPersistent`, consistent with existing readers.
- [x] Raw pointers safe: No new raw pointer usage.

### Thread Safety ✅
- [x] Thread annotations present: `DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_)` is present in `OnReadTextComplete`, `ClipboardPngReader::Read()`, and `ClipboardPngReader::OnRead()`.
- [x] No deadlocks: No locking introduced. Async callbacks are delivered on the same sequence.
- [x] No race conditions: All new code runs on the renderer main thread sequence. The `SyncRead*` browser-side implementations also run on the IO thread (same thread as existing `Read*`), so no new threading concerns.
- [x] Cross-thread calls safe: No cross-thread calls introduced. All async callbacks are invoked on the originating sequence by mojo.

### DCHECK Safety ✅
- [x] DCHECKs valid: `DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_)` is always valid — it verifies sequence affinity, which is guaranteed by mojo callback delivery.
- [x] Good error messages: Sequence checker provides clear crash messages in debug builds.
- [x] No DCHECK on user input: No DCHECKs added that depend on user/external input.

### Code Style ✅
- [x] Follows style guide: Method naming follows Chromium convention (`SyncRead*` prefix for sync variants, `OnRead*Complete` for callbacks). Consistent with existing codebase patterns.
- [x] Formatted with git cl format: `git ms format` ran with no changes produced.
- [x] Good naming: `SyncReadText`, `SyncReadHtml`, etc. clearly indicate the synchronous nature. `OnReadTextComplete` follows the `On*` callback naming convention.
- [x] Helpful comments: Comments explain "why" (e.g., "// Legacy [Sync] variants for DataTransfer/paste event callers", "// Async overload for Async Clipboard API path", "// Use async overload to avoid blocking the renderer main thread").

### Tests ✅
- [x] Bug scenario covered: 7 TDD tests directly verify the bug fix (3 blink_unittests, 4 content_unittests).
- [x] Edge cases covered: Tests cover async PNG read with data, async PNG read with empty clipboard, async PNG read with unbound host, sync variants returning correct data.
- [x] Descriptive names: `Bug474131935_ReadPngAsync`, `Bug474131935_ReadPngAsyncWithUnboundHost`, `Bug474131935_SyncReadTextReturnsCorrectData`, etc.
- [x] Not flaky: All tests pass consistently. Pre-existing flaky web test (`clipboard-read-unsupported-types-removal.html`) is documented as unrelated.

## Detailed Review

### File: `third_party/blink/public/mojom/clipboard/clipboard.mojom`

**Lines 116-200 (mojom changes)**: ✅ Correct
- Removing `[Sync]` from 8 read methods makes them truly asynchronous at the IPC layer. The renderer no longer sends a blocking sync IPC message.
- New `[Sync] SyncRead*` variants have identical signatures (same parameters, same return values) to their async counterparts, ensuring the browser-side implementation can be trivially shared.
- `ReadSvg` was already async (no `[Sync]` annotation) and was not modified — correct.
- `GetSequenceNumber` and `IsFormatAvailable` remain `[Sync]` — correct, these are lightweight metadata queries.

**Note**: The mojom comments clearly explain async vs sync variants, which is good for future maintainers.

### File: `third_party/blink/renderer/core/clipboard/system_clipboard.cc`

**Lines 101-104 (SyncReadAvailableTypes)**: ✅ Correct
- Sync `ReadAvailableTypes()` now calls `clipboard_->SyncReadAvailableTypes()`. This is a sync caller (DataTransfer), so it correctly uses the sync variant.

**Lines 118-136 (SyncReadText / async ReadPlainText)**: ✅ Correct
- Sync `ReadPlainText(buffer)` now calls `clipboard_->SyncReadText()`.
- The async `ReadPlainText(buffer, callback)` overload already existed and now routes through the truly-async `clipboard_->ReadText()`. No change needed here — the method was already using the right mojom method name.

**Lines 170-175 (SyncReadHtml)**: ✅ Correct
- Sync `ReadHTML()` now calls `clipboard_->SyncReadHtml()`.

**Lines 238 (SyncReadRtf)**: ✅ Correct

**Lines 259 (SyncReadPng)**: ✅ Correct

**Lines 267-275 (Async ReadPng)**: ✅ Correct
- New async `ReadPng(buffer, callback)` overload properly validates buffer type and host binding before dispatching.
- Returns empty `BigBuffer()` via callback on error, matching the pattern of other async methods and the sync version.

**Lines 345 (SyncReadFiles)**: ✅ Correct

**Lines 362-363 (SyncReadDataTransferCustomData)**: ✅ Correct

### File: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`

**Lines 483-496 (HandleReadTextWithPermission → OnReadTextComplete)**: ✅ Correct
- Replaced synchronous `ReadPlainText(buffer)` with async `ReadPlainText(buffer, callback)`.
- `OnReadTextComplete` has proper sequence checker, execution context null check, and resolves the promise — identical safety pattern to other async callbacks in this class (e.g., `OnPlatformPermissionResultForReadText`).
- `WrapPersistent(this)` prevents GC during async operation.

**Lines 514-517 (OnPlatformPermissionResultForReadText)**: ✅ Correct
- macOS path also updated to use async `ReadPlainText`. Same callback reuse (`OnReadTextComplete`) — clean and DRY.

### File: `third_party/blink/renderer/modules/clipboard/clipboard_reader.cc`

**Lines 45-60 (ClipboardPngReader async)**: ✅ Correct
- `Read()` now dispatches async `ReadPng(buffer, callback)` instead of blocking.
- `OnRead(mojo_base::BigBuffer data)` creates the blob from async data — identical logic to what was previously inline.
- `WrapPersistent(this)` prevents GC.
- `DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_)` in `OnRead`.
- Pattern is identical to `ClipboardTextReader::Read()` / `ClipboardTextReader::OnRead()` (lines 71-83) — good consistency.

### File: `content/browser/renderer_host/clipboard_host_impl.cc`

**Lines 969-1012 (SyncRead* implementations)**: ✅ Correct
- All 8 `SyncRead*` methods delegate directly to their corresponding `Read*` method with `std::move(callback)`.
- No code duplication of clipboard logic — the browser-side implementation is fully shared.
- No new security checks needed since `Read*` already has them.

### File: `content/test/mock_clipboard_host.cc` and `third_party/blink/renderer/core/testing/mock_clipboard_host.cc`

**SyncRead* mock implementations**: ✅ Correct
- Both mock classes add 8 `SyncRead*` methods that delegate to `Read*`, consistent with the production code pattern.
- Required because the mojom interface now declares these as pure virtual methods.

### File: `content/web_test/renderer/test_runner.cc`

**Line 2220 (SyncReadPng)**: ✅ Correct
- `ReadPng()` → `SyncReadPng()` — this is a sync caller (web test runner reads PNG synchronously) so it correctly uses the sync variant.

### File: `content/browser/renderer_host/clipboard_host_impl_unittest.cc`

**Lines 179, 200 (SyncReadAvailableTypes)**: ✅ Correct
- Existing tests that used sync out-parameter calling convention correctly updated to `SyncReadAvailableTypes`.

**Lines 1054-1068 (Bug474131935_SyncReadTextReturnsCorrectData)**: ✅ Correct
- Async `ReadText` now uses `TestFuture` callback pattern (correct for async mojom methods).
- Sync `SyncReadText` still uses out-parameter pattern (correct for `[Sync]` methods).
- Verifies both return the same data.

### File: `third_party/blink/renderer/core/clipboard/system_clipboard_test.cc`

**Lines 655-740 (Bug474131935 tests)**: ✅ Correct
- Three async `ReadPng` tests verify: data delivery, unbound host handling, and empty clipboard.
- Formatting changes (indentation of `BindOnce`) are consistent with `git cl format` output.

## Issues Found

### Critical (Must Fix)
None.

### Minor (Should Consider)
1. **Snapshot cache not populated in async ReadPng path**: The sync `ReadPng(buffer)` populates `snapshot_->SetPng(buffer, png)` for clipboard snapshot caching, but the async `ReadPng(buffer, callback)` does not. This is acceptable because the Async Clipboard API path doesn't use snapshots (snapshots are for DataTransfer sync callers), but it's worth noting for future reference.

2. **Other async read methods (ReadHtml, ReadRtf, etc.) not yet converted**: Only `ReadPng` and `ReadPlainText` have been converted to async on the renderer side. Other methods (`ReadHtml`, `ReadRtf`, `ReadFiles`, etc.) are now async at the mojom level but not yet used asynchronously by the Async Clipboard API callers. This is acknowledged as future work — the current fix covers the two primary Async Clipboard API read paths (`navigator.clipboard.read()` for images and `navigator.clipboard.readText()` for text).

### Informational
1. **Phase 2 (Browser UI thread)**: The browser UI thread still blocks on OS clipboard calls via `ui::Clipboard::GetForCurrentThread()`. For the delayed-clipboard-write scenario (Excel), the browser process will still block briefly while Excel generates the data. However, the renderer is now unblocked, which is the primary goal.
2. **`ReadSvg` was already async**: The `ReadSvg` mojom method never had `[Sync]` — it was already async. The `ClipboardSvgReader` already used the async pattern. No changes were needed.

## Linter Results

```
$ git cl presubmit --force
Not available in this environment (Edge project uses PR-based workflow, not Gerrit).
Workaround: Ran `git ms format` which produced no changes.
```

```
$ git ms format
No changes produced — code is already properly formatted.
```

**Status**: ✅ PASS (no formatting issues)

## Security Considerations
- **Input validation**: OK. No new input parsing introduced. All clipboard data is still sanitized through the existing `ClipboardHostImpl::Read*` methods which perform sanitization via `ui::Clipboard`.
- **IPC safety**: OK. The `SyncRead*` methods delegate to the exact same `Read*` implementations with the same security checks (renderer allowlist, clipboard access permissions, format sanitization). No privilege escalation possible — the sync and async paths share identical browser-side logic.
- **Memory safety**: OK. Mojo guarantees callback invocation (or connection error notification), preventing leaked callbacks. `WrapPersistent` prevents use-after-free in Blink GC objects. `std::move(callback)` ensures single-use semantics.
- **Denial of service**: The async pattern actually *improves* security posture — a malicious clipboard source can no longer freeze the renderer by delaying clipboard writes.

## Performance Considerations
- **Hot path affected**: Yes — `navigator.clipboard.read()` and `navigator.clipboard.readText()` are on the Async Clipboard API hot path. But the change *improves* performance by making these non-blocking.
- **New allocations**: The async path introduces one additional callback allocation per clipboard read (the `BindOnce` closure). This is negligible — a single heap allocation vs. blocking the entire renderer thread.
- **Async considerations**: The renderer main thread is no longer blocked during clipboard reads via the Async Clipboard API. This directly addresses the bug: when Excel (or any application) uses delayed clipboard writes, the renderer remains responsive while waiting for data.
- **No additional IPC round-trips**: The async mojom call is still a single IPC message + response, same as the sync call. The only difference is that the renderer doesn't block waiting for the response.

## Overall Verdict

| Category | Status |
|----------|--------|
| Correctness | ✅ |
| Safety | ✅ |
| Style | ✅ |
| Tests | ✅ |
| Security | ✅ |
| Performance | ✅ |

**Ready for CL**: ✅ YES

## Actions Before CL
- [x] All checks pass
- [x] Code formatted (no changes from `git ms format`)
- [x] Tests pass (7 TDD tests + 73 related tests + 9 web tests)
- [x] Security reviewed (IPC safety, memory safety, input validation)
- [x] Performance reviewed (async improvement, no regressions)
- [ ] Ready for upload
