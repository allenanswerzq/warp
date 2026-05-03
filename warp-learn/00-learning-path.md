# Learning Path — From UI Framework to Full App

> Track progress as we learn the entire Warp codebase.

---

## ✅ Phase 0: warpui Framework (DONE)

```
Docs 01-13 in warp-learn/

01  Core concepts        Entity, Model, View, Actions, Subscriptions, async
02  ModelHandle          Handle system, ref counting, access patterns
03  Elements             UI building blocks catalog
04  Flex layout          Flex, paint, dispatch_event
05  Scene                Draw buffer, layers, z-ordering
06  Presenter            Frame pipeline, invalidation
07  Platform traits      Delegate, Window, FontDB contracts
08  Fonts                Font pipeline
09  Events               Event enum, z-index filtering
10  Metal renderer       GPU rendering, shaders, SDF, instancing
11  Event loop           winit loop, all event types
12  OS Window            Window vs View, two Window structs
13  GPU landscape        Metal/Vulkan/wgpu/WGSL/MSL background
```

---

## ⬜ Phase A: Terminal Engine (NEXT)

The heart of the product. Warp IS a terminal.

```
Read order:
  1. crates/warp_terminal/src/lib.rs          → module overview
  2. crates/warp_terminal/src/model/          → grid, cells, cursor
  3. crates/warp_terminal/src/model/ansi/     → ANSI escape sequence parser
  4. crates/warp_terminal/src/shell/          → shell integration protocol
  5. app/src/terminal/model/terminal_model.rs → TerminalModel (main state)
  6. app/src/terminal/model/block.rs          → Block (command + output)
  7. app/src/terminal/model/blockgrid.rs      → BlockGrid (cells in a block)
  8. app/src/terminal/model/grid_handler.rs   → escape sequence handler
  9. app/src/terminal/model/alt_screen.rs     → fullscreen apps (vim, less)
 10. app/src/terminal/block_list_element.rs   → renders blocks to screen
 11. app/src/terminal/local_tty/             → PTY creation, shell starters

Key concepts:
  - Block-based terminal (vs traditional scrollback)
  - ANSI/VT escape sequence parsing
  - Shell integration protocol (command boundaries)
  - PTY (pseudo-terminal) communication
```

---

## ⬜ Phase B: Editor System

Used everywhere: command input, AI chat, notebooks, code editor.

```
Read order:
  1. crates/editor/src/lib.rs                → module overview
  2. crates/editor/src/model.rs              → CoreEditorModel trait hierarchy
  3. crates/editor/src/content/              → buffer content (uses SumTree)
  4. crates/editor/src/selection.rs          → selection handling
  5. crates/editor/src/render/               → text rendering pipeline
  6. app/src/editor/view/mod.rs              → EditorView, EditorModel
  7. app/src/editor/view/model/buffer/       → Buffer, Anchor, ReplicaId
  8. app/src/editor/view/model/display_map/  → buffer → screen position mapping
  9. app/src/code/editor/                    → code editor (LSP, file tree)

Key concepts:
  - SumTree-backed buffer
  - Anchor (stable position across edits)
  - DisplayMap (logical → screen coords)
  - Editor trait hierarchy (Core → Plain → Rich)
```

---

## ⬜ Phase C: AI & Agent System

The biggest differentiator — AI conversations, tool calls, streaming.

```
Read order:
  1. crates/ai/src/lib.rs                   → module overview
  2. crates/ai/src/agent/                   → agent execution engine
  3. crates/ai/src/skills/                  → skill definitions
  4. crates/ai/src/project_context/         → workspace context gathering
  5. app/src/ai/agent/conversation.rs       → AIConversation
  6. app/src/ai/agent/mod.rs                → Exchange, Action, Task
  7. app/src/ai/agent_sdk/driver.rs         → agent driver
  8. app/src/ai/blocklist/controller.rs     → AI UI controller
  9. app/src/ai/blocklist/agent_view/       → conversation view
 10. app/src/ai/ambient_agents/             → background agents

Key concepts:
  - Conversation → Exchange → Action → Task flow
  - Streaming responses (ResponseStreamId)
  - Tool calls and multi-step execution
  - Code diff validation
```

---

## ⬜ Phase D: App Layer

How everything is wired together at the top level.

```
Read order:
  1. app/src/bin/local.rs                   → dev entry point
  2. app/src/lib.rs                         → all module declarations
  3. app/src/root_view.rs                   → RootView (top-level)
  4. app/src/workspace/view.rs              → Workspace
  5. app/src/pane_group/mod.rs              → PaneGroup (split tabs)
  6. app/src/search/command_palette/        → command palette
  7. app/src/settings_view/mod.rs           → settings UI

Key concepts:
  - Startup sequence (ChannelState → warp::run())
  - View hierarchy (Root → Workspace → PaneGroup → Pane)
  - Feature flags and channel gating
```

---

## ⬜ Phase E: Server & Cloud

Auth, GraphQL, Warp Drive cloud sync.

```
Read order:
  1. crates/warp_server_client/src/         → auth, cloud API
  2. crates/graphql/src/                    → GraphQL queries/mutations
  3. app/src/cloud_object/mod.rs            → CloudObject trait
  4. app/src/drive/                         → Warp Drive items
  5. app/src/auth/                          → authentication
  6. app/src/server/                        → server communication

Key concepts:
  - CloudObject trait for synced items
  - GraphQL typed queries
  - WebSocket real-time updates
```

---

## ⬜ Phase F: Other Crates

Completions, settings, vim, persistence.

```
Read order:
  1. crates/warp_completer/                 → shell completion engine
  2. crates/settings/                       → settings framework
  3. crates/warp_core/                      → channels, telemetry
  4. crates/vim/                            → vim keybindings
  5. crates/persistence/                    → local data storage
  6. crates/warp_features/                  → feature flags
  7. crates/sum_tree/                       → the core data structure

Key concepts:
  - Completion parsers and signatures
  - Settings derive macros
  - Feature flag compile-time vs runtime gating
```

---

## Progress Tracker

```
Phase  Status       Docs Created
─────  ──────       ────────────
  0    ✅ DONE      01 through 13
  A    ⬜ NEXT      (start here)
  B    ⬜ TODO
  C    ⬜ TODO
  D    ⬜ TODO
  E    ⬜ TODO
  F    ⬜ TODO
```
