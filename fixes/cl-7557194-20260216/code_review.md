# Code Review: CL 7557194 — Update drop data if filenames change at drop

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7557194
**Author:** Ashish Kumar <ashishkum@microsoft.com>
**Reviewer:** Copilot

---

## Review Summary

| Category    | Status | Notes                                                                 |
|-------------|--------|-----------------------------------------------------------------------|
| Correctness | ⚠️     | Potential null dereference of `current_drag_data_` in else-if branch  |
| Style       | ⚠️     | Minor: evaluation order in if-init-statement is non-obvious           |
| Security    | ✅     | No new security concerns                                              |
| Performance | ✅     | No performance issues                                                 |
| Testing     | ⚠️     | Missing test for null `current_drag_data_` and feature-disabled case  |

---

## Detailed Findings

#### Issue #1: Potential null pointer dereference of `current_drag_data_`
**Severity**: Major
**File**: `content/browser/web_contents/web_contents_view_aura.cc`
**Line**: 1735
**Description**:
In `PerformDropCallback()`, the `else if` branch at line 1726 is entered when `target_rwh == current_rwh_for_drag_.get()` (i.e., the RWH hasn't changed). However, `current_drag_data_` can be null at this point.

Specifically, in `DragEnteredCallback()` (line 1424), `current_rwh_for_drag_` is set at line 1440 *before* the `allow_drag()` check. If `allow_drag()` returns false, `current_drag_data_` is set to `nullptr` at line 1470, but `current_rwh_for_drag_` remains valid. When `PerformDropCallback()` later runs:

1. `target_rwh != current_rwh_for_drag_.get()` is false (same RWH), so the `if` is skipped.
2. The `else if` branch is entered.
3. Line 1735 dereferences `current_drag_data_->filenames` — **crash if `current_drag_data_` is null**.

The existing null check at line 1749 (`if (!current_drag_data_) { return; }`) exists precisely because `current_drag_data_` can be null at this point, but the new code at line 1735 accesses it before that check.

**Suggestion**:
Add a null check for `current_drag_data_` before dereferencing:
```cpp
  } else if (current_drag_data_ &&
             std::optional<std::vector<ui::FileInfo>> filenames =
                 data->GetFilenames();
             base::FeatureList::IsEnabled(
                 features::kUpdateDropDataWhenFilenamesChange) &&
             filenames.has_value()) {
```
Or more readably, guard the inner block:
```cpp
  } else if (base::FeatureList::IsEnabled(
                 features::kUpdateDropDataWhenFilenamesChange) &&
             current_drag_data_) {
    std::optional<std::vector<ui::FileInfo>> filenames = data->GetFilenames();
    if (filenames.has_value() &&
        current_drag_data_->filenames != filenames.value()) {
      std::unique_ptr<DropData> drop_data = std::make_unique<DropData>();
      PrepareDropData(drop_data.get(), *data.get());
      DragEnteredCallback(drop_metadata, std::move(drop_data), target,
                          transformed_pt);
    }
  }
```

---

#### Issue #2: Unnecessarily complex if-init-statement with mixed evaluation order
**Severity**: Minor
**File**: `content/browser/web_contents/web_contents_view_aura.cc`
**Lines**: 1726–1730
**Description**:
The `else if` uses a C++17 init-statement to declare `filenames`, but the condition mixes the feature flag check and `filenames.has_value()` in a way that can be confusing:

```cpp
  } else if (std::optional<std::vector<ui::FileInfo>> filenames =
                 data->GetFilenames();
             base::FeatureList::IsEnabled(
                 features::kUpdateDropDataWhenFilenamesChange) &&
             filenames.has_value()) {
```

The feature flag check runs before `filenames.has_value()` due to `&&` short-circuit evaluation, which means `data->GetFilenames()` is always called even when the feature is disabled. This is wasteful (though minor) and the structure is harder to read than necessary.

**Suggestion**:
Check the feature flag first, then conditionally retrieve and inspect filenames (as shown in Issue #1's suggested fix). This avoids calling `data->GetFilenames()` when the feature is disabled and simplifies the logic.

---

#### Issue #3: Feature flag is enabled by default — is this intentional?
**Severity**: Minor / Question
**File**: `content/common/features.cc`
**Line**: 724
**Description**:
The feature `kUpdateDropDataWhenFilenamesChange` is defined with `base::FEATURE_ENABLED_BY_DEFAULT`. For a new behavioral change, it's common Chromium practice to initially ship with `FEATURE_DISABLED_BY_DEFAULT` and enable via Finch to allow controlled rollout and easy rollback. Enabling by default means the change ships to all users immediately without a kill switch via server-side configuration.

**Suggestion**:
Consider whether `base::FEATURE_DISABLED_BY_DEFAULT` with a Finch rollout is more appropriate, especially since this changes drop behavior that interacts with external applications like WinRAR.

---

#### Issue #4: Missing test for feature-disabled case
**Severity**: Minor
**File**: `content/browser/web_contents/web_contents_view_aura_unittest.cc`
**Description**:
The test `DropDataUpdatedWhenFilenamesChange` only tests the happy path where the feature flag is enabled (which it is by default). There is no test verifying that when `kUpdateDropDataWhenFilenamesChange` is disabled, the original drag-enter filenames are preserved and NOT updated at drop time.

**Suggestion**:
Add a test with `base::test::ScopedFeatureList` that disables `kUpdateDropDataWhenFilenamesChange` and verifies the original (drag-enter) filenames are retained.

---

#### Issue #5: Test does not cover the null `current_drag_data_` path
**Severity**: Minor
**File**: `content/browser/web_contents/web_contents_view_aura_unittest.cc`
**Description**:
Related to Issue #1 — there is no test where `current_drag_data_` is null when the `else if` branch is entered (e.g., when `allow_drag()` returned false during the initial drag enter). Such a test would have caught the potential null dereference.

**Suggestion**:
Add a test that simulates a drag enter where the delegate blocks the drag (so `current_drag_data_` becomes null), followed by a drop with changed filenames, to verify no crash occurs.

---

#### Issue #6: `filenames.value()` called twice — minor redundancy
**Severity**: Suggestion
**File**: `content/browser/web_contents/web_contents_view_aura.cc`
**Line**: 1735
**Description**:
`filenames.value()` is called on line 1735 for the comparison. Since `filenames` is a `std::optional<std::vector<ui::FileInfo>>`, `.value()` returns a reference so there's no copy, but using `*filenames` is more idiomatic for cases where `has_value()` was already checked.

**Suggestion**:
Use `*filenames` instead of `filenames.value()` after the `has_value()` check (consistent with Chromium style for optionals).

---

## Positive Observations

- **Well-scoped change**: The CL is focused and minimal — only the necessary files are modified.
- **Good comments**: The inline comment at lines 1731–1734 clearly explains the WinRAR scenario and why the update is needed.
- **Feature flag**: Using a feature flag allows the change to be reverted without a code change, which is good practice.
- **Comprehensive test**: The test covers both initial drag-enter state verification and post-drop updated filename verification, with platform-specific path handling.
- **Correct use of `PrepareDropData` + `DragEnteredCallback`**: The update path mirrors the existing target-RWH-changed path at lines 1722–1725, maintaining consistency.

---

## Overall Assessment

**Needs changes before approval**

The primary concern is the potential **null pointer dereference** of `current_drag_data_` (Issue #1), which could cause a crash in production if a drag is rejected by the delegate but the RWH remains the same. This should be fixed before landing.

The feature-flag default state (Issue #3) and missing negative test cases (Issues #4, #5) are secondary but worth addressing.
