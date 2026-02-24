# Code Review: CL 6978530 — [Clipboard] Implement on-demand reading in getType()

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Patch Set Reviewed:** 21 (latest)
**Files Changed:** 13 (+679/-121 lines)

---

## Review Summary

| Category    | Status | Notes                                                                                   |
|-------------|--------|-----------------------------------------------------------------------------------------|
| Correctness | ⚠️     | Potential null dereference, TOCTOU race in clipboard-change check, resolver lifetime edge cases |
| Style       | ⚠️     | Minor header inclusion issue, public members that could be protected/private             |
| Security    | ✅     | Clipboard change detection is a good security measure; sequence number check is sound    |
| Performance | ✅     | Core design achieves the goal of deferring reads; on-demand approach is well-motivated   |
| Testing     | ⚠️     | Good unit tests added, but missing edge-case coverage for concurrent getType rejection scenarios |

---

## Detailed Findings

### Issue #1: Potential null dereference of `GetSystemClipboard()` in `ResolveRead()`

**Severity**: Critical
**File**: `clipboard_promise.cc`
**Line**: 385 (in the diff, `ResolveRead()` method)
**Description**: When `ClipboardReadOnDemandEnabled()` is true, `ResolveRead()` calls `GetSystemClipboard()->SequenceNumber()` without null-checking the result of `GetSystemClipboard()`. While `GetExecutionContext()` is checked (via DCHECK) earlier in the method, `GetSystemClipboard()` can still return null if `GetLocalFrame()` returns null (e.g., if the frame was detached between the DCHECK and this line). The non-on-demand path also has this pattern but was pre-existing.

```cpp
clipboard_items = {MakeGarbageCollected<ClipboardItem>(
    item_mime_types_, GetSystemClipboard()->SequenceNumber(), // potential null deref
    GetExecutionContext(),
    /*sanitize_html_for_lazy_read=*/!will_read_unprocessed_html_,
    /*is_lazy_read=*/true)};
```

**Suggestion**: Add a null check for `GetSystemClipboard()` or pass `std::nullopt` for the sequence number when the system clipboard is unavailable:
```cpp
SystemClipboard* system_clipboard = GetSystemClipboard();
std::optional<absl::uint128> seq_num = system_clipboard
    ? std::make_optional(system_clipboard->SequenceNumber())
    : std::nullopt;
clipboard_items = {MakeGarbageCollected<ClipboardItem>(
    item_mime_types_, seq_num, ...)};
```

---

### Issue #2: TOCTOU race in `HasClipboardChangedSinceClipboardRead()`

**Severity**: Major
**File**: `clipboard_item.cc`
**Lines**: 250-263
**Description**: `HasClipboardChangedSinceClipboardRead()` is called from `ResolveFormatData()` *after* the clipboard data has already been read into a Blob. However, the clipboard could change between the moment the data was read and the moment the sequence number is checked. This means data that is actually stale could be delivered (if the clipboard changed and changed back to the same sequence number — though extremely unlikely with uint128), or valid data could be rejected (if the clipboard changed after the read completed but before the check).

More practically, the check is non-atomic: the sequence number is read via a Mojo call to the browser process. If multiple `getType()` calls are in flight, the sequence number could change between one read completing and the check occurring, causing inconsistent accept/reject behavior across formats.

**Suggestion**: Consider checking the sequence number *before* initiating the read (i.e., in `ReadRepresentationFromClipboardReader` or at the start of `getType()`), and/or documenting that the check is best-effort. The current placement is still valuable as a safety net but should be understood as a heuristic rather than a guarantee.

---

### Issue #3: `ClipboardReader` constructor dereferences `result_handler->GetExecutionContext()` without null check

**Severity**: Major
**File**: `clipboard_reader.cc`
**Lines**: 364-368 (constructor)
**Description**: The `ClipboardReader` constructor accesses `result_handler->GetExecutionContext()->GetTaskRunner(...)`. If `GetExecutionContext()` returns null (e.g., document already detached when `getType()` is called), this will crash. While `getType()` checks `GetExecutionContext()` before calling `ReadRepresentationFromClipboardReader()`, there is a potential timing issue: the context could be destroyed between the check in `getType()` and the `ClipboardReader` constructor.

```cpp
ClipboardReader::ClipboardReader(SystemClipboard* system_clipboard,
                                 ClipboardReaderResultHandler* result_handler)
    : clipboard_task_runner_(
          result_handler->GetExecutionContext()->GetTaskRunner(  // potential null deref
              TaskType::kUserInteraction)),
```

**Suggestion**: Add a null check, or assert that the execution context is valid:
```cpp
ExecutionContext* context = result_handler->GetExecutionContext();
CHECK(context);
clipboard_task_runner_ = context->GetTaskRunner(TaskType::kUserInteraction);
```

---

### Issue #4: Resolved promises are cached but rejected promises are not consistently erased

**Severity**: Major
**File**: `clipboard_item.cc`
**Lines**: 151-176, 211-214
**Description**: In `ResolveFormatData()`, when a blob is null or the clipboard changed, the resolver is rejected and then erased from `representations_with_resolvers_`. The comment in `getType()` says "Resolved data is cached; rejected resolvers are erased to allow retry." However, when `ResolveFormatData` resolves successfully (line 175), the resolver is *not* erased — it stays in the map. This means a subsequent `getType()` call for the same type will hit the `has_cached_resolver` branch and return the already-resolved promise. This is correct behavior for caching, but the resolved `ScriptPromiseResolver` object stays alive unnecessarily. This is a minor memory concern, not a correctness bug, but worth noting.

More importantly: if a user calls `getType("text/plain")` concurrently (two calls before the first resolves), the second call will hit the `has_cached_resolver` path and return the same promise, which is correct. But if the first call fails and the resolver is erased, and a third concurrent call was also issued (hitting `has_cached_resolver` when the resolver was still there), there could be inconsistent state.

**Suggestion**: This is acceptable for the current design but worth documenting more clearly. Consider adding a comment in `getType()` explaining the full lifecycle of entries in `representations_with_resolvers_`.

---

### Issue #5: `ContextDestroyed()` rejects promises but `getType()` doesn't check for detached context before returning cached promise

**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 199-201, 211-214, 285-293
**Description**: `getType()` checks `GetExecutionContext()` for the lazy-read path (line 199) before creating new resolvers. However, the `has_cached_resolver` check (line 211) does not re-verify that the context is still alive. After `ContextDestroyed()` is called, `representations_with_resolvers_` is cleared, so this *should* be safe (the find will fail). But there is a subtle timing window: if `ContextDestroyed()` runs between the `GetExecutionContext()` check and the map lookup, the map will be empty and the code will fall through to create a new resolver with a destroyed context's script state.

**Suggestion**: This is likely not exploitable in practice due to single-threaded execution, but consider moving the `GetExecutionContext()` check to after all early returns or adding a CHECK.

---

### Issue #6: `ClipboardReaderResultHandler::Trace()` has empty body

**Severity**: Minor
**File**: `clipboard_reader.h`
**Line**: 24
**Description**: The `Trace()` method in `ClipboardReaderResultHandler` has an empty body. While this is technically correct for a mixin with no traced members, it would be more idiomatic in Chromium to call the parent's `Trace()`:

```cpp
void Trace(Visitor* visitor) const override {}
```

**Suggestion**: Consider calling `GarbageCollectedMixin::Trace(visitor)` for consistency, though this is not strictly required:
```cpp
void Trace(Visitor* visitor) const override {
  GarbageCollectedMixin::Trace(visitor);
}
```

---

### Issue #7: `ClipboardItem` header includes `clipboard_reader.h` — introduces tight coupling

**Severity**: Minor
**File**: `clipboard_item.h`
**Line**: 15
**Description**: `clipboard_item.h` directly includes `clipboard_reader.h` to inherit from `ClipboardReaderResultHandler`. This creates a dependency where any consumer that includes `clipboard_item.h` also pulls in `clipboard_reader.h`. While this is required by the current design (inheritance), consider whether `ClipboardReaderResultHandler` should be in its own small header file to break this coupling.

**Suggestion**: Extract `ClipboardReaderResultHandler` into a separate header (e.g., `clipboard_reader_result_handler.h`) that both `clipboard_reader.h` and `clipboard_item.h` can include. This would keep the `ClipboardItem` → `ClipboardReader` dependency minimal.

---

### Issue #8: Missing test for concurrent `getType()` rejection when clipboard changes

**Severity**: Minor
**File**: `clipboard_unittest.cc`, web tests
**Description**: The web test `async-clipboard-lazy-read.html` tests clipboard change detection, and `async-custom-format-lazy-read-concurrent.tentative.https.html` tests concurrent reads. However, there is no test that verifies the behavior when:
1. Multiple `getType()` calls are in flight and the clipboard changes mid-flight — some should resolve, some should reject
2. `getType()` is called after `ContextDestroyed()` — should return empty promise
3. `getType()` is called with a type not in the ClipboardItem's types list for the lazy path

**Suggestion**: Add unit tests for these edge cases, particularly the context-destroyed scenario.

---

### Issue #9: `getType()` returns empty `ScriptPromise<Blob>()` without exception when context is destroyed

**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 199-201
**Description**: When `GetExecutionContext()` returns null (detached document), `getType()` returns `ScriptPromise<Blob>()` — an empty/undefined promise — without throwing an exception or rejecting. This is inconsistent with the non-lazy path which throws `DOMExceptionCode::kNotFoundError`. The spec may not cover this edge case, but returning an empty promise that never resolves or rejects could lead to memory leaks in JavaScript (hanging `.then()` callbacks).

**Suggestion**: Consider throwing a `DOMException` with `kNotAllowedError` ("Document detached") to match the behavior of `ContextDestroyed()`, or at minimum document why an empty promise is returned.

---

### Issue #10: Web test `async-clipboard-custom-format-lazy-read.html` doesn't verify lazy behavior

**Severity**: Suggestion
**File**: `async-clipboard-custom-format-lazy-read.html`
**Description**: The test verifies that custom format data can be read via `getType()`, but it doesn't verify the *lazy* aspect — i.e., that the data was not read during `clipboard.read()` but only when `getType()` was called. The unit test `ReadOnlyMimeTypesInClipboardRead` does verify this using mock tracking, but the web test only checks end-to-end correctness.

**Suggestion**: Consider adding a comment explaining that the lazy-read behavior verification is done in the unit test, or restructure the test name to reflect that it tests custom format correctness rather than laziness.

---

### Issue #11: `ClipboardPromise::OnRead` ignores `mime_type` parameter

**Severity**: Suggestion
**File**: `clipboard_promise.cc`
**Lines**: 475-478
**Description**: The comment "ClipboardPromise tracks representation index internally, so ignore mime_type" is clear and correct. However, the `mime_type` parameter is now part of the `ClipboardReaderResultHandler` interface signature. This means `ClipboardPromise` receives data it doesn't use. While this is fine for interface uniformity, consider adding a `[[maybe_unused]]` annotation or using the parameter name in a way that makes it clear it's intentionally ignored.

**Suggestion**: Use `[[maybe_unused]]` or leave as-is with the comment. The current approach is acceptable.

---

## Positive Observations

1. **Good architectural refactoring**: The introduction of `ClipboardReaderResultHandler` as an interface is a clean design that decouples `ClipboardReader` from `ClipboardPromise`, allowing `ClipboardItem` to directly receive read results. This addresses a previous review comment well.

2. **Clipboard change detection**: The use of sequence numbers to detect clipboard content changes between `read()` and `getType()` is a thoughtful security/correctness measure, and using `std::optional<absl::uint128>` to properly distinguish "unset" from "zero" is a good fix over the previous implementation.

3. **Context lifecycle management**: Implementing `ExecutionContextLifecycleObserver` in `ClipboardItem` to reject pending promises on document detachment is the correct pattern for preventing dangling callbacks.

4. **Active reader tracking**: Using `HeapHashMap<String, Member<ClipboardReader>> active_readers_` to keep readers alive until completion prevents use-after-free — this directly addresses a UAF concern raised in prior review comments.

5. **Feature flag gating**: The `ClipboardReadOnDemand` feature is properly gated behind a runtime-enabled feature (`status: "test"`), allowing incremental rollout and easy rollback.

6. **Good test coverage**: The CL adds both unit tests (with mock call tracking to verify lazy behavior) and web platform tests (end-to-end behavior verification including concurrent reads and clipboard change detection).

7. **Responsive to review feedback**: The iteration history shows the author addressed all prior review comments including extracting the interface, fixing null checks, using `std::optional`, and adding additional test coverage.

---

## Overall Assessment

**LGTM with minor comments**

The CL is well-structured and implements a meaningful performance optimization (deferring clipboard data reads to `getType()` time). The `ClipboardReaderResultHandler` interface extraction is a clean design, and the clipboard change detection mechanism is appropriate.

The main concerns are:
- **Critical**: The null dereference risk in `ResolveRead()` when calling `GetSystemClipboard()->SequenceNumber()` (Issue #1) should be addressed before landing.
- **Major**: The `ClipboardReader` constructor null dereference risk (Issue #3) should also be addressed.
- The TOCTOU nature of the clipboard change check (Issue #2) is inherent to the design and acceptable as a best-effort check, but should be documented.

After addressing Issues #1 and #3, this CL should be ready for approval.
