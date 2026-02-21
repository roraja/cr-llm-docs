# CL Review Summary: [Clipboard] Implementation of Lazy Read

**CL Number:** 6978530  
**CL Title:** [Clipboard] Implementation of lazy read  
**Author:** Shweta Bindal (shwetabindal@microsoft.com)  
**URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530  
**Status:** NEW (Patch Set 15)  
**Files Changed:** 10 (+478/−50 lines)  
**Bug:** [435051711](https://crbug.com/435051711)

---

## 1. Executive Summary

This CL implements lazy reading for the Async Clipboard API on the renderer side. Instead of eagerly reading all clipboard data formats when `navigator.clipboard.read()` is called, it now only enumerates the available MIME types. Actual data is fetched on-demand when `ClipboardItem.getType()` is called, improving performance by avoiding unnecessary clipboard data deserialization and reducing IPC overhead for formats that are never consumed by the page.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 3 | The dual-path logic (feature flag gating) adds significant branching complexity across 4+ files. Every code path has `if (ClipboardReadOnDemandEnabled())` branches, making the flow harder to follow. |
| Maintainability | 3 | The feature-flag-gated dual paths will be acceptable short-term during the flag rollout, but should be cleaned up once the feature is stable. Duplication across `clipboard_reader.cc` (5 identical conditional blocks) is a concern. |
| Extensibility | 4 | The callback-based design (`read_callbacks_` map) cleanly supports per-format on-demand reading and could be extended to additional formats. |
| Consistency | 3 | The CL follows existing Chromium/Blink patterns (MojoBindings, `ScriptPromiseResolver`, `GarbageCollected`), but introduces some inconsistencies: `ClipboardItem` now has two construction paths and two data storage mechanisms (`representations_` vs `mime_types_` + `representations_with_resolvers_`). |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        EAGER PATH (existing)                           │
│                                                                        │
│  clipboard.read() ──► ClipboardPromise::HandleReadWithPermission       │
│       │                       │                                        │
│       │           ReadAvailableCustomAndStandardFormats                 │
│       │                       │                                        │
│       │           OnReadAvailableFormatNames (populates                │
│       │               clipboard_item_data_ with placeholders)          │
│       │                       │                                        │
│       │           ReadNextRepresentation() ──► ClipboardReader::Read() │
│       │                       │           (reads ALL formats eagerly)  │
│       │           OnRead(blob) ──► populates clipboard_item_data_      │
│       │                       │                                        │
│       │           ResolveRead() ──► ClipboardItem(items)               │
│       │                       │       (data already resolved)          │
│       │                                                                │
│       ▼           getType(type) ──► returns already-resolved promise   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     LAZY PATH (new, behind flag)                       │
│                                                                        │
│  clipboard.read() ──► ClipboardPromise::HandleReadWithPermission       │
│       │                       │                                        │
│       │           ReadAvailableCustomAndStandardFormats                 │
│       │                       │                                        │
│       │           OnReadAvailableFormatNames (populates                │
│       │               item_mime_types_ with type strings only)         │
│       │                       │                                        │
│       │           ResolveRead() ──► ClipboardItem(mime_types,          │
│       │               sequence_number, clipboard_promise, lazy=true)   │
│       │               (NO data read yet)                               │
│       │                                                                │
│       ▼           getType(type) ──► creates ScriptPromiseResolver      │
│                       │               stores in representations_       │
│                       │               with_resolvers_                  │
│                       │                                                │
│                   ReadRepresentationFromClipboard(format, callback)    │
│                       │                                                │
│                   PostTask ──► ReadRepresentationFromClipboardReader    │
│                       │                                                │
│                   ClipboardReader::Read() ──► OnRead(blob, mime_type)  │
│                       │                                                │
│                   Callback ──► ResolveFormatData(mime_type, blob)      │
│                       │                                                │
│                   CheckSequenceNumber() ──► Resolve or Reject          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | Core lazy-read logic is sound, but there are potential issues: (1) sequence number check in `getType()` before read starts may be premature for detecting mid-read changes, (2) `ClipboardPromise` is held alive by `ClipboardItem` which can outlive the promise's normal lifecycle, (3) concurrent `getType()` calls for the same type may overwrite the resolver. |
| Efficiency | 4 | The lazy approach avoids reading unnecessary formats, which is the core goal. PostTask for clipboard reads adds minor overhead but enables proper sequencing. |
| Readability | 3 | Heavy use of feature-flag conditionals in `clipboard_reader.cc` (5 near-identical blocks) and `clipboard_promise.cc` hurts readability. Could benefit from helper methods to reduce duplication. |
| Test Coverage | 3 | Two meaningful unit tests and one web test are added, but edge cases (concurrent getType calls, context destruction during lazy read, multiple clipboard items) are not covered. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Lifetime / Prevent-GC Risk with `ClipboardPromise` Reference:**  
   `ClipboardItem` holds a `Member<ClipboardPromise>` to call back into it for on-demand reads. `ClipboardPromise` is an `ExecutionContextLifecycleObserver` designed to be short-lived. With the lazy read path, `ClipboardItem` may keep `ClipboardPromise` alive indefinitely (as long as JavaScript holds a reference to the `ClipboardItem`). If the execution context is destroyed, `ContextDestroyed()` fires on `ClipboardPromise`, but the `ClipboardItem` still holds a reference. Subsequent `getType()` calls after context destruction could invoke `ReadRepresentationFromClipboard` which checks `GetExecutionContext()` but the error path only calls the callback with `nullptr` — it doesn't clearly communicate the context-destroyed state to the user.

2. **Sequence Number Check Placement in `getType()`:**  
   The `CheckSequenceNumber()` call at the top of `getType()` (before the read starts) only catches clipboard changes between `read()` and `getType()`. However, the clipboard can also change *during* the async read triggered by `getType()`. The `ResolveFormatData` method does a second check, but this creates an asymmetry: synchronous exception (`ThrowDOMException`) vs. promise rejection (`DOMException`). Both paths should use promise rejection for consistency, since `getType()` returns a promise. **The current code throws a synchronous exception in `getType()` which is unexpected for an async API that returns a promise.**

3. **Race Condition with Concurrent `getType()` for Same Type:**  
   If `getType("text/plain")` is called twice before the first resolves, the second call hits `representations_with_resolvers_.find(type) != end()` and returns the existing promise. However, a new `ReadRepresentationFromClipboard` call is NOT made. If the first read fails, the resolver may never be resolved/rejected for subsequent callers, or if it succeeds, duplicate calls are harmless. This should be explicitly documented or handled.

### Major Issues (Should Fix)

1. **Duplicated Feature-Flag Branching in `clipboard_reader.cc`:**  
   Five nearly identical `if/else` blocks in `clipboard_reader.cc` for `OnRead(blob, mime_type)` vs `OnRead(blob)`. This should be refactored — either `OnRead` should always pass the mime_type (ignoring it when not needed), or a single helper should handle the dispatch. As reviewer Ashish Kumar noted: "Do we need a separate method for this? Blob already has the mime type."

2. **`WrapPersistent(this)` in `ReadRepresentationFromClipboard`:**  
   `WrapPersistent` prevents garbage collection of `ClipboardPromise`. Combined with `PostTask`, this can leak `ClipboardPromise` objects if the task runner is shutting down. Consider using `WrapWeakPersistent` with a null check, or using the cancellable closure pattern.

3. **Missing Validation for Empty `mime_types_`:**  
   In `ResolveRead()` (lazy path), if `item_mime_types_` is empty, a `ClipboardItem` with zero types is created and the promise resolves with it. The eager path skips entries with null blobs. The lazy path should handle the empty-types case consistently.

4. **`HashMap<String, base::OnceCallback>` is Not Garbage-Collected:**  
   `read_callbacks_` in `ClipboardPromise` stores `base::OnceCallback` which captures `WrapPersistent(this)` of `ClipboardItem`. This is a raw `HashMap`, not a `HeapHashMap`, but it stores pointers to GC-managed objects via the callback. If callbacks are never consumed (e.g., read fails without calling the callback), these prevent GC of `ClipboardItem`. Consider cleanup on `ContextDestroyed()`.

### Minor Issues (Nice to Fix)

1. **Comments and Naming:**
   - `CheckSequenceNumber()` — reviewer suggested `IsSequenceNumberValid()` which is more descriptive. (Acknowledged but appears not yet addressed in latest patchset.)
   - The commit message could be more descriptive about the renderer-side scope (per reviewer comment).

2. **`mock_clipboard_host.cc` Reset Logic:**  
   The call tracking booleans (`read_text_called_`, etc.) are reset in `Reset()` but not in the constructor initializer list body — they're initialized via default member initializers in the header. This is fine but the placement of the reset in `Reset()` should have a comment explaining it covers the mock-reuse scenario.

3. **Web Test Coverage:**  
   The web test `async-clipboard-lazy-read.html` only tests the clipboard-change-detection path. It doesn't test the happy path (successful lazy read) or edge cases (reading types not in clipboard, calling getType multiple times).

4. **Missing `EXPECT_FALSE(mock_clipboard_host()->WasReadHtmlCalled())` in test:**  
   Reviewer Prashant noted this is missing in `ReadOnlyMimeTypesInClipboardRead` test. Both text and HTML are written but only text read is checked.

### Suggestions (Optional)

1. **Consider Using `AbortController`/`AbortSignal` Pattern:**  
   For long-lived lazy reads, consider allowing cancellation via an `AbortSignal` passed through `ClipboardReadOptions`. This aligns with modern Web API patterns.

2. **Metrics for Lazy Read:**  
   Consider adding UMA histograms specifically for lazy read: time-between-read-and-getType, format-hit-rate (how many types enumerated vs. actually read), and clipboard-change-rejection-rate.

3. **Feature Flag Cleanup Plan:**  
   Document in the code or bug when the feature flag (`ClipboardReadOnDemand`) will be promoted from `"test"` to `"stable"`, and when the dual-path code will be cleaned up.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | What It Covers |
|------|---------------|
| `ReadOnlyMimeTypesInClipboardRead` (unit test) | Verifies that `clipboard.read()` in lazy mode only calls `ReadAvailableCustomAndStandardFormats` and NOT `ReadText`/`ReadHtml`. Proves lazy loading works. |
| `ClipboardItemGetTypeTest` (unit test) | Verifies that calling `getType("text/plain")` after lazy `read()` triggers the actual `ReadText` call on the mock clipboard host. |
| `async-clipboard-lazy-read.html` (web test) | Tests that `getType()` throws `DataError` when clipboard content changes between `read()` and `getType()`. |

### Missing Tests

1. **Happy path lazy read end-to-end:** Verify that `getType()` actually returns the correct blob data content (not just that `ReadText` was called).
2. **Multiple format reads:** Call `getType("text/plain")` and `getType("text/html")` on the same `ClipboardItem` and verify both resolve correctly.
3. **Concurrent `getType()` calls for same type:** Call `getType("text/plain")` twice and verify both promises resolve.
4. **Unknown type handling:** Call `getType("application/unsupported")` on a lazy-read `ClipboardItem` and verify it throws `NotFoundError`.
5. **Context destruction during lazy read:** Destroy the execution context between `read()` and `getType()` — verify no crash/leak.
6. **Custom format lazy read:** Test with web custom formats (prefixed with "web ").
7. **Feature-flag-off regression:** Verify existing clipboard behavior is unchanged when `ClipboardReadOnDemand` is disabled.

### Recommended Additional Tests

- **Stress test:** Rapid alternating `read()` and `getType()` calls with clipboard changes.
- **TSAN coverage:** The `ClipboardItemGetTypeTest` previously failed under TSAN (Patch Set 11). Ensure the fix is validated under thread sanitizer.

---

## 6. Security Considerations

| Concern | Assessment |
|---------|-----------|
| **Clipboard data exposure window** | In the eager path, clipboard data is read once and stored. In the lazy path, data is read at `getType()` time, which could be significantly later. This widens the window during which the page has implicit clipboard access after the permission grant. However, the sequence number check mitigates reading stale data from a different copy operation. |
| **Sequence number spoofing** | The sequence number comes from `SystemClipboard::SequenceNumber()` via Mojo IPC. A compromised renderer could bypass this check, but the lazy-read design is renderer-side only, so the browser process clipboard host still controls actual data access. No new attack surface. |
| **Cross-origin data leakage** | No new cross-origin risks. The lazy read still goes through the same permission checks and sanitization (HTML sanitizer, etc.) as the eager path. |
| **Denial of service** | A page could call `read()` many times, creating many `ClipboardPromise` objects that are kept alive by `ClipboardItem` references. This is mitigated by normal GC but worth monitoring. |

**Recommendation:** The security posture is acceptable. The main risk is the extended lifetime of `ClipboardPromise` objects, which should be addressed by proper cleanup in `ContextDestroyed()`.

---

## 7. Performance Considerations

| Aspect | Assessment |
|--------|-----------|
| **IPC reduction** | ✅ Positive: `clipboard.read()` no longer triggers N format reads over Mojo IPC. Only `ReadAvailableCustomAndStandardFormats` is called eagerly. Each `getType()` triggers one format read. Net benefit when pages don't read all formats. |
| **Memory** | Neutral to slightly positive: No pre-allocated blob data for unused formats. However, `ClipboardPromise` objects live longer. |
| **Latency** | Mixed: `clipboard.read()` resolves faster (no data reads). But `getType()` now incurs the full read latency that was previously hidden. For pages that immediately call `getType()` on all formats, total latency may be slightly higher due to PostTask overhead. |
| **PostTask overhead** | Minor concern: `ReadRepresentationFromClipboard` posts a task to read each format. Multiple concurrent `getType()` calls each post a separate task. Consider batching if this becomes measurable. |

**Benchmarking Recommendations:**
1. Measure `clipboard.read()` → `getType()` latency with the Async Clipboard API Performance tests.
2. Compare total time for "read all formats" scenario (eager vs. lazy) to quantify PostTask overhead.
3. Profile memory under sustained clipboard reads with lazy-read enabled.

---

## 8. Final Recommendation

**Verdict:** NEEDS_WORK

**Rationale:**

The CL implements a sound design for lazy clipboard reading that aligns with web platform trends toward on-demand data access. The core architecture (defer reads to `getType()` time, use sequence numbers for change detection) is correct. However, several issues need addressing before this is ready to land:

1. **Critical:** The synchronous exception in `getType()` for sequence number failure is inconsistent with the async promise-returning API contract. This should reject the promise instead.
2. **Critical:** The `ClipboardPromise` lifetime management needs clarification — holding a GC reference from `ClipboardItem` to `ClipboardPromise` creates an unusual ownership pattern that may cause issues when the execution context is destroyed.
3. **Major:** The duplicated feature-flag branching across 5 locations in `clipboard_reader.cc` should be consolidated.
4. **Major:** Cleanup of `read_callbacks_` on context destruction is needed to prevent leaks.
5. **Test gaps:** The happy-path end-to-end test (verifying actual data content) and concurrent-getType test are missing.

The CL has been through 15 patch sets with good reviewer engagement from Ashish Kumar and Prashant Nevase. Several of their comments appear addressed but the latest reviews (Patch Set 15) still have outstanding comments.

**Action Items for Author:**

1. **Change `getType()` sequence number failure from synchronous throw to promise rejection** — async APIs should not throw synchronously for runtime errors.
2. **Add cleanup of `read_callbacks_` in `ClipboardPromise::ContextDestroyed()`** to prevent leaking `ClipboardItem` pointers.
3. **Refactor `clipboard_reader.cc`** to eliminate the 5 duplicated `if (ClipboardReadOnDemandEnabled())` blocks — consider always passing mime_type to `OnRead()` and using a single overload.
4. **Add missing test assertion** `EXPECT_FALSE(mock_clipboard_host()->WasReadHtmlCalled())` in `ReadOnlyMimeTypesInClipboardRead` per reviewer feedback.
5. **Add an end-to-end test** that verifies `getType()` returns correct blob data content (not just that the underlying read was triggered).
6. **Address remaining reviewer comments** from Prashant Nevase (Patch Set 15 — 7 comments) and Ashish Kumar (Patch Set 15 — 2 comments).
7. **Rename `CheckSequenceNumber()`** to `IsSequenceNumberValid()` per reviewer suggestion.
8. **Consider `WrapWeakPersistent`** instead of `WrapPersistent` for the PostTask in `ReadRepresentationFromClipboard` to allow GC if the promise is no longer needed.

---

## 9. Comments for Gerrit

### Comment 1: `clipboard_item.cc` — `getType()` method (synchronous throw)

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_item.cc`  
**Location:** `getType()` method, around the `CheckSequenceNumber()` call before `representations_with_resolvers_`

> The `CheckSequenceNumber()` failure here throws a synchronous `DOMException` via `exception_state.ThrowDOMException()`. Since `getType()` returns a `ScriptPromise<Blob>`, callers expect errors to come as promise rejections, not synchronous throws. This is inconsistent with the rejection in `ResolveFormatData()` which correctly rejects the promise. Please change this to create a rejected promise instead:
> ```cpp
> auto* resolver = MakeGarbageCollected<ScriptPromiseResolver<Blob>>(
>     script_state, exception_state.GetContext());
> resolver->Reject(MakeGarbageCollected<DOMException>(
>     DOMExceptionCode::kDataError, "Clipboard data has changed"));
> return resolver->Promise();
> ```

### Comment 2: `clipboard_reader.cc` — Duplicated feature flag checks

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_reader.cc`  
**Location:** All 5 `OnRead`/`NextRead` methods

> There are 5 near-identical `if (ClipboardReadOnDemandEnabled())` blocks that differ only in the mime_type argument. Consider making `OnRead()` always accept a mime_type parameter (defaulting to empty string or the reader's known type), and having `ClipboardPromise::OnRead()` dispatch to the correct handler based on whether the feature is enabled. This would eliminate the duplication entirely. Alternatively, add a helper on the reader base class.

### Comment 3: `clipboard_promise.cc` — `ReadRepresentationFromClipboard` lifetime

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`  
**Location:** `ReadRepresentationFromClipboard()` method

> `WrapPersistent(this)` in the `PostTask` call prevents GC of `ClipboardPromise` until the task runs. If the execution context is destroyed before the posted task executes, the promise will be kept alive unnecessarily. Consider:
> 1. Using `WrapWeakPersistent(this)` with a null check in the callback, or
> 2. Clearing `read_callbacks_` in `ContextDestroyed()` to break the prevent-GC chain from callback → `ClipboardItem`.

### Comment 4: `clipboard_promise.h` — `read_callbacks_` cleanup

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_promise.h`  
**Location:** `read_callbacks_` member declaration

> `read_callbacks_` stores `base::OnceCallback<void(const String&, Blob*)>` which captures `WrapPersistent(ClipboardItem*)` via `BindOnce`. These callbacks are never cleared on context destruction. Please add cleanup in `ContextDestroyed()`:
> ```cpp
> read_callbacks_.clear();
> ```

### Comment 5: `clipboard_unittest.cc` — Missing assertion

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`  
**Location:** `ReadOnlyMimeTypesInClipboardRead` test, after `WasReadTextCalled` assertion

> Per Prashant's review comment: please add `EXPECT_FALSE(mock_clipboard_host()->WasReadHtmlCalled());` to verify that HTML read is also not triggered during `clipboard.read()`.

### Comment 6: `clipboard_item.h` — Naming

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_item.h`  
**Location:** `CheckSequenceNumber()` declaration

> nit: Consider renaming to `IsSequenceNumberValid()` as suggested by Prashant — it's more descriptive of the method's purpose as a predicate.

### Comment 7: `clipboard_promise.cc` — Missing comment on eager path

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`  
**Location:** `ResolveRead()` method, in the `else` branch (eager path)

> Per Prashant's comment: please add a comment explaining what this code path does vs. the new lazy path, e.g.:
> ```cpp
> // Eager read path: clipboard data has already been read into
> // clipboard_item_data_. Create a ClipboardItem with resolved promises.
> ```

### Comment 8: General — Test coverage

**File:** `third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`

> Consider adding tests for:
> 1. End-to-end happy path: verify `getType()` returns a blob with the correct content (not just that the mock was called).
> 2. Concurrent `getType()` calls for the same MIME type returning the same promise.
> 3. `getType()` for a type not in the clipboard returning `NotFoundError`.
