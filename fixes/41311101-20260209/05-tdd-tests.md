# TDD Tests: 41311101

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_RangeCountWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_DirectionForwardWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_DirectionNoneWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc) | `DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag` | ✅ PASSING |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/core/editing/dom_selection_test.cc](/third_party/blink/renderer/core/editing/dom_selection_test.cc)

**13 new tests added** (lines 63-322):

All tests use `ScopedRemoveVisibleSelectionInDOMSelectionForTest` to enable the `RemoveVisibleSelectionInDOMSelection` feature flag.

#### Failing Tests (TDD - prove the bug exists)

```cpp
// Key TDD test: selection on content inside a completely hidden body.
// VisibleSelection canonicalization returns IsNone() for display:none content
// (no layout box), causing rangeCount() to return 0 on the current code.
// With the fix (direct SelectionInDOMTree read), rangeCount() should return 1
// because the DOM selection is valid even without a layout box.
TEST_F(DOMSelectionTest, Bug41311101_RangeCountDisplayNoneWithFlag) {
  ScopedRemoveVisibleSelectionInDOMSelectionForTest scoped_feature(true);

  SetBodyContent("<span id='text'>Hidden text</span>");
  GetDocument().body()->setAttribute(html_names::kStyleAttr,
                                     AtomicString("display:none"));
  GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kTest);

  Element* span = GetDocument().getElementById(AtomicString("text"));
  Text* text_node = To<Text>(span->firstChild());

  Selection().SetSelection(
      SelectionInDOMTree::Builder()
          .Collapse(Position(text_node, 0))
          .Extend(Position(text_node, 6))
          .Build(),
      SetSelectionOptions());

  EXPECT_EQ(1u, GetDOMSelection()->rangeCount());
}

// Key TDD test: direction with all-hidden content.
TEST_F(DOMSelectionTest, Bug41311101_DirectionDisplayNoneWithFlag) {
  ScopedRemoveVisibleSelectionInDOMSelectionForTest scoped_feature(true);

  SetBodyContent("<span id='text'>Hidden text</span>");
  GetDocument().body()->setAttribute(html_names::kStyleAttr,
                                     AtomicString("display:none"));
  GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kTest);

  Element* span = GetDocument().getElementById(AtomicString("text"));
  Text* text_node = To<Text>(span->firstChild());

  Selection().SetSelection(
      SelectionInDOMTree::Builder()
          .Collapse(Position(text_node, 0))
          .Extend(Position(text_node, 6))
          .Build(),
      SetSelectionOptions::Builder().SetIsDirectional(true).Build());

  EXPECT_EQ("forward", GetDOMSelection()->direction());
}
```

**Why these fail**: The current `dom_selection.cc` does NOT check the `RemoveVisibleSelectionInDOMSelection` flag. It always calls `ComputeVisibleSelectionInDOMTree()` which invokes `CanonicalizeSelection()` → `CreateVisiblePosition()` → `CanonicalPositionOf()`. For `display:none` content, there are no layout objects, so `CanonicalPositionOf()` returns null, causing `CanonicalizeSelection()` to return an empty selection (`IsNone() == true`). This makes `rangeCount()` return 0 and `direction()` return "none" — even though the DOM selection (`SelectionInDOMTree`) is valid and non-empty.

After the fix, when the flag is enabled, `rangeCount()` and `direction()` will check `SelectionInDOMTree.IsNone()` directly (skipping `VisibleSelection` canonicalization), correctly returning 1 and "forward" respectively.

#### Passing Tests (regression tests for fixed behavior)

```cpp
TEST_F(DOMSelectionTest, Bug41311101_RangeCountWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_DirectionForwardWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_DirectionNoneWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_ToStringAcrossInlineBoundaryWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_ContainsNodeFullWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_ContainsNodePartialWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_DeleteFromDocumentWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_RangeCountInShadowDOMWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_RangeCountEmptyWithFlag)
TEST_F(DOMSelectionTest, Bug41311101_ToStringCollapsedWithFlag)
```

These tests cover all 7 affected methods (`rangeCount()`, `direction()`, `toString()`, `containsNode()`, `deleteFromDocument()`, `getRangeAt()`, and `CreateRangeFromSelectionEditor()` via `getRangeAt()`) with the flag enabled, including edge cases (empty selection, collapsed selection, shadow DOM, inline boundaries, partial containment). They currently pass because `VisibleSelection` canonicalization produces the same result as direct `SelectionInDOMTree` for simple visible DOM structures. They serve as regression tests to ensure the fix doesn't break correct behavior.

## Pre-Fix Test Execution

### Build Output
```
$ siso ninja -C out/release_x64 blink_unittests --offline
build finished
32.34s Build Succeeded: 2 steps - 0.06/s
```

### Test Run Output (SHOWS FAILURE)
```
$ out/release_x64/blink_unittests --gtest_filter="*DOMSelection*" --single-process-tests

[==========] Running 15 tests from 1 test suite.
[----------] 15 tests from DOMSelectionTest
[ RUN      ] DOMSelectionTest.ToStringSkipsUserSelectNone
[       OK ] DOMSelectionTest.ToStringSkipsUserSelectNone (41 ms)
[ RUN      ] DOMSelectionTest.ToStringSkipsNestedUserSelectNone
[       OK ] DOMSelectionTest.ToStringSkipsNestedUserSelectNone (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionForwardWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DirectionForwardWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionNoneWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DirectionNoneWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag
[       OK ] DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag (30 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag
../../third_party/blink/renderer/core/editing/dom_selection_test.cc:291: Failure
Expected equality of these values:
  1u
    Which is: 1
  GetDOMSelection()->rangeCount()
    Which is: 0
[  FAILED  ] DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag (68 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag
../../third_party/blink/renderer/core/editing/dom_selection_test.cc:321: Failure
Expected equality of these values:
  "forward"
  GetDOMSelection()->direction()
    Which is: "none"
[  FAILED  ] DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag (68 ms)
[----------] 15 tests from DOMSelectionTest (560 ms total)

[==========] 15 tests from 1 test suite ran. (572 ms total)
[  PASSED  ] 13 tests.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag
[  FAILED  ] DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag

 2 FAILED TESTS
```

### Failure Analysis

The 2 failing tests demonstrate the core bug:

1. **`Bug41311101_RangeCountDisplayNoneWithFlag`**: `rangeCount()` returns **0** (expected **1**)
   - The DOM selection is set to a valid range (text positions 0-6) inside `display:none` content
   - `SelectionInDOMTree.IsNone()` returns `false` (the selection exists at the DOM level)
   - `ComputeVisibleSelectionInDOMTree().IsNone()` returns `true` (canonicalization via layout tree fails for display:none)
   - Current code unconditionally uses `ComputeVisibleSelectionInDOMTree()`, so returns 0

2. **`Bug41311101_DirectionDisplayNoneWithFlag`**: `direction()` returns **"none"** (expected **"forward"**)
   - Same root cause: `ComputeVisibleSelectionInDOMTree().IsNone()` returns `true` for display:none content
   - Current code enters the `"none"` branch before checking anchor order

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code (2 out of 13 new tests)
- [x] Test failures accurately demonstrate the bug behavior (VisibleSelection canonicalization produces wrong IsNone() for display:none)
- [x] Tests cover the main bug scenario (rangeCount, direction with flag-gated path)
- [x] Tests cover relevant edge cases (empty selection, collapsed/caret, shadow DOM, inline boundaries, partial containment, deleteFromDocument)
- [x] Test code follows Chromium style guidelines (EditingTestBase, ScopedFeatureForTest, Chromium naming)

## Test Build Target

```bash
# Build
siso ninja -C out/release_x64 blink_unittests --offline

# Run all bug tests
out/release_x64/blink_unittests --gtest_filter="*DOMSelection*41311101*" --single-process-tests

# Run only the failing TDD tests
out/release_x64/blink_unittests --gtest_filter="*Bug41311101_RangeCountDisplayNoneWithFlag*:*Bug41311101_DirectionDisplayNoneWithFlag*" --single-process-tests
```

## Next Steps
Once the fix is implemented (Stage 6) — gating `rangeCount()` and `direction()` behind `RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()` to use `GetSelectionInDOMTree().IsNone()` instead of `ComputeVisibleSelectionInDOMTree().IsNone()` — all 13 tests should PASS.
