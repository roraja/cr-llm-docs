# Code Review: CL 7580274 — [Clipboard][Windows] Plumb data_dst through internal helpers

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7580274
**Author:** Hewro Hewei (ihewro@chromium.org)
**Files Changed:** `ui/base/clipboard/clipboard_win.cc` (+43/-27), `ui/base/clipboard/clipboard_win.h` (+20/-12)
**Bug:** 458194647

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ✅ | Mechanical plumbing, no logic change. Parameter consistently passed through. |
| Style | ⚠️ | Minor: removal of `result->clear()` is a subtle behavior change. Comment placement inconsistency. |
| Security | ✅ | No security-sensitive changes. |
| Performance | ✅ | No performance impact; `std::optional<DataTransferEndpoint>` passed by const-ref. |
| Testing | ⚠️ | No new tests, but the CL claims no behavior change. Existing tests should cover. |

## CL Chain Context

This CL is part of a series (bug 458194647) making clipboard reads non-blocking on Windows:
1. **CL 7574178** (MERGED): Async `ReadAvailableTypes` with ThreadPool offloading — plumbed `data_dst` through `ReadAvailableTypesInternal`.
2. **CL 7580274** (this CL): Plumbs `data_dst` through `ReadTextInternal`, `ReadAsciiTextInternal`, `ReadHTMLInternal`, `ReadFilenamesInternal` — purely mechanical, preparing for future use.
3. **CL 7578053** (NEW): Makes `ReadPng` non-blocking via `ReadAsync`.
4. **CL 7578233** (NEW): Makes `ReadSvg`/`ReadRTF`/`ReadDataTransferCustomData`/`ReadData` async.

The CL dependency chain is: 7580274 → 7578053 → 7578233. This CL is the foundation.

## Detailed Findings

#### Issue #1: Removal of `result->clear()` in synchronous ReadText
**Severity**: Minor
**File**: clipboard_win.cc
**Line**: 536-538 (new code)
**Description**: The old synchronous `ReadText` (taking `DataTransferEndpoint*`) had:
```cpp
result->clear();
*result = ReadTextInternal(buffer, GetClipboardWindow());
```
The new code is:
```cpp
*result = ReadTextInternal(buffer, base::OptionalFromPtr(data_dst), GetClipboardWindow());
```
The explicit `result->clear()` was removed. While assignment from the return value of `ReadTextInternal` will replace the contents anyway (so this is functionally equivalent), the same change applies to `ReadFilenames` (line 777-779) which also lost its `result->clear()`. This is correct behavior — the assignment operator replaces the value — but the commit message says "No behavior change intended," and this is a subtle semantic change that a reader might question.
**Suggestion**: This is actually fine since `operator=` fully replaces the value. No action needed, but worth noting for reviewers.

#### Issue #2: Comment placement style — `// static` and `// |data_dst|` on separate lines
**Severity**: Suggestion
**File**: clipboard_win.cc
**Lines**: 541-542, 580-581, 620-621, 782-783
**Description**: The pattern used is:
```cpp
// static
// |data_dst| is not used, but is kept as it may be used in the future.
```
This is consistent with the existing codebase pattern established in the predecessor CL (7574178) for `ReadAvailableTypesInternal`, `GetStandardFormatsInternal`, and `IsFormatAvailableInternal`. Good consistency.
**Suggestion**: No action needed — just confirming the pattern is consistent.

#### Issue #3: `data_dst` unused in all internal methods
**Severity**: Suggestion
**File**: clipboard_win.cc
**Lines**: 543-546, 582-585, 622-629, 784-787
**Description**: Every `*Internal` method now accepts `data_dst` but ignores it. The `[[maybe_unused]]` attribute or a `(void)data_dst;` suppression is not used. Since these are documented with `// |data_dst| is not used, but is kept as it may be used in the future.`, this follows Chromium convention (comments suffice, no compiler warnings expected since the parameter is named in the signature and could be referenced).
**Suggestion**: Consider whether Chromium style prefers `[[maybe_unused]]` for such parameters. In practice, the current approach is fine and matches the existing pattern in this file.

#### Issue #4: Consistency with CL 7578053 and CL 7578233
**Severity**: Suggestion
**File**: N/A
**Description**: The related CLs (7578053, 7578233) build on this CL. CL 7578053 "Make ReadPng non-blocking" lists this CL as a parent. CL 7578233 "Use async ReadSvg/ReadRTF/ReadDataTransferCustomData/ReadData" extends further. The plumbing in this CL correctly prepares the signature changes needed for the subsequent CLs. The `ReadAsync` template signature `base::OnceCallback<Result(HWND)>` means the internal methods, which now take `(buffer, data_dst, owner_window)`, will be partially applied via `base::BindOnce` with `buffer` and `data_dst` bound, leaving `HWND` as the remaining parameter — matching the template's expectation. This is correct.
**Suggestion**: No issues found.

#### Issue #5: Lambda in ReadHTML captures `data_dst` by const-ref
**Severity**: Minor
**File**: clipboard_win.cc
**Lines**: 455-466
**Description**: The lambda in `ReadHTML`:
```cpp
[](ClipboardBuffer buffer,
   const std::optional<DataTransferEndpoint>& data_dst,
   HWND owner_window) { ... }
```
is bound via `base::BindOnce(..., buffer, data_dst)`. When `base::BindOnce` binds `data_dst` (an `std::optional<DataTransferEndpoint>` const-ref), it will **copy** the optional into the bound state. When the lambda is later invoked, it receives a const-ref to that stored copy. This is safe — `base::BindOnce` always stores bound arguments by value. No dangling reference risk.
**Suggestion**: No action needed. This is correct behavior with `base::BindOnce`.

## Positive Observations

- **Mechanical and consistent**: The CL applies the same pattern uniformly across all four `*Internal` methods. The parameter insertion order is consistent (`buffer, data_dst, owner_window`).
- **Good use of `base::OptionalFromPtr`**: The synchronous overloads (taking `DataTransferEndpoint*`) correctly convert to `std::optional` using `base::OptionalFromPtr(data_dst)`, maintaining the bridge between the raw-pointer API and the optional-based internal API.
- **Clean follow-up to CL 7574178**: The pattern established in the predecessor (merged) CL is followed exactly, making the series easy to review.
- **Well-scoped**: The CL touches only the minimum code needed — no unrelated changes.
- **Removal of `result->clear()` is correct**: Since the result is assigned via `operator=` from the return value of the internal function, the explicit clear was redundant.

## Overall Assessment

**LGTM with minor comments**

This is a straightforward mechanical plumbing CL that adds `data_dst` as a parameter to internal static helper methods in `ClipboardWin`. The parameter is not yet used but prepares the codebase for future use (per bug 458194647). The changes are consistent, correct, and well-scoped. The related CLs (7578053, 7578233) build cleanly on top of this foundation.

The only noteworthy items are:
1. The removal of `result->clear()` calls is functionally correct but not mentioned in the commit message as an intentional cleanup — trivial.
2. No new tests, but since there's no behavior change, existing test coverage is sufficient.
