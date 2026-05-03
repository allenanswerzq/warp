# OS Window — The Physical Container

> Source: `crates/warpui/src/windowing/winit/window.rs`
> The OS-level window that holds the GPU surface. NOT a UI View.

---

## Window vs View — Two Different Things

```
OS Window (this file)                  warpui View (warpui_core)
─────────────────────                  ────────────────────────
A rectangle on your screen             A piece of UI logic
One per physical window                Many per window
Holds: OS handle + GPU renderer        Holds: data + render() → Elements
Lives in: WindowManager                Lives in: AppContext
Knows: size, position, title           Knows: how to draw UI

struct Window {                        struct TerminalView {
    inner: winit::Window,                  blocks: Vec<Block>,
    renderer: wgpu Renderer,               model: ModelHandle<...>,
    scene: Option<Scene>,              }
    titlebar_height: f32,              impl View for TerminalView {
}                                          fn render() → Element tree
                                       }
```

---

## What's Inside a Window

```rust
struct Window {
    callbacks: WindowCallbacks,           // how event loop talks to warpui
    inner: Option<Inner>,                 // the actual OS window + GPU
    scene: Option<Rc<Scene>>,             // last rendered frame
    titlebar_height: f32,                 // for drag-to-move detection
    capture_callback: Option<...>,        // for screenshots/tests
}

struct Inner {
    winit_window: Arc<winit::Window>,     // the OS window handle
    resources: Resources,                  // wgpu surface + device + renderer
    size: Vector2F,                        // current window size
}
```

---

## How They Relate

```
OS Window (this struct)           does NOT contain Views
    │
    ├── winit::Window             OS handle (size, position, focus, title)
    ├── wgpu Renderer             GPU drawing (Scene → pixels)
    └── Scene                     last frame's draw commands

AppContext (owns everything)      DOES contain Views
    │
    ├── windows[window_id]
    │     └── views: HashMap<EntityId, Box<dyn AnyView>>
    │           ├── RootView
    │           ├── Workspace
    │           ├── TerminalView
    │           └── SettingsView
    │
    └── presenters[window_id]     connects them
          └── Presenter
                build_scene() reads Views → produces Scene
                              Window renders Scene → pixels
```

---

## The Connection: How Views Get to the Window

```
1. View changes (notify)
2. Presenter.build_scene()
     → calls view.render() for changed views
     → layout() → paint() → Scene
3. Window.render(scene)
     → wgpu takes Scene → GPU → pixels on screen

Views never touch the Window directly.
Window never touches Views directly.
Presenter is the bridge.
```

---

## Window Lifecycle

```
OpenWindow event
  → create_window(window_target)      → winit::Window (OS window)
  → Resources::new(window, ...)        → wgpu device + surface + renderer
  → Window { inner, scene: None }

RedrawRequested
  → build_scene() if needed            → Scene from Presenter
  → window.render(scene, font_cache)   → wgpu draws to surface → screen

Resize
  → window.handle_resize()             → update surface size
  → request_redraw()                   → re-render at new size

CloseWindow event
  → drop_renderer()                    → release GPU resources
  → remove from WindowManager          → OS window destroyed
```

---

## Key Operations

```
window.size()              → current content size (logical pixels)
window.backing_scale_factor() → retina/HiDPI scale (1.0 or 2.0)
window.render_scene(scene) → give Scene to window, schedule redraw
window.request_redraw()    → tell winit "I need a RedrawRequested"
window.render(scene, fonts) → wgpu draws Scene to GPU surface
window.drag_window()       → OS window drag (titlebar click-drag)
window.toggle_maximized()  → OS maximize/restore
```

---

## Why Window Doesn't Hold Views

Because of warpui's **ownership rule**: AppContext owns everything.

```
If Window owned Views:
  Window → View → ModelHandle → needs AppContext → Window → CYCLE!

Actual design:
  AppContext owns Views (no cycle)
  AppContext owns Windows (via WindowManager)
  Presenter reads Views, writes Scene
  Window renders Scene (no View knowledge needed)
```

Clean separation: Window = glass pane. Views = what's displayed on the glass.

---

## Two Window Structs — Same Name, Different Jobs

There are actually **two** `Window` structs, connected by the same `WindowId`:

```
warpui_core::Window (in core/window.rs)     warpui::Window (in winit/window.rs)
───────────────────────────────────         ────────────────────────────────────
WHAT to display                             HOW to display it
Platform-agnostic                           Platform-specific

struct Window {                             struct Window {
    views: HashMap<EntityId,                    inner: winit::Window,  // OS handle
              Box<dyn AnyView>>,                renderer: wgpu,        // GPU
    root_view: Option<AnyViewHandle>,           scene: Option<Scene>,  // last frame
    focused_view: Option<EntityId>,             titlebar_height: f32,
}                                           }

Lives in: AppContext.windows                Lives in: WindowManager
Holds: all Views for this window            Holds: OS window + GPU renderer
```

### How they connect

```
AppContext
  │
  ├── windows: HashMap<WindowId, warpui_core::Window>  ← has the Views
  │     └── { views, root_view, focused_view }
  │
  ├── presenters: HashMap<WindowId, Presenter>          ← reads Views → Scene
  │
  └── window_manager: Box<dyn WindowManager>
        └── HashMap<WindowId, warpui::Window>           ← has OS + GPU
              └── { winit::Window, wpu renderer,g scene }

Same WindowId ties them together.
```

### Data flow between them

```
warpui_core::Window (Views)
        │
        │ Presenter.build_scene()
        │   reads views → layout → paint
        ▼
      Scene
        │
        │ Window.render(scene)
        │   wgpu → GPU → pixels
        ▼
warpui::Window (OS + GPU) → screen
```
