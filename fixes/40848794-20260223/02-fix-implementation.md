# Fix Implementation: 40848794

## Summary
Fixed double-click word selection on adjacent contenteditable spans by adding clamping logic to `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate()` in `visible_units.cc`. When the backward word boundary crosses into a sibling contenteditable span, the function now returns `FirstPositionInNode()` of the anchor's text node instead of returning null, symmetric with the forward adjustment function which already correctly uses `LastPositionInNode()`.

## Bug Details
- **Bug ID**: 40848794
- **Issue URL**: https://issues.chromium.org/issues/40848794
- **CL URL**: https://chromium-review.googlesource.com/c/chromium/src/+/7599040

## Root Cause
When contenteditable `<span>` elements are placed adjacent without whitespace, `TextOffsetMapping` concatenates text across element boundaries (e.g., `"SpanNumber1SpanNumber2SpanNumber3"`). The `WordBreakIterator` finds no word break between spans, so the backward word start is computed in a sibling span's text node. `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate()` then detects the position is outside the current editing root and returns null instead of clamping to the span boundary — causing double-click selection to fail.

## Solution
Added clamping logic to `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate()` to return `FirstPositionInNode()` of the anchor's text node when the found position is outside the editing root. This directly mirrors the existing pattern in `AdjustForwardPositionToAvoidCrossingEditingBoundariesTemplate()` (lines 302-313) which correctly clamps to `LastPositionInNode()`.

## Files Modified

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| third_party/blink/renderer/core/editing/visible_units.cc | Modify | +10/-2 |
| third_party/blink/renderer/core/editing/visible_units_word_test.cc | Modify | +47/-0 |
| third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html | Add | +65/-0 |

## Code Changes

### third_party/blink/renderer/core/editing/visible_units.cc

```cpp
// Before (buggy):
  // Return empty position if |pos| is not somewhere inside the editable
  // region containing this position
  if (highest_root && !highest_root->contains(pos.AnchorNode())) {
    return PositionWithAffinityTemplate<Strategy>();
  }

// After (fixed):
  // Return first position in the anchor's text node if |pos| is not somewhere
  // inside the editable region containing this position.
  if (highest_root && !highest_root->contains(pos.AnchorNode())) {
    const Node* first_editable = anchor.ComputeContainerNode();
    if (first_editable->IsTextNode()) {
      PositionTemplate<Strategy> first_position =
          PositionTemplate<Strategy>::FirstPositionInNode(*first_editable);
      if (anchor != first_position) {
        return PositionWithAffinityTemplate<Strategy>(first_position);
      }
    }
    return PositionWithAffinityTemplate<Strategy>();
  }
```

## Tests Added

| Test Type | File | Test Name |
|-----------|------|-----------|
| Unit Test | third_party/blink/renderer/core/editing/visible_units_word_test.cc | `VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans` |
| Unit Test | third_party/blink/renderer/core/editing/visible_units_word_test.cc | `VisibleUnitsWordTest.EndOfWordAdjacentContentEditableSpans` |
| Web Test | third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html | Double-click adjacent contenteditable spans |

## Verification

| Check | Result |
|-------|--------|
| Build | ✅ Pass |
| Unit Tests (17 VisibleUnitsWordTest) | ✅ Pass |
| Web Tests (35 mouse selection tests) | ✅ Pass |
| TDD: Tests failed before fix | ✅ Confirmed |
| TDD: Tests passed after fix | ✅ Confirmed |
| Code Review | ✅ Pass |
| CQ Dry Run | 🔄 Pending |

## Chrome Binary Location
```
/workspace/cr4/src/out/40848794/chrome
```

## How to Test
```bash
# Run Chrome with the fix
/workspace/cr4/src/out/40848794/chrome \
  --user-data-dir=/tmp/test-40848794 \
  /home/roraja/src/chromium-docs/bugfixer_dashboard/services/bd-build-service-go/contexts/ctx-cf1801a1646b30d7/repro.html
```

## Next Steps
1. Wait for CQ dry run to complete
2. Add reviewers to the CL
3. Address any review feedback
4. Land the change

## References
- CL: https://chromium-review.googlesource.com/c/chromium/src/+/7599040
- Bug: https://issues.chromium.org/issues/40848794
