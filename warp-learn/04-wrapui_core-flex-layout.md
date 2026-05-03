# Flex — The Main Layout Element

> Source: `crates/warpui_core/src/elements/flex/mod.rs`
> Warp's equivalent of CSS Flexbox. Arranges children in a row or column.

---

## One Dimension Only

A Flex arranges children along **one axis**. Nest them for 2D:

```
Flex::row()    → children go left-to-right
Flex::column() → children go top-to-bottom

┌─────────────────────────┐
│ ┌─────────┬───────────┐ │  ← Flex::row()
│ │ sidebar │  content  │ │
│ └─────────┴───────────┘ │
│ ┌─────────────────────┐ │  ← Flex::row()
│ │     status bar      │ │
│ └─────────────────────┘ │
└─────────────────────────┘
         ↑ Flex::column()
```

---

## Three Child Types

```
Regular child     → fixed size, laid out first
Shrinkable(flex)  → gets proportional space, can shrink to 0  (FlexFit::Loose)
Expanded(flex)    → gets proportional space, MUST fill it     (FlexFit::Tight)
```

```rust
Flex::row()
    .with_child(Text::new("fixed").finish())            // natural size
    .with_child(Expanded::new(1.0, content).finish())   // fills remaining
    .with_child(Shrinkable::new(1.0, side).finish())    // can shrink
```

---

## Layout Algorithm (two passes)

```
Pass 1: layout FIXED children → measure their sizes
  [Button 80px] [Label 40px]       fixed_space = 80+40+spacing
  remaining = parent_max - fixed_space

Pass 2: distribute remaining to FLEX children by ratio
  remaining = 200px, total_flex = 3.0
  Expanded(flex=1) → 200 × 1/3 = 67px
  Expanded(flex=2) → 200 × 2/3 = 133px
```

Input/output of `layout()`:

```
fn layout(constraint: SizeConstraint) -> Vector2F
  input:  "you can be between (min_w, min_h) and (max_w, max_h)"
  output: (width, height) — "I am this big"
```

---

## Paint (positioning on screen)

`paint(origin)` walks children, advancing position along the main axis:

```
Flex::row(), origin=(10,20), children A(80x40) B(60x50), spacing=5

  paint A at (10, 20)   → advance by 80+5
  paint B at (95, 20)   → advance by 60

  Screen:
  (10,20)       (95,20)
    ┌────────┐   ┌──────┐
    │   A    │   │  B   │
    └────────┘   └──────┘
```


## What Does paint() Actually Do?

Depends on the element:

```
Container elements (Flex, Align, ConstrainedBox)
  → just POSITIONING — calculates where each child goes,
    calls child.paint(position)

Leaf elements (Text, Rect, Icon, Image)
  → actually DRAWS PIXELS — writes commands to the Scene
```

```
paint() call tree              Scene (draw commands)
─────────────────              ─────────────────────
Container.paint()              (nothing)
  ├── Rect.paint()        →   push_quad(blue rect at 0,0)
  ├── Flex.paint()             (nothing, just positions)
  │   ├── Text.paint()   →   push_glyphs("Hello" at 10,5)
  │   └── Icon.paint()   →   push_icon(gear at 80,5)
  └── Text.paint()        →   push_glyphs("World" at 0,50)

                               Scene → Metal/wgpu → pixels on screen
```

## Scene — The Draw Command Buffer

Elements don't draw to the screen directly. They push commands to a `Scene`:

```
Elements paint() → Scene collects commands → GPU renders pixels
```

```rust
Scene {
    layers: Vec<Layer>,     // z-ordered (base UI, then popups on top)
}

Layer {
    rects: Vec<Rect>,       // colored boxes (background, border, shadow, radius)
    glyphs: Vec<Glyph>,     // individual text characters
    icons: Vec<Icon>,       // SVG/glyph icons
    images: Vec<Image>,     // raster images
    clip_bounds: Option,    // clipping rectangle
    hit_map: RTree,         // spatial index for mouse hit testing
}
```

Everything on screen = rects + glyphs + icons + images, in z-ordered layers.

Text::paint()      → scene.push glyph, glyph, glyph...
Container::paint() → scene.push rect (background)
Icon::paint()      → scene.push icon
Flex::paint()      → pushes nothing, just calls children's paint()

Scene layers:
  Layer 0: [rect, glyph, glyph, rect, glyph, icon, ...]
  Layer 1: [rect, glyph, ...]  ← popup/overlay on top

Metal/wgpu reads layers bottom-to-top → pixels on screen

---


## Alignment Options

### MainAxisAlignment (justify-content)

```
Start:        [A][B][C]............
Center:       .....[A][B][C].......
End:          ............[A][B][C]
SpaceBetween: [A]....[B]....[C]
SpaceEvenly:  ..[A]....[B]....[C]..
```

### CrossAxisAlignment (align-items)

```
For Flex::row(), cross axis = height:

Start:    |AA |     Center:  |   |     End:     |   |     Stretch: |AAA|
          |   |              |AA |               |   |              |AAA|
          |   |              |   |               |AA |              |AAA|
```

### MainAxisSize

```
Min → shrink-wrap (only as big as children need)
Max → fill parent (then use MainAxisAlignment for leftover space)
```

---

## The parent_data() Trick

Expanded/Shrinkable attach flex info via `parent_data()`:

```
Flex asks each child: "do you have FlexParentData?"
  → Regular child: None → layout with fixed size
  → Expanded:      Some(FlexParentData { flex: 1.0, fit: Tight }) → fill space
  → Shrinkable:    Some(FlexParentData { flex: 1.0, fit: Loose }) → can shrink
```

No type coupling — Flex reads flex data through `dyn Any` downcast.

Each parent layout element defines its own data type, and children that want to participate attach it.

It's the same pattern as dyn Any throughout warpui — type erasure with runtime downcast when you need the concrete data.

---

## CSS Flexbox Mapping

```
Flex::row()              = flex-direction: row
Flex::column()           = flex-direction: column
Expanded(flex)           = flex-grow (must fill)
Shrinkable(flex)         = flex-grow (can shrink to 0)
spacing                  = gap
MainAxisAlignment        = justify-content
CrossAxisAlignment       = align-items
MainAxisSize::Max        = width: 100% / height: 100%
Reverse orientation      = flex-direction: row-reverse / column-reverse
```

---

## dispatch_event — How Input Flows Through Elements

Events (mouse, keyboard) flow **top-down** through the element tree:

```
OS mouse click at (150, 80)
  │
  ▼
root.dispatch_event()
  → Container: passes to children unconditionally
    → Flex: passes to each child
      → EventHandler: hit test (150,80) inside my bounds?
        YES → call on_click() → return true (handled, stop)
        NO  → return false (propagate to siblings/parent)
      → Text: not interactive → return false
```

**Rules:**
1. Parent passes the event to ALL children (children decide if it applies)
2. Children do their own **hit testing** (is the mouse inside my bounds?)
3. Return `true` → "I handled it, stop propagating"
4. Return `false` → "Not mine, keep going"

```rust
// Flex's dispatch_event: just passes to all children
fn dispatch_event(&mut self, event, ctx, app) -> bool {
    let mut handled = false;
    for child in &mut self.children {
        handled |= child.dispatch_event(event, ctx, app);
    }
    handled
}

// EventHandler's dispatch_event: does actual work
fn dispatch_event(&mut self, event, ctx, app) -> bool {
    if event.is_click() && self.bounds.contains(event.position) {
        (self.on_click)(ctx);
        return true;  // handled!
    }
    false  // not for me
}
```

**Key difference from Actions:**
```
dispatch_event  = Element-level, top-down through element tree (mouse/keyboard)
dispatch_action = View-level, bottom-up through view tree (named actions, keybindings)
```
