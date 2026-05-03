# ModelHandle — The Handle System

> Source: `crates/warpui_core/src/core/model/handle.rs`

---

## Core Rule

**AppContext owns everything. Everyone else holds handles (just IDs).**

---

## Three Handle Types

```
  ModelHandle<T>          AnyModelHandle         WeakModelHandle<T>
  ──────────────          ──────────────         ─────────────────
  Strong, typed           Strong, type-erased    Weak, typed
  Keeps entity alive      Keeps entity alive     Does NOT keep alive
  clone → ref +1          clone → ref +1         No ref counting
  drop  → ref -1          drop  → ref -1         Just an ID
```

---

## ModelHandle<T> Internals

```rust
pub struct ModelHandle<T> {
    model_id: EntityId,                    // which entity
    model_type: PhantomData<T>,            // which type (0 bytes)
    ref_counts: Weak<Mutex<RefCounts>>,    // Weak → doesn't keep App alive
}
```

**Lifecycle:** `new → +1, clone → +1, drop → -1, zero → entity destroyed`

---

## Accessing Data

```rust
// Read: &T + &AppContext (look only, no identity needed)
handle.read(app, |model, ctx| model.title())

// Update: &mut T + &mut ModelContext<T> (change + emit/notify/spawn)
handle.update(app, |model, ctx| {
    model.set_title("new");
    ctx.notify();    // needs to know "I am EntityId(42)"
})
```

Why `ModelContext<T>` not `AppContext`? → `notify()`/`emit()` need to know
**which entity** is calling. `ModelContext<T>` = `&mut AppContext` + entity ID.

---

## Conversions

```
ModelHandle<T>  → .downgrade()   → WeakModelHandle<T>    (avoid ref cycles)
ModelHandle<T>  → .into()        → AnyModelHandle         (store mixed types)
AnyModelHandle  → .downcast::<T> → Option<ModelHandle<T>> (recover type)
WeakModelHandle → .upgrade(app)  → Option<ModelHandle<T>> (check if alive)
```

---

## Access Traits

```
render() gets &AppContext       → ReadModel  → can only read()
on_event() gets &mut Context    → UpdateModel → can read() + update()
```

---

## The Full Picture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                        AppContext                              │
  │                                                                │
  │  models: HashMap<EntityId, Box<dyn AnyModel>>                 │
  │          EntityId(1) → AuthState                               │
  │          EntityId(2) → TerminalModel                           │
  │          EntityId(3) → DriveIndex                              │
  │                                                                │
  │  ref_counts: Arc<Mutex<RefCounts>>                             │
  │              EntityId(1) → count: 3                            │
  │              EntityId(2) → count: 5                            │
  │              EntityId(3) → count: 1                            │
  └──────────┬──────────────────────┬─────────────────────────────┘
             │                      │
             │ Weak                 │ Weak
             ▼                      ▼
  ┌─────────────────────┐ ┌─────────────────────┐
  │ ModelHandle<Auth>    │ │ ModelHandle<Terminal>│
  │ model_id: 1          │ │ model_id: 2          │
  │ ref_counts: Weak     │ │ ref_counts: Weak     │
  │                      │ │                      │
  │  .read(app, |m| ...) │ │  .update(app, |m| ..)│
  │  "let me see Auth"   │ │  "let me change Term"│
  └─────────────────────┘ └─────────────────────┘
  held by: Workspace,      held by: BlockListView,
           SettingsView              AIController,
                                     PaneGroup
```
