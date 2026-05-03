# AI in the Terminal — How Agent Mode Works

> AI conversations render as rich content blocks inside the same block list
> as terminal commands. The AI can also run real shell commands via tool calls.

---

## AI Blocks Live in the Block List

```
Block List (what you see on screen):
┌─ Block 1 (terminal) ────────────┐
│ $ git status                     │  ← regular terminal block
│ On branch main                   │
└──────────────────────────────────┘
┌─ AIBlock (rich content) ─────────┐
│ 🤖 "Fix the build error"         │  ← AI conversation
│ I'll edit src/main.rs...          │     (ViewHandle<AIBlock>)
│ [Apply] [Reject]                  │
└──────────────────────────────────┘
┌─ Block 2 (terminal) ────────────┐
│ $ cargo build                    │  ← agent-requested command
│ Compiling...                     │     (agent told the shell to run this)
└──────────────────────────────────┘
┌─ AIBlock (rich content) ─────────┐
│ 🤖 Build succeeded!              │  ← agent's response after command
└──────────────────────────────────┘

Terminal blocks and AI blocks share the SAME block list.
```

---

## AI Models Inside TerminalView

```rust
TerminalView {
    // Terminal:
    model: TerminalModel,                   // blocks, grid, PTY
    input: ViewHandle<Input>,               // command input (shared)

    // AI:
    ai_controller: BlocklistAIController,   // orchestrates conversations
    ai_input_model: BlocklistAIInputModel,  // terminal vs AI input mode
    ai_context_model: BlocklistAIContextModel, // blocks attached as context
    ai_action_model: BlocklistAIActionModel,   // tool calls (edit, run cmd)
    agent_view_controller: AgentViewController, // agent view state

    rich_content_views: Vec<RichContent>,   // AIBlocks + banners + onboarding
    //  ↑ inserted into block list alongside terminal blocks
}
```

---

## How a Conversation Flows

```
1. User types "fix the build error" in Input (AI mode)
     → ai_controller sends request to server

2. Server streams response chunks
     → TaskCallback fires on main thread
     → ai_controller creates AIBlock
     → AIBlock inserted into rich_content_views
     → inserted into block list sumtree
     → notify() → render() → visible

3. Agent decides to run a command
     → ai_action_model receives tool call
     → ShellCommandExecutor writes to PTY
     → real terminal block created (command runs in shell)

4. Command completes
     → result sent back to server
     → server streams next response
     → new AIBlock inserted → render()
```

---

## Input Box Switches Modes

Same input, two modes:

```
Terminal mode:                    AI mode:
┌─────────────────────┐          ┌─────────────────────┐
│ $ git status        │          │ @ fix the build err  │
└─────────────────────┘          └─────────────────────┘
  → writes to PTY                  → sends to AI server

ai_input_model.is_ai_input_enabled() controls which mode.
User toggles with keybinding or typing @/$.
```

---

## Agent Tool Calls → Real Shell Commands

```
AI response: { "tool": "run_command", "command": "cargo build" }
  │
  ▼
ai_action_model → ShellCommandExecutor
  │
  ▼
ShellCommandExecutorEvent::ExecuteCommand
  │
  ▼
TerminalView → Event::ExecuteCommand → writes to PTY
  → shell runs "cargo build" for real
  → terminal block appears in block list
  → when done → result sent back to AI
```

---

## Agent View (Fullscreen AI Mode)

```
Normal terminal:                Agent view (fullscreen):
┌─────────────────────┐        ┌─────────────────────┐
│ Block 1 (terminal)  │        │ AIBlock (query)     │
│ Block 2 (terminal)  │   →    │ AIBlock (response)  │
│ AIBlock             │        │ AIBlock (tool call) │
│ [input]             │        │ [input]             │
└─────────────────────┘        └─────────────────────┘

agent_view_controller.is_active() → hides terminal blocks
Only shows AI blocks for the active conversation.
Enter: keybinding or typing in AI mode
Exit: Escape or back button
```

---

## Block List Rendering (Both Types Together)

```
BlockListElement iterates the sumtree:
  for each item:
    if terminal Block → BlockGridElement (cells → glyphs)
    if RichContent(AIBlock) → ChildView<AIBlock> (markdown, diffs, tools)
    if RichContent(Banner) → inline banner element

All rendered through the same:
  TerminalView.render() → Flex::column → BlockListElement → Scene → GPU
```

---

## Key AI Files

```
AI library crate:
  crates/ai/src/agent/           Agent execution engine
  crates/ai/src/skills/          Skill definitions
  crates/ai/src/project_context/ Workspace context

AI in terminal (app layer):
  app/src/ai/blocklist/
    controller.rs                BlocklistAIController (main orchestrator)
    block.rs                     AIBlock view (renders one exchange)
    agent_view/controller.rs     AgentViewController (fullscreen mode)
    action_model.rs              Tool call execution
    context_model.rs             Context management (attached blocks/text)
    input_model.rs               Terminal vs AI input mode
    inline_action/code_diff_view Code diff viewer

  app/src/ai/agent/
    conversation.rs              AIConversation state
    mod.rs                       AIAgentExchange, AIAgentAction
    api.rs                       Server communication
    driver.rs                    Agent execution driver
```

---

## Complete Code Path: User Query → Server → Pixels

### Step 1: User presses Enter in AI mode

```
app/src/terminal/input.rs → submit_ai_query()
  ├── reads buffer: "fix the build error"
  ├── if AgentView enabled & not active:
  │     emit(EnterAgentView) → enters agent view first
  └── ai_controller.send_user_query_in_new_conversation(query)
```

### Step 2: Controller prepares request

```
app/src/ai/blocklist/controller.rs → send_query()
  ├── start_new_conversation_for_request()
  │     → creates AIConversation in history model
  │     → emits StartedNewConversation
  ├── builds context (attached blocks, files, pwd, git)
  ├── builds AIAgentInput (query + context)
  └── send_request_input(RequestInput { inputs, task_id })
```

### Step 3: Request sent to server + AIBlock created

```
send_request_input():
  ├── creates exchange in history:
  │     history.append_exchange(exchange_id)
  │     → emits AppendedExchange
  │     → TerminalView creates AIBlock view (see Step 6)
  │
  ├── spawns streaming connection:
  │     ctx.spawn_stream_local(response_stream, on_item, on_done)
  │     HTTP SSE to: POST /api/agent/v1/conversation
  │
  └── emits SentRequest → Input clears buffer
```

### Step 4: Server streams response chunks

```
Server sends SSE:
  { "type": "text", "content": "I'll fix " }
  { "type": "text", "content": "the error " }
  { "type": "tool_call", "tool": "edit_file", ... }
  { "type": "finished" }

Each chunk → TaskCallback::ModelFromStream.on_item → main thread
```

### Step 5: Controller processes each chunk

```
handle_response_stream_event():
  ├── text chunk → updates exchange → notify() → AIBlock re-renders
  ├── tool_call → creates AIAgentAction → AIBlock shows [Accept]/[Reject]
  └── finished → emits FinishedReceivingOutput → status bar done
```

### Step 6: AIBlock created and rendered

```
TerminalView.handle_ai_history_model_event(AppendedExchange):
  ├── creates: ctx.add_typed_action_view(AIBlock::new(...))
  ├── inserts into block list sumtree alongside terminal blocks
  └── subscribes to AIBlock events

Rendering (each chunk triggers re-render):
  TerminalView.render() → BlockListElement iterates sumtree:
    ... terminal Block → BlockGridElement (cells → glyphs)
    ... AIBlock → ChildView<AIBlock> (markdown + tool call UI)
    → Scene → GPU → pixels
```

### Step 7: Tool call execution (agent runs a command)

```
User clicks [Accept] on "Run cargo build?":
  → ai_action_model executes action
  → ShellCommandExecutor → Event::ExecuteCommand
  → writes "cargo build\n" to PTY
  → real terminal Block appears in block list

Command completes:
  → ai_controller.resume_conversation(result_context)
  → server gets command output → streams next response
  → back to Step 4 (loop)
```

### Step 8: End-to-end data flow

```
User query → Input → AIController → HTTP SSE → Server
                                                  │
Server streams chunks ←────────────────────────────┘
  │
  ▼
TaskCallback (main thread)
  → AIController processes chunk
  → updates history model
  → notify() → flush_effects()
  → ViewNotification → window dirty
  → request_redraw()
  │
  ▼ (next frame)
RedrawRequested
  → TerminalView.render()
  → BlockListElement: [Block, AIBlock, Block, AIBlock]
  → layout() → paint() → Scene → GPU → pixels
```

---

## Summary

```
AI is NOT a separate panel. It's embedded in the terminal:

1. AIBlocks      = rich content in the SAME block list as terminal blocks
2. Input box     = SAME input, switches between terminal/AI mode
3. Tool calls    = AI runs REAL commands in the REAL shell via PTY
4. Block context = terminal blocks can be ATTACHED as AI context
5. Agent view    = fullscreen mode showing only AI conversation
6. Rendering     = same pipeline: render() → Elements → Scene → GPU
7. Server comm   = HTTP SSE streaming, processed via TaskCallback on main thread
8. Loop          = tool call → command runs → result → server → next response
```
