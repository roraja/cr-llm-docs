# CL 6978530 Review Summary: [Clipboard] Implementation of Lazy Read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530  
**Author:** Shweta Bindal <shwetabindal@microsoft.com>  
**Status:** NEW  
**Files Changed:** 10 (+478/−50 lines)  
**Bug:** [435051711](https://crbug.com/435051711)

---

## 1. Executive Summary

This CL implements "lazy read" (on-demand reading) for the Async Clipboard API in the Blink renderer. Instead of eagerly reading all clipboard data when `navigator.clipboard.read()` is called, it defers the actual data read until `ClipboardItem.getType()` is invoked for a specific MIME type. The implementation adds clipboard change detection via sequence number comparison and is gated behind a new `ClipboardReadOnDemand` runtime-enabled feature flag.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 3 | The lazy read concept is clear, but extensive use of runtime feature flag branching across multiple files adds complexity and makes the code harder to follow. |
| Maintainability | 2 | Pervasive `if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled())` checks in 4+ files create significant maintenance burden. When the feature ships, all branches must be cleaned up. |
| Extensibility | 3 | The callback-based architecture allows future format extensions, but the tight coupling between `ClipboardItem` and `ClipboardPromise` limits flexibility. |
| Consistency | 3 | Follows existing Chromium patterns for feature flags and promise-based APIs but introduces a new lifecycle pattern (`ClipboardItem` holding a back-reference to `ClipboardPromise`) that differs from existing clipboard architecture. |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    navigator.clipboard.read()                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     ClipboardPromise                                 │
│  HandleReadWithPermission()                                         │
│    └─> OnReadAvailableFormatNames()                                 │
│         ├─ [lazy read OFF] → ReadNextRepresentation() → OnRead()    │
│         │                    → ResolveRead() (eager, all data read)  │
│         └─ [lazy read ON]  → item_mime_types_ populated             │
│                             → ResolveRead() (no data read yet)      │
│                                                                     │
│  ReadRepresentationFromClipboard(format, callback)                  │
│    └─> PostTask → ReadRepresentationFromClipboardReader(format)     │
│         └─> ClipboardReader::Read()                                 │
│              └─> OnRead(blob, mime_type)                            │
│                   └─> callback(mime_type, blob)                     │
│                        └─> ClipboardItem::ResolveFormatData()       │
└──────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      ClipboardItem                                   │
│  types()  → returns mime_types_ (lazy) or representations_ (eager)  │
│  getType(type)                                                      │
│    ├─ CheckSequenceNumber() — verify clipboard hasn't changed       │
│    ├─ Create ScriptPromiseResolver                                  │
│    ├─ ClipboardPromise::ReadRepresentationFromClipboard()            │
│    └─ ResolveFormatData(mime_type, blob) → Resolve/Reject promise   │
└──────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     ClipboardReader                                   │
│  (ClipboardImageReader, ClipboardTextReader, ClipboardHtmlReader,    │
│   ClipboardSvgReader, ClipboardCustomFormatReader)                   │
│  Read() → OnRead() → promise_->OnRead(blob, mime_type)              │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | Core logic is sound but has potential issues with lifetime management, race conditions on concurrent `getType()` calls, and missing error handling paths. |
| Efficiency | 4 | The lazy-read approach is genuinely more efficient — avoids reading unused clipboard data formats. |
| Readability | 2 | Deeply nested feature flag branching and the bidirectional `ClipboardItem ↔ ClipboardPromise` reference obscure the control flow. |
| Test Coverage | 3 | Unit tests cover the happy path and basic lazy-loading verification, but lack negative test cases and edge cases. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Lifetime / GC safety of `ClipboardPromise` back-reference**  
   `ClipboardItem` holds a `Member<ClipboardPromise> clipboard_promise_` which creates a strong reference cycle: `ClipboardPromise` creates and resolves `ClipboardItem`, and `ClipboardItem` holds back a reference to `ClipboardPromise`. After the original `clipboard.read()` promise resolves, the `ClipboardPromise` object is retained indefinitely because `ClipboardItem` (which may live arbitrarily long in JS) holds it. This prevents GC of the `ClipboardPromise` and everything it references (script_promise_resolver_, permission_service_, etc.). This could be a significant memory leak, especially if web pages read clipboard data and hold onto `ClipboardItem` objects.

2. **`read_callbacks_` HashMap uses non-GC `base::OnceCallback` with GC pointers**  
   (`clipboard_promise.h`, line 220-221): `HashMap<String, base::OnceCallback<void(const String&, Blob*)>> read_callbacks_` stores callbacks that capture `WrapPersistent(this)` (from `ClipboardItem::getType` at `clipboard_item.cc:200`). The `WrapPersistent` prevents GC of the `ClipboardItem` while the callback exists, but the `HashMap` itself is not traced by GC. If the `ClipboardPromise` is destroyed before the callback fires, the persistent handle becomes dangling. The `ContextDestroyed()` method clears `clipboard_reader_` but does not clear `read_callbacks_`, which could leave orphaned persistent handles.

3. **Missing MIME type validation in `ReadRepresentationFromClipboard`**  
   (`clipboard_promise.cc`): `ReadRepresentationFromClipboard` does not validate that the requested `format` is one of the original `item_mime_types_`. A malicious or buggy call path could request arbitrary formats from the system clipboard, bypassing the format enumeration that was done during `read()`.

### Major Issues (Should Fix)

1. **Concurrent `getType()` race condition for the same MIME type**  
   In `ClipboardItem::getType()` (line 185-187), if `getType("text/plain")` is called twice concurrently, the second call finds the resolver already in `representations_with_resolvers_` and returns the same promise. However, the first call's `ReadRepresentationFromClipboard` callback in `ResolveFormatData` will use `WrapPersistent(this)` with `BindOnce`, meaning only the first callback resolves the promise. This is technically correct but fragile — the comment at line 194 says "Create the promise resolver first, then store it" without explaining the re-entry guard. Consider adding an explicit comment or DCHECK.

2. **`ResolveFormatData` does not clean up `representations_with_resolvers_` after reject**  
   (`clipboard_item.cc`, lines 145-149, 152-156): When the promise is rejected (blob is null or sequence number mismatch), the resolver is left in `representations_with_resolvers_`. A subsequent `getType()` call for the same MIME type would return the already-rejected promise (line 185-187), which is incorrect behavior — the user would never get a fresh attempt.

3. **`PostTask` in `ReadRepresentationFromClipboard` without checking execution context validity**  
   (`clipboard_promise.cc`): The method checks `GetExecutionContext()` at the start, but between the check and the `PostTask` callback execution, the context could be destroyed. The posted task `ReadRepresentationFromClipboardReader` does check `GetExecutionContext()` but if the context is destroyed between `PostTask` and execution, the `WrapPersistent(this)` prevents GC of `ClipboardPromise`, causing a leak until the task runs.

4. **No timeout or expiry mechanism for lazy-read `ClipboardItem`**  
   A `ClipboardItem` created via lazy read holds the `ClipboardPromise` alive indefinitely. If a web page stores the `ClipboardItem` and calls `getType()` hours later, it will still attempt to read from the system clipboard. While `CheckSequenceNumber()` catches clipboard changes, there should be a consideration for a reasonable time-to-live.

### Minor Issues (Nice to Fix)

1. **Missing blank line before `// static` comment**  
   (`clipboard_item.cc`, line 219): The `// static` comment for `ClipboardItem::supports` is on the same line grouping as `CheckSequenceNumber()` above it, with no blank separator line. This makes it hard to visually separate the two methods.

2. **Inconsistent error messages**  
   `ResolveFormatData` uses "Clipboard data has changed" for both null blob and sequence number mismatch cases. A null blob could occur for other reasons (e.g., format not actually present despite being advertised). Consider differentiating the error messages.

3. **`HeapVector<String> item_mime_types_` in `ClipboardPromise` is redundant**  
   Both `ClipboardPromise` and `ClipboardItem` store MIME types. After `ResolveRead()`, `ClipboardPromise::item_mime_types_` is no longer needed since `ClipboardItem` has its own copy. Consider clearing it after construction.

4. **Web test (`async-clipboard-lazy-read.html`) only tests change detection**  
   The web test only verifies the clipboard-changed scenario (DataError). It doesn't test the happy path where `getType()` successfully returns data via lazy read. Add a basic success test.

5. **`clipboard_reader.cc` has 5 identical `if/else` blocks**  
   The pattern `if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()) { promise_->OnRead(blob, mime_type); } else { promise_->OnRead(blob); }` is repeated 5 times across different reader classes. Consider having the readers always pass the MIME type, and having the non-lazy `OnRead(Blob*)` overload be the one that ignores it, or refactoring the dispatch to a single helper.

### Suggestions (Optional)

1. **Consider using a WeakMember for the `ClipboardPromise` back-reference**  
   Instead of `Member<ClipboardPromise> clipboard_promise_` in `ClipboardItem`, use `WeakMember<ClipboardPromise>` to avoid preventing GC. Check for null before use in `getType()` and `CheckSequenceNumber()`, rejecting the promise if the `ClipboardPromise` has been collected.

2. **Consider a strategy pattern instead of runtime flag branching**  
   The pervasive `if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled())` checks could be replaced with a strategy/policy object that encapsulates the lazy vs. eager behavior, making the code cleaner and easier to remove when the feature ships.

3. **Add UMA histograms for lazy-read specific metrics**  
   Track how often lazy read is used vs. eager, how often clipboard change detection triggers DataError, and the typical time between `read()` and `getType()` calls under lazy read.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | Description | Adequacy |
|------|-------------|----------|
| `ReadOnlyMimeTypesInClipboardRead` | Verifies `clipboard.read()` with lazy read only calls `ReadAvailableCustomAndStandardFormats` and not `ReadText`/`ReadHtml` | ✅ Good — verifies lazy behavior |
| `ClipboardItemGetTypeTest` | Verifies `getType("text/plain")` triggers `ReadText` on the mock host | ✅ Good — verifies on-demand read |
| `async-clipboard-lazy-read.html` | Web test verifying DataError when clipboard changes between `read()` and `getType()` | ⚠️ Only tests error path |
| Existing tests (`ReadTextTest`, `ReadItemsTest`, `ReadItemsTestWithUnsanitized`) | Updated to use new `SetUp()` / `WritePlainTextToClipboard` API | ✅ Regression coverage maintained |

### Missing Tests

1. **Happy path web test**: Verify that `getType()` successfully returns clipboard data when clipboard hasn't changed.
2. **Multiple format lazy read**: Test `getType()` for multiple different MIME types (`text/plain`, `text/html`, `image/png`) from the same `ClipboardItem`.
3. **Concurrent `getType()` calls**: Test calling `getType("text/plain")` and `getType("text/html")` simultaneously.
4. **Duplicate `getType()` calls**: Test calling `getType("text/plain")` twice on the same `ClipboardItem`.
5. **Unsupported type with lazy read**: Test `getType("application/unsupported")` — should throw `NotFoundError`.
6. **Context destruction during lazy read**: Test behavior when the document is destroyed between `read()` and `getType()`.
7. **Feature flag disabled path**: Ensure existing eager-read behavior is unaffected when `ClipboardReadOnDemand` is disabled (existing tests partially cover this but not explicitly).
8. **Image format lazy read**: Test `getType("image/png")` via lazy read path.
9. **Custom format lazy read**: Test `getType("web text/custom")` via lazy read path.

---

## 6. Security Considerations

| Concern | Assessment | Severity |
|---------|------------|----------|
| **Clipboard data exposure window** | Lazy read increases the time window between permission grant and data access. The data is only read when `getType()` is called, potentially long after permission was granted. Clipboard content could change to sensitive data during this window. | Medium |
| **Sequence number validation** | `CheckSequenceNumber()` mitigates stale-read attacks, but the check happens *before* the async read completes. A TOCTOU (time-of-check-time-of-use) issue exists: clipboard could change between `CheckSequenceNumber()` and `ClipboardReader::Read()`. The post-read check in `ResolveFormatData` partially mitigates this. | Low-Medium |
| **Format validation bypass** | `ReadRepresentationFromClipboard` does not re-validate that the requested format was in the original enumerated formats. If the callback chain is subverted, arbitrary formats could be read. | Low |
| **Permission persistence** | The `ClipboardPromise` (and its granted permission) is kept alive by `ClipboardItem`. A page could call `read()` once, store the item, and read clipboard data indefinitely without re-prompting. This is the same as the current eager-read model (permission is checked once), but lazy read makes it more exploitable since data access is deferred. | Medium |

**Recommendations:**
- Add format validation in `ReadRepresentationFromClipboard` against `item_mime_types_`.
- Consider whether `CheckSequenceNumber()` should also be called *after* the read completes (it currently is, in `ResolveFormatData`—good).
- Document the security model in the design doc, specifically the permission persistence implications.

---

## 7. Performance Considerations

| Aspect | Impact | Assessment |
|--------|--------|------------|
| **Reduced unnecessary clipboard reads** | Positive | The primary benefit — pages that call `read()` but only use `getType()` for a subset of formats will save the cost of reading/decoding unused formats (e.g., image decoding). |
| **Increased IPC latency for `getType()`** | Slightly negative | Each `getType()` now triggers an individual IPC to the browser process, whereas the eager path batches all reads together. If a page reads all formats, lazy read results in more IPC roundtrips. |
| **Memory overhead** | Neutral to slightly negative | `ClipboardPromise` lifetime is extended, holding its members alive. For short-lived clipboard operations, this is negligible. For pages that cache `ClipboardItem` objects, it could accumulate. |
| **PostTask overhead** | Negligible | `ReadRepresentationFromClipboard` uses `PostTask` to defer to the clipboard task runner, adding a minor scheduling delay. |

**Benchmarking Recommendations:**
- Measure IPC roundtrip time for lazy `getType()` vs. eager bulk read for various format counts (1, 2, 5 formats).
- Profile memory retention when `ClipboardItem` objects are held for extended periods.
- Test performance with large clipboard payloads (e.g., large images) where lazy-read savings are most significant.

---

## 8. Final Recommendation

**Verdict**: **NEEDS_WORK**

**Rationale:**  
The core design of lazy clipboard reading is sound and well-motivated — it aligns with the W3C spec direction and provides clear performance benefits for the common case where not all clipboard formats are consumed. However, the implementation has several issues that should be addressed before landing:

1. The `ClipboardItem → ClipboardPromise` strong reference cycle creates a potential memory leak pattern that needs architectural resolution (Critical).
2. The `read_callbacks_` HashMap with `WrapPersistent` captures and no cleanup in `ContextDestroyed` creates potential dangling reference issues (Critical).
3. Missing MIME type validation in the on-demand read path is a security concern (Critical).
4. Rejected promises in `representations_with_resolvers_` are not cleaned up, leading to incorrect behavior on retry (Major).
5. Test coverage gaps — particularly missing happy-path web tests and concurrent `getType()` tests (Major).

The feature flag gating (`ClipboardReadOnDemand`, status `"test"`) appropriately limits blast radius, and the overall approach follows Chromium patterns. With the identified issues addressed, this CL would be ready for review approval.

**Action Items for Author:**

1. **Resolve the memory leak**: Either use `WeakMember<ClipboardPromise>` in `ClipboardItem` or implement explicit cleanup (e.g., clear the back-reference when `ClipboardPromise` resolves).
2. **Clear `read_callbacks_` in `ContextDestroyed()`**: Add `read_callbacks_.clear()` to `ClipboardPromise::ContextDestroyed()` to prevent orphaned persistent handles.
3. **Add MIME type validation**: In `ReadRepresentationFromClipboard`, verify `format` exists in `item_mime_types_` before proceeding.
4. **Clean up rejected resolvers**: In `ResolveFormatData`, remove the entry from `representations_with_resolvers_` after rejecting, so subsequent `getType()` calls can retry.
5. **Add missing tests**: At minimum, add a happy-path web test and a concurrent `getType()` unit test.
6. **Refactor duplicated code in `clipboard_reader.cc`**: Consolidate the 5 identical `if/else` blocks into a shared helper or always pass the MIME type.
7. **Add a blank line** before `// static` on `ClipboardItem::supports` (line 219).
8. **Consider adding UMA metrics** specific to lazy-read to measure real-world adoption and behavior.

---

## 9. Comments for Gerrit

### File-Level Comments

---

**File: `clipboard_item.h`**  
**Line: 109** (`Member<ClipboardPromise> clipboard_promise_`)

> **[Critical]** This creates a strong reference from `ClipboardItem` back to `ClipboardPromise`. Since `ClipboardItem` is exposed to JavaScript and can be held indefinitely, this prevents GC of `ClipboardPromise` and all its referenced objects (resolver, permission service, etc.), potentially causing memory leaks. Consider using `WeakMember<ClipboardPromise>` instead, with null checks in `getType()` and `CheckSequenceNumber()`.

---

**File: `clipboard_item.cc`**  
**Lines: 141-160** (`ResolveFormatData`)

> **[Major]** When the promise is rejected (either due to null blob or sequence number mismatch), the resolver remains in `representations_with_resolvers_`. A subsequent `getType()` call for the same MIME type (line 185-187) would return the already-rejected promise, which is incorrect behavior. The entry should be removed from `representations_with_resolvers_` after rejection:
> ```cpp
> representations_with_resolvers_.erase(
>     representations_with_resolvers_.find(mime_type));
> ```

---

**File: `clipboard_item.cc`**  
**Lines: 197-200** (`getType`, lazy read path)

> **[Nit]** The callback `BindOnce(&ClipboardItem::ResolveFormatData, WrapPersistent(this))` uses `WrapPersistent` which prevents GC of `ClipboardItem` while the read is in flight. This is intentional, but adding a brief comment explaining this design choice would help future readers:
> ```cpp
> // WrapPersistent is needed to prevent GC of this ClipboardItem
> // while the async clipboard read is in progress.
> ```

---

**File: `clipboard_promise.cc`**  
(`ReadRepresentationFromClipboard`)

> **[Critical]** This method does not validate that the requested `format` is one of the formats enumerated in `item_mime_types_`. Consider adding a validation check:
> ```cpp
> if (!item_mime_types_.Contains(format)) {
>   std::move(callback).Run(format, nullptr);
>   return;
> }
> ```

---

**File: `clipboard_promise.cc`**  
(`ContextDestroyed`)

> **[Critical]** `read_callbacks_` is not cleared in `ContextDestroyed()`. The callbacks capture `WrapPersistent` pointers to `ClipboardItem` objects. If the context is destroyed while callbacks are pending, these persistent handles will prevent GC indefinitely. Add:
> ```cpp
> read_callbacks_.clear();
> ```

---

**File: `clipboard_reader.cc`**  
(5 occurrences of the `if/else` pattern)

> **[Suggestion]** The same `if (RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()) { promise_->OnRead(blob, mime); } else { promise_->OnRead(blob); }` pattern is repeated 5 times. Consider always passing the MIME type and having `OnRead(Blob*, const String&)` be the primary method, with `OnRead(Blob*)` being a wrapper for backward compatibility:
> ```cpp
> // In all readers, just call:
> promise_->OnRead(blob, mime_type);
> // And in OnRead(Blob* blob):
> void ClipboardPromise::OnRead(Blob* blob) {
>   // Legacy eager-read path
>   ...
> }
> ```

---

**File: `clipboard_promise.h`**  
**Line: 220** (`HashMap<String, base::OnceCallback<...>> read_callbacks_`)

> **[Nit]** This `HashMap` is not traced by GC (it contains non-GC types), which is correct. However, consider adding a comment noting that `ContextDestroyed()` must clear this map to avoid leaking `WrapPersistent` handles held by the callbacks.

---

**File: `clipboard_unittest.cc`**  
**Lines: 258-348** (new tests)

> **[Major]** Good start on test coverage! However, please add the following tests:
> 1. A test that verifies `getType()` succeeds and returns correct data when clipboard hasn't changed (full happy path with blob content verification).
> 2. A test for concurrent `getType()` calls with different MIME types.
> 3. A test that calls `getType()` with a type not in the clipboard's available formats.
> These are important to validate the lazy-read path end-to-end.

---

**File: `async-clipboard-lazy-read.html`**

> **[Minor]** This web test only covers the error case (clipboard changed → DataError). Please add a companion test that verifies the happy path: `read()` → `getType()` → data is correctly returned. Also consider adding a test that reads multiple formats from the same `ClipboardItem`.

---

**File: `runtime_enabled_features.json5`**

> **[Info]** `ClipboardReadOnDemand` is set to `status: "test"`, meaning it's only enabled in test/experimental contexts. This is appropriate for the current stage. Remember to add a `base::Feature` or Finch integration when ready to roll out.
