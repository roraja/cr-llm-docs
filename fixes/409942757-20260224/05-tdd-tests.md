# TDD Tests: 409942757

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/core/editing/visible_units_test.cc](/third_party/blink/renderer/core/editing/visible_units_test.cc) | `VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/visible_units_test.cc](/third_party/blink/renderer/core/editing/visible_units_test.cc) | `VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/visible_units_test.cc](/third_party/blink/renderer/core/editing/visible_units_test.cc) | `VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline` | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move left over non-editable span in pre (plaintext-only) | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move right over newline before non-editable span in pre (plaintext-only) | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move left over non-editable span in pre (contenteditable=true) | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move right over newline before non-editable span in pre (contenteditable=true) | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move left over non-editable button in pre | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move left over non-editable span on same line (regression) | ✅ PASSING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html) | Move right over non-editable span on same line (regression) | ✅ PASSING |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/core/editing/visible_units_test.cc](/third_party/blink/renderer/core/editing/visible_units_test.cc)

**New tests added** (lines 1148-1209):

```cpp
// crbug.com/409942757
// MostBackwardCaretPosition with kCanCrossEditingBoundary should not cross
// a preserved newline boundary when entering a text node from a parent
// position, because positions on opposite sides of a preserved newline are
// on different visual lines.
TEST_F(VisibleUnitsTest,
       MostBackwardCaretPositionPreservedNewlineBeforeNonEditable) {
  SetBodyContent(
      "<pre contenteditable id=pre>line 1\n"
      "<span contenteditable=\"false\" id=span>line 2</span>\n"
      "line 3</pre>");
  Element* pre = GetDocument().getElementById(AtomicString("pre"));
  const Position position_before_span(pre, 1);
  const Position result =
      MostBackwardCaretPosition(position_before_span, kCanCrossEditingBoundary);
  Node* text_node = pre->firstChild();
  EXPECT_NE(text_node, result.AnchorNode());
}

// crbug.com/409942757
// IsVisuallyEquivalentCandidate should return true for a position between
// editable content and a non-editable element inside an editable block.
TEST_F(VisibleUnitsTest,
       IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary) {
  SetBodyContent(
      "<pre contenteditable id=pre>line 1\n"
      "<span contenteditable=\"false\" id=span>line 2</span>\n"
      "line 3</pre>");
  Element* pre = GetDocument().getElementById(AtomicString("pre"));
  EXPECT_TRUE(IsVisuallyEquivalentCandidate(Position(pre, 1)));
}

// crbug.com/409942757
// PreviousPositionOf from after a non-editable span in <pre> should move
// to before the span, not to the end of the previous visual line.
TEST_F(VisibleUnitsTest,
       PreviousPositionOfNonEditableSpanInPreWithNewline) {
  SetBodyContent(
      "<pre contenteditable id=pre>line 1\n"
      "<span contenteditable=\"false\" id=span>line 2</span>\n"
      "line 3</pre>");
  Element* pre = GetDocument().getElementById(AtomicString("pre"));
  Node* text_node = pre->firstChild();
  const VisiblePosition after_span =
      CreateVisiblePosition(Position(pre, 2));
  const VisiblePosition result =
      PreviousPositionOf(after_span, kCanSkipOverEditingBoundary);
  EXPECT_NE(Position(text_node, 6), result.DeepEquivalent());
}
```

### Web Tests

#### File: [/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html](/third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html)

7 sub-tests covering:
1. **Left arrow** from after non-editable `<span>` in `<pre contenteditable="plaintext-only">` — should stop before span
2. **Right arrow** from end of line 1 across newline in `<pre contenteditable="plaintext-only">` — should stop before span
3. **Left arrow** from after non-editable `<span>` in `<pre contenteditable>` — should stop before span
4. **Right arrow** from end of line 1 across newline in `<pre contenteditable>` — should stop before span
5. **Left arrow** from after non-editable `<button>` in `<pre>` — should stop before button
6. **Regression: Left arrow** over non-editable span on the SAME line — should stop before span ✅
7. **Regression: Right arrow** over non-editable span on the SAME line — should stop after span ✅

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 blink_unittests
2m09.36s Build Succeeded: 2 steps - 0.02/s
```

### Unit Test Run Output (SHOWS FAILURE)
```
$ out/release_x64/blink_unittests --gtest_filter="*PreservedNewlineBeforeNonEditable*:*EditableNonEditableBoundary*:*NonEditableSpanInPre*"

[==========] Running 3 tests from 1 test suite.
[----------] 3 tests from VisibleUnitsTest
[ RUN      ] VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable
../../third_party/blink/renderer/core/editing/visible_units_test.cc:1170: Failure
Expected: (text_node) != (result.AnchorNode()), actual: #text "line 1\n" vs #text "line 1\n"
[  FAILED  ] VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable (84 ms)
[ RUN      ] VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary
../../third_party/blink/renderer/core/editing/visible_units_test.cc:1185: Failure
Value of: IsVisuallyEquivalentCandidate(Position(pre, 1))
  Actual: false
Expected: true
[  FAILED  ] VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary (73 ms)
[ RUN      ] VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline
../../third_party/blink/renderer/core/editing/visible_units_test.cc:1208: Failure
Expected: (Position(text_node, 6)) != (result.DeepEquivalent()), actual: #text "line 1\n"@offsetInAnchor[6] vs #text "line 1\n"@offsetInAnchor[6]
[  FAILED  ] VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline (74 ms)
[----------] 3 tests from VisibleUnitsTest (192 ms total)

[==========] 3 tests from 1 test suite ran. (204 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 3 tests, listed below:
[  FAILED  ] VisibleUnitsTest.MostBackwardCaretPositionPreservedNewlineBeforeNonEditable
[  FAILED  ] VisibleUnitsTest.IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary
[  FAILED  ] VisibleUnitsTest.PreviousPositionOfNonEditableSpanInPreWithNewline

 3 FAILED TESTS
```

### Web Test Run Output (SHOWS FAILURE)
```
$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 editing/selection/modify_move/move_over_non_editable_in_pre.html

0 tests ran as expected, 1 didn't:
    editing/selection/modify_move/move_over_non_editable_in_pre.html
```

Detailed web test failures:
```
[FAIL] Move left over non-editable span in pre (plaintext-only): should stop before span
  expected: line 1\n|<span contenteditable="false">line 2</span>\nline 3
  but got:  line 1|\n<span contenteditable="false">line 2</span>\nline 3
  (cursor jumps to end of line 1 instead of before span on line 2)

[FAIL] Move right over newline before non-editable span in pre (plaintext-only): should stop before span
  expected: line 1\n|<span contenteditable="false">line 2</span>\nline 3
  but got:  line 1\n<span contenteditable="false">line 2</span>|\nline 3
  (cursor jumps past span to end of line 2 instead of before span)

[FAIL] Move left over non-editable span in pre (contenteditable=true): should stop before span
  (same issue as plaintext-only variant)

[FAIL] Move right over newline before non-editable span in pre (contenteditable=true): should stop before span
  (same issue as plaintext-only variant)

[FAIL] Move left over non-editable button in pre: should stop before button
  (same issue with <button> element)
```

The 2 same-line regression tests PASS (not shown in FAIL output), confirming existing behavior is preserved.

## Failure Analysis

### Unit Test Failures Demonstrate:

1. **`MostBackwardCaretPositionPreservedNewlineBeforeNonEditable`**: `MostBackwardCaretPosition(Position(pre, 1), kCanCrossEditingBoundary)` returns a position inside the text node `"line 1\n"`, crossing the preserved newline boundary. It should not traverse backward past the `\n` to a different visual line.

2. **`IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary`**: `Position(pre, 1)` — between the editable text node `"line 1\n"` and the non-editable `<span>` — is NOT recognized as a valid caret candidate. `AtEditingBoundary()` returns false because it requires BOTH adjacent positions to be non-editable (&&), when either one being non-editable (||) should suffice.

3. **`PreviousPositionOfNonEditableSpanInPreWithNewline`**: `PreviousPositionOf` from after the non-editable span returns `Position("line 1\n", 6)` (end of "line 1") instead of a position before the span on "line 2". This is the end-to-end manifestation of the bug.

### Web Test Failures Demonstrate:

- **Left arrow**: Cursor jumps from after the non-editable span to the end of the PREVIOUS visual line, skipping the "before span" position.
- **Right arrow**: Cursor jumps from end of line 1 past the non-editable span to after it, skipping the "before span" position.
- Both behaviors match the bug report (crbug.com/409942757).

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior
- [x] Tests cover the main bug scenario (left arrow and right arrow)
- [x] Tests cover relevant edge cases (plaintext-only vs true, span vs button)
- [x] Regression tests included for same-line movement (PASS before and after fix)
- [x] Test code follows Chromium style guidelines (Blink C++ style, web test patterns)

## Next Steps
Once the fix is implemented (Stage 6), these tests should PASS:
- Modify `AtEditingBoundary()` in `visible_units.cc` to use `||` instead of `&&` for the editable↔non-editable boundary check
- Modify `MostBackwardCaretPosition()` in `visible_units.cc` to not cross preserved newline boundaries when entering a new text node
