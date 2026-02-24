# CL 6978530 Review Summary: [Clipboard] Implement on-demand reading in getType()

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530  
**Author:** Shweta Bindal (shwetabindal@microsoft.com)  
**Status:** NEW (Patch Set 21)  
**Files Changed:** 13 files, +679/-121 lines  
**Bug:** [435051711](https://crbug.com/435051711)

---

## 1. Executive Summary

This CL implements on-demand (lazy) reading for the Async Clipboard API's `getType()` method. Previously, `clipboard.read()` fetched all clipboard data upfront; this change defers actual data reading to `getType()`, so only the requested MIME types are read from the system clipboard. The feature is gated behind a new `ClipboardReadOnDemand` runtime flag (status: "test") and includes clipboard change detection that rejects with `DataError` if clipboard content changes between `read()` and `getType()`.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 3 | The dual-path logic (lazy vs. eager) in `getType()` and `types()` adds complexity. Feature flag checks are scattered across multiple methods rather than encapsulated in a single code path. |
| Maintainability | 3 | Two constructors, two data structures (`representations_` vs `mime_types_`), and two resolution strategies make future maintenance harder. Parallel data structures must be kept in sync. |
| Extensibility | 4 | The `ClipboardReaderResultHandler` interface is a clean abstraction that decouples `ClipboardReader` from `ClipboardPromise`, enabling `ClipboardItem` to act as a reader target. This pattern supports future extensions. |
| Consistency | 3 | The refactor of `ClipboardReader` to use `ClipboardReaderResultHandler` is consistent, but `ClipboardPromise` now has a `clipboard_reader_` member that appears unused for the new lazy path (only used for eager path cleanup). |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                     Web Page (JavaScript)                        │
│                                                                  │
│  const items = await navigator.clipboard.read();  // Step 1      │
│  const blob = await items[0].getType("text/plain"); // Step 2    │
└──────────────┬──────────────────────────────┬────────────────────┘
               │                              │
     ┌─────────▼─────────┐         ┌──────────▼──────────┐
     │  ClipboardPromise  │         │   ClipboardItem     │
     │                    │         │   (lazy read mode)  │
     │ • read() resolves  │         │                     │
     │   with MIME types  │         │ • getType() triggers│
     │   only (no data)   │         │   on-demand read    │
     │ • Creates lazy     │         │ • Caches resolved   │
     │   ClipboardItem    │         │   promises          │
     └─────────┬──────────┘         │ • Detects clipboard │
               │                    │   changes via       │
               │ Creates            │   sequence number   │
               ▼                    └──────────┬──────────┘
     ┌────────────────────┐                    │
     │ ClipboardReader    │◄───────────────────┘
     │ ResultHandler      │     getType() creates
     │ (interface)        │     ClipboardReader per type
     ├────────────────────┤
     │ • OnRead(blob,type)│
     │ • GetExecContext() │
     │ • GetLocalFrame()  │
     └────────┬───────────┘
              │
   ┌──────────▼──────────────────────────────┐
   │        ClipboardReader Subclasses       │
   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
   │ │ PngReader│ │TextReader│ │HtmlReader│ │
   │ └──────────┘ └──────────┘ └──────────┘ │
   │ ┌──────────┐ ┌────────────────────┐    │
   │ │SvgReader │ │CustomFormatReader  │    │
   │ └──────────┘ └────────────────────┘    │
   └──────────────────┬──────────────────────┘
                      │
              ┌───────▼───────┐
              │SystemClipboard│
              │ (Mojo IPC)    │
              └───────────────┘
```

**Before (Eager Read):**
```
read() → ReadAvailableFormats → ReadText → ReadHtml → ... → Resolve all → getType() returns cached
```

**After (Lazy Read, behind flag):**
```
read() → ReadAvailableFormats → Resolve with MIME types only → getType("text/plain") → ReadText → Resolve
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 3 | Core logic is sound, but there are edge cases around error handling, TOCTOU in clipboard change detection, and resolved-promise caching semantics (see Critical Issues). Multiple CI failures across patch sets suggest fragility. |
| Efficiency | 4 | The lazy read approach is a genuine performance improvement—only requested formats are read. The per-type `ClipboardReader` instantiation and `active_readers_` map are efficient for concurrent `getType()` calls. |
| Readability | 3 | Feature flag conditionals (`RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()`) appear in 8+ locations, creating code path duplication. The dual constructor pattern and parallel data structures reduce readability. |
| Test Coverage | 3 | Three web tests and two unit tests added. Coverage is adequate for happy paths but lacks edge cases (see Test Coverage Analysis). |

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **TOCTOU Race in Clipboard Change Detection (`clipboard_item.cc:167-173`)**  
   The clipboard change check in `ResolveFormatData()` reads the sequence number *after* the data has already been fetched. Between the `ClipboardReader::Read()` call and the `ResolveFormatData()` callback, the clipboard could change and then change *back* to the same sequence number, causing a false negative. While unlikely, this is a semantic correctness issue. Additionally, the sequence number is checked only when resolving—if the clipboard changes *during* the async read, the data returned may already be stale by the time it's checked. Consider validating the sequence number both *before initiating* the read and *after* receiving the data.

2. **Resolved Promise Caching Semantics May Cause Stale Data (`clipboard_item.cc:211-214`)**  
   When `getType()` returns a cached resolved promise (line 214), the data reflects the clipboard state at the time of the *first* `getType()` call, not subsequent calls. The comment says "ClipboardItem is a snapshot at read() time," but with lazy read, data is fetched at `getType()` time, not `read()` time. If the clipboard changes between two `getType("text/plain")` calls for the same ClipboardItem, the second call returns the first call's data without re-checking the sequence number. This should be clearly documented in comments as intentional behavior, or the cache should re-validate the sequence number.

### Major Issues (Should Fix)

1. **`ClipboardReaderResultHandler::Trace` is a No-Op (`clipboard_reader.h:24`)**  
   The `Trace` method in `ClipboardReaderResultHandler` is `void Trace(Visitor* visitor) const override {}`. While subclasses override `Trace`, if a subclass forgets to call through, GC may not trace properly. This is technically correct as a mixin, but the empty body should have a comment explaining that subclasses must provide their own tracing.

2. **Missing Null Check Before `GetSystemClipboard()->SequenceNumber()` in `clipboard_promise.cc:101`**  
   In `OnReadAvailableFormatNames()`, the code calls `GetSystemClipboard()->SequenceNumber()` without checking if `GetSystemClipboard()` returns null. `GetSystemClipboard()` can return null if `GetLocalFrame()` is null. This could cause a null pointer dereference.

3. **`ExecutionContextLifecycleObserver(nullptr)` in Non-Lazy Constructor (`clipboard_item.cc:102`)**  
   The non-lazy constructor passes `nullptr` to `ExecutionContextLifecycleObserver`, meaning `ContextDestroyed()` will never be called for eager-mode `ClipboardItem`s. This is intentional but fragile—if any future code adds lazy-read-like behavior to eager `ClipboardItem`s, pending promises won't be cleaned up. A defensive comment should be added.

4. **Feature Flag Conditional Sprawl**  
   `RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()` appears in `types()`, `getType()`, `ResolveFormatData()`, `ReadRepresentationFromClipboardReader()`, the lazy constructor, `OnReadAvailableFormatNames()`, and `ResolveRead()`. Consider encapsulating the lazy/eager branching at a higher level (e.g., a strategy pattern or subclass) rather than checking the flag in every method.

### Minor Issues (Nice to Fix)

1. **Redundant `return ScriptPromise<Blob>()` Pattern (`clipboard_item.cc:199-201`)**  
   When `GetExecutionContext()` is null, the method returns an empty promise without throwing. Per Chromium conventions, this should either throw or be documented as intentional silent failure.

2. **`representations_with_resolvers_` Erase-on-Reject But Not Erase-on-Resolve (`clipboard_item.cc:163,171,175`)**  
   Rejected resolvers are erased from `representations_with_resolvers_` to allow retry, but resolved resolvers remain cached. The asymmetry is intentional for caching, but the comment on line 212 ("rejected resolvers are erased to allow retry") should also clarify that resolved ones remain for deduplication.

3. **Web Test Missing HTTPS Context (`async-clipboard-lazy-read.html`, `async-clipboard-custom-format-lazy-read.html`)**  
   The layout tests under `clipboard/async-clipboard/` do not use HTTPS (no `.https.html` suffix), while the WPT test does. The Async Clipboard API requires a secure context. These tests may rely on test infrastructure overrides, but this should be verified.

4. **Unused `Blob` Forward Declaration in `clipboard_promise.h:31`**  
   `class Blob;` is forward-declared in `clipboard_promise.h` but `Blob` is already included via `clipboard_reader.h` → `blob.h`. The forward declaration is harmless but redundant.

### Suggestions (Optional)

1. **Consider Using `base::flat_map` Instead of `HeapHashMap` for `active_readers_`**  
   Given that the number of concurrent `getType()` calls is typically small (≤5 MIME types), a `base::flat_map` would have better cache locality. However, GC requirements may necessitate `HeapHashMap`.

2. **Add UMA Histogram for Lazy Read Latency**  
   The CL adds `Blink.Clipboard.Read.NumberOfFormats` for the lazy path but doesn't track the latency improvement. A histogram comparing eager vs. lazy read completion time would validate the performance benefit.

3. **Consider Merging Constructors with a Builder or Options Struct**  
   The two constructors for `ClipboardItem` have different parameter sets and behaviors. A builder pattern or a `ClipboardItemOptions` struct could unify construction and reduce the risk of calling the wrong constructor.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | Type | What It Covers |
|------|------|----------------|
| `ClipboardTest.ReadOnlyMimeTypesInClipboardRead` | Unit test | Verifies that `read()` only calls `ReadAvailableCustomAndStandardFormats`, not `ReadText`/`ReadHtml` |
| `ClipboardTest.ClipboardItemGetTypeTest` | Unit test | Verifies that `getType("text/plain")` triggers `ReadText` and not `ReadHtml` |
| `async-clipboard-lazy-read.html` | Layout test | Verifies that `getType()` rejects with `DataError` when clipboard changes after `read()` |
| `async-clipboard-custom-format-lazy-read.html` | Layout test | Verifies lazy read for custom formats (`web text/custom`) |
| `async-custom-format-lazy-read-concurrent.tentative.https.html` | WPT | Verifies concurrent `getType()` for multiple custom formats |

### Missing Tests

1. **Multiple `getType()` calls for the same MIME type** — Verify caching behavior (second call returns same resolved promise).
2. **`getType()` after `ContextDestroyed()`** — Verify promises are rejected with `NotAllowedError`.
3. **`getType()` for unsupported type in lazy mode** — Verify `NotFoundError` is thrown.
4. **`getType()` when `SystemClipboard` is null** (frame detached mid-operation) — Verify graceful rejection.
5. **Concurrent `getType()` for standard types** (text/plain + text/html simultaneously) — Only tested for custom formats.
6. **`getType()` retry after rejection** — Verify that rejected resolvers are erased and re-calling `getType()` creates a new reader.
7. **Feature flag disabled** — Verify existing eager-read behavior is unchanged when `ClipboardReadOnDemand` is disabled.
8. **Large clipboard data** — Performance test for lazy vs. eager read with large payloads (e.g., large PNG).

### Recommended Additional Tests

- A unit test that explicitly validates the **sequence number change detection** path where clipboard changes between `read()` and `getType()`.
- An integration test verifying that **`ContextDestroyed` correctly rejects all pending `getType()` promises**.
- A test for **image (PNG) lazy read** since all existing tests focus on text and custom formats.
- A test verifying that the **eager-read path is unaffected** when the feature flag is disabled (regression safety).

---

## 6. Security Considerations

### Clipboard Data Exposure
- **Positive:** The lazy read approach reduces the window of clipboard data exposure—data is only read when explicitly requested via `getType()`, not preemptively loaded into renderer memory.
- **Positive:** Clipboard change detection via sequence number prevents reading stale data that may have been replaced with sensitive content.

### Potential Concerns
1. **Sequence Number as Security Boundary:** The clipboard sequence number is not cryptographically secure; it's a monotonic counter. A malicious page could potentially time attacks around clipboard changes. However, this is the existing mechanism used throughout Chromium's clipboard code, so no new risk is introduced.
2. **Permission Re-validation:** The CL does not re-check clipboard-read permission at `getType()` time. The permission is checked once at `read()` time. This is acceptable per the spec, but if permission is revoked between `read()` and `getType()`, the read still succeeds. This matches the existing behavior for eager reads.
3. **No Cross-Origin Leaks:** The `ClipboardItem` is bound to the creating `ExecutionContext` via `ExecutionContextLifecycleObserver`, preventing cross-context data leaks.

### Recommendations
- No additional security measures are required beyond what is already implemented.

---

## 7. Performance Considerations

### Improvements
- **Lazy read eliminates unnecessary clipboard reads.** If a page calls `read()` but only calls `getType("text/plain")`, it no longer reads HTML, PNG, SVG, etc. This is a significant win for pages that only need one format.
- **Reduced IPC overhead.** Fewer Mojo calls to the browser process during `read()`.

### Potential Concerns
1. **Per-`getType()` Overhead:** Each `getType()` call now creates a new `ClipboardReader` object and initiates a separate IPC. If a page calls `getType()` for all available formats, the total cost may exceed the eager approach due to per-call overhead (N separate IPCs vs. N sequential reads in one batch).
2. **No Batching:** Concurrent `getType()` calls create independent readers. There's no mechanism to batch multiple reads into a single IPC.
3. **Promise Resolver Retention:** `representations_with_resolvers_` retains resolved promises indefinitely (until `ClipboardItem` is GC'd). For long-lived pages, this could accumulate memory.

### Benchmarking Recommendations
- Measure IPC latency for lazy read vs. eager read with 1, 2, and 5 format types.
- Profile memory usage for long-lived `ClipboardItem` objects with cached resolvers.
- Compare total wall-clock time for `read()` → `getType()` (all types) vs. eager `read()` → `getType()` (all types).

---

## 8. Final Recommendation

**Verdict:** NEEDS_WORK

**Rationale:**
The overall design is sound—the `ClipboardReaderResultHandler` interface is a clean abstraction, and the lazy read approach is a genuine performance improvement. However, the CL has several issues that should be addressed before landing:

1. The **null check for `GetSystemClipboard()` in `clipboard_promise.cc:101`** is a potential crash that must be fixed.
2. The **TOCTOU race in clipboard change detection** should be documented if it's an accepted limitation, or mitigated with a pre-read check.
3. The **extensive feature flag conditional sprawl** makes the code harder to maintain and review. While a full refactor isn't required for this CL, the author should consider at least grouping the branching at a higher level.
4. The CL has gone through **21 patch sets with multiple CI failures**, suggesting the implementation is still stabilizing. The latest patch set (21) passes, but the pattern indicates fragility.
5. **Test coverage gaps** exist for important edge cases (context destruction, retry after rejection, PNG lazy read, concurrent standard types).

**Action Items for Author:**

1. **Fix:** Add null check for `GetSystemClipboard()` before calling `SequenceNumber()` in `ClipboardPromise::OnReadAvailableFormatNames()` (line 101 of the lazy read path).
2. **Fix:** Add a comment in `ResolveFormatData()` explaining the TOCTOU limitation of the sequence number check, or add a pre-read sequence number validation in `ReadRepresentationFromClipboardReader()`.
3. **Add:** Unit test for `getType()` after `ContextDestroyed()` to verify promise rejection.
4. **Add:** Unit test for concurrent `getType()` with standard MIME types (text/plain + text/html).
5. **Add:** Unit test verifying eager-read path is unaffected when `ClipboardReadOnDemand` flag is disabled.
6. **Consider:** Adding a comment in the non-lazy constructor explaining why `ExecutionContextLifecycleObserver(nullptr)` is acceptable.
7. **Consider:** Reducing feature flag conditionals by restructuring `getType()` and `types()` to branch once at the top level.

---

## 9. Comments for Gerrit

### File: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`

**On the lazy read path in `OnReadAvailableFormatNames()` (around the `MakeGarbageCollected<ClipboardItem>` call):**
> `GetSystemClipboard()->SequenceNumber()` is called without a null check. `GetSystemClipboard()` can return null if `GetLocalFrame()` returns null (e.g., frame detached). This could cause a null pointer dereference. Please add a null check similar to how `HasClipboardChangedSinceClipboardRead()` handles it in `clipboard_item.cc`.

### File: `third_party/blink/renderer/modules/clipboard/clipboard_item.cc`

**On `ResolveFormatData()` (line ~167):**
> The clipboard change detection check happens *after* the data has been read. This creates a TOCTOU window where the clipboard could change and change back to the same sequence number between `Read()` and `ResolveFormatData()`. While this is unlikely, could you add a brief comment acknowledging this limitation? Alternatively, consider also checking the sequence number before initiating the read in `ReadRepresentationFromClipboardReader()`.

**On `getType()` cached resolver path (line ~211-214):**
> Nit: The comment says "rejected resolvers are erased to allow retry" — could you also mention that resolved resolvers are intentionally *retained* for deduplication/caching? This helps future readers understand the asymmetry between the resolve and reject paths.

**On `getType()` when `GetExecutionContext()` is null (line ~199-201):**
> This silently returns an empty promise without throwing an exception. Should this throw `InvalidStateError` instead? Or at minimum, add a comment explaining why silent failure is acceptable here (e.g., "Returning an empty promise is safe because the context is already destroyed and no JS will observe it").

### File: `third_party/blink/renderer/modules/clipboard/clipboard_item.h`

**On the non-lazy constructor (line ~101-102 in the diff):**
> Nit: `ExecutionContextLifecycleObserver(nullptr)` in the non-lazy constructor means `ContextDestroyed()` will never fire. This is correct but non-obvious. A brief comment like `// nullptr: non-lazy ClipboardItems don't observe context lifecycle.` would help.

### File: `third_party/blink/renderer/modules/clipboard/clipboard_reader.h`

**On `ClipboardReaderResultHandler::Trace` (line ~24):**
> The empty `Trace` body is correct for a mixin, but a comment like `// Subclasses must trace their own members.` would clarify intent and prevent accidental omission in future implementations.

### File: `third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc`

**General:**
> Great test coverage for the core lazy read path! Could you also add:
> 1. A test verifying `getType()` rejects after `ContextDestroyed()`.
> 2. A test verifying concurrent `getType()` for standard types (text/plain + text/html).
> 3. A test with the `ClipboardReadOnDemand` flag *disabled* to ensure no regression in the eager-read path.

### File: `third_party/blink/renderer/platform/runtime_enabled_features.json5`

**On the new flag:**
> `ClipboardReadOnDemand` is set to `status: "test"`. Is there a plan to promote this to `status: "experimental"` or `status: "stable"` in a follow-up CL? If so, it would be helpful to mention the intended rollout timeline in the commit message or bug.
