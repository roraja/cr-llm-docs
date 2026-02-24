# Fix Implementation: 487158322

## Summary
Added null checks for `GetExecutionContext()` in `MaybeDispatchClipboardChangeEvent()` and `GetSystemClipboard()` in `ClipboardChangeEventController` to prevent null pointer dereference crashes when these methods are called after the `LocalDOMWindow` has been detached during frame destruction. This matches the existing null-check pattern already used in `OnClipboardChanged()` in the same file.

## Branch
- **Branch name**: `fix/487158322`
- **Base commit**: latest_main

## Files Modified

### 1. [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc](/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc)

**Change 1 — Lines 102-107: `MaybeDispatchClipboardChangeEvent()`**

**Before**:
```cpp
void ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent() {
  ExecutionContext* context = GetExecutionContext();
  LocalDOMWindow& window = *To<LocalDOMWindow>(context);
```

**After**:
```cpp
void ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent() {
  ExecutionContext* context = GetExecutionContext();
  if (!context) {
    return;
  }
  LocalDOMWindow& window = *To<LocalDOMWindow>(context);
```

**Rationale**: When `GetExecutionContext()` returns null (because the `LocalDOMWindow` has been detached during frame destruction), the method must return early to avoid dereferencing the null pointer via `*To<LocalDOMWindow>(null)`. This is the primary crash fix, matching the existing pattern in `OnClipboardChanged()` at line 78.

**Change 2 — Lines 62-66: `GetSystemClipboard()`**

**Before**:
```cpp
SystemClipboard* ClipboardChangeEventController::GetSystemClipboard() const {
  ExecutionContext* context = GetExecutionContext();
  LocalFrame* local_frame = To<LocalDOMWindow>(context)->GetFrame();
  return local_frame->GetSystemClipboard();
}
```

**After**:
```cpp
SystemClipboard* ClipboardChangeEventController::GetSystemClipboard() const {
  ExecutionContext* context = GetExecutionContext();
  if (!context) {
    return nullptr;
  }
  LocalFrame* local_frame = To<LocalDOMWindow>(context)->GetFrame();
  return local_frame->GetSystemClipboard();
}
```

**Rationale**: Defensive hardening — `GetSystemClipboard()` is called from `DispatchClipboardChangeEvent()`, which is called from `OnPermissionResult()` (a Mojo async callback). The existing null check on `GetSystemClipboard()`'s return value in `DispatchClipboardChangeEvent()` (line 140-142) handles the `nullptr` return gracefully.

### 2. [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc)

**Lines added**: 249-296 (new test), 11 (new include)

**New include**:
```cpp
#include "third_party/blink/renderer/core/frame/frame.h"
```

**New test**:
```cpp
TEST_F(ClipboardChangeEventTest,
       NoCrashWhenFocusedFrameChangedAfterDetachment) {
  // ... reproduces the exact crash scenario from bug 487158322
  // Sets up clipboardchange listener, triggers clipboard change while unfocused
  // (setting fire_clipboardchange_on_focus_ = true), detaches the frame,
  // then verifies FocusedFrameChanged() does not crash.
}
```

**Rationale**: Regression test that reproduces the exact crash scenario — clipboard change while unfocused, frame detachment, then FocusedFrameChanged() notification with null execution context.

## Git Diff Summary
```
$ git diff --stat HEAD~1
 .../clipboard/clipboard_change_event_controller.cc   |  6 +++++
 .../clipboard/clipboard_change_event_controller_unittest.cc | 47 +++++++++++++++
 2 files changed, 53 insertions(+)
```

## Build Result
```
$ autoninja -C out/release_x64 blink_unittests
35.41s Build Succeeded: 5 steps - 0.14/s
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result
```
$ out/release_x64/blink_unittests --gtest_filter="*ClipboardChangeEvent*"
[1/6] ClipboardChangeEventTest.ClipboardChangeEventFiresWhenFocused (37 ms)
[2/6] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWhenNotFocused (29 ms)
[3/6] ClipboardChangeEventTest.ClipboardChangeEventFiresWithStickyActivation (29 ms)
[4/6] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWithoutStickyActivationOrPermission (29 ms)
[5/6] ClipboardChangeEventTest.StickyActivationTakesPrecedenceOverPermissionCheck (29 ms)
[6/6] ClipboardChangeEventTest.NoCrashWhenFocusedFrameChangedAfterDetachment (29 ms)
SUCCESS: all tests passed.
```

**Status**: ✅ TESTS PASS (6/6 — including the new regression test that previously CRASHED)

## Implementation Notes
- The fix follows Option 1 (primary) + Option 2 (secondary hardening) from the fix assessment
- Both null checks match the existing pattern in `OnClipboardChanged()` (line 76-79) for consistency
- No DCHECKs were added because null `ExecutionContext` is a valid runtime state during frame teardown, not a programming error
- The `fire_clipboardchange_on_focus_` flag is already set to `false` by `FocusedFrameChanged()` before calling `MaybeDispatchClipboardChangeEvent()`, so the early return does not leave stale state

## Known Issues
- None

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
