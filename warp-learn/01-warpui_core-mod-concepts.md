# warpui_core — Core Concepts & Abstractions

> Source: `crates/warpui_core/src/core/mod.rs`
> This is the **brain** of Warp's custom UI framework. Every view, model, action,
> event, and async callback flows through the types defined here.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AppContext                                │
│  (owns everything — all models, views, windows, subscriptions)  │
│                                                                  │
│  ┌──────────────────────┐    ┌────────────────────────────────┐ │
│  │   Models (no UI)     │    │   Windows                      │ │
│  │                      │    │  ┌──────────────────────────┐  │ │
│  │  AuthState           │    │  │ Window #1                │  │ │
│  │  TerminalModel       │    │  │  ┌─────────────────────┐ │  │ │
│  │  AIConversationModel │    │  │  │ Views (have UI)     │ │  │ │
│  │  DriveIndex          │    │  │  │  RootView           │ │  │ │
│  │  ...                 │    │  │  │  ├── Workspace      │ │  │ │
│  │                      │    │  │  │  │   ├── PaneGroup  │ │  │ │
│  │  id: EntityId only   │    │  │  │  │   │   └── Term.  │ │  │ │
│  └──────────────────────┘    │  │  │  ...                │ │  │ │
│           │                  │  │  │  id: WindowId +     │ │  │ │
│           │ emit/notify      │  │  │      EntityId       │ │  │ │
│           ▼                  │  │  └─────────────────────┘ │  │ │
│  ┌──────────────────────┐    │  └──────────────────────────┘  │ │
│  │ Subscriptions        │    │  ┌──────────────────────────┐  │ │
│  │ Observations         │    │  │ Window #2 ...            │  │ │
│  │ TaskCallbacks        │    │  └──────────────────────────┘  │ │
│  └──────────────────────┘    └────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Entity — The Universal Identity

Every "thing" in the framework is an `Entity`. It's the base trait for both
Models and Views.

```rust
trait Entity: 'static {
    type Event;  // what kind of events this entity can emit
}
```

Each entity gets a globally unique `EntityId`:

```
EntityId(0) → RootView
EntityId(1) → Workspace
EntityId(2) → TerminalModel
EntityId(3) → PaneGroup
...
```

**Key insight:** `EntityId` is shared across Models and Views — there's one
global counter, not separate namespaces.

---

## 2. Model vs View — The Two Entity Kinds

```
┌─────────────────────────────────────────────────┐
│                    Entity                        │
│              (has identity + events)             │
├─────────────────────┬───────────────────────────┤
│       Model         │          View              │
│                     │                            │
│  • Pure data/state  │  • Has data + render()     │
│  • No UI output     │  • Produces Element tree   │
│  • No WindowId      │  • Lives in a Window       │
│  • Located by:      │  • Located by:             │
│    EntityId only    │    WindowId + EntityId      │
│                     │                            │
│  Examples:          │  Examples:                 │
│  - AuthState        │  - RootView                │
│  - TerminalModel    │  - Workspace               │
│  - DriveIndex       │  - SettingsView            │
│  - AIConversation   │  - EditorView              │
└─────────────────────┴───────────────────────────┘
```

This is encoded directly in the `EntityLocation` enum:
```rust
enum EntityLocation {
    Model(EntityId),              // just an ID
    View(WindowId, EntityId),     // scoped to a window
}
```

**Why it matters:** A Model can be shared across windows (it has no window
affinity). A View always belongs to exactly one window.

---

## 3. View vs AnyView — The Type Erasure Bridge

```
  YOU write this:                    FRAMEWORK stores this:
┌──────────────────┐              ┌───────────────────────┐
│  trait View       │   blanket   │  trait AnyView         │
│                   │────impl────▶│                        │
│  fn render()      │             │  fn render()           │
│  fn on_focus()    │             │  fn as_any()           │
│  fn ui_name()     │             │  fn keymap_context()   │
│  ...              │             │  ...                   │
│                   │             │                        │
│  Fully typed:     │             │  Type-erased:          │
│  Self = concrete  │             │  &dyn AnyView          │
└──────────────────┘              └───────────────────────┘

impl<T: View> AnyView for T { ... }  // ← the blanket impl
```

**Rule:** Implement `View` → get `AnyView` for free.

**Why:** The framework needs to store all views (TerminalView, Workspace,
SettingsView...) in one `HashMap<EntityId, Box<dyn AnyView>>`. Without type
erasure, you'd need a separate HashMap per view type.

---

## 4. ViewType — Finding Handlers by Concrete Type

```rust
struct ViewType(TypeId);  // wrapper around Rust's TypeId
```

Used as the key in the action dispatch table:

```
actions: HashMap<ViewType, HashMap<String, Vec<ActionCallback>>>
                 ^^^^^^^^         ^^^^^^
                 "which type       "which action
                  of view"          name"
```

All instances of `TerminalView` share the same handlers.
All instances of `Workspace` share theirs.

```
ViewType::of::<TerminalView>()  ──→ { "clear": [...], "copy": [...] }
ViewType::of::<Workspace>()     ──→ { "new_tab": [...], "close_tab": [...] }
ViewType::of::<SettingsView>()  ──→ { "toggle": [...] }
```

---

## 5. Actions — The Action Dispatch System

### Three levels of action handling:

```
    User presses Ctrl+L
           │
           ▼
    ┌─────────────────────────────────────────┐
    │  1. Walk the responder chain (view tree) │
    │     deepest view → root view             │
    │                                          │
    │     TerminalView  → handled? ──YES──→ STOP
    │         │ NO                             │
    │     PaneGroup     → handled? ──YES──→ STOP
    │         │ NO                             │
    │     Workspace     → handled? ──YES──→ STOP
    │         │ NO                             │
    │     RootView      → handled? ──YES──→ STOP
    │         │ NO                             │
    ├─────────┼───────────────────────────────┤
    │  2. Try GlobalActionCallback             │
    │     (app-wide shortcuts)                 │
    ├──────────────────────────────────────────┤
    │  3. Log warning: "no one handled this"   │
    └──────────────────────────────────────────┘
```

### Four callback type aliases:

```
ActionCallback          → view-scoped, returns bool (handled?)
TypedActionCallback     → view-scoped, specific type (no bool needed)
GlobalActionCallback    → app-wide, includes caller Location for debugging
InvalidationCallback    → "this window needs a repaint"
```

---

## 6. Subscription & Observation — Reactive Data Flow

### Subscription = "tell me when entity X emits an event"

```
TerminalModel.emit("command_finished", data)
       │
       ├──→ BlockListView (subscribed) → re-renders
       ├──→ AIModel (subscribed) → checks auto-respond
       └──→ App (subscribed) → updates window title
```

### Observation = "tell me when entity X changes"

```
TerminalModel.notify()  // "my data changed"
       │
       └──→ BlockListView (observing) → re-renders
```

### Difference:

```
  Subscription                    Observation
  ─────────────                   ───────────
  Trigger: emit(event)            Trigger: notify()
  Payload: yes (custom data)      Payload: no (just "changed")
  Use: specific events            Use: any data change
  Like: addEventListener()        Like: React useEffect dependency
```

### Who can subscribe/observe:

```
enum Subscription {
    FromModel  { model_id, callback }       // Model → listens to Entity
    FromView   { window_id, view_id, cb }   // View  → listens to Entity
    FromApp    { callback }                 // App   → listens to Entity
}
```

Same pattern for `Observation`.

### SubscriptionKey — cleanup tracking:

```rust
enum SubscriptionKey {
    Model(EntityId),              // "Model #42 is listening"
    View(WindowId, EntityId),     // "View #7 in Window #1 is listening"
}
```

When an entity is destroyed, all its subscriptions are removed using these keys.

---

## 7. TaskCallback — Async Bridge

The framework is **synchronous** internally (single-threaded entity mutations).
Async work (HTTP, AI streaming, file I/O) is bridged back via `TaskCallback`.

### Two async patterns:

```
  Future (one result)              Stream (many results)
  ───────────────────              ─────────────────────
  fetch_profile()                  ai.stream_response()
       │                                │
       ▼                           ┌────┼────┐
   one value                       ▼    ▼    ▼
   FnOnce()                    chunk chunk chunk
                               FnMut  FnMut  FnMut
                                              │
                                              ▼ stream ends
                                           FnOnce (done)
```

Sync world                          Async world
┌──────────────┐                   ┌──────────────┐
│ Model / View │──spawn_future()──→│ async task    │
│              │                   │ (HTTP, AI...) │
│              │←─TaskCallback─────│ resolves      │
│  mutated     │  routes result    └──────────────┘
│  safely      │  back to entity
└──────────────┘

### Four variants in TaskCallback:

```
                        Future              Stream
                ┌───────────────────┬───────────────────────┐
    Model       │ ModelFromFuture   │ ModelFromStream        │
    (no window) │  callback: FnOnce │  on_item: FnMut       │
                │                   │  on_done: FnOnce      │
                ├───────────────────┼───────────────────────┤
    View        │ ViewFromFuture    │ ViewFromStream         │
    (in window) │  callback: FnOnce │  on_item: FnMut       │
                │                   │  on_done: FnOnce      │
                └───────────────────┴───────────────────────┘
```

### Real-world example — AI streaming:

```
AIConversationModel spawns a stream:

  ┌──────────────┐         ┌─────────────────┐
  │ async stream  │────────→│ TaskCallback::   │
  │ of AI chunks  │  chunk  │ ModelFromStream  │
  └──────────────┘    │    │                  │
                      ▼    │  on_item(model,  │──→ model.append(chunk)
                    chunk  │    chunk, ctx)    │    ctx.notify()
                      ▼    │                  │
                    chunk  │  on_item(...)     │──→ model.append(chunk)
                      ▼    │                  │
                    done   │  on_done(model,   │──→ model.set_complete()
                           │    ctx)           │
                           └─────────────────┘
```

---

## 8. RefCounts — Entity Lifecycle

How the framework knows when to destroy entities:

```
ModelHandle<T> created   → ref_counts.inc_entity(id)
ModelHandle<T> cloned    → ref_counts.inc_entity(id)
ModelHandle<T> dropped   → ref_counts.dec_model(id)
                           if count == 0 → mark as dropped

ViewHandle<T> dropped    → ref_counts.dec_view(window_id, view_id)
                           if count == 0 → mark as dropped
```

```rust
struct RefCounts {
    entity_counts: HashMap<EntityId, usize>,  // ref count per entity
    dropped: DroppedItems,                     // entities to clean up
}

struct DroppedItems {
    models: HashSet<EntityId>,                 // dead models
    views: HashSet<(WindowId, EntityId)>,      // dead views (need window)
}
```

---

## 9. Effect — The Deferred Work Queue

Instead of doing things immediately, the framework queues `Effect`s and
processes them in a batch:

```rust
enum Effect {
    Event { entity_id, payload }         // entity emitted an event
    ModelNotification { model_id }       // model data changed
    ViewNotification { window_id, id }   // view needs repaint
    Focus { window_id, view_id }         // focus changed
    TypedAction { window_id, id, action }// typed action dispatched
    GlobalAction { name, location, arg } // global shortcut fired
}
```

```
User does something
       │
       ▼
  Queue effects: [Focus, ViewNotification, TypedAction]
       │
       ▼
  flush_effects() processes them one by one
  (may generate more effects → keeps draining until empty)
```

---

## 10. Display & Window Types

```rust
DisplayId(usize)     // unique hardware display identifier
DisplayIdx::Primary  // "use the main monitor"
DisplayIdx::External(0)  // "use the 1st external monitor"

WindowId             // unique window identifier (from entity.rs)

AddWindowOptions {
    window_style,    // bordered, borderless, etc.
    window_bounds,   // position & size
    fullscreen_state,
    title,
    background_blur_radius_pixels,  // frosted glass effect
    ...
}
```

---

## Quick Reference Card

```
┌──────────────────┬───────────────────────────────────────────┐
│ Concept          │ One-liner                                 │
├──────────────────┼───────────────────────────────────────────┤
│ Entity           │ Anything with identity + events           │
│ EntityId         │ Global unique ID (shared Model/View space)│
│ Model            │ Data entity, no UI, no WindowId           │
│ View             │ UI entity, has render(), lives in Window  │
│ AnyView          │ Type-erased View for uniform storage      │
│ ViewType         │ TypeId wrapper for action dispatch table  │
│ Action           │ Named event that bubbles up the view tree │
│ ActionCallback   │ Handler for an action, returns bool       │
│ Subscription     │ "Notify me when X emits an event"         │
│ Observation      │ "Notify me when X's data changes"         │
│ TaskCallback     │ Bridge from async Future/Stream → entity  │
│ Effect           │ Deferred work queued for batch processing │
│ RefCounts        │ Ref-counted entity lifecycle management   │
│ Presenter        │ Orchestrates layout → paint → present     │
│ Element          │ One node in the view's UI tree            │
└──────────────────┴───────────────────────────────────────────┘
```
