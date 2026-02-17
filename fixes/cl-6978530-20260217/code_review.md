# Code Review: CL 6978530 — [Clipboard] Implementation of lazy read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Files Changed:** 10 files, +447/-43 lines

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Several logic issues: missing method definition, double exception throwing, incorrect UMA histogram, uninitialized `creation_time_` |
| Style | ⚠️ | Unnecessary include, removed documentation comment, missing newline at EOF |
| Security | ✅ | Clipboard change detection is a security improvement; sequence number checks are sound |
| Performance | ✅ | Lazy reading avoids unnecessary data reads; appropriate use of PostTask |
| Testing | ⚠️ | Good basic tests but missing edge cases for concurrent getType calls and error paths |

## Detailed Findings

### Issue #1: `SetClipboardPromise` declared but never defined
**Severity**: Critical
**File**: clipboard_item.h / clipboard_item.cc
**Line**: clipboard_item.h:84
**Description**: The method `SetClipboardPromise(ClipboardPromise* clipboard_promise, absl::uint128 sequence_number)` is declared as a public method in the header but has no implementation in the `.cc` file. This will cause a linker error if anything calls it. Additionally, it appears to be dead code since the new constructor already takes a `ClipboardPromise*` parameter.
**Suggestion**: Either implement the method in `clipboard_item.cc`, or remove the declaration if it's unnecessary (since the constructor already accepts both parameters).

### Issue #2: `ClipboardItem::Create(const HeapVector<String>&, ...)` declared but never defined
**Severity**: Critical
**File**: clipboard_item.h / clipboard_item.cc
**Line**: clipboard_item.h:43-44
**Description**: The static factory method `Create(const HeapVector<String>& mime_types, ExceptionState& exception_state)` is declared in the header but has no corresponding implementation in the `.cc` file. It also appears to never be called—the on-demand path in `ResolveRead()` constructs `ClipboardItem` directly via `MakeGarbageCollected`. This is dead code that will cause a linker error if anything tries to call it.
**Suggestion**: Remove this declaration since it's unused. The direct `MakeGarbageCollected<ClipboardItem>(mime_types, seq, this)` call in `ResolveRead()` is the correct approach.

### Issue #3: Double exception throwing in `getType()` fall-through path
**Severity**: Major
**File**: clipboard_item.cc
**Lines**: 196-204
**Description**: When `clipboard_promise_` is set but the MIME type is not found in `mime_types_`, the code throws a `NotFoundError` at line 197-198 (inside the `else` block). Then execution falls through to lines 201-203, which throw *another* `NotFoundError` on the same `ExceptionState`. Throwing on an `ExceptionState` that already has an exception is a DCHECK failure in debug builds.
**Suggestion**: Add `return ScriptPromise<Blob>();` after the `else` block at line 199, or restructure the control flow to ensure only one exception is thrown. For example:
```cpp
    } else {
      exception_state.ThrowDOMException(DOMExceptionCode::kNotFoundError,
                                        "The type was not found");
      return ScriptPromise<Blob>();
    }
```

### Issue #4: UMA histogram records wrong value when on-demand is enabled
**Severity**: Major
**File**: clipboard_promise.cc
**Lines**: 402-403
**Description**: `ResolveRead()` records `clipboard_item_data_.size()` to UMA at the top of the method. However, when `ClipboardReadOnDemandEnabled()` is true, `clipboard_item_data_` is never populated—`item_mime_types_` is used instead. This means the histogram will always record 0 when lazy reading is enabled, which produces misleading telemetry.
**Suggestion**: Record `item_mime_types_.size()` when on-demand is enabled:
```cpp
base::UmaHistogramCounts100(
    "Blink.Clipboard.Read.NumberOfFormats",
    RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()
        ? item_mime_types_.size()
        : clipboard_item_data_.size());
```

### Issue #5: `creation_time_` not initialized in new constructor
**Severity**: Minor
**File**: clipboard_item.cc
**Lines**: 73-81
**Description**: The new `ClipboardItem` constructor that takes `HeapVector<String>& mime_types` does not initialize `creation_time_` (unlike the existing constructor which sets `creation_time_(base::TimeTicks::Now())`). This means `creation_time_` will be default-constructed (zero/null). If `CaptureTelemetry` or any other code relies on a valid `creation_time_`, it could produce incorrect timing data.
**Suggestion**: Add `creation_time_(base::TimeTicks::Now())` to the initializer list, consistent with the other constructor.

### Issue #6: `ReadRepresentationFromClipboardReader` lacks null check for `ClipboardReader`
**Severity**: Major
**File**: clipboard_promise.cc
**Lines**: 344-351
**Description**: `ReadRepresentationFromClipboardReader` calls `ClipboardReader::Create()` and immediately calls `clipboard_reader->Read()` without checking for `nullptr`. In contrast, the existing `ReadNextRepresentation()` method (line 491-493) checks if the reader is null and handles it gracefully. If `Create()` returns null (e.g., for an unsupported type that somehow passed validation), this will crash.
**Suggestion**: Add a null check:
```cpp
void ClipboardPromise::ReadRepresentationFromClipboardReader(
    const String& format) {
  DCHECK(RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled());
  ClipboardReader* clipboard_reader = ClipboardReader::Create(
      GetLocalFrame()->GetSystemClipboard(), format, this,
      /*sanitize_html=*/!will_read_unprocessed_html_);
  if (!clipboard_reader) {
    OnRead(nullptr, format);
    return;
  }
  clipboard_reader->Read();
}
```

### Issue #7: `ReadRepresentationFromClipboardReader` lacks frame/context null check
**Severity**: Major
**File**: clipboard_promise.cc
**Lines**: 344-351
**Description**: `ReadRepresentationFromClipboardReader` is invoked asynchronously via `PostTask`. Between the time the task is posted and when it executes, the frame could have been destroyed. `GetLocalFrame()` could return null, and `GetLocalFrame()->GetSystemClipboard()` would then dereference a null pointer. The caller `ReadRepresentationFromClipboard` checks `GetExecutionContext()` but the posted task does not.
**Suggestion**: Add a null check at the beginning:
```cpp
void ClipboardPromise::ReadRepresentationFromClipboardReader(
    const String& format) {
  DCHECK(RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled());
  if (!GetExecutionContext()) {
    return;
  }
  // ... rest of method
}
```

### Issue #8: Unnecessary include of `tokens.h`
**Severity**: Minor
**File**: clipboard_promise.h
**Line**: 13
**Description**: The include `#include "third_party/blink/public/common/tokens/tokens.h"` was added but `absl::uint128` (the only new type used) comes from `absl/numeric/int128.h`, which is already transitively included through `clipboard_item.h` or can be directly included. The `tokens.h` header is unrelated to the types used here.
**Suggestion**: Replace with `#include "third_party/abseil-cpp/absl/numeric/int128.h"` or verify it's already available through existing transitive includes.

### Issue #9: `sequence_number_ == 0` used as sentinel for "non-lazy" items
**Severity**: Minor
**File**: clipboard_item.cc
**Lines**: 119-134, 162-163
**Description**: The code uses `sequence_number_ == 0` to distinguish lazy-read ClipboardItems from non-lazy ones (lines 121, 162). However, `sequence_number_` is of type `absl::uint128` and defaults to 0 for the non-lazy constructor. This coupling between a sentinel value and an implementation detail is fragile. If the system clipboard ever legitimately returns a 0 sequence number, the code would incorrectly use the non-lazy path.
**Suggestion**: Consider adding an explicit boolean member `is_lazy_read_` to clearly distinguish the two code paths, rather than relying on the sequence number value.

### Issue #10: Missing newline at end of web test file
**Severity**: Minor
**File**: async-clipboard-lazy-read.html
**Line**: 36
**Description**: The file doesn't end with a newline character (`\ No newline at end of file` in the diff). This is a minor style issue but can cause diff noise.
**Suggestion**: Add a trailing newline.

### Issue #11: Removed documentation comment for `supports()`
**Severity**: Minor
**File**: clipboard_item.h
**Lines**: 72-75 (original)
**Description**: The documentation comment for the `supports()` static method was removed:
```cpp
// Checks if a particular MIME type is supported by the Async Clipboard API.
// `type` refers to a MIME type or a custom MIME type with a "web " prefix.
// Spec: https://w3c.github.io/clipboard-apis/#dom-clipboarditem-supports
```
This appears to be an accidental deletion during code movement.
**Suggestion**: Restore the documentation comment above `static bool supports(const String& type);`.

### Issue #12: `ReadRepresentationFromClipboard` silently drops callback when no execution context
**Severity**: Minor
**File**: clipboard_promise.cc
**Lines**: 353-369
**Description**: When `GetExecutionContext()` returns null at line 358, the method returns early without executing or reporting the failure to the callback. The stored `ScriptPromiseResolver` in `ClipboardItem::representations_with_resolvers_` will never be resolved or rejected, leaving the promise permanently pending. This can cause memory leaks and hung UI.
**Suggestion**: Invoke the callback with `nullptr` blob before returning to properly reject the promise:
```cpp
if (!GetExecutionContext()) {
  std::move(callback).Run(format, nullptr);
  return;
}
```

### Issue #13: Test creates duplicate `DummyPageHolder` and `MockClipboardHostProvider`
**Severity**: Minor
**File**: clipboard_unittest.cc
**Lines**: 95-100
**Description**: The `ClipboardTest` constructor creates its own `DummyPageHolder` and `MockClipboardHostProvider`, but `PageTestBase` (the base class) already creates a `DummyPageHolder` in `SetUp()`. The tests then use both `GetFrame()` (from `PageTestBase`) for writing clipboard data and `mock_clipboard_host()` (from the custom provider) for verification. These may reference different clipboard hosts, potentially making the test assertions unreliable.
**Suggestion**: Verify that both providers bind to the same clipboard instance, or use `PageTestBase`'s built-in infrastructure exclusively. Consider overriding `SetUp()` instead of using the constructor.

### Issue #14: `ClipboardPromise` may be garbage-collected while async reads are pending
**Severity**: Major
**File**: clipboard_promise.cc / clipboard_item.cc
**Description**: After `ResolveRead()` resolves the outer promise at line 433, the `ClipboardPromise` object's only strong reference from GC roots is via `ClipboardItem::clipboard_promise_`. However, `ClipboardItem` stores the `ClipboardPromise` as a `Member<ClipboardPromise>`, which means the `ClipboardPromise` is traceable. But the `read_callbacks_` map uses `base::OnceCallback` which is NOT traced by GC (it's a `HashMap`, not `HeapHashMap`). The callbacks contain `WrapPersistent(this)` references to `ClipboardItem`, which should prevent premature GC. However, the `ClipboardReader` objects created in `ReadRepresentationFromClipboardReader` are `MakeGarbageCollected` but not stored in any member—they are only kept alive by the system clipboard's async callback. If there's a GC cycle between creating the reader and the callback completing, the reader could be collected.
**Suggestion**: Store a reference to the active `ClipboardReader` in the `ClipboardPromise` to prevent premature garbage collection, similar to how `clipboard_writer_` is stored.

### Issue #15: `CaptureTelemetry` not called in lazy-read path
**Severity**: Minor
**File**: clipboard_item.cc
**Lines**: 176-199
**Description**: When `ClipboardItemGetTypeCounterEnabled()` and `ClipboardReadOnDemandEnabled()` are both enabled, the lazy-read path in `getType()` never calls `CaptureTelemetry()`. The non-lazy path calls it at line 166-167. This means the `getType` counter telemetry will be lost when lazy reading is enabled.
**Suggestion**: Add `CaptureTelemetry` call in the lazy-read path:
```cpp
if (mime_types_.Contains(type)) {
  if (RuntimeEnabledFeatures::ClipboardItemGetTypeCounterEnabled()) {
    CaptureTelemetry(ExecutionContext::From(script_state), type);
  }
  // ... rest of the lazy-read code
}
```

## Positive Observations

- **Good use of feature flags**: The `ClipboardReadOnDemandEnabled` runtime feature flag allows gradual rollout and easy rollback.
- **Sequence number validation**: The clipboard change detection via sequence numbers is a solid security measure that prevents TOCTOU attacks between `read()` and `getType()`.
- **Test coverage for lazy loading**: The `ReadOnlyMimeTypesInClipboardRead` test directly verifies that data reading methods are NOT called during `read()`, which is an excellent way to validate the lazy loading behavior.
- **Mock infrastructure**: Adding call tracking to `MockClipboardHost` is clean and reusable for future tests.
- **Callback-per-MIME-type design**: Using `read_callbacks_` map keyed by MIME type correctly handles concurrent `getType()` calls for different formats.
- **Web test for change detection**: The web test validates the user-facing behavior of clipboard change detection end-to-end.

## Overall Assessment

**Needs changes before approval**

The CL implements a reasonable lazy-read architecture for the Clipboard API, but has several issues that need to be addressed:

1. **Two critical issues**: Undefined methods (`SetClipboardPromise` and `Create(HeapVector<String>)`) that will cause linker errors if referenced.
2. **Several major issues**: Double exception throwing in `getType()`, missing null checks in async paths, incorrect UMA histogram, and potential GC safety concerns.
3. **Multiple minor issues**: Wrong include, missing documentation, telemetry gaps.

The overall design is sound—deferring clipboard data reads to `getType()` time is a good optimization. However, the implementation needs more robust error handling for the asynchronous code paths, and the dead code declarations should be removed.
