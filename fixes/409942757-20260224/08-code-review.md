# Code Review: 409942757

## Review Summary
The fix correctly addresses the root cause of incorrect cursor positioning when navigating over `contenteditable="false"` elements inside `<pre>` blocks with preserved newlines. Two targeted changes were made to `visible_units.cc`: (1) `MostBackwardCaretPosition()` now avoids crossing preserved newline boundaries when entering a text node from a parent position, and (2) `AtEditingBoundary()` now recognizes positions between editable and non-editable siblings as valid caret candidates. Both changes are well-guarded, well-commented, and accompanied by comprehensive unit and web tests. No issues found.

## Checklist Results

### Correctness ✅
- [x] Fix addresses root cause — The two changes target the exact algorithmic decisions that caused the cursor to jump across visual lines
- [x] No logic errors — The `||` in `AtEditingBoundary` is guarded by `!AtFirstEditingPositionForNode() && !AtLastEditingPositionForNode()` to prevent false positives at container edges
- [x] Edge cases handled — Same-line non-editable elements, different element types (span, button), and both `contenteditable` and `plaintext-only` modes are tested
- [x] Error handling correct — `DynamicTo<Text>` safely handles non-text nodes, `data.length() > 0` prevents empty string access

### Crash Safety ✅
- [x] No null dereferences — `DynamicTo<Text>` returns nullptr safely if `current_node` is not a Text node; `layout_object->Style()` is safe as `layout_object` is confirmed non-null by the loop condition
- [x] No use-after-free — No ownership changes; all references are to existing DOM/layout objects
- [x] No invalid iterators — No iterator usage in the changed code
- [x] Bounds checked — `data.length() > 0` check before `data[data.length() - 1]` prevents out-of-bounds access

### Memory Safety ✅
- [x] Smart pointers used correctly — No new allocations; uses existing node/layout object references
- [x] No memory leaks — No `new` allocations introduced
- [x] Object lifetimes correct — All accessed objects (`text_node`, `data`, `layout_object`) are owned by the DOM tree and valid during the function scope
- [x] Raw pointers safe — `text_node` raw pointer from `DynamicTo<Text>` is used within the same scope, backed by DOM tree ownership

### Thread Safety ✅
- [x] Thread annotations present — N/A, editing code runs on the main thread
- [x] No deadlocks — No locks used
- [x] No race conditions — Single-threaded editing code path
- [x] Cross-thread calls safe — N/A

### DCHECK Safety ✅
- [x] DCHECKs valid — No new DCHECKs added
- [x] Good error messages — N/A
- [x] No DCHECK on user input — N/A

### Code Style ✅
- [x] Follows style guide — Uses Chromium naming conventions, proper `const` usage, standard patterns like `DynamicTo<>`
- [x] Formatted with git cl format — Confirmed, no changes produced
- [x] Good naming — `caret_max_offset`, `text_node`, `data` are clear and descriptive
- [x] Helpful comments — Comments explain "why" (crbug reference, rationale for the guard conditions), not just "what"

### Tests ✅
- [x] Bug scenario covered — 3 unit tests and 7 web test sub-tests directly covering the reported behavior
- [x] Edge cases covered — Same-line spans, buttons, both `contenteditable` and `plaintext-only` modes, regression tests for non-newline cases
- [x] Descriptive names — `MostBackwardCaretPositionPreservedNewlineBeforeNonEditable`, `IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary`, etc.
- [x] Not flaky — Tests use deterministic DOM setup and check structural positions, not timing-dependent values

## Detailed Review

### File: third_party/blink/renderer/core/editing/visible_units.cc

**Lines 868-882 (MostBackwardCaretPosition change)**: ✅ Correct
- Extracts `caret_max_offset` to a local variable (clean).
- `DynamicTo<Text>(current_node)` safely handles non-Text nodes by returning nullptr.
- `data.length() > 0` prevents empty string access before checking `data[data.length() - 1]`.
- `layout_object->Style()->ShouldPreserveBreaks()` correctly gates the fix to only preserved-whitespace contexts (e.g., `<pre>`).
- Returns `position` (the original input) rather than computing a position inside the text node, which correctly stays on the same visual line.
- The `crbug.com/409942757` comment links to the bug for future reference.

**Lines 1113-1123 (AtEditingBoundary change)**: ✅ Correct
- The old code required both adjacent positions to be non-editable (`&&`), which was too restrictive for positions between editable text and non-editable inline elements.
- The new code uses `||` but guards with `!AtFirstEditingPositionForNode() && !AtLastEditingPositionForNode()` to avoid false positives at container edges (where the "outside" is non-editable).
- The comment explains the rationale and links to the cases already handled above.
- The logical structure is clear: first/last positions use the existing conditions above; interior positions use the relaxed `||` condition.

**Note**: The comment block above `AtEditingBoundary` (lines 1090-1098) describes condition 3 as "both the next and previous visually equivalent positions are both non editable" — this is now slightly outdated since the condition was relaxed to `||`. However, the new inline comment at lines 1113-1118 explains the updated logic, which is sufficient. A future cleanup could update the header comment.

### File: third_party/blink/renderer/core/editing/visible_units_test.cc

**Lines 1147-1173 (MostBackwardCaretPositionPreservedNewlineBeforeNonEditable)**: ✅ Correct
- Tests the core fix: `MostBackwardCaretPosition(Position(pre, 1), kCanCrossEditingBoundary)` should not cross the preserved `\n` into the previous text node.
- Uses `EXPECT_NE` to verify the result anchor node is not the text node (i.e., didn't traverse backward across the newline).

**Lines 1175-1189 (IsVisuallyEquivalentCandidateAtEditableNonEditableBoundary)**: ✅ Correct
- Tests that `Position(pre, 1)` between editable text and non-editable span is a valid caret candidate.
- Clean, focused assertion.

**Lines 1191-1210 (PreviousPositionOfNonEditableSpanInPreWithNewline)**: ✅ Correct
- Tests the end-to-end behavior: `PreviousPositionOf` from after the span should not go to end of previous line.
- Uses `EXPECT_NE(Position(text_node, 6), result.DeepEquivalent())` to verify.

### File: third_party/blink/web_tests/editing/selection/modify_move/move_over_non_editable_in_pre.html

**All 7 sub-tests**: ✅ Correct
- Covers both `plaintext-only` and `contenteditable` modes.
- Tests left and right arrow movements.
- Tests different element types (span, button).
- Includes regression tests for same-line non-editable elements.
- Uses `selection_test` and `assert_selection.js` following existing web test patterns.
- Proper `testharness.js` boilerplate.

## Issues Found

### Critical (Must Fix)
None

### Minor (Should Consider)
1. **Slightly outdated header comment**: The comment block at lines 1090-1098 of `visible_units.cc` describes condition 3 as requiring "both the next and previous visually equivalent positions are both non editable". The implementation now uses `||` (either side non-editable). The inline comment at the change site explains the new logic, but the header could be updated for consistency. This is cosmetic and does not affect correctness.

### Informational
1. The `layout_object->Style()->ShouldPreserveBreaks()` check uses `layout_object` (the loop's current layout object) rather than the text node's own layout object. In this code path, `current_node != start_node` and the layout object was obtained from the current node's renderer, so they are consistent. This is correct.
2. The fix in `MostBackwardCaretPosition` could theoretically affect other scenarios where a text node ends with a preserved newline and is entered from a parent position. The comprehensive test coverage (65 related unit tests + 427 web tests all passing) provides confidence that no regressions were introduced.

## Linter Results

```
$ git cl format
(no output - no changes needed)

$ git cl presubmit --force
Running presubmit commit checks on branch fix/409942757 ...
Presubmit checks took 5.5s to calculate.
presubmit checks passed.
```

**Status**: ✅ PASS

## Security Considerations
- Input validation: ✅ OK — No new user input parsing. The fix only changes how existing DOM positions are canonicalized during cursor movement.
- IPC safety: ✅ OK — No IPC changes. All changes are within the Blink rendering engine's editing module.
- Memory safety: ✅ OK — No new allocations, no ownership changes, no raw pointer hazards. `DynamicTo<Text>` is a safe cast pattern. Bounds checking is present for string access.

## Performance Considerations
- Hot path affected: Minimal — `MostBackwardCaretPosition` is called during cursor movement, which is a user-initiated action (not a tight loop). The added check is O(1): one `DynamicTo` cast, one string length check, one character comparison, and one style property check.
- New allocations: None — The fix uses references to existing objects only.
- Async considerations: None — Editing code is synchronous on the main thread.

## Overall Verdict

| Category | Status |
|----------|--------|
| Correctness | ✅ |
| Safety | ✅ |
| Style | ✅ |
| Tests | ✅ |
| Security | ✅ |
| Performance | ✅ |

**Ready for CL**: ✅ YES

## Actions Before CL
- [x] All checks pass
- [x] Code formatted (`git cl format` — no changes)
- [x] Tests updated (3 unit tests + 7 web test sub-tests)
- [x] Presubmit passes
- [ ] Ready for upload
