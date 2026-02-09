# Fix Implementation: 41311101

## Summary
Gated all 7 `ComputeVisibleSelectionInDOMTree()` call sites in `DOMSelection` behind the existing `RemoveVisibleSelectionInDOMSelection` runtime feature flag. When enabled, `DOMSelection` reads directly from `SelectionInDOMTree` instead of going through `VisibleSelection` canonicalization, which incorrectly depends on layout tree traversal and returns `IsNone()` for `display:none` content.

## Bug Details
- **Bug ID**: 41311101
- **Issue URL**: https://issues.chromium.org/issues/41311101
- **CL URL**: https://chromium-review.googlesource.com/c/chromium/src/+/7559230

## Root Cause
`DOMSelection` methods call `ComputeVisibleSelectionInDOMTree()` which creates a `VisibleSelection` that calls `CanonicalizeSelection()` → `CreateVisiblePosition()` → layout tree traversal. Under LayoutNG, this layout tree traversal produces incorrect results — specifically returning `IsNone()==true` for `display:none` content (which has no layout box), causing `rangeCount()` to return 0 and `direction()` to return "none" even when a valid DOM selection exists.

## Solution
Replace all `ComputeVisibleSelectionInDOMTree()` calls in `DOMSelection` with direct `SelectionInDOMTree` reads when the `RemoveVisibleSelectionInDOMSelection` feature flag is enabled. For `IsNone()` checks, use `GetSelectionInDOMTree().IsNone()`. For normalized range computation, use `NormalizeRange(GetSelectionInDOMTree())` instead of `VisibleSelection::ToNormalizedEphemeralRange()`. All changes are fully gated behind the existing feature flag, ensuring zero risk to existing behavior when disabled.

## Files Modified

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| third_party/blink/renderer/core/editing/dom_selection.cc | Modify | +59/-16 |
| third_party/blink/renderer/core/editing/dom_selection_test.cc | Modify | +261/-0 |

## Code Changes

### third_party/blink/renderer/core/editing/dom_selection.cc

#### `direction()` — line 214
```cpp
// Before:
Selection().ComputeVisibleSelectionInDOMTree().IsNone()

// After:
(RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
     ? Selection().GetSelectionInDOMTree().IsNone()
     : Selection().ComputeVisibleSelectionInDOMTree().IsNone())
```

#### `rangeCount()` — lines 231-244
```cpp
// Before: Always calls UpdateStyleAndLayout + ComputeVisibleSelectionInDOMTree
// After: When flag enabled, skips layout update and reads SelectionInDOMTree directly
if (RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()) {
    if (Selection().GetSelectionInDOMTree().IsNone()) {
      return 0;
    }
} else {
    // original code preserved
}
```

#### `CreateRangeFromSelectionEditor()` — lines 689-716
```cpp
// New flag-gated path reads SelectionInDOMTree directly
// Uses NormalizeRange(selection) instead of FirstEphemeralRangeOf(VisibleSelection)
// Shadow adjustment logic preserved identically
```

#### `deleteFromDocument()`, `containsNode()`, `toString()`
```cpp
// All use the same ternary pattern:
RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
    ? NormalizeRange(Selection().GetSelectionInDOMTree())
    : Selection().ComputeVisibleSelectionInDOMTree().ToNormalizedEphemeralRange()
```

## Tests Added

| Test Type | File | Test Name |
|-----------|------|-----------|
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_RangeCountWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_DirectionForwardWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_DirectionNoneWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag` |
| Unit Test | dom_selection_test.cc | `DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag` |

## Verification

| Check | Result |
|-------|--------|
| Build | ✅ Pass |
| Unit Tests (13 new) | ✅ Pass |
| Web Tests (blink_tests) | ✅ Pass |
| Code Review | ✅ Pass |
| CQ Dry Run | 🔄 Pending |

## Chrome Binary Location
```
/workspace/cr1/src/out/41311101/chrome
```

## How to Test
```bash
# Run Chrome with the fix
/workspace/cr1/src/out/41311101/chrome \
  --user-data-dir=/tmp/test-41311101 \
  /home/roraja/src/chromium-docs/active/41311101-Make-DOMSelection-not-to-use-VisibleSelection/llm_out/repro.html
```

## Next Steps
1. Wait for CQ dry run to complete
2. Add reviewers to the CL
3. Address any review feedback
4. Land the change

## References
- CL: https://chromium-review.googlesource.com/c/chromium/src/+/7559230
- Bug: https://issues.chromium.org/issues/41311101
- Related CL 5387033: Canonicalize selection in `createLink` command
- Related CL 5393240: Canonicalize selection in `element.focus()`
- Related CL 5399116: Canonicalize selection after editing commands
- Related CL 5455404: Canonicalize selection after `selectAll` command
- Pending CL 7546713: Original upstream work for this change
