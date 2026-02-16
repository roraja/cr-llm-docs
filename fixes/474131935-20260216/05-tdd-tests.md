# TDD Tests: 474131935

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [system_clipboard_test.cc](/workspace/cr3/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.Bug474131935_ReadPngAsync` | ❌ FAILING (compile error) |
| Unit Test | [system_clipboard_test.cc](/workspace/cr3/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.Bug474131935_ReadPngAsyncWithUnboundHost` | ❌ FAILING (compile error) |
| Unit Test | [system_clipboard_test.cc](/workspace/cr3/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.Bug474131935_ReadPngAsyncEmpty` | ❌ FAILING (compile error) |
| Unit Test | [clipboard_host_impl_unittest.cc](/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc) | `ClipboardHostImplTest.Bug474131935_SyncReadTextReturnsCorrectData` | ❌ FAILING (compile error) |
| Unit Test | [clipboard_host_impl_unittest.cc](/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc) | `ClipboardHostImplTest.Bug474131935_SyncReadHtmlReturnsCorrectData` | ❌ FAILING (compile error) |
| Unit Test | [clipboard_host_impl_unittest.cc](/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc) | `ClipboardHostImplTest.Bug474131935_SyncReadPngReturnsCorrectData` | ❌ FAILING (compile error) |
| Unit Test | [clipboard_host_impl_unittest.cc](/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc) | `ClipboardHostImplTest.Bug474131935_SyncReadAvailableCustomAndStandardFormats` | ❌ FAILING (compile error) |

## Tests Created

### Unit Tests

#### File: [/workspace/cr3/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/workspace/cr3/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc)

**New tests added** (lines 649-740):

These tests validate the new async `ReadPng(buffer, callback)` overload on `SystemClipboard`. This overload is required so `ClipboardPngReader` can use the async pattern (like `ClipboardTextReader` and `ClipboardHtmlReader` already do), avoiding blocking the renderer main thread via `[Sync]` mojom IPC.

```cpp
// Bug 474131935: Test async ReadPng overload for non-blocking clipboard reads.
TEST_F(SystemClipboardTest, Bug474131935_ReadPngAsync) {
  auto buf = mojom::blink::ClipboardBuffer::kStandard;
  SkBitmap bitmap;
  ASSERT_TRUE(bitmap.tryAllocPixelsFlags(
      SkImageInfo::Make(4, 3, kN32_SkColorType, kOpaque_SkAlphaType), 0));
  clipboard_host()->WriteImage(bitmap);
  clipboard_host()->CommitWrite();

  bool callback_called = false;
  mojo_base::BigBuffer received_png;
  system_clipboard().ReadPng(
      buf,
      BindOnce([](bool* called, mojo_base::BigBuffer* out,
                  mojo_base::BigBuffer result) {
            *called = true;
            *out = std::move(result);
          },
          Unretained(&callback_called),
          Unretained(&received_png)));
  RunUntilIdle();

  EXPECT_TRUE(callback_called);
  EXPECT_GT(received_png.size(), 0u);
  SkBitmap decoded = gfx::PNGCodec::Decode(received_png);
  ASSERT_FALSE(decoded.isNull());
  EXPECT_EQ(decoded.width(), 4);
  EXPECT_EQ(decoded.height(), 3);
}

// Bug 474131935: Test async ReadPng with unbound clipboard host.
TEST_F(SystemClipboardTest, Bug474131935_ReadPngAsyncWithUnboundHost) {
  // Verifies graceful handling when mojo remote is unbound.
  // ...
}

// Bug 474131935: Test async ReadPng with empty clipboard.
TEST_F(SystemClipboardTest, Bug474131935_ReadPngAsyncEmpty) {
  // Verifies empty data returned when no image is present.
  // ...
}
```

#### File: [/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc](/workspace/cr3/src/content/browser/renderer_host/clipboard_host_impl_unittest.cc)

**New tests added** (lines 1043-1115):

These tests validate the new `SyncRead*` mojom methods on `ClipboardHost`. The fix adds `[Sync] SyncReadText`, `SyncReadHtml`, `SyncReadPng`, and `SyncReadAvailableCustomAndStandardFormats` as legacy variants for DataTransfer/paste event callers, while the existing `ReadText`, `ReadHtml`, `ReadPng` become non-`[Sync]` (async) for the Async Clipboard API.

```cpp
// Bug 474131935: Test that SyncReadText returns the same data as ReadText.
TEST_F(ClipboardHostImplTest, Bug474131935_SyncReadTextReturnsCorrectData) {
  {
    ui::ScopedClipboardWriter writer(ui::ClipboardBuffer::kCopyPaste);
    writer.WriteText(u"async clipboard test");
  }
  std::u16string async_result;
  mojo_clipboard()->ReadText(ui::ClipboardBuffer::kCopyPaste, &async_result);
  EXPECT_EQ(u"async clipboard test", async_result);

  std::u16string sync_result;
  mojo_clipboard()->SyncReadText(ui::ClipboardBuffer::kCopyPaste, &sync_result);
  EXPECT_EQ(u"async clipboard test", sync_result);
  EXPECT_EQ(async_result, sync_result);
}

// Bug 474131935: Test that SyncReadHtml returns the same data as ReadHtml.
TEST_F(ClipboardHostImplTest, Bug474131935_SyncReadHtmlReturnsCorrectData) { ... }

// Bug 474131935: Test that SyncReadPng returns the same data as ReadPng.
TEST_F(ClipboardHostImplTest, Bug474131935_SyncReadPngReturnsCorrectData) { ... }

// Bug 474131935: Test that SyncReadAvailableCustomAndStandardFormats works.
TEST_F(ClipboardHostImplTest,
       Bug474131935_SyncReadAvailableCustomAndStandardFormats) { ... }
```

## Pre-Fix Test Execution

### Build Output — system_clipboard_test.cc (blink_unittests)

```
$ autoninja --offline --fast_local -C out/release_x64 blink_unittests

FAILED: obj/third_party/blink/renderer/core/unit_tests/system_clipboard_test.o

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:669:7: error: too many arguments to function call, expected 1, have 2
  667 |   system_clipboard().ReadPng(
../../third_party/blink/renderer/core/clipboard/system_clipboard.h:91:24: note: 'ReadPng' declared here
   91 |   mojo_base::BigBuffer ReadPng(mojom::blink::ClipboardBuffer);

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:703:7: error: too many arguments to function call, expected 1, have 2
  701 |   system_clipboard().ReadPng(

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:728:7: error: too many arguments to function call, expected 1, have 2
  726 |   system_clipboard().ReadPng(

Build Failure: 593 done, 1 failed, 508 remaining
```

**Root cause**: The async `ReadPng(buffer, callback)` overload does not exist yet on `SystemClipboard`. Only the sync `ReadPng(buffer)` overload exists. The fix will add the async overload.

### Build Output — clipboard_host_impl_unittest.cc (content_unittests)

```
$ autoninja --offline --fast_local -C out/release_x64 content_unittests

FAILED: obj/content/test/content_unittests/clipboard_host_impl_unittest.o

../../content/browser/renderer_host/clipboard_host_impl_unittest.cc:1061:21: error: no member named 'SyncReadText' in 'blink::mojom::ClipboardHostProxy'
 1061 |   mojo_clipboard()->SyncReadText(ui::ClipboardBuffer::kCopyPaste, &sync_result);

../../content/browser/renderer_host/clipboard_host_impl_unittest.cc:1077:21: error: no member named 'SyncReadHtml' in 'blink::mojom::ClipboardHostProxy'
 1077 |   mojo_clipboard()->SyncReadHtml(ui::ClipboardBuffer::kCopyPaste, &markup, &url,

../../content/browser/renderer_host/clipboard_host_impl_unittest.cc:1093:21: error: no member named 'SyncReadPng' in 'blink::mojom::ClipboardHostProxy'
 1093 |   mojo_clipboard()->SyncReadPng(ui::ClipboardBuffer::kCopyPaste, &sync_png);

../../content/browser/renderer_host/clipboard_host_impl_unittest.cc:1113:21: error: no member named 'SyncReadAvailableCustomAndStandardFormats' in 'blink::mojom::ClipboardHostProxy'
 1113 |   mojo_clipboard()->SyncReadAvailableCustomAndStandardFormats(&sync_formats);

Build Failure: 1 failed
```

**Root cause**: The `SyncRead*` methods do not exist yet in the `ClipboardHost` mojom interface (`clipboard.mojom`). Currently, all read methods use `[Sync]` annotation directly (e.g., `[Sync] ReadText`). The fix will:
1. Remove `[Sync]` from `ReadText`, `ReadHtml`, `ReadPng`, etc. (making them async)
2. Add new `[Sync] SyncReadText`, `SyncReadHtml`, `SyncReadPng`, etc. for legacy callers

## Why These Tests Demonstrate the Bug

### SystemClipboard Tests (async ReadPng)
The bug is that `ClipboardPngReader::Read()` calls `SystemClipboard::ReadPng(buffer)` synchronously, which triggers a `[Sync]` mojom IPC call that blocks the renderer main thread. The fix adds an async `ReadPng(buffer, callback)` overload to `SystemClipboard` and converts `ClipboardPngReader` to use it. These tests verify the async overload exists and works correctly. The compile error proves the async API doesn't exist yet.

### ClipboardHostImpl Tests (SyncRead* variants)
The fix removes `[Sync]` from the existing read methods in `clipboard.mojom` and adds new `SyncRead*` methods for backward compatibility with legacy synchronous callers (DataTransfer/paste events). These tests verify the new `SyncRead*` methods exist and return correct data. The compile error proves the legacy sync variants don't exist yet — meaning the fix hasn't split the sync/async paths yet.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code (compile errors)
- [x] Test failures accurately demonstrate the bug behavior (missing async API surface)
- [x] Tests cover the main bug scenario (async ReadPng, SyncRead* variants)
- [x] Tests cover relevant edge cases (unbound host, empty clipboard)
- [x] Test code follows Chromium style guidelines (gtest patterns, naming conventions)

## HLD: Top 5 Approaches

### 1. Convert Mojom Read Methods to Async (⭐ RECOMMENDED — Implemented)
Remove `[Sync]` from `ClipboardHost` read methods (`ReadText`, `ReadHtml`, `ReadPng`, etc.) in `clipboard.mojom` for the Async Clipboard API path. Add `[Sync] SyncRead*` legacy variants for DataTransfer/paste callers. Update `SystemClipboard` to route sync callers through `SyncRead*` and async callers through the now-async `Read*` methods.

### 2. Move OS Clipboard Access to Background Thread in Browser Process
Keep `[Sync]` mojom but move `ui::Clipboard` reads in `ClipboardHostImpl` to `base::ThreadPool`. Still blocks renderer (sync IPC), and `ui::Clipboard` has thread affinity issues (`GetForCurrentThread()`, Windows `::OpenClipboard()`).

### 3. Add Timeout to Sync Mojom IPC Calls
Add timeout to sync calls so renderer unblocks after N seconds. Doesn't fix root cause — users still freeze up to timeout. Clipboard data may be lost if timeout is too short.

### 4. Introduce a New Async Mojom Interface (e.g., `AsyncClipboardHost`)
Create a separate mojom interface with only async methods. Clean separation but significant code duplication and over-engineering.

### 5. Use Mojo Associated Interface with `[NoInterrupt]` and Async Pattern
Remove `[Sync]` from all methods and use `RunLoop` in the renderer for legacy callers. Dangerous — nested RunLoops cause reentrancy issues and are explicitly discouraged in Chromium.

## Next Steps
Once the fix is implemented (Stage 6), these tests should PASS:
1. Add `ReadPng(buffer, callback)` async overload to `SystemClipboard` → SystemClipboard tests compile and pass
2. Add `[Sync] SyncRead*` methods to `clipboard.mojom` → ClipboardHostImpl tests compile and pass
3. Remove `[Sync]` from existing `ReadText`, `ReadHtml`, `ReadPng` etc. in mojom
4. Update `ClipboardPngReader::Read()` to use async `ReadPng(buffer, callback)`
5. Update `ClipboardPromise::HandleReadTextWithPermission()` to use async `ReadPlainText(buffer, callback)`
