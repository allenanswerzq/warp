# Presenter — The Frame Orchestrator

> Source: `crates/warpui_core/src/presenter.rs`
> One Presenter per window. Runs the full render pipeline each frame.

---

## What It Holds

```rust
Presenter {
    window_id,                                // which window I manage
    scene: Option<Rc<Scene>>,                 // last frame's draw commands
    rendered_views: HashMap<EntityId, Element>,// view → its element tree
    parents: HashMap<EntityId, EntityId>,      // child → parent (responder chain)
    text_layout_cache,                         // reuse text measurements
    position_cache,                            // cached element positions
}
```

---

## The Frame Pipeline (`build_scene`)

Called every time a window needs to redraw:

```
build_scene(window_size, scale_factor)
  │
  ├── 1. layout()        → "how big is everything?"
  │      root.layout(constraint) → recursive
  │
  ├── 2. after_layout()  → post-layout hook (rare)
  │
  ├── 3. paint()         → "draw everything"
  │      creates fresh Scene
  │      root.paint(0,0) → recursive → pushes rects/glyphs/icons
  │
  └── 4. return Scene    → Metal/wgpu renders pixels
```

View.render()  → Element created → stored in rendered_views
  → reused across multiple build_scene() calls
  → reused for event dispatch between frames
  → only replaced when the view is invalidated (notify/notify_view)
  → removed when the view is destroyed

---

## Invalidation (Partial Updates)

Only re-renders changed views, not everything:

```
invalidate({ updated: [view_3, view_7], removed: [view_5] })
  → view_3.render() → new element stored
  → view_7.render() → new element stored
  → view_5 element dropped

Then build_scene() re-layouts and re-paints with updated elements.
```

---

## View Tree & Responder Chain

Tracks parent-child between **views** (not elements):

```
parents: { view_4 → view_3, view_3 → view_2, view_2 → view_1 }

ancestors(view_4) → [view_1, view_2, view_3, view_4]
                     root ──────────────────→ deepest
```

Used by `dispatch_action` to walk bottom-up for action bubbling.

---

## Event Dispatch

```
OS event (mouse click)
  → Presenter.dispatch_event()
    → root element.dispatch_event()  (top-down through elements)
    → returns DispatchResult {
        handled,          // did someone handle it?
        actions,          // actions to fire on the view tree
        notified,         // views to re-render
        cursor_update,    // change mouse cursor?
      }
```

---

## Context Types It Creates

```
LayoutContext       → during layout  (text cache, view stack, window size)
AfterLayoutContext  → during after_layout
PaintContext        → during paint   (Scene, font cache, position cache)
EventContext        → during dispatch_event
```

---

## Full Picture: Where Presenter Fits

```
AppContext (owns everything)
  │
  ├── models: HashMap<EntityId, Model>
  ├── windows: HashMap<WindowId, Window>
  │     └── views: HashMap<EntityId, View>
  │
  └── presenters: HashMap<WindowId, Presenter>  ← one per window
        │
        └── build_scene() each frame:
              View.render() → Element tree
                → layout()  → sizes
                → paint()   → Scene (rects, glyphs, icons)
                              → GPU → pixels
```
