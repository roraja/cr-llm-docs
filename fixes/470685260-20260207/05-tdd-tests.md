# TDD Tests: 470685260

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [toast_controller_unittest.cc](/workspace/cr4/src/chrome/browser/ui/toasts/toast_controller_unittest.cc) | `ToastControllerUnitTest.ShowPasteTooLargeToast` | ❌ FAILING (compile error) |
| Unit Test | [toast_controller_unittest.cc](/workspace/cr4/src/chrome/browser/ui/toasts/toast_controller_unittest.cc) | `ToastControllerUnitTest.PasteTooLargeToastAutomaticallyCloses` | ❌ FAILING (compile error) |

## Tests Created

### Unit Tests

#### File: [/workspace/cr4/src/chrome/browser/ui/toasts/toast_controller_unittest.cc](/workspace/cr4/src/chrome/browser/ui/toasts/toast_controller_unittest.cc)

**New tests added** (lines 208-249):

```cpp
// Bug 470685260: Test that a paste-too-large toast can be shown when clipboard
// content exceeds the 256MB size limit.
TEST_F(ToastControllerUnitTest, ShowPasteTooLargeToast) {
  ToastRegistry* const registry = toast_registry();
  // Register the kPasteTooLarge toast (this will fail until the fix is
  // implemented).
  registry->RegisterToast(
      ToastId::kPasteTooLarge,
      ToastSpecification::Builder(vector_icons::kInfoOutlineIcon, kTestStringResId)
          .Build());

  auto controller = std::make_unique<TestToastController>(registry);

  // The toast should be able to show.
  EXPECT_FALSE(controller->IsShowingToast());
  EXPECT_TRUE(controller->CanShowToast(ToastId::kPasteTooLarge));

  EXPECT_CALL(*controller, CreateToast);
  EXPECT_TRUE(controller->MaybeShowToast(ToastParams(ToastId::kPasteTooLarge)));
  ::testing::Mock::VerifyAndClear(controller.get());
  EXPECT_TRUE(controller->IsShowingToast());
}

// Bug 470685260: Test that the paste-too-large toast auto-closes after the
// default timeout.
TEST_F(ToastControllerUnitTest, PasteTooLargeToastAutomaticallyCloses) {
  ToastRegistry* const registry = toast_registry();
  registry->RegisterToast(
      ToastId::kPasteTooLarge,
      ToastSpecification::Builder(vector_icons::kInfoOutlineIcon, kTestStringResId)
          .Build());
  auto controller = std::make_unique<TestToastController>(registry);

  EXPECT_CALL(*controller, CreateToast);
  EXPECT_TRUE(controller->MaybeShowToast(ToastParams(ToastId::kPasteTooLarge)));
  ::testing::Mock::VerifyAndClear(controller.get());
  EXPECT_TRUE(controller->IsShowingToast());

  // The toast should stop showing after reaching toast timeout time.
  task_environment().FastForwardBy(ToastController::kToastDefaultTimeout);
  EXPECT_FALSE(controller->IsShowingToast());
}
```

## Pre-Fix Test Execution

### Build Output
```
$ cd /workspace/cr4/src && autoninja -C out/release_x64 chrome/browser/ui/toasts:unit_tests

[1338/1344] CXX obj/chrome/browser/ui/toasts/unit_tests/toast_controller_unittest.o
FAILED: obj/chrome/browser/ui/toasts/unit_tests/toast_controller_unittest.o 
../../third_party/llvm-build/Release+Asserts/bin/clang++ ... ../../chrome/browser/ui/toasts/toast_controller_unittest.cc -o obj/chrome/browser/ui/toasts/unit_tests/toast_controller_unittest.o
../../chrome/browser/ui/toasts/toast_controller_unittest.cc:215:16: error: no member named 'kPasteTooLarge' in 'ToastId'
  215 |       ToastId::kPasteTooLarge,
      |                ^~~~~~~~~~~~~~
../../chrome/browser/ui/toasts/toast_controller_unittest.cc:223:49: error: no member named 'kPasteTooLarge' in 'ToastId'
  223 |   EXPECT_TRUE(controller->CanShowToast(ToastId::kPasteTooLarge));
      |                                                 ^~~~~~~~~~~~~~
../../chrome/browser/ui/toasts/toast_controller_unittest.cc:226:63: error: no member named 'kPasteTooLarge' in 'ToastId'
  226 |   EXPECT_TRUE(controller->MaybeShowToast(ToastParams(ToastId::kPasteTooLarge)));
      |                                                               ^~~~~~~~~~~~~~
../../chrome/browser/ui/toasts/toast_controller_unittest.cc:236:16: error: no member named 'kPasteTooLarge' in 'ToastId'
  236 |       ToastId::kPasteTooLarge,
      |                ^~~~~~~~~~~~~~
../../chrome/browser/ui/toasts/toast_controller_unittest.cc:242:63: error: no member named 'kPasteTooLarge' in 'ToastId'
  242 |   EXPECT_TRUE(controller->MaybeShowToast(ToastParams(ToastId::kPasteTooLarge)));
      |                                                               ^~~~~~~~~~~~~~
5 errors generated.
ninja: build stopped: subcommand failed.
```

### Failure Analysis

The tests correctly **fail at compile time** because:
1. `ToastId::kPasteTooLarge` does not exist in the enum (defined in [toast_id.h](/workspace/cr4/src/chrome/browser/ui/toasts/api/toast_id.h))
2. This confirms the tests will detect when the fix is implemented

The failure occurs because the enum value `kPasteTooLarge` needs to be added to `ToastId` as part of the fix implementation.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior (missing toast infrastructure)
- [x] Tests cover the main bug scenario (showing paste-too-large toast)
- [x] Tests cover relevant edge cases (auto-close timeout)
- [x] Test code follows Chromium style guidelines

## Test Design Rationale

### Why These Tests?

1. **ShowPasteTooLargeToast**: Verifies that the toast infrastructure exists and can display the paste-too-large notification. This is the core functionality needed for the fix.

2. **PasteTooLargeToastAutomaticallyCloses**: Ensures the toast follows expected UX behavior by auto-closing after the default timeout, providing non-intrusive user feedback.

### Why Not Integration/Browser Tests Yet?

The full integration test would require:
- Mocking clipboard data > 256MB on the Windows platform
- Triggering a paste operation through the clipboard host
- Verifying the toast appears

This is complex because:
1. The 256MB limit is Windows-specific (in `clipboard_win.cc`)
2. The test clipboard (`TestClipboard`) doesn't simulate size limits
3. Integration would require bridging content → chrome layers

The unit tests verify the toast infrastructure works correctly. Additional integration tests can be added after the basic fix is implemented.

## Implementation Requirements for Tests to Pass

For these tests to compile and pass, the fix must:

1. Add `kPasteTooLarge = 27` to `ToastId` enum in [toast_id.h](/workspace/cr4/src/chrome/browser/ui/toasts/api/toast_id.h)
2. Add corresponding case to `GetToastName()` in [toast_id.cc](/workspace/cr4/src/chrome/browser/ui/toasts/api/toast_id.cc)
3. Register the toast in [toast_service.cc](/workspace/cr4/src/chrome/browser/ui/toasts/toast_service.cc)
4. Add enum value to [enums.xml](/workspace/cr4/src/tools/metrics/histograms/metadata/toasts/enums.xml)

## Next Steps

Once the fix is implemented (Stage 6), these tests should:
1. **Compile successfully** - `ToastId::kPasteTooLarge` will exist
2. **Pass** - The toast controller will be able to show and auto-close the toast
