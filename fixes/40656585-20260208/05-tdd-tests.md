# TDD Tests: 40656585

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc) | `ClipboardTest.Bug40656585_BmpTypeIsSupported` | ❌ FAILING |
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc) | `ClipboardTest.Bug40656585_ExistingSupportedTypesUnchanged` | ✅ PASSING |
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc) | `ClipboardTest.Bug40656585_UnsupportedTypesStillFalse` | ✅ PASSING |
| Web Test | [/third_party/blink/web_tests/clipboard/async-clipboard/async-clipboard-bmp-support.html](/third_party/blink/web_tests/clipboard/async-clipboard/async-clipboard-bmp-support.html) | `ClipboardItem.supports() accepts image/bmp` | ❌ FAILING (env issue: V8 snapshot mismatch prevents content_shell from running) |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc)

**New tests added** (lines 209-228):

```cpp
// Tests that ClipboardItem::supports() returns true for image/bmp MIME type.
// Bug 40656585: image/bmp should be a supported clipboard type.
TEST_F(ClipboardTest, Bug40656585_BmpTypeIsSupported) {
  EXPECT_TRUE(ClipboardItem::supports(String("image/bmp")));
}

// Verify existing supported types still work alongside the new BMP type.
TEST_F(ClipboardTest, Bug40656585_ExistingSupportedTypesUnchanged) {
  EXPECT_TRUE(ClipboardItem::supports(String("image/png")));
  EXPECT_TRUE(ClipboardItem::supports(String("text/plain")));
  EXPECT_TRUE(ClipboardItem::supports(String("text/html")));
  EXPECT_TRUE(ClipboardItem::supports(String("image/svg+xml")));
}

// Verify unsupported types still return false.
TEST_F(ClipboardTest, Bug40656585_UnsupportedTypesStillFalse) {
  EXPECT_FALSE(ClipboardItem::supports(String("image/jpeg")));
  EXPECT_FALSE(ClipboardItem::supports(String("image/gif")));
  EXPECT_FALSE(ClipboardItem::supports(String("application/pdf")));
}
```

### Web Tests

#### File: [/third_party/blink/web_tests/clipboard/async-clipboard/async-clipboard-bmp-support.html](/third_party/blink/web_tests/clipboard/async-clipboard/async-clipboard-bmp-support.html)

```html
<!doctype html>
<meta charset="utf-8">
<title>Async clipboard BMP support</title>
<link rel="help" href="https://w3c.github.io/clipboard-apis/#async-clipboard-api">
<body>Body needed for test_driver.click()</body>
<script src="../../resources/testharness.js"></script>
<script src="../../resources/testharnessreport.js"></script>
<script>
// Bug 40656585: Support image/bmp for Clipboard API

test(function() {
  assert_true(ClipboardItem.supports('image/bmp'),
    'ClipboardItem.supports() should return true for image/bmp');
}, '40656585: ClipboardItem.supports() accepts image/bmp');

test(function() {
  assert_true(ClipboardItem.supports('image/png'));
  assert_true(ClipboardItem.supports('text/plain'));
  assert_true(ClipboardItem.supports('text/html'));
  assert_true(ClipboardItem.supports('image/svg+xml'));
}, '40656585: Existing supported types unchanged');

test(function() {
  assert_false(ClipboardItem.supports('image/jpeg'));
  assert_false(ClipboardItem.supports('image/gif'));
}, '40656585: Unsupported types still return false');

test(function() {
  const bmpData = new Uint8Array([0x42, 0x4D, /* ... BMP header ... */]);
  const blob = new Blob([bmpData], {type: 'image/bmp'});
  const item = new ClipboardItem({'image/bmp': blob});
  assert_true(item.types.includes('image/bmp'));
}, '40656585: ClipboardItem constructor accepts image/bmp blob');
</script>
```

**Note**: Web tests could not be executed due to a pre-existing V8 snapshot version mismatch in the build environment (`V8 binary version: 14.4.258` vs `Snapshot version: 14.6.198`). This is a build infrastructure issue, not related to the test code. The web test will fail with the expected assertion error once the environment is fixed.

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 blink_unittests
[3572/3572] LINK ./blink_unittests
12m22.05s Build Succeeded: 3572 steps - 4.81/s
```

### Test Run Output (SHOWS FAILURE)
```
$ out/release_x64/blink_unittests --gtest_filter="*Bug40656585*"

Using sharding settings from environment. This is shard 0/1
Using 1 parallel jobs.
Note: Google Test filter = ClipboardTest.Bug40656585_BmpTypeIsSupported:ClipboardTest.Bug40656585_ExistingSupportedTypesUnchanged:ClipboardTest.Bug40656585_UnsupportedTypesStillFalse
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from ClipboardTest
[ RUN      ] ClipboardTest.Bug40656585_BmpTypeIsSupported
../../third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc:212: Failure
Value of: ClipboardItem::supports(String("image/bmp"))
  Actual: false
Expected: true
Stack trace:
#0 0x6042295b0354 blink::ClipboardTest_Bug40656585_BmpTypeIsSupported_Test::TestBody() [../../third_party/blink/renderer/modules/clipboard/clipboard_unittest.cc:212:3]

[  FAILED  ] ClipboardTest.Bug40656585_BmpTypeIsSupported (88 ms)
[ RUN      ] ClipboardTest.Bug40656585_ExistingSupportedTypesUnchanged
[       OK ] ClipboardTest.Bug40656585_ExistingSupportedTypesUnchanged (28 ms)
[ RUN      ] ClipboardTest.Bug40656585_UnsupportedTypesStillFalse
[       OK ] ClipboardTest.Bug40656585_UnsupportedTypesStillFalse (27 ms)
[----------] 3 tests from ClipboardTest (156 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test suite ran. (168 ms total)
[  PASSED  ] 2 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] ClipboardTest.Bug40656585_BmpTypeIsSupported

 1 FAILED TEST
```

### Analysis of Test Results

- **`Bug40656585_BmpTypeIsSupported`**: ❌ **FAILED** — `ClipboardItem::supports("image/bmp")` returns `false` because `image/bmp` is not in the allowlist in `clipboard_item.cc:149-150`. This correctly demonstrates the bug.
- **`Bug40656585_ExistingSupportedTypesUnchanged`**: ✅ PASSED — Confirms existing types (`image/png`, `text/plain`, `text/html`, `image/svg+xml`) are supported.
- **`Bug40656585_UnsupportedTypesStillFalse`**: ✅ PASSED — Confirms unsupported types still return `false`.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior
- [x] Tests cover the main bug scenario
- [x] Tests cover relevant edge cases (existing types, unsupported types)
- [x] Test code follows Chromium style guidelines
- [x] Web test created for JS API surface coverage (blocked by environment issue)

## Root Cause Verified by Tests

The failing test proves the root cause identified in the assessment: `ClipboardItem::supports()` in `clipboard_item.cc` (line 149) has a hardcoded allowlist that does not include `image/bmp`. The fix will add `ui::kMimeTypeBmp` to this allowlist (after defining the constant in `clipboard_constants.h`).

## Next Steps
Once the fix is implemented (Stage 6), the `Bug40656585_BmpTypeIsSupported` test should PASS, while the other two tests should continue to pass.
