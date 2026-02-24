# Code Review: CL 7602194 — [Clipboard] Make Clipboard a singleton with thread affinity

**CL URL:** https://chromium-review.googlesource.com/c/chromium/src/+/7602194
**Author:** Hewro Hewei (ihewro@chromium.org)
**Reviewer:** Sky Malice (skym@chromium.org) — CR+1
**Files Changed:** 6 files, +124/−95 lines

---

## Review Summary

| Category    | Status | Notes |
|-------------|--------|-------|
| Correctness | ⚠️     | Data race in `ValidateThreadID()` on concurrent first access; redundant two-step destroy in `TestClipboard::Create()` |
| Style       | ✅     | Clean, follows Chromium conventions, good DEPRECATED annotations |
| Security    | ✅     | No new attack surface; thread-affinity enforcement is an improvement |
| Performance | ✅     | Removes lock contention and per-thread map lookups |
| Testing     | ⚠️     | Existing tests updated, but no new test explicitly validates the thread-affinity CHECK |

---

## Detailed Findings

#### Issue #1: Data race in `ValidateThreadID()` on concurrent first-call
**Severity**: Major
**File**: `ui/base/clipboard/clipboard.cc`
**Line**: 369–383
**Description**:
`ValidateThreadID()` uses plain (non-atomic) function-local statics `bound_tid` and `is_bound` to enforce single-thread access:

```cpp
static base::PlatformThreadId bound_tid;
static bool is_bound = false;

const base::PlatformThreadId tid = base::PlatformThread::CurrentId();
if (!is_bound) {
    bound_tid = tid;
    is_bound = true;
    return;
}
CHECK_EQ(bound_tid, tid);
```

While C++11 guarantees thread-safe *initialization* of function-local statics (i.e., zero-init of `bound_tid` and `false`-init of `is_bound`), subsequent reads and writes (`if (!is_bound)`, `is_bound = true`, `bound_tid = tid`) are **not synchronized**. If two threads call `ValidateThreadID()` concurrently before the clipboard is bound, both could see `is_bound == false`, both set `bound_tid` to their own TID, and neither triggers the `CHECK_EQ`. This defeats the purpose of the thread-affinity guard in the exact scenario it is designed to catch.

The old code used `ClipboardMapLock()` to synchronize concurrent access. Removing the lock without adding atomic operations introduces a data race that is undefined behavior per the C++ standard.

**Suggestion**: Use `std::atomic<bool>` and `std::atomic<base::PlatformThreadId>` with appropriate memory ordering, or use a `base::Lock` guarding the binding logic (only the bind path, not every call). A simpler approach:

```cpp
void Clipboard::ValidateThreadID() {
  static std::atomic<base::PlatformThreadId> bound_tid{0};
  const base::PlatformThreadId tid = base::PlatformThread::CurrentId();

  base::PlatformThreadId expected = 0;
  if (bound_tid.compare_exchange_strong(expected, tid,
                                         std::memory_order_relaxed)) {
    return;  // Successfully bound to this thread.
  }
  CHECK_EQ(expected, tid);
}
```

This eliminates the second static and ensures the binding is atomic.

---

#### Issue #2: Redundant two-step destroy in `TestClipboard::Create()`
**Severity**: Minor
**File**: `ui/base/clipboard/test/test_clipboard.cc`
**Line**: 62–63
**Description**:
```cpp
std::unique_ptr<Clipboard> existing = Clipboard::Take();
existing.reset();
```
`Take()` returns the singleton; then `existing.reset()` destroys it. The explicit `.reset()` is unnecessary — the `unique_ptr` would be destroyed at end of scope anyway. Alternatively, this entire block could be replaced by a single call to `Clipboard::DestroyClipboard()`, which is more readable and self-documenting:

```cpp
Clipboard::DestroyClipboard();
```

Or even simpler, since the returned value is unused:
```cpp
std::ignore = Clipboard::Take();
```

**Suggestion**: Replace with `Clipboard::DestroyClipboard()` for clarity and consistency with the new API being introduced in this CL.

---

#### Issue #3: `IsSupportedClipboardBuffer()` still calls deprecated `GetForCurrentThread()`
**Severity**: Suggestion
**File**: `ui/base/clipboard/clipboard.cc`
**Line**: 57
**Description**:
The `IsSupportedClipboardBuffer()` static method still calls `ui::Clipboard::GetForCurrentThread()` which is now deprecated. Since this CL introduces `Get()` as the preferred replacement and the commit message lists `GetForCurrentThread` migration as a follow-up, this is not blocking, but could be addressed in this CL trivially:

```cpp
// Before:
ui::Clipboard* clipboard = ui::Clipboard::GetForCurrentThread();
// After:
ui::Clipboard* clipboard = ui::Clipboard::Get();
```

**Suggestion**: Consider updating this call site in this CL since it's in the same file being modified, to reduce follow-up churn.

---

#### Issue #4: Missing test for thread-affinity enforcement
**Severity**: Minor
**File**: `ui/base/clipboard/test/test_clipboard_unittest.cc`
**Line**: N/A
**Description**:
The CL adds a `CHECK_EQ(bound_tid, tid)` to enforce thread affinity, but no test validates that accessing the clipboard from a wrong thread triggers the CHECK (e.g., via `EXPECT_CHECK_DEATH`). While this is a defensive mechanism and difficult to test in all scenarios, a simple death test would verify the mechanism works:

```cpp
TEST(ClipboardDeathTest, AccessFromWrongThreadCrashes) {
  Clipboard::Get();  // Bind to current thread.
  base::Thread other("other");
  other.Start();
  EXPECT_CHECK_DEATH(other.task_runner()->PostTask(
      FROM_HERE, base::BindOnce([] { Clipboard::Get(); })));
}
```

**Suggestion**: Add a death test in a follow-up CL to validate the thread-affinity CHECK.

---

#### Issue #5: `SetClipboardForCurrentThread()` behavioral change
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard.cc`
**Line**: 101–105
**Description**:
`SetClipboardForCurrentThread()` now delegates to `SetClipboardForOverride()`, which has a `DCHECK(!GlobalClipboard())`. The old per-thread implementation allowed each thread to have its own clipboard entry without conflicting. With the singleton model, calling `SetClipboardForCurrentThread()` when a clipboard already exists will DCHECK-fail. Callers that previously relied on setting a clipboard after `GetForCurrentThread()` lazily created one will now crash in debug builds.

The commit message lists this API as deprecated for incremental migration, which is appropriate. This behavioral change is intentional but callers of `SetClipboardForCurrentThread` should be audited.

**Suggestion**: Ensure all existing callers of `SetClipboardForCurrentThread` have been audited for this narrower precondition. The `TestClipboard::Create()` correctly handles this by calling `Take()` first, but production callers may not.

---

#### Issue #6: `ValidateThreadID` statics are never reset
**Severity**: Minor
**File**: `ui/base/clipboard/clipboard.cc`
**Line**: 369–383
**Description**:
The `bound_tid` and `is_bound` statics are never reset. After `DestroyClipboard()`, the clipboard singleton is destroyed but the thread binding persists for the lifetime of the process. If a test destroys and recreates the clipboard, or if clipboard teardown and recreation happens during shutdown/restart scenarios, the binding from the first initialization permanently pins clipboard access to that thread.

In tests, `TestClipboard::Create()` calls `Take()` + `SetClipboardForOverride()`, and all these call `ValidateThreadID()`, which will CHECK that they happen on the originally bound thread. This is fine for normal test runs but could be problematic for tests that need to create clipboards on different threads, or if the thread binding outlives what's expected.

**Suggestion**: Consider adding a mechanism (perhaps test-only) to reset the thread binding when the clipboard is destroyed. This could be a private static method `ResetThreadBindingForTesting()` called from `DestroyClipboard()` or from a test-only helper.

---

## Positive Observations

- **Well-structured incremental migration**: The CL introduces new singleton APIs (`Get()`, `Take()`, `DestroyClipboard()`, `OnPreShutdownClipboard()`, `SetClipboardForOverride()`) while keeping old APIs as thin wrappers. This is a clean pattern for incremental migration.
- **Clear deprecation annotations**: Every legacy API in the header is marked `// DEPRECATED:` with a pointer to the preferred replacement. This is excellent for discoverability.
- **Meaningful code simplification**: Removing `ClipboardMap`, `ClipboardMapLock()`, `AllowedThreads()`, and the per-thread infrastructure significantly reduces complexity (~30 lines of infrastructure deleted).
- **Good commit message**: The commit message clearly explains the motivation, what was done, and lays out a concrete plan for follow-up CLs with numbered steps.
- **Java test fix**: Moving `cleanupNativeForTesting()` inside `runOnUiThreadBlocking` correctly ensures cleanup happens on the thread where the clipboard was created, matching the new thread-affinity requirement. This also fixed the Patch Set 1 test failure.
- **`TestClipboard::Create()` handles pre-existing singleton**: The comment explaining why `Take()` is needed before `SetClipboardForOverride()` in browser tests is helpful for future maintainers.

---

## Overall Assessment

**Needs minor changes before approval.**

The design is sound and the incremental migration approach is well-executed. The main concern is the data race in `ValidateThreadID()` (Issue #1), which should be addressed before landing — the function uses non-atomic statics without synchronization, creating a window where concurrent first-access from two threads could silently bypass the thread-affinity CHECK. Using `std::atomic` with `compare_exchange_strong` is a minimal fix.

The remaining issues are minor and could be addressed in this CL or in follow-ups at the author's discretion.
