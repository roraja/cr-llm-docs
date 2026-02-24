# Code Review: 41194596

## Review Summary
The fix correctly addresses Ctrl+Up/Down paragraph navigation in textareas on Windows by snapping caret positions to paragraph boundaries after `NextParagraphPosition()`/`PreviousParagraphPosition()` return an x-offset-preserved position in a different paragraph. The backward case implements a two-step approach (first to start of current paragraph, then to start of previous) matching native Windows behavior (e.g., WordPad). The code is safe, follows existing patterns, and is well-tested with both unit and web tests. No critical issues found.

## Checklist Results

### Correctness ✅
- [x] Fix addresses root cause — the root cause is that `NextParagraphPosition()`/`PreviousParagraphPosition()` preserve x-offset across paragraph boundaries; the fix snaps to `StartOfParagraph()` when crossing a boundary
- [x] No logic errors — `InSameParagraph()` guard ensures snapping only occurs when actually crossing a paragraph boundary; null check prevents crash at document boundaries
- [x] Edge cases handled — document start/end (pos is null or same paragraph), already at paragraph start (backward case), non-editable islands within contenteditable
- [x] Error handling correct — no new error paths introduced; existing DCHECK patterns preserved

### Crash Safety ✅
- [x] No null dereferences — `pos.IsNotNull()` checked before passing to `InSameParagraph()` which DCHECKs validity; `current` comes from `EndForPlatform()`/`StartForPlatform()` which are always valid in this context
- [x] No use-after-free — all positions are stack-local `VisiblePositionInFlatTree` values (value types, not pointers)
- [x] No invalid iterators — no iterator usage introduced
- [x] Bounds checked — N/A, no array access

### Memory Safety ✅
- [x] Smart pointers used correctly — N/A, no heap allocations introduced
- [x] No memory leaks — all values are stack-local
- [x] Object lifetimes correct — `VisiblePositionInFlatTree` values have automatic storage duration
- [x] Raw pointers safe — no raw pointers introduced

### Thread Safety ✅
- [x] Thread annotations present — N/A, all code runs on the main thread (editing code)
- [x] No deadlocks — no locks introduced
- [x] No race conditions — single-threaded editing path
- [x] Cross-thread calls safe — N/A

### DCHECK Safety ✅
- [x] DCHECKs valid — existing `InSameParagraph()` DCHECKs on position validity are protected by `IsNotNull()` guard
- [x] Good error messages — existing DCHECK pattern preserved (prints position on failure)
- [x] No DCHECK on user input — all DCHECKs are on internal state validity

### Code Style ✅
- [x] Follows style guide — uses scoped `case` blocks with `{}`, `const auto` for enum values, `const` references matching surrounding code
- [x] Formatted with git cl format — confirmed, no changes needed
- [x] Good naming — `boundary_rule`, `first_step_rule`, `para_start`, `current` are descriptive and consistent with surrounding code
- [x] Helpful comments — comments explain "why" (Windows behavior, crbug reference, kCanSkipOverEditingBoundary rationale)

### Tests ✅
- [x] Bug scenario covered — `MoveParagraphForwardWrappingText` directly tests the reported bug (wrapping text in contenteditable with forward paragraph movement)
- [x] Edge cases covered — backward from mid-paragraph, backward from paragraph start, web tests for non-editable crossing
- [x] Descriptive names — test names clearly describe the scenario being tested
- [x] Not flaky — tests use Ahem font with fixed dimensions, deterministic layout

## Detailed Review

### File: selection_modifier.cc

**Lines 507-523 (Forward `kParagraph` case)**: ✅ Correct
- Retrieves `boundary_rule` from `ModifyParagraphCrossEditingoundaryEnabled()` feature flag — matches the same pattern used in `NextParagraphPosition()` at line 100-102 and `PreviousParagraphPosition()` at line 120-122.
- Calls `NextParagraphPosition()` then conditionally snaps to `StartOfParagraph()`.
- The `InSameParagraph()` guard correctly prevents snapping when at document end (position couldn't advance to a new paragraph).
- The `pos.IsNotNull()` check prevents passing null to `InSameParagraph()` which DCHECKs validity.

**Lines 713-745 (Backward `kParagraph` case)**: ✅ Correct
- Two-step approach: first try `StartOfParagraph(current)`, and only if already at paragraph start, fall back to `PreviousParagraphPosition()` + snap.
- The `first_step_rule` uses `kCanSkipOverEditingBoundary` when editable, matching the `kParagraphBoundary` case at lines 753-760. This correctly handles non-editable islands within contenteditable regions.
- Uses `DeepEquivalent()` comparison (not `==`) to check if already at paragraph start — this is the correct comparison for `VisiblePositionInFlatTree` since `==` may not do what we want with different affinities.

**⚠️ Note**: The `first_step_rule` respects `MoveToParagraphStartOrEndSkipsNonEditableEnabled()` feature flag but the second step (after `PreviousParagraphPosition()`) uses `boundary_rule` from `ModifyParagraphCrossEditingoundaryEnabled()`. This is intentional — the first step (paragraph boundary) and second step (paragraph crossing) use different feature flags, matching the existing architectural separation between "move to paragraph boundary" and "cross paragraph boundary" behaviors.

### File: selection_modifier_test.cc

**Helper methods (lines 34-45)**: ✅ Clean
- `MoveForwardByParagraph()` and `MoveBackwardByParagraph()` follow the exact same pattern as the existing `MoveForwardByLine()` helper at line 28.

**Test `MoveParagraphForwardWrappingText` (lines 480-496)**: ✅ Good
- Tests the exact bug scenario: wrapping text (50px wide with Ahem 10px, so 5 chars per line) with Ctrl+Down.
- Cursor at `AAA|AA AAAAA` (mid first paragraph), forward paragraph should land at `|BBBBB BBBBB` (start of second paragraph).

**Test `MoveParagraphBackwardFromMidParagraph` (lines 500-515)**: ✅ Good
- Verifies backward from mid-paragraph goes to start of current paragraph (not previous).

**Test `MoveParagraphBackwardFromParagraphStart` (lines 519-534)**: ✅ Good
- Verifies backward from paragraph start goes to start of previous paragraph.

### File: move-by-paragraph-wrapping-text.html (new)

**All 30 lines**: ✅ Good
- New web test file covering the three bug scenarios using `assert_selection()`.
- Uses Ahem font for deterministic layout, matching test infrastructure conventions.
- Properly includes testharness.js and assert_selection.js.

### File: move-by-paragraph.html (existing, 3 expectation changes)

**Line 10 change**: ✅ Correct
- `foo|<div>bar</div>` → `foo<div>|bar</div>` — backward from `bar|` now correctly goes to `|bar` (start of current paragraph = start of the `<div>`) instead of `foo|` (which was a position in the previous paragraph).

**Lines 27-28 change**: ✅ Correct
- Backward from a position after non-editable content now goes to start of current paragraph, not previous.

**Line 88 change**: ✅ Correct
- Forward across non-editable now goes to `|P2` (start of next paragraph) instead of `P2|` (end of it).

## Issues Found

### Critical (Must Fix)
None.

### Minor (Should Consider)
None.

### Informational
- The `ModifyParagraphCrossEditingoundaryEnabled` feature flag name has a typo ("oundary" instead of "Boundary") — this is a pre-existing issue in the codebase, not introduced by this change.

## Linter Results

```
$ git cl format
(no output — no changes needed)

$ git cl presubmit --force
** Presubmit Messages: 1 **
If this change has an associated bug, add Bug: [bug number] or Fixed: [bug number].

Presubmit checks took 6.6s to calculate.
presubmit checks passed.
```

**Status**: ✅ PASS (one informational message about adding Bug: line to commit message — will be added when creating CL)

## Security Considerations
- Input validation: ✅ OK — no user input parsing; operates on internal DOM positions
- IPC safety: ✅ OK — no IPC involved; all changes are within Blink renderer editing code
- Memory safety: ✅ OK — all values are stack-local; no heap allocations; no raw pointer manipulation

## Performance Considerations
- Hot path affected: Minimal — paragraph navigation (Ctrl+Up/Down) is user-initiated and infrequent
- New allocations: `StartOfParagraph()` creates `VisiblePositionInFlatTree` on the stack — negligible cost; this same function is already called in `InSameParagraph()`
- Async considerations: N/A — all synchronous, single-threaded editing code
- The backward case adds one extra `StartOfParagraph()` call compared to the old code, but this is offset by potentially avoiding the more expensive `PreviousParagraphPosition()` call when the caret is mid-paragraph

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
- [x] Code formatted
- [x] Tests updated
- [x] Presubmit passes
- [ ] Ready for upload
