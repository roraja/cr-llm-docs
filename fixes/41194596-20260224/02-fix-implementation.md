# Fix Implementation: 41194596

## Summary
Fixed Ctrl+Up/Down paragraph navigation in textareas on Windows by snapping caret positions to paragraph boundaries after `NextParagraphPosition()`/`PreviousParagraphPosition()` return an x-offset-preserved position in a different paragraph. For backward movement, implemented two-step navigation matching native Windows behavior (first to start of current paragraph, then to start of previous paragraph).

## Bug Details
- **Bug ID**: 41194596
- **Issue URL**: https://issues.chromium.org/issues/41194596
- **CL URL**: https://chromium-review.googlesource.com/c/chromium/src/+/7599041

## Root Cause
`NextParagraphPosition()` and `PreviousParagraphPosition()` in `selection_modifier.cc` preserve the horizontal x-offset when crossing paragraph boundaries, causing the caret to land at arbitrary mid-paragraph positions instead of paragraph boundaries. The functions iterate line-by-line using `NextLinePosition()`/`PreviousLinePosition()` with an `x_point` parameter that preserves the horizontal caret position. When the iteration crosses a paragraph boundary, the loop exits and returns the position on the target paragraph's visual line at whatever x-offset was being tracked, rather than at the paragraph boundary.

## Solution
In `ModifyMovingForward()` and `ModifyMovingBackward()` for `kParagraph` granularity, after calling `NextParagraphPosition()`/`PreviousParagraphPosition()`, snap to `StartOfParagraph()` when the caret has crossed into a different paragraph. For backward movement, implement two-step navigation: first move to start of current paragraph, then to start of previous paragraph (matching Windows WordPad behavior). Use `kCanSkipOverEditingBoundary` for the first backward step to correctly handle non-editable islands within contenteditable regions.

## Files Modified

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| third_party/blink/renderer/core/editing/selection_modifier.cc | Modify | +48/-7 |
| third_party/blink/renderer/core/editing/selection_modifier_test.cc | Modify | +74/-0 |
| third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html | Add | +30/-0 |
| third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph.html | Modify | +4/-4 |

## Code Changes

### third_party/blink/renderer/core/editing/selection_modifier.cc

**Forward paragraph movement (lines 507-525):**
```cpp
case TextGranularity::kParagraph: {
  const auto boundary_rule =
      RuntimeEnabledFeatures::ModifyParagraphCrossEditingoundaryEnabled()
          ? kCanCrossEditingBoundary
          : kCannotCrossEditingBoundary;
  const VisiblePositionInFlatTree& current = EndForPlatform();
  VisiblePositionInFlatTree pos = NextParagraphPosition(
      current,
      LineDirectionPointForBlockDirectionNavigation(selection_.Focus()));
  // Snap to the start of the target paragraph so the caret lands at the
  // paragraph boundary instead of at the preserved x-offset position.
  // See https://crbug.com/41194596.
  if (pos.IsNotNull() && !InSameParagraph(current, pos, boundary_rule)) {
    pos = StartOfParagraph(pos, boundary_rule);
  }
  return pos;
}
```

**Backward paragraph movement (lines 713-746):**
```cpp
case TextGranularity::kParagraph: {
  const auto boundary_rule =
      RuntimeEnabledFeatures::ModifyParagraphCrossEditingoundaryEnabled()
          ? kCanCrossEditingBoundary
          : kCannotCrossEditingBoundary;
  const VisiblePositionInFlatTree& current = StartForPlatform();
  const auto first_step_rule =
      RuntimeEnabledFeatures::
                  MoveToParagraphStartOrEndSkipsNonEditableEnabled() &&
              IsEditablePosition(current.DeepEquivalent())
          ? kCanSkipOverEditingBoundary
          : kCannotCrossEditingBoundary;
  const VisiblePositionInFlatTree& para_start =
      StartOfParagraph(current, first_step_rule);
  if (current.DeepEquivalent() != para_start.DeepEquivalent()) {
    pos = para_start;
  } else {
    pos = PreviousParagraphPosition(
        current,
        LineDirectionPointForBlockDirectionNavigation(selection_.Focus()));
    if (pos.IsNotNull() && !InSameParagraph(current, pos, boundary_rule)) {
      pos = StartOfParagraph(pos, boundary_rule);
    }
  }
  break;
}
```

## Tests Added

| Test Type | File | Test Name |
|-----------|------|-----------|
| Unit Test | third_party/blink/renderer/core/editing/selection_modifier_test.cc | `SelectionModifierTest.MoveParagraphForwardWrappingText` |
| Unit Test | third_party/blink/renderer/core/editing/selection_modifier_test.cc | `SelectionModifierTest.MoveParagraphBackwardFromMidParagraph` |
| Unit Test | third_party/blink/renderer/core/editing/selection_modifier_test.cc | `SelectionModifierTest.MoveParagraphBackwardFromParagraphStart` |
| Web Test | third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html | Wrapping text paragraph navigation |
| Web Test | third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph.html | Updated expectations |

## Verification

| Check | Result |
|-------|--------|
| Build | ✅ Pass |
| Unit Tests (18/18 SelectionModifierTest) | ✅ Pass |
| Web Tests (428/428 modify_move) | ✅ Pass |
| TDD Verification (3 tests failed before fix, pass after) | ✅ Pass |
| Code Review | ✅ Pass |
| CQ Dry Run | 🔄 Pending |

## Chrome Binary Location
```
/workspace/cr4/src/out/41194596/chrome
```

## How to Test
```bash
# Run Chrome with the fix
/workspace/cr4/src/out/41194596/chrome \
  --user-data-dir=/tmp/test-41194596 \
  /home/roraja/src/chromium-docs/bugfixer_dashboard/services/bd-build-service-go/contexts/ctx-35e24be0929e9270/repro.html
```

## Next Steps
1. Wait for CQ dry run to complete
2. Add reviewers to the CL
3. Address any review feedback
4. Land the change

## References
- CL: https://chromium-review.googlesource.com/c/chromium/src/+/7599041
- Bug: https://issues.chromium.org/issues/41194596
