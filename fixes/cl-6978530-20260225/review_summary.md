# Review Summary — Multi-Model Merged Review

> **Models**: claude-opus-4.6-fast, gpt-5.3-codex  
> **Models reporting**: 2/2

---

## 📋 Review by **claude-opus-4.6-fast**

# CL 6978530 Review Summary: [Clipboard] Implement on-demand reading in getType()

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Reviewers:** Ashish Kumar, Prashant Nevase, Rohan Raja
**Files Changed:** 13 files, +711/-121 lines
**Latest Patch Set:** 25 (dry run passed)

---

## 1. Executive Summary

This CL implements lazy/on-demand clipboard data reading in the Async Clipboard API. Previously, `clipboard.read()` fetched all clipboard data upfront; this change defers actual data reading to `ClipboardItem.getType()`, improving performance when only specific formats are needed. The implementation introduces a `ClipboardReaderResultHandler` interface to decouple `ClipboardReader` from `ClipboardPromise`, allowing `ClipboardItem` to directly receive read results and manage its own lazy-read lifecycle. The feature is gated behind a runtime-enabled feature flag (`ReadClipboardDataOnClipboardItemGetType`) with `status: "test"`.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 3 | Dual code paths (lazy vs eager) add complexity; feature-flag checks are scattered throughout |
| Maintainability | 3 | Many `RuntimeEnabledFeatures::ReadClipboardDataOnClipboardItemGetTypeEnabled()` checks create branching that will need cleanup |
| Extensibility | 4 | `ClipboardReaderResultHandler` interface is a clean abstraction enabling future extensions |
| Consistency | 3 | Naming and pattern usage is mostly consistent, but `ClipboardItem` now has dual roles (data holder + lifecycle observer) |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BEFORE (Eager Read)                               │
│                                                                     │
│  clipboard.read() ──► ClipboardPromise                              │
│                          │                                          │
│                          ├── ReadAvailableFormats()                  │
│                          ├── ReadNextRepresentation() [loops]        │
│                          │     └── ClipboardReader::Create()         │
│                          │           └── reader.Read()               │
│                          │                 └── OnRead(blob)          │
│                          │                       └── next repr...   │
│                          └── ResolveRead()                          │
│                                └── ClipboardItem(data+blobs)        │
│                                      └── getType() → resolved blob  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    AFTER (Lazy Read)                                 │
│                                                                     │
│  clipboard.read() ──► ClipboardPromise                              │
│                          │                                          │
│                          ├── ReadAvailableFormats()                  │
│                          └── ResolveRead() [immediate, no data]     │
│                                └── ClipboardItem(mime_types only)   │
│                                                                     │
│  clipboardItem.getType("text/plain") ──► ClipboardItem              │
│                          │                                          │
│                          ├── Check ExecutionContext alive            │
│                          ├── Check clipboard sequence number        │
│                          ├── Create ScriptPromiseResolver            │
│                          ├── ClipboardReader::Create(this)          │
│                          │     └── reader.Read()                    │
│                          │           └── OnRead(blob, mime_type)    │
│                          │                 └── ResolveFormatData()  │
│                          └── Return promise                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              ClipboardReaderResultHandler (New Interface)            │
│                                                                     │
│  «interface» ClipboardReaderResultHandler                           │
│  ├── OnRead(Blob*, String& mime_type)                               │
│  ├── GetExecutionContext()                                          │
│  └── GetLocalFrame()                                                │
│         ▲                    ▲                                      │
│         │                    │                                      │
│  ClipboardPromise     ClipboardItem                                 │
│  (eager path)         (lazy path)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | Several lifecycle/crash concerns identified (see Key Findings) |
| Efficiency | 4 | Core goal achieved: avoids reading unused formats |
| Readability | 3 | Dual paths add cognitive overhead; scattered feature flag checks |
| Test Coverage | 3 | Unit tests added but missing edge cases (context destruction, concurrent reads for standard types, error paths) |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Potential NULL dereference of `GetSystemClipboard()` in `ResolveRead()` (clipboard_promise.cc:386)**

   In the lazy-read path of `ResolveRead()`:
   ```cpp
   clipboard_items = {MakeGarbageCollected<ClipboardItem>(
       item_mime_types_, GetSystemClipboard()->SequenceNumber(), ...)};
   ```
   `GetSystemClipboard()` can return `nullptr` if the `LocalFrame` has been detached between when the formats were read and when `ResolveRead()` executes. Calling `->SequenceNumber()` on a null pointer would crash. The eager path at line 406 has the same issue (`GetSystemClipboard()->SequenceNumber()`). While there's a `DCHECK(GetExecutionContext())` at line 372, a valid `ExecutionContext` does not guarantee a valid `LocalFrame` (and hence `SystemClipboard`). **This is a potential NULL crash.**

   **Fix:** Add a null check for `GetSystemClipboard()` before dereferencing it. If null, pass `std::nullopt` as the sequence number.

2. **`ClipboardReader` constructor dereferences `result_handler->GetExecutionContext()` without null check (clipboard_reader.cc:368-370)**

   ```cpp
   ClipboardReader::ClipboardReader(SystemClipboard* system_clipboard,
                                    ClipboardReaderResultHandler* result_handler)
       : clipboard_task_runner_(
             result_handler->GetExecutionContext()->GetTaskRunner(
                 TaskType::kUserInteraction)),
   ```
   When `ClipboardItem::ReadRepresentationFromClipboardReader()` is called, it creates a `ClipboardReader` via `ClipboardReader::Create()`. If the `ExecutionContext` has been destroyed between when `getType()` was invoked and when the reader is constructed, `GetExecutionContext()` returns `nullptr`, and calling `->GetTaskRunner()` will crash.

   While `getType()` does check `GetExecutionContext()` at line 196, the execution context can be destroyed asynchronously between that check and the actual `ClipboardReader` construction (though this is unlikely on a single-threaded main thread). Nevertheless, this is a defensive programming concern worth addressing.

   **Fix:** Add a null check for `result_handler->GetExecutionContext()` in the `ClipboardReader` constructor or in `ReadRepresentationFromClipboardReader()` before calling `Create()`.

3. **Resolving promise with null blob without error (clipboard_item.cc:171)**

   In `ResolveFormatData()`, if `blob` is `nullptr` and the clipboard hasn't changed, the code calls:
   ```cpp
   representations_with_resolvers_.at(mime_type)->Resolve(blob);
   ```
   This resolves the promise with a null Blob. The web spec for `getType()` says it should return a Blob. Resolving with null could cause downstream JavaScript to crash when calling methods on the returned value (e.g., `blob.text()`). The original eager-read path skips null blobs entirely (clipboard_promise.cc:395-397).

   **Fix:** When `blob` is null in the lazy path, the promise should be rejected with a `DOMException` (e.g., `NotFoundError` or `DataError`) rather than resolved with null.

### Major Issues (Should Fix)

1. **`ContextDestroyed()` may reject already-resolved promises**

   In `ClipboardItem::ContextDestroyed()` (clipboard_item.cc:291-298):
   ```cpp
   for (auto& entry : representations_with_resolvers_) {
     entry.value->Reject(detached_error);
   }
   representations_with_resolvers_.clear();
   ```
   The `representations_with_resolvers_` map stores ALL resolvers, including those that have already been resolved (via `ResolveFormatData`). Calling `Reject()` on an already-resolved `ScriptPromiseResolver` is a no-op in Blink (it's protected), but it's wasteful and semantically incorrect. Also, in `ResolveFormatData()`, once a promise is resolved, the entry is NOT removed from `representations_with_resolvers_` — it's kept so that `getType()` can return the cached promise (line 218-219). This means `ContextDestroyed()` will attempt to reject resolved promises.

   **Fix:** Either (a) remove entries from `representations_with_resolvers_` after resolve/reject and maintain a separate cache of resolved promises, or (b) track which resolvers are still pending.

2. **`ClipboardItem` as `ExecutionContextLifecycleObserver` with null context for non-lazy path**

   The non-lazy constructor (clipboard_item.cc:103) passes `nullptr` to `ExecutionContextLifecycleObserver`:
   ```cpp
   : ExecutionContextLifecycleObserver(nullptr),
   ```
   This means `GetExecutionContext()` will always return `nullptr` for non-lazy `ClipboardItem` instances. While the lazy-read code paths check `is_lazy_read_` before using execution context features, the `GetExecutionContext()` override on `ClipboardItem` (line 96-98) is a public virtual method that could be called from any code path, returning null unexpectedly. This is fragile.

   **Mitigation:** This is somewhat mitigated by the `is_lazy_read_` checks, but it would be cleaner to only inherit from `ExecutionContextLifecycleObserver` when lazy mode is active, or at least document this clearly.

3. **No protection against `getType()` being called after `ContextDestroyed()` for cached resolvers**

   If `getType("text/plain")` is called, a reader is created and the resolver is stored. If the context is destroyed before the read completes, `ContextDestroyed()` rejects the promise and clears maps. However, the in-flight `ClipboardReader` holds a `Member<ClipboardReaderResultHandler>` (pointing to the `ClipboardItem`). When the reader's async callback fires, it calls `result_handler_->OnRead()`, which calls `ResolveFormatData()`. At this point, `representations_with_resolvers_` has been cleared, so the early return at line 158-160 fires harmlessly. **This is safe** — no crash here. But the active reader still holds a reference to the `ClipboardItem`, keeping it alive unnecessarily.

   **Note:** This is actually handled correctly because `ContextDestroyed()` also calls `active_readers_.clear()`, which drops the reader references. The reader's callback may still fire (via `WrapPersistent`), but the `ClipboardItem`'s `OnRead` will harmlessly find no matching resolver. This is acceptable.

### Minor Issues (Nice to Fix)

1. **Redundant feature-flag check in `types()` method (clipboard_item.cc:137-139)**

   The `types()` method checks both the feature flag AND `is_lazy_read_`. Since `is_lazy_read_` can only be true when the feature flag is enabled (enforced by the DCHECK in the constructor), the feature flag check is redundant. This applies to `getType()` as well (line 177-179).

   **Fix:** Simply check `is_lazy_read_` instead of both conditions.

2. **`HeapVector<String> mime_types_` should not be a `HeapVector`**

   `String` in Blink/WTF is not a garbage-collected type that needs to be traced through the GC heap. Using `HeapVector<String>` is technically valid but unnecessary. A `Vector<String>` would be more appropriate and avoid unnecessary GC overhead.

   **Note:** Looking more carefully, `HeapVector<String>` is used because `String` can hold GC'd `StringImpl` in Oilpan builds. This may actually be required. Leaving this as advisory.

3. **TODO comment suggests removing `ClipboardReaderResultHandler` once stable (clipboard_reader.h:19-20)**

   The comment says "Remove this interface once lazy read getType is stable." However, this interface is a clean abstraction improvement. Consider keeping it even after the feature is stable, or update the TODO to say the eager path should be removed instead.

4. **Empty `Trace()` in `ClipboardReaderResultHandler` (clipboard_reader.h:26)**

   `void Trace(Visitor* visitor) const override {}` — this empty implementation exists because `GarbageCollectedMixin` requires it, but it should ideally call the parent `GarbageCollectedMixin::Trace(visitor)` for correctness, even though the parent is also empty.

### Suggestions (Optional)

1. **Consider consolidating clipboard change checks**: `HasClipboardChangedSinceClipboardRead()` is called in both `getType()` (proactive check) and `ResolveFormatData()` (on read completion). This double-check is good for safety, but the second call involves an IPC to get the sequence number. Consider caching the result if performance is a concern.

2. **Consider adding a `is_resolved_` flag to avoid redundant resolve/reject calls**: Rather than leaving resolved `ScriptPromiseResolver`s in the map and relying on Blink's internal "already settled" protection, tracking state explicitly would be more robust and self-documenting.

3. **Feature flag naming**: `ReadClipboardDataOnClipboardItemGetType` is quite long. Consider a shorter name like `ClipboardLazyRead` for readability (though the current name is more descriptive).

---

## 5. Test Coverage Analysis

### What Tests Exist

1. **Unit Tests (clipboard_unittest.cc):**
   - `ReadOnlyMimeTypesInClipboardRead` — Verifies that `clipboard.read()` only calls `ReadAvailableCustomAndStandardFormats` and NOT `ReadText`/`ReadHtml` (proving lazy loading).
   - `ClipboardItemGetTypeTest` — Verifies that `getType("text/plain")` triggers the actual `ReadText` call and returns a correctly-sized Blob.
   - Pre-existing tests (`ClipboardReadTextTest`, `ClipboardReadTest`, `ClipboardReadSelectiveTypesTest`) continue to pass.

2. **Web Tests:**
   - `async-clipboard-lazy-read.html` — Tests clipboard change detection (DataError when clipboard changes between `read()` and `getType()`).
   - `async-clipboard-custom-format-lazy-read.html` — Tests lazy read with custom formats (web prefix).
   - `async-custom-format-lazy-read-concurrent.tentative.https.html` — Tests concurrent `getType()` calls for multiple custom formats.

### What Tests Are Missing

1. **Context destruction during in-flight read** — No test verifies behavior when the document/frame is detached while a `getType()` read is in progress.
2. **Concurrent `getType()` for standard types** — The concurrent test only covers custom formats. Missing test for concurrent `getType("text/plain")` and `getType("text/html")`.
3. **Null blob handling** — No test verifies the behavior when `ClipboardReader` returns a null blob (e.g., empty clipboard for a specific type).
4. **Multiple `getType()` calls for the same type** — No test verifies that calling `getType("text/plain")` twice returns the same promise.
5. **`getType()` for unsupported type** — No test verifies that `getType("application/pdf")` throws `NotFoundError`.
6. **Error path when `SystemClipboard` is unavailable** — No test covers the case where `GetSystemClipboard()` returns null during lazy read.

### Recommended Additional Tests

- Test calling `getType()` after context destruction (should throw `NotAllowedError`).
- Test that concurrent standard-format reads (text + HTML) both resolve correctly.
- Test that calling `getType()` with the same type twice returns identical results.
- Test the null Blob case (if it remains resolved rather than rejected, test that behavior).

---

## 6. Security Considerations

1. **Clipboard change detection is a security feature**: The sequence number check between `read()` and `getType()` prevents TOCTOU (Time-of-Check-Time-of-Use) attacks where clipboard content could be swapped between the permission grant and the actual data read. This is correctly implemented.

2. **HTML sanitization preserved**: The `sanitize_html_for_lazy_read_` flag correctly propagates the sanitization setting from `ClipboardPromise` to `ClipboardItem`, ensuring that HTML content is still sanitized by default in the lazy path.

3. **Permission model unchanged**: The permission check still happens during `clipboard.read()` in `ClipboardPromise`. The lazy `getType()` call does not re-check permissions, which is correct since the permission was already granted for the read operation. However, there's a subtle concern: the time between permission grant and actual data access is now longer (user-controlled), which could theoretically allow a permission to be revoked in between. This is mitigated by the clipboard change detection.

4. **No new IPC surfaces**: The lazy read uses the same `SystemClipboard` IPC methods (`ReadPlainText`, `ReadHTML`, etc.) as the eager path. No new attack surface is introduced.

---

## 7. Performance Considerations

1. **Primary benefit achieved**: The main performance improvement is that `clipboard.read()` now only fetches available format names (one IPC call) instead of reading all format data (N IPC calls). Data is only read when `getType()` is explicitly called.

2. **Potential regression for "read all" usage**: If a web app calls `getType()` for every available type, the lazy approach may be slightly slower than the eager approach because each `getType()` triggers a separate `ClipboardReader` creation and IPC call, whereas the eager path batches reads sequentially. However, this is a minor concern since the reads are now user-driven.

3. **Sequence number check overhead**: `HasClipboardChangedSinceClipboardRead()` makes a synchronous IPC call to get the current clipboard sequence number. This is called both in `getType()` (before read) and in `ResolveFormatData()` (after read), resulting in 2 IPC calls per `getType()` invocation beyond the actual data read. Consider caching the check or making it asynchronous.

4. **Benchmarking recommendation**: Measure the end-to-end latency of `clipboard.read()` + `getType()` vs. the old `clipboard.read()` for the common case of reading 1-2 types. Also measure memory usage since `ClipboardItem` now holds additional state (resolvers map, active readers map, etc.).

---

## 8. Final Recommendation

**Verdict**: NEEDS_WORK

**Rationale:**

The architectural approach is sound — introducing `ClipboardReaderResultHandler` to decouple `ClipboardReader` from `ClipboardPromise` is a clean design, and the lazy-read optimization will provide real performance benefits for web apps that only need specific clipboard formats. The clipboard change detection via sequence numbers is a good safety mechanism.

However, there are critical crash risks that must be addressed before landing:

1. The NULL dereference risk on `GetSystemClipboard()->SequenceNumber()` in `ResolveRead()` is a real crash scenario that can occur when a frame is detached between format enumeration and read resolution.
2. Resolving a promise with a null Blob is semantically incorrect and will cause JavaScript runtime errors.
3. The `ClipboardReader` constructor's unconditional dereference of `GetExecutionContext()` needs defensive null handling.

Additionally, test coverage should be strengthened for edge cases around context destruction and concurrent reads for standard types.

**Action Items for Author:**

1. **[Critical]** Add null check for `GetSystemClipboard()` in `ClipboardPromise::ResolveRead()` (line 386 and 406) before calling `->SequenceNumber()`. If null, pass `std::nullopt` as sequence number.
2. **[Critical]** In `ClipboardItem::ResolveFormatData()`, reject the promise with a `DOMException` when `blob` is null rather than resolving with null.
3. **[Critical]** Add null check for `GetExecutionContext()` before constructing `ClipboardReader` in `ReadRepresentationFromClipboardReader()`.
4. **[Major]** Consider removing resolved/rejected entries from `representations_with_resolvers_` or tracking pending state to avoid redundant reject calls in `ContextDestroyed()`.
5. **[Minor]** Simplify redundant feature-flag + `is_lazy_read_` checks — just check `is_lazy_read_`.
6. **[Test]** Add tests for context destruction during in-flight read, concurrent standard-type reads, and null blob scenarios.

---

## 9. Comments for Gerrit

### clipboard_item.cc — `ResolveFormatData()` (line 171)

> **[Critical/Bug]** When `blob` is null (read failure or unsupported format), this resolves the promise with null. This will cause JavaScript errors when the caller tries to use the Blob (e.g., `blob.text()`). The eager path (in `ClipboardPromise::OnRead`) skips null blobs. The lazy path should reject the promise with a `DOMException` (e.g., `DataError` with message "Failed to read clipboard data") instead of resolving with null.

### clipboard_promise.cc — `ResolveRead()` (line 386)

> **[Critical/Crash]** `GetSystemClipboard()` can return nullptr if the LocalFrame has been destroyed. Calling `->SequenceNumber()` will crash. Add a null check:
> ```cpp
> SystemClipboard* system_clipboard = GetSystemClipboard();
> std::optional<absl::uint128> seq_num = system_clipboard
>     ? std::make_optional(system_clipboard->SequenceNumber())
>     : std::nullopt;
> ```
> The same issue exists at line 406 for the non-lazy path.

### clipboard_item.cc — `ReadRepresentationFromClipboardReader()` (line 246)

> **[Defensive]** Consider adding a `GetExecutionContext()` null check before calling `ClipboardReader::Create()`. The `ClipboardReader` constructor (clipboard_reader.cc:368) unconditionally dereferences `result_handler->GetExecutionContext()` to get a task runner. While `getType()` checks context validity before reaching here, adding a defensive check would prevent crashes if the context is destroyed between those two points (even though unlikely on the main thread).

### clipboard_item.cc — `ContextDestroyed()` (line 294)

> **[Suggestion]** This iterates over all entries in `representations_with_resolvers_` including already-resolved ones. While double-reject is safe in Blink (no-op), it would be cleaner to either remove entries after resolution or track pending status. Consider erasing the entry in `ResolveFormatData()` after resolving and maintaining a separate `HeapHashMap` for caching the returned promise.

### clipboard_item.cc — `types()` (line 137-139)

> **[Nit]** The feature flag check is redundant with `is_lazy_read_` since `is_lazy_read_` can only be true when the feature is enabled (enforced by DCHECK in constructor). Simplify to just `if (is_lazy_read_)`.

### clipboard_reader.h — `ClipboardReaderResultHandler::Trace()` (line 26)

> **[Nit]** Consider calling `GarbageCollectedMixin::Trace(visitor)` in the body for correctness, even though the parent implementation is also empty.

### clipboard_unittest.cc — Test coverage

> **[Request]** Please add tests for:
> 1. Calling `getType()` after the execution context is destroyed.
> 2. Concurrent `getType()` calls for standard types (text/plain + text/html).
> 3. Behavior when the clipboard reader returns a null Blob.
> 4. Calling `getType()` twice for the same MIME type returns the same promise.

---

## Object Lifecycle Analysis

### ClipboardItem Lifecycle

**Creation:**
- **Lazy path:** Created in `ClipboardPromise::ResolveRead()` via `MakeGarbageCollected<ClipboardItem>(mime_types, sequence_number, execution_context, sanitize_html, is_lazy_read=true)`. The `ClipboardItem` is garbage-collected (via Oilpan). It registers itself as an `ExecutionContextLifecycleObserver` with the provided `ExecutionContext`.
- **Eager path:** Created in `ClipboardPromise::ResolveRead()` with pre-resolved data. Passes `nullptr` as the `ExecutionContext` to `ExecutionContextLifecycleObserver`, meaning `ContextDestroyed()` will never be called for eager items.

**Alive:**
- The `ClipboardItem` is kept alive by JavaScript references (the resolved promise holds a reference to the `ClipboardItem` array).
- When `getType()` is called, the `ClipboardItem` creates `ScriptPromiseResolver` objects (stored in `representations_with_resolvers_`) and `ClipboardReader` objects (stored in `active_readers_`). Both are `Member<>` pointers traced by the GC, keeping them alive.
- The `ClipboardReader` holds a `Member<ClipboardReaderResultHandler>` pointing back to the `ClipboardItem`, creating a reference cycle that is handled by Oilpan's tracing GC (not reference counting).

**Destruction:**
- When JavaScript releases all references and no active readers exist, the GC can collect the `ClipboardItem`.
- If the `ExecutionContext` is destroyed first, `ContextDestroyed()` is called, which rejects all pending promises and clears `active_readers_`, breaking the reference cycle.

### ClipboardReader Lifecycle

**Creation:** Created by `ClipboardReader::Create()` in either `ClipboardPromise::ReadNextRepresentation()` (eager) or `ClipboardItem::ReadRepresentationFromClipboardReader()` (lazy). It's a `GarbageCollected` object.

**Alive:**
- **Eager path:** Stored in `ClipboardPromise::clipboard_reader_` (`Member<ClipboardReader>`). Only one reader is active at a time. Each reader is replaced when the next representation is read.
- **Lazy path:** Stored in `ClipboardItem::active_readers_` (`HeapHashMap<String, Member<ClipboardReader>>`), keyed by MIME type. Multiple readers can be active concurrently (one per type). This correctly handles the previous review feedback about concurrent `getType()` calls.
- The reader also stays alive via `WrapPersistent(this)` in async callbacks (e.g., `BindOnce(&ClipboardHtmlReader::OnRead, WrapPersistent(this))`). This prevents premature GC while an async clipboard read is in flight.

**Destruction:**
- **Lazy path:** Removed from `active_readers_` in `ResolveFormatData()` after the read completes. Also cleared in `ContextDestroyed()`.
- **Eager path:** Cleared in `ClipboardPromise::ContextDestroyed()`.
- The `WrapPersistent` prevents GC until the callback fires. After the callback completes and the `Member<>` references are cleared, the GC can collect the reader.

### ScriptPromiseResolver Lifecycle

**Creation:** Created in `ClipboardItem::getType()` via `MakeGarbageCollected<ScriptPromiseResolver<Blob>>(script_state, ...)`. Stored in `representations_with_resolvers_`.

**Alive:** Kept alive by the `Member<ScriptPromiseResolver<Blob>>` in the `HeapHashMap`. The associated JavaScript `Promise` also holds a reference.

**Destruction:** The resolver is settled (resolved or rejected) in `ResolveFormatData()` or `ContextDestroyed()`. After settlement, the entry remains in the map (for caching), but the resolver's internal state prevents double-settlement. The resolver (and the entire `ClipboardItem`) can be GC'd once JavaScript releases the promise reference.

### Key Chromium Constructs Used

1. **`Member<T>`** — A traced pointer for Oilpan GC. Used for `ClipboardReader`, `ScriptPromiseResolver`, and `ClipboardReaderResultHandler`. These are automatically traced during GC marking, preventing premature collection. Unlike raw pointers, `Member<T>` participates in Oilpan's garbage collection tracing.

2. **`WrapPersistent(this)`** — Creates a persistent handle that prevents the GC from collecting the object. Used in `BindOnce` callbacks to ensure the object survives until the callback fires. This is critical for async clipboard operations where the callback may fire after the JavaScript context has released its references. **Risk:** If the callback never fires (e.g., Mojo pipe broken), the persistent handle creates a leak. In practice, Mojo guarantees callback invocation (or destructor-based cleanup).

3. **`HeapHashMap<String, Member<T>>`** — A GC-traced hash map. Used for `representations_with_resolvers_` and `active_readers_`. The GC traces both keys and values during marking.

4. **`ExecutionContextLifecycleObserver`** — Provides a `ContextDestroyed()` callback when the associated `ExecutionContext` (typically a `Document`) is destroyed. This is the primary cleanup mechanism for pending promises and active readers. **Important:** For non-lazy `ClipboardItem`, the context is `nullptr`, so `ContextDestroyed()` is never called. This is intentional since non-lazy items have no pending state.

5. **`GarbageCollectedMixin`** — Used for `ClipboardReaderResultHandler`. Enables a class to participate in Oilpan tracing without being directly garbage-collected itself. Subclasses (`ClipboardPromise`, `ClipboardItem`) provide the actual `Trace()` implementations.

6. **`MakeCrossThreadHandle(this)` / `MakeUnwrappingCrossThreadHandle`** — Used in `ClipboardTextReader`, `ClipboardHtmlReader`, `ClipboardSvgReader` for cross-thread callbacks (encoding on background thread, then calling back to main thread). These are safe because they wrap the GC'd object in a handle that can be transferred across threads, and unwrapping on the main thread produces a normal traced reference.

### Crash Safety Assessment

| Scenario | Safety | Notes |
|----------|--------|-------|
| `getType()` after context destroyed | ✅ Safe | `getType()` checks `GetExecutionContext()` first |
| Context destroyed during in-flight read | ✅ Safe | `ContextDestroyed()` clears maps; late `OnRead()` callback harmlessly finds no resolver |
| Multiple concurrent `getType()` calls | ✅ Safe | Each reader stored separately in `active_readers_` keyed by MIME type |
| `getType()` after clipboard change | ✅ Safe | Sequence number check rejects promise |
| `GetSystemClipboard()` returns null in `ResolveRead()` | ❌ **Crash** | No null check before `->SequenceNumber()` |
| Null blob from `ClipboardReader` | ⚠️ Logic bug | Promise resolved with null instead of rejected |
| GC during async read | ✅ Safe | `WrapPersistent` + `Member<>` prevents premature collection |
| `ReadRepresentationFromClipboardReader` with null context | ⚠️ Potential crash | `ClipboardReader` constructor dereferences context unconditionally |


---

## 📋 Review by **gpt-5.3-codex**

# CL 6978530 Review Summary: [Clipboard] Implement on-demand reading in getType()

## 1. Executive Summary

This CL moves Async Clipboard `read()` from eager payload loading to lazy, on-demand loading in `ClipboardItem.getType()`, which is a good performance direction and is correctly feature-flagged (`ReadClipboardDataOnClipboardItemGetType`, status `"test"`).  
The design introduces `ClipboardReaderResultHandler` so `ClipboardReader` can report results to either `ClipboardPromise` (legacy eager path) or `ClipboardItem` (new lazy path), and adds lifecycle cleanup via `ExecutionContextLifecycleObserver`.  
However, there are still null-dereference crash risks in new call sites that dereference `GetSystemClipboard()`/`GetExecutionContext()` without guarding nullable lifecycle transitions.

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | Lazy flow is conceptually clear; dual lazy/eager paths add some branching complexity. |
| Maintainability | 3 | New abstraction is clean, but many runtime-feature branches and nullable frame/context assumptions need hardening. |
| Extensibility | 4 | `ClipboardReaderResultHandler` cleanly decouples reader completion target and supports future removal of eager path. |
| Consistency | 4 | Follows Blink/Oilpan patterns (`Member<>`, `Trace`, lifecycle observer), with a few missing defensive checks. |

### Architecture Diagram

```mermaid
graph TD
  JS["JS: navigator.clipboard.read() / item.getType(type)"]
  CP["ClipboardPromise\n(Entry point, permission + read orchestration)"]
  CI["ClipboardItem\n(lazy mode: MIME list + on-demand read)"]
  CR["ClipboardReader\n(per-type read/encode)"]
  IF["ClipboardReaderResultHandler\n(GarbageCollectedMixin)"]
  SC["SystemClipboard (renderer-side)"]
  CH["ClipboardHost (browser process)"]

  JS --> CP
  CP -->|lazy mode| CI
  JS -->|getType()| CI
  CI --> CR
  CP --> CR
  CR --> IF
  IF -.implemented by.-> CP
  IF -.implemented by.-> CI
  CR --> SC --> CH
```

### Lifecycle Summary (HLD+LLD focus)

- **`ClipboardPromise`**
  - **Created** by `CreateForRead/CreateForReadText/...` with `MakeGarbageCollected`.
  - **Alive because** it is GC-traced and held while JS promise + async callbacks are active.
  - **Destroyed/teardown** via Oilpan GC; `ContextDestroyed()` rejects pending API promise and clears `clipboard_writer_` / `clipboard_reader_`.
- **`ClipboardItem` (lazy path)**
  - **Created** in `ResolveRead()` with MIME types + clipboard sequence + `ExecutionContext*`.
  - **Alive because** JS references + GC-traced maps (`representations_with_resolvers_`, `active_readers_`).
  - **Destroyed/teardown** via GC; `ContextDestroyed()` rejects pending `getType()` resolvers and clears active readers.
- **`ClipboardReader`**
  - **Created** per requested MIME type (`ClipboardReader::Create`).
  - **Alive because** `Member<>` references (`clipboard_reader_` or `active_readers_`) plus callback persistence (`WrapPersistent(this)`, cross-thread handles).
  - **Destroyed** after callback completion and reference removal.
- **`ScriptPromiseResolver<Blob>`**
  - **Created** on first lazy `getType(type)`.
  - **Alive because** cached in `HeapHashMap<String, Member<...>>`.
  - **Settled** in `ResolveFormatData()` or `ContextDestroyed()`.

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | Core lazy-flow logic is good, but nullable lifecycle edges still expose crash risk. |
| Efficiency | 4 | Avoids eager reads/decodes and improves common case where only one type is consumed. |
| Readability | 3 | Mostly readable; duplicated branch logic and mixed eager/lazy paths increase cognitive load. |
| Test Coverage | 3 | Good happy-path coverage; critical detach/null/lifecycle race edges are under-tested. |

## 4. Key Findings

### Critical Issues (Must Fix)

- **Issue 1: NULL crash risk from unguarded `GetSystemClipboard()` dereferences in `clipboard_promise.cc`.**  
  `GetSystemClipboard()` now returns nullable, but several call sites dereference immediately (e.g., `ResolveRead()` sequence number fetch, `HandleReadWithPermission`, `HandleReadTextWithPermission`, platform-permission callbacks, `ReadNextRepresentation`). A detached frame with still-live execution context can crash.
- **Issue 2: NULL crash risk in `ClipboardReader` ctor (`clipboard_reader.cc`) due to unconditional `result_handler->GetExecutionContext()->GetTaskRunner(...)`.**  
  Constructor assumes non-null execution context; if lifecycle timing changes or detach interleaves before reader creation, this becomes a hard null dereference.

### Major Issues (Should Fix)

- **Issue 1: `ClipboardItem::ResolveFormatData()` resolves promise with `nullptr` blob.**  
  This yields success-shaped JS resolution with `null`, which causes downstream runtime failures and inconsistent error semantics; should reject with a clear `DOMException`.
- **Issue 2: Missing lifecycle regression tests for detach/null transitions in lazy flow.**  
  No explicit test for read-resolve then frame detach before `getType()`, or detach during in-flight reader callback.

### Minor Issues (Nice to Fix)

- **Issue 1:** `ContextDestroyed()` rejects all cached resolvers, including already-settled ones (safe but noisy).  
- **Issue 2:** Redundant feature-flag checks where `is_lazy_read_` already captures mode.

### Suggestions (Optional)

- Add centralized helper for null-safe clipboard access in `ClipboardPromise` to eliminate repeated nullable deref hazards.
- After rollout, remove eager-path compatibility scaffolding and simplify flow around `ClipboardReaderResultHandler`.

## 5. Test Coverage Analysis

- **What tests exist**
  - Unit: `ReadOnlyMimeTypesInClipboardRead`, `ClipboardItemGetTypeTest`.
  - Web: `async-clipboard-lazy-read.html` (DataError on clipboard mutation), `async-clipboard-custom-format-lazy-read.html`, concurrent custom-format lazy read WPT internal test.
- **What tests are missing**
  - Detach after `read()` but before `getType()`.
  - Detach during in-flight reader callback.
  - Same-type concurrent `getType()` idempotence.
  - Explicit null-reader/null-blob failure behavior assertions.
- **Recommended additional tests**
  - Add unit test that simulates frame/context teardown between lazy resolver creation and callback completion.
  - Add test asserting no crash + deterministic rejection when `GetSystemClipboard()` is null.
  - Add same-type double `getType()` test to verify promise reuse + no duplicate reads.

## 6. Security Considerations

- Positive: sequence-number revalidation before and at resolve mitigates stale-data/TOCTOU clipboard misuse.
- No new privilege boundary is added; same clipboard IPC endpoints and permission flow remain.
- Recommendation: harden null-path error handling to avoid crash-based denial-of-service surfaces in renderer lifecycle edge cases.

## 7. Performance Considerations

- Positive: lazy mode removes unnecessary upfront reads/encoding for unused MIME types.
- Tradeoff: workloads that always consume all types may pay extra per-`getType()` overhead (additional callback/setup/sequence checks).
- Recommendation: benchmark single-type/common-case latency and all-types worst-case to confirm expected win envelope.

## 8. Final Recommendation

**Verdict**: NEEDS_WORK

**Rationale**:  
The CL is directionally strong and the object-ownership model largely follows correct Oilpan lifecycle patterns (`Member<>`, tracing, `ExecutionContextLifecycleObserver`, `WrapPersistent`, cross-thread handles), which significantly reduces UAF risk.  
The remaining blocker is null-safety in lifecycle races: there are still unguarded nullable dereferences in clipboard/context access paths that can crash the renderer.

**Action Items for Author**:
1. Add null guards (or explicit rejection path) at all `GetSystemClipboard()` dereference sites in `clipboard_promise.cc`, especially where `SequenceNumber()` and read APIs are called.
2. Defensively guard `GetExecutionContext()` before constructing/initializing `ClipboardReader` task runner.
3. Change lazy null-blob handling to reject with `DOMException` instead of resolving with `null`.
4. Add detach/lifecycle race regression tests for lazy `getType()` and callback completion after teardown.

## 9. Comments for Gerrit

1. **`third_party/blink/renderer/modules/clipboard/clipboard_promise.cc` (`ResolveRead`, `HandleReadWithPermission`, related read paths):**  
   "`GetSystemClipboard()` is nullable, but these paths dereference it directly. Please add null checks and reject with detached-document style errors instead of crashing when frame is gone."

2. **`third_party/blink/renderer/modules/clipboard/clipboard_reader.cc` (`ClipboardReader` ctor):**  
   "Constructor dereferences `result_handler->GetExecutionContext()` unconditionally to fetch task runner. Please guard or DCHECK the invariant and fail gracefully in release mode."

3. **`third_party/blink/renderer/modules/clipboard/clipboard_item.cc` (`ResolveFormatData`):**  
   "When blob is null, current code resolves promise with null; please reject with a deterministic `DOMException` to keep getType error semantics consistent and avoid success-shaped null values."

4. **`third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`:**  
   "Please add lifecycle tests for detach between `read()` and `getType()`, and detach during in-flight lazy read callback, validating rejection + no crash."

## Appendix: Chromium Construct Safety Notes (requested focus)

- **`Member<T>` / `HeapHashMap<..., Member<T>>`**: Oilpan-traced strong references; prevent premature GC and UAF when traced in `Trace()`.
- **`GarbageCollectedMixin` (`ClipboardReaderResultHandler`)**: allows polymorphic GC-traced interface refs (`Member<ClipboardReaderResultHandler>`) without raw-pointer ownership hazards.
- **`ExecutionContextLifecycleObserver`**: ensures teardown callback (`ContextDestroyed`) so pending promises/readers are explicitly rejected/cleared on document detach.
- **`WrapPersistent(this)`**: pins GC object across async callback boundaries; correct for avoiding callback-after-collection UAF (with potential temporary retention).
- **`MakeCrossThreadHandle` / `MakeUnwrappingCrossThreadHandle`**: cross-thread safe persistent handle transfer used by reader encoding paths; prevents cross-thread UAF during background work.


---

## 🔀 Cross-Model Summary

This document merges reviews from **2** models: claude-opus-4.6-fast, gpt-5.3-codex.

### Model Coverage

| Model | Contributed |
|-------|------------|
| claude-opus-4.6-fast | ✅ Yes |
| gpt-5.3-codex | ✅ Yes |
