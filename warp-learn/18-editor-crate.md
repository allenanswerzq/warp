# Editor Systems — Two Separate Implementations

> Warp has TWO editor systems. They share SumTree but are otherwise independent.

---

## The Two Systems

```
SYSTEM A: Editor Crate (code editor, notebooks)
  crates/editor/src/                    traits + render + Buffer
  app/src/code/editor/                  CodeEditorModel, CodeEditorView
  app/src/notebooks/editor/             NotebooksEditorModel

SYSTEM B: App EditorModel (terminal input, AI chat)
  app/src/editor/view/model/            EditorModel + CRDT Buffer
  app/src/editor/view/element.rs        EditorElement
  app/src/editor/view/snapshot.rs       ViewSnapshot
```

```
System A (Code/Notebooks)         System B (Terminal/AI input)
─────────────────────────         ──────────────────────────
trait CoreEditorModel              struct EditorModel (standalone)
trait PlainTextEditorModel         does NOT implement CoreEditorModel
trait RichTextEditorModel

crate Buffer (SumTree<TextChunk>)  CRDT Buffer (SumTree<Fragment>)
crate RenderState                  own DisplayMap + selections
RichTextElement (crate)            EditorElement (app)

Used by:                           Used by:
  CodeEditorView                     Terminal input box
  NotebooksEditorView                AI chat input
                                     Search bars
```

---

## System A: Editor Crate (Code Editor + Notebooks)

### Crate architecture (crates/editor/src/)

```
model.rs          trait CoreEditorModel → PlainTextEditorModel → RichTextEditorModel
editor.rs         trait EditorView (what app views implement)
content/          Buffer (SumTree-backed), Anchor, edit ops, undo, find
render/
  model/          RenderState (caches layout, tracks which lines changed)
  layout.rs       text layout helpers
  element/        RichTextElement<V: EditorView> — with RenderableBlock
decoration/       syntax highlighting, underlines
```

### Who implements the traits

```rust
// Code Editor:
impl CoreEditorModel for CodeEditorModel { ... }      // app/src/code/editor/model.rs
impl PlainTextEditorModel for CodeEditorModel { ... }

// Notebooks:
impl CoreEditorModel for NotebooksEditorModel { ... }  // app/src/notebooks/editor/model.rs
impl RichTextEditorModel for NotebooksEditorModel { ... }
```

### RichTextElement — the rendering Element

```
crates/editor/src/render/element/mod.rs → RichTextElement<V: EditorView>
  implements Element trait (layout + paint + dispatch_event)
  reads from CoreEditorModel via RenderState
  handles: syntax colors, line numbers, code folding, diffs, embedded items

CodeEditorView.render():
  RichTextElement::<CodeEditorView>::new(render_state, ...)
  → wrapped in EditorWrapper (adds gutter, line numbers, diff markers)
  → Box<dyn Element>
```

---

## System B: App EditorModel (Terminal Input + AI Chat)

### Architecture (app/src/editor/view/)

```
model/
  mod.rs          EditorModel (standalone struct, NOT a trait impl)
  buffer/         CRDT Buffer (SumTree<Fragment>, collaborative)
  display_map/    DisplayMap (buffer → screen positions)
  selections/     Selection types, DrawableSelection
element.rs        EditorElement (custom Element)
snapshot.rs       ViewSnapshot (frozen state per frame)
```

### EditorModel — CRDT-based collaborative buffer

```rust
struct EditorModel {
    buffer_and_display_map: BufferAndDisplayMaps,  // CRDT Buffer + DisplayMap
    interaction_state: InteractionState,            // editable/selectable/disabled
    vim_visual_tails: Vec<Anchor>,                  // vim mode
    max_buffer_len: Option<usize>,                  // input length limit
}

// CRDT Buffer has:
//   fragments: SumTree<Fragment>    ← text stored as CRDT fragments
//   ReplicaId                       ← for shared session collaboration
//   remote peer selections          ← show other users' cursors
```

### EditorElement — the rendering Element

```
app/src/editor/view/element.rs → EditorElement
  implements Element trait (layout + paint + dispatch_event)
  reads from EditorModel via ViewSnapshot
  handles: cursor, selection, soft wrap, autosuggestions, prompt notch

TerminalView → Input.render():
  EditorElement::new(view_snapshot, scroll_state, ...)
  → Box<dyn Element>
```

---

## What They Share

```
Both use SumTree:
  System A: SumTree<TextChunk>     (text storage)
  System B: SumTree<Fragment>      (CRDT fragments)
            SumTree<Run>           (styled text runs)

Both draw to the same Scene:
  scene.draw_glyph() for text characters
  scene.draw_rect() for selections, cursors, highlights

Both implement Element trait:
  layout() → measure text, compute visible range
  paint() → draw glyphs + rects to Scene
  dispatch_event() → handle mouse/keyboard
```

---

## Why Two Systems?

```
Terminal input needs:              Code editor needs:
───────────────────                ──────────────────
CRDT collaborative editing         Single-user editing
  (shared session cursors)         Syntax highlighting
  (remote peer selections)         Code folding
Simple text buffer                 Rich text (markdown, embeds)
Autosuggestions                    Line numbers + gutter
Command x-ray tooltip              Git diff markers
No code folding                    LSP integration
No syntax highlighting             Hidden lines
```

---

## Element tree comparison

### Terminal input (System B)

```
TerminalView.render()
  └── ChildView(Input)
        └── Input.render()
              └── EditorElement         ← app's own Element
                    reads ViewSnapshot
                    draws: text, cursor, selection, autosuggestion
```

### Code editor (System A)

```
CodeEditorView.render()
  └── EditorWrapper
        └── RichTextElement<CodeEditorView>  ← crate's Element
              reads RenderState
              draws: text, cursor, selection, syntax colors,
                     embedded items, fold markers
        + line numbers gutter
        + diff markers
        + code folding UI
```

---

## Keystroke → Pixels (System B: terminal input)

```
User types 'x':
  → EditorElement.dispatch_event(TypedCharacters)
  → dispatches EditorAction::UserInsert("x")
  → EditorModel.insert("x")
  → CRDT Buffer.edit(insert at cursor)
    → SumTree<Fragment> updated
    → selections adjusted
    → undo stack records edit
  → emit EditorModelEvent::Edited
  → notify() → re-render
  → EditorElement.layout() → measure lines
  → EditorElement.paint() → glyphs + cursor to Scene
  → GPU → pixels
```

## Keystroke → Pixels (System A: code editor)

```
User types 'x':
  → RichTextElement.dispatch_event(TypedCharacters)
  → dispatches editor action
  → CodeEditorModel.user_insert("x")     (via CoreEditorModel trait)
  → crate Buffer.edit(insert at cursor)
    → SumTree<TextChunk> updated
    → anchors adjusted
    → buffer_version++
  → RenderState.add_pending_edit(delta)   (incremental layout)
  → notify() → re-render
  → RichTextElement.layout() → measure ONLY changed lines
  → RichTextElement.paint() → glyphs + syntax + cursor to Scene
  → GPU → pixels
```
