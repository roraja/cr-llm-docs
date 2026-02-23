# TDD Tests: 362302769

## Test Summary

| Test Type | File | Test Name | Status Before Fix |
|-----------|------|-----------|-------------------|
| Web Test | [/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay.html](/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay.html) | Bug 362302769: dragleave from iframe to overlay | ❌ FAILING |

## Tests Created

### Web Tests

#### File: [/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay.html](/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay.html)

**New test file** — Parent document that creates an iframe with a full-size draggable element, positions an overlay element (z-index: 10) over the right half of the iframe, and uses `eventSender` to drag from the non-overlapped left half to the overlapped right half. Verifies that `dragleave` fires inside the iframe when the parent frame's drag target switches from the iframe to the overlay.

```html
<!DOCTYPE html>
<html>
<head>
<style>
body { margin: 0; padding: 0; }
#container { position: relative; width: 400px; height: 400px; }
#iframe { position: absolute; top: 0; left: 0; width: 300px; height: 300px; border: none; }
#overlay {
  position: absolute;
  top: 0;
  left: 150px;
  width: 200px;
  height: 300px;
  background: rgba(255, 0, 0, 0.5);
  z-index: 10;
}
</style>
</head>
<body>
<p>Bug 362302769: Tests that dragleave fires inside an iframe when dragging to an
overlapping parent element. This test requires eventSender.</p>
<div id="container">
  <iframe id="iframe" src="resources/drag-from-iframe-to-overlay-inner.html"></iframe>
  <div id="overlay">Overlay</div>
</div>
<pre id="log"></pre>
<script>
// ... eventSender-based drag simulation from iframe to overlay
// Checks events array for 'dragleave-inner' and 'dragenter-overlay'
// Reports PASS if both present, FAIL otherwise
</script>
</body>
</html>
```

#### File: [/third_party/blink/web_tests/fast/events/resources/drag-from-iframe-to-overlay-inner.html](/third_party/blink/web_tests/fast/events/resources/drag-from-iframe-to-overlay-inner.html)

**New helper file** — Inner iframe content with a full-size draggable div that reports drag events to the parent via `parent.logFromIframe()` (same-origin direct call).

```html
<!DOCTYPE html>
<html style="height: 100%; margin: 0;">
<body style="height: 100%; margin: 0; padding: 0;">
<div id="draggable" draggable="true"
     style="width: 100%; height: 100%; background: lightblue;">
  Drag me
</div>
<script>
var draggableEl = document.getElementById('draggable');
draggableEl.addEventListener('dragstart', function(e) {
  e.dataTransfer.setData('text/plain', 'test');
  if (parent.logFromIframe)
    parent.logFromIframe('dragstart-inner');
});
draggableEl.addEventListener('dragleave', function(e) {
  if (parent.logFromIframe)
    parent.logFromIframe('dragleave-inner');
});
// ... other event listeners
</script>
</body>
</html>
```

#### File: [/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay-expected.txt](/third_party/blink/web_tests/fast/events/drag-from-iframe-to-overlay-expected.txt)

**Expected output** (after fix):

```
Bug 362302769: Tests that dragleave fires inside an iframe when dragging to an
overlapping parent element. This test requires eventSender.

Overlay
dragstart-inner
dragleave-inner
dragenter-overlay
dragover-overlay
dragover-overlay
drop-overlay
dragend-inner

PASS: dragleave fired inside iframe and dragenter fired on overlay
```

## Pre-Fix Test Execution

### Build Output
```
$ autoninja -C out/release_x64 content_shell
3m57.03s Build Succeeded: 2447 steps - 10.32/s
```

### Test Run Output (SHOWS FAILURE — confirms bug exists)
```
$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 --no-retry-failures fast/events/drag-from-iframe-to-overlay.html

[1/1] fast/events/drag-from-iframe-to-overlay.html failed unexpectedly (text diff)
0 tests ran as expected, 1 didn't:
    fast/events/drag-from-iframe-to-overlay.html
```

### Actual Output (pre-fix):
```
Bug 362302769: Tests that dragleave fires inside an iframe when dragging to an
overlapping parent element. This test requires eventSender.

Overlay
dragstart-inner
dragenter-overlay
dragover-overlay
dragover-overlay
drop-overlay
dragend-inner

FAIL: dragleave did NOT fire inside iframe (Bug 362302769)
```

**Key observation**: The actual output shows `dragstart-inner` fires (drag begins in iframe), `dragenter-overlay` fires (overlay receives enter), but **NO `dragleave-inner`** — the iframe's draggable element is never notified that the drag left it. This directly demonstrates bug 362302769.

### Existing Test Validation (no regression)
```
$ python3 third_party/blink/tools/run_web_tests.py -t release_x64 --no-retry-failures fast/events/drag-in-frames.html

The test ran as expected.
```

## Test Design Rationale

The test is specifically designed to trigger the exact bug condition:

1. **Full-size draggable in iframe**: The draggable `<div>` fills the entire iframe (100% width/height). This ensures that when the cursor moves to the overlay position, the **iframe's own hit test** still finds the same draggable element (`drag_target_ == new_target` in the child frame).

2. **Overlay partially covering iframe**: The overlay (z-index: 10) covers the right half of the iframe. This means the **parent frame's hit test** switches from the iframe element to the overlay element when the cursor moves right.

3. **Drag from non-overlapped to overlapped area**: The drag starts at x=50 (left half, no overlay) and moves to x=250 (right half, under overlay). This triggers the parent's `UpdateDragAndDrop()` target-changed branch where `drag_target_` (iframe) ≠ `new_target` (overlay).

4. **The bug**: At line 1438-1440 of `event_handler.cc`, the parent calls `UpdateDragAndDrop()` on the child frame instead of `CancelDragAndDrop()`. Since the child frame's hit test still finds the same element, it dispatches `dragover` instead of `dragleave`.

## TDD Verification Checklist

- [x] Tests were created BEFORE implementing the fix
- [x] Tests correctly FAIL on the current (unfixed) code
- [x] Test failures accurately demonstrate the bug behavior
- [x] Tests cover the main bug scenario (drag from iframe to overlapping parent element)
- [x] Tests cover the correct event sequence (dragstart → dragleave → dragenter → dragover → drop → dragend)
- [x] Test code follows Chromium web test conventions (dumpAsText, eventSender, testRunner)
- [x] Existing drag-in-frames test still passes (no regression risk)

## Next Steps
Once the fix is implemented (Stage 6) — changing `UpdateDragAndDrop()` to `CancelDragAndDrop()` at line 1438-1440 of `event_handler.cc` — this test should PASS because:
1. `CancelDragAndDrop()` will fire `dragleave` on the child frame's current drag target
2. `CancelDragAndDrop()` will call `ClearDragState()` to reset the child frame's drag state
3. The `dragleave-inner` event will appear in the log, matching the expected output
