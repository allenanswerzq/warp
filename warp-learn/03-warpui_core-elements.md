# Elements — Warp's UI Building Blocks

> Source: `crates/warpui_core/src/elements/`

---

## What Are Elements?

**Elements are Warp's "HTML tags."** Short-lived objects describing one frame of UI.
Views create element trees in `render()`, the framework does layout → paint → discard.

---

## The Element Trait (3 core methods)

```rust
trait Element {
    fn layout(&mut self, constraint: SizeConstraint, ...) -> Vector2F;  // how big am I?
    fn paint(&mut self, origin: Vector2F, ...);                         // draw me
    fn dispatch_event(&mut self, event, ...) -> bool;                   // do I handle this?
}
```

---

## Element Catalog

```
LAYOUT
  flex/              Flexbox row/column (THE main layout tool)
  align              Align child (center, top-left, etc.)
  constrained_box    Force min/max width/height
  container          Box with padding, background, border, radius
  stack/             Z-stack (children layered on top)
  empty              Spacer, renders nothing
  percentage         Size as % of parent

CONTENT
  text               Plain text label
  formatted_text     Rich text (bold, italic, links)
  icon               SVG/glyph icons
  image              Raster images
  rect               Colored rectangle
  shimmering_text    Loading placeholder animation

INTERACTION
  event_handler      Catch mouse/keyboard events
  hoverable          Hover state + tooltips
  drag/              Drag and drop
  dismiss            Click-outside-to-close
  selectable_area    Text selection

SCROLLING / LISTS
  scrollable         Basic scroll container
  list               Virtualized list (only renders visible)
  uniform_list       All rows same height (fast)
  viewported_list    Viewport-tracked
  shared_scrollbar   Shared across elements

CONTAINERS
  child_view         Embed another View as an element
  clipped            Clip overflow (CSS overflow: hidden)
  table/             Rows, columns, headers
```

---

## How They Compose

```rust
fn render(&self, app: &AppContext) -> Box<dyn Element> {
    Container::new(
        Flex::column()
            .with_child(Text::new("Hello").finish())
            .with_child(
                EventHandler::new(Text::new("Click").finish())
                    .on_click(|_, ctx| { ... })
                    .finish()
            )
            .finish()
    ****)
    .with_background(color)
    .with_padding(8.0)
    .finish()
}
```

`.finish()` boxes the element: `ConcreteType` → `Box<dyn Element>`.

---

## Event Flow (top-down)

```
OS mouse click
  → root.dispatch_event()
    → Container → Flex → each child
      → EventHandler: hit test ✓ → on_click() → returns true (stop)
      → Text: not interactive → returns false (propagate)
```

---

## Key Rules

```
Elements   = throwaway (created each render, discarded after paint)
Views      = persistent (hold state, produce elements via render())
Layout     = constraint-based (parent gives min/max, child returns size)
ParentElement = any element with children gets with_child() for free
```

---

## Element Quick Reference

### Layout

```
Flex::column() / Flex::row()     THE main layout — vertical/horizontal
Expanded(flex, child)            MUST fill proportional space in Flex
Shrinkable(flex, child)          CAN shrink to 0 in Flex
Stack                            Z-stack — layers children (for overlays)
Align                            Position child (center, top-left, etc.)
Container                        Box with background, padding, border, radius
ConstrainedBox                   Force min/max width/height
Empty                            Spacer, renders nothing
```

### Content

```
Text                             Plain text label
Icon                             SVG/glyph icon
Rect                             Colored rectangle (backgrounds, cursor)
ChildView                        Embed another View's element tree
```

### Interaction

```
EventHandler                     Catch mouse/keyboard events
Hoverable                        Track hover state + tooltips
DropTarget                       File drag-and-drop area
```

### Scrolling

```
Scrollable / NewScrollable       Scrollable container
Clipped                          Clip overflow (CSS overflow: hidden)
```

### Positioning

```
SavePosition                     Cache element position for lookups
PositionedElementAnchor          Anchor overlays (TopMiddle, BottomMiddle)
```

---

## Common Pattern

```rust
Container::new(
    Flex::row()
        .with_child(Icon::new(...).finish())
        .with_child(Expanded::new(1., Text::new("content").finish()).finish())
        .with_child(button.finish())
        .finish()
)
.with_background_color(...)
.with_padding(8.)
.with_corner_radius(4.)
.finish()
```

---

## How TerminalView Uses Elements

```
Stack                                    ← root (overlays go on top)
  └── Flex::column()                     ← main vertical layout
        ├── Shrinkable(1., output)       ← output fills space, can shrink
        │     └── BlockListElement       ← scrollable block list
        │         OR AltScreenElement    ← fullscreen (vim/less)
        └── ChildView(input)             ← command input box

  Overlays (positioned on Stack):
    ├── tooltip, context menu, find bar
    ├── agent progress, banners
    └── onboarding callout
```
