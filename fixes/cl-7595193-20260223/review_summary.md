# CL 7595193 — Review Summary

**CL Title:** [Editing] Fix incorrect text selection behavior with :after pseudo-element and `<br>` element  
**Author:** Tanu Jain (tanujain@microsoft.com)  
**URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7595193  
**Files Changed:** 4 files, +129/−2 lines  
**Bug:** [crbug.com/357889508](https://crbug.com/357889508)

---

## 1. Executive Summary

This CL fixes a text selection bug where extending selection by line (Shift+Down/Up) misbehaves when a line contains only pseudo-elements (e.g., `::after` with `display:inline-block`) after a `<br>` element. The root cause was that `CanBeCaretContainer()` in `AbstractLineBox` treated pseudo-element-only lines as valid caret targets, but `PositionForPoint()` would fail because pseudo-elements have no DOM nodes for caret positioning, causing the selection to extend in the wrong direction. The fix uses `NonPseudoNode()` instead of the implicit `GetNode()` check to skip such lines, gated behind a new runtime feature flag `SkipPseudoOnlyLinesInLineNavigation`.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 4 | The change is well-scoped and the commit message clearly explains the root cause, the fix, and the rationale. |
| Maintainability | 3 | The runtime feature flag adds code complexity; once the flag is permanently enabled, the old branch should be cleaned up. |
| Extensibility | 4 | Uses the existing `NonPseudoNode()` pattern already established by `MoveToFirstNonPseudoLeaf()` / `MoveToLastNonPseudoLeaf()`. |
| Consistency | 4 | Aligns with the existing pseudo-element filtering pattern used in `InlineCursor` navigation methods. |

### Architecture Diagram

```
                         Selection Extension (Shift+Down)
                                    │
                                    ▼
                        SelectionModifier::Modify()
                                    │
                                    ▼
                    PreviousLinePosition / NextLinePosition
                                    │
                                    ▼
                         AbstractLineBox::NextLine()
                                    │
                                    ▼
                   ┌─── CanBeCaretContainer() ───┐
                   │                              │
                   │  For each inline leaf:       │
                   │  OLD: GetLayoutObject()      │
                   │       + IsInlineLeaf()       │
                   │       → return true          │
                   │                              │
                   │  NEW (with flag):            │
                   │  GetLayoutObject()           │
                   │  + IsInlineLeaf()            │
                   │  + NonPseudoNode() ──────────│──► Skip pseudo-only lines
                   │       → return true          │
                   └──────────────┬───────────────┘
                                  │
                                  ▼
                      PositionForPoint() on valid line
                                  │
                                  ▼
                         Selection updated correctly
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | The core fix (using `NonPseudoNode()`) is correct and consistent with existing patterns. The feature flag ensures safe rollout. |
| Efficiency | 5 | No performance impact — `NonPseudoNode()` is a trivial inline check (tests `IsPseudoElement()`). |
| Readability | 3 | The nested `if` with the feature flag check makes the code harder to follow. A refactored structure would improve clarity. |
| Test Coverage | 4 | Includes both a C++ unit test and a web platform test covering forward/backward extension and the original bug scenario. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

- **None identified.** The fix is correct and well-targeted.

### Major Issues (Should Fix)

1. **Nested conditional structure is unnecessarily complex.** The current code uses a nested `if` inside the feature flag check with an `else` that duplicates the original return. This can be simplified:
   ```cpp
   if (current.GetLayoutObject() && current.IsInlineLeaf()) {
     if (!RuntimeEnabledFeatures::SkipPseudoOnlyLinesInLineNavigationEnabled() ||
         current.GetLayoutObject()->NonPseudoNode()) {
       return true;
     }
   }
   ```
   This is semantically identical, reduces nesting by one level, and is easier to reason about.

2. **Unit test assertion is weak.** The C++ test (`ExtendByLineWithInlineBlockPseudoAfterBr`) checks that `Anchor <= Focus` but does not validate the specific position the focus lands on (e.g., that it's within `"second"` text). The web test is more thorough in this regard, but the C++ test should verify the expected endpoint for stronger regression protection.

### Minor Issues (Nice to Fix)

1. **Feature flag cleanup plan not documented.** The flag `SkipPseudoOnlyLinesInLineNavigation` is set to `status: "stable"`, meaning it's enabled by default. There should be a plan (or a follow-up bug) to remove the flag and the old code path after sufficient bake time, to avoid accumulating dead code.

2. **Comment in `CanBeCaretContainer` could be more precise.** The comment says "Skip lines that only contain pseudo-elements" but the check actually skips individual pseudo-element leaf nodes. If a line has both a pseudo-element and a real node, it still returns `true` when the real node is encountered. The comment should clarify this is per-leaf filtering, not whole-line skipping.

3. **Web test `extend_by_line_with_pseudo_after_br.html` test 2 ("Extend backward skips pseudo-only line") may be imprecise.** The test starts at offset 5 in div2, extends backward, and asserts focus is in div1. However, backward line extension from div2 would go to the last line of div1 — it's unclear whether it should skip the pseudo-only line in backward direction or land on the first (real content) line. The test assertion may pass for the wrong reason if the pseudo-only line is the second line of the same block-level element.

### Suggestions (Optional)

1. **Consider adding a test case for `::before` pseudo-elements** as well, since the comment mentions both `::before` and `::after` but the tests only exercise `::after`.

2. **Consider testing `kMove` (caret movement) in addition to `kExtend` (selection extension).** The `CanBeCaretContainer()` method affects both operations, but only extension is tested.

3. **Consider edge case: line with mixed pseudo-element and real content.** The current fix correctly handles this case (returns `true` when a real node is found), but an explicit test would document this expected behavior.

---

## 5. Test Coverage Analysis

### Existing Tests
- **C++ unit test** (`SelectionModifierTest.ExtendByLineWithInlineBlockPseudoAfterBr`): Tests forward extension by line with `::after` pseudo and `<br>`, verifying selection extends forward (not backward).
- **Web test** (`extend_by_line_with_pseudo_after_br.html`): Three test cases covering:
  1. Forward extension skipping pseudo-only line
  2. Backward extension skipping pseudo-only line
  3. Forward extension never causing backward selection (original bug repro)

### Missing Tests
- **No test for `::before` pseudo-elements** — the fix applies to all pseudo-elements, but only `::after` is tested.
- **No caret movement test** (`kMove` alteration) — `CanBeCaretContainer()` is used for both move and extend.
- **No test for mixed pseudo + real content on the same line** — verifying that such lines are NOT skipped.
- **No test for `display:block` pseudo-elements** — only `display:inline-block` is tested. A `display:block` pseudo creates a different layout scenario.
- **No test with the feature flag disabled** — while the flag is stable, a test verifying the old behavior when the flag is off would ensure the flag mechanism works correctly.

### Recommended Additional Tests
1. A C++ test for `::before` with `display:inline-block` and `<br>`.
2. A C++ test for caret movement (not just selection extension).
3. A C++ test verifying a line with mixed pseudo + real content IS treated as a valid caret container.

---

## 6. Security Considerations

- **No security implications identified.** The change is confined to selection/caret positioning logic in the editing module. It does not affect DOM access patterns, cross-origin boundaries, or any security-sensitive subsystems. The use of `NonPseudoNode()` is a well-established safe API within the layout object hierarchy.

---

## 7. Performance Considerations

- **No performance concerns.** The additional `NonPseudoNode()` check is an O(1) inline operation (a boolean flag test on `IsPseudoElement()`). It is called only during line navigation (user-driven keyboard interaction), not during layout or paint. The feature flag check (`RuntimeEnabledFeatures::SkipPseudoOnlyLinesInLineNavigationEnabled()`) is also trivially inexpensive.

- **No benchmarking recommended.** The change is on a rarely-exercised code path (line-by-line selection extension) with negligible added cost.

---

## 8. Final Recommendation

**Verdict:** APPROVED_WITH_COMMENTS

**Rationale:**  
The fix correctly addresses the root cause of the selection bug by filtering out pseudo-element-only lines from caret container candidates, consistent with the existing `MoveToFirstNonPseudoLeaf()` / `MoveToLastNonPseudoLeaf()` patterns. The runtime feature flag provides a safe rollback mechanism. Both C++ and web tests are provided. The code is correct and the change is minimal.

The main feedback items are about code clarity (simplifying the nested conditional) and test strength (asserting specific focus positions and covering additional pseudo-element variants).

**Action Items for Author:**
1. **Simplify the nested conditional** in `CanBeCaretContainer()` to reduce nesting and improve readability (see Major Issue #1 above).
2. **Strengthen the C++ test** to assert the specific focus position (e.g., verify it lands within the "second" text node).
3. **File a follow-up bug** for feature flag cleanup once the change has sufficient bake time on stable.
4. *(Optional)* Add test coverage for `::before`, caret movement (`kMove`), and mixed pseudo+real content lines.

---

## 9. Comments for Gerrit

### Comment 1 — `selection_modifier_line.cc` (line 72-82)
**File:** `third_party/blink/renderer/core/editing/selection_modifier_line.cc`  
**Line:** 72  
**Severity:** Major (readability)

> nit: The nested `if` can be simplified to a single condition to reduce nesting:
> ```cpp
> if (current.GetLayoutObject() && current.IsInlineLeaf()) {
>   if (!RuntimeEnabledFeatures::SkipPseudoOnlyLinesInLineNavigationEnabled() ||
>       current.GetLayoutObject()->NonPseudoNode()) {
>     return true;
>   }
> }
> ```
> This is semantically equivalent and easier to follow.

---

### Comment 2 — `selection_modifier_test.cc` (line 222-226)
**File:** `third_party/blink/renderer/core/editing/selection_modifier_test.cc`  
**Line:** 222  
**Severity:** Minor (test quality)

> The assertion `Anchor <= Focus` verifies directionality but not the specific position. Consider asserting that the focus lands within the expected node (e.g., the "second" text node in the second div) for stronger regression protection. The web test is more thorough here, but the C++ test would benefit from a similar level of specificity.

---

### Comment 3 — `selection_modifier_line.cc` (line 74-76)
**File:** `third_party/blink/renderer/core/editing/selection_modifier_line.cc`  
**Line:** 74  
**Severity:** Minor (comment accuracy)

> The comment says "Skip lines that only contain pseudo-elements" but the check is per-leaf, not per-line. A line with both pseudo-element leaves and real content leaves will still be treated as a valid caret container (correct behavior). Consider updating the comment to: "Skip pseudo-element inline leaves since they don't have DOM nodes for caret positioning. If any non-pseudo leaf exists on this line, the line remains valid."

---

### Comment 4 — `runtime_enabled_features.json5` (line 4959-4966)
**File:** `third_party/blink/renderer/platform/runtime_enabled_features.json5`  
**Line:** 4959  
**Severity:** Suggestion

> Since this flag is `status: "stable"` (enabled by default), please file a follow-up bug to track removal of the flag and the old code path after the change has baked on stable. This avoids accumulating dead code behind permanently-enabled flags.

---

### Comment 5 — General
**Severity:** Suggestion

> Nice fix! The root cause analysis is clear and the approach is consistent with existing pseudo-element filtering in `InlineCursor`. A couple of additional test cases would further strengthen coverage:
> - `::before` pseudo-element variant
> - Caret movement (`kMove`) in addition to selection extension (`kExtend`)
> - Mixed pseudo + real content on the same line (verifying it's NOT skipped)
