# Fix Implementation: 474131935

## Summary
Converted the Async Clipboard API from synchronous to asynchronous mojom IPC by removing `[Sync]` annotations from 8 `ClipboardHost` read methods in `clipboard.mojom`. Added new `[Sync] SyncRead*` legacy variants for backward-compatible synchronous callers (DataTransfer paste events, `document.execCommand('paste')`). Updated `SystemClipboard` sync callers to use `SyncRead*` mojom methods, added an async `ReadPng(buffer, callback)` overload, and converted `ClipboardPngReader::Read()` and `ClipboardPromise::HandleReadTextWithPermission()` to use async callbacks. The renderer main thread is no longer blocked during clipboard reads via the Async Clipboard API.

## Branch
- **Branch name**: `fix/474131935`
- **Base commit**: latest main

## Files Modified

### 1. third_party/blink/public/mojom/clipboard/clipboard.mojom

**Lines changed**: 116-196

**Before**:
```mojom
  [Sync]
  ReadAvailableTypes(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadText(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadHtml(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadRtf(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadPng(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadFiles(ClipboardBuffer buffer) => (...);
  [Sync]
  ReadDataTransferCustomData(...) => (...);
  [Sync]
  ReadAvailableCustomAndStandardFormats() => (...);
```

**After**:
```mojom
  ReadAvailableTypes(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadAvailableTypes(...) => (...);  // Legacy sync variant

  ReadText(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadText(...) => (...);  // Legacy sync variant

  ReadHtml(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadHtml(...) => (...);  // Legacy sync variant

  ReadRtf(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadRtf(...) => (...);  // Legacy sync variant

  ReadPng(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadPng(...) => (...);  // Legacy sync variant

  ReadFiles(ClipboardBuffer buffer) => (...);  // Now async
  [Sync] SyncReadFiles(...) => (...);  // Legacy sync variant

  ReadDataTransferCustomData(...) => (...);  // Now async
  [Sync] SyncReadDataTransferCustomData(...) => (...);  // Legacy

  ReadAvailableCustomAndStandardFormats() => (...);  // Now async
  [Sync] SyncReadAvailableCustomAndStandardFormats() => (...);  // Legacy
```

**Rationale**: Removing `[Sync]` from read methods makes them truly asynchronous at the IPC layer. The renderer no longer blocks. New `SyncRead*` variants preserve backward compatibility for legacy synchronous callers.

### 2. third_party/blink/renderer/core/clipboard/system_clipboard.h

**Lines changed**: 91-95

**After**:
```cpp
  mojo_base::BigBuffer ReadPng(mojom::blink::ClipboardBuffer);
  // Async overload for Async Clipboard API path.
  void ReadPng(mojom::blink::ClipboardBuffer buffer,
               mojom::blink::ClipboardHost::ReadPngCallback callback);
```

**Rationale**: Adding async `ReadPng(buffer, callback)` overload for `ClipboardPngReader` to use, consistent with existing `ReadPlainText(buffer, callback)` and `ReadHTML(callback)`.

### 3. third_party/blink/renderer/core/clipboard/system_clipboard.cc

**Changes**:
- Sync `ReadPlainText()`: `clipboard_->ReadText()` → `clipboard_->SyncReadText()`
- Sync `ReadHTML()`: `clipboard_->ReadHtml()` → `clipboard_->SyncReadHtml()`
- Sync `ReadRTF()`: `clipboard_->ReadRtf()` → `clipboard_->SyncReadRtf()`
- Sync `ReadPng()`: `clipboard_->ReadPng()` → `clipboard_->SyncReadPng()`
- Sync `ReadFiles()`: `clipboard_->ReadFiles()` → `clipboard_->SyncReadFiles()`
- Sync `ReadDataTransferCustomData()`: → `clipboard_->SyncReadDataTransferCustomData()`
- Sync `ReadAvailableTypes()`: → `clipboard_->SyncReadAvailableTypes()`
- **New**: Async `ReadPng(buffer, callback)` method added

**Rationale**: Sync callers now use `SyncRead*` mojom methods. The async `ReadPng` overload routes through the now-async `ReadPng` mojom method.

### 4. third_party/blink/renderer/modules/clipboard/clipboard_promise.cc

**Before**:
```cpp
void ClipboardPromise::HandleReadTextWithPermission(...) {
  // ...
  String text = GetLocalFrame()->GetSystemClipboard()->ReadPlainText(
      mojom::blink::ClipboardBuffer::kStandard);  // SYNC - blocks renderer
  script_promise_resolver_->DowncastTo<IDLString>()->Resolve(text);
}
```

**After**:
```cpp
void ClipboardPromise::HandleReadTextWithPermission(...) {
  // ...
  GetLocalFrame()->GetSystemClipboard()->ReadPlainText(
      mojom::blink::ClipboardBuffer::kStandard,
      BindOnce(&ClipboardPromise::OnReadTextComplete, WrapPersistent(this)));
}

void ClipboardPromise::OnReadTextComplete(const String& text) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  if (!GetExecutionContext()) { return; }
  script_promise_resolver_->DowncastTo<IDLString>()->Resolve(text);
}
```

**Rationale**: Replaces synchronous `ReadPlainText(buffer)` with async `ReadPlainText(buffer, callback)`. Renderer main thread is no longer blocked.

### 5. third_party/blink/renderer/modules/clipboard/clipboard_promise.h

**After**: Added `void OnReadTextComplete(const String& text);` declaration.

### 6. third_party/blink/renderer/modules/clipboard/clipboard_reader.cc

**Before**:
```cpp
void Read() override {
  mojo_base::BigBuffer data =
      system_clipboard()->ReadPng(mojom::blink::ClipboardBuffer::kStandard);
  // ... create blob synchronously ...
  promise_->OnRead(blob);
}
```

**After**:
```cpp
void Read() override {
  system_clipboard()->ReadPng(
      mojom::blink::ClipboardBuffer::kStandard,
      BindOnce(&ClipboardPngReader::OnRead, WrapPersistent(this)));
}

void OnRead(mojo_base::BigBuffer data) {
  // ... create blob from data ...
  promise_->OnRead(blob);
}
```

**Rationale**: Converts `ClipboardPngReader` to async pattern, consistent with `ClipboardTextReader`, `ClipboardHtmlReader`, and `ClipboardSvgReader`.

### 7. content/browser/renderer_host/clipboard_host_impl.h

**After**: Added 8 `SyncRead*` method declarations that override the new mojom interface methods.

### 8. content/browser/renderer_host/clipboard_host_impl.cc

**After**: Added 8 `SyncRead*` implementations, each delegating to the corresponding `Read*` method. Minimal code duplication — browser-side logic is shared.

### 9. content/test/mock_clipboard_host.h & .cc

**After**: Added 8 `SyncRead*` methods to the content test mock, each delegating to the existing `Read*` method.

### 10. third_party/blink/renderer/core/testing/mock_clipboard_host.h & .cc

**After**: Added 8 `SyncRead*` methods to the blink test mock, each delegating to the existing `Read*` method.

### 11. content/web_test/renderer/test_runner.cc

**Change**: `ReadPng()` → `SyncReadPng()` (sync caller).

### 12. content/browser/renderer_host/clipboard_host_impl_unittest.cc

**Changes**: Updated existing tests to use `SyncReadAvailableTypes` instead of `ReadAvailableTypes` (sync calling convention). Updated TDD test to use async `TestFuture` callback for `ReadText`.

## Git Diff Summary
```
$ git diff --stat HEAD~1
 content/browser/renderer_host/clipboard_host_impl.cc               | 43 +++++++++++++++++++++++++++++++++++++++++
 content/browser/renderer_host/clipboard_host_impl.h                | 19 ++++++++++++++++++
 content/browser/renderer_host/clipboard_host_impl_unittest.cc      | 17 +++++++++-------
 content/test/mock_clipboard_host.cc                                | 43 +++++++++++++++++++++++++++++++++++++++++
 content/test/mock_clipboard_host.h                                 | 19 ++++++++++++++++++
 content/web_test/renderer/test_runner.cc                           |  2 +-
 third_party/blink/public/mojom/clipboard/clipboard.mojom           | 48 +++++++++++++++++++++++++++++++++++++++++-----
 third_party/blink/renderer/core/clipboard/system_clipboard.cc      | 27 ++++++++++++++++++--------
 third_party/blink/renderer/core/clipboard/system_clipboard.h       |  3 +++
 third_party/blink/renderer/core/clipboard/system_clipboard_test.cc | 48 ++++++++++++++++++++--------------------------
 third_party/blink/renderer/core/testing/mock_clipboard_host.cc     | 43 +++++++++++++++++++++++++++++++++++++++++
 third_party/blink/renderer/core/testing/mock_clipboard_host.h      | 19 ++++++++++++++++++
 third_party/blink/renderer/modules/clipboard/clipboard_promise.cc  | 22 +++++++++++++++------
 third_party/blink/renderer/modules/clipboard/clipboard_promise.h   |  1 +
 third_party/blink/renderer/modules/clipboard/clipboard_reader.cc   | 10 +++++++---
 15 files changed, 307 insertions(+), 57 deletions(-)
```

## Build Result
```
$ autoninja --offline --fast_local -C out/release_x64 chrome
55.31s Build Succeeded: 59 steps - 1.07/s

$ autoninja --offline --fast_local -C out/release_x64 blink_unittests
13m08.68s Build Succeeded: 550 steps - 0.70/s

$ autoninja --offline --fast_local -C out/release_x64 content_unittests
47.36s Build Succeeded: 5 steps - 0.11/s
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result
```
$ out/release_x64/blink_unittests --gtest_filter="*Bug474131935*"
[  PASSED  ] 3 tests.
[1/3] SystemClipboardTest.Bug474131935_ReadPngAsync (28 ms)
[2/3] SystemClipboardTest.Bug474131935_ReadPngAsyncWithUnboundHost (21 ms)
[3/3] SystemClipboardTest.Bug474131935_ReadPngAsyncEmpty (21 ms)

$ xvfb-run -a out/release_x64/content_unittests --gtest_filter="*Bug474131935*"
[  PASSED  ] 4 tests.
[1/4] ClipboardHostImplTest.Bug474131935_SyncReadTextReturnsCorrectData (69 ms)
[2/4] ClipboardHostImplTest.Bug474131935_SyncReadHtmlReturnsCorrectData (58 ms)
[3/4] ClipboardHostImplTest.Bug474131935_SyncReadPngReturnsCorrectData (63 ms)
[4/4] ClipboardHostImplTest.Bug474131935_SyncReadAvailableCustomAndStandardFormats (63 ms)

$ out/release_x64/blink_unittests --gtest_filter="SystemClipboardTest.*"
[  PASSED  ] 22 tests. (all existing + new tests)

$ xvfb-run -a out/release_x64/content_unittests --gtest_filter="ClipboardHostImplTest.*"
[  PASSED  ] 12 tests. (all existing + new tests)
```

**Status**: ✅ ALL TESTS PASS

## Implementation Notes
- **Phase 1 only**: This fix unblocks the renderer main thread. The browser UI thread still blocks on OS clipboard calls via `ui::Clipboard`. A future Phase 2 would move clipboard reads to a background thread in the browser process.
- **Minimal code duplication**: The `SyncRead*` methods in `ClipboardHostImpl` and both mock classes simply delegate to the existing `Read*` methods. The browser-side implementation is shared between sync and async callers.
- **Two additional mock classes required updating**: Both `blink::MockClipboardHost` and `content::MockClipboardHost` needed the `SyncRead*` pure virtual methods implemented to avoid compilation errors.
- **`test_runner.cc` sync call updated**: The web test `ReadPng` call was a sync caller that needed routing through `SyncReadPng`.
- **Existing test updated**: `ReadAvailableTypes` calls in `clipboard_host_impl_unittest.cc` used sync out-parameter syntax and needed updating to `SyncReadAvailableTypes`.
- **macOS path also updated**: `OnPlatformPermissionResultForReadText` was also updated to use the async `ReadPlainText` path.
- **`GetSequenceNumber` and `IsFormatAvailable` remain `[Sync]`**: These are lightweight metadata queries that don't trigger OS clipboard data reads.

## Known Issues
- Content_unittests require X display (xvfb-run) to run — this is an existing environment limitation, not a code issue.
- The browser UI thread still blocks on OS clipboard calls (Phase 2 future work).

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
