# Code Review: CL 7564913 — [Fonts] Fix canvas measureText() precision at small font sizes

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7564913
**Author:** Sejal Anand <sejalanand@microsoft.com>
**Patch Set:** 11
**Reviewer:** Automated Review

---

## Review Summary

| Category    | Status | Notes                                                                                  |
|-------------|--------|----------------------------------------------------------------------------------------|
| Correctness | ⚠️     | `GetPathBoundsForGlyph` rounds path bounds back to integers when `!isSubpixel()`, potentially negating the precision fix |
| Style       | ⚠️     | Minor duplication of subpixel-rounding logic; otherwise clean                          |
| Security    | ✅     | No security concerns                                                                    |
| Performance | ⚠️     | Per-glyph `getPath()` call replaces precomputed `InkBounds()` with no caching          |
| Testing     | ⚠️     | Single WPT visual test; no unit tests for `PreciseBoundsForGlyph` directly             |

---

## Detailed Findings

### Issue #1: `GetPathBoundsForGlyph` rounds precise bounds back to integers when subpixel is off
**Severity**: Major
**File**: `third_party/blink/renderer/platform/fonts/skia/skia_text_metrics.cc`
**Lines**: 141-145
**Description**:
The shared helper `GetPathBoundsForGlyph` applies `roundOut` → `SkIRect` → `set` when `!font.isSubpixel()`, which rounds the float-precision path bounds back to integer boundaries. This is the same rounding that `SkFontGetBoundsForGlyph` did on non-Apple platforms previously, so `SkFontGetPreciseBoundsForGlyph` may not actually provide improved precision when subpixel rendering is disabled.

The function is named "Precise" but its output depends on the subpixel state. If canvas rendering contexts typically have subpixel enabled this may be acceptable in practice (the WPT test passes on trybots), but the behavior should be explicitly documented. Reviewer Dileep Maurya also flagged this at line 160.

**Suggestion**:
Either:
1. Remove the `roundOut` logic from `GetPathBoundsForGlyph` and only apply it in `SkFontGetBoundsForGlyph` (which preserves existing behavior), making `PreciseBoundsForGlyph` always truly precise, OR
2. Add a comment in `SkFontGetPreciseBoundsForGlyph` explaining why the rounding is acceptable (e.g., canvas always uses subpixel rendering), and add a `DCHECK(font.isSubpixel())` or document the limitation.

Option 1 is preferred — `GetPathBoundsForGlyph` should return raw path bounds, and each caller can decide whether to round:

```cpp
static void GetPathBoundsForGlyph(const SkFont& font,
                                  Glyph glyph,
                                  SkRect* bounds) {
  if (const auto path = font.getPath(glyph)) {
    *bounds = path->getBounds();
  } else {
    *bounds = font.getBounds(glyph, nullptr);
  }
  // Caller is responsible for rounding if needed.
}

void SkFontGetBoundsForGlyph(const SkFont& font, Glyph glyph, SkRect* bounds) {
#if BUILDFLAG(IS_APPLE)
  GetPathBoundsForGlyph(font, glyph, bounds);
#else
  *bounds = font.getBounds(glyph, nullptr);
#endif
  if (!font.isSubpixel()) {
    SkIRect ir;
    bounds->roundOut(&ir);
    bounds->set(ir);
  }
}

void SkFontGetPreciseBoundsForGlyph(const SkFont& font,
                                    Glyph glyph,
                                    SkRect* bounds) {
  GetPathBoundsForGlyph(font, glyph, bounds);
  // Intentionally no rounding — callers need float precision.
}
```

---

### Issue #2: Performance regression — per-glyph `getPath()` replaces precomputed `InkBounds()`
**Severity**: Major
**File**: `third_party/blink/renderer/core/html/canvas/text_metrics.cc`
**Lines**: 195-206
**Description**:
The original code used `item.InkBounds()` which returns a single precomputed rect per shape result run. The new code iterates every glyph via `ForEachGlyph` and calls `font_data->PreciseBoundsForGlyph(glyph)` for each one, which internally calls `SkFont::getPath()`. This involves:
- Glyph outline extraction from the font backend
- Path construction
- Path bounds computation

For text-heavy canvas workloads (e.g., games, data visualizations), this could be a measurable regression. Reviewer Dileep Maurya raised the same concern.

**Suggestion**:
Consider one or more of:
1. Add a glyph bounds cache (e.g., `glyph_to_precise_bounds_map_` in `SimpleFontData`) keyed by glyph ID, similar to how other per-glyph data is cached.
2. Benchmark the change with a canvas text performance test (e.g., repeatedly calling `measureText()` in a tight loop) and document the results.
3. If caching is deferred, add a TODO with a tracking bug.

---

### Issue #3: Subpixel rounding code duplication
**Severity**: Minor
**File**: `third_party/blink/renderer/platform/fonts/skia/skia_text_metrics.cc`
**Lines**: 141-145 and 156-160
**Description**:
The `!isSubpixel()` rounding pattern appears twice:
1. In `GetPathBoundsForGlyph` (lines 141-145)
2. In the `#else` branch of `SkFontGetBoundsForGlyph` (lines 156-160)

This duplication could lead to maintenance divergence.

**Suggestion**:
If Issue #1 is addressed by removing the rounding from `GetPathBoundsForGlyph`, this duplication is eliminated. Otherwise, extract the rounding into a small inline helper like `RoundBoundsIfNotSubpixel()`.

---

### Issue #4: Gaurav's design question — new method vs updating existing `BoundsForGlyph`
**Severity**: Suggestion
**File**: `third_party/blink/renderer/core/html/canvas/text_metrics.cc`
**Line**: 373
**Description**:
Reviewer Gaurav Kumar asked: "Rather than creating a new method why not update the content of [BoundsForGlyph] so that other places where this is used also has float-precision bounds."

This is a valid design question. The CL introduces a parallel API surface (`PreciseBoundsForGlyph` alongside `BoundsForGlyph`, `PlatformBoundsForGlyph`). The author should clarify why the existing API cannot be updated:
- Is there a concern about changing behavior for non-canvas callers?
- Are other callers depending on the integer-quantized bounds for pixel-alignment?

**Suggestion**:
Add a code comment near the declaration of `PreciseBoundsForGlyph` in `simple_font_data.h` explaining the distinction from `BoundsForGlyph`, e.g.:
```cpp
// Returns float-precision bounds from glyph outlines (via SkFont::getPath()),
// unlike BoundsForGlyph which may return integer-quantized bounds from Skia's
// internal representation. Use for APIs requiring sub-pixel accuracy (e.g.,
// Canvas measureText() at small font sizes).
gfx::RectF PreciseBoundsForGlyph(Glyph) const;
```

---

### Issue #5: Test coverage — only visual/integration test, no unit test
**Severity**: Minor
**File**: `third_party/blink/web_tests/external/wpt/html/canvas/element/text/2d.text.measure.actualBoundingBox.small-font.html`
**Description**:
The WPT test is a good integration test that validates the end-to-end behavior. However, there are no unit tests for:
- `SkFontGetPreciseBoundsForGlyph` returning true float bounds
- `SimpleFontData::PreciseBoundsForGlyph` behavior
- Edge cases: bitmap-only glyphs (emoji), zero-size fonts, empty glyphs

The test also only covers a single font size (1.5px) and a single character ('A').

**Suggestion**:
Consider adding a `blink_unittests` test in `skia_text_metrics_test.cc` or `simple_font_data_test.cc` that verifies `PreciseBoundsForGlyph` returns non-integer bounds for a known glyph at a small font size.

---

### Issue #6: WPT test uses yellow text instead of conventional green/red
**Severity**: Suggestion
**File**: `third_party/blink/web_tests/external/wpt/html/canvas/element/text/2d.text.measure.actualBoundingBox.small-font.html`
**Lines**: 60-61
**Description**:
WPT convention is to use green for pass and red for fail. The test uses yellow text on red background. The author explained this replicates the original bug report scenario, which is reasonable context, but future test readers may find the color choice confusing.

The center-pixel check `pixel[1] > 128` relies on the green channel, which works because yellow = (255, 255, 0) has green=255 while red = (255, 0, 0) has green=0. This is functional but non-obvious.

**Suggestion**:
Consider switching to green text on red background for WPT convention, or at minimum add a comment explaining the yellow choice, e.g.:
```javascript
// Yellow text on red background - replicates the centering scenario
// from the original bug report (crbug.com/479240778).
```

---

## Positive Observations

- **Good problem decomposition**: The fix cleanly separates the path-based precision logic into a reusable helper (`GetPathBoundsForGlyph`) and a public API (`PreciseBoundsForGlyph`), keeping `BoundsForGlyph` unchanged for existing callers.
- **Correct fallback handling**: The code properly falls back to `SkFont::getBounds()` for bitmap-only glyphs (e.g., color emoji) that have no outlines.
- **Good commit message**: Clearly describes the problem (integer-quantized bounds), the fix (path-based float precision), and the fallback behavior.
- **Responsive to review feedback**: The author addressed naming, lambda parameter style, and commit message formatting suggestions.
- **Refactoring of Apple-specific code**: Extracting the path-based bounds logic into `GetPathBoundsForGlyph` is a nice cleanup that removes the `#if BUILDFLAG(IS_APPLE)` duplication.

---

## Overall Assessment

**Needs changes before approval**

The core approach (using `SkFont::getPath()` for float-precision glyph bounds) is sound and addresses a real user-facing bug. However, two significant issues should be addressed:

1. **Correctness (Issue #1)**: `GetPathBoundsForGlyph` applies `roundOut` to integers when subpixel is off, which may negate the precision improvement. The rounding should be removed from the shared helper and kept only in `SkFontGetBoundsForGlyph`, so `PreciseBoundsForGlyph` always returns true float-precision bounds.

2. **Performance (Issue #2)**: Replacing precomputed `InkBounds()` with per-glyph `getPath()` calls is a potential regression for text-heavy workloads. At minimum, benchmarks should confirm acceptable impact. Ideally, a per-glyph bounds cache should be added.

The remaining issues (duplication, documentation, test coverage) are minor and can be addressed incrementally.
