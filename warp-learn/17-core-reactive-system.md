# Reactive System — Effects, Subscriptions, Observations

> How notify()/emit() → flush_effects() → render() works end-to-end.
> The heartbeat that connects data changes to pixels on screen.

---

## Two Reactive Mechanisms

```
emit(event) + subscribe     notify() + observe
───────────────────────     ──────────────────
"WHAT happened" + data      "SOMETHING changed" (no data)
targeted action             generic refresh

Example:                    Example:
  emit(CommandFinished{      notify()
    exit_code: 0,             → observer: ctx.notify() → re-render
    duration: 5s
  })
  → subscriber: show notification

Use subscribe when: you care about WHAT happened
Use observe when:   you just need to re-render
```

---

## Wiring Up (at setup time)

```rust
// Subscribe: "when TerminalModel emits a specific event, call me"
ctx.subscribe_to_model(&terminal_handle, |me, event, ctx| {
    match event {
        CommandFinished(data) => me.show_notification(data),
        Bell => me.play_sound(),
    }
});

// Observe: "when TerminalModel changes at all, re-render"
ctx.observe(&terminal_handle, |me, _, ctx| {
    ctx.notify();  // mark my view for re-render
});
```

Stored in AppContext:
```
subscriptions: HashMap<EntityId, Vec<Subscription>>
observations:  HashMap<EntityId, Vec<Observation>>
```

---

## The Effects Queue

emit() and notify() don't do anything immediately — they push to a queue:

```rust
ctx.emit(SomeEvent)  → pending_effects.push(Effect::Event { ... })
ctx.notify()         → pending_effects.push(Effect::ModelNotification { ... })
```

`pending_effects: VecDeque<Effect>` — a flat list, drained synchronously.

---

## flush_effects() — The Drain Loop

Runs synchronously when `update()` returns. NOT on a separate thread.

```
update(|model, ctx| {
    model.do_something();
    ctx.emit(BlockCompleted{...});    // → queues Effect::Event
    ctx.notify();                      // → queues Effect::ModelNotification
})  // ← update returns here
│
└── flush_effects() runs immediately:
    │
    ├── Effect::Event(BlockCompleted)
    │     → calls all subscribers for this model
    │     → subscriber might call ctx.notify() → queues MORE effects
    │
    ├── Effect::ModelNotification
    │     → calls all observers for this model
    │     → observer calls ctx.notify() on its View → queues ViewNotification
    │
    ├── Effect::ViewNotification
    │     → window_invalidations[window_id].updated.insert(view_id)
    │
    └── (keeps draining until queue is empty)
    │
    After queue empty:
    → invalidation_callback → window.request_redraw()
```

---

## The Cascade

```
emit(BlockCompleted)
  │ subscriber callback fires
  │   └── ctx.notify() on this View
  │         │ queues Effect::ViewNotification
  │         │
  ▼         ▼
notify() on Model
  │ observer callback fires
  │   └── ctx.notify() on observing View
  │         │ queues Effect::ViewNotification
  │         ▼
  ▼       window_invalidations[window].updated.insert(view_id)
done    → request_redraw()
                │
                ▼ (next event loop iteration)
          RedrawRequested
                │
                ▼
          presenter.invalidate() → re-call render() for changed views
          build_scene() → layout → paint → Scene → GPU → pixels
```

---

## Batching (Why This Design)

```
model.set_title("new");     // notify → queues effect
model.set_status(Done);     // notify → queues effect (deduplicated!)
model.update_grid();        // notify → queues effect (deduplicated!)

flush_effects():
  only ONE ModelNotification (dedup in notify())
  → ONE ViewNotification
  → ONE request_redraw()
  → ONE render()

10 notify() calls = 1 render. Not 10.
```

---

## No Recursion Problem

```
Without effect queue (dangerous):
  notify() → observer runs → notify() → observer runs → notify() → STACK OVERFLOW

With effect queue (safe):
  notify() → push to queue → return immediately
  flush_effects() → flat loop, drains one at a time
  cascading effects just add to the end of the queue
```

---

## Who Calls flush_effects()?

```rust
// Every App::update() call:
pub fn update(&mut self, callback: F) -> T {
    state.pending_flushes += 1;
    let result = callback(&mut state);
    state.flush_effects();  // ← always flushes after your code
    result
}

// Also in AppContextRefMut Drop:
impl Drop for AppContextRefMut<'_> {
    fn drop(&mut self) {
        self.0.flush_effects();  // ← flushes when mutable access ends
    }
}
```

---

## Full Timeline: PTY Output → Pixels

```
PTY thread:
  pty.read(bytes) → event_listener.send_wakeup_event()

Main thread event loop:
  wakeup_rx fires → handle_terminal_wakeup()
    │
    ├── terminal_model.update(|model, ctx| {
    │     model.process_bytes(bytes);        // update grid
    │     ctx.emit(BlockCompleted{...});      // push Effect::Event
    │     ctx.notify();                       // push Effect::ModelNotification
    │   });
    │
    ├── flush_effects():
    │     Effect::Event → subscriber callbacks
    │       └── some call ctx.notify() → push ViewNotification
    │     Effect::ModelNotification → observer callbacks
    │       └── call ctx.notify() → push ViewNotification
    │     Effect::ViewNotification → mark window dirty
    │
    ├── invalidation_callback → window.request_redraw()
    │
    ▼ (next event loop iteration)
    
  RedrawRequested:
    presenter.invalidate() → re-call render() for dirty views
    build_scene() → layout() → paint() → Scene
    window.render(scene) → GPU → pixels on screen
```

---

## All Synchronous on Main Thread

```
Everything in one call stack:
  update() → your code → flush_effects() → callbacks → more effects → done

No race conditions.
No locks needed for the effect queue.
The ONLY async part: request_redraw() → OS → RedrawRequested (next frame).
```

---

## Quick Reference

```
emit(event)     → push Effect::Event         → subscribers get data
notify()        → push Effect::ModelNotif     → observers get wake-up
ctx.notify()    → push Effect::ViewNotif      → window marked dirty
                   (on a View context)

flush_effects() → drains queue synchronously after every update()
                → cascading: callbacks can queue more effects
                → deduplicates: multiple notify() = one re-render

request_redraw() → OS schedules RedrawRequested
render()         → builds new Element tree for changed views
build_scene()    → layout + paint → Scene → GPU → pixels
```
