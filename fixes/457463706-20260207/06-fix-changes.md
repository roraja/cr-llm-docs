# Fix Implementation: 457463706

## Summary
This fix replaces the `PlatformEventDispatcher`/`PlatformEventController` pattern with a simpler `ClipboardChangeObserver` interface for dispatching clipboard change events. The change decouples `SystemClipboard` from the generic event dispatch pattern, moves clipboard data storage from `SystemClipboard` to `ClipboardChangeEventController`, and establishes direct data flow from Mojo callbacks to the event controller without intermediate storage.

## Branch
- **Branch name**: `fix/457463706_FIXED_TESTPASS`
- **Base commit**: c19ce9c8b51ed741cab836dbe33ad7cdc081cfec
- **Fix commit**: 225f9e30b9fa33333f546a11c496f74d49d1b1b3

## Files Modified

### 1. [/workspace/cr4/src/third_party/blink/renderer/core/clipboard/clipboard_change_observer.h](/workspace/cr4/src/third_party/blink/renderer/core/clipboard/clipboard_change_observer.h) (NEW FILE)

**Lines**: 1-32

**Content**:
```cpp
// Copyright 2025 The Chromium Authors
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

#ifndef THIRD_PARTY_BLINK_RENDERER_CORE_CLIPBOARD_CLIPBOARD_CHANGE_OBSERVER_H_
#define THIRD_PARTY_BLINK_RENDERER_CORE_CLIPBOARD_CLIPBOARD_CHANGE_OBSERVER_H_

#include "third_party/blink/renderer/core/core_export.h"
#include "third_party/blink/renderer/platform/bindings/bigint.h"
#include "third_party/blink/renderer/platform/heap/garbage_collected.h"
#include "third_party/blink/renderer/platform/wtf/text/wtf_string.h"
#include "third_party/blink/renderer/platform/wtf/vector.h"

namespace blink {

// Observer interface for clipboard change notifications.
// Observers receive direct callbacks with clipboard change data.
class CORE_EXPORT ClipboardChangeObserver : public GarbageCollectedMixin {
 public:
  // Called when the system clipboard data has changed.
  // |types| contains the MIME types available on the clipboard.
  // |change_id| is a unique identifier for this clipboard state.
  virtual void OnClipboardDataChanged(const Vector<String>& types,
                                      const BigInt& change_id) = 0;

 protected:
  virtual ~ClipboardChangeObserver() = default;
};

}  // namespace blink

#endif  // THIRD_PARTY_BLINK_RENDERER_CORE_CLIPBOARD_CLIPBOARD_CHANGE_OBSERVER_H_
```

**Rationale**: New observer interface that follows established patterns in Blink (similar to `FocusChangedObserver`), passing clipboard change data directly to observers rather than requiring a fetch-back pattern.

---

### 2. [/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard.h](/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard.h)

**Lines changed**: 8-21, 39-68, 124, 247-250

**Before**:
```cpp
class CORE_EXPORT SystemClipboard final
    : public GarbageCollected<SystemClipboard>,
      public PlatformEventDispatcher,
      public mojom::blink::ClipboardListener {
  // ...
  void StartListening(LocalDOMWindow*) override;
  void StopListening() override;
  // ...
  struct ClipboardChangeData {
    const Vector<String> types;
    const BigInt change_id;
  };
  const ClipboardChangeData& GetClipboardChangeEventData();
  // ...
  std::optional<ClipboardChangeData> clipboard_change_data_;
```

**After**:
```cpp
class CORE_EXPORT SystemClipboard final
    : public GarbageCollected<SystemClipboard>,
      public mojom::blink::ClipboardListener {
  // ...
  void SetClipboardChangeObserver(
      ClipboardChangeObserver* observer,
      scoped_refptr<base::SequencedTaskRunner> task_runner);
  // ...
  WeakMember<ClipboardChangeObserver> clipboard_change_observer_;
```

**Rationale**: 
1. Remove `PlatformEventDispatcher` inheritance - no longer needed
2. Remove `ClipboardChangeData` struct and `clipboard_change_data_` storage - data now passed directly to observer
3. Remove `GetClipboardChangeEventData()` method - replaced with direct observer callback
4. Add `SetClipboardChangeObserver()` for direct observer registration
5. Use `WeakMember` for observer to allow garbage collection

---

### 3. [/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard.cc](/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard.cc)

**Lines changed**: 458-462, 638-667

**Before**:
```cpp
void SystemClipboard::Trace(Visitor* visitor) const {
  PlatformEventDispatcher::Trace(visitor);
  visitor->Trace(clipboard_);
  visitor->Trace(clipboard_listener_receiver_);
}
// ...
void SystemClipboard::OnClipboardDataChanged(const Vector<String>& types,
                                             const absl::uint128& change_id) {
  clipboard_change_data_.emplace(types, BigInt(change_id));
  // TODO(crbug.com/457463706): Reevaluate whether this is the right
  // abstraction, possibly use a clipboard-specific interface here.
  NotifyControllers();
}

const SystemClipboard::ClipboardChangeData&
SystemClipboard::GetClipboardChangeEventData() {
  CHECK(!!clipboard_change_data_);
  return clipboard_change_data_.value();
}

void SystemClipboard::StartListening(LocalDOMWindow* window) {
  if (!clipboard_listener_receiver_.is_bound() && clipboard_.is_bound()) {
    clipboard_->RegisterClipboardListener(
        clipboard_listener_receiver_.BindNewPipeAndPassRemote(
            window->GetTaskRunner(TaskType::kUserInteraction)));
  }
}

void SystemClipboard::StopListening() {
  clipboard_listener_receiver_.reset();
}
```

**After**:
```cpp
void SystemClipboard::Trace(Visitor* visitor) const {
  visitor->Trace(clipboard_);
  visitor->Trace(clipboard_listener_receiver_);
  visitor->Trace(clipboard_change_observer_);
}
// ...
void SystemClipboard::SetClipboardChangeObserver(
    ClipboardChangeObserver* observer,
    scoped_refptr<base::SequencedTaskRunner> task_runner) {
  clipboard_change_observer_ = observer;
  if (observer) {
    if (!clipboard_listener_receiver_.is_bound() && clipboard_.is_bound()) {
      DCHECK(task_runner);
      clipboard_->RegisterClipboardListener(
          clipboard_listener_receiver_.BindNewPipeAndPassRemote(
              std::move(task_runner)));
    }
  } else {
    clipboard_listener_receiver_.reset();
  }
}

void SystemClipboard::OnClipboardDataChanged(const Vector<String>& types,
                                             const absl::uint128& change_id) {
  if (clipboard_change_observer_) {
    clipboard_change_observer_->OnClipboardDataChanged(types,
                                                       BigInt(change_id));
  }
}
```

**Rationale**: 
1. Replace `StartListening()`/`StopListening()` with unified `SetClipboardChangeObserver()` method
2. Remove `NotifyControllers()` call - directly call observer's method instead
3. Pass data directly to observer, eliminating intermediate storage
4. Trace the new `clipboard_change_observer_` member

---

### 4. [/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.h](/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.h)

**Lines changed**: 11, 23-74

**Before**:
```cpp
#include "third_party/blink/renderer/core/frame/platform_event_controller.h"
// ...
class MODULES_EXPORT ClipboardChangeEventController final
    : public GarbageCollected<ClipboardChangeEventController>,
      public Supplement<Navigator>,
      public PlatformEventController,
      public FocusChangedObserver {
  // ...
  void DidUpdateData() override;
  void RegisterWithDispatcher() override;
  void UnregisterWithDispatcher() override;
  bool HasLastData() override;
```

**After**:
```cpp
#include "third_party/blink/renderer/core/clipboard/clipboard_change_observer.h"
// ...
class MODULES_EXPORT ClipboardChangeEventController final
    : public GarbageCollected<ClipboardChangeEventController>,
      public Supplement<Navigator>,
      public ClipboardChangeObserver,
      public FocusChangedObserver {
  // ...
  ~ClipboardChangeEventController() override;
  void OnClipboardDataChanged(const Vector<String>& types,
                              const BigInt& change_id) override;
  void StartListening();
  void StopListening();
  // ...
  struct ClipboardChangeData {
    Vector<String> types;
    BigInt change_id;
  };
  std::optional<ClipboardChangeData> pending_clipboard_data_;
  bool is_listening_ = false;
```

**Rationale**: 
1. Replace `PlatformEventController` inheritance with `ClipboardChangeObserver`
2. Add explicit `StartListening()`/`StopListening()` methods
3. Store `pending_clipboard_data_` locally (was in `SystemClipboard`)
4. Add destructor for cleanup
5. Add `is_listening_` flag to track registration state

---

### 5. [/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc](/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller.cc)

**Lines changed**: 20-98, 116-159

**Before**:
```cpp
ClipboardChangeEventController::ClipboardChangeEventController(
    Navigator& navigator,
    EventTarget* event_target)
    : Supplement<Navigator>(navigator),
      PlatformEventController(*navigator.DomWindow()),
      FocusChangedObserver(navigator.DomWindow()->GetFrame()->GetPage()),
      event_target_(event_target) {}

void ClipboardChangeEventController::DidUpdateData() {
  OnClipboardChanged();
}

bool ClipboardChangeEventController::HasLastData() {
  return true;
}

void ClipboardChangeEventController::RegisterWithDispatcher() {
  SystemClipboard* clipboard = GetSystemClipboard();
  if (clipboard) {
    clipboard->AddController(this, GetSupplementable()->DomWindow());
  }
}

void ClipboardChangeEventController::UnregisterWithDispatcher() {
  SystemClipboard* clipboard = GetSystemClipboard();
  if (clipboard) {
    clipboard->RemoveController(this);
  }
}

void ClipboardChangeEventController::Trace(Visitor* visitor) const {
  Supplement<Navigator>::Trace(visitor);
  PlatformEventController::Trace(visitor);
  FocusChangedObserver::Trace(visitor);
  visitor->Trace(event_target_);
}

void ClipboardChangeEventController::DispatchClipboardChangeEvent() {
  SystemClipboard* clipboard = GetSystemClipboard();
  if (!clipboard) {
    return;
  }
  const auto& clipboardchange_data = clipboard->GetClipboardChangeEventData();
  event_target_->DispatchEvent(*ClipboardChangeEvent::Create(
      clipboardchange_data.types, clipboardchange_data.change_id));
  // ...
}
```

**After**:
```cpp
ClipboardChangeEventController::ClipboardChangeEventController(
    Navigator& navigator,
    EventTarget* event_target)
    : Supplement<Navigator>(navigator),
      FocusChangedObserver(navigator.DomWindow()->GetFrame()->GetPage()),
      event_target_(event_target) {}

ClipboardChangeEventController::~ClipboardChangeEventController() {
  StopListening();
}

void ClipboardChangeEventController::OnClipboardDataChanged(
    const Vector<String>& types,
    const BigInt& change_id) {
  pending_clipboard_data_.emplace(ClipboardChangeData{types, change_id});
  // ...
  MaybeDispatchClipboardChangeEvent();
}

void ClipboardChangeEventController::StartListening() {
  if (is_listening_) {
    return;
  }
  SystemClipboard* clipboard = GetSystemClipboard();
  ExecutionContext* context = GetExecutionContext();
  if (clipboard && context) {
    clipboard->SetClipboardChangeObserver(
        this, context->GetTaskRunner(TaskType::kUserInteraction));
    is_listening_ = true;
  }
}

void ClipboardChangeEventController::StopListening() {
  if (!is_listening_) {
    return;
  }
  SystemClipboard* clipboard = GetSystemClipboard();
  if (clipboard) {
    clipboard->SetClipboardChangeObserver(nullptr, nullptr);
  }
  is_listening_ = false;
}

void ClipboardChangeEventController::Trace(Visitor* visitor) const {
  Supplement<Navigator>::Trace(visitor);
  FocusChangedObserver::Trace(visitor);
  visitor->Trace(event_target_);
}

void ClipboardChangeEventController::DispatchClipboardChangeEvent() {
  if (!pending_clipboard_data_) {
    return;
  }
  event_target_->DispatchEvent(*ClipboardChangeEvent::Create(
      pending_clipboard_data_->types, pending_clipboard_data_->change_id));
  // ...
}
```

**Rationale**: 
1. Remove `PlatformEventController` base class initialization and methods
2. Implement `ClipboardChangeObserver::OnClipboardDataChanged()` to receive data directly
3. Store clipboard data locally in `pending_clipboard_data_`
4. Add destructor to unregister observer
5. Simplify `DispatchClipboardChangeEvent()` to use local data
6. Add null checks in `GetSystemClipboard()` for safety

---

### 6. [/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard.cc](/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard.cc)

**Lines changed**: Minor update to call `StartListening()` when adding event listener

**Rationale**: Integrate with new observer pattern when clipboard change listeners are added.

---

### 7. [/workspace/cr4/src/third_party/blink/renderer/core/clipboard/build.gni](/workspace/cr4/src/third_party/blink/renderer/core/clipboard/build.gni)

**Lines changed**: Added `clipboard_change_observer.h` to source list

**Rationale**: Include the new header file in the build.

---

### 8. [/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc](/workspace/cr4/src/third_party/blink/renderer/core/clipboard/system_clipboard_test.cc)

**Lines changed**: Updated tests to use new observer pattern

**Rationale**: Tests now use `SetClipboardChangeObserver()` instead of `AddController()`.

---

### 9. [/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc](/workspace/cr4/src/third_party/blink/renderer/modules/clipboard/clipboard_change_event_controller_unittest.cc)

**Lines changed**: 31-509

**Changes**:
- Updated existing tests to call `OnClipboardDataChanged()` directly instead of `DidUpdateData()`
- Enabled new TDD tests: `Bug457463706_DirectObserverCallback`, `Bug457463706_ObserverRegistration`
- All tests now use the new observer pattern

**Rationale**: Tests now exercise the new direct observer pattern.

## Git Diff Summary
```
$ git diff --stat
 third_party/blink/renderer/core/clipboard/build.gni                         |   1 +
 third_party/blink/renderer/core/clipboard/clipboard_change_observer.h       |  32 ++++
 third_party/blink/renderer/core/clipboard/system_clipboard.cc               |  47 +++---
 third_party/blink/renderer/core/clipboard/system_clipboard.h                |  30 ++--
 third_party/blink/renderer/core/clipboard/system_clipboard_test.cc          |  87 ++++++-----
 third_party/blink/renderer/modules/clipboard/clipboard.cc                   |   4 +-
 .../blink/renderer/modules/clipboard/clipboard_change_event_controller.cc   |  73 +++++----
 .../blink/renderer/modules/clipboard/clipboard_change_event_controller.h    |  30 ++--
 .../modules/clipboard/clipboard_change_event_controller_unittest.cc         | 288 ++++++++++++++++++++++++++++++++----
 9 files changed, 438 insertions(+), 154 deletions(-)
```

## Build Result
```
$ autoninja -C out/release_x64 blink_unittests
ninja: Entering directory `out/release_x64'
ninja: no work to do.
```

**Status**: ✅ BUILD SUCCESS

## Quick Test Result
```
$ out/release_x64/blink_unittests --gtest_filter="ClipboardChangeEventTest*"
[==========] Running 10 tests from 1 test suite.
[----------] 10 tests from ClipboardChangeEventTest
[ RUN      ] ClipboardChangeEventTest.ClipboardChangeEventFiresWhenFocused
[       OK ] ClipboardChangeEventTest.ClipboardChangeEventFiresWhenFocused (38 ms)
[ RUN      ] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWhenNotFocused
[       OK ] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWhenNotFocused (37 ms)
[ RUN      ] ClipboardChangeEventTest.ClipboardChangeEventFiresWithStickyActivation
[       OK ] ClipboardChangeEventTest.ClipboardChangeEventFiresWithStickyActivation (54 ms)
[ RUN      ] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWithoutStickyActivationOrPermission
[       OK ] ClipboardChangeEventTest.ClipboardChangeEventNotFiredWithoutStickyActivationOrPermission (51 ms)
[ RUN      ] ClipboardChangeEventTest.StickyActivationTakesPrecedenceOverPermissionCheck
[       OK ] ClipboardChangeEventTest.StickyActivationTakesPrecedenceOverPermissionCheck (50 ms)
[ RUN      ] ClipboardChangeEventTest.Bug457463706_DirectObserverCallback
[       OK ] ClipboardChangeEventTest.Bug457463706_DirectObserverCallback (50 ms)
[ RUN      ] ClipboardChangeEventTest.Bug457463706_ObserverRegistration
[       OK ] ClipboardChangeEventTest.Bug457463706_ObserverRegistration (40 ms)
[ RUN      ] ClipboardChangeEventTest.Bug457463706_EventContainsCorrectTypes
[       OK ] ClipboardChangeEventTest.Bug457463706_EventContainsCorrectTypes (35 ms)
[ RUN      ] ClipboardChangeEventTest.Bug457463706_MultipleChangesUsesLatestData
[       OK ] ClipboardChangeEventTest.Bug457463706_MultipleChangesUsesLatestData (36 ms)
[ RUN      ] ClipboardChangeEventTest.Bug457463706_DataFlowThroughController
[       OK ] ClipboardChangeEventTest.Bug457463706_DataFlowThroughController (34 ms)
[----------] 10 tests from ClipboardChangeEventTest (439 ms total)
[==========] 10 tests from 1 test suite ran. (452 ms total)
[  PASSED  ] 10 tests.
```

**Status**: ✅ TESTS PASS

## Implementation Notes
- The fix follows the design documented in `04-architecture-lld.md`
- Data flow is now direct: Mojo callback → SystemClipboard → Observer → Event dispatch
- No intermediate storage in `SystemClipboard` - data passed directly to observer
- `WeakMember` used for observer to allow garbage collection
- Destructor ensures cleanup when controller is destroyed
- TDD tests (`Bug457463706_DirectObserverCallback`, `Bug457463706_ObserverRegistration`) are now enabled and passing

## Known Issues
- None identified

## Next Steps
- Full build verification (Stage 7)
- Complete test suite (Stage 7)
- Code review (Stage 8)
