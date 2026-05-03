# Events — The Input Event System

> Source: `crates/warpui_core/src/event.rs`

---

## Event Enum — All Input Types

```rust
enum Event {
    // Keyboard
    KeyDown { keystroke, chars, details, is_composing }

    // Mouse buttons
    LeftMouseDown { position, modifiers, click_count, is_first_mouse }
    LeftMouseUp { position, modifiers }
    LeftMouseDragged { position, modifiers }
    MiddleMouseDown { position, ... }
    RightMouseDown { position, ... }
    BackMouseDown { position, ... }      // side button (back)
    ForwardMouseDown { position, ... }   // side button (forward)

    // Mouse movement
    MouseMoved { position, cmd, shift, is_synthetic }

    // Scroll
    ScrollWheel { position, delta, precise, modifiers }

    // Modifier keys
    ModifierStateChanged { mouse_position, modifiers, key_code }

    // Text input (IME)
    TypedCharacters { chars }
    SetMarkedText { text, selected_range }
    ClearMarkedText

    // Drag & drop
    DragAndDropFiles { position, paths }
    DragFiles { position, paths }
    DragFileExit
}
```

---

## DispatchedEvent — Z-Index Filtering

Events are wrapped in `DispatchedEvent` before dispatch. The key method:

```rust
event.at_z_index(my_z_index, ctx) → Option<&Event>
```

- **Keyboard events** → always pass through (not position-based)
- **Mouse clicks/scroll** → checks `is_covered(position, z_index)`:
  - Higher layer blocks it → `None` (skip)
  - Nothing above → `Some(event)` (handle it)
- **MouseMoved** → always passes through (hover needs it everywhere)

```
Layer 1: [dropdown]    ← click here
Layer 0: [button]      ← also here, but BLOCKED

button.at_z_index(Normal(0)) → None (covered by layer 1)
dropdown.at_z_index(Normal(1)) → Some(event) (nothing above)
```

---

## Event Flow Summary

```
OS event
  → Platform (macOS/winit/WASM) converts to Event enum
  → Presenter.dispatch_event(event)
    → wraps in DispatchedEvent
    → root element.dispatch_event() — top-down through element tree
      → each element calls event.at_z_index() to check coverage
      → handled? return true (stop) or false (propagate)
    → returns DispatchResult (actions to fire, views to notify)
  → AppContext processes DispatchResult
    → fires actions (bottom-up through view responder chain)
    → notifies views (triggers re-render)
```
