# High-Level Design: 474131935

## 1. Executive Summary

The Async Clipboard API (`navigator.clipboard.read()` / `readText()`) in Chromium blocks the renderer main thread because the underlying mojom IPC calls between the renderer and browser processes are marked `[Sync]`. On the browser side, the `ui::Clipboard` read operations perform blocking OS clipboard calls on the UI thread. When a source application (e.g., Microsoft Excel) uses delayed clipboard writes, the OS call blocks until the source app finishes generating data, freezing the entire browser. The fix involves converting the synchronous mojom read methods to asynchronous ones for the Async Clipboard API code path, while preserving synchronous variants for legacy callers (DataTransfer/paste events).

## 2. System Context

### 2.1 Overview

The Clipboard subsystem in Chromium spans four architectural layers:

1. **Web Platform Layer** – The `Clipboard` interface exposed to JavaScript via `navigator.clipboard`, returning Promises.
2. **Blink Renderer Layer** – `ClipboardPromise` orchestrates permission checks and data reads; `ClipboardReader` subclasses convert clipboard data to Blobs; `SystemClipboard` wraps the mojom IPC remote to the browser.
3. **Browser Process Layer** – `ClipboardHostImpl` implements the `ClipboardHost` mojom interface, bridging renderer requests to the platform clipboard.
4. **Platform Abstraction Layer** – `ui::Clipboard` provides OS-agnostic clipboard access, with platform-specific implementations (e.g., `ClipboardWin`, `ClipboardMac`, `ClipboardOzone`).

The bug occurs at the boundary between layers 2 and 3: the mojom IPC interface uses `[Sync]` calls for most read methods, blocking the renderer. On the browser side (layer 3→4), `ui::Clipboard` read methods wrap synchronous OS calls, blocking the UI thread.

### 2.2 Related Specs
- [W3C Clipboard API and events](https://w3c.github.io/clipboard-apis/) – `read()` and `readText()` return Promises, expected to be non-blocking
- [crbug.com/443355](https://crbug.com/443355) – "Synchronous clipboard reads are deprecated" – confirms the long-term direction of removing `[Sync]`
- [crbug.com/474131935](https://issues.chromium.org/issues/474131935) – This bug report
- [crbug.com/815425](https://crbug.com/815425) – Related: clipboard lock contention with rdpclip.exe

## 3. Component Architecture

### 3.1 Major Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `Clipboard` (Web API) | `/third_party/blink/renderer/modules/clipboard/clipboard.{h,cc}` | Exposes `navigator.clipboard.read()`/`readText()`/`write()`/`writeText()` to JavaScript; returns ScriptPromise |
| `ClipboardPromise` | `/third_party/blink/renderer/modules/clipboard/clipboard_promise.{h,cc}` | Orchestrates permission checks, coordinates format enumeration and per-format reads, resolves/rejects the JS Promise |
| `ClipboardReader` | `/third_party/blink/renderer/modules/clipboard/clipboard_reader.{h,cc}` | Base class + per-format subclasses (PNG, Text, HTML, SVG, RTF, Custom) that read clipboard data and encode it into Blobs |
| `SystemClipboard` | `/third_party/blink/renderer/core/clipboard/system_clipboard.{h,cc}` | Renderer-side wrapper around `mojom::blink::ClipboardHost` mojo remote; provides both sync and async read/write methods |
| `ClipboardHost` (mojom) | `/third_party/blink/public/mojom/clipboard/clipboard.mojom` | Mojo interface definition for renderer↔browser IPC; most read methods marked `[Sync]` |
| `ClipboardHostImpl` | `/content/browser/renderer_host/clipboard_host_impl.{h,cc}` | Browser-side implementation of `ClipboardHost` mojom; runs on the UI thread; delegates to `ui::Clipboard` |
| `ui::Clipboard` | `/ui/base/clipboard/clipboard.{h,cc}` | Platform-agnostic clipboard abstraction; async callback methods wrap synchronous platform calls |
| `ClipboardWin` | `/ui/base/clipboard/clipboard_win.cc` | Windows implementation; calls `::OpenClipboard()`, `::GetClipboardData()`, handles `WM_RENDERFORMAT` |
| `ClipboardMac` | `/ui/base/clipboard/clipboard_mac.mm` | macOS implementation via NSPasteboard |
| `ClipboardOzone` | `/ui/base/clipboard/clipboard_ozone.cc` | Linux/ChromeOS implementation via Ozone (X11/Wayland) |

### 3.2 Component Diagram

```mermaid
graph TB
    subgraph "Web Page"
        JS["JavaScript<br/>navigator.clipboard.read()"]
    end

    subgraph "Renderer Process (Blink)"
        CB["Clipboard<br/>(EventTarget, Web API)"]
        CP["ClipboardPromise<br/>(Orchestration + Permissions)"]
        CR["ClipboardReader<br/>(PNG/Text/HTML/SVG/RTF)"]
        SC["SystemClipboard<br/>(Mojo Remote Wrapper)"]
    end

    subgraph "Mojo IPC Boundary"
        MOJOM["ClipboardHost Mojom<br/>⚠️ [Sync] read methods"]
    end

    subgraph "Browser Process"
        CHI["ClipboardHostImpl<br/>(DocumentService)"]
        UC["ui::Clipboard<br/>(Platform Abstraction)"]
    end

    subgraph "OS Layer"
        WIN["Windows: OpenClipboard / GetClipboardData"]
        MAC["macOS: NSPasteboard"]
        LIN["Linux: X11 / Wayland"]
    end

    JS --> CB
    CB --> CP
    CP --> CR
    CR --> SC
    SC -->|"Sync & Async IPC"| MOJOM
    MOJOM --> CHI
    CHI --> UC
    UC --> WIN
    UC --> MAC
    UC --> LIN

    style MOJOM fill:#ff6b6b,color:#fff
    style WIN fill:#ffa500,color:#fff
```

## 4. Process Architecture

### 4.1 Process Boundaries

Two Chrome processes are involved:

| Process | Thread | Components | Role |
|---------|--------|------------|------|
| **Renderer** | Main thread | Clipboard, ClipboardPromise, ClipboardReader, SystemClipboard | Handles JS API, permission checks, data format conversion |
| **Renderer** | Worker pool | ClipboardReader (encoding) | UTF-8 encoding for text/html/svg (off main thread) |
| **Browser** | UI thread | ClipboardHostImpl, ui::Clipboard | Receives mojom IPC, performs OS clipboard access |

**The problem**: Both the renderer main thread and browser UI thread block during a synchronous clipboard read. The renderer blocks on `[Sync]` mojom IPC, and the browser blocks on OS clipboard APIs (especially `::GetClipboardData()` on Windows when delayed writes are involved).

### 4.2 IPC Flow – Current (Buggy)

```mermaid
sequenceDiagram
    participant JS as JavaScript
    participant CP as ClipboardPromise<br/>(Renderer Main Thread)
    participant SC as SystemClipboard<br/>(Renderer Main Thread)
    participant CHI as ClipboardHostImpl<br/>(Browser UI Thread)
    participant UC as ui::Clipboard<br/>(Browser UI Thread)
    participant OS as OS Clipboard API
    participant App as Source App<br/>(e.g. Excel)

    JS->>CP: navigator.clipboard.read()
    CP->>CP: Check permission (async)
    CP->>SC: ReadAvailableCustomAndStandardFormats(callback)
    Note over SC: ⚠️ Mojom [Sync] — renderer blocks
    SC->>CHI: [Sync] ReadAvailableCustomAndStandardFormats()
    CHI->>UC: ReadAvailableCustomAndStandardFormats()
    UC->>OS: GetClipboardData()
    OS-->>UC: format list
    UC-->>CHI: formats
    CHI-->>SC: response
    Note over SC: Renderer unblocks

    loop For each format (e.g. text/html, image/png)
        CP->>SC: ReadHtml() / ReadPng() / ReadText()
        Note over SC: ⚠️ Mojom [Sync] — renderer blocks AGAIN
        SC->>CHI: [Sync] ReadHtml() / ReadPng()
        CHI->>UC: ReadHTML() / ReadPng()
        UC->>OS: GetClipboardData(format)

        alt Delayed clipboard write (Excel)
            OS->>App: WM_RENDERFORMAT
            Note over OS,App: ⏳ BLOCKS for seconds<br/>while Excel generates data
            App-->>OS: Data written
        end

        OS-->>UC: data bytes
        UC-->>CHI: callback(data)
        CHI-->>SC: response
        Note over SC: Renderer unblocks
    end

    CP-->>JS: Promise resolves with ClipboardItems
```

### 4.3 IPC Flow – Fixed (Async)

```mermaid
sequenceDiagram
    participant JS as JavaScript
    participant CP as ClipboardPromise<br/>(Renderer Main Thread)
    participant SC as SystemClipboard<br/>(Renderer Main Thread)
    participant CHI as ClipboardHostImpl<br/>(Browser UI Thread)
    participant BG as Background Thread<br/>(ThreadPool)
    participant OS as OS Clipboard API
    participant App as Source App

    JS->>CP: navigator.clipboard.read()
    CP->>CP: Check permission (async)
    CP->>SC: ReadAvailableCustomAndStandardFormats(callback)
    Note over SC: ✅ Async mojom — renderer continues
    SC->>CHI: ReadAvailableCustomAndStandardFormats()
    Note over SC: Renderer main thread FREE

    CHI->>BG: PostTask(clipboard read)
    BG->>OS: GetClipboardData()

    alt Delayed clipboard write
        OS->>App: WM_RENDERFORMAT
        Note over OS,App: ⏳ Blocks background thread only
        App-->>OS: Data written
    end

    OS-->>BG: format list
    BG-->>CHI: PostTask(callback with results)
    CHI-->>SC: async response
    SC-->>CP: callback(formats)

    loop For each format
        CP->>SC: ReadHtml(callback) / ReadPng(callback)
        Note over SC: ✅ Async mojom — renderer continues
        SC->>CHI: ReadHtml() / ReadPng()
        CHI->>BG: PostTask(clipboard read)
        BG->>OS: GetClipboardData(format)
        OS-->>BG: data
        BG-->>CHI: PostTask(callback)
        CHI-->>SC: async response
        SC-->>CP: callback(data)
    end

    CP-->>JS: Promise resolves with ClipboardItems
```

## 5. Data Flow

### 5.1 Normal Flow (Expected — Async, Non-Blocking)

```mermaid
flowchart LR
    A["JS: clipboard.read()"] --> B["Permission Check<br/>(async)"]
    B --> C["Enumerate Formats<br/>(async IPC)"]
    C --> D["Read Each Format<br/>(async IPC)"]
    D --> E["Encode to Blob<br/>(background thread)"]
    E --> F["Resolve Promise<br/>(ClipboardItems)"]

    style A fill:#4CAF50,color:#fff
    style F fill:#4CAF50,color:#fff
```

### 5.2 Buggy Flow (Current — Sync, Blocking)

```mermaid
flowchart LR
    A["JS: clipboard.read()"] --> B["Permission Check<br/>(async ✅)"]
    B --> C["Enumerate Formats<br/>⚠️ SYNC IPC BLOCKS"]
    C --> D["Read Each Format<br/>⚠️ SYNC IPC BLOCKS"]
    D --> E["OS Clipboard<br/>⚠️ BLOCKS on delayed write"]
    E --> F["Resolve Promise"]

    style C fill:#ff6b6b,color:#fff
    style D fill:#ff6b6b,color:#fff
    style E fill:#ff6b6b,color:#fff
```

### 5.3 Data Transformation Pipeline

```mermaid
flowchart TB
    subgraph "OS Clipboard"
        RAW["Raw Clipboard Data<br/>(CF_HTML, CF_UNICODETEXT, PNG, etc.)"]
    end

    subgraph "Browser Process"
        READ["ui::Clipboard::Read*()<br/>Raw bytes from OS"]
        SANITIZE["ClipboardHostImpl<br/>Sanitization & validation"]
    end

    subgraph "Renderer Process"
        SC_DATA["SystemClipboard<br/>Receives sanitized data"]
        READER["ClipboardReader<br/>Format-specific processing"]
        ENCODE["Background Thread<br/>UTF-8 encoding"]
        BLOB["Blob creation"]
    end

    subgraph "JavaScript"
        ITEM["ClipboardItem<br/>with typed Blobs"]
    end

    RAW --> READ
    READ --> SANITIZE
    SANITIZE -->|"Mojom IPC"| SC_DATA
    SC_DATA --> READER
    READER --> ENCODE
    ENCODE --> BLOB
    BLOB --> ITEM
```

## 6. Key Interfaces

### 6.1 Public APIs (Web Platform)

- `navigator.clipboard.read(options?)` → `Promise<ClipboardItem[]>` — Reads all available formats as ClipboardItems
- `navigator.clipboard.readText()` → `Promise<string>` — Reads plain text from clipboard
- `navigator.clipboard.write(items)` → `Promise<void>` — Writes ClipboardItems to clipboard
- `navigator.clipboard.writeText(text)` → `Promise<void>` — Writes plain text to clipboard

### 6.2 Mojom Interface (IPC Boundary)

**`ClipboardHost` (clipboard.mojom)** — The critical interface where the bug manifests:

| Method | Sync? | Used by Async API? | Issue |
|--------|-------|--------------------|-------|
| `ReadAvailableTypes()` | `[Sync]` | Yes (format enumeration) | Blocks renderer |
| `ReadAvailableCustomAndStandardFormats()` | `[Sync]` | Yes (format enumeration) | Blocks renderer |
| `ReadText()` | `[Sync]` | Yes (readText path) | Blocks renderer |
| `ReadHtml()` | `[Sync]` | Yes (read path) | Blocks renderer |
| `ReadPng()` | `[Sync]` | Yes (read path) | Blocks renderer |
| `ReadRtf()` | `[Sync]` | Yes (read path) | Blocks renderer |
| `ReadFiles()` | `[Sync]` | Yes (read path) | Blocks renderer |
| `ReadDataTransferCustomData()` | `[Sync]` | Yes (read path) | Blocks renderer |
| `ReadSvg()` | **Async** | Yes | ✅ Already non-blocking |
| `ReadUnsanitizedCustomFormat()` | **Async** | Yes | ✅ Already non-blocking |

### 6.3 Internal Interfaces

- `SystemClipboard::ReadPlainText(buffer)` → `String` — Sync read (calls `[Sync]` mojom)
- `SystemClipboard::ReadPlainText(buffer, callback)` → `void` — Async read (still calls `[Sync]` mojom under the hood)
- `SystemClipboard::ReadHTML(callback)` → `void` — Async read
- `SystemClipboard::ReadPng(buffer)` → `mojo_base::BigBuffer` — Sync read only (no async overload)
- `ClipboardHostImpl::ReadText(buffer, callback)` — Browser-side handler
- `ui::Clipboard::ReadText(buffer, endpoint, callback)` — Platform abstraction; wraps sync call, NOT truly async

## 7. Threading Model

### 7.1 Thread Responsibilities

| Thread | Process | Responsibilities |
|--------|---------|-----------------|
| **Renderer Main Thread** | Renderer | Runs all JS, Blink DOM, ClipboardPromise orchestration, SystemClipboard mojo calls |
| **Renderer Worker Pool** | Renderer | ClipboardReader encoding (UTF-8 for text/html/svg) via `worker_pool::PostTask()` |
| **Browser UI Thread** | Browser | Runs ClipboardHostImpl, receives mojom IPC, calls ui::Clipboard |
| **Browser IO Thread** | Browser | Mojo IPC message routing |

### 7.2 Current Blocking Points

1. **Renderer Main Thread** blocks on `[Sync]` mojom IPC to browser (entire thread suspended until response)
2. **Browser UI Thread** blocks on `ui::Clipboard::ReadText()` → OS clipboard API (e.g., `::GetClipboardData()` on Windows)
3. On Windows, `::GetClipboardData()` sends `WM_RENDERFORMAT` to source app and blocks until response

### 7.3 Fixed Threading Model

After the fix:
1. **Renderer Main Thread** — makes async mojom call, continues processing events
2. **Browser UI Thread** — receives async mojom request, posts clipboard read to **ThreadPool**
3. **Browser ThreadPool** — performs blocking OS clipboard read
4. Result flows back: ThreadPool → UI thread → mojom response → Renderer callback

### 7.4 Synchronization & Locking

- `ui::Clipboard` uses `GetForCurrentThread()` — thread-local singleton, not shared across threads
- Windows: `::OpenClipboard()` has thread affinity; must be called from the thread that created the clipboard window
- The fix must create a dedicated clipboard-reading thread or handle thread affinity correctly on Windows
- `ScopedClipboard::Acquire()` on Windows retries `::OpenClipboard()` up to 5 times with 5ms sleep between retries

## 8. External Dependencies

### 8.1 Other Chrome Components
- **PermissionService** (`blink::mojom::PermissionService`) — Checks `clipboard-read` permission before reads
- **DocumentService** — `ClipboardHostImpl` lifetime is tied to `RenderFrameHost`
- **BrowserInterfaceBroker** — Registers `ClipboardHost` mojom binding via `browser_interface_binders.cc`
- **worker_pool** — Used by `ClipboardReader` for background encoding tasks

### 8.2 Platform APIs
- **Windows**: `::OpenClipboard()`, `::GetClipboardData()`, `::CloseClipboard()`, `WM_RENDERFORMAT`/`WM_RENDERALLFORMATS`
- **macOS**: `NSPasteboard` general pasteboard
- **Linux**: X11 selections / Wayland data offers via Ozone

### 8.3 Web Standards
- [W3C Clipboard API and events](https://w3c.github.io/clipboard-apis/) — Defines the Promise-based async API contract

## 9. Top 5 Fix Approaches

### Approach 1: Convert Mojom Read Methods to Async ⭐ RECOMMENDED (Implemented)

**Description**: Remove `[Sync]` annotation from clipboard read methods in `clipboard.mojom` for the Async Clipboard API path. Add new `[Sync]` variants (e.g., `SyncReadText`) for legacy callers (DataTransfer paste events, `document.execCommand('paste')`). Update `SystemClipboard` to use async mojom calls with callbacks. On the browser side, move `ui::Clipboard` reads to a background thread.

**Rationale**: Directly addresses the root cause. The codebase already has async patterns — `ReadSvg()` and `ReadUnsanitizedCustomFormat()` are already non-`[Sync]` in mojom, and `ClipboardTextReader`/`ClipboardHtmlReader`/`ClipboardSvgReader` already use callback-based reads. This approach extends the existing pattern to all read methods.

**Complexity**: Medium-High | **Risk**: Medium

**Pros**: Fixes root cause; non-breaking for legacy callers; follows existing patterns in codebase.
**Cons**: Significant number of files; must maintain both sync and async code paths.

### Approach 2: Move OS Clipboard Access to Background Thread (Browser-Side Only)

**Description**: Keep `[Sync]` mojom annotations but move actual `ui::Clipboard` reads in `ClipboardHostImpl` to a `base::ThreadPool` background thread. The sync mojom IPC still blocks the renderer, but total blocking time is reduced to IPC overhead.

**Complexity**: Medium | **Risk**: High

**Pros**: Fewer files changed; doesn't change IPC contract.
**Cons**: **Still blocks renderer main thread** on sync IPC. `ui::Clipboard` is thread-local (`GetForCurrentThread()`). Windows `::OpenClipboard()` has thread affinity. Architecturally incompatible.

### Approach 3: Add Timeout to Sync Mojom IPC Calls

**Description**: Add a timeout to sync mojom calls so the renderer unblocks after N seconds and the Promise rejects with a timeout error.

**Complexity**: Low | **Risk**: Medium

**Pros**: Simple change; prevents indefinite freezes.
**Cons**: Doesn't fix root cause. Users still experience freezes up to timeout. Data loss if timeout is too short. Poor UX.

### Approach 4: Introduce a Separate `AsyncClipboardHost` Mojom Interface

**Description**: Create a new mojom interface (`AsyncClipboardHost`) with purely async read methods. The Async Clipboard API uses this new interface; the existing `ClipboardHost` continues serving legacy callers.

**Complexity**: High | **Risk**: Low

**Pros**: Clean separation; zero risk to existing sync callers; designed async from the start.
**Cons**: Significant code duplication; higher maintenance burden; over-engineered for this problem.

### Approach 5: Use Mojo Async Methods with `base::RunLoop` Sync Wrappers for Legacy Callers

**Description**: Remove `[Sync]` from all read methods. For legacy synchronous callers, use `base::RunLoop` on the renderer side to simulate synchronous behavior.

**Complexity**: High | **Risk**: High

**Pros**: Single mojom interface; clean async path.
**Cons**: Nested `base::RunLoop` in the renderer causes reentrancy issues. Chromium coding guidelines explicitly prohibit this pattern. Can cause deadlocks.

### Recommended Approach: **Approach 1 — Convert Mojom Read Methods to Async**

This approach is recommended because:
1. It directly addresses the root cause at the correct architectural layer
2. It follows existing patterns already in the codebase (`ReadSvg`, `ReadUnsanitizedCustomFormat`)
3. It can be implemented incrementally without breaking legacy sync callers
4. The renderer-side async infrastructure (callback overloads in `SystemClipboard`, background encoding in `ClipboardReader`) largely already exists
5. It aligns with the long-term direction (crbug.com/443355: "Synchronous clipboard reads are deprecated")

## 10. Impact of Fix

### 10.1 Components Affected

| Component | Change Type | Description |
|-----------|------------|-------------|
| `clipboard.mojom` | **Modify** | Add async variants of read methods (or remove `[Sync]` and add `[Sync]` legacy variants) |
| `SystemClipboard` | **Modify** | Route Async Clipboard API through async mojom calls |
| `ClipboardPromise` | **Modify** | Use async `ReadPlainText(callback)` in `HandleReadTextWithPermission()` |
| `ClipboardReader` | **Modify** | Convert `ClipboardPngReader::Read()` to use async `ReadPng(callback)` |
| `ClipboardHostImpl` | **Modify** | Post clipboard reads to background thread for async methods |
| `ui::Clipboard` | **Modify** | Make callback-based read methods truly async (post to background thread instead of wrapping sync) |

### 10.2 Risk Assessment

- **Scope**: Medium — Changes span mojom, renderer, and browser layers, but follow established patterns
- **Risk Level**: Medium — Two calling patterns (sync for paste events, async for Async API) must coexist. Clipboard locking semantics must be preserved. Platform thread affinity (Windows `::OpenClipboard()`) requires careful handling.
- **Testing Coverage**: Existing web platform tests for `navigator.clipboard.read()`/`readText()` cover functional correctness. New tests needed for: async behavior verification (renderer stays responsive), delayed clipboard write scenarios, and thread-safety of background clipboard reads.
- **Backward Compatibility**: Legacy synchronous callers (`DataTransfer`, `document.execCommand('paste')`) continue using `[Sync]` mojom methods — no breaking changes for them.
