# Review Summary: CL 7564913 — [Fonts] Fix canvas measureText() precision at small font sizes

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7564913
**Author:** Sejal Anand <sejalanand@microsoft.com>
**Bug:** [crbug.com/479240778](https://crbug.com/479240778)
**Files Changed:** 6 (+127/-13 lines)

---

## 1. Executive Summary

This CL fixes a precision bug in the HTML Canvas `measureText()` API where `actualBoundingBox*` metrics return inaccurate values at small font sizes (e.g., 1.5px). The root cause is that Skia's `SkGlyph` stores glyph bounds as integers (`int16_t`/`uint16_t`), causing large relative errors when font sizes are small (e.g., 50%+ error for a 1.3px glyph reported as 1px or 2px). The fix introduces a new `PreciseBoundsForGlyph` method that computes glyph bounds from float-precision path outlines via `SkFont::getPath()` instead of relying on `SkFont::getBounds()`, and applies it only to the Canvas `TextMetrics` measurement path.

---

## 2. Design Assessment

### Architecture Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Clarity | 5 | Clean separation: new `PreciseBoundsForGlyph` path is parallel to existing `BoundsForGlyph`, with clear naming that communicates intent. Well-commented commit message and inline documentation. |
| Maintainability | 4 | Good refactoring — extracted shared `GetPathBoundsForGlyph` helper from Apple-specific code. Minor concern: subpixel-rounding logic is duplicated between the helper and `SkFontGetBoundsForGlyph`. |
| Extensibility | 4 | New API is additive; existing callers are unaffected. However, the `PreciseBoundsForGlyph` method could benefit from a comment limiting its intended scope to canvas. |
| Consistency | 4 | Follows existing patterns (`PlatformBoundsForGlyph` → `SkFontGetBoundsForGlyph`). Both call sites in `TextMetrics` use the new precise path consistently. |

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    JavaScript: ctx.measureText("A")              │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│              Blink Core — HTML Canvas                             │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ TextMetrics::GetRunsAndGlyphBounds()                    │     │
│  │   └─ ForEachGlyph → PreciseBoundsForGlyph() ✨ NEW     │     │
│  │ TextMetrics::getActualBoundingBox()                     │     │
│  │   └─ ForEachGlyph → PreciseBoundsForGlyph() ✨ NEW     │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│              Blink Platform — Fonts                               │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ SimpleFontData                                          │     │
│  │   ├─ BoundsForGlyph()        (existing, unchanged)     │     │
│  │   └─ PreciseBoundsForGlyph() ✨ NEW                    │     │
│  └──────────────────────┬──────────────────────────────────┘     │
│                         │                                        │
│  ┌──────────────────────▼──────────────────────────────────┐     │
│  │ skia_text_metrics                                       │     │
│  │   ├─ SkFontGetBoundsForGlyph()        (refactored)     │     │
│  │   ├─ SkFontGetPreciseBoundsForGlyph() ✨ NEW           │     │
│  │   └─ GetPathBoundsForGlyph()          ✨ NEW (static)  │     │
│  └──────────────────────┬──────────────────────────────────┘     │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│              Skia (Third-party)                                   │
│  ┌──────────────────────────────────┐                            │
│  │ SkFont::getPath() → SkPath::getBounds()  (float precision)   │
│  │ SkFont::getBounds()              (integer fallback for emoji) │
│  └──────────────────────────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Assessment

### Code Quality

| Aspect | Rating (1-5) | Comments |
|--------|--------------|----------|
| Correctness | 4 | Core approach is sound. One concern: `GetPathBoundsForGlyph` applies `roundOut` for non-subpixel fonts, which may negate precision gains in that configuration. In practice, canvas typically uses subpixel positioning, so the fix works. |
| Efficiency | 3 | Per-glyph `SkFont::getPath()` replaces precomputed per-run `InkBounds()` — this is O(n) in glyphs vs O(1), with path extraction per glyph. No caching is added. Acceptable for typical short `measureText()` calls but could regress on text-heavy canvas workloads. |
| Readability | 5 | Clear function naming (`PreciseBoundsForGlyph`), good inline comments explaining why path-based bounds are used, clean lambda callbacks. |
| Test Coverage | 3 | Single WPT visual integration test validates the fix scenario. No unit tests for the new functions; no edge-case tests for bitmap glyphs, zero-size fonts, or the `getActualBoundingBox()` path. |

---

## 4. Key Findings

### Critical Issues (Must Fix)

_None identified._ The CL is functionally correct for its primary use case.

### Major Issues (Should Fix)

- **Issue 1: `GetPathBoundsForGlyph` rounds precise bounds to integers when `!isSubpixel()`**
  - **File:** `skia_text_metrics.cc`, lines 141-145
  - **Impact:** The function named "Precise" may still return integer-quantized bounds when subpixel rendering is disabled. This is because the `roundOut` logic is in the shared helper rather than only in `SkFontGetBoundsForGlyph`.
  - **Recommendation:** Move the `roundOut` logic out of `GetPathBoundsForGlyph` and into `SkFontGetBoundsForGlyph` only. Let `SkFontGetPreciseBoundsForGlyph` return raw float-precision path bounds unconditionally. This makes the "Precise" name truthful regardless of subpixel state:
    ```cpp
    static void GetPathBoundsForGlyph(const SkFont& font,
                                      Glyph glyph,
                                      SkRect* bounds) {
      if (const auto path = font.getPath(glyph)) {
        *bounds = path->getBounds();
      } else {
        *bounds = font.getBounds(glyph, nullptr);
      }
      // No rounding — caller decides.
    }
    ```

- **Issue 2: Performance regression — per-glyph `getPath()` replaces precomputed `InkBounds()`**
  - **File:** `text_metrics.cc`, lines 195-206
  - **Impact:** `GetRunsAndGlyphBounds()` now extracts glyph outlines and computes path bounds per glyph. For text-heavy canvas workloads (data visualizations, games), this could be a measurable regression.
  - **Recommendation:** (a) Add a per-glyph bounds cache in `SimpleFontData` keyed by glyph ID, or (b) run a targeted performance benchmark (`measureText()` in a tight loop on long strings) and document acceptable results, or (c) add a TODO with a tracking bug for future caching.

### Minor Issues (Nice to Fix)

- **Issue 1: Subpixel-rounding code duplication**
  - **File:** `skia_text_metrics.cc`, lines 141-145 and 156-160
  - The `!isSubpixel()` → `roundOut` pattern appears in both `GetPathBoundsForGlyph` and the `#else` branch of `SkFontGetBoundsForGlyph`. Addressing Major Issue #1 would eliminate this duplication.

- **Issue 2: Missing documentation on `PreciseBoundsForGlyph` intended scope**
  - **File:** `simple_font_data.h`
  - The new public method should have a comment explaining its distinction from `BoundsForGlyph` and its intended use case (canvas `measureText()` only), to prevent accidental adoption by general layout code.

### Suggestions (Optional)

- **Suggestion 1:** Switch the WPT test from yellow text to green text for WPT convention (green=pass, red=fail), or at minimum add a comment explaining the yellow choice relates to the original bug repro.

- **Suggestion 2:** Add a C++ unit test in `skia_text_metrics_test.cc` or `simple_font_data_test.cc` verifying that `PreciseBoundsForGlyph` returns non-integer bounds for a known glyph at a small font size, and testing the bitmap-only fallback path.

- **Suggestion 3:** Consider whether `SkFontGetBoundsForGlyph` on non-Apple platforms could also delegate to `GetPathBoundsForGlyph` (as Apple already does), to further reduce code duplication. The author may have intentionally avoided this for performance reasons in the general layout path.

---

## 5. Test Coverage Analysis

### Existing Tests

| Test | Type | What It Validates |
|------|------|-------------------|
| `2d.text.measure.actualBoundingBox.small-font.html` | WPT (visual integration) | Uses 1.5px CanvasTest font, measures "A", centers it using `actualBoundingBox` metrics, scales up 32×, asserts center pixel is yellow (text) not red (background). Clever: tests the *consequence* of precision rather than exact float values, making it robust across platforms. |

### Missing Tests

1. **Unit test for `SkFontGetPreciseBoundsForGlyph`** — verify float-precision bounds at small font sizes vs integer bounds.
2. **Bitmap-only glyph fallback test** — verify emoji/color glyphs hit the `getBounds()` fallback path without crashing and return reasonable bounds.
3. **Zero-size font edge case test** — verify `PreciseBoundsForGlyph` returns empty rect when font size is 0.
4. **`getActualBoundingBox()` path test** — the second call site (line 373 in `text_metrics.cc`) is not directly exercised by the WPT test, which tests via `Update()` → `GetRunsAndGlyphBounds()`.
5. **Multiple characters / RTL text test** — verify union of per-glyph bounds works correctly for multi-glyph strings.

### Recommended Additional Tests

- A `blink_unittests` C++ test comparing `PreciseBoundsForGlyph` vs `BoundsForGlyph` at a small font size (e.g., 1.5px) to verify the former has non-integer precision.
- A WPT test with a bitmap-only font (or emoji) to exercise the fallback path.

---

## 6. Security Considerations

**No security concerns identified.**

- All code runs within the renderer process on the main thread.
- No new IPC/Mojo interactions are introduced.
- `SkFont::getPath()` is a well-established Skia API; no new attack surface.
- Input validation is preserved (zero-size font guard, static_assert on glyph size).
- No user-controlled data flows into new unsafe operations.

---

## 7. Performance Considerations

### Analysis

| Aspect | Before | After | Impact |
|--------|--------|-------|--------|
| Bounds source in `GetRunsAndGlyphBounds` | Per-run `item.InkBounds()` (precomputed, O(1) per run) | Per-glyph `PreciseBoundsForGlyph()` via `getPath()` (O(n) per run) | **Regression risk for long strings** |
| `SkFont::getPath()` cost | Not called in this path | Called per glyph (path extraction + bounds computation) | Moderate cost per call |
| Caching | N/A | None added | Repeated glyphs (e.g., "aaaa") re-extract paths |

### Risk Assessment

- **Low risk** for typical `measureText()` usage (short strings, few glyphs).
- **Medium risk** for text-heavy canvas workloads (data visualization with thousands of labels calling `measureText()` in tight loops).
- The same `getPath()` approach is already used on Apple platforms in `SkFontGetBoundsForGlyph`, so it's a proven technique.

### Benchmarking Recommendations

1. Run the existing canvas text performance benchmarks (if any) before/after this CL.
2. Create a micro-benchmark: `measureText()` called 10,000 times on strings of varying lengths (1, 10, 100 characters).
3. Profile `getPath()` vs `getBounds()` cost per glyph at various font sizes.
4. If regression is confirmed, add a per-glyph precise bounds cache in `SimpleFontData`.

---

## 8. Final Recommendation

**Verdict**: **APPROVED_WITH_COMMENTS**

**Rationale:**

The CL correctly addresses a real user-facing bug where Canvas `measureText()` returns imprecise bounding box values at small font sizes. The approach is sound — using `SkFont::getPath()` for float-precision bounds is a proven technique (already used on Apple platforms) and the fix is well-scoped to only affect Canvas `TextMetrics`, minimizing blast radius.

The code is well-structured with clean refactoring (shared `GetPathBoundsForGlyph` helper), clear naming, and good inline comments. The WPT test cleverly validates the fix by testing the *consequence* of precision (text centering) rather than fragile exact values.

Two issues warrant attention before landing:

1. **The subpixel-rounding in `GetPathBoundsForGlyph`** should be moved to `SkFontGetBoundsForGlyph` only, so that the "Precise" variant always returns true float-precision bounds. This is a correctness/clarity concern — the current code works in practice (canvas uses subpixel rendering) but the design is misleading.

2. **The performance impact** of per-glyph `getPath()` without caching should be evaluated. At minimum, a TODO with a tracking bug for adding a cache would be appropriate. For typical `measureText()` usage the impact is negligible, but data-viz workloads could be affected.

The remaining issues (documentation, test coverage, code duplication) are minor and can be addressed incrementally.

**Action Items for Author:**

1. **[Should Fix]** Move `roundOut` logic from `GetPathBoundsForGlyph` into `SkFontGetBoundsForGlyph` only, so `SkFontGetPreciseBoundsForGlyph` always returns raw float bounds.
2. **[Should Fix]** Evaluate performance impact of per-glyph `getPath()` on text-heavy canvas workloads. Consider adding a per-glyph bounds cache, or at minimum a TODO with tracking bug.
3. **[Nice to Fix]** Add a comment on `PreciseBoundsForGlyph` declaration in `simple_font_data.h` explaining its distinction from `BoundsForGlyph` and intended scope.
4. **[Nice to Fix]** Consider adding a C++ unit test for `PreciseBoundsForGlyph` verifying float-precision bounds at small font sizes.

---

## 9. Comments for Gerrit

### Comment 1 — `skia_text_metrics.cc` (lines 132-146): Subpixel rounding in shared helper

> **File:** `third_party/blink/renderer/platform/fonts/skia/skia_text_metrics.cc`
> **Line:** 141
>
> The `roundOut` logic in `GetPathBoundsForGlyph` means `SkFontGetPreciseBoundsForGlyph` will still return integer-quantized bounds when `!font.isSubpixel()`. This makes the "Precise" name misleading in non-subpixel configurations.
>
> Consider moving the rounding into `SkFontGetBoundsForGlyph` only, and having `GetPathBoundsForGlyph` return raw path bounds. Then `SkFontGetPreciseBoundsForGlyph` would be truly precise regardless of subpixel state:
>
> ```cpp
> static void GetPathBoundsForGlyph(const SkFont& font,
>                                   Glyph glyph,
>                                   SkRect* bounds) {
>   if (const auto path = font.getPath(glyph)) {
>     *bounds = path->getBounds();
>   } else {
>     *bounds = font.getBounds(glyph, nullptr);
>   }
> }
> ```

### Comment 2 — `text_metrics.cc` (lines 195-206): Performance consideration

> **File:** `third_party/blink/renderer/core/html/canvas/text_metrics.cc`
> **Line:** 195
>
> This changes from using a precomputed per-run `InkBounds()` to per-glyph `PreciseBoundsForGlyph()` with `getPath()` extraction. For typical short `measureText()` calls this should be fine, but for text-heavy canvas workloads (e.g., data visualization calling `measureText()` thousands of times) this could be a measurable regression.
>
> Have you benchmarked this? If not, could you add a TODO with a tracking bug for adding a per-glyph precise bounds cache in `SimpleFontData`?

### Comment 3 — `simple_font_data.h`: Documentation

> **File:** `third_party/blink/renderer/platform/fonts/simple_font_data.h`
> **Line:** (at `PreciseBoundsForGlyph` declaration)
>
> nit: Consider adding a comment explaining why this is separate from `BoundsForGlyph`:
> ```cpp
> // Returns float-precision bounds from glyph outlines (via SkFont::getPath()),
> // unlike BoundsForGlyph which may return integer-quantized bounds. Intended
> // for Canvas measureText() which requires sub-pixel accuracy at small sizes.
> gfx::RectF PreciseBoundsForGlyph(Glyph) const;
> ```

### Comment 4 — WPT test: Overall positive

> **File:** `third_party/blink/web_tests/external/wpt/html/canvas/element/text/2d.text.measure.actualBoundingBox.small-font.html`
>
> Nice test design — testing the *consequence* of precision (text centering) rather than exact float values makes this robust across platforms. The scale-up trick to make 1.5px text visible is clever.
>
> Optional nit: consider adding a brief comment explaining why yellow is used instead of the conventional WPT green.

---

*Review generated on 2026-02-26. Based on analysis of HLD, LLD, and detailed code review.*
