# Fix Implementation: 409942757

## Summary
Fixed incorrect cursor positioning when navigating with arrow keys around `contenteditable="false"` inline elements within `<pre>` elements. Two changes were made to `visible_units.cc`: (1) expanded `AtEditingBoundary()` to recognize positions between editable and non-editable sibling content as valid editing boundaries, and (2) modified `MostBackwardCaretPosition()` to avoid crossing preserved newline boundaries when entering a text node from a parent/sibling position.

## Bug Details
- **Bug ID**: 409942757
- **Issue URL**: https://issues.chromium.org/issues/409942757
- **CL URL**: https://chromium-review.googlesource.com/c/chromium/src/+/7599179

## Root Cause
The position `Position(<pre>, 1)` — between a text node ending with `\n` and a `<span contenteditable="false">` — was not recognized as a valid caret candidate. `AtEditingBoundary()` required both adjacent positions to be non-editable (`&&`), but only one side was non-editable. Additionally, `MostBackwardCaretPosition()` traversed into the preceding text node across a preserved newline, causing `CanonicalPosition` to resolve to the wrong visual line.

## Solution
1. **`AtEditingBoundary()`**: Changed the condition from requiring both adjacent positions to be non-editable (`&&`) to requiring either side to be non-editable (`||`), with guards to avoid false positives at container edges.
2. **`MostBackwardCaretPosition()`**: Added a check to avoid crossing preserved newline boundaries when entering a text node from a parent position, returning the original position instead.

## Files Modified

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| third_party/blink/renderer/core/editing/visible_units.cc | Modify | +26/-6 |
| third_party/blink/renderer/core/editing/visible_units_test.cc | Add | +61 |
| third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html | Add | +62 |

## Code Changes

### third_party/blink/renderer/core/editing/visible_units.cc

#### Change 1: `AtEditingBoundary()` (~lines 1100-1125)

```cpp
// Before:
return next_position.IsNotNull() &&
       !IsEditable(*next_position.AnchorNode()) &&
       prev_position.IsNotNull() && !IsEditable(*prev_position.AnchorNode());

// After:
return !positions.AtFirstEditingPositionForNode() &&
       !positions.AtLastEditingPositionForNode() &&
       ((next_position.IsNotNull() &&
         !IsEditable(*next_position.AnchorNode())) ||
        (prev_position.IsNotNull() &&
         !IsEditable(*prev_position.AnchorNode())));
```

#### Change 2: `MostBackwardCaretPosition()` (~lines 860-881)

```cpp
// Added before returning position inside text node:
if (auto* text_node = DynamicTo<Text>(current_node)) {
  const String& data = text_node->data();
  if (data.length() > 0 && data[data.length() - 1] == '\n' &&
      layout_object->Style()->ShouldPreserveBreaks()) {
    return position;
  }
}
```

## Tests Added

| Test Type | File | Test Name |
|-----------|------|-----------|
| Unit Test | third_party/blink/renderer/core/editing/visible_units_test.cc | `VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable` |
| Unit Test | third_party/blink/renderer/core/editing/visible_units_test.cc | `VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary` |
| Unit Test | third_party/blink/renderer/core/editing/visible_units_test.cc | `VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline` |
| Web Test | third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html | 7 sub-tests covering left/right movement in pre with non-editable spans |

## Verification

| Check | Result |
|-------|--------|
| Build | ✅ Pass |
| Unit Tests (TDD - 3 new) | ✅ Pass |
| Unit Tests (VisibleUnitsTest - 50/51) | ✅ Pass (1 pre-existing failure) |
| Unit Tests (Related editing - 65) | ✅ Pass |
| Web Tests (new - 7 sub-tests) | ✅ Pass |
| Web Tests (modify_move suite - 426/427) | ✅ Pass (1 pre-existing failure) |
| Code Review | ✅ Pass |
| CQ Dry Run | 🔄 Pending |

## Chrome Binary Location
```
/workspace/cr4/src/out/409942757/chrome
```

## How to Test
```bash
# Run Chrome with the fix
/workspace/cr4/src/out/409942757/chrome \
  --user-data-dir=/tmp/test-409942757 \
  /home/roraja/src/chromium-docs/bugfixer_dashboard/services/bd-build-service-go/contexts/ctx-a7204b9836a8ce8c/repro.html
```

## Next Steps
1. Wait for CQ dry run to complete
2. Add reviewers to the CL
3. Address any review feedback
4. Land the change

## References
- CL: https://chromium-review.googlesource.com/c/chromium/src/+/7599179
- Bug: https://issues.chromium.org/issues/409942757
- Spec: https://html.spec.whatwg.org/multipage/interaction.html#contenteditable
