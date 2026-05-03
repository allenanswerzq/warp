# app/src/terminal/ — Warp's Block-Based Terminal

> Source: `app/src/terminal/`
> Where the raw terminal emulator becomes Warp's block-based, AI-powered terminal.

---

## The Key Idea: Blocks

```
Traditional terminal:              Warp terminal:
──────────────────                 ──────────────
One continuous scrollback          Segmented into Blocks

$ git status                       ┌─ Block 1 ──────────────┐
On branch main                     │ $ git status            │ ← HeaderGrid
Changes not staged:                │ On branch main          │ ← BlockGrid
  modified: README.md              │   modified: README.md   │
                                   └─────────────────────────┘
$ ls                               ┌─ Block 2 ──────────────┐
file1.txt  file2.txt               │ $ ls                    │
                                   │ file1.txt  file2.txt    │
                                   └─────────────────────────┘

Each Block = one command + its output
→ copy, share, re-run, AI-explain per block
```

---

## Folder Map

```
terminal/
├── model/                   DATA — terminal state
│   ├── terminal_model.rs    TerminalModel (the main state machine)
│   ├── block.rs             Block (command + output)
│   ├── blocks.rs            Block list management
│   ├── blockgrid.rs         BlockGrid (cells within a block)
│   ├── header_grid.rs       HeaderGrid (command input area)
│   ├── alt_screen.rs        AltScreen (vim, less, htop)
│   ├── session.rs           Session state (shell connection)
│   ├── selection.rs         Text selection
│   ├── find.rs              Search in output
│   ├── secrets.rs           Detect & mask secrets
│   └── ansi/                ANSI handler (block-aware)
│
├── view/view.rs             UI — TerminalView
├── block_list_element.rs    UI — renders block list
├── blockgrid_element.rs     UI — renders one block's grid
├── grid_renderer.rs         UI — cells → glyphs + colors
│
├── local_tty/               PTY creation & shell starters
├── input/input.rs           Keystroke → bytes → PTY
├── history/                 Command history
├── ssh/                     SSH connections
├── shared_session/          Session sharing
├── wsl/                     Windows Subsystem for Linux
└── bootstrap.rs             Inject Warp config into shell
```

---

## Core Types

```
TerminalModel          THE state machine — owns everything
  ├── blocks: Blocks       list of all Blocks
  ├── session: Session     shell connection, type, version
  ├── alt_screen           fullscreen apps (vim/less)
  ├── selection            text selection state
  └── terminal_mode        cursor/mouse/paste mode flags

Block                  One command + its output
  ├── header_grid          the command line ("$ git status")
  ├── block_grid           the output (cells with colors)
  ├── block_id             unique identifier
  └── status               running / finished / failed

BlockGrid              Grid of cells within a block (extends warp_terminal grid)
HeaderGrid             Grid for the command input area
AltScreen              Fullscreen buffer (when vim/less takes over)
```

---

## How a Keystroke Becomes Output

```
User presses 'g'
  │
  ├── input/input.rs       convert keystroke to bytes
  ├── local_tty/           write bytes to PTY
  │
  ▼ shell receives 'g', processes it, echoes back with ANSI escapes
  │
  ├── PTY output arrives
  ├── model/ansi/          parse escape sequences (block-aware)
  ├── blockgrid.rs         update cells in current block
  ├── terminal_model       notify() → views re-render
  │
  ├── block_list_element   builds Element tree for all blocks
  ├── grid_renderer        cells → glyphs + rect colors
  ├── Scene                draw commands
  └── GPU                  pixels on screen
```

---

## Block Lifecycle

```
1. Shell shows prompt   → new Block created (HeaderGrid ready)
2. User types command   → HeaderGrid updated via input reporting
3. User presses Enter   → command sent to shell
4. Shell outputs text   → BlockGrid fills with output cells
5. Shell shows prompt   → Block marked complete, new Block starts
```

How Warp knows where blocks start/end: **shell integration protocol**.
Warp injects hooks into the shell that send special escape sequences
at prompt-start, command-start, and command-end.

---

## AltScreen — Fullscreen Apps

When you run `vim`, `less`, `htop`, etc., they take over the entire screen:

```
Normal mode:               Alt screen mode:
┌─ Block 1 ───────┐       ┌──────────────────────┐
│ $ git log        │       │ vim ~/.bashrc         │
│ commit abc123    │  →    │ # Bash config         │
│ commit def456    │       │ export PATH=...       │
└──────────────────┘       │ ~                     │
┌─ Block 2 ───────┐       │ ~                     │
│ $ vim ~/.bashrc  │       └──────────────────────┘
└──────────────────┘       (blocks hidden, alt screen shown)
```

When the app exits, alt screen is discarded and blocks reappear.

---

## warp_terminal vs app/src/terminal/

```
warp_terminal (crate)            app/src/terminal/ (app layer)
─────────────────────            ──────────────────────────────
Raw cell grid                    Block-based model
ANSI parsing                     Block-aware ANSI handling
Terminal modes                   Shell integration protocol
No UI                            Views, Elements, rendering
Like xterm's core                What makes it "Warp"
```

---

## Entry Point — How the App Creates a Terminal

```
App starts
  → RootView
    → Workspace
      → PaneGroup
        → Pane (a tab)
          → PaneView<TerminalView>    ← wraps TerminalView in a tab
            → TerminalView            ← THE terminal UI

Key files:
  app/src/pane_group/pane/terminal_pane.rs  → TerminalPaneView = PaneView<TerminalView>
  app/src/pane_group/pane/mod.rs            → renders ChildView::<PaneView<TerminalView>>
  app/src/workspace/action.rs               → WorkspaceAction::AddTerminalTab
```

When user clicks "New Tab":
```
WorkspaceAction::AddTerminalTab
  → PaneGroup creates new Pane
  → Pane creates TerminalPaneView
  → TerminalPaneView creates TerminalView
  → TerminalView creates TerminalModel + starts local_tty event loop
  → terminal is running
```

---

## UI Rendering — The Element Tree

```
TerminalView.render()  (view.rs — 25000+ lines!)
│
├── Flex::column()
│   │
│   ├── OUTPUT AREA (one of these):
│   │   │
│   │   ├── BlockListElement              ← normal mode
│   │   │   ├── Block 1
│   │   │   │   ├── HeaderGrid element    ← "$ git status"
│   │   │   │   ├── BlockGridElement      ← output cells
│   │   │   │   │   └── grid_renderer     ← cells → glyphs + rects
│   │   │   │   └── status bar            ← timing, exit code
│   │   │   ├── Block 2 ...
│   │   │   └── Block N ...
│   │   │
│   │   ├── AltScreenElement              ← fullscreen (vim/less)
│   │   │   └── full grid + selections
│   │   │
│   │   └── viewer_loading                ← shared session loading
│   │
│   └── INPUT AREA:
│       └── ChildView(input)              ← command input box
│           └── EditorView                ← reuses the editor system
│
└── Stack overlays:
    ├── tooltips, context menus
    ├── agent progress, banners
    └── onboarding callouts
```

### How cells become pixels

```
BlockGridElement.paint()
  for each row in grid:
    for each cell in row:
      background → scene.draw_rect(bounds, bg_color)
      character  → scene.draw_glyph(position, glyph_id, fg_color)
  cursor       → scene.draw_rect(cursor_bounds, cursor_color)
  selections   → scene.draw_rect(selection_bounds, highlight_color)
```

### Three input modes

```
PinnedToBottom:           PinnedToTop:              Waterfall:
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ blocks...     │         │ [input]       │         │ blocks above  │
│               │         │ blocks...     │         │ [input]       │
│ [input]       │         │               │         │ blocks below  │
└───────────────┘         └───────────────┘         └───────────────┘
```

### Key rendering files

```
view.rs                    TerminalView.render() — orchestrates everything
block_list_element.rs      scrollable list of blocks
blockgrid_element.rs       one block's grid → Element
blockgrid_renderer.rs      grid cells → Scene draw calls
grid_renderer.rs           individual cells → glyphs + background rects
alt_screen_element.rs      fullscreen apps (vim/less)
terminal_size_element.rs   detect resize → update PTY
waterfall_gap_element.rs   gap in waterfall mode
```

---

## How to Navigate This Folder (50+ files)

Every file serves one of three concerns:

```
DATA IN (shell → Warp)         STATE (the model)              DATA OUT (Warp → screen)
──────────────────────         ─────────────────              ───────────────────────
local_tty/    PTY pipe         terminal_model.rs  THE state   view/view.rs       View
  shell.rs    shell starters   block.rs           one block   block_list_element  blocks
  server/     read/write loop  blocks.rs          block list  blockgrid_element   grid
input/        keys → PTY       blockgrid.rs       cells       grid_renderer       glyphs
bootstrap.rs  inject config    header_grid.rs     cmd area    color.rs            colors
ssh/          remote PTY       alt_screen.rs      vim/less    links.rs            URLs
wsl/          WSL PTY          session.rs         connection  prompt/             prompt
remote_tty/   SSH PTY          selection.rs       select
model/ansi/   parse escapes    find.rs            search
                               history/           cmd history
                               secrets.rs         mask secrets
```

---

## Read These 5 Files First

```
1. terminal_model.rs   → the struct, its fields
                         "what state does the terminal hold?"

2. block.rs            → Block struct
                         "what is a block?"

3. blockgrid.rs        → BlockGrid
                         "how are cells stored in a block?"

4. view/view.rs        → TerminalView
                         "how does render() build the element tree?"

5. local_tty/mod.rs    → PTY lifecycle
                         "how does Warp talk to the shell?"
```

Everything else is features built on top:

```
Core (read these)              Features (reference later)
─────────────────              ─────────────────────────
terminal_model.rs              selection.rs, find.rs
block.rs, blockgrid.rs         secrets.rs, links.rs
view.rs                        history/, prompt/
local_tty/                     ssh/, wsl/, shared_session/
                               alt_screen.rs, bootstrap.rs
                               cli_agent.rs, warpify/
                               settings.rs, keys.rs
```

---

## The One Mental Model

```
┌─────────────────────────────────────────────────────────┐
│                    TerminalModel                         │
│                                                          │
│  blocks: [Block, Block, Block, ...]                     │
│            │                                             │
│            ├── Block                                     │
│            │     ├── HeaderGrid (command: "git status")  │
│            │     ├── BlockGrid (output cells)            │
│            │     └── status: Running/Complete/Failed     │
│            │                                             │
│  session: Session (shell type, version, PTY handle)     │
│  alt_screen: Option<AltScreen>                          │
│  selection: Option<Selection>                           │
│  mode: TermMode (cursor, mouse, paste flags)            │
│                                                          │
│  IN: PTY → ansi parser → updates blocks                 │
│  OUT: view reads blocks → Elements → Scene → GPU        │
└─────────────────────────────────────────────────────────┘
```

---

## TTY Connections — Three Ways to Talk to a Shell

### local_tty/ — Local shell (the main path)

```
Warp ←─PTY pipe─→ Shell on your machine (bash, zsh, fish, pwsh)

local_tty/
├── mod.rs         PTY creation & lifecycle
├── shell.rs       Shell starters (bash, zsh, fish, pwsh, WSL)
├── event_loop.rs  Background thread: read PTY → parse ANSI → update model
├── server/        Clean helper process to spawn shells (Unix only)
│   ├── mod.rs       fork server at startup (before GPU/network loaded)
│   ├── client.rs    Warp side: request "spawn bash please"
│   ├── event_loop.rs Server side: receive requests, create PTYs
│   ├── api.rs       Request/response types
│   └── protocol.rs  Serialization over Unix socket
└── spawner/       PTY creation (platform-specific)
```

**event_loop.rs** runs on a background thread per terminal tab:
```
loop {
    poll.wait()                         // sleep until data arrives (mio)
    bytes = pty.read()                  // read shell output
    terminal_model.lock()               // lock shared state
    parser.parse_bytes(terminal, bytes) // ANSI → update grid cells
    terminal_model.unlock()             // UI thread can now re-render

    if channel has message:
      Input(bytes) → pty.write(bytes)   // send keystrokes to shell
      Resize(size) → pty.resize(size)   // resize PTY
      Shutdown     → break
}
```

**server/** exists because fork() leaks parent state to child:
```
Problem: Warp has GPU handles, sockets, threads...
         fork+exec shell → shell inherits all that junk

Solution: Fork a clean process VERY early (before GPU/network)
          When user opens tab → ask clean process to spawn shell
          → shell gets clean environment
```

### remote_tty/ — Remote shell via WebSocket

```
Warp ←─WebSocket─→ SSH Proxy Server ←─PTY─→ Shell on remote machine
```

Same architecture, different transport:
```
local_tty                        remote_tty
  PTY file descriptor              WebSocket connection
  mio poll thread                  async WebSocket stream
  pty.read()/write()               sink.send()/stream.recv()
  raw bytes                        binary messages = data
                                   text messages = control (resize)

Both → parser.parse_bytes() → update same TerminalModel
```

```
Startup:
  1. WebSocket connect to ws://server/create?rows=24&cols=80
  2. Send env vars + shell bootstrap script
  3. Split into:
     - stream listener: server → Warp (shell output)
     - writer task: Warp → server (keystrokes + resize)
```

### wsl/ — Windows Subsystem for Linux

```
Warp on Windows ←─PTY─→ WSL shell (bash/zsh inside Linux on Windows)
```

Special shell starter that launches through WSL instead of native Windows shells.

### ssh/ — SSH connections

```
Warp ←─→ remote server via SSH protocol
```

SSH session management, key auth, connection lifecycle.

### shared_session/ — Session sharing

```
Warp instance A ←─→ shared session ←─→ Warp instance B
```

Multiple Warp windows/users can share the same terminal session.

---

## How They All Connect

```
                                        Shell command           Platform
                                        ─────────────           ────────
                    ┌── local_tty ────── bash/zsh/fish          macOS/Linux (Unix PTY)
                    │   (same code)
TerminalModel ◄─────┼── local_tty ────── powershell.exe         Windows (ConPTY)
 (same model)      │   (same code)
                    ├── local_tty ────── wsl.exe -d Ubuntu bash Windows (ConPTY → WSL)
                    │   (same code,       └→ WslShellStarter just changes the command
                    │    same event_loop)
                    │
                    └── remote_tty ──── WebSocket ── SSH proxy ── remote shell

wsl/ folder only detects available WSL distributions from Windows registry.
WSL shells use the SAME local_tty event loop — just a different ShellStarter.

All paths end at the same place:
  parser.parse_bytes(terminal_model, bytes)
  → update cells → notify → render
```
