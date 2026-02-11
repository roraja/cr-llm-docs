# Code Review: CL 7452739 — [Editing] Control text truncation based on caret selection

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7452739
**Author:** Shweta Bindal <shwetabindal@microsoft.com>
**Reviewer:** Code Review Bot
**Date:** 2026-02-09

---

## Review Summary

| Category    | Status | Notes |
|-------------|--------|-------|
| Correctness | ⚠️     | Dead declaration (`CollectElementsFromSelection`); parent-element heuristic may miss cases; layout invalidation reason misleading |
| Style       | ⚠️     | Minor: commit message references `ContainsSelection()` but code uses `ContainsEntireSelection()`; two unrelated bug fixes bundled |
| Security    | ✅     | No security concerns identified |
| Performance | ✅     | +1 bit per LayoutObject is negligible; layout invalidation guarded by `text-overflow` check |
| Testing     | ❌     | Core mechanism lacks unit tests; selection controller test doesn't exercise the actual fallback path; significant coverage gaps |

---

## Detailed Findings

### Issue #1: Dead declaration — `CollectElementsFromSelection` declared but never defined
**Severity**: Major
**File**: `selection_editor.h`
**Line**: ~109 (new private method declaration)
**Description**: `CollectElementsFromSelection(const SelectionInDOMTree&, HeapHashSet<Member<Element>>&)` is declared as a private method in the header, and `heap_hash_set.h`, `element_traversal.h`, and `range.h` are added as includes in `selection_editor.cc`, but no definition of this method exists in the `.cc` file. This is dead code that will cause a linker error if ever called, and the unused includes add unnecessary compile-time cost.
**Suggestion**: Either remove the declaration and the associated unused includes, or implement the method if it is needed for the intended design. If it was part of an earlier iteration that was refactored into `UpdateContainsEntireSelectionFlagForElements`, clean it up.

---

### Issue #2: Selection controller test does not exercise the fallback code path
**Severity**: Critical (Testability)
**File**: `selection_controller_test.cc`
**Line**: ~594 (new test `ExpandWithGranularityWithInputInUserSelectNone`)
**Description**: The test directly calls `ExpandWithGranularity()` and asserts it returns a range. However, the actual fix in `selection_controller.cc` adds a *fallback* that activates **only** when `ExpandWithGranularity` returns a caret (collapsed selection). If the test's call to `ExpandWithGranularity` already returns a range, the fallback code at lines 1096–1115 of `selection_controller.cc` is **never exercised** by any test. This means the new code path has zero test coverage.
**Suggestion**: Either:
1. Construct a test scenario where `ExpandWithGranularity` actually returns a caret (triggering the fallback), or
2. Test through the full `HandleTripleClick` code path which invokes both `ExpandWithGranularity` and the fallback, or
3. Add a separate test that directly tests `StartOfParagraph`/`EndOfParagraph` with `kCanCrossEditingBoundary` on the problematic DOM structure.

---

### Issue #3: Weak test assertion — does not verify selection extent
**Severity**: Major (Testability)
**File**: `selection_controller_test.cc`
**Line**: ~626 (`EXPECT_NE(selection.Anchor(), selection.Focus())`)
**Description**: The assertion only checks that anchor and focus are different (i.e., it's a range selection). It does not verify *what* text is selected. The selection could be a single character and this test would still pass. For a paragraph expansion test, the assertion should verify the selection spans the full paragraph content.
**Suggestion**: Replace with position-specific assertions, e.g.:
```cpp
EXPECT_EQ(selection.Anchor(), PositionInFlatTree(text_node, 0));
EXPECT_EQ(selection.Focus(), PositionInFlatTree(text_node, text_node->length()));
```
Or at minimum assert `selection.IsRange()` together with the expected start/end positions or selected text content.

---

### Issue #4: No unit test for `UpdateContainsEntireSelectionFlagForElements`
**Severity**: Major (Testability)
**File**: `selection_editor.cc` / `selection_editor.h`
**Description**: `UpdateContainsEntireSelectionFlagForElements` is the **core mechanism** of this CL — it manages the `ContainsEntireSelection` bitfield and triggers layout invalidation. There are no C++ unit tests for this method. It is only tested indirectly through WPT reftests (visual comparison), which cannot verify:
- The flag is correctly set on the right `LayoutObject`
- The flag is correctly cleared when selection moves away
- Layout invalidation occurs only for elements with non-clip `text-overflow`
- Behavior when selection becomes `IsNone()`
- Behavior with cross-element selections (anchor/focus in different parents)
**Suggestion**: Add `SelectionEditorTest` cases (or extend existing selection tests) that:
1. Set a selection inside an element, verify `ContainsEntireSelection()` is true on its layout object
2. Move the selection out, verify the flag is cleared
3. Verify `SetNeedsLayout` is called only for non-clip `text-overflow` elements
4. Verify behavior when anchor and focus have different parent elements

---

### Issue #5: No test for ellipsis restoration when caret leaves
**Severity**: Major (Testability)
**File**: N/A (missing test)
**Description**: There is no test (unit or WPT) that verifies ellipsis rendering **resumes** when the caret/selection is moved out of an element with `text-overflow: ellipsis`. All existing tests only check that ellipsis is suppressed when the caret is present. The reverse transition (caret leaves → ellipsis returns) is untested.
**Suggestion**: Add a WPT test or testharness.js test that:
1. Focuses an element with `text-overflow: ellipsis` (ellipsis suppressed)
2. Moves focus/selection out of the element
3. Verifies ellipsis is rendered again (via visual comparison or computed layout metrics)

---

### Issue #6: No test for `<input>` element with `text-overflow: ellipsis`
**Severity**: Major (Testability)
**File**: N/A (missing test)
**Description**: The old StyleAdjuster hack specifically handled focused `<input>` placeholder elements. The old test `text-overflow-input-focus-placeholder.html` is deleted, but no new test replaces it for `<input type="text">` with `text-overflow: ellipsis`. The new WPT tests only cover `contenteditable` divs and `<textarea>`. Given that `<input>` elements use shadow DOM (the selection's anchor/focus live inside shadow DOM, not the visible `<input>` block), the parent-element heuristic in `UpdateContainsEntireSelectionFlagForElements` may not correctly identify the containing block for inputs.
**Suggestion**: Add a WPT reftest for `<input type="text" style="text-overflow: ellipsis">` with a focused caret, and verify ellipsis is suppressed. This is a potential regression from removing the old hack.

---

### Issue #7: Parent-element heuristic may miss the correct LayoutBlockFlow
**Severity**: Major
**File**: `selection_editor.cc`
**Line**: ~155–170 (`UpdateContainsEntireSelectionFlagForElements`)
**Description**: The code determines the "containing element" by checking `anchor->parentElement() == focus->parentElement()`. This sets the `ContainsEntireSelection` flag on the *parent element's* LayoutObject. However, in `inline_layout_algorithm.cc`, the flag is read from `node_.GetLayoutBlockFlow()`. If the parent element is not the LayoutBlockFlow that performs truncation (e.g., with anonymous block boxes, inline wrappers, or shadow DOM in `<input>`/`<textarea>`), the flag check will miss.

For `<input>` elements, the selection lives in shadow DOM. The parent of anchor/focus will be a shadow DOM inner element, not the visible `<input>` or its LayoutBlockFlow. This means the flag may be set on the wrong LayoutObject and the LayoutBlockFlow won't see it.
**Suggestion**: Instead of using `parentElement()`, walk up to the nearest `LayoutBlockFlow` ancestor of the selection's anchor/focus positions and set the flag there. This would correctly handle shadow DOM, anonymous blocks, and complex nesting.

---

### Issue #8: Misleading layout invalidation reason
**Severity**: Minor
**File**: `selection_editor.cc`
**Line**: ~175, ~185
**Description**: `layout_invalidation_reason::kStyleChange` is used when triggering layout invalidation, but the style didn't actually change — the selection changed. This could confuse debugging/tracing tools that use invalidation reasons for diagnostics.
**Suggestion**: Add a new `layout_invalidation_reason::kSelectionChangeForTextOverflow` (or similar) to accurately describe why layout was invalidated. This aids in performance debugging and tracing.

---

### Issue #9: Commit message inconsistency
**Severity**: Minor
**File**: `/COMMIT_MSG`
**Description**: The commit message refers to `LayoutObject::ContainsSelection()` and `LayoutBlockFlow::ContainsSelection()`, but the actual code uses `ContainsEntireSelection()`. This inconsistency makes it harder to grep the codebase when investigating related code from git blame.
**Suggestion**: Update the commit message to use the actual method name `ContainsEntireSelection()`.

---

### Issue #10: Two unrelated bug fixes bundled in one CL
**Severity**: Minor
**File**: `selection_controller.cc`, `selection_controller_test.cc`
**Description**: The triple-click paragraph expansion fix (crbug.com/446113738) in `selection_controller.cc` is completely unrelated to the text-overflow ellipsis suppression (crbug.com/40731275). The two changes share no code paths and address different bugs. Bundling them in one CL makes bisection harder if either change causes a regression, and complicates review.
**Suggestion**: Split into two separate CLs: one for the text-overflow ellipsis mechanism and one for the triple-click fix.

---

### Issue #11: Feature flag gating — no `<input>` coverage in WPT
**Severity**: Minor
**File**: `runtime_enabled_features.json5`
**Description**: The feature flag `TextOverflowClipWithCaretSelection` has `status: "test"`, meaning it's only enabled in test builds. The WPT tests (in `external/wpt/`) are designed to be run across browsers, but they won't activate without the feature flag. There's no documentation or bug tracking the promotion timeline from `"test"` to `"stable"`.
**Suggestion**: Add a comment in the feature flag entry or a note in the tracking bug about the expected promotion timeline. Also ensure the WPT tests have appropriate `<meta>` tags or test expectations that handle the feature being disabled.

---

### Issue #12: No test for range selection (non-caret) in ellipsed element
**Severity**: Minor (Testability)
**File**: N/A (missing test)
**Description**: The CSS Overflow spec allows suppression during "selecting" — meaning range selections, not just carets. The `UpdateContainsEntireSelectionFlagForElements` method handles both caret and range selections (it checks anchor/focus parent equality, not selection type). However, there is no test that verifies ellipsis is suppressed when a range selection exists within an element with `text-overflow: ellipsis`.
**Suggestion**: Add a test where a range selection is made within an ellipsed element and verify ellipsis suppression.

---

### Issue #13: Missing spec compliance for cross-element selection
**Severity**: Minor
**File**: `selection_editor.cc`
**Description**: The spec says UAs MAY suppress ellipsis during "selecting." The current implementation only handles selections where anchor and focus share the same parent element. If a user drags a selection that starts in one element and ends in another (both with `text-overflow: ellipsis`), neither element will have its flag set (since `anchor->parentElement() != focus->parentElement()`). This is technically spec-compliant (the "may" is permissive), but may lead to inconsistent UX where ellipsis is sometimes suppressed and sometimes not depending on selection topology.
**Suggestion**: Document this limitation. Consider extending the heuristic to handle cross-element selections in a follow-up CL.

---

## Positive Observations

- **Clean architectural approach**: Replacing the StyleAdjuster hack with a layout-level mechanism is the correct design direction. The old TODO comment explicitly called for this approach, and this CL delivers it.
- **Spec-aligned implementation**: The CSS Overflow L3 spec uses "may" language, and the implementation correctly takes advantage of this permissive wording. Suppressing at layout time rather than mutating computed style is architecturally superior.
- **Feature flag gating**: Using a runtime feature flag (`TextOverflowClipWithCaretSelection` with `status: "test"`) for the new behavior is good practice for a significant rendering change. It allows incremental rollout and easy rollback.
- **Good use of bitfield**: Adding `ContainsEntireSelection` as a single bit in `LayoutObjectBitfields` has minimal memory impact due to alignment padding.
- **New WPT tests**: Adding WPT tests for `contenteditable` and `<textarea>` elements with proper spec references (`link rel="help"`) is good practice and contributes upstream test coverage.
- **Comprehensive platform baselines**: Updating PNG baselines across linux, mac, and windows platforms shows thorough cross-platform testing.

---

## Overall Assessment

**Needs changes before approval**

The core architectural approach is sound and spec-compliant, replacing a known hack with a proper layout-level mechanism. However, there are significant testability gaps that must be addressed before landing:

1. **Critical**: The selection controller test (`ExpandWithGranularityWithInputInUserSelectNone`) does not exercise the actual fallback code path it claims to test. The new code in `selection_controller.cc` (lines 1096–1115) has zero test coverage.

2. **Major**: The core mechanism (`UpdateContainsEntireSelectionFlagForElements`) has no C++ unit tests. Only visual reftests cover it indirectly.

3. **Major**: No test for `<input>` elements, which is a potential regression since the old hack specifically targeted `<input>` placeholders and the old test is deleted without replacement.

4. **Major**: Dead code (`CollectElementsFromSelection` declaration without definition) should be removed.

5. **Major**: The parent-element heuristic may not correctly identify the `LayoutBlockFlow` for `<input>` and `<textarea>` elements due to shadow DOM, which could mean the feature doesn't actually work for these element types despite having WPT tests.

The CL should also ideally be split to separate the triple-click fix (crbug.com/446113738) from the text-overflow mechanism (crbug.com/40731275).
