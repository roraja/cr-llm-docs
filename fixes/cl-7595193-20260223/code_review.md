# Code Review: CL 7595193

## [Editing] Fix incorrect text selection behavior with :after pseudo-element and \<br\> element

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7595193
**Author:** Tanu Jain <tanujain@microsoft.com>
**Files Changed:** 4 (+129/-2)

---

### Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ✅ | Fix is logically sound; `NonPseudoNode()` correctly filters pseudo-element-only lines |
| Style | ⚠️ | Nested conditional can be simplified; minor test assertion weakness |
| Security | ✅ | No security concerns |
| Performance | ✅ | No performance impact; check is within existing loop on inline leaves |
| Testing | ⚠️ | Good coverage of the bug scenario, but missing flag-disabled test case and edge cases |

---

### Detailed Findings

#### Issue #1: Nested conditional logic can be simplified
**Severity**: Minor
**File**: `third_party/blink/renderer/core/editing/selection_modifier_line.cc`
**Line**: 72-82
**Description**: The current nested `if/else` structure with the feature flag check is unnecessarily complex. The two branches can be collapsed into a single boolean expression.
**Current code**:
```cpp
if (current.GetLayoutObject() && current.IsInlineLeaf()) {
  if (RuntimeEnabledFeatures::
          SkipPseudoOnlyLinesInLineNavigationEnabled()) {
    if (current.GetLayoutObject()->NonPseudoNode()) {
      return true;
    }
  } else {
    return true;
  }
}
```
**Suggestion**: Simplify to:
```cpp
if (current.GetLayoutObject() && current.IsInlineLeaf()) {
  if (!RuntimeEnabledFeatures::
          SkipPseudoOnlyLinesInLineNavigationEnabled() ||
      current.GetLayoutObject()->NonPseudoNode()) {
    return true;
  }
}
```
This is functionally equivalent, removes a level of nesting, and is easier to reason about. It reads as: "return true if the feature is disabled (old behavior) OR if this is a real DOM node."

#### Issue #2: C++ test assertion is weaker than necessary
**Severity**: Minor
**File**: `third_party/blink/renderer/core/editing/selection_modifier_test.cc`
**Line**: 223-226
**Description**: The test checks `result.Anchor() < result.Focus() || result.Anchor() == result.Focus()` (effectively `<=`). The `==` case means no extension happened at all, which should also be a failure for `kExtend` + `kForward`. A strict `<` check would be a stronger assertion. Additionally, the test could verify that the focus actually lands in the "second" div's content, similar to the web test.
**Suggestion**:
```cpp
EXPECT_TRUE(result.Anchor() < result.Focus())
    << "Selection should extend forward, not stay in place or go backward. "
    << "Anchor: " << result.Anchor() << ", Focus: " << result.Focus();
```

#### Issue #3: No test for feature-flag-disabled behavior
**Severity**: Minor
**File**: `third_party/blink/renderer/core/editing/selection_modifier_test.cc`
**Line**: 201
**Description**: The C++ test only tests with the flag enabled (its default `"stable"` state). Since a `RuntimeEnabledFeature` flag is being introduced as a kill switch, it would be good practice to have a test demonstrating the old (broken) behavior when the flag is disabled. This validates the flag actually controls the code path and documents the regression it guards against. Use `ScopedSkipPseudoOnlyLinesInLineNavigationForTest` (auto-generated from the feature name) to disable it.
**Suggestion**: Add a companion test:
```cpp
TEST_F(SelectionModifierTest,
       ExtendByLineWithInlineBlockPseudoAfterBr_FlagDisabled) {
  ScopedSkipPseudoOnlyLinesInLineNavigationForTest disable_flag(false);
  // ... same setup, verify old (buggy) behavior occurs ...
}
```

#### Issue #4: Web test lacks `-expected.txt` baseline
**Severity**: Suggestion
**File**: `third_party/blink/web_tests/editing/selection/modify_extend/extend_by_line_with_pseudo_after_br.html`
**Description**: This is a `testharness.js` test, so it's self-validating and doesn't strictly need a `-expected.txt` file. However, review the test expectations for all platforms (the CL history shows Patch Set 1 failed on `mac-rel` for this test, then Patch Set 2 fixed it). Confirm the expectations are now stable across all platforms.

#### Issue #5: Web test third case assertion could be stronger
**Severity**: Suggestion
**File**: `third_party/blink/web_tests/editing/selection/modify_extend/extend_by_line_with_pseudo_after_br.html`
**Line**: 69-72
**Description**: The third test case ("Extend forward never causes backward selection") checks `range.toString().length > 0` and verifies the selection didn't go backward, but only handles the case where anchor and focus are in the same node. If anchor and focus are in different nodes but focus is before anchor in document order, this assertion wouldn't catch it. The `assert_true(range.toString().length > 0)` partially covers this, but a `compareDocumentPosition` check would be more robust.
**Suggestion**:
```js
const position = selection.anchorNode.compareDocumentPosition(selection.focusNode);
assert_true(
    position === 0
        ? selection.focusOffset >= selection.anchorOffset
        : !!(position & Node.DOCUMENT_POSITION_FOLLOWING),
    'Focus should be at or after anchor in document order');
```

---

### Positive Observations

- **Root cause analysis is solid.** The commit message clearly explains the problem: pseudo-elements don't have DOM nodes for caret positioning, causing `PositionForPoint()` to fail. The fix correctly targets `CanBeCaretContainer()`.
- **Consistent with existing patterns.** The commit message correctly notes that `MoveToFirstNonPseudoLeaf()` and `MoveToLastNonPseudoLeaf()` already use `NonPseudoNode()` for the same purpose — confirmed by reviewing `inline_cursor.cc` lines 1018-1029 and 1087+.
- **Feature flag for safe rollback.** Adding a `RuntimeEnabledFeature` as a kill switch is good Chromium practice for behavior changes in editing code. This allows disabling the fix via Finch if regressions are discovered.
- **Good test coverage.** Both a C++ unit test and a web platform test are provided, covering forward extension, backward extension, and the original bug's signature behavior.
- **Minimal, surgical change.** The fix is only 12 lines of logic change in one function, with well-scoped impact.
- **The fallback path works correctly.** When `CanBeCaretContainer()` returns false for the adjacent line, the callers (`PreviousLinePosition`/`NextLinePosition`) fall through to `PreviousRootInlineBoxCandidatePosition`/`NextRootInlineBoxCandidatePosition`, which correctly find the next valid line across block boundaries.

---

### Overall Assessment

**LGTM with minor comments**

This is a well-targeted fix for a real selection bug. The root cause analysis is correct, the fix is consistent with existing Blink patterns, and the feature flag provides a safe rollback mechanism. The main suggestions are:

1. **Simplify the nested conditional** (Issue #1) — this is the most actionable feedback and would improve readability.
2. **Strengthen the C++ test assertion** (Issue #2) — use strict `<` instead of `<=`.
3. **Consider adding a flag-disabled test** (Issue #3) — to validate the kill switch works.

None of these are blocking; the CL is correct and safe to land as-is.
