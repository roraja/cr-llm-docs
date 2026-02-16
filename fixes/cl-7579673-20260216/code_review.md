# Code Review: CL 7579673 — Async Clipboard API: Convert sync mojom IPC to async

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7579673
**Author:** Rohan Raja <roraja@microsoft.com>
**Reviewer:** GitHub Copilot (automated)

---

## Review Summary

| Category    | Status | Notes                                                                                   |
|-------------|--------|-----------------------------------------------------------------------------------------|
| Correctness | ⚠️     | Async ReadPlainText callback used but no async overload visible in SystemClipboard; some async Read* methods have no callers |
| Style       | ✅     | Consistent naming, good comments, follows Chromium conventions                          |
| Security    | ✅     | No new attack surface; sync variants preserve existing permission checks                |
| Performance | ✅     | Core goal achieved — unblocks renderer main thread for Async Clipboard API reads        |
| Testing     | ⚠️     | Good coverage for SyncRead* parity and async ReadPng; missing tests for async ReadPlainText callback lifecycle |

---

## Detailed Findings

#### Issue #1: ReadAvailableTypes async method has no callers
**Severity**: Minor
**File**: third_party/blink/public/mojom/clipboard/clipboard.mojom
**Line**: ~117 (ReadAvailableTypes)
**Description**: The `ReadAvailableTypes` mojom method has been converted from `[Sync]` to async, but no code in the blink renderer actually calls the async version. All callers in `SystemClipboard::ReadAvailableTypes()` were updated to call `SyncReadAvailableTypes`. This means `ReadAvailableTypes` is dead code on the async path.
**Suggestion**: Either add an async overload to `SystemClipboard` for `ReadAvailableTypes` (with callback) for future use, or add a `// TODO` comment noting it will be wired up in a follow-up CL. Same applies to async `ReadText`, `ReadHtml`, `ReadRtf`, `ReadFiles`, and `ReadDataTransferCustomData` — these mojom methods are now async but only the sync variants are actually called from SystemClipboard. Only `ReadPng` and `ReadPlainText` (which wraps `ReadText`) have async SystemClipboard overloads that are actively used.

#### Issue #2: OnReadTextComplete does not check script_promise_resolver_ for null
**Severity**: Major
**File**: third_party/blink/renderer/modules/clipboard/clipboard_promise.cc
**Line**: ~490-496 (OnReadTextComplete)
**Description**: `OnReadTextComplete` checks `GetExecutionContext()` but does not check whether `script_promise_resolver_` is null before calling `DowncastTo<IDLString>()->Resolve(text)`. Since this is now an async callback, there is a wider window where the promise could be rejected or cleaned up (e.g., by `ContextDestroyed()`) before the callback fires. If `script_promise_resolver_` is null or already settled, this could crash or produce undefined behavior.
**Suggestion**: Add a null check: `if (!script_promise_resolver_) return;` before resolving. This follows the defensive pattern already used in other async callback handlers in the codebase.

#### Issue #3: ClipboardPngReader::OnRead does not check promise_ for null
**Severity**: Minor
**File**: third_party/blink/renderer/modules/clipboard/clipboard_reader.cc
**Line**: ~53-60 (OnRead)
**Description**: `ClipboardPngReader::OnRead` calls `promise_->OnRead(blob)` without checking if `promise_` is still valid. While this is pre-existing code that was moved into a separate callback, the async pattern widens the window for `promise_` to become invalid. The same pattern exists in all other `ClipboardReader` subclasses (`ClipboardTextReader`, `ClipboardHtmlReader`, etc.), so this is a systemic pre-existing concern rather than a regression.
**Suggestion**: Consider adding a null check for `promise_` before calling `OnRead`. However, since this matches the existing pattern in all other readers, it could be addressed in a separate cleanup CL.

#### Issue #4: Missing SyncReadSvg variant
**Severity**: Suggestion
**File**: third_party/blink/public/mojom/clipboard/clipboard.mojom
**Line**: ~147 (ReadSvg)
**Description**: `ReadSvg` was already async (no `[Sync]` annotation) before this CL, so no `SyncReadSvg` variant was added. This is correct — no sync caller exists for SVG. However, this creates an asymmetry in the mojom interface: 7 out of 8 Read methods have Sync variants, but ReadSvg does not. A brief comment explaining why would aid future readers.
**Suggestion**: Add a comment near `ReadSvg`: `// No SyncReadSvg needed — SVG is only read via the Async Clipboard API.`

#### Issue #5: Duplicate Change-Id in commit message
**Severity**: Minor
**File**: (commit message)
**Description**: The commit message contains two `Change-Id:` lines (`Ie216c41c...` and `Ifab4848b...`). Only one Change-Id should be present per commit. This may cause issues with Gerrit tracking.
**Suggestion**: Remove the duplicate `Change-Id` line, keeping only the correct one.

#### Issue #6: HandleReadTextWithPermission — second call site also uses async but lacks context check
**Severity**: Minor
**File**: third_party/blink/renderer/modules/clipboard/clipboard_promise.cc
**Line**: ~511-514 (second HandleReadTextWithPermission call site)
**Description**: The second `HandleReadTextWithPermission` code path (the non-Mac path around line 511) also dispatches to `OnReadTextComplete` via async callback. However, the permission check result is already validated at this point. If the execution context is destroyed between the permission check and the async callback completing, `OnReadTextComplete` handles it via `GetExecutionContext()` check, but as noted in Issue #2, `script_promise_resolver_` could still be null.
**Suggestion**: Same fix as Issue #2 — add null check for `script_promise_resolver_` in `OnReadTextComplete`.

#### Issue #7: Test names use Bug prefix — consider crbug convention
**Severity**: Suggestion
**File**: content/browser/renderer_host/clipboard_host_impl_unittest.cc, third_party/blink/renderer/core/clipboard/system_clipboard_test.cc
**Line**: Various
**Description**: Test names use `Bug474131935_` prefix. Chromium convention typically uses `crbug.com/` references in comments rather than embedding bug numbers in test names. While not wrong, the `Bug` prefix in test names is unusual for Chromium.
**Suggestion**: Consider renaming to descriptive test names (e.g., `SyncReadTextReturnsCorrectData`, `ReadPngAsync`) and referencing the bug in a comment. However, this is a minor style preference.

---

## Positive Observations

- **Clean architectural separation**: The approach of adding `SyncRead*` variants for legacy callers while making the primary `Read*` methods async is well-structured and minimizes disruption to existing synchronous paths (DataTransfer/paste events).
- **Comprehensive mock implementations**: Both `content/test/mock_clipboard_host` and `blink/renderer/core/testing/mock_clipboard_host` are updated consistently, ensuring tests work correctly with the new interface.
- **Good test coverage**: The CL adds targeted tests for SyncRead* parity (text, HTML, PNG, formats) and async ReadPng behavior including edge cases (unbound host, empty clipboard).
- **Correct use of WrapPersistent**: Async callbacks in `ClipboardPngReader` and `ClipboardPromise` correctly use `WrapPersistent(this)` to prevent premature garbage collection during async operations.
- **Minimal blast radius**: Only the callers that benefit from async (Async Clipboard API paths) are converted; existing sync callers are transparently routed to `SyncRead*` methods.
- **Consistent commenting**: Each mojom method pair has clear comments indicating which callers use async vs. sync variants.

---

## Overall Assessment

**LGTM with minor comments**

The CL achieves its stated goal of unblocking the renderer main thread for Async Clipboard API reads. The architecture is sound, the mojom changes are clean, and tests cover the key scenarios. The main concern is the missing null check for `script_promise_resolver_` in `OnReadTextComplete` (Issue #2), which should be addressed before landing to prevent potential crashes in edge cases where the execution context is torn down during the async callback window. The other issues are minor and could be addressed in follow-up CLs.
