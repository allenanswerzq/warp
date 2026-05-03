# Platform Traits — The Contract Between Core and Backends

> Source: `crates/warpui_core/src/platform/mod.rs`
> These traits define what each platform backend (macOS/Windows/Linux/WASM) must implement.

---

## The Five Key Traits

```
warpui_core defines:              warpui implements (per platform):
────────────────────              ──────────────────────────────────
trait Delegate                    mac/delegate.rs, wasm/mod.rs
trait Window                      mac/window.rs, winit window
trait WindowContext                mac/window.rs, winit window
trait WindowManager               mac/app.rs, winit event_loop
trait FontDB                      mac/fonts.rs, cosmic-text
```

---

## `Delegate` — App-Level Platform Services

```
clipboard, cursor, URL opening, file picker,
notifications, IME, accessibility, app termination
```

The grab-bag of "things the OS provides." One per app.

---

## `Window` + `WindowContext` — Per-Window Interface

```rust
trait WindowContext {
    fn size(&self) -> Vector2F;              // window content size
    fn origin(&self) -> Vector2F;            // position on screen
    fn backing_scale_factor(&self) -> f32;   // retina/HiDPI scale
    fn render_scene(&self, scene: Rc<Scene>); // ← THE handoff to GPU
    fn request_redraw(&self);                // schedule next frame
}

trait Window: WindowContext {
    fn minimize(&self);
    fn toggle_fullscreen(&self);
    fn supports_transparency(&self) -> bool;
    fn graphics_backend(&self) -> GraphicsBackend;
    ...
}
```

**`render_scene()`** is the bridge: Presenter calls `build_scene()` → returns `Rc<Scene>` → platform window calls `render_scene(scene)` → GPU renderer draws it.

---

## `WindowManager` — Window Lifecycle

```rust
trait WindowManager {
    fn open_window(&mut self, id, options, callbacks);
    fn remove_window(&mut self, id);
    fn active_window_id(&self) -> Option<WindowId>;
    fn platform_window(&self, id) -> Option<Rc<dyn Window>>;
    ...
}
```

---

## `FontDB` — Font Loading & Glyph Rasterization

```rust
trait FontDB {
    fn load_from_bytes(&mut self, name, bytes) → FamilyId;
    fn select_font(family_id, properties) → FontId;
    fn font_metrics(font_id) → Metrics;
    fn glyph_for_char(font_id, char) → GlyphId;
    fn glyph_advance(font_id, glyph_id) → Vector2I;
    fn rasterize_glyph(font_id, size, glyph_id, ...) → RasterizedGlyph;
    fn text_layout_system() → &dyn TextLayoutSystem;
    ...
}
```

The font pipeline:
```
char 'A' → glyph_for_char() → GlyphId(65)
         → glyph_advance()  → how far to move cursor
         → rasterize_glyph() → pixel bitmap for the glyph cache
```

---

## How It All Connects

```
warpui_core (platform-agnostic)         warpui (platform-specific)
───────────────────────────             ────────────────────────────
AppContext holds:                       macOS implements:
  platform_delegate: Box<dyn Delegate>    Delegate (Cocoa)
  window_manager: Box<dyn WindowManager>  WindowManager (AppKit)
  font_db: Box<dyn FontDB>               FontDB (Core Text)
                                          Window (NSWindow + Metal)
Presenter.build_scene() → Scene
  Window.render_scene(scene) ──────→    Metal/wgpu renderer.draw(scene)
```
