# TDD Tests: 40848794

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/core/editing/visible_units_word_test.cc](/workspace/cr4/src/third_party/blink/renderer/core/editing/visible_units_word_test.cc) | `VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/visible_units_word_test.cc](/workspace/cr4/src/third_party/blink/renderer/core/editing/visible_units_word_test.cc) | `VisibleUnitsWordTest.EndOfWordAdjacentContentEditableSpans` | ✅ PASSING (forward adjustment already correct) |
| Web Test | [/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html](/workspace/cr4/src/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html) | double-click-adjacent-span-2 select second span | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html](/workspace/cr4/src/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html) | double-click-adjacent-span-1 select first span | ✅ PASSING (first span not affected by bug) |
| Web Test | [/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html](/workspace/cr4/src/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html) | double-click-adjacent-span-3 select third span | ❌ FAILING |
| Web Test | [/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html](/workspace/cr4/src/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html) | double-click-adjacent-span-no-spellcheck select second span | ❌ FAILING |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/core/editing/visible_units_word_test.cc](/workspace/cr4/src/third_party/blink/renderer/core/editing/visible_units_word_test.cc)

**Tests added** (lines 147-194):

```cpp
// https://crbug.com/40848794
// Word boundary detection should respect contenteditable boundaries on
// adjacent contenteditable spans without whitespace between them.
TEST_F(VisibleUnitsWordTest, StartOfWordAdjacentContentEditableSpans) {
  // When the cursor is inside the second adjacent contenteditable span,
  // StartOfWord should return a valid position at the beginning of that span,
  // not cross into the first span.
  SetBodyContent(
      "<div>"
      "<span contenteditable=\"true\">SpanNumber1</span>"
      "<span contenteditable=\"true\" id=\"target\">SpanNumber2</span>"
      "<span contenteditable=\"true\">SpanNumber3</span>"
      "</div>");
  const Element* target =
      GetDocument().getElementById(AtomicString("target"));
  // Position the caret inside "SpanNumber2" (at offset 5, i.e. "SpanN|umber2")
  const Position position(target->firstChild(), 5);
  const Position result = StartOfWordPosition(position);
  // The result should be a valid, connected position.
  ASSERT_FALSE(result.IsNull());
  ASSERT_TRUE(result.IsConnected());
  // The start of word should be at the beginning of the second span's text,
  // not crossing into the first span.
  EXPECT_EQ(Position(target->firstChild(), 0), result);
}

TEST_F(VisibleUnitsWordTest, EndOfWordAdjacentContentEditableSpans) {
  // When the cursor is inside the second adjacent contenteditable span,
  // EndOfWord should return a valid position at the end of that span,
  // not cross into the third span.
  SetBodyContent(
      "<div>"
      "<span contenteditable=\"true\">SpanNumber1</span>"
      "<span contenteditable=\"true\" id=\"target\">SpanNumber2</span>"
      "<span contenteditable=\"true\">SpanNumber3</span>"
      "</div>");
  const Element* target =
      GetDocument().getElementById(AtomicString("target"));
  // Position the caret inside "SpanNumber2" (at offset 5, i.e. "SpanN|umber2")
  const Position position(target->firstChild(), 5);
  const Position result = EndOfWordPosition(position);
  // The result should be a valid, connected position.
  ASSERT_FALSE(result.IsNull());
  ASSERT_TRUE(result.IsConnected());
  // The end of word should be at the end of the second span's text,
  // not crossing into the third span.
  EXPECT_EQ(Position(target->firstChild(), 11), result);
}
```

### Web Tests

#### File: [/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html](/workspace/cr4/src/third_party/blink/web_tests/editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html)

Tests double-click word selection on adjacent contenteditable spans (Scenario 2 from bug report — no whitespace between span tags). Uses `eventSender` to simulate double-click on target span and `selection_test` framework to verify the expected selection covers the entire word.

**Test cases:**
1. **double-click-adjacent-span-2**: Double-click on second span → expects `^SpanNumber2|` selected
2. **double-click-adjacent-span-1**: Double-click on first span → expects `^SpanNumber1|` selected (regression test)
3. **double-click-adjacent-span-3**: Double-click on third span → expects `^SpanNumber3|` selected
4. **double-click-adjacent-span-no-spellcheck**: Same as #2 without `spellcheck="false"` attribute

## Pre-Fix Test Execution

### Unit Test Build Output
```
$ autoninja -C out/release_x64 blink_unittests
ninja: no work to do.
8.27s Build Succeeded: 0 steps - 0.00/s
```

### Unit Test Run Output (SHOWS FAILURE)
```
$ out/release_x64/blink_unittests --gtest_filter="VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans:VisibleUnitsWordTest.EndOfWordAdjacentContentEditableSpans"

[==========] Running 2 tests from 1 test suite.
[----------] 2 tests from VisibleUnitsWordTest
[ RUN      ] VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans
../../third_party/blink/renderer/core/editing/visible_units_word_test.cc:166: Failure
Value of: result.IsNull()
  Actual: true
Expected: false
[  FAILED  ] VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans (80 ms)
[ RUN      ] VisibleUnitsWordTest.EndOfWordAdjacentContentEditableSpans
[       OK ] VisibleUnitsWordTest.EndOfWordAdjacentContentEditableSpans (30 ms)
[==========] 2 tests from 1 test suite ran. (128 ms total)
[  PASSED  ] 1 test.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] VisibleUnitsWordTest.StartOfWordAdjacentContentEditableSpans
```

**Analysis**: `StartOfWordPosition()` returns a **null position** when the backward word boundary crosses from SpanNumber2 into SpanNumber1's editing context. This is because `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate()` returns null instead of clamping to the first position in the anchor's text node.

### Web Test Run Output (SHOWS FAILURE)
```
$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html

[1/1] editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html failed unexpectedly (asserts failed)

0 tests ran as expected, 1 didn't:
    editing/selection/mouse/doubleclick-adjacent-contenteditable-spans.html
```

**Detailed failures from actual output:**

| Test Case | Expected | Actual (Buggy) |
|-----------|----------|----------------|
| span-2 select | `^SpanNumber2\|` | `SpanN^umber2\|` |
| span-3 select | `^SpanNumber3\|` | `SpanN^umber3\|` |
| no-spellcheck span-2 | `^SpanNumber2\|` | `SpanN^umber2\|` |

The buggy behavior shows only partial word selection (`SpanN^umber2|` instead of `^SpanNumber2|`), confirming the bug: double-clicking text in adjacent contenteditable spans does not select the whole word.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior
- [x] Tests cover the main bug scenario (double-click on 2nd span)
- [x] Tests cover relevant edge cases (1st span, 3rd span, no-spellcheck)
- [x] Test code follows Chromium style guidelines

## Next Steps
Once the fix is implemented (Stage 6) in `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate()` in `visible_units.cc` to clamp to `FirstPositionInNode()` instead of returning null, these tests should PASS.
