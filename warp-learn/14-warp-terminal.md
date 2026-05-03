# warp_terminal — The Terminal Emulation Engine

> Source: `crates/warp_terminal/src/`
> The low-level terminal emulator. Parses ANSI escapes, maintains the character grid.
> Knows nothing about UI — the app layer reads this data and renders it.

---

## Three Modules

```
warp_terminal/src/
├── model/           The terminal state (grid, cells, modes, escapes)
├── shell/           Shell detection and integration protocol
└── shared_session/  Session sharing between windows
```

---

## model/ — The Character Grid

### What the terminal IS: a grid of cells

```
Terminal = a 2D grid of Cells, e.g. 80 columns × 24 rows

Row 0: [~][/][p][r][o][j][e][c][t][ ][ ][ ]...
Row 1: [$][ ][g][i][t][ ][s][t][a][t][u][s]...
Row 2: [O][n][ ][b][r][a][n][c][h][ ][m][a]...
         ↑
       each [] = one Cell
```

### Cell — one character position

```rust
Cell {
    content: CharOrStr,    // 'A', '😀', or grapheme cluster
    fg: Color,             // foreground color (256 colors + RGB)
    bg: Color,             // background color
    flags: Flags,          // bitflags: bold, italic, underline, wide, etc.
}

Flags: BOLD | ITALIC | UNDERLINE | STRIKEOUT | DIM | HIDDEN
       | INVERSE | WRAPLINE | WIDE_CHAR | WIDE_CHAR_SPACER
```

### Grid → Row → Cell hierarchy

```
grid/
├── cell.rs           Cell struct + Flags bitflags
├── row.rs            Row = Vec<Cell> with line metadata
├── dimensions.rs     Dimensions trait (width × height)
├── flat_storage/     Efficient scrollback storage (chunked)
└── cell_type.rs      Cell content type classifications
```

---

## model/ansi/ — Colors & Escape Parsing

```
ansi/
├── mod.rs                          Color enum (Named, Indexed, RGB)
└── control_sequence_parameters.rs  Parse CSI parameters (e.g. "31;1" → [31, 1])
```

16 named colors + 256 indexed + 24-bit RGB:
```
NamedColor::Red    → \e[31m
Color::Indexed(82) → \e[38;5;82m
Color::Rgb(r,g,b)  → \e[38;2;r;g;bm
```

---

## model/escape_sequences — The ANSI Protocol

This is how shells control the terminal — by sending special byte sequences:

```
Category          Example              Meaning
────────          ───────              ───────
Cursor movement   \e[H                 move to top-left
                  \e[10;5H             move to row 10, col 5
                  \e[A                 move up one line

Text styling      \e[1m                bold on
                  \e[31m               red foreground
                  \e[0m                reset all styles

Screen control    \e[2J                clear entire screen
                  \e[K                 clear to end of line

Terminal modes    \e[?25l              hide cursor
                  \e[?25h              show cursor
                  \e[?1049h            switch to alt screen (vim)
                  \e[?2004h            enable bracketed paste

Kitty keyboard    \e[>1u               push keyboard mode
                  \e[<u                pop keyboard mode
```

### C0 control characters (single bytes)

```
BEL (0x07)  → bell/beep
BS  (0x08)  → backspace
HT  (0x09)  → tab
LF  (0x0A)  → line feed (newline)
CR  (0x0D)  → carriage return
ESC (0x1B)  → start of escape sequence
```

---

## model/mode.rs — Terminal Mode Flags

Bitflags tracking what the terminal is currently doing:

```
TermMode {
    SHOW_CURSOR          cursor visible?
    APP_CURSOR           application cursor keys mode
    BRACKETED_PASTE      wrap paste in escape sequences
    MOUSE_REPORT_CLICK   report mouse clicks to app
    MOUSE_MOTION         report mouse movement
    LINE_WRAP            wrap at end of line
    FOCUS_IN_OUT         report focus changes
    KEYBOARD_*           Kitty keyboard protocol flags
}
```

---

## shell/ — Shell Detection & Integration

```
shell/
└── mod.rs    ShellType enum + shell config

ShellType: Zsh | Bash | Fish | PowerShell
```

What it tracks per shell:
```
shell_type           which shell
version              shell version (for feature gating)
options              shell options (e.g. histignorespace)
input_reporting_sequence()  → trigger to read shell's input buffer
should_add_command_to_history()
has_autocd()
```

### Input reporting (how Warp reads what you're typing)

```
Warp sends ESC-i → shell's bound function runs
→ shell reads its input buffer
→ sends it back wrapped in DCS escape
→ Warp knows current input for autocomplete
```

---

## How Data Flows

```
Shell process
  │ outputs bytes: "Hello\e[31m World\e[0m\n"
  ▼
PTY (pseudo-terminal pipe)
  │
  ▼
warp_terminal ANSI parser
  │ parses escape sequences
  │ updates grid cells
  ▼
model/grid (Cell grid with colors + flags)
  │
  ▼
app/src/terminal/ reads the grid
  │ builds Elements (BlockListElement)
  │ paint() → Scene
  ▼
GPU renders → pixels on screen
```

---

## Other Files

```
block_id.rs      Unique BlockId for each command block
block_index.rs   Index into the block list
indexing.rs      Line/Column position types (type-safe)
char_or_str.rs   Efficient storage: single char or multi-char grapheme
mouse.rs         Mouse tracking modes and button encoding
```

---

## Key Insight: This Crate vs App Layer

```
warp_terminal (this crate)           app/src/terminal/
─────────────────────────           ──────────────────
Low-level terminal emulator          High-level terminal features
Grid of cells + ANSI parsing         Block-based model (command+output)
No UI, no Views                      Views, Elements, user interaction
Like xterm's core                    Like Warp's special sauce
```

`warp_terminal` is what makes it a terminal.
`app/src/terminal/` is what makes it Warp.
