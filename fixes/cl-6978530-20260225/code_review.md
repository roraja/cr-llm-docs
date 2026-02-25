# Code Review — Multi-Model Merged Review

> **Models**: claude-opus-4.6-fast, gpt-5.3-codex  
> **Models reporting**: 2/2

---

## 📋 Review by **claude-opus-4.6-fast**

# Code Review: CL 6978530 — [Clipboard] Implement on-demand reading in getType()

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Patch Set Reviewed:** 25
**Reviewer:** Automated Review (with focus on NULL crashes, UAF, object lifecycle)

---

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Several null-resolve and lifecycle edge-cases; see detailed findings |
| Style | ⚠️ | Minor inconsistencies; generally follows Chromium style |
| Security | ✅ | Clipboard change detection is sound; no injection or privilege issues |
| Performance | ✅ | Lazy loading is the point of this CL; design is efficient |
| Testing | ⚠️ | Core paths tested, but critical edge cases missing |

---

## High-Level Design (HLD)

### What This CL Does

This CL implements **lazy/on-demand clipboard data reading**. Previously, `clipboard.read()` fetched all clipboard data formats (text, HTML, PNG, etc.) upfront and returned a `ClipboardItem` containing resolved Blob data. This CL defers the actual data fetching to when `getType()` is called on the `ClipboardItem`, so only formats the web page actually requests are read from the system clipboard.

### Architecture Change

**Before:**
```
clipboard.read() → ReadAvailableFormats → ReadNextRepresentation (all formats) → Resolve ClipboardItem with Blob data
getType("text/plain") → Return already-resolved promise from representations_
```

**After (when feature enabled):**
```
clipboard.read() → ReadAvailableFormats → Resolve ClipboardItem with MIME type list only (no data)
getType("text/plain") → Create ClipboardReader → Read from system clipboard → Resolve promise with Blob
```

### Key Abstractions Introduced

1. **`ClipboardReaderResultHandler`** — A new `GarbageCollectedMixin` interface extracted from `ClipboardPromise` to decouple `ClipboardReader` from `ClipboardPromise`. Both `ClipboardPromise` and `ClipboardItem` implement this interface.

2. **Lazy `ClipboardItem` constructor** — A new constructor accepts only MIME type strings (no data), an `ExecutionContext*`, a sequence number, and a `sanitize_html` flag. This creates a "lazy" ClipboardItem.

3. **`active_readers_` map** — Tracks in-flight `ClipboardReader` instances per MIME type to keep them alive until completion and prevent duplicate reads.

4. **`representations_with_resolvers_` map** — Caches `ScriptPromiseResolver<Blob>` instances per MIME type, ensuring multiple `getType()` calls for the same type share the same promise.

---

## Low-Level Design (LLD): Object Lifecycle Analysis

### Object: `ClipboardItem` (lazy-read mode)

**Creation:**
- Created in `ClipboardPromise::ResolveRead()` via `MakeGarbageCollected<ClipboardItem>(item_mime_types_, SequenceNumber, ExecutionContext, sanitize, true)`.
- Passed `ExecutionContext*` to `ExecutionContextLifecycleObserver`, making it observe context destruction.
- The `ClipboardItem` is returned to JavaScript as part of the resolved `ClipboardItem[]` array.

**Alive:**
- Kept alive by the JavaScript garbage collector as long as JS holds a reference to it (the returned array from `clipboard.read()`).
- When `getType()` is called, it creates `ScriptPromiseResolver<Blob>` entries in `representations_with_resolvers_` and `ClipboardReader` instances in `active_readers_`. These are `Member<>` pointers (traced by Oilpan GC), so they keep the sub-objects alive while the ClipboardItem itself is alive.

**Destruction:**
- When JavaScript drops all references, the GC will trace and collect the `ClipboardItem` along with all its `Member<>` fields.
- If the `ExecutionContext` is destroyed first, `ContextDestroyed()` fires: rejects all pending promise resolvers and clears `active_readers_`.

### Object: `ClipboardReader` (and subclasses)

**Creation:**
- Created in `ClipboardItem::ReadRepresentationFromClipboardReader()` via `ClipboardReader::Create(system_clipboard, format, this, sanitize)`.
- Stored in `active_readers_` (`HeapHashMap<String, Member<ClipboardReader>>`).

**Alive:**
- Kept alive by `active_readers_` in the owning `ClipboardItem`.
- Some readers use `WrapPersistent(this)` in async callbacks (e.g., `BindOnce(&ClipboardTextReader::OnRead, WrapPersistent(this))`), which creates a **persistent handle** preventing GC collection until the callback fires. `WrapPersistent` creates a prevent-GC barrier: the GC cannot collect the reader while a persistent handle exists, even if the `Member<>` reference from `active_readers_` is cleared.
- Cross-thread operations (e.g., `ClipboardTextReader::EncodeOnBackgroundThread`) use `MakeCrossThreadHandle(this)`, which similarly prevents collection until the cross-thread callback completes.

**Destruction:**
- After `OnRead()` callback completes, `ResolveFormatData()` calls `active_readers_.erase(mime_type)`, removing the `Member<>` reference.
- However, readers may still be alive if `WrapPersistent` handles are held by pending Mojo callbacks. Once the callback fires, the persistent handle is released, and the reader becomes eligible for GC.
- On `ContextDestroyed()`, `active_readers_.clear()` drops all `Member<>` references, but pending `WrapPersistent` handles may keep readers alive until Mojo callbacks fire or are cancelled.

### Object: `ClipboardReaderResultHandler` (interface / mixin)

- A `GarbageCollectedMixin` — has no independent lifecycle. Its lifetime is the lifetime of the implementing class (`ClipboardItem` or `ClipboardPromise`).
- Referenced by `ClipboardReader` via `Member<ClipboardReaderResultHandler> result_handler_`, which is a strong traced reference. This means: **the ClipboardReader prevents its result_handler (i.e., ClipboardItem) from being garbage collected while the reader exists.**

### Object: `ScriptPromiseResolver<Blob>`

**Creation:**
- Created in `ClipboardItem::getType()` when a lazy type is requested for the first time.

**Alive:**
- Stored in `representations_with_resolvers_` (`HeapHashMap<String, Member<ScriptPromiseResolver<Blob>>>`).
- Kept alive as long as the `ClipboardItem` is alive.

**Resolution/Rejection:**
- Resolved in `ResolveFormatData()` when the reader completes.
- Rejected in `ResolveFormatData()` if clipboard changed.
- Rejected in `ContextDestroyed()` if the document is detached.
- **Note:** Calling `Reject()` on an already-resolved/rejected resolver is a **no-op** in Blink's `ScriptPromiseResolver` (it checks `is_settled_`). So double-settlement in `ContextDestroyed()` is safe but wasteful.

### Key Chromium Constructs Used

| Construct | Usage | Explanation |
|-----------|-------|-------------|
| `Member<T>` | `Member<ClipboardReader>`, `Member<ScriptPromiseResolver<Blob>>`, `Member<ClipboardReaderResultHandler>` | Oilpan GC traced pointer. Prevents the pointed-to object from being collected as long as the owner is alive and traced. Must be used in `Trace()` method. |
| `WrapPersistent(this)` | Used in `ClipboardReader` subclasses for Mojo callbacks | Creates a **persistent GC root** — prevents the wrapped object from being garbage collected even if no other `Member<>` points to it. Released when the callback fires or the persistent handle is destroyed. **Danger:** if the callback never fires, the object leaks. |
| `MakeCrossThreadHandle(this)` / `MakeUnwrappingCrossThreadHandle` | Used in text/HTML/SVG readers for background thread encoding | Similar to `WrapPersistent` but safe for cross-thread use. The object is prevented from GC until the cross-thread callback completes. |
| `MakeGarbageCollected<T>(...)` | Throughout | Allocates a GC-managed object on the Oilpan heap. The object lives until the GC determines it's unreachable. |
| `ExecutionContextLifecycleObserver` | `ClipboardItem`, `ClipboardPromise` | Observes the lifecycle of an `ExecutionContext`. `ContextDestroyed()` is called when the context (e.g., document) is torn down. This is the primary mechanism for cleaning up pending async operations when a page navigates or the frame is destroyed. |
| `HeapHashMap<K, Member<V>>` | `representations_with_resolvers_`, `active_readers_` | A GC-aware hash map where values are traced `Member<>` pointers. All entries are traced during GC marking. |
| `GarbageCollectedMixin` | `ClipboardReaderResultHandler` | A mixin class that can be used with multiple inheritance in GC-managed types. Requires a `Trace()` method. Cannot be instantiated directly — only used as a base class in `GarbageCollected<T>` hierarchies. |

---

## Detailed Findings

#### Issue #1: Resolving getType() Promise with null Blob
**Severity**: Major
**File**: `clipboard_item.cc`
**Lines**: 241-250 (`ReadRepresentationFromClipboardReader`)
**Description**:
When `GetSystemClipboard()` returns null or `ClipboardReader::Create()` returns null, `ResolveFormatData(format, nullptr)` is called. This calls `resolver->Resolve(nullptr)`, which resolves the JavaScript `getType()` promise with `null` instead of a `Blob`.

The JavaScript Async Clipboard API spec says `getType()` returns `Promise<Blob>`. Resolving with null means JavaScript code like:
```js
const blob = await clipboardItem.getType('text/plain');
const text = await blob.text(); // TypeError: Cannot read properties of null
```
...will throw an unexpected `TypeError` instead of receiving a meaningful error.

**Lifecycle Impact**: No crash in C++, but incorrect JavaScript behavior. The promise resolves successfully (with null), so the caller has no way to know the read failed unless they explicitly null-check the Blob.

**Suggestion**: Reject the promise with a `DOMException` (e.g., `DataError` or `NotFoundError`) instead of resolving with null:
```cpp
if (!system_clipboard) {
  if (auto it = representations_with_resolvers_.find(format);
      it != representations_with_resolvers_.end()) {
    it->value->Reject(MakeGarbageCollected<DOMException>(
        DOMExceptionCode::kDataError, "Failed to read clipboard data."));
  }
  return;
}
```

---

#### Issue #2: Potential Null Dereference in `ClipboardReader` Constructor
**Severity**: Major
**File**: `clipboard_reader.cc`
**Lines**: 366-372 (constructor)
**Description**:
The `ClipboardReader` base class constructor unconditionally dereferences `result_handler->GetExecutionContext()`:
```cpp
ClipboardReader::ClipboardReader(SystemClipboard* system_clipboard,
                                 ClipboardReaderResultHandler* result_handler)
    : clipboard_task_runner_(
          result_handler->GetExecutionContext()->GetTaskRunner(
              TaskType::kUserInteraction)),  // ← CRASH if GetExecutionContext() is null
```

While the current call path from `ClipboardItem::getType()` checks `!GetExecutionContext()` before reaching this point (line 196), this is fragile. The constructor assumes its precondition without defense.

**Lifecycle Impact**: If `GetExecutionContext()` returns null (e.g., if the context is destroyed between the check in `getType()` and the reader creation — impossible today since both are synchronous on the same thread, but fragile for future changes), this is a null pointer dereference crash.

**Suggestion**: Add a defensive null check or a `CHECK()`:
```cpp
ClipboardReader::ClipboardReader(SystemClipboard* system_clipboard,
                                 ClipboardReaderResultHandler* result_handler)
    : clipboard_task_runner_(
          result_handler->GetExecutionContext()
              ? result_handler->GetExecutionContext()->GetTaskRunner(
                    TaskType::kUserInteraction)
              : nullptr),
```
Or, better, add a `DCHECK(result_handler->GetExecutionContext())` to document the precondition.

---

#### Issue #3: `ContextDestroyed()` Attempts to Reject Already-Settled Resolvers
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 291-299 (`ContextDestroyed`)
**Description**:
`ContextDestroyed()` iterates all entries in `representations_with_resolvers_` and calls `Reject()` on each one. However, some resolvers may have already been resolved by `ResolveFormatData()`. The `ResolveFormatData` method resolves the promise but **does not remove the resolver from the map**.

While `ScriptPromiseResolver::Reject()` is a no-op on an already-settled resolver, this wastes cycles and is semantically sloppy.

**Suggestion**: Either (a) remove the resolver from `representations_with_resolvers_` after resolving it in `ResolveFormatData`, or (b) add a brief comment in `ContextDestroyed()` noting that double-settlement is harmless.

---

#### Issue #4: `WrapPersistent(this)` in ClipboardReader Subclasses Creates Persistent Roots After Context Destruction
**Severity**: Major
**File**: `clipboard_reader.cc`
**Lines**: Various (e.g., line 74, 141, 227, 307)
**Description**:
Several `ClipboardReader` subclasses use `WrapPersistent(this)` in Mojo callbacks:
```cpp
system_clipboard()->ReadPlainText(
    mojom::blink::ClipboardBuffer::kStandard,
    BindOnce(&ClipboardTextReader::OnRead, WrapPersistent(this)));
```

When `ContextDestroyed()` fires on the `ClipboardItem`, `active_readers_.clear()` drops the `Member<>` references to readers. **But if a Mojo callback is still pending**, the `WrapPersistent` handle keeps the reader alive. When the Mojo callback fires, the reader calls `result_handler_->OnRead(blob, mime_type)`, which calls `ClipboardItem::ResolveFormatData()`.

At this point, `ClipboardItem::ContextDestroyed()` has already run. `representations_with_resolvers_` has been cleared. So `ResolveFormatData` will find `representations_with_resolvers_.find(mime_type) == end()` and return early (line 158-161). **This is safe** — no crash occurs.

However, the reader also calls `result_handler_->GetExecutionContext()` in some reader subclasses (e.g., `ClipboardHtmlReader::Read()` line 135). After `ContextDestroyed()`, `GetExecutionContext()` returns null, but the code correctly null-checks it:
```cpp
if (ExecutionContext* context = result_handler_->GetExecutionContext()) {
    context->CountUse(...);
}
```

**Assessment**: The current code is **safe** against UAF here because:
1. `Member<ClipboardReaderResultHandler>` in the reader prevents the ClipboardItem from being GC'd while the reader exists.
2. The reader's `WrapPersistent` prevents the reader itself from being GC'd until the callback fires.
3. All call-sites properly null-check `GetExecutionContext()`.

But this creates a subtle lifecycle extension: the `ClipboardItem` (and its script wrapper) may live longer than expected due to the persistent handle chain. This could delay GC of the entire ClipboardItem and its associated JS wrapper.

**Suggestion**: Consider cancelling pending Mojo callbacks in `ContextDestroyed()` rather than relying on post-destruction null checks. Alternatively, add a comment documenting this intentional lifecycle extension.

---

#### Issue #5: `HasClipboardChangedSinceClipboardRead()` Calls `GetSystemClipboard()` After Potential Context Destruction
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 256-268 (`HasClipboardChangedSinceClipboardRead`)
**Description**:
`HasClipboardChangedSinceClipboardRead()` is called from two places:
1. `getType()` — after checking `GetExecutionContext()` is non-null (safe).
2. `ResolveFormatData()` — which is called from `OnRead()`, which is called from clipboard reader callbacks.

In path (2), the `ExecutionContext` may have been destroyed (see Issue #4). `GetSystemClipboard()` calls `GetLocalFrame()` which calls `GetExecutionContext()`. After `ContextDestroyed()`, `GetExecutionContext()` returns null, so `GetLocalFrame()` returns null, so `GetSystemClipboard()` returns null. Then `HasClipboardChangedSinceClipboardRead()` returns `true`, and `ResolveFormatData()` tries to reject the promise — but the promise resolver was already cleared by `ContextDestroyed()`, so the `find()` returns `end()` and the function returns early.

**Assessment**: Safe due to the early return in `ResolveFormatData`. The control flow is convoluted but correct.

---

#### Issue #6: Non-Lazy Constructor Passes `nullptr` to `ExecutionContextLifecycleObserver`
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 98-103
**Description**:
The non-lazy constructor initializes `ExecutionContextLifecycleObserver(nullptr)`. This means for ClipboardItems created via the JS `new ClipboardItem({...})` constructor or via the non-lazy read path, the observer is unbound. `ContextDestroyed()` will never be called, and `GetExecutionContext()` returns null.

This is safe because:
- Non-lazy items always have `is_lazy_read_ == false`.
- `getType()` takes the non-lazy branch (line 177-193) which doesn't touch `representations_with_resolvers_` or call `HasClipboardChangedSinceClipboardRead()`.

**Suggestion**: Add a comment on the non-lazy constructor explaining why `nullptr` is passed:
```cpp
// Non-lazy ClipboardItems don't observe the execution context because they
// hold pre-resolved data and don't need cleanup on context destruction.
```

---

#### Issue #7: No Validation That `ScriptPromiseResolver` is Created in a Valid ScriptState
**Severity**: Minor
**File**: `clipboard_item.cc`
**Lines**: 224-225
**Description**:
```cpp
auto* resolver = MakeGarbageCollected<ScriptPromiseResolver<Blob>>(
    script_state, exception_state.GetContext());
```
The code doesn't check `script_state->ContextIsValid()` before creating the resolver. While `getType()` checks `GetExecutionContext()` earlier, `script_state` validity is a separate concern (the V8 context could be invalid even if the ExecutionContext is alive, e.g., during early teardown).

**Suggestion**: Add a check:
```cpp
if (!script_state->ContextIsValid()) {
    exception_state.ThrowDOMException(DOMExceptionCode::kNotAllowedError,
                                      "Document detached.");
    return ScriptPromise<Blob>();
}
```

---

#### Issue #8: Missing Test Coverage for Edge Cases
**Severity**: Major
**File**: `clipboard_unittest.cc`
**Description**:
The tests cover the happy path (lazy read, getType for text), but several critical edge cases are not tested:

1. **getType() after context destruction** — Call `getType()` on a ClipboardItem after navigating away from the page. Should throw `NotAllowedError`.
2. **Concurrent getType() for the same type** — Should return the same cached promise.
3. **getType() for unsupported type** — Should throw `NotFoundError`.
4. **getType() when system clipboard returns null blob** — Should reject (per Issue #1) or resolve with null (current behavior).
5. **Promise rejection text matching** — Tests verify `IsFulfilled()` / `IsRejected()` but don't verify the error type/message.

**Suggestion**: Add tests for at least items 1-3 above.

---

#### Issue #9: Feature Flag Check Scattered Across Code
**Severity**: Suggestion
**File**: `clipboard_item.cc`, `clipboard_promise.cc`
**Description**:
`RuntimeEnabledFeatures::ReadClipboardDataOnClipboardItemGetTypeEnabled()` is checked in many places: `ClipboardItem::types()`, `ClipboardItem::getType()`, `ClipboardItem::ResolveFormatData()`, `ClipboardItem::ReadRepresentationFromClipboardReader()`, `ClipboardPromise::ResolveRead()`, `ClipboardPromise::OnReadAvailableFormatNames()`. This makes the code harder to follow.

**Suggestion**: Consider consolidating the feature check to fewer decision points, or add a helper method like `bool ClipboardItem::IsLazyReadEnabled() const { return is_lazy_read_; }` and rely on the `is_lazy_read_` flag (which is already set based on the feature flag at construction time). Several checks like the `DCHECK` in `ResolveFormatData()` could use `DCHECK(is_lazy_read_)` instead.

---

#### Issue #10: `ClipboardPromise::ReadNextRepresentation` Stores Reader but Overwrites on Concurrent Calls
**Severity**: Minor
**File**: `clipboard_promise.cc`
**Lines**: 464-474 (in diff)
**Description**:
`ClipboardPromise::ReadNextRepresentation()` now stores the reader:
```cpp
clipboard_reader_ = clipboard_reader;
```
This is a single `Member<ClipboardReader>`. If `ReadNextRepresentation` were called concurrently, the previous reader would be overwritten. However, `ReadNextRepresentation` is called sequentially (only after the previous representation completes via `OnRead → ReadNextRepresentation`), so this is safe. The earlier review comment from Rohan Raja about concurrent overwrite was addressed by moving lazy reads to `ClipboardItem` with a per-type map.

**Assessment**: Safe in the non-lazy path (sequential). The lazy path correctly uses `active_readers_` (a map) in `ClipboardItem`.

---

## Positive Observations

1. **Good design decision**: Introducing the `ClipboardReaderResultHandler` interface properly decouples `ClipboardReader` from `ClipboardPromise`, following the Interface Segregation Principle. Both `ClipboardPromise` and `ClipboardItem` can now be result handlers without tight coupling.

2. **Clipboard change detection**: The sequence number comparison logic (`HasClipboardChangedSinceClipboardRead()`) is a sensible security measure to prevent stale/inconsistent clipboard data from being returned.

3. **Promise caching**: The `representations_with_resolvers_` map correctly ensures that multiple `getType()` calls for the same MIME type return the same promise, avoiding duplicate clipboard reads.

4. **Proper GC tracing**: All new `Member<>` fields are properly traced in `Trace()` methods. The `ClipboardReaderResultHandler::Trace()` mixin trace is called. `ExecutionContextLifecycleObserver::Trace()` is called.

5. **Active reader tracking**: The `active_readers_` map is a clean solution to the concurrent reader problem identified in earlier reviews. Each MIME type has at most one active reader, tracked by the map.

6. **Feature flag gating**: The new behavior is properly gated behind `ReadClipboardDataOnClipboardItemGetType` with `status: "test"`, allowing gradual rollout.

7. **Test infrastructure additions**: The `MockClipboardHost` call-tracking booleans (`WasReadTextCalled()`, etc.) are a clean addition for verifying lazy loading behavior.

---

## Overall Assessment

**Needs changes before approval**

The CL is architecturally sound and addresses the performance concern well. The object lifecycle is generally well-managed through proper use of Oilpan GC constructs (`Member<>`, `WrapPersistent`, `GarbageCollectedMixin`, `ExecutionContextLifecycleObserver`). There are no **UAF crashes** in the current code paths due to proper GC tracing and null checks.

However, the following must be addressed:

1. **Issue #1 (Major)**: Resolving with null Blob is incorrect behavior — should reject the promise instead.
2. **Issue #2 (Major)**: Add defensive null check or DCHECK in `ClipboardReader` constructor for `GetExecutionContext()`.
3. **Issue #8 (Major)**: Critical edge cases are not tested (context destruction, concurrent getType, null blob resolution).

Issues #3, #4, #6, #7 are recommended improvements but not blockers.


---

## 📋 Review by **gpt-5.3-codex**

### Review Summary
| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ⚠️ | Potential null-deref paths from unchecked `GetSystemClipboard()` results in multiple lazy-read read-path callsites. |
| Style | ✅ | Changes are mostly consistent with Blink patterns and naming. |
| Security | ✅ | No new injection or privilege-escalation surface; permissions checks remain in place. |
| Performance | ✅ | Lazy `getType()` design avoids eager decoding/Blob creation and reduces work for unused formats. |
| Testing | ⚠️ | Good new lazy-read coverage, but no explicit detach/lifecycle race regression test for null-frame/null-clipboard transitions. |

### Detailed Findings

#### Issue #1: Unchecked `GetSystemClipboard()` can crash in lifecycle race paths
**Severity**: Major
**File**: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`
**Line**: 366, 386, 406, 467, 506, 513, 534, 555, 662
**Description**: `GetSystemClipboard()` now returns nullable (`nullptr` when `LocalFrame` is gone), but several callsites dereference it without checking. In lifecycle races (document/frame detaches between permission/read callbacks and execution), this can become a NULL crash rather than a graceful rejection. This is especially important for lazy-read because read completion now spans async callbacks and delayed `getType()` timing.
**Suggestion**: Guard each use of `GetSystemClipboard()` and reject the pending promise with `NotAllowedError` (or existing detached-document error path) when null. At minimum, null-check before `ReadAvailableCustomAndStandardFormats`, `SequenceNumber`, `ReadPlainText`, `WritePlainText`, and macOS permission calls.

#### Issue #2: Missing regression test for detach-between-read-and-getType lifecycle
**Severity**: Minor
**File**: `third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`
**Line**: 254-365
**Description**: Tests validate lazy-read behavior and targeted format reads, but do not cover the critical object-lifecycle transition where `ClipboardPromise`/`ClipboardItem` outlive frame availability and must reject without crashing. Given new observer wiring (`ExecutionContextLifecycleObserver`) and deferred reads, this is a key crash-prevention scenario.
**Suggestion**: Add a unit/integration test that detaches the document/frame after `clipboard.read()` resolves and before `getType()`, then verifies rejection (`NotAllowedError`/detached path) and no crash.

### Lifecycle / UAF / NULL-safety Analysis
- `ClipboardPromise` is GC-managed and `ExecutionContextLifecycleObserver`-backed; it now also implements `ClipboardReaderResultHandler` and owns `clipboard_reader_` to keep active readers alive.
- In lazy mode, `ClipboardPromise::ResolveRead()` creates `ClipboardItem` with MIME types + sequence number + execution context; no Blob payload is eagerly materialized.
- `ClipboardItem` is GC-managed, observes execution-context lifecycle, and stores per-type pending resolvers in `representations_with_resolvers_`; it also keeps in-flight readers alive in `active_readers_` (strong `Member<ClipboardReader>` references).
- `ClipboardReader` uses Blink GC (`Member<>`) and async safety primitives (`WrapPersistent(this)`, cross-thread handles) so callbacks do not UAF the reader object. Result delivery back to owner is via `ClipboardReaderResultHandler::OnRead(blob, mime_type)`.
- On document detach, `ClipboardItem::ContextDestroyed()` rejects all pending resolvers and clears in-flight readers; this is a good anti-UAF pattern. Remaining crash risk is primarily NULL deref from unguarded clipboard/frame access, not stale-pointer deref.

### Positive Observations
- Good architectural split: `ClipboardReaderResultHandler` cleanly decouples reader completion from `ClipboardPromise` and enables lazy `ClipboardItem` reads.
- `ClipboardItem` now explicitly tracks context lifecycle and rejects pending promises on detach, which is the correct failure mode for async API consumers.
- New tests verify core lazy-loading behavior (enumerate types first, defer actual format read to `getType()`) and include concurrent custom-format `getType()` coverage in web tests.

### Overall Assessment
Needs changes before approval.


---

## 🔀 Cross-Model Summary

This document merges reviews from **2** models: claude-opus-4.6-fast, gpt-5.3-codex.

### Model Coverage

| Model | Contributed |
|-------|------------|
| claude-opus-4.6-fast | ✅ Yes |
| gpt-5.3-codex | ✅ Yes |
