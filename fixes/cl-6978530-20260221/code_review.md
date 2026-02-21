# Code Review: CL 6978530 — [Clipboard] Implementation of lazy read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Patch Set Reviewed:** PS 15 (ecd09ea88730)
**Files Changed:** 10 files, +478/-50 lines

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Race condition with concurrent `getType()` calls; `clipboard_reader_` overwrite; missing cleanup in `ContextDestroyed` |
| Style | ⚠️ | Duplicated feature-check patterns in `clipboard_reader.cc`; blank line nit in `clipboard_item.h`; missing blank line before `// static` |
| Security | ⚠️ | `read_callbacks_` cleanup gap on context destruction could leak Persistent handles |
| Performance | ✅ | Lazy read design is sound; defers data reads until `getType()` is called |
| Testing | ⚠️ | Missing edge-case tests: concurrent `getType()` calls, unsupported types on lazy path, context destruction during read |

---

## Detailed Findings

### Issue #1: Race condition — `clipboard_reader_` overwritten on concurrent `getType()` calls
**Severity**: Critical
**File**: `clipboard_promise.cc`
**Line**: ~357 (`ReadRepresentationFromClipboardReader`)
**Description**: `clipboard_reader_` is a single `Member<ClipboardReader>` on `ClipboardPromise`. When `ReadRepresentationFromClipboard` is called multiple times for different MIME types (e.g., `item.getType("text/plain")` and `item.getType("text/html")` called in quick succession), each call posts a task that creates a new `ClipboardReader` and assigns it to `clipboard_reader_`. The second assignment overwrites the first reader while it may still be performing an async read. This could cause the first reader to be garbage collected prematurely, dropping its callback.

**Suggestion**: Use a `HeapHashMap<String, Member<ClipboardReader>>` keyed by format, or a `HeapVector<Member<ClipboardReader>>`, so that multiple readers can be active simultaneously. Alternatively, serialize reads by queuing format requests.

---

### Issue #2: `read_callbacks_` not cleaned up in `ContextDestroyed()`
**Severity**: Major
**File**: `clipboard_promise.cc`
**Line**: ~848 (`ContextDestroyed`)
**Description**: When the execution context is destroyed, `ContextDestroyed()` clears `clipboard_writer_` and `clipboard_reader_`, but does **not** clear `read_callbacks_`. Each callback in `read_callbacks_` captures a `WrapPersistent(this)` handle to the `ClipboardItem`. If these callbacks are never executed, the persistent handles are never released, preventing the `ClipboardItem` from being garbage collected.

**Suggestion**: Add `read_callbacks_.clear();` in `ContextDestroyed()` to release all pending persistent handles. Alternatively, iterate and invoke each callback with `nullptr` to properly reject the pending promises.

```cpp
void ClipboardPromise::ContextDestroyed() {
  script_promise_resolver_->Reject(MakeGarbageCollected<DOMException>(
      DOMExceptionCode::kNotAllowedError, "Document detached."));
  clipboard_writer_.Clear();
  clipboard_reader_.Clear();
  read_callbacks_.clear();  // Release WrapPersistent handles
}
```

---

### Issue #3: Missing blank line before `// static` in `clipboard_item.cc`
**Severity**: Minor
**File**: `clipboard_item.cc`
**Line**: ~229
**Description**: The diff shows `CheckSequenceNumber()` ending immediately before `// static` for `ClipboardItem::supports()` without a blank line separator:

```cpp
  return sequence_number_token == sequence_number_;
}
// static
bool ClipboardItem::supports(const String& type) {
```

**Suggestion**: Add a blank line before `// static` per Chromium style.

---

### Issue #4: Duplicated feature-check pattern in `clipboard_reader.cc`
**Severity**: Minor
**File**: `clipboard_reader.cc`
**Lines**: ~54, ~124, ~212, ~294, ~324
**Description**: The same `if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled())` / `else` branching pattern for choosing between `promise_->OnRead(blob, mime_type)` and `promise_->OnRead(blob)` is repeated five times across all reader subclasses. This is fragile and adds maintenance burden for future reader implementations.

**Suggestion**: Consider adding a single helper method in the `ClipboardReader` base class:

```cpp
void ClipboardReader::NotifyRead(Blob* blob, const String& mime_type) {
  if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()) {
    promise_->OnRead(blob, mime_type);
  } else {
    promise_->OnRead(blob);
  }
}
```

Each subclass then calls `NotifyRead(blob, mime_type)` instead of duplicating the branch. This was also flagged by a reviewer (Ashish Kumar) in the inline comments.

---

### Issue #5: `CheckSequenceNumber()` called twice on the lazy read path
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: ~190, ~153-158
**Description**: In `getType()`, when the lazy path is taken, `CheckSequenceNumber()` is called at line ~190 *before* issuing the read. Then in `ResolveFormatData()`, `CheckSequenceNumber()` is called *again* at line ~153 after the data arrives. The first check is largely redundant — if the clipboard changed between `read()` and `getType()`, the second check in `ResolveFormatData()` would catch it after the actual data read. The first check adds an extra IPC round-trip to `GetSequenceNumber()`.

Note: A reviewer (Ashish Kumar) already flagged this, suggesting the check belongs in `ResolveFormatData` only, and the author marked it as "Done" — verify this has been applied.

**Suggestion**: Remove the `CheckSequenceNumber()` call from `getType()` and rely solely on the check in `ResolveFormatData()`, which happens after the read completes and is the definitive validation point. If keeping the early check is intentional (to fail fast), document this rationale.

---

### Issue #6: `ResolveFormatData` — duplicate rejection logic
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: ~143-162
**Description**: `ResolveFormatData` has two separate rejection paths that produce the same DOMException:

```cpp
if (blob == nullptr) {
  representations_with_resolvers_.at(mime_type)->Reject(
      MakeGarbageCollected<DOMException>(DOMExceptionCode::kDataError,
                                         "Clipboard data has changed"));
  return;
}

if (!CheckSequenceNumber()) {
  representations_with_resolvers_.at(mime_type)->Reject(
      MakeGarbageCollected<DOMException>(DOMExceptionCode::kDataError,
                                         "Clipboard data has changed"));
  return;
}
```

**Suggestion**: Combine the conditions. Also, a `nullptr` blob might indicate a read failure rather than clipboard change; consider differentiating the error messages:

```cpp
if (blob == nullptr) {
  representations_with_resolvers_.at(mime_type)->Reject(
      MakeGarbageCollected<DOMException>(DOMExceptionCode::kDataError,
                                         "Failed to read clipboard data"));
  return;
}
if (!CheckSequenceNumber()) {
  representations_with_resolvers_.at(mime_type)->Reject(
      MakeGarbageCollected<DOMException>(DOMExceptionCode::kDataError,
                                         "Clipboard data has changed"));
  return;
}
```

---

### Issue #7: `GetSequenceNumberToken()` defensive null checks return `0`
**Severity**: Minor
**File**: `clipboard_promise.cc`
**Lines**: ~831-840
**Description**: `GetSequenceNumberToken()` returns `0` when `frame` or `clipboard` is null. However, `0` could theoretically be a valid sequence number (e.g., initial clipboard state). If `sequence_number_` stored in the `ClipboardItem` also happens to be `0` (which is the default), the comparison in `CheckSequenceNumber()` would incorrectly succeed.

**Suggestion**: Use `std::optional<absl::uint128>` as the return type, or document that `0` is a reserved invalid sentinel value that the system clipboard never returns.

---

### Issue #8: `representations_with_resolvers_` not cleaned up when promise rejected
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: ~143-162
**Description**: When `ResolveFormatData` rejects a promise resolver, the entry remains in `representations_with_resolvers_`. A subsequent `getType()` call for the same MIME type would find the existing (now rejected) resolver at line ~196 and return its already-rejected promise. This means the caller cannot retry. This may be intentional, but it's worth documenting.

**Suggestion**: Either erase the entry after rejection to allow retry, or add a comment explaining that rejection is permanent per `ClipboardItem` instance.

---

### Issue #9: Web test lacks `ClipboardReadOnDemand` feature flag activation
**Severity**: Major
**File**: `async-clipboard-lazy-read.html`
**Line**: 1-36
**Description**: The web test in `async-clipboard-lazy-read.html` does not enable the `ClipboardReadOnDemand` runtime feature flag. The feature is set to `status: "test"` in `runtime_enabled_features.json5`, which means it's enabled in layout tests via the test runner. However, the test should still be explicit about its dependency, and it's unclear if the test runner automatically enables `"test"` features for all web tests.

Additionally, the test only covers the clipboard-change-detection path (writing new content between `read()` and `getType()`). It does not test the happy path where `getType()` succeeds on a lazy-read `ClipboardItem`.

**Suggestion**: 
1. Add a positive test case where `getType()` succeeds without clipboard changes.
2. Consider adding a test for calling `getType()` with an unsupported type.

---

### Issue #10: Unit test `ClipboardItemGetType` helper returns `String` instead of `Blob`
**Severity**: Minor
**File**: `clipboard_unittest.cc`
**Lines**: ~52-68
**Description**: The `ClipboardItemGetType` helper class extends `ThenCallable<IDLSequence<ClipboardItem>, ClipboardItemGetType, IDLString>` and returns `"SUCCESS"` as a string. However, the `React` method calls `getType()` which returns a `ScriptPromise<Blob>`, but never awaits this inner promise. The test only verifies that `getType()` was *called* without error, not that the returned blob contains the correct data.

**Suggestion**: The test should await the `ScriptPromise<Blob>` returned by `getType()` to verify the blob content matches what was written to the clipboard. As-is, the test proves `getType()` doesn't immediately throw, but not that the lazy read correctly delivers data.

---

### Issue #11: Blank line inconsistency in `clipboard_item.h`
**Severity**: Minor
**File**: `clipboard_item.h`
**Line**: ~79
**Description**: The diff removes a blank line between `CustomFormats()` and the `// ScriptWrappable` comment:

```diff
-
   // ScriptWrappable
   void Trace(Visitor*) const override;
```

This reduces visual separation between the public API section and the Trace override.

**Suggestion**: Retain the blank line for readability.

---

## Positive Observations

- **Sound architecture**: The lazy-read design correctly defers data reads to `getType()` time, which can significantly reduce IPC overhead for `clipboard.read()` when web pages only need a subset of available formats.
- **Proper feature gating**: All new behavior is guarded behind `RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()`, allowing safe rollout and rollback.
- **Change detection**: The sequence-number-based clipboard change detection is a good security measure that prevents stale data from being returned.
- **Good test coverage for the happy path**: The `ReadOnlyMimeTypesInClipboardRead` test validates the core lazy-loading behavior by verifying that `ReadText`/`ReadHtml` are not called during `clipboard.read()`.
- **Mock infrastructure additions**: Adding call-tracking to `MockClipboardHost` is clean and reusable for future tests.
- **Proper GC tracing**: New `Member<>` fields are properly added to `Trace()` methods in both `ClipboardItem` and `ClipboardPromise`.

---

## Overall Assessment

**Needs changes before approval**

The core design of lazy clipboard reading is sound and well-motivated. However, there are two issues that should be addressed before landing:

1. **Critical**: The `clipboard_reader_` single-member race condition (Issue #1) means concurrent `getType()` calls for different formats will likely fail. This is a real usage pattern (e.g., `Promise.all([item.getType("text/plain"), item.getType("text/html")])`).

2. **Major**: The `read_callbacks_` leak in `ContextDestroyed()` (Issue #2) could cause memory leaks in production when pages are navigated away while clipboard reads are in-flight.

The remaining issues are minor style/documentation items that can be addressed in follow-up patches.
