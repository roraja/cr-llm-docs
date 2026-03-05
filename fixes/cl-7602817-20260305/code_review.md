# Code Review: CL 7602817 — Make DOMSelection not use VisibleSelection

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7602817
**Author:** Rohan Raja (roraja@microsoft.com)
**Reviewer:** Sambamurthy Bandaru

---

## Review Summary

| Category    | Status | Notes |
|-------------|--------|-------|
| Correctness | ⚠️     | Build failure due to linker errors in tests; one semantic concern in `CreateRangeFromSelectionEditor` around `ComputeRange` vs `ToNormalizedEphemeralRange` differences |
| Style       | ✅     | Follows Chromium conventions; good comments explaining rationale |
| Security    | ✅     | No security concerns; null checks added appropriately |
| Performance | ✅     | Achieves the stated goal of eliminating expensive `UpdateStyleAndLayout` calls on every selection property access |
| Testing     | ⚠️     | Good test coverage added but tests have linker errors preventing compilation |

---

## Detailed Findings

#### Issue #1: Build Failure — Linker Error for `CorrectedSelectionAfterCommand` and `SelectionForUndoStep::From`
**Severity**: Critical
**File**: `editing_commands_utilities_test.cc`
**Line**: 125, 165 (in the new test code)
**Description**: The CL fails to compile on both Mac and Windows trybots. The linker reports:
- `undefined symbol: blink::CorrectedSelectionAfterCommand(blink::SelectionForUndoStep const&, blink::Document*)`
- `undefined symbol: blink::SelectionForUndoStep::From(blink::SelectionInDOMTree const&)`

This indicates that `editing_commands_utilities_test.cc` references `CorrectedSelectionAfterCommand` and `SelectionForUndoStep::From`, but the test target's BUILD.gn dependencies don't link in the object files containing these symbols. The test file is listed in `editing/build.gni` under the `unit_tests` source set, and it likely needs an additional dependency on the target that compiles `selection_for_undo_step.cc` and `editing_commands_utilities.cc`. Alternatively, these symbols may not be exported from their respective libraries.

**Suggestion**: Check the BUILD.gn for the `blink_unittests` target and ensure the source set or component containing `selection_for_undo_step.cc` is a dependency. Specifically, verify that `SelectionForUndoStep::From()` is available to the test target. This is the most critical issue blocking this CL from landing.

---

#### Issue #2: Semantic Difference Between `ComputeRange()` and `ToNormalizedEphemeralRange()`
**Severity**: Major
**File**: `dom_selection.cc`
**Lines**: 805, 832
**Description**: In `deleteFromDocument()` (line 805) and `containsNode()` (line 832), the code now uses `GetDOMSelection().ComputeRange()` in place of the old `Selection().ComputeVisibleSelectionInDOMTree().ToNormalizedEphemeralRange()`.

`ToNormalizedEphemeralRange()` performs additional normalization:
- For caret selections, it calls `MostBackwardCaretPosition().ParentAnchoredEquivalent()` to move upstream.
- For range selections, it calls `MostForwardCaretPosition().ParentAnchoredEquivalent()` for the start and `MostBackwardCaretPosition().ParentAnchoredEquivalent()` for the end.

`ComputeRange()` simply returns `EphemeralRange(ComputeStartPosition(), ComputeEndPosition())` without any position normalization.

The commit message states that canonicalization is moved to write time via `CorrectedSelectionAfterCommand`, but not all selection-setting code paths go through `CorrectedSelectionAfterCommand`. Selections set directly via `SetSelection()` (e.g., from JavaScript `Selection.addRange()`, `Selection.collapse()`, etc.) bypass `CorrectedSelectionAfterCommand`. This means `deleteFromDocument()` and `containsNode()` may receive raw, non-normalized positions that previously were always normalized at read time.

**Suggestion**: Verify that all write-time code paths that can set the selection also apply the necessary canonicalization, or document why it's acceptable for JS-originated selections to skip canonicalization in these methods. Consider whether `ComputeRange()` usage here needs a `ParentAnchoredEquivalent()` pass similar to what `CreateRangeFromSelectionEditor()` does.

---

#### Issue #3: Redundant `CreateVisibleSelection` Call in `CorrectedSelectionAfterCommand`
**Severity**: Minor
**File**: `editing_commands_utilities.cc`
**Lines**: 673-674
**Description**: The new code in `CorrectedSelectionAfterCommand` still calls `CreateVisibleSelection(passed_selection.AsSelection()).AsSelection()` and then applies `ParentAnchoredEquivalent()` on top. This means the code:
1. Creates a `VisibleSelection` (which does its own canonicalization including `CreateVisiblePosition` calls for anchor/focus).
2. Gets the selection back as `SelectionInDOMTree`.
3. Then applies `ParentAnchoredEquivalent()` on the already-canonicalized positions.

The stated goal is to remove `VisibleSelection` usage from `DOMSelection`, but this code still relies on `VisibleSelection` at write time. This is not necessarily wrong (since the feature flag gates it), but the intermediate `VisibleSelection` creation still triggers the expensive canonicalization that the CL aims to avoid. The net effect is moving the cost from read-time to write-time, which is beneficial since writes are less frequent than reads, but worth noting.

**Suggestion**: Consider adding a comment explaining that the `CreateVisibleSelection` call is intentionally retained at write time to preserve correctness, and that the performance win comes from avoiding it on every read.

---

#### Issue #4: `element_fragment_anchor.cc` — `CreateVisiblePosition` May Return Null
**Severity**: Minor
**File**: `element_fragment_anchor.cc`
**Lines**: 196-198
**Description**: When the feature is enabled, the code calls:
```cpp
pos = CreateVisiblePosition(
          Position::FirstPositionInOrBeforeNode(*anchor_node_))
          .DeepEquivalent();
```
`CreateVisiblePosition` can return a null `VisiblePosition` (e.g., if the position is in a non-rendered subtree), in which case `DeepEquivalent()` returns a null `Position`. The subsequent `pos.IsConnected()` check on line 202 handles this case correctly (a null position is not connected), so this is safe. However, this is a subtle correctness point worth a brief comment.

**Suggestion**: Consider adding a brief comment noting that `pos` may be null and `IsConnected()` handles that case.

---

#### Issue #5: `CreateRangeFromSelectionEditor` — Document Selection Path Computes Positions Twice
**Severity**: Minor
**File**: `dom_selection.cc`
**Lines**: 689-698
**Description**: In the document selection path (lines 689-698), the code computes `anchor` and `focus` via `ParentAnchoredEquivalent()` on lines 682-683, then computes `start` and `end` from `ComputeStartPosition().ParentAnchoredEquivalent()` and `ComputeEndPosition().ParentAnchoredEquivalent()`. The `start`/`end` are what's actually used to construct the `EphemeralRange`, meaning the `anchor`/`focus` computation on lines 682-683 is done but not used in this code path (they're only used to check `IsInShadowTree()`).

This is not a bug but is slightly wasteful — the `ParentAnchoredEquivalent()` is called on anchor/focus even though only the `AnchorNode()` is needed for the shadow tree check. The original code only needed the anchor's `AnchorNode()`.

**Suggestion**: Consider checking shadow tree membership before calling `ParentAnchoredEquivalent()` on anchor/focus, or compute only the anchor node needed for the shadow check without full `ParentAnchoredEquivalent()`. Alternatively, restructure to avoid the double computation. This is a micro-optimization and not critical.

---

#### Issue #6: Two `Change-Id` Lines in Commit Message
**Severity**: Minor
**File**: Commit message
**Description**: The commit message contains two `Change-Id` lines:
```
Change-Id: Id7dfdcdbb88e22e18bcd35e3369ace4842637d79
Co-authored-by: Copilot <...>
Change-Id: Ic7e0351645d38e5c2f98a9e705017825403eb92e
```
Only the last `Change-Id` is used by Gerrit. The first one appears to be a leftover from an earlier version. This won't cause functional issues but is against Chromium commit message conventions.

**Suggestion**: Remove the first `Change-Id` line from the commit message.

---

#### Issue #7: Inconsistent Brace Style for Single-Statement `if` Blocks
**Severity**: Minor (Suggestion)
**File**: `dom_selection.cc`
**Lines**: 190-192 (new braces added), vs. other unchanged `if` statements without braces
**Description**: The CL adds braces to the `if (GetDOMSelection().IsCaret())` block (lines 190-192), which is a good Chromium style practice. However, surrounding unchanged code still has single-statement `if` blocks without braces (e.g., line 187: `return "None";`, line 702: `return EphemeralRange();`). This creates inconsistency within the same function. Per Chromium style, adding braces to touched lines is fine; this is merely an observation.

**Suggestion**: No action required. The selective addition of braces to modified lines is acceptable.

---

#### Issue #8: Test `CaretPositionBeforeTableAfterEditCommand` Has Weak Assertion
**Severity**: Minor
**File**: `dom_selection_test.cc`
**Lines**: ~230 (in the new test)
**Description**: The test `CaretPositionBeforeTableAfterEditCommand` asserts:
```cpp
EXPECT_TRUE(anchor == div || anchor == table);
```
This assertion is very permissive — it accepts either the div or the table as the anchor node, which means the test doesn't actually verify the specific behavior change this CL introduces (that the position should be canonicalized to the parent). A more precise test would assert exactly which node is expected based on whether the feature flag is enabled.

**Suggestion**: Use `EXPECT_EQ(table, anchor)` since with `GetDOMSelection()` returning raw positions (and no write-time canonicalization for `SetSelection` calls), the anchor should remain `table`. If the intent is to test the write-time canonicalization path, the test should go through a code path that invokes `CorrectedSelectionAfterCommand`, not direct `SetSelection`.

---

## Positive Observations

- **Well-structured feature flag**: All changes are gated behind `RemoveVisibleSelectionInDOMSelectionEnabled()`, enabling safe rollback. This is excellent practice for a behavioral change of this scope.
- **Thorough null checking**: The rewritten `CreateRangeFromSelectionEditor()` adds comprehensive null checks for `ParentAnchoredEquivalent()` results, properly handling edge cases like detached nodes (referencing crbug.com/889737).
- **Good commit message**: The commit message clearly explains the motivation (eliminating expensive layout calls), the approach (moving canonicalization from read-time to write-time), and lists all key changes.
- **Extensive test coverage**: 175 lines of new tests in `dom_selection_test.cc` and 83 lines in `editing_commands_utilities_test.cc` covering various scenarios including caret, range, containsNode, deleteFromDocument, and table positions.
- **Performance improvement**: Removing `UpdateStyleAndLayout` from `GetVisibleSelection()` (called on every selection property access from JS) is a meaningful performance win for web pages that frequently query selection state.
- **WPT test fix**: The deletion of `anchor-removal-expected.txt` indicates this CL fixes a previously-failing WPT test, which improves web platform compatibility.

---

## Overall Assessment

**Needs changes before approval**

The CL has a clear and valuable goal (removing expensive layout-triggering calls from DOMSelection read paths) and is well-structured with a feature flag for safe rollback. However, there are two blocking issues:

1. **Critical**: The CL does not compile — linker errors for `CorrectedSelectionAfterCommand` and `SelectionForUndoStep::From` in the test target must be fixed.

2. **Major**: The semantic difference between `ComputeRange()` and `ToNormalizedEphemeralRange()` in `deleteFromDocument()` and `containsNode()` needs to be analyzed more carefully. Not all selection-setting paths go through `CorrectedSelectionAfterCommand`, so selections set directly from JS may have non-normalized positions that these methods previously handled via `ToNormalizedEphemeralRange()`.

Once the build is fixed and the `ComputeRange()` vs `ToNormalizedEphemeralRange()` semantic gap is addressed (or explicitly documented as acceptable), this CL should be ready for another review pass.
