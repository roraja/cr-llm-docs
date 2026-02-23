# TDD Tests: 474131935

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.ReadPngAsync` | ❌ FAILING (compile error) |
| Unit Test | [/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.ReadPngAsyncEmpty` | ❌ FAILING (compile error) |
| Unit Test | [/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc) | `SystemClipboardTest.ReadPngAsyncWithUnboundClipboardHost` | ❌ FAILING (compile error) |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc)

**New tests added** (3 tests appended before `#if BUILDFLAG(IS_MAC)` section):

```cpp
// Bug 474131935: Tests for async ReadPng overload.
// The Async Clipboard API's ClipboardPngReader needs a non-blocking ReadPng
// path. These tests verify the async ReadPng overload on SystemClipboard.

TEST_F(SystemClipboardTest, ReadPngAsync) {
  auto buf = mojom::blink::ClipboardBuffer::kStandard;

  // Create a test bitmap and put it into the clipboard.
  SkBitmap bitmap;
  ASSERT_TRUE(bitmap.tryAllocPixelsFlags(
      SkImageInfo::Make(4, 3, kN32_SkColorType, kOpaque_SkAlphaType), 0));
  clipboard_host()->WriteImage(bitmap);
  clipboard_host()->CommitWrite();

  // Read PNG asynchronously using the callback-based overload.
  bool callback_called = false;
  mojo_base::BigBuffer result_png;
  system_clipboard().ReadPng(
      buf,
      base::BindLambdaForTesting([&](mojo_base::BigBuffer png) {
        callback_called = true;
        result_png = std::move(png);
      }));

  // The callback is dispatched asynchronously via mojo.
  RunUntilIdle();

  EXPECT_TRUE(callback_called);
  EXPECT_GT(result_png.size(), 0u);

  // Verify the PNG data is valid by decoding it.
  SkBitmap decoded = gfx::PNGCodec::Decode(result_png);
  ASSERT_FALSE(decoded.isNull());
  EXPECT_EQ(decoded.width(), 4);
  EXPECT_EQ(decoded.height(), 3);
}

TEST_F(SystemClipboardTest, ReadPngAsyncEmpty) {
  auto buf = mojom::blink::ClipboardBuffer::kStandard;

  // Clipboard starts empty — async ReadPng should return empty buffer.
  bool callback_called = false;
  mojo_base::BigBuffer result_png;
  system_clipboard().ReadPng(
      buf,
      base::BindLambdaForTesting([&](mojo_base::BigBuffer png) {
        callback_called = true;
        result_png = std::move(png);
      }));

  RunUntilIdle();

  EXPECT_TRUE(callback_called);
  EXPECT_EQ(result_png.size(), 0u);
}

TEST_F(SystemClipboardTest, ReadPngAsyncWithUnboundClipboardHost) {
  auto buf = mojom::blink::ClipboardBuffer::kStandard;

  reset_remote_and_validate_buffer();

  // With unbound clipboard host, async ReadPng should invoke callback
  // immediately with empty data.
  bool callback_called = false;
  mojo_base::BigBuffer result_png;
  system_clipboard().ReadPng(
      buf,
      base::BindLambdaForTesting([&](mojo_base::BigBuffer png) {
        callback_called = true;
        result_png = std::move(png);
      }));

  // Callback should be invoked synchronously for the unbound case.
  EXPECT_TRUE(callback_called);
  EXPECT_EQ(result_png.size(), 0u);
}
```

### Test Rationale

These tests exercise the **async `ReadPng` overload** on `SystemClipboard` that must be added as part of the fix for bug 474131935. The bug is that `ClipboardPngReader::Read()` calls the **sync** `SystemClipboard::ReadPng(buffer)` method, blocking the renderer's main thread. The fix adds an async overload `ReadPng(buffer, callback)` and updates `ClipboardPngReader` to use it.

| Test | What it validates |
|------|------------------|
| `ReadPngAsync` | Normal case: writes an image to the mock clipboard, reads it back via the async overload, verifies the callback receives valid PNG data |
| `ReadPngAsyncEmpty` | Empty clipboard: async ReadPng should invoke the callback with an empty `BigBuffer` (size 0) |
| `ReadPngAsyncWithUnboundClipboardHost` | Error case: when the mojo remote is unbound, the callback should be invoked immediately with empty data (no crash, no hang) |

These tests mirror the existing patterns:
- `ReadPngAsync` follows the pattern of `SystemClipboardTest::Png` (sync test)
- `ReadPngAsyncWithUnboundClipboardHost` follows `SystemClipboardTest::ReadPngWithUnboundClipboardHost`
- The callback-based pattern matches how `GetPlatformPermissionStateCallback` is tested

## Pre-Fix Test Execution

### Build Output (MUST SHOW FAILURE)
```
$ autoninja -C out/release_x64 blink_unittests

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:660:7: error: too many arguments to function call, expected 1, have 2
  658 |   system_clipboard().ReadPng(
      |   ~~~~~~~~~~~~~~~~~~~~~~~~~~
  659 |       buf,
  660 |       base::BindLambdaForTesting([&](mojo_base::BigBuffer png) {
      |       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  661 |         callback_called = true;
      |         ~~~~~~~~~~~~~~~~~~~~~~~
  662 |         result_png = std::move(png);
      |         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  663 |       }));
      |       ~~
../../third_party/blink/renderer/core/clipboard/system_clipboard.h:91:24: note: 'ReadPng' declared here
   91 |   mojo_base::BigBuffer ReadPng(mojom::blink::ClipboardBuffer);
      |                        ^       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:694:7: error: too many arguments to function call, expected 1, have 2
  (same pattern for ReadPngAsyncEmpty)

../../third_party/blink/renderer/core/clipboard/system_clipboard_test.cc:716:7: error: too many arguments to function call, expected 1, have 2
  (same pattern for ReadPngAsyncWithUnboundClipboardHost)

3 errors generated.
1m00.38s Build Failure: 3407 done, 1 failed, 4 remaining
```

### Why the tests fail

The tests call `system_clipboard().ReadPng(buf, callback)` — a 2-argument async overload — but `SystemClipboard::ReadPng` currently only has a 1-argument sync overload:

```cpp
// system_clipboard.h (current)
mojo_base::BigBuffer ReadPng(mojom::blink::ClipboardBuffer);  // sync only
```

The fix will add:
```cpp
// system_clipboard.h (after fix)
mojo_base::BigBuffer ReadPng(mojom::blink::ClipboardBuffer);  // existing sync
void ReadPng(mojom::blink::ClipboardBuffer buffer,
             mojom::blink::ClipboardHost::ReadPngCallback callback);  // NEW async
```

Once the async overload is added, these tests will compile and the assertions will verify:
1. The callback is invoked with valid PNG data
2. Empty clipboard returns empty buffer via callback
3. Unbound remote invokes callback immediately with empty data

## Sync vs Async Mojom Assessment

### Question: Must we change ClipboardHost mojom from `[Sync]` to Async?

**Answer: NO — it is NOT required.**

The `[Sync]` annotation in `clipboard.mojom` generates **both** sync and async C++ client bindings. The renderer can already call the async variant (`clipboard_->ReadPng(buffer, std::move(callback))`) without any mojom changes. This is proven by the fact that `ReadText` and `ReadHtml` are also `[Sync]` in mojom, but `ClipboardTextReader` and `ClipboardHtmlReader` already use the async variant without issues.

Chromium does NOT block UI for sync IPC — rather, sync mojo IPC blocks the **calling thread** (the renderer main thread). The browser process handles the request asynchronously regardless. The fix is to have `ClipboardPngReader` call the async mojo variant that is already generated, matching the pattern used by all other clipboard readers.

See [/mojo/public/cpp/bindings/README.md](/mojo/public/cpp/bindings/README.md) — `[Sync]` merely enables sync calling; it does not prevent async calling. Removing `[Sync]` would break legitimate sync callers like `DataObjectItem::GetAsFile()`.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code (compile error: async `ReadPng` overload missing)
- [x] Test failures accurately demonstrate the bug behavior (the missing async API is the root cause)
- [x] Tests cover the main bug scenario (async PNG read with valid data)
- [x] Tests cover relevant edge cases (empty clipboard, unbound mojo remote)
- [x] Test code follows Chromium style guidelines (matches existing `SystemClipboardTest` patterns)

## Next Steps
Once the fix is implemented (Stage 6):
1. Add `void ReadPng(ClipboardBuffer, ReadPngCallback)` to `SystemClipboard` (.h and .cc)
2. Update `ClipboardPngReader::Read()` to use the async overload
3. These 3 tests should compile and PASS
4. Existing sync `ReadPng` tests (`Png`, `ReadPngWithUnboundClipboardHost`) should continue to PASS
