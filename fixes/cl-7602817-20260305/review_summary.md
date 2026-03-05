# CL Review Summary: Make DOMSelection not use VisibleSelection

**CL Number:** 7602817
**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7602817
**Author:** Rohan Raja (roraja@microsoft.com)
**Status:** NEW (Patch Set 1)
**Bug:** chromium:41311101
**Feature Flag:** `RemoveVisibleSelectionInDOMSelection`
**Files Changed:** 10 files, +335/−44 lines

---

## 1. Executive Summary

This CL refactors `DOMSelection` to eliminate its dependency on `VisibleSelection`, replacing the expensive `GetVisibleSelection()` method (which triggers `UpdateStyleAndLayout` on every read) with a lightweight `GetDOMSelection()` that returns raw `SelectionInDOMTree` positions directly. Position canonicalization (via `ParentAnchoredEquivalent()`) is moved from read-time in `DOMSelection` to write-time in `CorrectedSelectionAfterCommand()`, improving performance while fixing an incorrect anchor-snapping behavior in the WPT anchor-removal test. All changes are gated behind the `RemoveVisibleSelectionInDOMSelection` runtime feature flag for safe rollback.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|:------------:|----------|
| Clarity | 4 | Clear separation of read-time vs write-time canonicalization. Well-documented in commit message and code comments. |
| Maintainability | 4 | Reduces coupling between DOMSelection and layout system. Feature flag allows safe incremental rollout. |
| Extensibility | 4 | The raw-position approach aligns better with the Selection API spec, making future spec compliance easier. |
| Consistency | 3 | The caret-browsing canonicalization in `element_fragment_anchor.cc` is a separate special case that breaks the clean pattern. Could benefit from consolidation. |

**Overall Design Rating: 3.75/5**

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BEFORE (Current)                             │
│                                                                     │
│  JS API Read (e.g., anchorNode)                                     │
│       │                                                             │
│       ▼                                                             │
│  DOMSelection::GetVisibleSelection()                                │
│       │                                                             │
│       ├──► UpdateStyleAndLayout()  ◄── EXPENSIVE (every read!)      │
│       │                                                             │
│       └──► ComputeVisibleSelectionInDOMTree()                       │
│                │                                                    │
│                └──► Canonicalize positions (ParentAnchoredEquiv.)   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        AFTER (This CL)                              │
│                                                                     │
│  Editing Command Write Path                                         │
│       │                                                             │
│       ▼                                                             │
│  CorrectedSelectionAfterCommand()                                   │
│       │                                                             │
│       ├──► CreateVisibleSelection() + ParentAnchoredEquivalent()    │
│       │    (canonicalization at WRITE TIME — once per command)       │
│       │                                                             │
│       └──► Store canonical positions in FrameSelection              │
│                                                                     │
│  JS API Read (e.g., anchorNode)                                     │
│       │                                                             │
│       ▼                                                             │
│  DOMSelection::GetDOMSelection()                                    │
│       │                                                             │
│       └──► Selection().GetSelectionInDOMTree()                      │
│            (direct return — NO layout, NO canonicalization)          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|:------------:|----------|
| Correctness | 3 | Sound approach with good null-position guards, but **CL fails to compile** due to missing `CORE_EXPORT` on `CorrectedSelectionAfterCommand` and `SelectionForUndoStep` (see Critical Issues). |
| Efficiency | 5 | Eliminates `UpdateStyleAndLayout()` on every selection property read — a significant performance win. |
| Readability | 4 | Code is well-commented explaining why canonicalization moved. The `CreateRangeFromSelectionEditor` rewrite is clear. |
| Test Coverage | 4 | Good unit test additions (83 lines for utilities, 175 lines for DOMSelection). Missing some edge cases (see Test Coverage section). |

**Overall Implementation Rating: 4.0/5**

---

## 4. Key Findings

### Critical Issues (Must Fix)

1. **Build Failure: Undefined symbol `CorrectedSelectionAfterCommand`**
   - **File:** `editing_commands_utilities_test.cc`
   - **Details:** The test directly calls `CorrectedSelectionAfterCommand()` which is not exported with `CORE_EXPORT` in `editing_commands_utilities.h` (line 147). The test binary (`blink_unittests`) links against the core module as a shared library, requiring the symbol to be exported. This is confirmed by the CI failure on both macOS and Windows trybots.
   - **Fix:** Add `CORE_EXPORT` to `CorrectedSelectionAfterCommand` declaration in `editing_commands_utilities.h`.

2. **Build Failure: Undefined symbol `SelectionForUndoStep::From`**
   - **File:** `editing_commands_utilities_test.cc`
   - **Details:** The test uses `SelectionForUndoStep::From()` (line 125, 165) which is declared in `selection_for_undo_step.h` without `CORE_EXPORT` on the class. The class declaration at line 16 is `class SelectionForUndoStep final` — missing the export macro.
   - **Fix:** Add `CORE_EXPORT` to the `SelectionForUndoStep` class declaration, or restructure the test to avoid directly calling these symbols (e.g., use `EditingTestBase` helper methods).

### Major Issues (Should Fix)

1. **Incomplete feature flag coverage — other read paths may still assume canonicalized positions**
   - **Details:** While `DOMSelection` methods are updated, other code outside `DOMSelection` that calls `Selection().GetSelectionInDOMTree()` directly may still expect canonicalized positions after editing commands. A search for all callers of `GetSelectionInDOMTree()` should be performed to ensure no regressions.
   - **Recommendation:** Audit all `GetSelectionInDOMTree()` callers outside of `DOMSelection` to verify they don't depend on canonicalized positions at read time.

2. **`element_fragment_anchor.cc` — CreateVisiblePosition could return null DeepEquivalent**
   - **File:** `element_fragment_anchor.cc:194-197`
   - **Details:** `CreateVisiblePosition()` can return a null `VisiblePosition` if no valid position exists. The `.DeepEquivalent()` would then return a null `Position`, which is checked by `pos.IsConnected()` on line 200 — but `IsConnected()` on a null position may not behave as expected. An explicit null check would be more robust.
   - **Recommendation:** Add `if (pos.IsNull()) return;` after the `DeepEquivalent()` call.

3. **Dual `Change-Id` in commit message**
   - **Details:** The commit message contains two `Change-Id` lines (`Id7dfdcdbb88e22e18bcd35e3369ace4842637d79` and `Ic7e0351645d38e5c2f98a9e705017825403eb92e`). This may cause issues with Gerrit processing. The first one appears to be from a previous iteration and should be removed.

### Minor Issues (Nice to Fix)

1. **`CreateRangeFromSelectionEditor` — double canonicalization for document selections**
   - **File:** `dom_selection.cc:690-697`
   - **Details:** For document selections, `ComputeStartPosition()` and `ComputeEndPosition()` are called on the selection and then `ParentAnchoredEquivalent()` is applied. However, the anchor and focus were already canonicalized with `ParentAnchoredEquivalent()` earlier (line 680-681). For non-reversed selections, `ComputeStartPosition()` returns the anchor which was already canonicalized. This means `ParentAnchoredEquivalent()` is called twice on the same positions — it's idempotent, but wasteful.
   - **Recommendation:** Use the already-canonicalized anchor/focus to compute start/end ordering instead.

2. **Inconsistent bracing style**
   - **File:** `dom_selection.cc:189-191`
   - **Details:** Added braces `{` `}` around a single-statement `if` block for `IsCaret()`. While Chromium style allows both, the surrounding code uses brace-free single-line `if` statements. Be consistent within the function.

3. **Comment references removed code path**
   - **File:** `editing_commands_utilities.cc:666-671`
   - **Details:** The comment says "previously done at read time in FirstEphemeralRangeOf(VisibleSelection)" — this is accurate but could benefit from linking to the bug or the specific function that changed.

### Suggestions (Optional)

1. **Consider adding a performance test/benchmark**
   - The primary motivation is eliminating `UpdateStyleAndLayout` calls on selection reads. A microbenchmark showing the improvement would strengthen the CL.

2. **Consider adding a `DCHECK` in `GetDOMSelection()` for debug builds**
   - A `DCHECK` verifying the selection is in a valid state (e.g., anchor node is connected) could help catch regressions early in development builds.

3. **Consider documenting the write-time canonicalization contract**
   - Add a comment on `CorrectedSelectionAfterCommand` documenting that callers are expected to receive positions that have been through `ParentAnchoredEquivalent()`, so downstream code (e.g., `DOMSelection`) can safely use them without re-canonicalization.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test File | New Tests | Description |
|-----------|-----------|-------------|
| `editing_commands_utilities_test.cc` | +83 lines (2 tests) | Tests `CorrectedSelectionAfterCommand` applies `ParentAnchoredEquivalent()` for positions inside/after tables |
| `dom_selection_test.cc` | +175 lines (8 tests) | Tests `rangeCount`, `type`, `containsNode`, `deleteFromDocument`, `getRangeAt`, and caret-near-table behavior |
| `apply_block_element_command_test.cc` | Updated expectations | Caret position changes from inside-table to before-table |
| `insert_paragraph_separator_command_test.cc` | Updated expectations | Same caret position change pattern |
| `drag_and_drop_into_removed_on_focus.html` | Updated expectations | Caret position shifts from inside `<span>` to after `<span>` |
| WPT `anchor-removal-expected.txt` | Deleted | Test now passes (previously expected failure) |

### What's Missing

1. **Shadow DOM selection tests** — The `CreateRangeFromSelectionEditor` rewrite includes shadow DOM adjustment logic (`ShadowAdjustedNode`, `ShadowAdjustedOffset`) but no tests exercise this path with the new code.

2. **Detached node edge case tests** — The code guards against null positions from `ParentAnchoredEquivalent()` (referencing crbug.com/889737) but no test explicitly creates this condition.

3. **Multi-range selection tests** — `getComposedRanges()` behavior with the new path is not tested.

4. **Feature flag toggle tests** — No test verifies behavior with the feature flag *disabled* to ensure backward compatibility.

5. **Caret browsing path test** — The `element_fragment_anchor.cc` change adds canonicalization for caret browsing, but no test covers this specific path.

### Recommended Additional Tests

- Add a test in `dom_selection_test.cc` for selection across shadow DOM boundaries
- Add a test for `CreateRangeFromSelectionEditor` with a detached node (Position returns null from `ParentAnchoredEquivalent()`)
- Add a test with the feature flag disabled to verify no regression
- Add a web test for caret browsing with fragment anchors

---

## 6. Security Considerations

| Consideration | Risk | Assessment |
|---------------|------|------------|
| **Cross-origin information leak** | Low | `DOMSelection` already restricts access based on document origin. The change doesn't modify access control logic. |
| **Shadow DOM encapsulation** | Low-Medium | The `ShadowAdjustedNode/Offset` logic is preserved but rewritten. A shadow DOM boundary crossing bug could leak information. The logic appears correct but deserves focused testing. |
| **Null pointer dereference** | Low | Multiple null checks added (`anchor.IsNull()`, `focus.IsNull()`, `!anchor_node`, `!focus_node`). The guards appear comprehensive. |
| **Use-after-free** | Low | Positions reference nodes by pointer. Since `GetDOMSelection()` no longer forces layout, positions won't be invalidated by layout-triggered GC. This is actually *safer* than the old path. |

**Recommendation:** No significant security concerns. The change reduces attack surface by eliminating unnecessary layout updates that could interact with mutation observers or other re-entrant code paths.

---

## 7. Performance Considerations

| Aspect | Impact | Assessment |
|--------|--------|------------|
| **Layout elimination on reads** | **High positive** | Eliminates `UpdateStyleAndLayout(DocumentUpdateReason::kSelection)` from every selection property access (`anchorNode`, `anchorOffset`, `focusNode`, `focusOffset`, `type`, `rangeCount`, `direction`, `containsNode`, `deleteFromDocument`). This is the primary win. |
| **Write-time canonicalization cost** | Low negative | `ParentAnchoredEquivalent()` is called on anchor + focus in `CorrectedSelectionAfterCommand()`. This is a tree walk that's O(depth) — typically trivial. |
| **Double canonicalization in `CreateRangeFromSelectionEditor`** | Low negative | For document selections, `ParentAnchoredEquivalent()` is called 4 times (anchor, focus, start, end) instead of the minimum 2. Negligible impact. |
| **Caret browsing path** | Neutral | `CreateVisiblePosition().DeepEquivalent()` adds a visible position computation, but this path is rarely exercised. |

### Benchmarking Recommendations

1. **Selection API microbenchmark** — Measure `window.getSelection().anchorNode` access latency before/after on a complex page with dirty layout.
2. **Editing command throughput** — Measure typing speed in a `contenteditable` with a large document to ensure write-time canonicalization doesn't add noticeable latency.
3. **Selection-heavy web apps** — Test on real-world editors (Google Docs, VS Code web) with the feature flag enabled.

---

## 8. Final Recommendation

**Verdict: NEEDS_WORK**

**Rationale:**
The design is sound and the performance motivation is strong. Moving canonicalization from read-time to write-time is the correct architectural direction, and the implementation is largely well-executed with good test coverage. However, the CL currently **fails to compile** due to missing `CORE_EXPORT` annotations on `CorrectedSelectionAfterCommand` and `SelectionForUndoStep`, which is a blocking issue confirmed by CI on both macOS and Windows. Additionally, the `element_fragment_anchor.cc` change could benefit from a null-position safety check, and the dual `Change-Id` in the commit message needs cleanup.

**Action Items for Author:**

1. **[MUST FIX]** Add `CORE_EXPORT` to `CorrectedSelectionAfterCommand` declaration in `editing_commands_utilities.h` and to the `SelectionForUndoStep` class declaration in `selection_for_undo_step.h` to fix the linker errors.
2. **[MUST FIX]** Remove the duplicate `Change-Id` from the commit message.
3. **[SHOULD FIX]** Add an explicit null check after `CreateVisiblePosition().DeepEquivalent()` in `element_fragment_anchor.cc:196`.
4. **[SHOULD FIX]** Audit other callers of `GetSelectionInDOMTree()` outside `DOMSelection` to ensure they don't depend on canonicalized positions.
5. **[NICE TO FIX]** Eliminate double `ParentAnchoredEquivalent()` calls in `CreateRangeFromSelectionEditor` for document selections.
6. **[NICE TO FIX]** Add shadow DOM and detached-node edge case tests.

---

## 9. Comments for Gerrit

### Comment 1 — Build Failure (editing_commands_utilities.h)
**File:** `third_party/blink/renderer/core/editing/commands/editing_commands_utilities.h`, line 147
**Severity:** Critical

> The trybots are failing with `undefined symbol: CorrectedSelectionAfterCommand`. This function needs `CORE_EXPORT` since it's called from the test binary which links against core as a shared library:
> ```cpp
> CORE_EXPORT SelectionInDOMTree CorrectedSelectionAfterCommand(const SelectionForUndoStep&, Document*);
> ```
> Similarly, `SelectionForUndoStep` in `selection_for_undo_step.h` needs `CORE_EXPORT` on the class declaration since `SelectionForUndoStep::From()` is used in the test.

### Comment 2 — Null Safety (element_fragment_anchor.cc)
**File:** `third_party/blink/renderer/core/page/scrolling/element_fragment_anchor.cc`, line 194-197
**Severity:** Major

> `CreateVisiblePosition()` can return a null `VisiblePosition` (e.g., if the node is not in a valid rendering context). The subsequent `.DeepEquivalent()` would return a null `Position`. While `pos.IsConnected()` on line 200 may catch this, it would be more robust to add an explicit null check:
> ```cpp
> pos = CreateVisiblePosition(
>           Position::FirstPositionInOrBeforeNode(*anchor_node_))
>           .DeepEquivalent();
> if (pos.IsNull())
>   return;
> ```

### Comment 3 — Commit Message Cleanup
**Severity:** Minor

> The commit message has two `Change-Id` lines. Please remove the first one (`Id7dfdcdbb88e22e18bcd35e3369ace4842637d79`) — Gerrit will use the second one.

### Comment 4 — Double Canonicalization (dom_selection.cc)
**File:** `third_party/blink/renderer/core/editing/dom_selection.cc`, lines 690-697
**Severity:** Nit

> For document selections, `ComputeStartPosition().ParentAnchoredEquivalent()` is called, but the anchor and focus were already canonicalized on lines 680-681. Since `ComputeStartPosition()` returns whichever of anchor/focus comes first, and both are already canonical, the second `ParentAnchoredEquivalent()` is redundant. Consider reusing the already-canonicalized values:
> ```cpp
> const Position start = anchor <= focus ? anchor : focus;
> const Position end = anchor <= focus ? focus : anchor;
> ```

### Comment 5 — Test Coverage
**Severity:** Minor

> Nice test additions! Consider also adding:
> - A test for selection across shadow DOM boundaries through `CreateRangeFromSelectionEditor`
> - A test verifying behavior with the feature flag disabled (backward compat)
> - A test for the `element_fragment_anchor.cc` caret browsing path
