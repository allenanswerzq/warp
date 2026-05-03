# Event Loop — The Heart of the winit Platform

> Source: `crates/warpui/src/windowing/winit/event_loop/mod.rs`
> One loop for the entire app. Receives all OS events + Warp's own events.

---

## How It Starts

```rust
// In app.rs:
let event_loop = winit::event_loop::EventLoop::new();   // OS event loop
let inner = super::EventLoop::new(ui_app, callbacks..);  // Warp's handler

event_loop.run(move |evt, window_target| {
    inner.handle_event(evt, window_target);   // called for EVERY event
});
// ↑ infinite loop — never returns
```

---

## Two Sources of Events

```
OS (user actions)                     Warp code (async, other threads)
  mouse click                           proxy.send_event(CustomEvent::...)
  key press
  window resize
  file drag & drop
       │                                       │
       └──────────────┬────────────────────────┘
                      ▼
        handle_event(evt, window_target)
                      │
                      ├── Event::WindowEvent { window_id, .. }  ← from OS
                      ├── Event::UserEvent(CustomEvent::...)     ← from Warp
                      └── Event::LoopExiting                     ← app shutdown
```

---

## OS Events (automatic from winit)

```
User does something → OS → winit → Event::WindowEvent

WindowEvent::KeyboardInput    → convert to warpui Keystroke
WindowEvent::MouseInput       → LeftMouseDown/Up, RightMouseDown, etc.
WindowEvent::CursorMoved      → MouseMoved or LeftMouseDragged
WindowEvent::MouseWheel       → ScrollWheel
WindowEvent::Touch            → converted to mouse events (tap/scroll/drag)
WindowEvent::Resized          → resize window + re-render
WindowEvent::Focused          → track active window
WindowEvent::CloseRequested   → ask app if ok to close
WindowEvent::DroppedFile      → debounce → DragAndDropFiles
WindowEvent::Ime              → IME text input (CJK etc.)
WindowEvent::RedrawRequested  → build_scene() + render()
```

---

## Custom Events (sent by Warp code via proxy)

```rust
// Anyone with a proxy can send:
proxy.send_event(CustomEvent::OpenWindow { .. });
proxy.send_event(CustomEvent::CloseWindow { .. });
proxy.send_event(CustomEvent::Terminate(..));
proxy.send_event(CustomEvent::SetCursorShape(..));
proxy.send_event(CustomEvent::RunTask(..));          // run async task on main thread
proxy.send_event(CustomEvent::UpdateUIApp(closure)); // mutate AppContext
```

Used for: async task completion, cross-thread communication,
timers, notifications, clipboard, GPU error recovery.

---

## window_target: ActiveEventLoop

Not another event loop — a **handle** to the running loop.
Needed because some operations can only happen inside the loop:

```rust
window_target.exit();                    // quit app
window_target.set_control_flow(Wait);    // how loop waits
window_target.owned_display_handle();    // for wgpu surface
// Also required for creating windows (winit enforces this)
```

---

## Per-Window State

One loop, many windows. Each window tracked by `winit::WindowId`:

```rust
struct State {
    windows: HashMap<winit::WindowId, WindowState>,
}

struct WindowState {
    window_id: crate::WindowId,     // warpui's ID (not winit's)
    modifiers: ModifiersState,       // ctrl/alt/shift/cmd
    current_mouse_button_pressed,    // for drag detection
    last_mouse_button_pressed,       // for double/triple click
    last_cursor_position,            // for hover/drag
    scroll_velocity,                 // for momentum scrolling
    pending_drag_drop_files,         // debounced file drops
}
```

---

## Rendering (RedrawRequested)

```
WindowEvent::RedrawRequested
  → build_scene() if needed (Presenter → Scene)
  → window.render(scene, font_cache) (wgpu → pixels)
  → if GPU error: recreate renderer and retry
```

---

## Event Conversion Flow

```
winit::WindowEvent
  → convert_window_event()     → ConvertedEvent
  → handle_window_event()      → dispatch to warpui
    → callbacks.dispatch_event(warpui::Event)
      → Presenter.dispatch_event()
        → element tree dispatch (top-down)
        → action dispatch (bottom-up through views)
        → returns DispatchResult { handled, actions, ... }
```

---

## All Event Types & Their Relationships

Three layers of events flow through the system:

### Layer 1: winit::Event (from OS)

```
winit::Event
  ├── NewEvents(Init)                  → app startup, one-time init
  ├── WindowEvent { window_id, event } → per-window OS events (see below)
  ├── UserEvent(CustomEvent)           → Warp's self-sent events (see below)
  └── LoopExiting                      → app shutting down
```

### Layer 2a: winit::WindowEvent (OS → per window)

```
INPUT
  KeyboardInput        → key press/release
  ModifiersChanged     → ctrl/alt/shift/cmd state
  CursorMoved          → mouse position changed
  MouseInput           → button press/release
  MouseWheel           → scroll wheel / trackpad scroll
  Touch                → touchscreen tap/drag/scroll
  Ime                  → IME composition (CJK input, emoji picker)

WINDOW LIFECYCLE
  RedrawRequested      → time to render a frame
  Resized              → window size changed
  Focused(bool)        → window gained/lost focus
  CloseRequested       → user clicked X button
  Destroyed            → window is gone
  Moved                → window dragged to new position
  ScaleFactorChanged   → moved to monitor with different DPI

FILES
  DroppedFile          → file dragged&dropped onto window

THEME
  ThemeChanged         → system light/dark mode changed
```

### Layer 2b: CustomEvent (Warp code → event loop)

```
WINDOW MANAGEMENT
  OpenWindow { window_id, options }       → create new OS window
  CloseWindow { window_id, mode }         → close a window
  FocusWindow { window_id }               → bring window to front
  ActiveWindowChanged                      → coalesced focus change
  RequestUserAttention { window_id }       → bounce dock icon
  StopRequestingUserAttention { window_id }

APP LIFECYCLE
  Terminate(mode)                          → quit the app
  AboutToSleep                             → system suspending (Linux)
  ResumedFromSleep                         → system resumed (Linux)

ASYNC / THREADING
  RunTask(Runnable)                        → run async task on main thread
  UpdateUIApp(closure)                     → mutate AppContext from any thread

INPUT
  Clipboard(Paste)                         → paste from clipboard
  SetCursorShape(cursor)                   → change mouse cursor
  ActiveCursorPositionUpdated              → update IME position
  GlobalShortcutTriggered(keystroke)       → system-wide hotkey
  MomentumScroll { window_id }             → touch momentum animation tick
  SoftKeyboardInput(input)                 → mobile WASM keyboard (wasm only)

SYSTEM
  InternetConnected                        → network came up
  InternetDisconnected                     → network went down
  SystemThemeChanged                       → light/dark mode changed
  SendNotification { window_id, info }     → show desktop notification
  RequestNotificationPermissions(cb)       → ask OS for permission

FILES
  DragAndDropFilesDebounced { window_id }  → debounced file drop

DISPLAY
  VisualViewportResized { w, h }           → soft keyboard resize (wasm only)
```

### Layer 3: ConvertedEvent (internal, after conversion)

```
winit::WindowEvent → convert_window_event() → ConvertedEvent

ConvertedEvent::Event(warpui::Event)       → dispatched to element tree
ConvertedEvent::KeyDownWithTypedCharacters  → key + fallback TypedCharacters
ConvertedEvent::Resize                      → window resize handling
ConvertedEvent::WindowMoved                 → window position update
ConvertedEvent::ModifierKeyChanged          → modifier key press/release
ConvertedEvent::MoveWindowBy               → touch-based window drag
```

### Layer 4: warpui::Event (final, dispatched to elements)

```
KEYBOARD
  KeyDown { keystroke, chars }
  TypedCharacters { chars }                → text input (if KeyDown unhandled)
  ModifierStateChanged { modifiers }
  ModifierKeyChanged { key_code, state }

MOUSE
  LeftMouseDown { position, click_count }
  LeftMouseUp { position }
  LeftMouseDragged { position }
  MiddleMouseDown { position }
  RightMouseDown { position }
  BackMouseDown { position }               → mouse side button
  ForwardMouseDown { position }            → mouse side button
  MouseMoved { position, is_synthetic }
  ScrollWheel { position, delta, precise }

IME
  SetMarkedText { text, range }            → IME composition preview
  ClearMarkedText                          → IME composition done

FILES
  DragAndDropFiles { paths, location }
  DragFiles { paths, location }            → dragging over (hover)
  DragFileExit                             → drag left window
```

### How They Flow

```
OS                    winit             Warp internal        warpui elements
──                    ─────             ─────────────        ───────────────
mouse click    →  WindowEvent      →  ConvertedEvent  **** →  warpui::Event
                  ::MouseInput        ::Event              ::LeftMouseDown
                                                              │
key press      →  WindowEvent      →  ConvertedEvent   →  warpui::Event
                  ::KeyboardInput     ::KeyDownWith...     ::KeyDown
                                                              │
                                                              ▼
                                                      element.dispatch_event()
                                                              │
async done     →  UserEvent        →  (direct)         →  model.update()
                  ::RunTask                                    │
                                                              ▼
                                                        notify → redraw
```

### Summary

Layer          Why it exists
─────          ─────────────
winit::Event       third-party API, can't change it
WindowEvent        OS-level, uses OS concepts
CustomEvent        thread-safety bridge (background → main thread)
ConvertedEvent     convert winit quirks → clean warpui format
warpui::Event      platform-agnostic, what elements actually consume


PRODUCER side:              CONSUMER side:
ctx.notify()                winit event loop (handle_event)
  → queues effect             → processes RedrawRequested
  → flush_effects()           → calls render() + build_scene()
  → window.request_redraw()
  → OS gets the request
  → OS sends RedrawRequested back to the event loop


notify() is called somewhere (any thread, any callback)
     │
     ▼
flush_effects() → marks window dirty → request_redraw()
     │
     ▼
OS queues a RedrawRequested event
     │
     ▼
Event loop picks it up on the NEXT iteration
     │
     ▼
redraw_window() → render() → build_scene() → GPU → pixels
