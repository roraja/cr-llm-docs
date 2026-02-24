# Fix Implementation: 487158322

## Summary
Fixed a null pointer dereference crash in `ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent()` by adding null checks for `GetExecutionContext()`. The crash occurred during frame detachment when `FocusController::NotifyFocusChangedObservers()` invoked `FocusedFrameChanged()` on the controller after the `LocalDOMWindow` had already been destroyed. A defensive null check was also added to `GetSystemClipboard()` for the async Mojo callback path.

## Bug Details
- **Bug ID**: 487158322
- **Issue URL**: https://issues.chromium.org/issues/487158322
- **CL URL**: https://chromium-review.googlesource.com/c/chromium/src/+/7603956

## Root Cause
During `Page::WillBeDestroyed()`, `Frame::Detach()` calls `FocusController::FrameDetached()` which calls `NotifyFocusChangedObservers()`. This invokes `ClipboardChangeEventController::FocusedFrameChanged()`, which calls `MaybeDispatchClipboardChangeEvent()`. At this point, `GetExecutionContext()` returns null because the `LocalDOMWindow` has already been detached. The code unconditionally dereferences the null pointer via `*To<LocalDOMWindow>(context)`, causing a crash.

## Solution
Added an early-return null check for `GetExecutionContext()` at the beginning of `MaybeDispatchClipboardChangeEvent()`, matching the existing pattern already used in `OnClipboardChanged()` in the same file. Also added a defensive null check in `GetSystemClipboard()` to protect against the async `OnPermissionResult` Mojo callback path.

## Files Modified

| File | Change Type | Lines Changed |
|------|-------------|---------------|
| third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc | Modify | +6/-0 |
| third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc | Modify | +47/-0 |

## Code Changes

### third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc

```cpp
// Change 1: GetSystemClipboard() — defensive null check
SystemClipboard* ClipboardChangeEventController::GetSystemClipboard() const {
  ExecutionContext* context = GetExecutionContext();
  if (!context) {
    return nullptr;
  }
  LocalFrame* local_frame = To<LocalDOMWindow>(context)->GetFrame();
  return local_frame->GetSystemClipboard();
}

// Change 2: MaybeDispatchClipboardChangeEvent() — primary crash fix
void ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent() {
  ExecutionContext* context = GetExecutionContext();
  if (!context) {
    return;
  }
  LocalDOMWindow& window = *To<LocalDOMWindow>(context);
  // ... rest of method unchanged
}
```

### third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc

```cpp
// New regression test reproducing the exact crash scenario
TEST_F(ClipboardChangeEventTest,
       NoCrashWhenFocusedFrameChangedAfterDetachment) {
  // Sets up clipboardchange listener, triggers clipboard change while unfocused
  // (setting fire_clipboardchange_on_focus_ = true), detaches the frame,
  // then verifies FocusedFrameChanged() does not crash.
}
```

## Tests Added

| Test Type | File | Test Name |
|-----------|------|-----------|
| Unit Test | third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc | `ClipboardChangeEventTest.NoCrashWhenFocusedFrameChangedAfterDetachment` |

## Verification

| Check | Result |
|-------|--------|
| Build | ✅ Pass |
| Unit Tests (6 ClipboardChangeEvent) | ✅ Pass |
| Broader Tests (97 related tests) | ✅ Pass |
| TDD (test crashed before fix, passes after) | ✅ Pass |
| Code Review | ✅ Pass |
| CQ Dry Run | 🔄 Pending |

## Chrome Binary Location
```
/workspace/cr1/src/out/487158322/chrome
```

## How to Test
```bash
# Run Chrome with the fix
/workspace/cr1/src/out/487158322/chrome \
  --user-data-dir=/tmp/test-487158322 \
  /home/roraja/src/chromium-docs/bugfixer_dashboard/services/bd-build-service-go/contexts/ctx-fc9ddccb4f1b2b91/repro.html
```

## Next Steps
1. Wait for CQ dry run to complete
2. Add reviewers to the CL
3. Address any review feedback
4. Land the change

## References
- CL: https://chromium-review.googlesource.com/c/chromium/src/+/7603956
- Bug: https://issues.chromium.org/issues/487158322
- Existing null-check pattern: `OnClipboardChanged()` in same file (line 78)
- Crash trigger: `FocusController::NotifyFocusChangedObservers()` during frame detachment
