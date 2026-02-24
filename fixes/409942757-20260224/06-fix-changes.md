# Fix Implementation: 409942757

## Summary
Fixed incorrect cursor positioning when navigating with arrow keys around `contenteditable="false"` inline elements within `<pre>` elements. Two changes were made to `visible_units.cc`: (1) expanded `AtEditingBoundary()` to recognize positions between editable and non-editable sibling content as valid editing boundaries where the caret can be placed, and (2) modified `MostBackwardCaretPosition()` to avoid crossing preserved newline boundaries when entering a text node from a parent/sibling position.

## Branch
- **Branch name**: `fix/409942757`
- **Base commit**: `4c87ae60706b7` (latest_main)

## Files Modified

### 1. [/third_party/blink/renderer/core/editing/visible_units.cc](/third_party/blink/renderer/core/editing/visible_units.cc)

#### Change 1: `AtEditingBoundary()` (lines ~1100-1125)

**Before**:
```cpp
  return next_position.IsNotNull() &&
         !IsEditable(*next_position.AnchorNode()) &&
         prev_position.IsNotNull() && !IsEditable(*prev_position.AnchorNode());
```

**After**:
```cpp
  // A position between editable and non-editable siblings (e.g., between
  // editable text and a non-editable span) is an editing boundary where the
  // caret should be placeable. Only apply this for positions that are not at
  // the first/last editing position of their node, since those cases are
  // already handled above.
  return !positions.AtFirstEditingPositionForNode() &&
         !positions.AtLastEditingPositionForNode() &&
         ((next_position.IsNotNull() &&
           !IsEditable(*next_position.AnchorNode())) ||
          (prev_position.IsNotNull() &&
           !IsEditable(*prev_position.AnchorNode())));
```

**Rationale**: The old condition required BOTH adjacent positions to be non-editable (`&&`). For `Position(<pre>, 1)` between editable text and a non-editable span, only one side is non-editable, so the position was incorrectly rejected as a caret candidate. The fix uses `||` but guards with `!AtFirstEditingPositionForNode() && !AtLastEditingPositionForNode()` to avoid making edge-of-contenteditable positions (where outside is non-editable) into false boundaries.

#### Change 2: `MostBackwardCaretPosition()` (lines ~860-881)

**Before**:
```cpp
      return PositionTemplate<Strategy>(
          current_node,
          text_layout_object->CaretMaxOffset() + text_start_offset);
```

**After**:
```cpp
      const unsigned caret_max_offset =
          text_layout_object->CaretMaxOffset() + text_start_offset;
      // crbug.com/409942757: When entering a text node from a parent/sibling
      // position (going backward), if the text ends with a preserved newline,
      // any position we return inside this text node is on a different visual
      // line from the starting position. Return the original position to avoid
      // crossing this visual line boundary.
      if (auto* text_node = DynamicTo<Text>(current_node)) {
        const String& data = text_node->data();
        if (data.length() > 0 && data[data.length() - 1] == '\n' &&
            layout_object->Style()->ShouldPreserveBreaks()) {
          return position;
        }
      }
      return PositionTemplate<Strategy>(current_node, caret_max_offset);
```

**Rationale**: When `MostBackwardCaretPosition` traverses backward from a parent-level position into a child text node ending with a preserved `\n` (in `<pre>` mode), any position inside the text node is on a different visual line from the starting position. The fix returns the original position to prevent crossing this visual line boundary. Without this, `CanonicalPosition` would incorrectly canonicalize `Position(<pre>, 1)` to a position on the previous visual line.

## Git Diff Summary
```
$ git diff --stat
 third_party/blink/renderer/core/editing/visible_units.cc      | 32 ++++++++++++++++++++++-----
 third_party/blink/renderer/core/editing/visible_units_test.cc  | 61 +++++++++++++++++++++++++++++++++++++++++++++++++++
 2 files changed, 87 insertions(+), 6 deletions(-)
```

## Build Result
```
$ autoninja -C out/release_x64 blink_unittests
1m04.82s Build Succeeded: 6 steps - 0.09/s

$ autoninja -C out/release_x64 blink_tests
40.05s Build Succeeded: 48 steps - 1.20/s
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result

### Unit Tests
```
$ out/release_x64/blink_unittests --gtest_filter="*PreservedNewlineBeforeNonEditable*:*EditableNonEditableBoundary*:*NonEditableSpanInPre*"
[  PASSED  ] 3 tests.
  VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable (39 ms)
  VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary (31 ms)
  VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline (31 ms)
```

### All VisibleUnitsTest suite
```
$ out/release_x64/blink_unittests --gtest_filter="VisibleUnitsTest.*"
50/51 tests passed.
1 test failed (PRE-EXISTING):
  VisibleUnitsTest.previousPositionOfNoPreviousPosition (pre-existing, confirmed failing on baseline)
```

### Web Tests
```
$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 editing/selection/modify_move/move_over_non_editable_in_pre.html
The test ran as expected.

$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 editing/selection/modify_move/
427 tests ran as expected (426 passed, 1 didn't)
1 failure (PRE-EXISTING):
  editing/selection/modify_move/move_right_character_over_ineditable.html (pre-existing, confirmed failing on baseline)
```

**Status**: ✅ ALL NEW TESTS PASS (pre-existing failures confirmed on baseline)

## Implementation Notes
- The LLD's `MostBackwardCaretPosition` fix needed adjustment: `CaretMaxOffset()` returns 7 (full text length including `\n`), not 6 as assumed. The fix was adapted to check if the text node ends with a preserved `\n` character using `data[data.length() - 1]` rather than checking `data[caret_max_offset]`.
- The LLD's `AtEditingBoundary` fix (`&&` → `||`) was too broad — it caused a regression where positions at the edge of contenteditable containers (where outside is non-editable) became false editing boundaries. Added guards `!AtFirstEditingPositionForNode() && !AtLastEditingPositionForNode()` to restrict the condition to interior positions only.
- The `PositionIterator` normalizes `Position(pre, 1)` into `Position("line 1\n", 7)` at initialization, so `last_visible.DeprecatedComputePosition()` returns the text node position. Fix was changed to return the original `position` parameter instead.

## Known Issues
- Pre-existing test failure: `VisibleUnitsTest.previousPositionOfNoPreviousPosition` — unrelated to this fix.
- Pre-existing web test failure: `editing/selection/modify_move/move_right_character_over_ineditable.html` — unrelated to this fix.
- WPT caret tests could not be run due to missing `aioquic` module (infrastructure issue).

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
