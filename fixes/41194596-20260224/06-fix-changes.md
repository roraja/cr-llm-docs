# Fix Implementation: 41194596

## Summary
Fixed Ctrl+Up/Down paragraph navigation in textareas by snapping the result of `NextParagraphPosition()`/`PreviousParagraphPosition()` to `StartOfParagraph()` when the caret crosses a paragraph boundary. For backward movement, implemented two-step navigation matching native Windows behavior: first move to start of current paragraph, then to start of previous paragraph. Used `kCanSkipOverEditingBoundary` for the first step to correctly handle non-editable islands within editable content while staying within the editable region.

## Branch
- **Branch name**: `fix/41194596`
- **Base commit**: latest_main

## Files Modified

### 1. [/third_party/blink/renderer/core/editing/selection_modifier.cc](/third_party/blink/renderer/core/editing/selection_modifier.cc)

**Lines changed**: 507-525 (forward), 713-746 (backward)

**Before (forward, lines 507-510)**:
```cpp
    case TextGranularity::kParagraph:
      return NextParagraphPosition(
          EndForPlatform(),
          LineDirectionPointForBlockDirectionNavigation(selection_.Focus()));
```

**After (forward)**:
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

**Rationale**: After `NextParagraphPosition()` finds a position in the next paragraph, snap to `StartOfParagraph()` so the caret lands at the paragraph boundary. The `InSameParagraph()` guard ensures no change when the position couldn't move to a new paragraph (e.g., at document end).

**Before (backward, lines 700-704)**:
```cpp
    case TextGranularity::kParagraph:
      pos = PreviousParagraphPosition(
          StartForPlatform(),
          LineDirectionPointForBlockDirectionNavigation(selection_.Focus()));
      break;
```

**After (backward)**:
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

**Rationale**: Implements two-step backward paragraph navigation matching Windows behavior. Uses `kCanSkipOverEditingBoundary` for the first step (going to start of current paragraph) to correctly handle non-editable islands within contenteditable regions while staying within the editable context.

### 2. [/third_party/blink/renderer/core/editing/selection_modifier_test.cc](/third_party/blink/renderer/core/editing/selection_modifier_test.cc)

**Lines changed**: Added helper methods and 3 new tests (~74 lines)

**Changes**: Added `MoveForwardByParagraph()` and `MoveBackwardByParagraph()` helper methods, plus three unit tests:
- `MoveParagraphForwardWrappingText` — verifies forward paragraph movement lands at start of next paragraph
- `MoveParagraphBackwardFromMidParagraph` — verifies backward movement from mid-paragraph goes to start of current paragraph
- `MoveParagraphBackwardFromParagraphStart` — verifies backward movement from paragraph start goes to start of previous paragraph

### 3. [/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph.html](/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph.html)

**Lines changed**: 10, 27-28, 88

**Changes**: Updated 3 test expectations to reflect the corrected paragraph navigation behavior:
- Backward from `bar|`: now goes to `|bar` (start of current paragraph) instead of `foo|`
- Backward cross non-editable: now goes to `|Paragraph 1:...` (start of current paragraph) instead of `P1|`
- Forward cross non-editable: now goes to `|P2` (start of next paragraph) instead of `P2|`

### 4. [/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html](/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html)

**Lines changed**: New file (30 lines, created in Stage 5)

**Changes**: New web test for the bug scenario — wrapping text in contenteditable with paragraph movement.

## Git Diff Summary
```
$ git diff --stat HEAD~1
 .../editing/selection_modifier.cc               | 55 +++++++++++++++++++----
 .../editing/selection_modifier_test.cc           | 74 +++++++++++++++++++++++++
 .../modify_move/move-by-paragraph-wrapping-text.html | 30 ++++++++++
 .../modify_move/move-by-paragraph.html           |  8 ++-
 4 files changed, 156 insertions(+), 11 deletions(-)
```

## Build Result
```
$ autoninja -C out/release_x64 blink_unittests content_shell
Build Succeeded: 6 steps
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result
```
$ out/release_x64/blink_unittests --gtest_filter="SelectionModifierTest.*"
[==========] Running 8 tests from 1 test suite.
[  PASSED  ] 8 tests.
SUCCESS: all tests passed.

$ run_web_tests.py move-by-paragraph.html move-by-paragraph-wrapping-text.html
All 2 tests ran as expected.

$ run_web_tests.py editing/selection/modify_move/
All 429 tests ran as expected (428 passed, 1 didn't).
```

**Status**: ✅ TESTS PASS

## Implementation Notes
- Used `kCanSkipOverEditingBoundary` (matching `kParagraphBoundary` pattern) for the backward first-step to correctly handle non-editable islands within contenteditable without crossing out of editable regions.
- Both `ModifyParagraphCrossEditingoundaryEnabled` and `MoveToParagraphStartOrEndSkipsNonEditableEnabled` feature flags are respected, matching existing Chromium patterns.
- Only the `kMove` alteration is affected; `kExtend` (Shift+Ctrl+Up/Down selection extension) is unchanged.
- The helper functions `NextParagraphPosition()` and `PreviousParagraphPosition()` are NOT modified.

## Known Issues
- None discovered during implementation.

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
