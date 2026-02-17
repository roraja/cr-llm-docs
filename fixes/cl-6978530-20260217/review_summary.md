# CL 6978530 Review Summary: [Clipboard] Implementation of Lazy Read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal (shwetabindal@microsoft.com)
**Status:** NEW | **Branch:** main
**Files Changed:** 10 (+447/-43 lines)
**Bug:** [435051711](https://crbug.com/435051711)

---

## 1. Executive Summary

This CL implements lazy (on-demand) clipboard reading for the Async Clipboard API in Blink's renderer. Instead of eagerly reading all clipboard data formats upfront during `clipboard.read()`, the new implementation defers actual data reading until `ClipboardItem.getType()` is called for a specific MIME type. This is gated behind a new `ClipboardReadOnDemand` runtime-enabled feature flag (currently in "test" status), and includes clipboard change detection via sequence numbers to ensure data consistency between the initial `read()` call and subsequent `getType()` calls.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 3 | The dual-path logic (lazy vs eager) with feature flag checks scattered across multiple files makes the flow harder to follow. The use of `sequence_number_ == 0` as a sentinel is non-obvious. |
| Maintainability | 2 | Heavy use of `RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()` branching in every file creates parallel code paths that must be maintained in sync. 14+ feature flag checks across 5 files. |
| Extensibility | 3 | The callback-based lazy loading mechanism is reasonable, but the tight coupling between `ClipboardItem` and `ClipboardPromise` (circular dependency) limits future refactoring. |
| Consistency | 3 | Follows existing Chromium patterns (feature flags, GC tracing, mojo), but introduces inconsistencies: new constructor pattern vs existing `Create()` factory, mixed use of `HashMap` and `HeapHashMap`. |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    JavaScript (Web Page)                         │
│  const items = await navigator.clipboard.read();                │
│  const blob = await items[0].getType("text/plain");             │
└────────────┬───────────────────────────────┬────────────────────┘
             │ read()                        │ getType()
             ▼                               ▼
┌────────────────────────┐    ┌──────────────────────────────────┐
│   ClipboardPromise     │◄───│       ClipboardItem              │
│                        │    │                                  │
│ • CreateForRead()      │    │ • mime_types_ (lazy)             │
│ • OnReadAvailable-     │    │ • representations_ (eager)       │
│   FormatNames()        │    │ • representations_with_resolvers_│
│ • ReadRepresentation-  │    │ • sequence_number_               │
│   FromClipboard()      │    │ • clipboard_promise_ (back-ref)  │
│ • OnRead(blob, mime)   │    │                                  │
│ • read_callbacks_      │    │ • getType() → triggers lazy read │
│ • GetSequenceNumber-   │    │ • CheckSequenceNumber()          │
│   Token()              │    │ • ResolveFormatData()            │
└────────┬───────────────┘    └──────────────────────────────────┘
         │
         ▼
┌────────────────────────┐    ┌──────────────────────────────────┐
│   ClipboardReader      │───►│     SystemClipboard (mojo)       │
│                        │    │                                  │
│ • PNG / Text / HTML /  │    │ • ReadText()                     │
│   SVG / Custom readers │    │ • ReadHtml()                     │
│ • OnRead() → callback  │    │ • ReadAvailableCustomAnd-        │
│   to ClipboardPromise  │    │   StandardFormats()              │
│                        │    │ • SequenceNumber()               │
└────────────────────────┘    └──────────────────────────────────┘

LAZY READ FLOW:
read() ──► enumerate formats only ──► resolve with ClipboardItem(mime_types)
                                                    │
getType("text/plain") ──► CheckSequenceNumber() ──► ReadRepresentationFromClipboard()
                                                    │
                              ──► ClipboardReader ──► SystemClipboard.ReadText()
                                                    │
                              ──► OnRead(blob, mime) ──► callback ──► ResolveFormatData()
                                                                        │
                                                              resolve ScriptPromise<Blob>
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 2 | Multiple bugs: double `NotFoundError` exception throw, missing `return` after first throw, no telemetry in lazy path, `read_callbacks_` not GC-traced, null blob error message misleading, missing `ContextDestroyed` cleanup for pending lazy read promises. |
| Efficiency | 4 | The core optimization (deferring data reads) is sound and should reduce unnecessary IPC for unused MIME types. |
| Readability | 3 | Reasonable naming and structure, but the dual-path branching and `sequence_number_ == 0` sentinel reduce clarity. |
| Test Coverage | 3 | Two new unit tests and one web test, but missing important scenarios: concurrent `getType()` calls for different types, context destruction during lazy read, re-calling `getType()` for same type, unsupported type handling. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Double exception throw in `getType()` (clipboard_item.cc:196-204):** When the lazy path is active, `clipboard_promise_` is set, and the type is not in `mime_types_`, the code throws `NotFoundError` at line 197-198 but does NOT return. Execution falls through to lines 201-203 which throw `NotFoundError` again. Throwing two exceptions on the same `ExceptionState` is undefined behavior in Blink and will likely crash.

   ```cpp
   // Line 196-199: throws but doesn't return
   } else {
     exception_state.ThrowDOMException(DOMExceptionCode::kNotFoundError,
                                       "The type was not found");
   }
   // Falls through to line 201-203: throws AGAIN
   exception_state.ThrowDOMException(DOMExceptionCode::kNotFoundError,
                                     "The type was not found");
   return ScriptPromise<Blob>();
   ```

2. **`read_callbacks_` HashMap is not GC-traced (clipboard_promise.h:220-221):** The `read_callbacks_` map stores `base::OnceCallback` objects that capture `WrapPersistent(this)` pointers to `ClipboardItem`. Since this is a plain `HashMap` (not `HeapHashMap`), the GC cannot see these persistent handles. While `WrapPersistent` prevents premature collection, the callbacks themselves aren't cleaned up in `ContextDestroyed()`, creating potential use-after-free if the context is destroyed while a lazy read is in-flight.

3. **`SetClipboardPromise()` declared in header but never defined (clipboard_item.h:84):** The method `SetClipboardPromise(ClipboardPromise*, absl::uint128)` is declared in the header but has no implementation in the `.cc` file. This will cause a linker error if any code tries to call it.

4. **`Create()` factory method for lazy path declared but not implemented (clipboard_item.h:43-44):** `static ClipboardItem* Create(const HeapVector<String>& mime_types, ExceptionState&)` is declared in the header but not defined in the `.cc` file. Dead code that will cause linker errors if referenced.

### Major Issues (Should Fix)

1. **Missing telemetry in lazy read path:** When `ClipboardReadOnDemand` is enabled, `getType()` bypasses the `CaptureTelemetry()` call entirely (only called in the eager path at line 166-167). This means `ClipboardItemGetTypeCounter` metrics will not be recorded for lazy reads, making it impossible to track adoption and usage patterns.

2. **Misleading error message when blob is null (clipboard_item.cc:142-145):** When `ResolveFormatData()` receives a null blob (meaning the read failed), it rejects with "Clipboard data has changed." However, a null blob typically means the clipboard data couldn't be read or is empty — not that it changed. This is a separate condition from the sequence number check on line 149.

3. **No cleanup of pending promises on `ContextDestroyed()` (clipboard_promise.cc:832-838):** When the execution context is destroyed, `ContextDestroyed()` rejects the main promise but does not reject any pending per-type promises stored in `ClipboardItem::representations_with_resolvers_`. These orphaned resolvers may leak or cause UAF.

4. **Circular reference between `ClipboardItem` and `ClipboardPromise`:** `ClipboardItem` holds a `Member<ClipboardPromise>` (clipboard_promise_), and `ClipboardPromise` creates `ClipboardItem` objects. While this won't cause GC leaks (Oilpan handles cycles), it creates a tight coupling that makes the ownership model unclear and complicates reasoning about object lifetimes.

5. **`ClipboardTest` constructor creates redundant `DummyPageHolder` (clipboard_unittest.cc:95-99):** `PageTestBase` (the parent class) already creates a `DummyPageHolder` internally. The test constructor creates a second one (`page_holder_`), which means there are two competing page contexts. This could lead to subtle test flakiness.

### Minor Issues (Nice to Fix)

1. **`sequence_number_ == 0` used as sentinel (clipboard_item.cc:163):** Using `0` as a sentinel value for "not a lazy read item" is fragile. A sequence number could theoretically be 0 in edge cases. Consider using `std::optional<absl::uint128>` or a separate boolean flag `is_lazy_read_` for clarity.

2. **Redundant `.at()` lookups in `ResolveFormatData()` (clipboard_item.cc:143, 150, 156):** The same key is looked up 2-3 times. Store the iterator from `find()` and reuse it.

3. **Missing newline at end of web test file (async-clipboard-lazy-read.html):** The file is missing a trailing newline, which is a Chromium style requirement.

4. **Comment on `supports()` method removed (clipboard_item.h:72-75):** The useful doc-comment for the `supports()` static method was deleted in this CL. This appears accidental — comments should be preserved.

5. **Inconsistent error handling in `ReadRepresentationFromClipboard()` (clipboard_promise.cc:358-359):** When `GetExecutionContext()` returns null, the method silently returns without executing the callback. The caller's promise will never resolve or reject, leaving it permanently pending.

### Suggestions (Optional)

1. **Consider using a strategy/policy pattern** instead of scattering `RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()` checks across 5 files. A `ClipboardReadStrategy` base class with `EagerClipboardReadStrategy` and `LazyClipboardReadStrategy` subclasses would isolate the logic and make it easier to eventually remove the feature flag.

2. **Add DCHECK for `clipboard_promise_` in lazy path:** In `getType()`, when `sequence_number_ != 0` but `clipboard_promise_` is null, the code falls through to throw `NotFoundError` even for valid types. Add a DCHECK or handle this edge case explicitly.

3. **Consider adding histogram metrics** for lazy read latency (time between `read()` and `getType()`) and clipboard change detection frequency to inform future optimization decisions.

4. **Web test could be more comprehensive:** The current web test only tests clipboard change detection. Add tests for: successful lazy read, multiple `getType()` calls, and unsupported type handling.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | What it covers |
|------|---------------|
| `ReadOnlyMimeTypesInClipboardRead` (unit) | Verifies that `CreateForRead` only calls `ReadAvailableCustomAndStandardFormats` and NOT `ReadText`/`ReadHtml` — proving lazy loading works |
| `ClipboardItemGetTypeTest` (unit) | Verifies that `getType("text/plain")` on a lazy-read `ClipboardItem` successfully triggers `ReadText` on the mock clipboard host |
| `async-clipboard-lazy-read.html` (web test) | Verifies that `getType()` throws `DataError` when clipboard content changes between `read()` and `getType()` |

### Missing Tests

| Gap | Priority | Description |
|-----|----------|-------------|
| Concurrent `getType()` calls | High | Call `getType("text/plain")` and `getType("text/html")` simultaneously — tests the `read_callbacks_` map and concurrent reader handling |
| `getType()` with unsupported type | High | Call `getType("application/octet-stream")` when it's not in `mime_types_` — should throw `NotFoundError` (currently double-throws due to bug) |
| Context destruction during lazy read | High | Destroy the execution context while a `getType()` promise is pending — tests cleanup and prevents UAF |
| Repeated `getType()` for same type | Medium | Call `getType("text/plain")` twice — second call should return the cached resolver's promise |
| `getType()` for image/png | Medium | Test lazy read of binary data (PNG), not just text |
| Feature flag disabled | Medium | Verify the eager path is unchanged when `ClipboardReadOnDemand` is disabled |
| Empty clipboard | Low | Call `read()` on an empty clipboard with lazy read enabled |
| Web custom format ("web " prefix) | Medium | Test lazy read with custom MIME types |

### Recommended Additional Tests

1. **Stress test:** Multiple rapid `getType()` calls to different MIME types in quick succession
2. **Sequence number edge case:** Test behavior when sequence number is 0 (if theoretically possible)
3. **Permission revocation:** Test behavior if clipboard permission is revoked between `read()` and `getType()`

---

## 6. Security Considerations

### Clipboard Data Exposure
- **Positive:** Lazy reading reduces the window during which sensitive clipboard data (passwords, tokens) is held in renderer memory. Data is only read when explicitly requested, which is a security improvement.
- **Risk:** The `ClipboardPromise` object persists longer (until all `getType()` calls complete), maintaining access to the system clipboard. If the renderer process is compromised, the attacker has a longer window to call `getType()` on stale `ClipboardItem` objects.

### Sequence Number Bypass
- **Risk (Low):** The sequence number check in `CheckSequenceNumber()` relies on the `SystemClipboard` mojo interface. A compromised renderer could potentially bypass this check. However, this is within the existing trust model — the browser process enforces the actual clipboard access permissions.

### Stale Clipboard References
- **Risk (Medium):** `ClipboardItem` objects holding `clipboard_promise_` references keep the promise (and its access to the clipboard IPC channel) alive longer than necessary. If a web page stores `ClipboardItem` references indefinitely, the `ClipboardPromise` remains alive. Consider adding a timeout or invalidation mechanism.

### Recommendations
1. Add a TTL (time-to-live) for lazy `ClipboardItem` objects — reject `getType()` if called too long after `read()`
2. Clear `clipboard_promise_` reference after all known `mime_types_` have been resolved to reduce the attack surface
3. Ensure the feature flag stays in "test" status until the security implications are fully reviewed by the security team

---

## 7. Performance Considerations

### Positive Impact
- **Reduced IPC:** The primary benefit — `clipboard.read()` no longer reads all data formats eagerly. For pages that only need one type (e.g., `text/plain`), this eliminates unnecessary reads of HTML, PNG, SVG, and custom formats.
- **Reduced memory:** Blob objects for unused formats are never created, reducing peak memory usage.
- **Lower latency for `read()`:** The `read()` call resolves faster since it only enumerates formats without reading data.

### Potential Concerns
- **`getType()` latency increase:** Each `getType()` call now triggers a fresh IPC round-trip to the browser process. For pages that always read all types, the total latency may be higher than the eager approach (sequential IPC calls vs. batched reads).
- **No batching of `getType()` calls:** If a page calls `getType("text/plain")` and `getType("text/html")` simultaneously, they create separate `ClipboardReader` instances and separate IPC calls. Consider batching if this pattern is common.
- **Posted task overhead:** `ReadRepresentationFromClipboard()` posts a task via `PostTask()` to read data. This adds one extra task queue hop compared to the eager path.

### Benchmarking Recommendations
1. Measure `clipboard.read()` → `getType()` round-trip latency vs. eager `clipboard.read()` for single-type and multi-type scenarios
2. Profile memory usage difference for pages that call `read()` but never call `getType()`
3. Test with large clipboard payloads (large images) to measure the benefit of deferred reads

---

## 8. Final Recommendation

**Verdict:** NEEDS_WORK

**Rationale:**

The overall design approach — deferring clipboard data reads to `getType()` time — is sound and well-motivated. The feature flag gating and sequence number-based change detection are appropriate architectural choices. However, the implementation has several correctness bugs that must be fixed before this CL can land:

1. The **double exception throw** in `getType()` is a crash-causing bug that must be fixed.
2. The **unimplemented `SetClipboardPromise()` and `Create()` methods** will cause linker errors if referenced.
3. The **missing cleanup in `ContextDestroyed()`** creates potential use-after-free scenarios.
4. The **missing telemetry** in the lazy path is a data regression.

The test coverage, while a good start, needs to cover more edge cases — particularly concurrent `getType()` calls and context destruction scenarios.

**Action Items for Author:**

1. **Fix the double exception throw** in `ClipboardItem::getType()` — add `return ScriptPromise<Blob>();` after the first throw at line 198, or restructure the control flow.
2. **Remove or implement** `SetClipboardPromise()` and the second `Create()` overload — if unused, remove from the header.
3. **Add cleanup logic** in `ClipboardPromise::ContextDestroyed()` to reject pending per-type promises and clear `read_callbacks_`.
4. **Add `CaptureTelemetry()` call** in the lazy read path of `getType()`.
5. **Fix the null blob error message** in `ResolveFormatData()` to distinguish between "read failed" and "clipboard changed."
6. **Add missing tests** for concurrent `getType()` calls, context destruction, and unsupported type handling.
7. **Add trailing newline** to `async-clipboard-lazy-read.html`.
8. **Restore the removed comment** on the `supports()` method declaration.
9. **Handle the silent failure** in `ReadRepresentationFromClipboard()` when `GetExecutionContext()` is null — reject the callback.
10. **Consider redundant `DummyPageHolder`** in test constructor — verify it's actually needed or remove.

---

## 9. Comments for Gerrit

### File: `third_party/blink/renderer/modules/clipboard/clipboard_item.cc`

**Line 196-204 (getType method) — CRITICAL BUG:**
```
The `else` branch at line 196 throws `NotFoundError` but does NOT return,
causing execution to fall through to lines 201-203 which throw a second
`NotFoundError`. Double-throwing on ExceptionState is undefined behavior
in Blink and will likely crash.

Fix: Add `return ScriptPromise<Blob>();` after the throw at line 198, or
restructure the if/else to have a single throw point at the end.
```

**Line 142-145 (ResolveFormatData) — Misleading error:**
```
When blob is null, the error message says "Clipboard data has changed"
but a null blob typically means the clipboard data couldn't be read (e.g.,
empty clipboard or encoding error), not that it changed. Consider a more
accurate message like "Failed to read clipboard data" to distinguish from
the sequence number check on line 149.
```

**Line 163 (getType) — Sentinel value concern:**
```
Using `sequence_number_ == 0` as a sentinel for "not a lazy-read item" is
fragile. Consider using `std::optional<absl::uint128>` or a dedicated
`is_lazy_read_` boolean for clarity and correctness.
```

**Line 187-195 (getType lazy path) — Missing telemetry:**
```
The lazy read path doesn't call `CaptureTelemetry()`, which means
`ClipboardItemGetTypeCounter` metrics won't be recorded when the feature
is enabled. This will create a data gap in usage metrics.
```

### File: `third_party/blink/renderer/modules/clipboard/clipboard_item.h`

**Line 84 — Dead declaration:**
```
`SetClipboardPromise()` is declared here but never defined in the .cc file.
This will cause a linker error if called. Either implement it or remove
the declaration.
```

**Line 43-44 — Dead declaration:**
```
`static ClipboardItem* Create(const HeapVector<String>&, ExceptionState&)`
is declared but never defined. Remove if unused.
```

**Line 72-75 — Removed comment:**
```
The doc-comment for `supports()` was removed. This appears accidental —
please restore it.
```

### File: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`

**Line 358-359 (ReadRepresentationFromClipboard) — Silent failure:**
```
When `GetExecutionContext()` is null, this returns without invoking the
callback. This leaves the caller's ScriptPromiseResolver permanently
pending (never resolved or rejected). Consider invoking the callback with
`nullptr` to allow proper error handling, or reject the resolver directly.
```

### File: `third_party/blink/renderer/modules/clipboard/clipboard_promise.h`

**Line 220-221 — Missing cleanup:**
```
`read_callbacks_` stores `OnceCallback`s that capture `WrapPersistent`
pointers. These are not cleaned up in `ContextDestroyed()`. If the context
is destroyed while a lazy read is in-flight, the pending callbacks and
their captured persistent handles may leak. Add cleanup in
`ContextDestroyed()` to clear `read_callbacks_` and reject any pending
per-type resolvers.
```

### File: `third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`

**Line 95-99 (ClipboardTest constructor) — Redundant PageHolder:**
```
`PageTestBase` already creates a `DummyPageHolder` internally. This
constructor creates a second one. Is this intentional? If so, add a
comment explaining why. If not, consider using the parent's page holder
to avoid having two competing page contexts.
```

**General — Missing test coverage:**
```
Please add tests for:
1. Concurrent getType() calls for different MIME types
2. Context destruction while a getType() promise is pending
3. getType() with a type not in mime_types_ (unsupported type)
4. Repeated getType() calls for the same type (should reuse resolver)
```

### File: `third_party/blink/web_tests/clipboard/async-clipboard/async-clipboard-lazy-read.html`

**Line 36 — Missing trailing newline:**
```
nit: Add a trailing newline at end of file per Chromium style.
```
