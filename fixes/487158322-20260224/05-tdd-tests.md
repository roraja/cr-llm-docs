# TDD Tests: 487158322

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc) | `ClipboardChangeEventTest.NoCrashWhenFocusedFrameChangedAfterDetachment` | ❌ CRASHED |

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc)

**New test added** (lines 256-298):

```cpp
// Regression test for crbug.com/487158322: Verify no crash when
// FocusedFrameChanged() is triggered during frame detachment with
// fire_clipboardchange_on_focus_ set to true, which causes
// MaybeDispatchClipboardChangeEvent() to be called with a null
// execution context.
TEST_F(ClipboardChangeEventTest,
       NoCrashWhenFocusedFrameChangedAfterDetachment) {
  ExecutionContext* execution_context = GetFrame().DomWindow();
  GetFrame().GetSystemClipboard()->OnClipboardDataChanged({"text/plain"}, 1);

  SetSecureOrigin(execution_context);
  SetPageFocus(true);

  auto* clipboard_change_event_handler =
      MakeGarbageCollected<EventCountingListener>();
  GetDocument().addEventListener(event_type_names::kClipboardchange,
                                 clipboard_change_event_handler, false);
  auto* clipboard_change_event_controller =
      MakeGarbageCollected<ClipboardChangeEventController>(
          *GetFrame().DomWindow()->navigator(), &GetDocument());

  // Simulate a clipboard change while page is not focused, so that
  // fire_clipboardchange_on_focus_ is set to true internally.
  SetPageFocus(false);
  clipboard_change_event_controller->DidUpdateData();
  test::RunPendingTasks();
  EXPECT_EQ(clipboard_change_event_handler->Count(), 0);

  // Detach the frame without re-focusing first. Frame::Detach() calls
  // DetachImpl() which destroys the window (GetExecutionContext() returns
  // null), then FocusController::FrameDetached() → SetFocusedFrame(nullptr)
  // → NotifyFocusChangedObservers() → FocusedFrameChanged().
  // With fire_clipboardchange_on_focus_ true, this triggers
  // MaybeDispatchClipboardChangeEvent() with a null execution context.
  // Before fix: null dereference crash. After fix: early return, no crash.
  GetFrame().Detach(FrameDetachType::kRemove);

  // If we reach here, no crash occurred.
  EXPECT_EQ(clipboard_change_event_handler->Count(), 0);
}
```

**New include added** (line 11):

```cpp
#include "third_party/blink/renderer/core/frame/frame.h"
```

### Web Tests

Not applicable — this is a C++ lifecycle/null-pointer crash that is best reproduced via unit test. The crash occurs during frame detachment, which is a Blink-internal operation not directly triggerable from a web test in a reliable, deterministic way.

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 blink_unittests
59.31s Build Succeeded: 2 steps - 0.03/s
```

### Test Run Output (CRASH CONFIRMED)
```
$ out/release_x64/blink_unittests --gtest_filter="*ClipboardChangeEvent*"

[1/6] ClipboardChangeEventTest.ClipboardChangeEventFiresWhenFocused (37 ms)
[2/6] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWhenNotFocused (28 ms)
[3/6] ClipboardChangeEventTest.ClipboardChangeEventFiresWithStickyActivation (29 ms)
[4/6] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWithoutStickyActivationOrPermission (29 ms)
[5/6] ClipboardChangeEventTest.StickyActivationTakesPrecedenceOverPermissionCheck (29 ms)
[6/6] ClipboardChangeEventTest.NoCrashWhenFocusedFrameChangedAfterDetachment (CRASHED)

Crash stack trace (key frames):
#5 blink::ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent()
#6 blink::FocusController::NotifyFocusChangedObservers()
#7 blink::FocusController::SetFocusedFrame()
#8 blink::Frame::Detach()
#9 blink::ClipboardChangeEventTest_NoCrashWhenFocusedFrameChangedAfterDetachment_Test::TestBody()
    [clipboard_change_event_controller_unittest.cc:290]

1 test crashed:
    ClipboardChangeEventTest.NoCrashWhenFocusedFrameChangedAfterDetachment
```

The crash occurs at exactly the same call chain as reported in bug 487158322:
`Frame::Detach()` → `FocusController::SetFocusedFrame()` → `NotifyFocusChangedObservers()` → `MaybeDispatchClipboardChangeEvent()` — null dereference on `*To<LocalDOMWindow>(context)` when `context` is null.

## Test Design Rationale

The test reproduces the exact crash scenario from the bug report:

1. **Setup**: Create a `ClipboardChangeEventController` with a valid window and clipboard data
2. **Trigger deferred dispatch**: Simulate a clipboard change while the page is unfocused, which sets `fire_clipboardchange_on_focus_ = true` inside the controller
3. **Detach the frame**: Call `GetFrame().Detach(FrameDetachType::kRemove)` which:
   - Calls `LocalFrame::DetachImpl()` → `DomWindow()->FrameDestroyed()` → `NotifyContextDestroyed()` → makes `Navigator::DomWindow()` return null
   - Then calls `FocusController::FrameDetached(this)` → `SetFocusedFrame(nullptr)` → `NotifyFocusChangedObservers()` → `FocusedFrameChanged()`
4. **Crash point**: `FocusedFrameChanged()` sees `fire_clipboardchange_on_focus_ == true`, calls `MaybeDispatchClipboardChangeEvent()`, which calls `GetExecutionContext()` (returns null), then `*To<LocalDOMWindow>(null)` → **null dereference crash**

All 5 existing tests continue to pass, confirming the new test doesn't affect existing functionality.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL (CRASH) on the current (unfixed) code
- [x] Test failure accurately demonstrates the bug behavior (same crash stack as bug report)
- [x] Tests cover the main bug scenario (null execution context during FocusedFrameChanged after detachment)
- [x] Test code follows Chromium style guidelines
- [x] Existing tests continue to pass

## Next Steps
Once the fix is implemented (Stage 6) — adding a null check for `GetExecutionContext()` in `MaybeDispatchClipboardChangeEvent()` — this test should PASS without crashing.
