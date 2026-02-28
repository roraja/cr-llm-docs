# CL 6978530 — Review Summary

**CL Title:** [Clipboard] Implement on-demand reading in getType()
**Author:** Shweta Bindal (shwetabindal@microsoft.com)
**URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Status:** NEW
**Bug:** [435051711](https://crbug.com/435051711)
**Files Changed:** 20 files, +929 / −121 lines

---

## 1. Executive Summary

This CL implements lazy (on-demand) clipboard data reading for the Async Clipboard API. Previously, `clipboard.read()` eagerly fetched all clipboard data upfront; this change defers actual OS-level data reading to the `ClipboardItem.getType()` call so that only requested formats are read, improving performance when a web page only needs a subset of available clipboard formats. A clipboard change detection mechanism is also introduced, rejecting with `DataError` if clipboard contents change between `read()` and `getType()`.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|:------------:|----------|
| Clarity | 4 | Clear separation of lazy vs. eager paths via runtime feature flag; well-documented commit message and design doc reference. |
| Maintainability | 3 | Dual code paths (lazy vs. eager) guarded by `RuntimeEnabledFeatures` checks are scattered throughout; once the feature ships stable these should be collapsed. |
| Extensibility | 4 | The `ClipboardReaderResultHandler` interface abstraction makes it straightforward to add new read targets beyond `ClipboardPromise` and `ClipboardItem`. |
| Consistency | 4 | Follows existing Chromium/Blink patterns: `GarbageCollectedMixin`, `ExecutionContextLifecycleObserver`, `ScriptPromiseResolver`, runtime feature flags. |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        Web Page (JS)                             │
│  clipboard.read() ──► ClipboardItem[] ──► item.getType(mime)     │
└─────────┬─────────────────────────────────────────┬──────────────┘
          │                                         │
          ▼                                         ▼
┌─────────────────────┐               ┌──────────────────────────┐
│   ClipboardPromise   │               │     ClipboardItem        │
│ (read orchestrator)  │               │  (lazy read holder)      │
│                      │               │                          │
│ • read(): fetches    │               │ • getType(): creates     │
│   MIME type list     │               │   ScriptPromiseResolver, │
│   only (lazy path)   │               │   calls ClipboardReader  │
│ • Creates            │               │ • Checks sequence number │
│   ClipboardItem with │               │   for change detection   │
│   mime_types_ only   │               │ • Implements             │
│                      │               │   ClipboardReaderResult- │
│                      │               │   Handler interface      │
└─────────┬────────────┘               └──────────┬───────────────┘
          │                                       │
          │  Both implement                       │
          │  ClipboardReaderResultHandler          │
          │                                       │
          ▼                                       ▼
┌────────────────────────────────────────────────────────────────┐
│                     ClipboardReader                             │
│  (format-specific subclasses: Png, Text, Html, Svg, Custom)    │
│                                                                │
│  • Accepts ClipboardReaderResultHandler* (was ClipboardPromise*)│
│  • Reads from SystemClipboard                                  │
│  • Calls result_handler_->OnRead(blob, mime_type)              │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   SystemClipboard     │
                    │  (Mojo IPC to        │
                    │   ClipboardHost)      │
                    └──────────────────────┘
```

**Key design decisions:**
1. **Feature-flagged:** `ReadClipboardDataOnClipboardItemGetType` (status: `"test"`) gates the new behavior.
2. **Interface extraction:** `ClipboardReaderResultHandler` replaces the direct `ClipboardPromise*` dependency in `ClipboardReader`, enabling both `ClipboardPromise` (eager/legacy path) and `ClipboardItem` (lazy path) to receive read results.
3. **Clipboard change detection:** Uses `SystemClipboard::SequenceNumber()` to detect if clipboard contents changed between `read()` and `getType()`.
4. **Lifecycle management:** `ClipboardItem` now observes `ExecutionContext` lifecycle to reject pending promises if the document is detached.

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|:------------:|----------|
| Correctness | 4 | Logic is sound: lazy read, change detection via sequence numbers, proper lifecycle handling. Minor concerns around edge cases noted below. |
| Efficiency | 4 | Achieves the primary goal — avoids unnecessary OS clipboard reads. Promise caching for repeated `getType()` calls is well-implemented. |
| Readability | 3 | Many `RuntimeEnabledFeatures::ReadClipboardDataOnClipboardItemGetTypeEnabled()` checks make control flow harder to follow. Method naming is clear. |
| Test Coverage | 4 | Good coverage: unit tests for lazy loading verification and getType data retrieval; web tests for change detection edge cases and deferred OS calls. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

*None identified.* The core logic is correct and well-tested.

### Major Issues (Should Fix)

1. **Potential null dereference in `GetSystemClipboard()` callers within `ClipboardPromise`:** In `HandleRead()` and `ResolveRead()`, `GetSystemClipboard()` is called without null-checking the return value. While `GetLocalFrame()` has a `DCHECK` that it's non-null, the new `GetSystemClipboard()` helper could return `nullptr` if `GetLocalFrame()` is null. Several call sites like `GetSystemClipboard()->SequenceNumber()` in `ResolveRead()` would crash. The legacy path used `GetLocalFrame()->GetSystemClipboard()` which had the same risk, but adding a proper helper should include null safety.

2. **`ClipboardItem` constructor passes `nullptr` to `ExecutionContextLifecycleObserver` in the eager path:** The existing constructor now initializes `ExecutionContextLifecycleObserver(nullptr)`. While this works because `ContextDestroyed()` won't be called without an execution context, it means `GetExecutionContext()` returns `nullptr` for non-lazy `ClipboardItem`s. The `GetExecutionContext()` override unconditionally delegates to the lifecycle observer, which could cause issues if any code path calls it on a non-lazy item. Consider adding a guard or documenting this invariant more explicitly.

3. **Repeated map lookups in `ResolveFormatData()`:** The method calls `representations_with_resolvers_.find(mime_type)` followed by multiple `.at(mime_type)` calls. Use the iterator from `find()` to avoid redundant hash lookups:
   ```cpp
   auto it = representations_with_resolvers_.find(mime_type);
   if (it == representations_with_resolvers_.end()) return;
   if (HasClipboardChangedSinceClipboardRead()) {
     it->value->Reject(...);
     return;
   }
   ```

### Minor Issues (Nice to Fix)

1. **Inconsistent mock clipboard host tracking between `content/test/` and `blink/core/testing/`:** The `content/test/mock_clipboard_host.h` tracks `read_unsanitized_custom_format_called_` while the blink-side `mock_clipboard_host.h` does not. This asymmetry could confuse future test authors. Consider aligning the tracking capabilities.

2. **Extra blank line in `types()` method:** There's a trailing blank line inside the `if (is_lazy_read_)` branch before the `} else {` that is inconsistent with Chromium style.

3. **Missing `const` on `HasClipboardChangedSinceClipboardRead()`:** This method doesn't modify state and should be `const`:
   ```cpp
   bool HasClipboardChangedSinceClipboardRead() const;
   ```

4. **Comment missing period in blink mock:** `// Reset call tracking` should end with a period per Chromium style: `// Reset call tracking.`

### Suggestions (Optional)

1. **Add a TODO for feature flag cleanup:** Once `ReadClipboardDataOnClipboardItemGetType` reaches stable, the dual-path code in `clipboard_item.cc`, `clipboard_promise.cc`, and `clipboard_promise.h` should be collapsed. Consider adding explicit `// TODO(crbug.com/435051711)` markers at each branch point.

2. **Consider using `base::expected` or error enum instead of multiple `DOMException` string literals:** The error messages "Clipboard data has changed", "Failed to read clipboard data." are duplicated. Extract these as constants to avoid drift.

3. **Telemetry timing for lazy vs. eager:** The `CaptureTelemetry` call in `getType()` for the lazy path captures timing relative to `ClipboardItem` creation. Consider adding a histogram distinguishing lazy vs. eager reads to measure the actual performance improvement.

4. **`ClipboardReader` no longer includes `clipboard_promise.h`:** This is a clean decoupling. Consider adding a brief comment in `clipboard_reader.h` noting that `ClipboardReaderResultHandler` is the callback interface replacing the former direct `ClipboardPromise` dependency.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test File | Type | Coverage |
|-----------|------|----------|
| `clipboard_unittest.cc` | C++ unit test | +167 lines. Tests: `ReadOnlyMimeTypesInClipboardRead` verifies lazy loading (no `ReadText`/`ReadHtml` during `read()`); `ClipboardItemGetTypeTest` verifies `getType()` triggers actual data read and returns correct blob. |
| `async-clipboard-lazy-read.html` | Web test | Tests clipboard change detection — `getType()` throws `DataError` after clipboard changes. |
| `async-clipboard-lazy-read-deferred-os-call.html` | Web test | Verifies OS read calls are deferred to `getType()` for both `text/plain` and custom formats using `testRunner.getClipboardReadState()`. |
| `async-clipboard-lazy-read-change-detection-edge-cases.html` | Web test | Edge cases: rewriting same data invalidates read; removing format invalidates read; previously successful `getType()` throws after clipboard change. |

### Missing Tests / Recommended Additions

1. **PNG/SVG lazy read test:** No web test or unit test specifically validates lazy reading for `image/png` or `image/svg+xml` formats through `getType()`.

2. **Concurrent `getType()` calls:** The promise caching path (`has_cached_resolver`) should be tested — calling `getType("text/plain")` twice before the first resolves should return the same promise.

3. **`ContextDestroyed` rejection test:** No test verifies that pending `getType()` promises are correctly rejected with `NotAllowedError` when the document is detached/destroyed.

4. **Feature flag disabled path regression test:** Ensure the existing (eager) behavior remains correct when `ReadClipboardDataOnClipboardItemGetType` is disabled.

5. **Error path test:** `ReadRepresentationFromClipboardReader` with a null `SystemClipboard` (e.g., detached frame) should reject the promise correctly.

6. **`getType()` for unsupported type on lazy item:** Calling `getType("application/unsupported")` on a lazy `ClipboardItem` should throw `NotFoundError`.

---

## 6. Security Considerations

1. **Clipboard change detection is a positive security feature:** By checking sequence numbers, the CL prevents stale clipboard data from being silently returned, which could lead to data confusion attacks where an attacker swaps clipboard content between `read()` and `getType()`.

2. **Permission model unchanged:** The lazy read path still goes through the same permission checks via `ClipboardPromise::CreateForRead()` before any data is surfaced. No permission bypass is introduced.

3. **Detached document handling:** `ContextDestroyed()` properly rejects pending promises, preventing data leaks to detached frames. This is correct behavior.

4. **No new IPC surface for data:** The lazy path reuses existing `SystemClipboard` Mojo calls (`ReadText`, `ReadHtml`, etc.). No new privileged operations are introduced — only the timing of existing operations changes.

**Recommendation:** No security concerns. The change improves the security posture through change detection.

---

## 7. Performance Considerations

1. **Primary performance win:** For pages that call `clipboard.read()` but only consume one format via `getType()`, this eliminates unnecessary OS clipboard reads for other formats (e.g., reading only `text/plain` avoids decoding `image/png`). This is especially impactful for large clipboard payloads (images, rich HTML).

2. **Potential overhead:** Each `getType()` call now incurs an additional `SequenceNumber()` Mojo IPC to verify clipboard hasn't changed. For pages that read all formats, this adds N extra IPC calls compared to the eager path. This is a reasonable trade-off since the common case is reading 1-2 formats.

3. **Promise caching:** The `representations_with_resolvers_` map ensures that multiple `getType()` calls for the same format don't trigger redundant clipboard reads. This is efficient.

4. **Benchmarking recommendations:**
   - Measure latency of `clipboard.read()` → `getType("text/plain")` vs. the old eager path when clipboard contains multiple formats (text + HTML + image).
   - Measure the overhead of the extra `SequenceNumber()` call in the lazy path.
   - Profile memory usage — `ClipboardItem` now holds additional state (`representations_with_resolvers_`, `mime_types_`, lifecycle observer registration).

---

## 8. Final Recommendation

**Verdict:** APPROVED_WITH_COMMENTS

**Rationale:**
This is a well-designed performance optimization that follows established Chromium patterns. The interface extraction (`ClipboardReaderResultHandler`) is a clean refactoring that decouples `ClipboardReader` from `ClipboardPromise`. The clipboard change detection and lifecycle management are correctly implemented. The feature is properly gated behind a runtime flag (`status: "test"`), allowing safe incremental rollout. Test coverage is good, with both unit tests and web tests covering the primary scenarios and edge cases. The issues identified are minor to moderate and do not block landing.

**Action Items for Author:**
1. Add null-safety checks for `GetSystemClipboard()` return value in `ClipboardPromise` callers (e.g., `ResolveRead()` line calling `GetSystemClipboard()->SequenceNumber()`).
2. Fix redundant map lookups in `ResolveFormatData()` — use iterator from `find()` instead of repeated `.at()` calls.
3. Mark `HasClipboardChangedSinceClipboardRead()` as `const`.
4. Add a test for concurrent `getType()` calls returning the same cached promise.
5. Add a test for `ContextDestroyed` rejecting pending `getType()` promises.
6. Consider adding TODO comments at `RuntimeEnabledFeatures` branch points for future cleanup when the flag ships stable.

---

## 9. Comments for Gerrit

### Overall Comment

> **LGTM with comments.**
>
> Great work on the lazy read implementation! The `ClipboardReaderResultHandler` interface extraction is a clean abstraction, and the change detection via sequence numbers is well thought out. The feature flag approach is appropriate for incremental rollout.
>
> A few items to address before landing — see inline comments.

### File-Specific Comments

**`third_party/blink/renderer/modules/clipboard/clipboard_item.cc` — `ResolveFormatData()`**

> nit: The `find()` result is discarded and then `.at(mime_type)` is called multiple times. Please use the iterator from `find()` to avoid redundant hash lookups:
> ```cpp
> auto it = representations_with_resolvers_.find(mime_type);
> if (it == representations_with_resolvers_.end()) return;
> // use it->value instead of .at(mime_type)
> ```

**`third_party/blink/renderer/modules/clipboard/clipboard_item.h` — `HasClipboardChangedSinceClipboardRead()`**

> nit: This method doesn't modify any member state — please mark it `const`.

**`third_party/blink/renderer/modules/clipboard/clipboard_promise.cc` — `ResolveRead()`**

> The call `GetSystemClipboard()->SequenceNumber()` could dereference null if the frame was detached between the permission check and reaching this point. Please add a null check or early return.

**`third_party/blink/renderer/modules/clipboard/clipboard_item.h` — `ExecutionContextLifecycleObserver(nullptr)` in eager constructor**

> Consider adding a brief comment explaining why `nullptr` is passed for the non-lazy path, e.g.:
> ```cpp
> // Non-lazy ClipboardItems don't need lifecycle observation since their data
> // is already resolved.
> ```

**`third_party/blink/renderer/modules/clipboard/clipboard_item.cc` — `getType()` lazy path**

> Optional: Consider adding a test for the promise caching behavior — calling `getType("text/plain")` twice should return the same promise without triggering a second `ClipboardReader::Read()`.

**`third_party/blink/renderer/core/testing/mock_clipboard_host.cc` — Reset comment**

> nit: `// Reset call tracking` → `// Reset call tracking.` (trailing period per Chromium style)

**`third_party/blink/renderer/platform/runtime_enabled_features.json5`**

> The feature is at `"test"` status, which is appropriate for initial rollout. Please add a TODO in the commit message or code to track when this should graduate to `"stable"` and the dual-path code should be cleaned up.
