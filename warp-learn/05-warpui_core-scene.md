# Scene — The Draw Command Buffer

> Source: `crates/warpui_core/src/scene.rs`

---

## What Is Scene?

The buffer between elements and the GPU. Elements don't draw pixels directly —
they push commands to the Scene, the GPU renderer reads it.

```
Elements paint() → Scene collects commands → Metal/wgpu renders pixels
```

---

## Four Primitive Draw Types

Everything on screen is made of these:

```
Rect   → colored box (background, border, corner radius, drop shadow)
Glyph  → one text character (font, position, color)
Icon   → SVG icon (position, color, opacity)
Image  → raster image (position, opacity, corner radius)
```

---

## Layers (Z-Ordering)

Scene organizes draw commands into **layers**, rendered bottom-to-top:

```
Scene {
    layers: Vec<Layer>,           // normal layers [0, 1, 2, ...]
    overlay_layers: Vec<Layer>,   // always on top of ALL normal layers
}

Layer {
    rects, glyphs, icons, images, // draw commands
    clip_bounds,                   // clipping rectangle
    hit_map: RTree,                // spatial index for mouse hit testing
}
```

Two kinds:
```
ZIndex::Normal(n)   → regular layers, bottom-to-top
ZIndex::Overlay(n)  → always rendered ON TOP of all normal layers
```

---

## How Elements Choose Their Layer

Stack-based: Scene has an `active_layer_index_stack`. Draw calls go to
whatever layer is on top of the stack.

```
Regular elements:  just draw to current active layer (don't change it)
Popups/overlays:   push a new layer, draw, then pop it
```

```
scene.start_layer(bounds)     ← push new layer onto stack
  scene.draw_rect(...)        → goes to NEW layer
  scene.draw_glyph(...)       → goes to NEW layer
scene.stop_layer()            ← pop back to previous layer
  scene.draw_rect(...)        → goes to ORIGINAL layer
```

### Example: dropdown menu

```
Container.paint()              active: layer 0
  Flex.paint()                 active: layer 0
    Text.paint()               → draw glyphs to layer 0
    Dropdown.paint()
      scene.start_layer()      active: layer 1 (NEW)
        items.paint()          → draw to layer 1 (on top!)
      scene.stop_layer()       active: layer 0 (back)
    Button.paint()             → draw to layer 0
```

Result:
```
Layer 0: [text, button]           ← rendered first (behind)
Layer 1: [dropdown items]         ← rendered second (on top)
```

---

## Hit Testing

Each layer has a spatial index (`RTree`) for fast "is the mouse over something?" queries:

```rust
scene.is_covered(point)  → "is this point covered by a HIGHER layer?"
```

Used by `dispatch_event` to know if a click should go to the dropdown
(layer 1) or the button behind it (layer 0).

---

## Draw API

```rust
scene.draw_rect_with_hit_recording(bounds)  → push Rect + register in hit_map
scene.draw_rect_without_hit_recording(bounds) → push Rect only (perf optimization)
scene.draw_image(bounds, asset, opacity)    → push Image
scene.draw_icon(bounds, asset, opacity, color) → push Icon
// Glyphs pushed via text rendering pipeline
```

---

## Full Pipeline

```
View.render()        → builds Element tree
  layout()           → each element computes its size
  paint()            → each element pushes to Scene
                        ├── Rect → scene.draw_rect()
                        ├── Glyph → (text pipeline)
                        ├── Icon → scene.draw_icon()
                        └── start/stop_layer for z-ordering
  Scene              → layers of {rects, glyphs, icons, images}
  GPU Renderer       → Metal (macOS) or wgpu (Win/Linux)
  Screen             → pixels!
```
