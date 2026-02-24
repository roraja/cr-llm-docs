# TDD Tests: Clipboardchange Event Null Dereference Crash

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/workspace/cr1/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc) | `ClipboardChangeEventTest.NoCrashWhenFocusChangedAfterContextDestroyed` | ❌ CRASHED (SEGV_MAPERR 0x234) |
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/workspace/cr1/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc) | `ClipboardChangeEventTest.NoCrashWhenClipboardChangedAfterContextDestroyed` | ✅ PASSING (existing null check covers this) |
| Unit Test | [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/workspace/cr1/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc) | `ClipboardChangeEventTest.NoCrashDuringTeardownWithPendingFocusDispatch` | ❌ CRASHED (SEGV_MAPERR 0x234) |

## Top 5 Approaches to Fix and Recommended One

### Approach 1: Minimal Null Check in `MaybeDispatchClipboardChangeEvent()` Only
- **What**: Add `if (!context) return;` before the dereference at line 104.
- **Pros**: Smallest possible change (2 lines); directly fixes the primary crash site.
- **Cons**: Does NOT fix the same null-dereference risk in `GetSystemClipboard()` (line 64), reachable via `DispatchClipboardChangeEvent()` → `GetSystemClipboard()`.
- **Risk**: Low for the single crash, but **incomplete** — leaves a known crash path unguarded.
- **Verdict**: Insufficient. Fixes only 1 of 2 known crash sites.

### Approach 2: Comprehensive Null Checks in ALL Vulnerable Methods ⭐ RECOMMENDED
- **What**: Add null checks for `GetExecutionContext()` in `MaybeDispatchClipboardChangeEvent()`, `GetSystemClipboard()`, and `FocusedFrameChanged()`.
- **Pros**: Fixes all known crash paths; follows existing pattern in `OnClipboardChanged()` (line 78); minimal change (~8 lines across 3 methods in 1 file); semantically correct (no events should fire during teardown).
- **Cons**: Slightly more lines than Approach 1, but all follow the same trivial pattern.
- **Risk**: Low. Each change is a standard null-guard-and-early-return.
- **Verdict**: **Recommended.** Complete, consistent, minimal risk.

### Approach 3: Unregister `FocusChangedObserver` During Frame Detachment
- **What**: Override `ContextDestroyed()` to unregister the controller from `FocusController` before context becomes null.
- **Pros**: Architecturally correct; prevents the call entirely.
- **Cons**: More complex (4+ files); `FocusController` doesn't expose `RemoveFocusChangedObserver()`; risk of breaking other observers.
- **Risk**: Medium. Higher complexity and broader blast radius.
- **Verdict**: Over-engineered for this fix.

### Approach 4: Add DCHECK + Null Check Guard Pattern
- **What**: Add `DCHECK(GetExecutionContext())` at entry points plus null-check early returns for production safety.
- **Pros**: Catches unexpected null states in debug builds; documents expectations.
- **Cons**: Null state during teardown is **expected** (not a bug), so `DCHECK` would fire on legitimate code paths in debug/test builds.
- **Risk**: Medium. DCHECKs firing in valid teardown scenarios is actively harmful.
- **Verdict**: Inappropriate. Null context during teardown is expected.

### Approach 5: Use `WeakMember<LocalDOMWindow>` for Explicit Lifetime Tracking
- **What**: Store `WeakMember<LocalDOMWindow>` in the controller and check validity instead of routing through `GetSupplementable()->DomWindow()`.
- **Pros**: Makes lifetime management explicit; idiomatic Oilpan pattern.
- **Cons**: Over-engineering; adds redundancy with `Supplement<Navigator>` pattern; two sources of truth; requires header changes.
- **Risk**: Medium. Architectural change to lifetime management for a simple null-check fix.
- **Verdict**: Over-engineered. Adds complexity without proportional benefit.

## Tests Created

### Unit Tests

#### File: [/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/workspace/cr1/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc)

**3 new tests added** (lines 249–352):

```cpp
// Test 1: Direct call to FocusedFrameChanged() after FrameDestroyed()
TEST_F(ClipboardChangeEventTest,
       NoCrashWhenFocusChangedAfterContextDestroyed) {
  // Setup: secure origin, unfocused page, clipboard data available
  // Action: DidUpdateData() while unfocused → fire_clipboardchange_on_focus_ = true
  // Then: DomWindow()->FrameDestroyed() → GetExecutionContext() returns null
  // Then: FocusedFrameChanged() → MaybeDispatchClipboardChangeEvent()
  // Before fix: CRASH at *To<LocalDOMWindow>(nullptr)
  // After fix: early return, no crash
}

// Test 2: DidUpdateData() after FrameDestroyed()
TEST_F(ClipboardChangeEventTest,
       NoCrashWhenClipboardChangedAfterContextDestroyed) {
  // Setup: secure origin, focused page, controller created
  // Then: DomWindow()->FrameDestroyed() → GetExecutionContext() returns null
  // Then: DidUpdateData() → OnClipboardChanged() → null check → early return
  // This already works (OnClipboardChanged has existing null check)
}

// Test 3: Natural teardown crash path via Page::WillBeDestroyed()
TEST_F(ClipboardChangeEventTest,
       NoCrashDuringTeardownWithPendingFocusDispatch) {
  // Setup: secure origin, page focused (sets focused_frame_ = main_frame)
  // Create controller, unfocus page, DidUpdateData() → fire_clipboardchange_on_focus_ = true
  // Let PageTestBase::TearDown() destroy the page naturally
  // Crash path: Page::WillBeDestroyed() → Frame::Detach() →
  //   DomWindow()->FrameDestroyed() (context destroyed) →
  //   FocusController::FrameDetached() → SetFocusedFrame(nullptr) →
  //   NotifyFocusChangedObservers() → FocusedFrameChanged() →
  //   MaybeDispatchClipboardChangeEvent() → CRASH
  // Before fix: CRASH (SEGV_MAPERR at offset 0x234 from null)
  // After fix: early return, no crash
}
```

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 blink_unittests
56.84s Build Succeeded: 64 steps - 1.13/s
```

### Test Run Output — Test 1 (CRASHED ✅ Proves Bug)
```
$ out/release_x64/blink_unittests --gtest_filter="*NoCrashWhenFocusChangedAfterContextDestroyed"

[==========] Running 1 test from 1 test suite.
[ RUN      ] ClipboardChangeEventTest.NoCrashWhenFocusChangedAfterContextDestroyed
Received signal 11 SEGV_MAPERR 000000000234
#4 blink::LocalFrame::PagePopupOwner()
#5 blink::ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent()
#6 blink::ClipboardChangeEventTest_NoCrashWhenFocusChangedAfterContextDestroyed_Test::TestBody()
    [clipboard_change_event_controller_unittest.cc:289]
[1/1] ClipboardChangeEventTest.NoCrashWhenFocusChangedAfterContextDestroyed (CRASHED)
```

**Analysis**: The crash occurs at `MaybeDispatchClipboardChangeEvent()` when
`GetExecutionContext()` returns null after `FrameDestroyed()`. The SEGV at address
0x234 (= 564 decimal) is the offset of a member within `LocalDOMWindow` being
dereferenced through a null pointer from `*To<LocalDOMWindow>(nullptr)`.

### Test Run Output — Test 2 (PASSED ✅ Existing Guard)
```
$ out/release_x64/blink_unittests --gtest_filter="*NoCrashWhenClipboardChangedAfterContextDestroyed"

[==========] Running 1 test from 1 test suite.
[ RUN      ] ClipboardChangeEventTest.NoCrashWhenClipboardChangedAfterContextDestroyed
[       OK ] (37 ms)
[  PASSED  ] 1 test.
```

**Analysis**: `OnClipboardChanged()` at line 78 already has `if (!context) return;`.
This test confirms the existing guard works and serves as a regression guard.

### Test Run Output — Test 3 (CRASHED ✅ Proves Bug via Real Teardown Path)
```
$ out/release_x64/blink_unittests --gtest_filter="*NoCrashDuringTeardownWithPendingFocusDispatch"

[==========] Running 1 test from 1 test suite.
[ RUN      ] ClipboardChangeEventTest.NoCrashDuringTeardownWithPendingFocusDispatch
Received signal 11 SEGV_MAPERR 000000000234
#5 blink::ClipboardChangeEventController::MaybeDispatchClipboardChangeEvent()
#6 blink::FocusController::NotifyFocusChangedObservers()
#7 blink::FocusController::SetFocusedFrame()
#8 blink::Frame::Detach()
#9 blink::Page::WillBeDestroyed()
#10 blink::DummyPageHolder::~DummyPageHolder()
#11 blink::PageTestBase::TearDown()
[1/1] ClipboardChangeEventTest.NoCrashDuringTeardownWithPendingFocusDispatch (CRASHED)
```

**Analysis**: This test reproduces the EXACT crash path from the bug report:
`Page::WillBeDestroyed()` → `Frame::Detach()` → `DomWindow()->FrameDestroyed()`
(destroying the execution context) → `FocusController::FrameDetached()` →
`SetFocusedFrame(nullptr)` → `NotifyFocusChangedObservers()` →
`ClipboardChangeEventController::FocusedFrameChanged()` →
`MaybeDispatchClipboardChangeEvent()` → CRASH. This is identical to the real-world
crash stack trace from the bug report.

### Existing Tests (All Still Pass ✅)
```
$ out/release_x64/blink_unittests --gtest_filter="*ClipboardChangeEventFires*:*ClipboardChangeEventNotFired*:*StickyActivation*"

[  PASSED  ] 5 tests.
  ClipboardChangeEventTest.ClipboardChangeEventFiresWhenFocused (36 ms)
  ClipboardChangeEventTest.ClipboardChangeEventNotFiredWhenNotFocused (30 ms)
  ClipboardChangeEventTest.ClipboardChangeEventFiresWithStickyActivation (31 ms)
  ClipboardChangeEventTest.ClipboardChangeEventNotFiredWithoutStickyActivationOrPermission (31 ms)
  ClipboardChangeEventTest.StickyActivationTakesPrecedenceOverPermissionCheck (32 ms)
```

## Key Technical Insight

The root cause of the crash is that `ExecutionContextClient::GetExecutionContext()` returns null
when `IsContextDestroyed()` is true:

```cpp
ExecutionContext* ExecutionContextClient::GetExecutionContext() const {
  return execution_context_ && !execution_context_->IsContextDestroyed()
             ? execution_context_.Get()
             : nullptr;
}
```

`IsContextDestroyed()` becomes true when `LocalDOMWindow::FrameDestroyed()` calls
`NotifyContextDestroyed()`. In `Frame::Detach()`, `FrameDestroyed()` is called BEFORE
`FocusController::FrameDetached()`, so by the time `FocusedFrameChanged()` is called
on the controller, `GetExecutionContext()` already returns null.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL (CRASH) on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior (null dereference at MaybeDispatchClipboardChangeEvent)
- [x] Tests cover the main bug scenario (Test 1: direct call after FrameDestroyed)
- [x] Tests cover the real-world crash path (Test 3: natural teardown via Page::WillBeDestroyed)
- [x] Tests cover the existing guard (Test 2: OnClipboardChanged already handles null)
- [x] Test code follows Chromium style guidelines
- [x] Existing tests continue to pass

## Next Steps
Once the fix is implemented (Stage 6), these tests should PASS:
1. Add `if (!context) return;` in `MaybeDispatchClipboardChangeEvent()` — fixes Tests 1 and 3
2. Add `if (!context) return nullptr;` in `GetSystemClipboard()` — fixes secondary crash path
3. Add `if (!GetExecutionContext()) return;` in `FocusedFrameChanged()` — belt-and-suspenders guard
