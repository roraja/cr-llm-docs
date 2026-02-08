# TDD Tests: 40105911 — WON'T FIX

## Decision: No Tests Written — Issue Confirmed But WON'T FIX

### Verification of Special Instructions

> **Special Instruction**: "Double check if the issue still exists today. If you confirm that the issue is not there, don't proceed with any fix. Simply call it won't fix."

The issue **does still exist** in the current codebase. However, per the Stage 1 assessment (`01-fix-assessment.md`), this bug is classified as **WON'T FIX**. Since the recommendation is not to fix this bug, writing TDD tests (which must fail before a fix and pass after) serves no purpose — there will be no fix to validate.

---

## Complete Analysis

### Issue Status: Still Present

The eager format reading pattern remains in the current code at `/workspace/cr3/src/third_party/blink/renderer/modules/clipboard/clipboard_promise.cc`:

1. **`ReadNextRepresentation()` (line 432)**: Still iterates through ALL available clipboard formats sequentially, creating a `ClipboardReader` for each, which reads from the system clipboard AND encodes the data into a Blob.

2. **`ResolveRead()` (line 371)**: Still wraps each result with `ToResolvedPromise<V8UnionBlobOrString>()` (line 390), creating already-resolved promises before constructing the `ClipboardItem`.

3. **`OnRead()` (line 453)**: After each format is read+encoded, increments the index and recursively calls `ReadNextRepresentation()` for the next format — confirming the eager sequential processing.

4. **`SelectiveClipboardFormatRead` feature (runtime_enabled_features.json5 line 4860)**: Still has `status: "test"` — not shipped to stable, so web developers cannot use `clipboard.read({types: [...]})` to filter formats in production.

### Why WON'T FIX

This aligns with the Stage 1 assessment for the following reasons:

1. **P3/S4 priority** — lowest practical priority. Open since November 2019 (6+ years) with no fix attempted by any Chromium contributor.

2. **Not a correctness bug** — The clipboard API behaves correctly per the W3C spec. This is purely a performance optimization request about *when* internal encoding happens.

3. **Partial mitigations exist**:
   - `SelectiveClipboardFormatRead` feature allows filtering types (status: "test")
   - Text, HTML, and SVG encoding already happens on background threads
   - `ClipboardItemGetTypeCounter` telemetry (status: "stable") tracks real-world usage patterns

4. **Architectural constraints** — The system clipboard must be read atomically during `read()` to avoid TOCTOU issues. Any lazy approach still needs to snapshot raw data upfront; savings are limited to deferred encoding.

5. **High risk, low reward** — A proper fix requires significant architectural changes to security-sensitive clipboard code. The performance gain from deferring encoding (which already uses background threads) is modest compared to the risk of regressions.

6. **Web developer workarounds exist** — `clipboard.readText()` for plain text, and `clipboard.read({types: [...]})` once `SelectiveClipboardFormatRead` ships.

### Code Evidence

```cpp
// clipboard_promise.cc:432 — Eager sequential reading (BUG PATTERN)
void ClipboardPromise::ReadNextRepresentation() {
  if (clipboard_representation_index_ == clipboard_item_data_.size()) {
    ResolveRead();  // All formats already read+encoded
    return;
  }
  ClipboardReader* clipboard_reader = ClipboardReader::Create(
      GetLocalFrame()->GetSystemClipboard(),
      clipboard_item_data_[clipboard_representation_index_].first, this,
      /*sanitize_html=*/!will_read_unprocessed_html_);
  clipboard_reader->Read();  // Reads AND encodes immediately
}

// clipboard_promise.cc:371 — Wraps in already-resolved promises
void ClipboardPromise::ResolveRead() {
  for (const auto& item : clipboard_item_data_) {
    auto promise =
        ToResolvedPromise<V8UnionBlobOrString>(script_state, item.second);
    items.emplace_back(item.first, promise);
  }
  // ClipboardItem gets pre-resolved promises — getType() has nothing to defer
}
```

### SelectiveClipboardFormatRead Feature Status

```json5
// runtime_enabled_features.json5:4860
{
  name: "SelectiveClipboardFormatRead",
  status: "test",  // NOT shipped to stable
}
```

## TDD Verification Checklist

- [ ] ~~Tests were created BEFORE implementing the fix~~ — N/A (WON'T FIX)
- [ ] ~~Tests correctly FAIL on the current (unfixed) code~~ — N/A (WON'T FIX)
- [x] Issue verified as still present in current codebase
- [x] WON'T FIX decision documented with complete rationale
- [x] Code evidence provided confirming the eager reading pattern

## Next Steps

No fix will be implemented. This stage is complete with a WON'T FIX determination.
