# Fix Implementation: 41311101

## Summary
Gated all 7 `ComputeVisibleSelectionInDOMTree()` call sites in `DOMSelection` behind the existing `RemoveVisibleSelectionInDOMSelection` runtime feature flag. When the flag is enabled, `DOMSelection` reads directly from `SelectionInDOMTree` (the DOM-level source of truth) via `GetSelectionInDOMTree()` instead of going through `VisibleSelection` canonicalization, which incorrectly depends on layout tree traversal under LayoutNG. For `IsNone()` checks (`direction()`, `rangeCount()`), the replacement is `GetSelectionInDOMTree().IsNone()`. For normalized range computation (`deleteFromDocument()`, `containsNode()`, `toString()`), the replacement is `NormalizeRange(GetSelectionInDOMTree())`. For `CreateRangeFromSelectionEditor()`, the entire method body is duplicated under the flag check using `SelectionInDOMTree` APIs instead of `VisibleSelection`.

## Branch
- **Branch name**: `fix/41311101`
- **Base commit**: latest_main (95da2fe818f27)

## Files Modified

### 1. [/third_party/blink/renderer/core/editing/dom_selection.cc](/third_party/blink/renderer/core/editing/dom_selection.cc)

**Lines changed**: 6 locations across the file

#### Change 1: `direction()` — line 214

**Before**:
```cpp
      Selection().ComputeVisibleSelectionInDOMTree().IsNone()) {
```

**After**:
```cpp
      (RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
           ? Selection().GetSelectionInDOMTree().IsNone()
           : Selection().ComputeVisibleSelectionInDOMTree().IsNone())) {
```

**Rationale**: When the flag is enabled, check `IsNone()` directly on the DOM selection state without layout-dependent canonicalization that incorrectly returns `IsNone()==true` for `display:none` content under LayoutNG.

#### Change 2: `rangeCount()` — lines 231-241

**Before**:
```cpp
  DomWindow()->document()->UpdateStyleAndLayout(
      DocumentUpdateReason::kSelection);

  if (Selection().ComputeVisibleSelectionInDOMTree().IsNone()) {
    return 0;
  }
```

**After**:
```cpp
  if (RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()) {
    if (Selection().GetSelectionInDOMTree().IsNone()) {
      return 0;
    }
  } else {
    DomWindow()->document()->UpdateStyleAndLayout(
        DocumentUpdateReason::kSelection);

    if (Selection().ComputeVisibleSelectionInDOMTree().IsNone()) {
      return 0;
    }
  }
```

**Rationale**: When the flag is enabled, no `UpdateStyleAndLayout()` is needed because `GetSelectionInDOMTree()` doesn't depend on layout. The `IsNone()` check reads directly from the DOM selection state.

#### Change 3: `CreateRangeFromSelectionEditor()` — lines 689-711

**Before**:
```cpp
EphemeralRange DOMSelection::CreateRangeFromSelectionEditor() const {
  const VisibleSelection& selection = GetVisibleSelection();
  // ... uses selection.Anchor(), selection.Focus(), selection.IsAnchorFirst()
}
```

**After**:
```cpp
EphemeralRange DOMSelection::CreateRangeFromSelectionEditor() const {
  if (RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()) {
    const SelectionInDOMTree& selection = Selection().GetSelectionInDOMTree();
    if (selection.IsNone())
      return EphemeralRange();
    // ... uses selection.Anchor(), selection.Focus(), selection.IsAnchorFirst()
    // ... NormalizeRange(selection) replaces FirstEphemeralRangeOf(VisibleSelection)
  }
  // ... original code as fallback
}
```

**Rationale**: Reads `SelectionInDOMTree` directly; `FirstEphemeralRangeOf(VisibleSelection)` replaced with `NormalizeRange(SelectionInDOMTree)` since both produce a normalized ephemeral range. Shadow adjustment logic preserved identically.

#### Change 4: `deleteFromDocument()` — lines 820-825

**Before**:
```cpp
  Range* selected_range = CreateRange(Selection()
                                          .ComputeVisibleSelectionInDOMTree()
                                          .ToNormalizedEphemeralRange());
```

**After**:
```cpp
  Range* selected_range = CreateRange(
      RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
          ? NormalizeRange(Selection().GetSelectionInDOMTree())
          : Selection()
                .ComputeVisibleSelectionInDOMTree()
                .ToNormalizedEphemeralRange());
```

**Rationale**: `VisibleSelection::ToNormalizedEphemeralRange()` internally calls `NormalizeRange(AsSelection())`. The replacement `NormalizeRange(GetSelectionInDOMTree())` does the same normalization on the already-canonicalized-at-source selection.

#### Change 5: `containsNode()` — lines 853-858

**Before**:
```cpp
  const EphemeralRange selected_range = Selection()
                                            .ComputeVisibleSelectionInDOMTree()
                                            .ToNormalizedEphemeralRange();
```

**After**:
```cpp
  const EphemeralRange selected_range =
      RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
          ? NormalizeRange(Selection().GetSelectionInDOMTree())
          : Selection()
                .ComputeVisibleSelectionInDOMTree()
                .ToNormalizedEphemeralRange();
```

**Rationale**: Same pattern as `deleteFromDocument()`.

#### Change 6: `toString()` — lines 919-924

**Before**:
```cpp
  const EphemeralRange range = Selection()
                                   .ComputeVisibleSelectionInDOMTree()
                                   .ToNormalizedEphemeralRange();
```

**After**:
```cpp
  const EphemeralRange range =
      RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()
          ? NormalizeRange(Selection().GetSelectionInDOMTree())
          : Selection()
                .ComputeVisibleSelectionInDOMTree()
                .ToNormalizedEphemeralRange();
```

**Rationale**: Same pattern as `deleteFromDocument()` and `containsNode()`.

## Git Diff Summary
```
$ git diff --stat
 third_party/blink/renderer/core/editing/dom_selection.cc      |  75 +++++++++++---
 third_party/blink/renderer/core/editing/dom_selection_test.cc | 266 ++++++++++++++++++++++++++++++++++++++++++++++++++
 2 files changed, 325 insertions(+), 16 deletions(-)
```

Note: `dom_selection_test.cc` changes (266 lines) are from Stage 5 (TDD test creation), not this stage.

## Build Result
```
$ siso ninja -C out/release_x64 blink_unittests --offline
[0/11835] 5.10s S CXX obj/third_party/blink/renderer/core/core/dom_selection.o
[1/183] 13.98s S AR obj/third_party/blink/renderer/core/libblink_core.a
...
33.26s Build Succeeded: 5 steps - 0.15/s
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result
```
$ out/release_x64/blink_unittests --gtest_filter="*DOMSelection*41311101*" --single-process-tests
[==========] Running 13 tests from 1 test suite.
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountWithFlag (42 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionForwardWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DirectionForwardWithFlag (30 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionNoneWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DirectionNoneWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ToStringAcrossInlineBoundaryWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ContainsNodeFullWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ContainsNodePartialWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DeleteFromDocumentWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag
[       OK ] DOMSelectionTest.Bug41311101_GetRangeAtAcrossInlineBoundaryWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountInShadowDOMWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountEmptyWithFlag (32 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag
[       OK ] DOMSelectionTest.Bug41311101_ToStringCollapsedWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag
[       OK ] DOMSelectionTest.Bug41311101_RangeCountDisplayNoneWithFlag (31 ms)
[ RUN      ] DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag
[       OK ] DOMSelectionTest.Bug41311101_DirectionDisplayNoneWithFlag (32 ms)
[==========] 13 tests from 1 test suite ran. (437 ms total)
[  PASSED  ] 13 tests.
```

**Status**: ✅ ALL 13 TESTS PASS (including 2 previously failing TDD tests)

## Implementation Notes
- All changes are gated behind `RuntimeEnabledFeatures::RemoveVisibleSelectionInDOMSelectionEnabled()` — when the flag is disabled, behavior is exactly as before
- The `GetVisibleSelection()` private helper method is retained for the fallback path (used by `CreateRangeFromSelectionEditor()` when flag is disabled)
- `UpdateStyleAndLayout()` calls are retained where `NormalizeRange()` is used (it internally calls `MostBackwardCaretPosition()` which needs layout), but skipped in `rangeCount()` when using direct `GetSelectionInDOMTree().IsNone()` check
- The fix follows the exact design from `04-architecture-lld.md` with no deviations

## Known Issues
- None discovered during implementation

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
