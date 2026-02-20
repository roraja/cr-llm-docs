# Code Review: CL 6978530 — [Clipboard] Implementation of lazy read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Patch Set:** 15
**Files Changed:** 10 (+478/−50)

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Race condition with `clipboard_reader_` overwrite; null blob rejection message misleading; missing callback invocation on early return |
| Style | ⚠️ | Minor style issues: blank line, missing comment spacing, `DCHECK` usage |
| Security | ✅ | Sequence number checks and permission flow look correct |
| Performance | ✅ | Lazy read defers clipboard data fetching until `getType()` — good design |
| Testing | ⚠️ | Missing test for concurrent `getType()` calls on different MIME types; missing test for `getType()` on unsupported type; web test coverage is minimal |

## Detailed Findings

---

#### Issue #1: Race condition — `clipboard_reader_` overwritten on concurrent `getType()` calls
**Severity**: Major
**File**: `clipboard_promise.cc`
**Line**: 357
**Description**: `ReadRepresentationFromClipboardReader` assigns `clipboard_reader_ = clipboard_reader` each time it is called. If a user calls `getType("text/plain")` and `getType("text/html")` in quick succession before the first read completes, the second call overwrites `clipboard_reader_`, and the first `ClipboardReader` may be garbage-collected while its async read is still in flight. The old reader's callback would reference a destroyed object or its result would silently vanish.

In the non-lazy path this wasn't a problem because `ReadNextRepresentation()` reads formats sequentially. But the lazy path allows truly concurrent reads.

**Suggestion**: Either (a) store `clipboard_reader_` per-format (e.g., a `HeapHashMap<String, Member<ClipboardReader>>`) so each in-flight read has its own reference, or (b) serialize the lazy reads through a queue. At minimum, the `ClipboardReader` should be prevented from being collected while its async work is pending.

---

#### Issue #2: `ReadRepresentationFromClipboardReader` silently drops read on missing `ExecutionContext`
**Severity**: Major
**File**: `clipboard_promise.cc`
**Line**: 347-349
**Description**: When `GetExecutionContext()` returns null, the method returns without calling the callback stored in `read_callbacks_`. This leaves the `ScriptPromiseResolver` in `ClipboardItem::representations_with_resolvers_` permanently unresolved, leaking the promise and preventing garbage collection of the associated JS objects.

**Suggestion**: Invoke the callback with `nullptr` before returning:
```cpp
if (!GetExecutionContext()) {
  OnRead(nullptr, format);
  return;
}
```

---

#### Issue #3: Misleading error message when `blob == nullptr`
**Severity**: Minor
**File**: `clipboard_item.cc`
**Line**: 145-148
**Description**: `ResolveFormatData` rejects with "Clipboard data has changed" when `blob` is null. However, a null blob could indicate a read failure (e.g., unsupported format, empty clipboard) — not necessarily a clipboard data change. This may confuse web developers debugging clipboard issues.

**Suggestion**: Use a more accurate message like "Failed to read clipboard data" or differentiate the error based on the actual cause.

---

#### Issue #4: Duplicate `HashMap` lookups in `ResolveFormatData`
**Severity**: Minor
**File**: `clipboard_item.cc`
**Line**: 141-159
**Description**: `representations_with_resolvers_.at(mime_type)` is called 3 times (null check, reject path, resolve path). While not a correctness issue, this is 3 hash lookups that could be 1.

**Suggestion**: Cache the iterator/pointer:
```cpp
auto* resolver = representations_with_resolvers_.at(mime_type);
if (!blob) {
  resolver->Reject(...);
  return;
}
if (!CheckSequenceNumber()) {
  resolver->Reject(...);
  return;
}
resolver->Resolve(blob);
```

---

#### Issue #5: `getType()` returns same promise for duplicate calls but doesn't handle already-resolved state
**Severity**: Major
**File**: `clipboard_item.cc`
**Line**: 185-188
**Description**: When `getType()` is called a second time for the same MIME type, it returns the cached promise from `representations_with_resolvers_`. If the first call already resolved or rejected this promise, the second call returns a settled promise, which is correct in the success case. However, if the first call's sequence number check fails (rejecting the promise), a subsequent call after a *new* `clipboard.read()` reusing the same `ClipboardItem` would still return the rejected promise. The code does not clear the rejected resolver, so the item becomes permanently "broken" for that type.

**Suggestion**: Consider removing rejected resolvers from the map, or document that `ClipboardItem` objects are single-use for lazy reads.

---

#### Issue #6: `ClipboardItem` holds a persistent reference to `ClipboardPromise`, creating a lifecycle concern
**Severity**: Major
**File**: `clipboard_item.h`
**Line**: 110 (Member<ClipboardPromise> clipboard_promise_)
**Description**: The `ClipboardItem` is returned to JavaScript and may be held indefinitely. It retains a strong reference (`Member<ClipboardPromise>`) to the `ClipboardPromise` which itself holds a `Member<ScriptPromiseResolverBase>`, `HeapMojoRemote<PermissionService>`, and the entire clipboard state. This keeps the `ClipboardPromise` alive as long as the JS object is alive, potentially long after the initial `clipboard.read()` has resolved.

The `ClipboardPromise` was designed as a short-lived helper object that is created, does its work, and is collected. Anchoring it to a potentially long-lived JS-exposed object changes its lifecycle semantics significantly.

**Suggestion**: Consider having `ClipboardItem` hold only the data it needs to perform lazy reads (e.g., a reference to `SystemClipboard` and the permission state) rather than the entire `ClipboardPromise`. Alternatively, add a cleanup step that clears the `clipboard_promise_` reference once the `ClipboardPromise`'s main promise has resolved.

---

#### Issue #7: `read_callbacks_` is `HashMap` (not traced by GC) but stores `base::OnceCallback` holding `WrapPersistent(this)` pointers
**Severity**: Minor
**File**: `clipboard_promise.h`
**Line**: 220-221
**Description**: `read_callbacks_` is a `HashMap<String, base::OnceCallback<...>>` which is not a `Heap*` container and therefore not traced by the GC. The callbacks contain persistent pointers (`WrapPersistent(this)` from `ClipboardItem`), which is correct — `WrapPersistent` prevents GC collection. However, if a callback is never invoked (e.g., due to Issue #2 or a context destruction race), the `Persistent` handle creates a leak.

**Suggestion**: Ensure all callbacks are either invoked or cleared in `ContextDestroyed()`. Currently `ContextDestroyed()` does not clear `read_callbacks_`.

---

#### Issue #8: Missing blank line before `// static` comment
**Severity**: Minor (Style)
**File**: `clipboard_item.cc`
**Line**: 219
**Description**: There should be a blank line before `// static`:
```cpp
}
// static    <-- Missing blank line after closing brace
bool ClipboardItem::supports(const String& type) {
```

**Suggestion**: Add a blank line before `// static`.

---

#### Issue #9: `OnReadAvailableFormatNames` skips `check_types_to_read` filtering in lazy path
**Severity**: Minor
**File**: `clipboard_promise.cc`
**Line**: 463-476
**Description**: When `ClipboardReadOnDemandEnabled()` is true, the code initializes `item_mime_types_` with `format_names.size()` capacity but still applies the `check_types_to_read` filter in the loop. This is correct behavior, but the capacity reservation doesn't account for filtering (same issue exists in the non-lazy path, to be fair). Not a bug, but inconsistent.

---

#### Issue #10: Web test only covers the "clipboard changed" rejection case
**Severity**: Minor
**File**: `async-clipboard-lazy-read.html`
**Line**: 1-36
**Description**: The web test only tests that `getType()` throws `DataError` when clipboard contents change between `read()` and `getType()`. There is no positive test case — i.e., verifying that `getType()` successfully returns the correct data when the clipboard hasn't changed. There's also no test for:
- Multiple `getType()` calls on different formats
- Calling `getType()` with a type not in the clipboard
- The `types()` accessor returning the correct types in lazy mode

**Suggestion**: Add at least a positive test case and a test for the `types()` property.

---

#### Issue #11: Feature flag interaction with `SelectiveClipboardFormatRead`
**Severity**: Minor
**File**: `clipboard_promise.cc`
**Line**: 456-462
**Description**: When both `ClipboardReadOnDemandEnabled` and `SelectiveClipboardFormatReadEnabled` are true, and `read_clipboard_item_types_` is empty, the code calls `ResolveRead()` which creates a lazy `ClipboardItem` with an empty `item_mime_types_`. This is technically correct (no types to read), but the lazy `ClipboardItem` object still holds a reference to `ClipboardPromise` even though it will never be used. Consider skipping the `ClipboardPromise` reference in this case.

---

#### Issue #12: Unit tests use `ScopedClipboardReadOnDemandForTest` implicitly via `status: "test"`
**Severity**: Minor
**File**: `clipboard_unittest.cc`
**Line**: 260-348
**Description**: The new tests (`ReadOnlyMimeTypesInClipboardRead`, `ClipboardItemGetTypeTest`) rely on `ClipboardReadOnDemand` being enabled via `status: "test"` in `runtime_enabled_features.json5`. However, the existing tests were not written with this feature in mind. Since the feature is now enabled in test mode, it changes the behavior of the existing tests (`ClipboardReadTypesTest`, `ClipboardItemGetTypeTelemetryTest`, `ClipboardReadFilterTypes`) — they now also take the lazy path instead of the eager read path.

This means the existing tests no longer exercise the non-lazy code path when run in test mode, reducing coverage of the legacy path.

**Suggestion**: Either add explicit `ScopedClipboardReadOnDemandForTest` scoping to the new tests (enabling it) and ensure the old tests explicitly disable it, or add test variants that cover both paths.

---

## Positive Observations

- **Good design**: The lazy read approach is a sound performance optimization — deferring actual data reads until `getType()` is called avoids unnecessary IPC and data copies when web pages only inspect `types()`.
- **Sequence number checking**: The clipboard change detection using sequence numbers is a correct approach to handle the TOCTOU window between `read()` and `getType()`.
- **Proper use of `WrapPersistent`**: The callback binding in `getType()` correctly uses `WrapPersistent(this)` to prevent premature garbage collection of the `ClipboardItem`.
- **Feature flag guarding**: The entire lazy read path is properly gated behind `RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()`, ensuring it doesn't affect existing behavior when disabled.
- **Mock call tracking in tests**: The addition of `WasReadTextCalled()` etc. to `MockClipboardHost` is a clean way to verify lazy loading behavior.
- **Reviewer comments addressed**: The author has addressed all inline comments from the previous review round.

## Overall Assessment

**Needs changes before approval**

The CL implements a useful optimization (lazy clipboard reading), but has several issues that should be addressed:

1. **Critical**: The `clipboard_reader_` overwrite race (Issue #1) is the most serious concern. Concurrent `getType()` calls — which are the natural usage pattern — will corrupt the reader state.
2. **Important**: The promise leak on `ExecutionContext` destruction (Issue #2) and the leaked callbacks in `ContextDestroyed()` (Issue #7) should be fixed.
3. **Important**: The `ClipboardPromise` lifecycle concern (Issue #6) may cause memory pressure if JS code holds onto `ClipboardItem` objects. This warrants discussion on the design.
4. **Testing**: The existing tests may unintentionally change behavior due to the feature flag being test-enabled (Issue #12). The web test coverage is thin (Issue #10).

The design direction is sound and the code is generally well-structured. Addressing the race condition and lifecycle issues would make this ready for landing.
