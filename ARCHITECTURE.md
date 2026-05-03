# Warp Architecture Guide

## What is Warp?

Warp is a **modern, GPU-accelerated terminal emulator** written entirely in Rust. It reimagines the terminal as a block-based, AI-powered development environment with a custom UI framework, integrated code editor, cloud-synced workflows, and an AI agent system.

---

## Repository Structure

```
warp/
├── app/                    # Main application crate ("warp")
│   ├── src/
│   │   ├── ai/             # AI agent system, conversations, blocklist UI
│   │   ├── terminal/       # Terminal model, blocks, PTY, alt screen
│   │   ├── editor/         # Buffer, display map, editor view
│   │   ├── code/           # Code editor, file tree, LSP integration
│   │   ├── workspace/      # Workspace view orchestration
│   │   ├── pane_group/     # Tab/pane layout management
│   │   ├── drive/          # Warp Drive (cloud commands, workflows)
│   │   ├── notebooks/      # Rich-text notebooks
│   │   ├── search/         # Command palette, AI context menus
│   │   ├── settings_view/  # Settings UI pages
│   │   ├── code_review/    # Diff views, review comments
│   │   ├── auth/           # Authentication
│   │   ├── server/         # Server communication
│   │   └── ...
│   ├── assets/             # Images, icons
│   └── resources/          # Bundled resources
├── crates/                 # ~40+ library crates
│   ├── warpui/             # Custom UI: rendering, fonts, windowing
│   ├── warpui_core/        # UI primitives: elements, events, text layout
│   ├── warp_terminal/      # Terminal emulation engine
│   ├── editor/             # Text editor engine (warp_editor)
│   ├── ai/                 # AI/LLM integration library
│   ├── warp_core/          # Core: channels, telemetry, session
│   ├── warp_completer/     # Shell completion engine
│   ├── graphql/            # GraphQL client (warp_graphql)
│   ├── warp_server_client/ # Server API client
│   └── ...
├── resources/              # Bundled skills, Linux resources
├── script/                 # Build, deploy, CI scripts
└── specs/                  # Product & technical specs
```

---

## Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Application (app/)                     │
│  AI · Terminal · Editor · Code · Drive · Notebooks · ... │
├────────────────────┬────────────────────────────────────┤
│   UI Framework     │        Domain Libraries            │
│  warpui            │  ai, warp_completer, lsp,          │
│  warpui_core       │  warp_terminal, warp_editor,       │
│  ui_components     │  repo_metadata, computer_use       │
├────────────────────┼────────────────────────────────────┤
│   Core Services    │     Server / Cloud                  │
│  warp_core         │  warp_server_client                 │
│  settings          │  warp_graphql                       │
│  persistence       │  http_client, websocket             │
│  warp_logging      │  firebase                           │
├────────────────────┴────────────────────────────────────┤
│                   Foundation Crates                       │
│  string-offset · sum_tree · fuzzy_match · markdown_parser│
│  warp_util · command · warp_files · channel_versions     │
└─────────────────────────────────────────────────────────┘
```

### Layer Roles

| Layer | Crates | Responsibility |
|-------|--------|----------------|
| **Application** | `app` (the `warp` crate) | Top-level binary. Wires together all features, defines views, handles routing |
| **UI Framework** | `warpui`, `warpui_core`, `ui_components` | Custom GPU-rendered retained-mode UI: elements, events, text, scene graph |
| **Domain** | `ai`, `warp_terminal`, `warp_editor`, `lsp`, `warp_completer`, etc. | Feature-specific logic decoupled from the UI |
| **Core** | `warp_core`, `settings`, `persistence`, `warp_logging` | Cross-cutting concerns: telemetry, config, storage |
| **Server/Cloud** | `warp_server_client`, `warp_graphql`, `http_client`, `websocket` | Communication with Warp's backend services |
| **Foundation** | `string-offset`, `sum_tree`, `fuzzy_match`, `markdown_parser`, `warp_util` | Shared low-level data structures and utilities |

---

## Custom UI Framework (warpui + warpui_core)

Warp does **not** use GPUI, egui, iced, or any off-the-shelf Rust UI crate. It has a fully custom, GPU-accelerated retained-mode UI framework.

### Rendering Stack

| Platform | GPU API | Windowing | Text Shaping |
|----------|---------|-----------|--------------|
| macOS | **Metal** (native) | Cocoa / AppKit | Core Text |
| Linux / Windows | **wgpu** (Vulkan / DX12) | winit | cosmic-text |
| WASM | **wgpu** (WebGL) | web-sys | cosmic-text |

> There is an `experimental-wgpu-renderer` feature flag, indicating work to unify all platforms on wgpu.

### warpui_core Modules

| Module | Purpose |
|--------|---------|
| `core` | Application lifecycle, entity system |
| `elements` | Element tree (the "DOM" equivalent) |
| `rendering` | Abstract render layer |
| `scene` | Scene graph for GPU submission |
| `text` / `text_layout` | Text measurement and layout |
| `event` | Input event handling |
| `keymap` | Keybinding system |
| `presenter` | Coordinates layout → paint → present |
| `windowing` | Window management abstractions |
| `assets` | Asset loading and caching |
| `clipboard` | Clipboard integration |
| `ui_components` | Common widgets (buttons, inputs, etc.) |

### warpui Platform Backends

| Module | Files |
|--------|-------|
| `platform/mac/` | Cocoa app, Metal renderer, AppKit windows |
| `platform/linux/` | Linux-specific adaptations |
| `platform/windows/` | Windows-specific adaptations |
| `platform/wasm/` | Browser target |
| `rendering/wgpu/` | Cross-platform wgpu renderer with custom shaders |

---

## Terminal Engine

Warp's terminal is **block-based** — each command and its output form a discrete "Block" rather than a continuous scrollback buffer.

### Key Types

| Type | Location | Role |
|------|----------|------|
| `TerminalModel` | `app/src/terminal/model/terminal_model.rs` | Core terminal state machine |
| `Block` | `app/src/terminal/model/block.rs` | A single command + output unit |
| `BlockGrid` | `app/src/terminal/model/blockgrid.rs` | Grid of terminal cells within a block |
| `HeaderGrid` | `app/src/terminal/model/header_grid.rs` | Command input area of a block |
| `GridHandler` | `app/src/terminal/model/grid_handler.rs` | Processes terminal escape sequences |
| `AltScreen` | `app/src/terminal/model/alt_screen.rs` | Alternate screen buffer (vim, less, etc.) |
| `BlockListElement` | `app/src/terminal/block_list_element.rs` | UI element rendering the block list |

### Terminal Crate (`warp_terminal`)

| Module | Purpose |
|--------|---------|
| `model/ansi/` | ANSI/VT escape sequence parser |
| `model/` | Terminal grid, cursor, cell model |
| `shell/` | Shell integration protocol |

### PTY & Shell

- Local PTY management lives in `app/src/terminal/local_tty/`
- Shell starters are platform-specific (e.g. `WslShellStarter` for Windows WSL)
- Remote sessions use `remote_server` crate for SSH-like connections
- Shared sessions via `terminal/shared_session/`

---

## AI & Agent System

The AI system is one of Warp's largest subsystems, spanning the `ai` library crate and the `app/src/ai/` application module.

### AI Library Crate (`crates/ai/`)

| Module | Purpose |
|--------|---------|
| `agent` | Agent execution, conversations, exchanges, tasks |
| `skills` | Agent skill definitions and execution |
| `project_context` | Workspace/project context gathering |
| `index` | Code indexing for AI context |
| `diff_validation` | Validates AI-generated diffs |
| `document` | Document model for AI context |

### AI Application Module (`app/src/ai/`)

| Module | Purpose |
|--------|---------|
| `blocklist/` | AI conversation UI rendered as a block list |
| `blocklist/agent_view/` | Agent conversation view + controller |
| `blocklist/inline_action/` | Inline code actions (CodeDiffView) |
| `agent/` | Conversations, exchanges, tasks, API |
| `agent_sdk/` | Agent driver, error handling |
| `ambient_agents/` | Background AI agents |
| `agent_management/` | Agent configuration UI |
| `execution_profiles/` | Client execution profiles |
| `agent_conversations_model.rs` | Conversation list model |
| `persisted_workspace.rs` | AI workspace persistence |

### Key AI Types

| Type | Role |
|------|------|
| `AIConversation` | A multi-turn AI conversation |
| `AIAgentExchange` | A single request-response pair |
| `AIAgentAction` | An action the agent wants to take |
| `Task` / `TaskId` | A unit of agent work |
| `BlocklistAIController` | Main AI UI controller |
| `AgentViewController` | Agent conversation view state |
| `ResponseStreamId` | Identifies a streaming response |

---

## Editor System

The editor is split across a library crate and the application layer.

### Editor Library (`crates/editor/`)

| Module | Purpose |
|--------|---------|
| `content/` | Document content model (text buffer, syntax, decorations) |
| `render/` | Text rendering pipeline |

### Editor Application (`app/src/editor/`)

| Type | Role |
|------|------|
| `EditorModel` | Editor state: buffer, selections, undo |
| `EditorView` | Main editor view element |
| `EditorElement` | Low-level GPU rendering element |
| `Buffer` | Text buffer with CRDT-like `ReplicaId` for collaboration |
| `DisplayMap` / `DisplayPoint` | Maps buffer positions to screen coordinates |
| `Anchor` | Stable position reference that survives edits |

### Code Editor (`app/src/code/`)

Built on top of the editor engine, adds:
- `CodeEditorView` / `CodeEditorModel` — Full code editing experience
- `FileTreeView` — File explorer sidebar
- LSP integration (diagnostics, go-to-definition, completions)
- Code review with diff views and comments

---

## Workspace & Layout

```
RootView
└── Workspace
    ├── PaneGroup (split layout)
    │   ├── Pane (tab container)
    │   │   ├── Terminal tab
    │   │   ├── Code editor tab
    │   │   ├── Notebook tab
    │   │   └── AI conversation tab
    │   └── Pane
    │       └── ...
    ├── SettingsView (modal overlay)
    ├── Search / CommandPalette
    └── Modals, Toasts, Banners
```

| Type | Location | Role |
|------|----------|------|
| `RootView` | `app/src/root_view.rs` | Top-level view, owns app state |
| `Workspace` | `app/src/workspace/view.rs` | Workspace orchestration |
| `PaneGroup` | `app/src/pane_group/mod.rs` | Recursive split pane layout |
| `SettingsView` | `app/src/settings_view/mod.rs` | Settings UI host |

---

## Warp Drive (Cloud)

Warp Drive syncs commands, workflows, notebooks, and environment variables to the cloud.

| Type / Module | Role |
|---------------|------|
| `DriveIndex` | Local index of all Drive items |
| `WarpDriveItemId` | Unique identifier for a Drive item |
| `CloudObject` (trait) | Interface for cloud-synced objects |
| `CloudModel` | Persistence model for cloud objects |
| `SharingDialog` | UI for sharing Drive items with teams |
| `WorkflowModal` | UI for creating/editing workflows |

### Server Communication

| Crate | Role |
|-------|------|
| `warp_server_client` | Auth, REST/GraphQL client for Warp backend |
| `warp_graphql` | Typed GraphQL queries and mutations |
| `http_client` | HTTP request abstraction |
| `websocket` | Real-time updates |

---

## Completion Engine

| Crate / Module | Role |
|----------------|------|
| `warp_completer` | Core completion engine |
| `warp_completer/completer/` | Completion logic and ranking |
| `warp_completer/parsers/` | Shell command parsers |
| `warp_completer/signatures/` | Command signature database |
| `command-signatures-v2/` | V2 signature data (JS-based) |
| `fuzzy_match` | Fuzzy text matching for filtering |
| `app/src/input_suggestions.rs` | UI for showing suggestions |
| `app/src/completer/` | Application-level completion orchestration |

---

## Other Notable Subsystems

### Settings
- `crates/settings/` — Settings framework with derive macros
- `app/src/settings/` + `app/src/settings_view/` — Per-page settings UI (AI, appearance, billing, teams, MCP servers, features)

### Notebooks
- `app/src/notebooks/` — Rich-text notebook editing with `RichTextEditorView`, shareable via Drive

### Code Review
- `app/src/code_review/` — `CodeReviewView`, `DiffStateModel`, `AttachedReviewComment`

### Platform Support
- `crates/computer_use/` — OS-level automation with per-platform implementations (`mac/`, `linux/`, `windows/`)
- `crates/isolation_platform/` — Sandboxing / isolation
- `crates/ipc/` — Inter-process communication
- `crates/remote_server/` — Remote terminal connections

### Feature Flags
- `crates/warp_features/` — Feature flag definitions
- Channel-gated rollout: Stable → Preview → Dogfood

---

## Key Design Patterns

### 1. Block-Based Terminal
Traditional terminals use a raw scrollback buffer. Warp segments output into **Blocks** — each Block wraps a command and its output, enabling per-command actions (copy, share, re-run, AI explain).

### 2. Retained-Mode Custom UI
`warpui_core` implements a retained-mode element tree with a scene graph. Views declare their element hierarchy; the framework handles layout, hit-testing, event dispatch, and GPU rendering.

### 3. Agent-Conversation Architecture
AI interactions are modeled as `AIConversation` → `AIAgentExchange` → `AIAgentAction`. Conversations persist, support streaming responses, and agents can execute multi-step tasks with tool calls.

### 4. Cloud-Synced Objects
A `CloudObject` trait provides a uniform interface for items that sync to Warp Drive (workflows, notebooks, env var collections). Sync happens via GraphQL mutations.

### 5. Multi-Platform with Platform Crates
Platform-specific code is isolated into per-OS modules (`mac/`, `linux/`, `windows/`, `wasm/`) within `warpui` and `computer_use`, keeping the application layer platform-agnostic.

### 6. Crate Isolation
The ~40+ crate workspace enforces compile-time boundaries. Feature code in `crates/` cannot depend on `app/`, keeping libraries reusable and independently testable.

---

## Build & Run

```bash
# Build the app
cargo build

# Run with fast_dev feature (faster iteration)
./script/run --dont-open --features fast_dev

# Run presubmit checks
./script/presubmit

# Run tests for a specific crate
cargo test -p warp_terminal
```

---

## Dependency Flow (Simplified)

```
app (warp)
 ├─→ ai ─→ lsp, warp_core, warp_editor, warp_terminal
 ├─→ warp_terminal ─→ string-offset, warp_editor
 ├─→ warp_editor ─→ string-offset, sum_tree, markdown_parser, warpui_core
 ├─→ warpui ─→ warpui_core, warp_core
 ├─→ warp_completer ─→ warp_editor, warp_js, command
 ├─→ warp_server_client ─→ warp_graphql, warp_editor
 ├─→ warp_core ─→ warp_editor, warpui_core
 ├─→ repo_metadata ─→ warp_util, warpui_core
 └─→ settings, persistence, warp_logging, ...
```

> `warp_editor` is the most depended-upon crate — nearly every crate links against it, as it provides the core text/content model used across the terminal, AI, code editor, and notebooks.
