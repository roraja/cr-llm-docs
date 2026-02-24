# TDD Tests: 41194596

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/core/editing/selection_modifier_test.cc](/third_party/blink/renderer/core/editing/selection_modifier_test.cc) | `SelectionModifierTest.MoveParagraphForwardWrappingText` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/selection_modifier_test.cc](/third_party/blink/renderer/core/editing/selection_modifier_test.cc) | `SelectionModifierTest.MoveParagraphBackwardFromMidParagraph` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/selection_modifier_test.cc](/third_party/blink/renderer/core/editing/selection_modifier_test.cc) | `SelectionModifierTest.MoveParagraphBackwardFromParagraphStart` | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html](/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html) | `41194596: Forward/Backward paragraph in wrapping text` | ❌ FAILING (expected) |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/core/editing/selection_modifier_test.cc](/third_party/blink/renderer/core/editing/selection_modifier_test.cc)

**Helper methods added** (lines 36-49):

```cpp
  std::string MoveForwardByParagraph(SelectionModifier& modifier) {
    modifier.Modify(SelectionModifyAlteration::kMove,
                    SelectionModifyDirection::kForward,
                    TextGranularity::kParagraph);
    return GetSelectionTextFromBody(modifier.Selection().AsSelection());
  }

  std::string MoveBackwardByParagraph(SelectionModifier& modifier) {
    modifier.Modify(SelectionModifyAlteration::kMove,
                    SelectionModifyDirection::kBackward,
                    TextGranularity::kParagraph);
    return GetSelectionTextFromBody(modifier.Selection().AsSelection());
  }
```

**Test 1: Forward paragraph movement in wrapping text** (lines 483-500):

```cpp
// https://crbug.com/41194596
TEST_F(SelectionModifierTest, MoveParagraphForwardWrappingText) {
  LoadAhem();
  InsertStyleElement("div { font: 10px/10px Ahem; width: 50px; }");
  const SelectionInDOMTree selection = SetSelectionTextToBody(
      "<div contenteditable>"
      "<div>AAA|AA AAAAA</div>"
      "<div>BBBBB BBBBB</div>"
      "</div>");
  SelectionModifier modifier(GetFrame(), selection);
  EXPECT_EQ(
      "<div contenteditable>"
      "<div>AAAAA AAAAA</div>"
      "<div>|BBBBB BBBBB</div>"
      "</div>",
      MoveForwardByParagraph(modifier));
}
```

**Test 2: Backward paragraph movement from mid-paragraph** (lines 503-520):

```cpp
// https://crbug.com/41194596
TEST_F(SelectionModifierTest, MoveParagraphBackwardFromMidParagraph) {
  LoadAhem();
  InsertStyleElement("div { font: 10px/10px Ahem; width: 50px; }");
  const SelectionInDOMTree selection = SetSelectionTextToBody(
      "<div contenteditable>"
      "<div>AAAAA AAAAA</div>"
      "<div>BBB|BB BBBBB</div>"
      "</div>");
  SelectionModifier modifier(GetFrame(), selection);
  EXPECT_EQ(
      "<div contenteditable>"
      "<div>AAAAA AAAAA</div>"
      "<div>|BBBBB BBBBB</div>"
      "</div>",
      MoveBackwardByParagraph(modifier));
}
```

**Test 3: Backward paragraph movement from paragraph start** (lines 523-540):

```cpp
// https://crbug.com/41194596
TEST_F(SelectionModifierTest, MoveParagraphBackwardFromParagraphStart) {
  LoadAhem();
  InsertStyleElement("div { font: 10px/10px Ahem; width: 50px; }");
  const SelectionInDOMTree selection = SetSelectionTextToBody(
      "<div contenteditable>"
      "<div>AAAAA AAAAA</div>"
      "<div>|BBBBB BBBBB</div>"
      "</div>");
  SelectionModifier modifier(GetFrame(), selection);
  EXPECT_EQ(
      "<div contenteditable>"
      "<div>|AAAAA AAAAA</div>"
      "<div>BBBBB BBBBB</div>"
      "</div>",
      MoveBackwardByParagraph(modifier));
}
```

### Web Tests

#### File: [/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html](/third_party/blink/web_tests/editing/selection/modify_move/move-by-paragraph-wrapping-text.html)

Three test cases using `assert_selection` with a narrow (50px) contenteditable div and Ahem font to force text wrapping:
1. Forward paragraph movement should land at start of next paragraph
2. Backward paragraph movement from mid-paragraph should go to start of current paragraph
3. Backward paragraph movement from start of paragraph should go to start of previous paragraph

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 blink_unittests
[1/6] CXX obj/third_party/blink/renderer/core/unit_tests/selection_modifier_test.o
[2/3] LINK ./blink_unittests
51.17s Build Succeeded: 3 steps
```

### Test Run Output (ALL TESTS FAIL — confirms bug detection)
```
$ out/release_x64/blink_unittests --gtest_filter="*MoveParagraph*Wrapping*:*MoveParagraph*MidParagraph*:*MoveParagraph*ParagraphStart*"

[==========] Running 3 tests from 1 test suite.
[----------] 3 tests from SelectionModifierTest

[ RUN      ] SelectionModifierTest.MoveParagraphForwardWrappingText
../../third_party/blink/renderer/core/editing/selection_modifier_test.cc:497: Failure
Expected equality of these values:
  "<div contenteditable><div>AAAAA AAAAA</div><div>|BBBBB BBBBB</div></div>"
  MoveForwardByParagraph(modifier)
    Which is: "<div contenteditable><div>AAAAA AAAAA</div><div>BBB|BB BBBBB</div></div>"
[  FAILED  ] SelectionModifierTest.MoveParagraphForwardWrappingText (88 ms)

[ RUN      ] SelectionModifierTest.MoveParagraphBackwardFromMidParagraph
../../third_party/blink/renderer/core/editing/selection_modifier_test.cc:517: Failure
Expected equality of these values:
  "<div contenteditable><div>AAAAA AAAAA</div><div>|BBBBB BBBBB</div></div>"
  MoveBackwardByParagraph(modifier)
    Which is: "<div contenteditable><div>AAAAA AAA|AA</div><div>BBBBB BBBBB</div></div>"
[  FAILED  ] SelectionModifierTest.MoveParagraphBackwardFromMidParagraph (80 ms)

[ RUN      ] SelectionModifierTest.MoveParagraphBackwardFromParagraphStart
../../third_party/blink/renderer/core/editing/selection_modifier_test.cc:537: Failure
Expected equality of these values:
  "<div contenteditable><div>|AAAAA AAAAA</div><div>BBBBB BBBBB</div></div>"
  MoveBackwardByParagraph(modifier)
    Which is: "<div contenteditable><div>AAAAA |AAAAA</div><div>BBBBB BBBBB</div></div>"
[  FAILED  ] SelectionModifierTest.MoveParagraphBackwardFromParagraphStart (79 ms)

[==========] 3 tests from 1 test suite ran. (266 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 3 tests, listed below:
[  FAILED  ] SelectionModifierTest.MoveParagraphForwardWrappingText
[  FAILED  ] SelectionModifierTest.MoveParagraphBackwardFromMidParagraph
[  FAILED  ] SelectionModifierTest.MoveParagraphBackwardFromParagraphStart

 3 FAILED TESTS
```

### Bug Behavior Demonstrated by Failures

| Test | Expected (Correct) | Actual (Buggy) | Root Cause |
|------|-------------------|----------------|------------|
| Forward from `AAA\|AA` | `\|BBBBB` (start of next para) | `BBB\|BB` (x=30px offset preserved) | `NextParagraphPosition` preserves x-offset across paragraph boundary |
| Backward from `BBB\|BB` | `\|BBBBB` (start of current para) | `AAA\|AA` in prev para (x=30px) | No two-step logic; jumps to previous paragraph at x-offset |
| Backward from `\|BBBBB` | `\|AAAAA` (start of prev para) | `AAAAA \|AAAAA` (x=0 on prev para's 2nd visual line) | `PreviousParagraphPosition` lands on wrong visual line boundary |

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior (x-offset preservation)
- [x] Tests cover the main bug scenario (forward paragraph movement in wrapping text)
- [x] Tests cover relevant edge cases (backward from mid-paragraph, backward from paragraph start)
- [x] Test code follows Chromium style guidelines (Blink C++ conventions, Ahem font usage)

## Next Steps
Once the fix is implemented (Stage 6), these tests should PASS:
- `ModifyMovingForward()` kParagraph case: snap to `StartOfParagraph()` after crossing boundary
- `ModifyMovingBackward()` kParagraph case: two-step logic (start of current, then start of previous)
