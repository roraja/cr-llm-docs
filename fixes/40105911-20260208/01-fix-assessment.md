# Fix Assessment: 40105911

## Executive Summary

**Verdict: WON'T FIX** — The issue described in bug 40105911 still exists in the current codebase: `clipboard.read()` eagerly reads and encodes ALL available clipboard formats into Blobs before resolving the promise, rather than lazily encoding them when `ClipboardItem.getType()` is called. However, this is a low-priority (P3/S4) architectural optimization request from 2019 that has partial mitigations already in place, and a proper fix would require significant refactoring with non-trivial TOCTOU (time-of-check-time-of-use) implications. The bug should remain as-is.

## Bug Analysis

### Problem Statement
When a web page calls `navigator.clipboard.read()`, Chromium's implementation eagerly reads ALL available clipboard format representations from the system clipboard, encodes each one into a Blob (text→UTF-8, HTML→sanitized+encoded, PNG→raw bytes, SVG→sanitized+encoded), and stores them as already-resolved promises inside the returned `ClipboardItem`. This means CPU and memory are spent encoding formats that may never be consumed by the calling JavaScript.

### Expected Behavior
Per the bug report's proposed design:
1. `clipboard.read()` should return raw/unencoded clipboard data representations
2. Format encoding (sanitization, UTF-8 conversion, etc.) should be deferred until `ClipboardItem.getType()` is called for a specific format
3. Only the formats actually requested via `getType()` should incur encoding cost

### Actual Behavior
1. `ClipboardPromise::ReadNextRepresentation()` iterates through ALL available formats sequentially
2. For each format, a `ClipboardReader` subclass reads from the system clipboard AND encodes the data (some on background threads)
3. `ClipboardPromise::ResolveRead()` wraps each result with `ToResolvedPromise()`, creating already-resolved promises
4. `ClipboardItem` is constructed with these pre-resolved promises — `getType()` simply unwraps them

### Triggering Conditions
This happens on every call to `navigator.clipboard.read()` when the clipboard contains multiple formats (e.g., copying rich text from a browser gives text/plain, text/html, and image/png). All formats are eagerly processed regardless of which ones the web app will actually consume.

## Root Cause Analysis

### Code Investigation
I examined the complete clipboard read pipeline in Blink's clipboard module:
- `ClipboardPromise::HandleRead()` → permission check → `HandleReadWithPermission()`
- `HandleReadWithPermission()` → `ReadAvailableCustomAndStandardFormats()` → `OnReadAvailableFormatNames()`
- `OnReadAvailableFormatNames()` → builds list of ALL formats → `ReadNextRepresentation()`
- `ReadNextRepresentation()` → creates `ClipboardReader` for each format → reads AND encodes
- After all formats processed → `ResolveRead()` → wraps in `ToResolvedPromise` → creates `ClipboardItem`

### Key Files Identified
- [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L344](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L344) - `HandleReadWithPermission()`: Entry point after permission granted
- [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L402](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L402) - `OnReadAvailableFormatNames()`: Builds format list, initiates sequential reading
- [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L432](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L432) - `ReadNextRepresentation()`: Core loop that reads each format
- [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L371](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L371) - `ResolveRead()`: Creates ClipboardItem with already-resolved promises
- [/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc#L322](/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc#L322) - `ClipboardReader::Create()`: Factory for format-specific readers
- [/third_party/blink/renderer/modules/clipboard/clipboard_item.cc#L118](/third_party/blink/renderer/modules/clipboard/clipboard_item.cc#L118) - `ClipboardItem::getType()`: Simply unwraps pre-resolved promise
- [/third_party/blink/renderer/platform/runtime_enabled_features.json5#L4860](/third_party/blink/renderer/platform/runtime_enabled_features.json5#L4860) - `SelectiveClipboardFormatRead` feature (status: "test", not shipped)

### Root Cause
**Location**: [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L432](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc#L432)  
**Function**: `ClipboardPromise::ReadNextRepresentation()`  
**Issue**: The method sequentially iterates through ALL available clipboard formats, creating a `ClipboardReader` for each, which reads from the system clipboard AND encodes the data into a Blob. This loop completes before `ResolveRead()` is called at line 371, which then creates a `ClipboardItem` with already-resolved promises via `ToResolvedPromise()` at line 390. There is no mechanism to defer reading/encoding to `getType()`.

### Code Flow Diagram
```
clipboard.read() [JS]
    │
    ▼
ClipboardPromise::CreateForRead()          [clipboard_promise.cc:107]
    │
    ▼
ClipboardPromise::HandleRead()             [clipboard_promise.cc:245]
    │  (checks SelectiveClipboardFormatRead flag, validates options)
    ▼
ValidatePreconditions() → permission check
    │
    ▼
HandleReadWithPermission()                 [clipboard_promise.cc:344]
    │
    ▼
SystemClipboard::ReadAvailableCustomAndStandardFormats()
    │
    ▼
OnReadAvailableFormatNames()               [clipboard_promise.cc:402]
    │  (builds clipboard_item_data_ list with ALL supported formats)
    ▼
ReadNextRepresentation()                   [clipboard_promise.cc:432]  ◄── BUG HERE
    │  (loops through EVERY format)
    │
    ├──► ClipboardReader::Create()         [clipboard_reader.cc:322]
    │       │
    │       ▼
    │    ClipboardPngReader::Read()         → reads PNG from system clipboard
    │    ClipboardTextReader::Read()        → reads text, encodes UTF-8 on bg thread
    │    ClipboardHtmlReader::Read()        → reads HTML, sanitizes, encodes on bg thread  
    │    ClipboardSvgReader::Read()         → reads SVG, sanitizes, encodes on bg thread
    │       │
    │       ▼
    │    promise_->OnRead(blob)            → stores Blob in clipboard_item_data_
    │       │
    │       ▼
    └──► ++index → ReadNextRepresentation()  (next format)
              │
              ▼  (when all formats done)
         ResolveRead()                     [clipboard_promise.cc:371]
              │  
              ▼  
         ToResolvedPromise() for each item  ◄── All data already materialized
              │
              ▼
         ClipboardItem created with resolved promises
              │
              ▼
         getType() → simply unwraps already-resolved promise (no lazy loading)
```

## Fix Options

### Option 1: Lazy Encoding in ClipboardItem::getType()
- **Description**: Store raw clipboard data (unencoded strings/bytes) in ClipboardItem during `read()`. Defer encoding (UTF-8 conversion, HTML sanitization, etc.) until `getType()` is called.
- **Files to modify**: 
  - [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc) - Change `ReadNextRepresentation()` to read raw data without encoding
  - [/third_party/blink/renderer/modules/clipboard/clipboard_item.cc](/third_party/blink/renderer/modules/clipboard/clipboard_item.cc) - Implement lazy encoding in `getType()`
  - [/third_party/blink/renderer/modules/clipboard/clipboard_item.h](/third_party/blink/renderer/modules/clipboard/clipboard_item.h) - Add storage for raw data
  - [/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc](/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc) - Split reading from encoding
- **Complexity**: High
- **Risk**: High
- **Pros**: Fully addresses the bug; formats only encoded when needed; saves CPU/memory for unused formats
- **Cons**: Major architectural change; requires storing raw clipboard data; HTML sanitization currently happens on main thread before bg-thread encoding — splitting these is non-trivial; TOCTOU risk if raw data is held too long; breaks assumption that `getType()` returns instantly

### Option 2: Ship SelectiveClipboardFormatRead Feature
- **Description**: The `SelectiveClipboardFormatRead` feature flag (currently status: "test") already allows callers to specify which types they want via `clipboard.read({types: ['text/plain']})`. Ship this to stable so web apps can opt-in to reading only the formats they need.
- **Files to modify**: 
  - [/third_party/blink/renderer/platform/runtime_enabled_features.json5#L4860](/third_party/blink/renderer/platform/runtime_enabled_features.json5#L4860) - Change status from "test" to "stable"
- **Complexity**: Low
- **Risk**: Low
- **Pros**: Already implemented and tested; minimal code change; addresses practical concern of reading unneeded formats; web-facing API
- **Cons**: Doesn't address lazy encoding (requested formats are still eagerly encoded); requires web developers to opt-in; doesn't fix the default case

### Option 3: Parallel Format Reading Instead of Sequential
- **Description**: Change `ReadNextRepresentation()` from sequential to parallel reading of all formats. Currently, formats are read one at a time; reading them in parallel would reduce total wall-clock time.
- **Files to modify**: 
  - [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc) - Restructure to launch all readers simultaneously
- **Complexity**: Medium
- **Risk**: Medium
- **Pros**: Reduces wall-clock time for multi-format reads; doesn't change architecture significantly
- **Cons**: Doesn't save total CPU/memory (all formats still read); increases peak memory usage; more complex concurrency management; doesn't address the core bug

### Option 4: Hybrid — Read Raw, Create Unresolved Promises
- **Description**: During `read()`, read raw data from system clipboard (to snapshot atomically) but don't encode it. Store raw data and create *unresolved* promises in ClipboardItem. When `getType()` is called, encode the raw data and resolve the promise.
- **Files to modify**: 
  - [/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc](/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc) - Read raw without encoding
  - [/third_party/blink/renderer/modules/clipboard/clipboard_item.cc](/third_party/blink/renderer/modules/clipboard/clipboard_item.cc) - Store raw data, resolve on getType()
  - [/third_party/blink/renderer/modules/clipboard/clipboard_item.h](/third_party/blink/renderer/modules/clipboard/clipboard_item.h) - New raw data storage members
  - [/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc](/third_party/blink/renderer/modules/clipboard/clipboard_reader.cc) - Factor out encoding from reading
- **Complexity**: High
- **Risk**: Medium
- **Pros**: Addresses core bug exactly as spec authors intended; atomic read preserves TOCTOU safety; encoding is truly lazy
- **Cons**: Significant refactoring; raw data storage increases complexity; need to handle case where raw data is GC'd before getType(); encoding on getType() path may cause user-visible delay

## Recommended Approach

**None — WON'T FIX**

This bug should remain in its current state for the following reasons:

1. **Priority/Severity**: P3/S4 — this is the lowest practical priority. The bug has been open for 6+ years (filed Nov 2019) with no fix attempted by any Chromium contributor.

2. **Not a correctness bug**: The clipboard API behaves correctly per the W3C spec. The issue is purely a performance optimization request about *when* encoding happens internally.

3. **Partial mitigations already exist**:
   - `SelectiveClipboardFormatRead` feature (status: "test") allows filtering which types to read
   - Text, HTML, and SVG encoding already happens on background threads, mitigating main-thread impact
   - Telemetry (`ClipboardItemGetTypeCounter`, status: "stable") tracks the time between `read()` and `getType()`, allowing the team to gather data on real-world usage patterns

4. **Architectural constraints**: The bug author themselves noted: *"we need to provide the information read during the clipboard.read call, in case clipboard content changes"* — the system clipboard must be read atomically during `read()` to avoid TOCTOU issues. Any lazy approach still needs to snapshot raw data upfront, making the savings primarily about deferred encoding, not deferred reading.

5. **Risk vs. reward**: Options 1 and 4 require significant architectural changes to the clipboard pipeline, touching multiple files in a security-sensitive area. The performance gain from deferring encoding is modest (especially since encoding already uses background threads), while the risk of introducing regressions or security issues is non-trivial.

6. **Web developer workaround**: Web developers who only need specific formats can already minimize overhead by using `clipboard.readText()` for plain text, or will be able to use `clipboard.read({types: [...]})` once `SelectiveClipboardFormatRead` ships.

## Testing Strategy
- **Unit tests needed**: N/A (won't fix)
- **Web tests needed**: N/A (won't fix)
- **Manual verification**: The issue can be confirmed by examining the code flow — `ReadNextRepresentation()` still iterates all formats eagerly, and `ResolveRead()` still uses `ToResolvedPromise()` at line 390 of clipboard_promise.cc

## Related Code References
- Bug report: https://issues.chromium.org/issues/40105911
- Original Monorail bug: https://crbug.com/chromium/1020703
- SelectiveClipboardFormatRead feature: https://chromestatus.com/feature/5203433409871872
- Similar format model design doc: https://docs.google.com/document/d/1XOMFG9-7NYyvqE_pv0qz3M626NZHx7P20uuUmDAyt8Q/edit
- W3C Clipboard API spec: https://w3c.github.io/clipboard-apis/
- Related telemetry feature: `ClipboardItemGetTypeCounter` (status: "stable") in runtime_enabled_features.json5
