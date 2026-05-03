# Warp Codebase — Deep Learning Plan

A structured 8-phase study plan for deeply understanding the entire Warp codebase. Each phase builds on the previous one. For every phase you'll find: **what to read**, **what to focus on**, **exercises** to solidify understanding, and **estimated effort**.

> Tip: keep a personal scratch file while reading. Jot down every type or pattern that surprises you — those are the load-bearing design decisions.

---

## Phase 0 — Orientation & Tooling (Day 1)

**Goal:** Know where everything lives, how to build, and how to run tests.

### Read
| File | Why |
|------|-----|
| [ARCHITECTURE.md](ARCHITECTURE.md) | High-level map you already have |
| [Cargo.toml](Cargo.toml) (workspace section) | See every crate and default-members |
| [rust-toolchain.toml](rust-toolchain.toml) | Which Rust version/components |
| [script/run](script/run) | How the app is launched |
| [script/presubmit](script/presubmit) | What CI checks |

### Do
```bash
# Build the project (takes a while first time)
cargo build

# Run fast-dev build
./script/run --dont-open --features fast_dev

# Run tests for a small crate to verify your setup
cargo test -p fuzzy_match
cargo test -p sum_tree
```

### Checkpoint
- [ ] You can build and run Warp locally.
- [ ] You can run tests for any individual crate.

---

## Phase 1 — Foundation Crates (Days 2–3)

**Goal:** Internalize the small data-structure and utility crates that every other crate depends on. These are self-contained and have excellent test coverage.

### Study order

#### 1.1 `string-offset` *(~215 lines)*
| File | Focus |
|------|-------|
| [crates/string-offset/src/lib.rs](crates/string-offset/src/lib.rs) | `CharOffset`, `ByteOffset` — type-safe position wrappers |
| [crates/string-offset/src/lib_tests.rs](crates/string-offset/src/lib_tests.rs) | How offset conversions are tested |

**Key concept:** Warp wraps raw `usize` positions in newtypes (`CharOffset`, `ByteOffset`) to prevent mixing byte vs. character vs. grapheme positions at compile time. This pattern appears *everywhere* in the editor and terminal.

#### 1.2 `sum_tree` *(~530 lines)*
| File | Focus |
|------|-------|
| [crates/sum_tree/src/lib.rs](crates/sum_tree/src/lib.rs) | `SumTree<T>`, `Item` trait, `Summary` trait |
| [crates/sum_tree/src/cursor.rs](crates/sum_tree/src/cursor.rs) | `Cursor` — how you traverse and query the tree |
| [crates/sum_tree/src/lib_test.rs](crates/sum_tree/src/lib_test.rs) | Insert, edit, seek, filter operations |

**Key concept:** This is the **most architecturally important data structure** in Warp. It's a copy-on-write B-tree where every node caches a `Summary`. The editor buffer, display map, and terminal grid all use it for O(log n) queries by any summarized dimension (byte offset, line number, character count, etc.).

**Exercise:** Write a tiny program that creates a `SumTree`, inserts items, and uses a `Cursor` to seek by different summary dimensions.

#### 1.3 `command` *(~17 lines)*
| File | Focus |
|------|-------|
| [crates/command/src/lib.rs](crates/command/src/lib.rs) | Cross-platform `Command` wrapper |

Trivially small. Just note how it wraps `std::process::Command` with platform flags (e.g., `CREATE_NO_WINDOW` on Windows).

#### 1.4 `fuzzy_match` *(~1000 lines)*
| File | Focus |
|------|-------|
| [crates/fuzzy_match/src/lib.rs](crates/fuzzy_match/src/lib.rs) | `match_indices`, `match_wildcard_pattern` |

Used by the command palette, completions, and file finder.

#### 1.5 `markdown_parser` *(~1600 lines)*
| File | Focus |
|------|-------|
| [crates/markdown_parser/src/lib.rs](crates/markdown_parser/src/lib.rs) | Fragment types, parsing pipeline |
| [crates/markdown_parser/src/markdown_parser.rs](crates/markdown_parser/src/markdown_parser.rs) | The actual parser |

Used to render AI responses, notebook cells, and any rich-text content.

#### 1.6 `warp_util`
| File | Focus |
|------|-------|
| [crates/warp_util/src/lib.rs](crates/warp_util/src/lib.rs) | Utility re-exports: paths, file types, assets |

### Checkpoint
- [ ] You understand `SumTree`, `Cursor`, `Item`, and `Summary` well enough to sketch them on a whiteboard.
- [ ] You can explain why `CharOffset` and `ByteOffset` are separate types.
- [ ] All foundation crate tests pass on your machine.

---

## Phase 2 — UI Framework: warpui_core (Days 4–7)

**Goal:** Understand the custom retained-mode UI framework that everything is painted on.

### 2.1 Core entity system
| File | Focus |
|------|-------|
| [crates/warpui_core/src/lib.rs](crates/warpui_core/src/lib.rs) | Top-level re-exports — this is the public API |
| [crates/warpui_core/src/core/mod.rs](crates/warpui_core/src/core/mod.rs) | `Entity`, `Model`, `View`, `Window`, `Action` |
| [crates/warpui_core/src/core/entity.rs](crates/warpui_core/src/core/entity.rs) | `Entity` trait — the base identity trait for **everything** |
| [crates/warpui_core/src/core/view.rs](crates/warpui_core/src/core/view.rs) | How views declare their element tree |
| [crates/warpui_core/src/core/model.rs](crates/warpui_core/src/core/model.rs) | Data model entities (non-visual) |
| [crates/warpui_core/src/core/action.rs](crates/warpui_core/src/core/action.rs) | `Action` trait — event dispatch |
| [crates/warpui_core/src/core/window.rs](crates/warpui_core/src/core/window.rs) | Window lifecycle |

**Key concepts:**
- `Entity` is the base trait — gives you identity, lifecycle, and change tracking
- `View` is an Entity that produces an element tree
- `Model` is an Entity without a visual representation
- `Action`s are dispatched through the element tree (like DOM events bubbling)

### 2.2 Element tree
| File | Focus |
|------|-------|
| [crates/warpui_core/src/elements/mod.rs](crates/warpui_core/src/elements/mod.rs) | Element catalog and `Element` trait |
| Layout elements: `flex.rs`, `align.rs`, `container.rs`, `constrained_box.rs` | Flexbox-like layout |
| Interaction: `hoverable.rs`, `event_handler.rs`, `drag.rs` | Input events |
| Display: `text.rs`, `icon.rs`, `image.rs`, `formatted_text_element.rs` | Rendering |
| Scrolling: `scrollable.rs`, `new_scrollable.rs` | Virtualized scroll |

### 2.3 Rendering pipeline
| File | Focus |
|------|-------|
| [crates/warpui_core/src/presenter.rs](crates/warpui_core/src/presenter.rs) | `Presenter` — orchestrates layout → paint → present |
| [crates/warpui_core/src/scene.rs](crates/warpui_core/src/scene.rs) | `Scene` — what gets submitted to the GPU |
| [crates/warpui_core/src/rendering/](crates/warpui_core/src/rendering/) | Abstract rendering traits |

### 2.4 Platform backends (warpui)
| File | Focus |
|------|-------|
| [crates/warpui/src/lib.rs](crates/warpui/src/lib.rs) | Thin wrapper, re-exports warpui_core |
| [crates/warpui/src/platform/mac/](crates/warpui/src/platform/mac/) | Metal renderer, Cocoa windowing |
| [crates/warpui/src/platform/windows/](crates/warpui/src/platform/windows/) | wgpu + winit on Windows |
| [crates/warpui/src/rendering/wgpu/](crates/warpui/src/rendering/wgpu/) | wgpu renderer, shaders |

### Exercise
Trace the lifecycle of a single button click:
1. OS event comes in through `platform/`
2. Dispatched through the element tree
3. An `Action` is fired
4. A `Model` or `View` handles it
5. The element tree is re-evaluated
6. `Presenter` runs layout → paint → present
7. `Scene` is submitted to Metal/wgpu

### Checkpoint
- [ ] You can explain the `Entity` → `View`/`Model` → `Element` hierarchy.
- [ ] You understand how layout, painting, and event dispatch work.
- [ ] You can find where a platform-specific key event enters the framework.

---

## Phase 3 — Terminal Engine (Days 8–11)

**Goal:** Understand how Warp emulates a terminal and how the block-based model works.

### 3.1 Low-level terminal model (`warp_terminal` crate)
| File | Focus |
|------|-------|
| [crates/warp_terminal/src/lib.rs](crates/warp_terminal/src/lib.rs) | Entry point, module structure |
| [crates/warp_terminal/src/model/](crates/warp_terminal/src/model/) | Grid, cells, cursor, scrollback |
| [crates/warp_terminal/src/model/ansi/](crates/warp_terminal/src/model/ansi/) | ANSI/VT escape sequence parser |
| [crates/warp_terminal/src/shell/](crates/warp_terminal/src/shell/) | Shell integration protocol |

**Key traits:** `Dimensions`, `ModeProvider`, `LineLength`, `Cell`

### 3.2 Block-based terminal (`app/src/terminal/`)
| File | Focus |
|------|-------|
| `app/src/terminal/model/terminal_model.rs` | `TerminalModel` — the main state machine |
| `app/src/terminal/model/block.rs` | `Block` — one command + output |
| `app/src/terminal/model/blockgrid.rs` | `BlockGrid` — cells within a block |
| `app/src/terminal/model/header_grid.rs` | `HeaderGrid` — command input area |
| `app/src/terminal/model/grid_handler.rs` | `GridHandler` — escape-sequence handler |
| `app/src/terminal/model/alt_screen.rs` | `AltScreen` — fullscreen apps (vim, less) |
| `app/src/terminal/block_list_element.rs` | `BlockListElement` — renders the block list |

### 3.3 PTY & shell interaction
| File | Focus |
|------|-------|
| `app/src/terminal/local_tty/` | Local PTY creation and management |
| `app/src/terminal/local_tty/shell.rs` | Shell starters (including `WslShellStarter`) |
| `app/src/terminal/shared_session/` | Session sharing |

### Exercise
1. Trace what happens when you type `ls` and press Enter:
   - Input goes to the PTY
   - Shell outputs ANSI sequences
   - `GridHandler` parses them
   - A new `Block` is created
   - `BlockListElement` renders it
2. Read how `AltScreen` takes over when you open `vim`.

### Checkpoint
- [ ] You can explain the Block abstraction vs. a traditional scrollback buffer.
- [ ] You know where ANSI parsing happens and how grid cells are updated.
- [ ] You understand how shell integration protocol segments commands from output.

---

## Phase 4 — Editor System (Days 12–15)

**Goal:** Understand the multi-layer editor used for command input, code editing, AI responses, and notebooks.

### 4.1 Editor library (`crates/editor/`)
| File | Focus |
|------|-------|
| [crates/editor/src/lib.rs](crates/editor/src/lib.rs) | Module structure, key trait re-exports |
| [crates/editor/src/model.rs](crates/editor/src/model.rs) | `CoreEditorModel`, `PlainTextEditorModel`, `RichTextEditorModel` |
| [crates/editor/src/editor.rs](crates/editor/src/editor.rs) | `EditorView` trait |
| [crates/editor/src/content/](crates/editor/src/content/) | Buffer content model (uses `SumTree`!) |
| [crates/editor/src/selection.rs](crates/editor/src/selection.rs) | Selection handling |
| [crates/editor/src/render/](crates/editor/src/render/) | Text rendering pipeline |

**Key insight:** The editor model is a trait hierarchy:
```
CoreEditorModel          (base: any editable text)
  └── PlainTextEditorModel   (plain text)
  └── RichTextEditorModel    (styled text, embeds)
```

### 4.2 App-level editor (`app/src/editor/`)
| File | Focus |
|------|-------|
| `app/src/editor/view/mod.rs` | `EditorView`, `EditorModel`, `EditorAction` |
| `app/src/editor/view/element.rs` | `EditorElement` — GPU rendering |
| `app/src/editor/view/model/buffer/mod.rs` | `Buffer` — CRDT-like with `ReplicaId` |
| `app/src/editor/view/model/buffer/anchor.rs` | `Anchor` — stable position that survives edits |
| `app/src/editor/view/model/display_map/mod.rs` | `DisplayMap`, `DisplayPoint` — logical → screen |

### 4.3 Code editor (`app/src/code/`)
| File | Focus |
|------|-------|
| `app/src/code/editor/view.rs` | `CodeEditorView` |
| `app/src/code/editor/model.rs` | `CodeEditorModel` |
| `app/src/code/file_tree/view.rs` | `FileTreeView` |
| `app/src/code/local_code_editor.rs` | `LocalCodeEditorView` |

### Exercise
1. Trace how an `Anchor` stays valid when text is inserted before it (hint: `ReplicaId` + logical clocks).
2. Understand how `DisplayMap` maps buffer positions through soft wraps and folds to screen coordinates.

### Checkpoint
- [ ] You can draw the trait hierarchy: `CoreEditorModel` → `PlainTextEditorModel` → `RichTextEditorModel`.
- [ ] You understand `Buffer` → `SumTree` → `Anchor` → `DisplayMap` → screen.
- [ ] You know where LSP diagnostics enter the code editor model.

---

## Phase 5 — AI & Agent System (Days 16–20)

**Goal:** Understand the full AI pipeline — from user prompt to streamed response to tool execution.

### 5.1 AI library (`crates/ai/`)
| File | Focus |
|------|-------|
| [crates/ai/src/lib.rs](crates/ai/src/lib.rs) | Modules overview |
| [crates/ai/src/agent/](crates/ai/src/agent/) | Agent execution engine |
| [crates/ai/src/llm_id.rs](crates/ai/src/llm_id.rs) | `LLMId` — model selection |
| [crates/ai/src/skills/](crates/ai/src/skills/) | Skill definitions |
| [crates/ai/src/project_context/](crates/ai/src/project_context/) | Workspace context gathering |
| [crates/ai/src/index/](crates/ai/src/index/) | Code indexing for context |
| [crates/ai/src/diff_validation/](crates/ai/src/diff_validation/) | Validating AI-generated diffs |

### 5.2 AI application module (`app/src/ai/`)
| File | Focus |
|------|-------|
| `app/src/ai/agent/mod.rs` | `AIAgentExchange`, `AIAgentAction`, `AIAgentActionId` |
| `app/src/ai/agent/conversation.rs` | `AIConversation`, `ConversationStatus` |
| `app/src/ai/agent/task.rs` | `Task`, `TaskId` |
| `app/src/ai/agent/api.rs` | `ServerConversationToken` — server communication |
| `app/src/ai/agent_sdk/driver.rs` | `AgentDriverError` — agent execution driver |
| `app/src/ai/blocklist/controller.rs` | `BlocklistAIController` — main AI UI controller |
| `app/src/ai/blocklist/agent_view/controller.rs` | `AgentViewController`, `AgentViewState` |
| `app/src/ai/blocklist/block.rs` | `AIBlock` — AI output as a terminal block |
| `app/src/ai/blocklist/inline_action/code_diff_view.rs` | `CodeDiffView` — inline code changes |
| `app/src/ai/ambient_agents/mod.rs` | `AmbientAgentTaskId` — background agents |
| `app/src/ai/execution_profiles/profiles.rs` | `ClientProfileId` — execution profiles |

### 5.3 Data flow
```
User types prompt
  → BlocklistAIController receives input
  → AIConversation is created/continued
  → AIAgentExchange is sent to server (via ServerConversationToken)
  → Streaming response arrives (ResponseStreamId)
  → Agent decides on AIAgentActions (tool calls, code edits, terminal commands)
  → Actions are executed (AgentDriver)
  → Results stream back as AIBlocks
  → CodeDiffView shows inline changes
```

### Exercise
1. Find where MCP (Model Context Protocol) server connections are established.
2. Trace a tool-call action from the agent response through to execution.
3. Read how `diff_validation` checks AI-generated code changes before applying.

### Checkpoint
- [ ] You can draw the `AIConversation` → `AIAgentExchange` → `AIAgentAction` → `Task` flow.
- [ ] You understand how the agent SDK driver orchestrates multi-step tool calls.
- [ ] You know how ambient agents differ from interactive conversations.

---

## Phase 6 — Application Layer (Days 21–25)

**Goal:** Understand how everything is wired together at the top level.

### 6.1 Startup & entry points
| File | Focus |
|------|-------|
| `app/src/bin/local.rs` | Dev build entry — loads `ChannelState`, calls `warp::run()` |
| `app/src/bin/stable.rs` | Production entry |
| `app/src/lib.rs` | The `run()` function, all module declarations |

**Startup sequence:**
1. Load channel config (local/stable/preview/oss)
2. Set feature flags via `ChannelState`
3. `warp::run()` initializes the UI framework, creates the root window

### 6.2 View hierarchy
| File | Focus |
|------|-------|
| `app/src/root_view.rs` | `RootView` — top-level container |
| `app/src/workspace/view.rs` | `Workspace` — manages panes, tabs, overlays |
| `app/src/pane_group/mod.rs` | `PaneGroup` — recursive split pane layout |
| `app/src/pane_group/pane/` | Individual pane with tabs |
| `app/src/settings_view/mod.rs` | `SettingsView` |
| `app/src/search/command_palette/view.rs` | Command palette |

### 6.3 Cloud & Drive
| File | Focus |
|------|-------|
| `app/src/cloud_object/mod.rs` | `CloudObject` trait, `Space` enum |
| `app/src/drive/mod.rs` | `DriveObjectType`, Warp Drive |
| `app/src/drive/index.rs` | `DriveIndex` — local index of cloud items |
| `app/src/server/` | Server communication orchestration |
| `app/src/auth/auth_state.rs` | `AuthState` |

### 6.4 Remaining subsystems
| Module | Key files | Focus |
|--------|-----------|-------|
| Notebooks | `app/src/notebooks/` | Rich-text notebook editing |
| Code Review | `app/src/code_review/` | Diff views, comments |
| Completions | `app/src/completer/`, `app/src/input_suggestions.rs` | Shell autocomplete UI |
| Env Vars | `app/src/env_vars/` | Environment variable management |
| Workflows | `app/src/workflows/` | Saved command workflows |
| Themes | `app/src/themes/` | Theme system |
| Settings | `app/src/settings/`, `crates/settings/` | Settings framework |

### Exercise
1. Starting from `RootView`, trace the full element tree down to a terminal block.
2. Find how a keybinding (e.g., Ctrl+P for command palette) flows from OS event to `CommandPalette::show()`.

### Checkpoint
- [ ] You can trace the full startup → window → workspace → pane → terminal path.
- [ ] You understand how `CloudObject` syncs to the backend.
- [ ] You know how feature flags gate functionality at compile time vs. runtime.

---

## Phase 7 — Testing & Integration (Days 26–28)

**Goal:** Understand the testing infrastructure and write your own tests.

### 7.1 Unit tests
| Pattern | Example |
|---------|---------|
| Inline `#[cfg(test)] mod tests` | Most crates |
| External test file (`lib_tests.rs`) | `string-offset`, `sum_tree` |
| Test utilities module | `app/src/test_util/` |

### 7.2 Integration tests
| File | Focus |
|------|-------|
| [crates/integration/src/test.rs](crates/integration/src/test.rs) | Test framework entry — `TestStep`, `Builder` |
| [crates/integration/tests/](crates/integration/tests/) | Test suites |
| [crates/integration/src/bin/](crates/integration/src/bin/) | Manual test runners |

The integration test framework provides:
- `Builder` — constructs a test terminal environment
- `TestStep` — declarative step-based testing
- Terminal simulation with real escape-sequence processing
- UI assertion helpers

### 7.3 Running tests
```bash
# Unit tests for a specific crate
cargo test -p sum_tree
cargo test -p warp_terminal

# All default-member tests
cargo test

# Integration tests (requires special setup)
cargo test -p integration

# Run presubmit (what CI runs)
./script/presubmit
```

### Exercise
1. Write a unit test for `fuzzy_match` that tests a new edge case.
2. Read one integration test end-to-end and understand each `TestStep`.

### Checkpoint
- [ ] You've run tests at the unit, crate, and integration level.
- [ ] You can write a new test using the `Builder`/`TestStep` framework.

---

## Phase 8 — Mastery Projects (Days 29–35+)

**Goal:** Solidify your understanding by working on real changes.

### Project ideas (increasing difficulty)

1. **Add a new keybinding** — Wire a new keyboard shortcut through `keymap` → `Action` → handler. Touches: warpui_core, app.

2. **Add a new setting** — Create a new user-visible setting with the `settings` derive macro. Touches: settings crate, settings_view.

3. **Fix a terminal rendering edge case** — Find and fix a subtle ANSI handling bug. Touches: warp_terminal, GridHandler.

4. **Add a new completion source** — Extend the completer with a new data source. Touches: warp_completer, input_suggestions.

5. **Build a new AI skill** — Add a new skill to the agent system. Touches: crates/ai/skills, app/src/ai.

6. **Add a new Element to warpui_core** — Create a new layout or display element. Touches: warpui_core/elements.

---

## Quick Reference: Most Important Files

| What | Where |
|------|-------|
| Workspace Cargo.toml | [Cargo.toml](Cargo.toml) |
| App entry (dev) | [app/src/bin/local.rs](app/src/bin/local.rs) |
| App lib (all modules) | [app/src/lib.rs](app/src/lib.rs) |
| Root view | [app/src/root_view.rs](app/src/root_view.rs) |
| Entity trait | [crates/warpui_core/src/core/entity.rs](crates/warpui_core/src/core/entity.rs) |
| Element trait | [crates/warpui_core/src/elements/mod.rs](crates/warpui_core/src/elements/mod.rs) |
| SumTree | [crates/sum_tree/src/lib.rs](crates/sum_tree/src/lib.rs) |
| Terminal model | `app/src/terminal/model/terminal_model.rs` |
| Block model | `app/src/terminal/model/block.rs` |
| ANSI parser | [crates/warp_terminal/src/model/ansi/](crates/warp_terminal/src/model/ansi/) |
| Editor model | [crates/editor/src/model.rs](crates/editor/src/model.rs) |
| Buffer | `app/src/editor/view/model/buffer/mod.rs` |
| AI conversation | `app/src/ai/agent/conversation.rs` |
| AI controller | `app/src/ai/blocklist/controller.rs` |
| Feature flags | [crates/warp_features/](crates/warp_features/) |
| Integration tests | [crates/integration/src/test.rs](crates/integration/src/test.rs) |
