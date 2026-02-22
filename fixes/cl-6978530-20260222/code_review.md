# Code Review: CL 6978530 — [Clipboard] Implementation of lazy read

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/6978530
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Reviewers:** Ashish Kumar, Prashant Nevase
**Files Changed:** 10 files, +478/-50 lines

---

## Review Summary

| Category    | Status | Notes |
|-------------|--------|-------|
| Correctness | ⚠️     | Concurrent `getType()` race condition; `read_callbacks_` leak on context destruction; missing custom format support in lazy path |
| Style       | ⚠️     | Minor formatting issues; missing blank lines; some naming could be improved |
| Security    | ⚠️     | Potential UAF vector on context destruction; `sequence_number_` default of 0 can false-match error state |
| Performance | ✅     | Core design (lazy read) is a performance improvement; sequence number check may incur IPC |
| Testing     | ⚠️     | Good basic coverage; missing edge case tests for concurrent `getType()`, context destruction, and custom formats |

---

## High-Level Design (HLD) — Clipboard Read Architecture

### Existing (Eager) Read Architecture

```
┌──────────────┐    clipboard.read()    ┌──────────────────┐
│  JavaScript  │ ─────────────────────► │ ClipboardPromise │
│  (web page)  │                        │   ::CreateForRead │
└──────┬───────┘                        └────────┬─────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ ValidatePreconditions    │
       │                                │ (Permission check)       │
       │                                └────────┬─────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ HandleReadWithPermission │
       │                                │ → ReadAvailableCustom    │
       │                                │   AndStandardFormats()   │
       │                                └────────┬─────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ OnReadAvailableFormatNames│
       │                                │ → Populates              │
       │                                │   clipboard_item_data_   │
       │                                │ → ReadNextRepresentation │
       │                                └────────┬─────────────────┘
       │                                         │
       │                              ┌──────────┴──────────┐
       │                              │  FOR EACH FORMAT:   │
       │                              │                     │
       │                              ▼                     │
       │                     ┌─────────────────┐            │
       │                     │ ClipboardReader  │            │
       │                     │ ::Create(format) │            │
       │                     │ ::Read()         │            │
       │                     │ (IPC + encode)   │            │
       │                     └───────┬─────────┘            │
       │                             │ OnRead(blob)         │
       │                             ▼                      │
       │                     ┌─────────────────┐            │
       │                     │ Store blob in    │            │
       │                     │ clipboard_item_  │◄───────────┘
       │                     │ data_[i]         │
       │                     └───────┬─────────┘
       │                             │ All done
       │                             ▼
       │                     ┌─────────────────┐
       │                     │ ResolveRead()    │
       │                     │ → Create         │
       │                     │   ClipboardItem  │
       │                     │   with resolved  │
       │                     │   data           │
       │                     └───────┬─────────┘
       │                             │
       ◄─────────────────────────────┘
       │  Promise<ClipboardItem[]> resolved
       ▼
 ┌──────────────┐
 │  JS calls    │  item.getType("text/plain")
 │  getType()   │──► Returns already-resolved promise
 └──────────────┘
```

**Key property:** All data is read eagerly during `clipboard.read()`. By the time
`getType()` is called, blobs are already available.

### New (Lazy) Read Architecture

```
┌──────────────┐    clipboard.read()    ┌──────────────────┐
│  JavaScript  │ ─────────────────────► │ ClipboardPromise │
│  (web page)  │                        │   ::CreateForRead │
└──────┬───────┘                        └────────┬─────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ ValidatePreconditions    │
       │                                │ (Permission check)       │
       │                                └────────┬─────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ HandleReadWithPermission │
       │                                │ → ReadAvailableCustom    │
       │                                │   AndStandardFormats()   │
       │                                └────────┬─────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ OnReadAvailableFormatNames│
       │                                │ → Stores MIME types in   │
       │                                │   item_mime_types_       │
       │                                │ → ResolveRead() IMMEDIATELY
       │                                └────────┬─────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌──────────────────────────┐
       │                                │ ResolveRead()            │
       │                                │ → Create ClipboardItem   │
       │                                │   with mime_types ONLY   │
       │                                │   (NO data yet)          │
       │                                │ → Pass `this` (Promise)  │
       │                                │   to ClipboardItem       │
       │                                └────────┬─────────────────┘
       │                                         │
       ◄─────────────────────────────────────────┘
       │  Promise<ClipboardItem[]> resolved (no data yet!)
       ▼
 ┌──────────────┐
 │  JS calls    │  item.getType("text/plain")
 │  getType()   │
 └──────┬───────┘
        │
        ▼
 ┌─────────────────────────────────────────────────────┐
 │ ClipboardItem::getType()                            │
 │  1. CheckSequenceNumber() — clipboard changed?      │
 │  2. Create ScriptPromiseResolver<Blob>              │
 │  3. Store in representations_with_resolvers_        │
 │  4. Call clipboard_promise_→                        │
 │     ReadRepresentationFromClipboard(format, cb)     │
 └─────────────────┬───────────────────────────────────┘
                   │
                   ▼
 ┌─────────────────────────────────────────────────────┐
 │ ClipboardPromise::ReadRepresentationFromClipboard   │
 │  1. Store callback in read_callbacks_[format]       │
 │  2. PostTask → ReadRepresentationFromClipboardReader│
 └─────────────────┬───────────────────────────────────┘
                   │ (async via PostTask)
                   ▼
 ┌─────────────────────────────────────────────────────┐
 │ ReadRepresentationFromClipboardReader               │
 │  → ClipboardReader::Create(format)                  │
 │  → clipboard_reader_ = reader (OVERWRITES!)         │
 │  → reader→Read()                                    │
 └─────────────────┬───────────────────────────────────┘
                   │ (async IPC + background encoding)
                   ▼
 ┌─────────────────────────────────────────────────────┐
 │ ClipboardPromise::OnRead(blob, mime_type)           │
 │  → Find callback in read_callbacks_[mime_type]      │
 │  → Execute callback → ClipboardItem::ResolveFormatData│
 └─────────────────┬───────────────────────────────────┘
                   │
                   ▼
 ┌─────────────────────────────────────────────────────┐
 │ ClipboardItem::ResolveFormatData(mime_type, blob)   │
 │  1. Check blob != nullptr                           │
 │  2. CheckSequenceNumber() — clipboard changed?      │
 │  3. Resolve/Reject the ScriptPromiseResolver        │
 └─────────────────────────────────────────────────────┘
```

---

## Low-Level Design (LLD) — Object Lifecycles & Data Types

### Object Lifecycle: `ClipboardPromise`

**Type:** `GarbageCollected<ClipboardPromise>`, `ExecutionContextLifecycleObserver`

**Creation:** `MakeGarbageCollected<ClipboardPromise>(context, resolver, exception_state)` in `CreateForRead()`.

**Ownership graph (eager read):**
```
ScriptPromiseResolver → (weak via Oilpan trace)
    └── ClipboardPromise (alive during read pipeline)
          ├── script_promise_resolver_ (Member)
          ├── clipboard_writer_ (Member)
          ├── clipboard_reader_ (Member) — single reader at a time
          ├── permission_service_ (HeapMojoRemote)
          └── clipboard_item_data_ (HeapVector)
```
After `ResolveRead()` resolves the outer promise, no strong JS references keep `ClipboardPromise` alive **unless** something else traces it. In the eager path, this is fine — the data is already in the `ClipboardItem`.

**Ownership graph (lazy read):**
```
JavaScript ──► ClipboardItem
                ├── clipboard_promise_ (Member<ClipboardPromise>) ★ NEW
                │     ├── clipboard_reader_ (Member) — ONLY latest reader!
                │     ├── read_callbacks_ (HashMap<String, OnceCallback>)
                │     │     └── WrapPersistent(ClipboardItem) — prevents GC
                │     └── item_mime_types_ (HeapVector)
                └── representations_with_resolvers_ (HeapHashMap)
                      └── ScriptPromiseResolver<Blob> per type
```
In the lazy path, `ClipboardPromise` is kept alive by `ClipboardItem::clipboard_promise_` (traced). The `ClipboardItem` is kept alive by JS. So the `ClipboardPromise` lifecycle extends until the `ClipboardItem` is GC'd.

**Destruction:** Via Oilpan GC when no more references exist, or `ContextDestroyed()` is called.

**⚠️ LIFECYCLE CONCERN:** After `ContextDestroyed()`, the `ClipboardPromise` rejects the main promise and clears `clipboard_writer_` and `clipboard_reader_`, but does NOT:
- Clear `read_callbacks_` (which holds `WrapPersistent` references)
- Reject pending promises in `representations_with_resolvers_` (owned by `ClipboardItem`)

### Object Lifecycle: `ClipboardItem`

**Type:** `ScriptWrappable` (GarbageCollected)

**Creation (eager):** `MakeGarbageCollected<ClipboardItem>(items, sequence_number)` — called from `ResolveRead()` with already-resolved blob data.

**Creation (lazy):** `MakeGarbageCollected<ClipboardItem>(mime_types, sequence_number, this, true)` — called from `ResolveRead()` with only MIME type names, no data.

**Ownership graph (lazy):**
```
ClipboardItem (ScriptWrappable, held by JS)
  ├── representations_ (HeapVector) — EMPTY in lazy path
  ├── mime_types_ (HeapVector<String>) — list of available types
  ├── representations_with_resolvers_ (HeapHashMap<String, Member<ScriptPromiseResolver<Blob>>>)
  │     └── Populated lazily as getType() is called
  ├── clipboard_promise_ (Member<ClipboardPromise>) — back-reference
  ├── custom_format_types_ (Vector<String>) — ⚠️ EMPTY, not populated!
  ├── sequence_number_ (absl::uint128) — captured at creation time
  ├── is_lazy_read_ (bool) — true
  ├── creation_time_ (base::TimeTicks)
  └── last_get_type_calls_ (HashMap<String, base::TimeTicks>)
```

**Destruction:** Via Oilpan GC when JS releases the reference.

**⚠️ LIFECYCLE CONCERN:** `ClipboardItem` has a strong reference (`Member`) to `ClipboardPromise`. This extends the `ClipboardPromise`'s lifetime to match the `ClipboardItem`'s, even if the `ClipboardPromise` has finished its main job (resolving the outer read promise). This is necessary for lazy reads but means the `ClipboardPromise` and all its members (including `permission_service_`, `script_promise_resolver_`, etc.) stay alive longer than needed.

### Object Lifecycle: `ClipboardReader`

**Type:** `GarbageCollected<ClipboardReader>` (abstract base class)

**Subclasses:** `ClipboardPngReader`, `ClipboardTextReader`, `ClipboardHtmlReader`, `ClipboardSvgReader`, `ClipboardCustomFormatReader`

**Creation:** `ClipboardReader::Create(system_clipboard, format, promise, sanitize_html)` — factory method.

**Ownership (eager):**
```
ClipboardPromise
  └── clipboard_reader_ (Member) — single reader, sequential reads
        ├── promise_ (Member<ClipboardPromise>) — back-reference
        └── system_clipboard_ (Member<SystemClipboard>)
```

**Ownership (lazy):**
```
ClipboardPromise
  └── clipboard_reader_ (Member) — ⚠️ ONLY holds LATEST reader!
        ├── promise_ (Member<ClipboardPromise>)
        └── system_clipboard_ (Member<SystemClipboard>)

In-flight readers (not in clipboard_reader_):
  └── Kept alive by WrapPersistent/MakeCrossThreadHandle in async callbacks
```

**Destruction:** Via Oilpan GC after read completes and references are released.

**⚠️ LIFECYCLE CONCERN (UAF RISK):** In the concurrent lazy read scenario:
1. `getType("text/plain")` → creates `ClipboardTextReader` A → `clipboard_reader_ = A`
2. `getType("text/html")` → creates `ClipboardHtmlReader` B → `clipboard_reader_ = B` (A is overwritten!)
3. Reader A is doing async encoding on a background thread. It's kept alive by `MakeCrossThreadHandle(this)` in the worker pool task.
4. If `ContextDestroyed()` is called, only `clipboard_reader_` (B) is cleared. Reader A is not tracked and may try to call `promise_->OnRead()` after context destruction.

### Data Types Used

| Data Type | Location | Description |
|-----------|----------|-------------|
| `absl::uint128` | `ClipboardItem::sequence_number_`, `ClipboardPromise::GetSequenceNumberToken()` | 128-bit unsigned integer representing clipboard sequence number. Used to detect clipboard changes between `read()` and `getType()`. Default value of 0 is problematic (see Issue #8). |
| `HeapVector<String>` | `ClipboardItem::mime_types_`, `ClipboardPromise::item_mime_types_` | Oilpan-traced vector of WTF::String. Stores MIME type names (e.g., "text/plain", "text/html", "image/png"). Used in lazy path instead of `clipboard_item_data_`. |
| `HeapHashMap<String, Member<ScriptPromiseResolver<Blob>>>` | `ClipboardItem::representations_with_resolvers_` | Oilpan-traced hash map from MIME type string to a ScriptPromiseResolver. Each resolver corresponds to one `getType()` call. Populated lazily. |
| `Member<ClipboardPromise>` | `ClipboardItem::clipboard_promise_` | Oilpan traced pointer (strong reference). Prevents GC of `ClipboardPromise` while `ClipboardItem` is alive. |
| `HashMap<String, base::OnceCallback<void(const String&, Blob*)>>` | `ClipboardPromise::read_callbacks_` | Non-traced hash map storing one-shot callbacks per MIME type. The callbacks hold `WrapPersistent(ClipboardItem)` references. **Not cleared in ContextDestroyed.** |
| `Member<ClipboardReader>` | `ClipboardPromise::clipboard_reader_` | Oilpan traced pointer to the current/latest ClipboardReader. In lazy path, overwritten by concurrent reads. |
| `MemberScriptPromise<V8UnionBlobOrString>` | Inside `representations_` pairs | Traced V8 promise wrapping either a Blob or String union type. Used in eager path only. |
| `base::OnceCallback<void(const String&, Blob*)>` | Passed from `ClipboardItem::getType()` to `ClipboardPromise::ReadRepresentationFromClipboard()` | One-shot callback bound to `ClipboardItem::ResolveFormatData` via `WrapPersistent`. Consumed when the read completes. |
| `bool is_lazy_read_` | `ClipboardItem` | Flag distinguishing lazy-read ClipboardItems from eager-read ones. |
| `CrossThreadHandle<T>` | Inside `ClipboardReader` subclasses | Prevents GC of reader objects while background thread encoding is in progress. Releases when the cross-thread task completes. |
| `WrapPersistent(this)` | Various `BindOnce` calls | Creates a persistent GC root preventing the bound object from being collected. Used for async callbacks that outlive the normal trace graph. |

### UAF Analysis Summary

| Scenario | Object at Risk | Root Cause | Severity |
|----------|---------------|------------|----------|
| Concurrent `getType()` + `ContextDestroyed()` | `ClipboardReader` (previous readers) | `clipboard_reader_` only holds latest; `ContextDestroyed()` only clears latest | **Critical** |
| Context destroyed with pending `read_callbacks_` | `ClipboardItem` via `WrapPersistent` | `read_callbacks_` not cleared in `ContextDestroyed()` | **Major** (memory leak / use-after-destroy) |
| Context destroyed with pending `representations_with_resolvers_` | `ScriptPromiseResolver` objects | Not rejected on context destruction | **Minor** (resolvers in detached context are harmless) |
| `GetSequenceNumberToken()` returns 0 when frame is null | N/A (logic error) | 0 matches default `sequence_number_` | **Minor** (false positive "valid" check) |

---

## Detailed Findings

### Issue #1: Concurrent `getType()` calls overwrite `clipboard_reader_`

**Severity**: Critical
**File**: `clipboard_promise.cc`
**Line**: 357
**Description**: In `ReadRepresentationFromClipboardReader()`, `clipboard_reader_ = clipboard_reader` overwrites the previous reader. When multiple `getType()` calls are made concurrently (e.g., `getType("text/plain")` and `getType("text/html")`), only the latest reader is tracked. Previous readers doing background work are kept alive by `CrossThreadHandle`/`WrapPersistent` but are invisible to the `ClipboardPromise`. When `ContextDestroyed()` is called, only the latest reader is cleared (`clipboard_reader_.Clear()`), leaving earlier in-flight readers potentially calling back into a destroyed context.

In the eager read path, this was safe because `ReadNextRepresentation()` is sequential—one reader at a time, advancing `clipboard_representation_index_` only after `OnRead()`. The lazy path breaks this invariant.

**Suggestion**: Use a `HeapHashMap<String, Member<ClipboardReader>>` (keyed by MIME type) instead of a single `Member<ClipboardReader>` to track all in-flight readers. Clear all of them in `ContextDestroyed()`.

```cpp
// In clipboard_promise.h:
HeapHashMap<String, Member<ClipboardReader>> active_readers_;

// In ReadRepresentationFromClipboardReader:
active_readers_.Set(format, clipboard_reader);

// In OnRead(blob, mime_type):
active_readers_.erase(mime_type);

// In ContextDestroyed:
active_readers_.clear();
```

---

### Issue #2: `read_callbacks_` not cleared in `ContextDestroyed()`

**Severity**: Critical
**File**: `clipboard_promise.cc`
**Line**: 844-852
**Description**: `ContextDestroyed()` clears `clipboard_writer_` and `clipboard_reader_` but does not clear `read_callbacks_`. Each callback in `read_callbacks_` holds a `WrapPersistent(ClipboardItem)` reference (from the `BindOnce` in `getType()`, line 200 of `clipboard_item.cc`). These persistent references prevent the `ClipboardItem` from being GC'd even after context destruction. Additionally, if a reader completes after context destruction and calls `OnRead(blob, mime_type)`, it will find and execute the callback, calling `ClipboardItem::ResolveFormatData()` which may try to resolve/reject a `ScriptPromiseResolver` on a destroyed context.

**Suggestion**: Clear `read_callbacks_` in `ContextDestroyed()`:
```cpp
void ClipboardPromise::ContextDestroyed() {
  script_promise_resolver_->Reject(MakeGarbageCollected<DOMException>(
      DOMExceptionCode::kNotAllowedError, "Document detached."));
  clipboard_writer_.Clear();
  clipboard_reader_.Clear();
  read_callbacks_.clear();  // Release WrapPersistent references
}
```

---

### Issue #3: Custom format types not populated in lazy read constructor

**Severity**: Major
**File**: `clipboard_item.cc`
**Line**: 73-85
**Description**: The new `ClipboardItem` constructor for lazy read only stores MIME types in `mime_types_` but does not process them through `Clipboard::ParseWebCustomFormat()`. The existing constructor (line 87-122) parses web custom formats (types with "web " prefix) and populates `custom_format_types_`. This means `CustomFormats()` always returns empty for lazy-read `ClipboardItem`s, potentially breaking web custom format support.

**Suggestion**: Process custom formats in the lazy read constructor:
```cpp
ClipboardItem::ClipboardItem(const HeapVector<String>& mime_types, ...)
    : ... {
  for (const auto& mime_type : mime_types) {
    String web_custom_format = Clipboard::ParseWebCustomFormat(mime_type);
    if (!web_custom_format.empty()) {
      String web_custom_format_string =
          String::Format("%s%s", ui::kWebClipboardFormatPrefix,
                         web_custom_format.Utf8().c_str());
      mime_types_.emplace_back(web_custom_format_string);
      custom_format_types_.push_back(web_custom_format_string);
    } else {
      mime_types_.emplace_back(mime_type);
    }
  }
}
```

---

### Issue #4: `GetSequenceNumberToken()` returning 0 matches default `sequence_number_`

**Severity**: Major
**File**: `clipboard_promise.cc` / `clipboard_item.cc`
**Lines**: 832-842 / 74
**Description**: `GetSequenceNumberToken()` returns `0` when the frame or clipboard is null (error case). The lazy-read `ClipboardItem` constructor has `sequence_number` default value of `0`. If for some reason the sequence number is 0 at creation time, then `CheckSequenceNumber()` would return `true` even when the clipboard is unavailable (frame destroyed), because `0 == 0`. This is a defense-in-depth concern.

**Suggestion**: Use `std::optional<absl::uint128>` for the sequence number, or return a distinct sentinel value (though `absl::uint128` makes this awkward). At minimum, add a `DCHECK` that the sequence number is non-zero at `ClipboardItem` construction in the lazy read path.

---

### Issue #5: Misleading error message when blob is null in `ResolveFormatData`

**Severity**: Minor
**File**: `clipboard_item.cc`
**Line**: 145-149
**Description**: When `blob == nullptr` in `ResolveFormatData()`, the promise is rejected with "Clipboard data has changed". However, a null blob can result from a read failure (e.g., empty clipboard format data, encoding failure), not necessarily a clipboard change. The error message is misleading.

**Suggestion**: Use a more accurate error message:
```cpp
if (blob == nullptr) {
    representations_with_resolvers_.at(mime_type)->Reject(
        MakeGarbageCollected<DOMException>(DOMExceptionCode::kDataError,
                                           "Failed to read clipboard data"));
    return;
}
```

---

### Issue #6: `ReadRepresentationFromClipboardReader` doesn't check for null `GetLocalFrame()`

**Severity**: Major
**File**: `clipboard_promise.cc`
**Line**: 344-358
**Description**: `ReadRepresentationFromClipboardReader()` checks `GetExecutionContext()` but then calls `GetLocalFrame()->GetSystemClipboard()` without null-checking `GetLocalFrame()`. While `GetExecutionContext()` being non-null often implies a valid frame, there could be edge cases (e.g., detached frames) where the execution context exists but the frame is gone.

**Suggestion**: Add a null check:
```cpp
void ClipboardPromise::ReadRepresentationFromClipboardReader(
    const String& format) {
  DCHECK(RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled());
  if (!GetExecutionContext()) {
    return;
  }
  LocalFrame* frame = GetLocalFrame();
  if (!frame) {
    OnRead(nullptr, format);
    return;
  }
  ClipboardReader* clipboard_reader = ClipboardReader::Create(
      frame->GetSystemClipboard(), format, this,
      /*sanitize_html=*/!will_read_unprocessed_html_);
  ...
}
```

---

### Issue #7: Missing blank line before `// static` comment

**Severity**: Suggestion
**File**: `clipboard_item.cc`
**Line**: 218-219
**Description**: Chromium style requires a blank line before function definitions. Line 218 (`}`) is immediately followed by `// static` on line 219 with no blank line.

**Suggestion**: Add a blank line:
```cpp
  return sequence_number_token == sequence_number_;
}

// static
bool ClipboardItem::supports(const String& type) {
```

---

### Issue #8: Extra blank line in `types()` method

**Severity**: Suggestion
**File**: `clipboard_item.cc`
**Line**: 131
**Description**: There's a trailing blank line inside the `if` block of the `types()` method, which is unnecessary per Chromium style.

**Suggestion**: Remove the extra blank line after `types.push_back(item);`.

---

### Issue #9: `representations_with_resolvers_` promises not rejected on context destruction

**Severity**: Minor
**File**: `clipboard_item.cc` / `clipboard_promise.cc`
**Description**: When `ContextDestroyed()` is called, pending `ScriptPromiseResolver<Blob>` objects in `ClipboardItem::representations_with_resolvers_` are not explicitly rejected. While these resolvers are in a detached context and may be harmless, best practice is to explicitly reject them to avoid leaving promises in a permanently pending state.

**Suggestion**: Either have `ClipboardPromise::ContextDestroyed()` notify the `ClipboardItem` to reject pending resolvers, or have `ClipboardItem` register as a lifecycle observer itself. Alternatively, add cleanup logic to `ClipboardItem::ResolveFormatData` to check context validity before resolving.

---

### Issue #10: Telemetry incomplete for lazy read path

**Severity**: Minor
**File**: `clipboard_item.cc`
**Line**: 190-192
**Description**: In the lazy read path, `CaptureTelemetry()` is only called for the first `getType()` call per type (because subsequent calls return cached promises from `representations_with_resolvers_`). This differs from the eager path behavior. Additionally, the lazy read path does not record `Blink.Clipboard.Read.NumberOfFormats` based on actual formats read — it records based on available formats at the time of `clipboard.read()`.

**Suggestion**: Document this intentional difference, or add lazy-read-specific telemetry.

---

### Issue #11: Test coverage gaps

**Severity**: Minor
**File**: `clipboard_unittest.cc`
**Description**: The tests cover:
- ✅ Lazy loading (MIME types only, no data read during `clipboard.read()`)
- ✅ `getType()` triggering actual read

Missing tests:
- ❌ Concurrent `getType()` calls for different MIME types
- ❌ `getType()` for unsupported types with lazy read enabled
- ❌ Context destruction with pending `getType()` calls
- ❌ Web custom format types with lazy read
- ❌ `getType()` called after clipboard changes (the web test covers this but not the unit test)
- ❌ Calling `getType()` on the same type twice returns the same promise

**Suggestion**: Add unit tests for the above edge cases. The concurrent `getType()` test is particularly important given Issue #1.

---

### Issue #12: `ClipboardItemGetType` test helper returns `IDLString` but is chained with `ScriptPromise<IDLSequence<ClipboardItem>>`

**Severity**: Suggestion
**File**: `clipboard_unittest.cc`
**Line**: 53-78
**Description**: `ClipboardItemGetType` inherits from `ThenCallable<IDLSequence<ClipboardItem>, ClipboardItemGetType, IDLString>`. Its `React()` method returns a `String`, discarding the actual blob promise. This means the test verifies that `getType()` doesn't throw, but doesn't verify the blob content. The `ClipboardItemGetTypeTest` test asserts `result == "SUCCESS"` but never checks the actual text content matches "TestStringForGetType".

**Suggestion**: Extend the test to also verify the blob content by chaining on the blob promise and reading its text.

---

### Issue #13: Review comment from Prashant about feature flag guard (not addressed)

**Severity**: Minor
**File**: `clipboard_promise.cc`
**Line**: 346, 522
**Description**: Reviewer Prashant asked "is it guaranteed that this will not get called when feature is not enabled?" for `ReadRepresentationFromClipboardReader` and `OnRead(blob, mime_type)`. Both have `DCHECK(RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled())` which is debug-only. In release builds, these methods could be called without the feature enabled if there's a code path bug.

**Suggestion**: Consider using `CHECK` instead of `DCHECK` for security-critical feature gates, or add early returns with `if (!RuntimeEnabledFeatures::ClipboardReadOnDemandEnabled()) return;`.

---

## Positive Observations

1. **Good architectural approach**: Lazy reading is a solid performance optimization — deferring clipboard data reads until `getType()` is called avoids unnecessary IPC and memory allocation for formats the web page never consumes.

2. **Feature flag gating**: All new behavior is gated behind `ClipboardReadOnDemand` runtime feature flag with `"test"` status, enabling safe incremental rollout.

3. **Sequence number validation**: The double-check pattern (at `getType()` time and at resolve time) provides good protection against stale clipboard data.

4. **Proper Oilpan tracing**: New members (`representations_with_resolvers_`, `clipboard_promise_`, `mime_types_`) are correctly added to `Trace()` methods.

5. **Callback-per-MIME-type design**: Using `read_callbacks_` keyed by MIME type correctly handles the case where different format reads complete in different orders.

6. **Test infrastructure**: Adding call tracking (`WasReadTextCalled()`, `WasReadHtmlCalled()`) to `MockClipboardHost` is a clean approach to verify lazy loading behavior.

7. **Web platform test**: The `async-clipboard-lazy-read.html` test covers the important clipboard-change-detection scenario.

---

## Overall Assessment

**Needs changes before approval**

The core design is sound, but there are several issues that need addressing before this CL is ready:

1. **Critical**: The concurrent `getType()` race condition with `clipboard_reader_` overwrite (Issue #1) and missing cleanup of `read_callbacks_` in `ContextDestroyed()` (Issue #2) are potential UAF/memory leak vectors that must be fixed.

2. **Major**: Missing custom format type support in the lazy read constructor (Issue #3) could break web custom format clipboard operations. The null-check gap in `ReadRepresentationFromClipboardReader` (Issue #6) is a crash risk.

3. **Moderate**: The sequence number 0-collision (Issue #4), misleading error messages (Issue #5), and incomplete test coverage (Issue #11) should be addressed.

The overall architecture of lazy reading is a good improvement, and the feature flag gating provides a safety net. Once the critical and major issues are addressed, this CL would be in good shape.
