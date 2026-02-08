# Fix Implementation: 40105911

## Summary
**Status: WON'T FIX** — Bug 40105911 (ClipboardItem prematurely generates all available formats) was verified as still present in the current codebase, but all analysis stages unanimously concluded WON'T FIX. This is a P3/S4 performance optimization request from November 2019 that has never been attempted by any Chromium contributor, has partial mitigations already in place, and would require ~290 lines of changes across 7 files in a security-sensitive area with TOCTOU implications.

## Bug Details
- **Bug ID**: 40105911
- **Issue URL**: https://issues.chromium.org/issues/40105911
- **CL URL**: N/A — No CL created (WON'T FIX)

## Root Cause
`ClipboardPromise::ReadNextRepresentation()` (clipboard_promise.cc line 432) iterates through ALL available clipboard formats, creating a `ClipboardReader` for each that reads AND encodes data eagerly. `ResolveRead()` then wraps each pre-encoded Blob in already-resolved promises via `ToResolvedPromise()` (line 390). When `ClipboardItem.getType()` is called, it simply unwraps the promise — there is no lazy loading mechanism.

## Solution
**No fix implemented.** The issue is a low-priority performance optimization (P3/S4) that:
- Has been open 6+ years with no fix attempted by any contributor
- Is not a correctness bug — API behaves correctly per W3C Clipboard API spec
- Has partial mitigations (background thread encoding, `clipboard.readText()`, `SelectiveClipboardFormatRead` test flag)
- Would require high-risk refactoring in security-sensitive clipboard code

## Files Modified

None. No code changes were made.

## Code Changes

No code changes — WON'T FIX.

## Tests Added

None — WON'T FIX.

## Verification

| Check | Result |
|-------|--------|
| Build | ⏭️ Skipped (no changes) |
| Unit Tests | ⏭️ Skipped (no changes) |
| Web Tests | ⏭️ Skipped (no changes) |
| Manual Verification | ⏭️ Skipped (no fix) |
| Code Review | ⏭️ N/A (no changes) |
| CQ Dry Run | ⏭️ Skipped (no CL) |
| Issue Still Exists | ✅ Confirmed |
| WON'T FIX Rationale | ✅ Documented |

## Chrome Binary Location
N/A — No fix binary produced.

## Issue Verification: Still Present

### Evidence 1: Eager sequential reading
**File**: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc` (line 432)
```cpp
void ClipboardPromise::ReadNextRepresentation() {
  // Iterates through ALL formats, reads AND encodes data eagerly
  ClipboardReader* clipboard_reader = ClipboardReader::Create(...);
  clipboard_reader->Read();  // Reads AND encodes eagerly
}
```

### Evidence 2: Pre-resolved promises
**File**: `third_party/blink/renderer/modules/clipboard/clipboard_promise.cc` (line 390)
```cpp
auto promise = ToResolvedPromise<V8UnionBlobOrString>(script_state, item.second);
// Creates ALREADY-RESOLVED promises — getType() has nothing to defer
```

### Evidence 3: SelectiveClipboardFormatRead not shipped
**File**: `third_party/blink/renderer/platform/runtime_enabled_features.json5`
```json5
{
  name: "SelectiveClipboardFormatRead",
  status: "test",  // NOT shipped to stable
}
```

## WON'T FIX Rationale

| Factor | Assessment |
|--------|------------|
| **Priority** | P3 / Severity S4 — lowest practical priority |
| **Age** | Filed November 1, 2019 — 6+ years, no fix attempted |
| **Status** | "New" — no Chromium contributor has attempted a fix |
| **Type** | Performance optimization, not a correctness bug |
| **Spec compliance** | API behaves correctly per W3C Clipboard API spec |
| **Fix complexity** | ~290 lines across 7 files in security-sensitive area |
| **Fix risk** | TOCTOU implications, new error handling paths, extended object lifetimes |
| **Fix reward** | Modest — encoding already runs on background threads |
| **Existing mitigations** | Background thread encoding, `clipboard.readText()`, `SelectiveClipboardFormatRead` (test-only), `ClipboardItemGetTypeCounter` telemetry |

## Next Steps
1. Bug should remain in "New" status on the issue tracker
2. If the Chromium team decides to pursue this in the future, the detailed LLD in `04-architecture-lld.md` provides a complete implementation blueprint (Option 4: Hybrid approach)
3. Shipping `SelectiveClipboardFormatRead` to stable would partially mitigate the issue for opt-in web developers

## References
- Bug report: https://issues.chromium.org/issues/40105911
- Original Monorail bug: https://crbug.com/chromium/1020703
- SelectiveClipboardFormatRead feature: https://chromestatus.com/feature/5203433409871872
- Similar format model design doc: https://docs.google.com/document/d/1XOMFG9-7NYyvqE_pv0qz3M626NZHx7P20uuUmDAyt8Q/edit
- W3C Clipboard API spec: https://w3c.github.io/clipboard-apis/
