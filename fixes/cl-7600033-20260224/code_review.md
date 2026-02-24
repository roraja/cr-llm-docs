# Code Review: CL 7600033

## Make ClipboardPngReader use async ReadPng to avoid blocking renderer

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7600033
**Author:** Rohan Raja <roraja@microsoft.com>
**Files Changed:** 4 files, +93/-3 lines

---

## Review Summary

| Category | Status | Notes |
|----------|--------|-------|
| Correctness | ✅ | Logic is correct and follows established async patterns |
| Style | ⚠️ | Minor commit message issue; code style is good |
| Security | ✅ | No new attack surface; proper input validation |
| Performance | ✅ | Directly addresses the blocking renderer issue |
| Testing | ✅ | Three well-structured tests covering key scenarios |

---

## Detailed Findings

#### Issue #1: Duplicate Change-Id in commit message
**Severity**: Minor
**File**: (commit message)
**Line**: N/A
**Description**: The commit message contains two `Change-Id:` lines:
- `Change-Id: I427e065fd7416edbe887a76725672b97b738aa61`
- `Change-Id: I87a0dd1143aa8b5e2db62ed482266fff71fdb66c`

Only one Change-Id should be present. Having two may cause confusion in Gerrit, though Gerrit typically uses the last one.
**Suggestion**: Remove the first `Change-Id` line and keep only the final one.

---

#### Issue #2: Async ReadPng does not interact with snapshot cache
**Severity**: Suggestion
**File**: system_clipboard.cc
**Line**: 267-275
**Description**: The synchronous `ReadPng(buffer)` checks `snapshot_` and caches the result (lines 254-262), but the new async `ReadPng(buffer, callback)` bypasses the snapshot entirely. While this is **consistent** with how all other async overloads work (ReadHTML async, ReadSvg async, ReadPlainText async all skip snapshots), it's worth documenting this deliberate design choice for future maintainers.

The consequence is that if `ScopedSystemClipboardSnapshot` is active, the async ReadPng will still go through mojo IPC rather than returning cached data. This is acceptable because:
1. The async path is only used by the Async Clipboard API, not by paste/copy code paths where snapshots are relevant.
2. All other async overloads follow the same pattern.

**Suggestion**: Consider adding a brief comment such as:
```cpp
// Async overload — snapshot cache is intentionally skipped, consistent with
// other async Read* methods. The async path is only used by the Async
// Clipboard API.
```
This is purely informational and not blocking.

---

#### Issue #3: `BindOnce` without explicit namespace in clipboard_reader.cc
**Severity**: Suggestion
**File**: clipboard_reader.cc
**Line**: 49
**Description**: `BindOnce` is used without an explicit `base::` namespace prefix. This works because of using declarations in WTF, but explicit `base::BindOnce` would be slightly clearer. However, the existing `ClipboardTextReader`, `ClipboardHtmlReader`, and `ClipboardSvgReader` in the same file all use the same unqualified `BindOnce`, so this is consistent with the local convention.
**Suggestion**: No change needed — consistent with surrounding code. Noting for awareness only.

---

## Positive Observations

- **Follows established patterns precisely**: The new async `ReadPng` overload is structurally identical to `ReadPlainText(buffer, callback)`, `ReadHTML(callback)`, and `ReadSvg(callback)`. The error handling (unbound clipboard → invoke callback with empty data) matches perfectly.

- **Clean separation of Read and OnRead**: Splitting the synchronous `Read()` into `Read()` (initiation) + `OnRead()` (callback) in `ClipboardPngReader` is the exact same pattern used by `ClipboardTextReader`, `ClipboardHtmlReader`, and other readers. The `private:` access specifier was correctly repositioned.

- **Correct lifetime management**: Using `WrapPersistent(this)` in the `BindOnce` call correctly prevents the GC from collecting the `ClipboardPngReader` before the mojo callback fires.

- **Thorough test coverage**: Three tests cover the important scenarios:
  - `ReadPngAsync`: Normal path — writes bitmap, reads async, validates PNG decoding including dimensions.
  - `ReadPngAsyncEmpty`: Empty clipboard edge case.
  - `ReadPngAsyncWithUnboundClipboardHost`: Error path — verifies synchronous callback invocation when clipboard host is unbound.

- **No mojom changes needed**: The CL correctly leverages the fact that `[Sync]` mojo interfaces generate both sync and async C++ bindings, avoiding unnecessary mojom changes.

- **Well-written commit message**: Clear problem statement (blocking renderer with delayed clipboard writes like Excel), concise solution description, and reference to the bug.

- **`DCHECK_CALLED_ON_VALID_SEQUENCE`**: Correctly placed in `OnRead()` to verify the callback executes on the expected sequence.

---

## Overall Assessment

**LGTM with minor comments**

This is a clean, well-structured CL that correctly converts the only remaining synchronous clipboard read in the Async Clipboard API to use the async mojo binding. The code follows all established patterns, handles edge cases properly, and has good test coverage.

The only actionable item is the duplicate `Change-Id` in the commit message (Issue #1). The snapshot cache observation (Issue #2) is informational and not blocking.
