# Codex CLI Main Loop Architecture Analysis

## Overview

Codex CLI is an AI coding assistant that interacts with remote LLM inference servers (like OpenAI's Responses API) and executes tools locally. This document provides a top-down overview of how Codex's main loop works, focusing on:

1. **Remote inference interaction** - How Codex communicates with LLM servers
2. **Local tool execution** - How tools are executed locally with sandboxing
3. **Information flow** - How data flows between the model, tools, and UI

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI Entry Point                         │
│                    (codex-rs/cli/src/main.rs)                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      TUI Layer (codex-tui)                       │
│  - User interface (ratatui)                                      │
│  - Handles user input                                           │
│  - Displays events from Codex core                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Codex Core (codex-core)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Codex::spawn()                              │   │
│  │  - Creates Session with channels                         │   │
│  │  - Spawns submission_loop() task                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         Submission Channel (tx_sub / rx_sub)              │   │
│  │  Operations: UserTurn, UserInput, Interrupt, etc.        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │          Event Channel (tx_event / rx_event)              │   │
│  │  Events: TokenCount, TurnDiff, Error, etc.                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Session & Turn Management                     │
│  - submission_loop() processes operations                       │
│  - spawn_task() creates tasks (RegularTask, ReviewTask, etc.)   │
│  - run_turn() executes a single conversation turn                │
└─────────────────────────────────────────────────────────────────┘
```

## Main Loop Flow

### 1. Initialization

```
User runs: codex [prompt]
    │
    ▼
CLI main.rs
    │
    ▼
TUI::run_main()
    │
    ▼
Codex::spawn(config, auth, history, session_source)
    │
    ├─► Creates channels:
    │   - tx_sub / rx_sub (Submission channel, capacity 64)
    │   - tx_event / rx_event (Event channel, unbounded)
    │
    ├─► Creates Session with:
    │   - conversation_id (UUID)
    │   - SessionServices (MCP, exec, etc.)
    │   - ContextManager (for conversation history)
    │
    └─► Spawns submission_loop() task
```

### 2. Submission Loop

The `submission_loop()` is the central dispatcher that processes operations:

```rust
async fn submission_loop(sess: Arc<Session>, config: Arc<Config>, rx_sub: Receiver<Submission>) {
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::UserTurn { items, ... } => {
                // Create new turn context
                // Spawn RegularTask to handle the turn
            }
            Op::UserInput { items } => {
                // Try to inject into current task
                // Or spawn new RegularTask
            }
            Op::Interrupt => {
                // Cancel current task
            }
            Op::ExecApproval { id, decision } => {
                // Notify waiting tool call
            }
            // ... other operations
        }
    }
}
```

### 3. Turn Execution Flow

When a `UserTurn` or `UserInput` operation is received, a task is spawned that calls `run_turn()`:

```
┌─────────────────────────────────────────────────────────────┐
│                    run_turn() Flow                          │
└─────────────────────────────────────────────────────────────┘

1. Build Prompt
   ├─► Input items (conversation history + user input)
   ├─► Tool specs (from ToolRouter)
   └─► Parallel tool calls flag (based on model support)

2. Stream from Model
   ├─► ModelClient::stream(prompt)
   │   ├─► Build request payload (instructions, input, tools)
   │   ├─► POST to Responses API (or Chat Completions API)
   │   └─► Parse SSE stream (Server-Sent Events)
   │
   └─► Process ResponseEvent stream:
       ├─► ResponseEvent::Created
       ├─► ResponseEvent::OutputTextDelta (streaming text)
       ├─► ResponseEvent::OutputItemDone (tool call or message)
       ├─► ResponseEvent::OutputItemAdded (item started)
       └─► ResponseEvent::Completed (turn finished)

3. Process Each ResponseItem
   ├─► If tool call (FunctionCall, LocalShellCall, CustomToolCall):
   │   ├─► ToolRouter::build_tool_call() → ToolCall
   │   ├─► ToolCallRuntime::handle_tool_call()
   │   │   ├─► ToolOrchestrator::run()
   │   │   │   ├─► Check approval policy
   │   │   │   ├─► Select sandbox (None, Seatbelt, Landlock, etc.)
   │   │   │   ├─► Execute tool in sandbox
   │   │   │   └─► If denied, retry without sandbox (if allowed)
   │   │   │
   │   │   └─► ToolRegistry::dispatch() → ToolRuntime
   │   │       ├─► Function tools (read_file, write_file, etc.)
   │   │       ├─► LocalShell (shell command execution)
   │   │       ├─► UnifiedExec (unified execution runtime)
   │   │       ├─► MCP tools (Model Context Protocol)
   │   │       └─► Custom tools
   │   │
   │   └─► Return ResponseInputItem (tool output)
   │
   └─► If message (assistant text):
       └─► Emit events for UI display

4. Collect Results
   ├─► Wait for all tool calls to complete
   ├─► Process items (via process_items())
   │   ├─► Record in conversation history
   │   └─► Build ResponseInputItems for next turn
   └─► Return TurnRunResult
```

## Remote Inference Interaction

### ModelClient Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    ModelClient::stream()                      │
└─────────────────────────────────────────────────────────────┘

1. Build Request Payload
   ├─► Instructions (base + user + developer instructions)
   ├─► Input (conversation history as ResponseItems)
   ├─► Tools (JSON schema for available tools)
   ├─► Tool choice ("auto" or specific tool)
   ├─► Parallel tool calls (enabled if model supports)
   ├─► Reasoning params (effort, summary config)
   └─► Output schema (if specified)

2. HTTP Request
   ├─► Provider: OpenAI Responses API, Chat Completions, or OSS
   ├─► Authentication: API key or OAuth token
   ├─► Headers:
   │   ├─► conversation_id / session_id
   │   ├─► chatgpt-account-id (if ChatGPT auth)
   │   └─► x-openai-subagent (if subagent session)
   └─► POST with JSON payload

3. SSE Stream Processing
   ├─► Parse Server-Sent Events
   │   ├─► event: response.created
   │   ├─► event: response.output_text.delta (streaming text)
   │   ├─► event: response.output_item.done (tool call ready)
   │   ├─► event: response.output_item.added (item started)
   │   ├─► event: response.reasoning_text.delta (reasoning stream)
   │   └─► event: response.completed (turn finished)
   │
   └─► Convert to ResponseEvent enum
       └─► Forward to turn processing loop
```

### Request/Response Flow

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Codex CLI  │─────────►│  ModelClient │────────►│  LLM Server  │
│              │         │              │         │  (OpenAI/etc)│
└──────────────┘         └──────────────┘         └──────────────┘
      │                         │                          │
      │  1. Build Prompt        │                          │
      │     - History           │                          │
      │     - Tools             │                          │
      │     - Instructions      │                          │
      │                         │                          │
      │  2. stream(prompt)     │                          │
      │────────────────────────►│                          │
      │                         │  3. POST /responses     │
      │                         │─────────────────────────►│
      │                         │                          │
      │                         │  4. SSE Stream          │
      │                         │◄─────────────────────────│
      │  5. ResponseEvents      │                          │
      │◄────────────────────────│                          │
      │                         │                          │
      │  6. Process items      │                          │
      │     - Tool calls        │                          │
      │     - Messages          │                          │
      │                         │                          │
      │  7. Execute tools       │                          │
      │     locally             │                          │
      │                         │                          │
      │  8. Build next turn     │                          │
      │     input with tool     │                          │
      │     outputs             │                          │
      │                         │                          │
      │  9. stream(prompt)     │                          │
      │     (with tool outputs) │                          │
      │────────────────────────►│                          │
      │                         │                          │
      └─────────────────────────┴──────────────────────────┘
```

## Local Tool Execution

### Tool Execution Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│              Tool Execution with Sandboxing                   │
└─────────────────────────────────────────────────────────────┘

1. Tool Call Detection
   ├─► ResponseItem::FunctionCall { name, arguments, call_id }
   ├─► ResponseItem::LocalShellCall { action, call_id }
   ├─► ResponseItem::CustomToolCall { name, input, call_id }
   └─► ToolRouter::build_tool_call() → ToolCall

2. Tool Routing
   ├─► ToolRegistry::dispatch(ToolInvocation)
   │   ├─► Function tools (read_file, write_file, search, etc.)
   │   ├─► LocalShell (shell command execution)
   │   ├─► UnifiedExec (unified execution runtime)
   │   ├─► MCP tools (via MCP connection manager)
   │   └─► Custom tools
   │
   └─► Returns ToolRuntime implementation

3. Tool Orchestration (ToolOrchestrator::run())
   ├─► Approval Check
   │   ├─► If approval_policy == AskForApproval::OnRequest
   │   │   ├─► Assess command risk (sandbox assessment)
   │   │   ├─► Send ExecApprovalRequestEvent to UI
   │   │   └─► Wait for user decision
   │   │
   │   └─► If denied → return ToolError::Rejected
   │
   ├─► Sandbox Selection
   │   ├─► Based on sandbox_policy:
   │   │   ├─► SandboxPolicy::Never → SandboxType::None
   │   │   ├─► SandboxPolicy::WorkspaceWrite → Platform sandbox
   │   │   └─► SandboxPolicy::DangerFullAccess → SandboxType::None
   │   │
   │   └─► Platform-specific:
   │       ├─► macOS → Seatbelt
   │       ├─► Linux → Landlock + seccomp
   │       └─► Windows → Restricted token
   │
   ├─► First Attempt (in sandbox)
   │   ├─► ToolRuntime::run(request, sandbox_attempt)
   │   ├─► If success → return output
   │   └─► If sandbox denied → escalate
   │
   └─► Escalation (if sandbox denied and allowed)
       ├─► Ask for approval to retry without sandbox
       ├─► If approved → retry with SandboxType::None
       └─► Return result
```

### Tool Types

1. **Function Tools** (`codex-core/src/tools/handlers/`)
   - `read_file` - Read file contents
   - `write_file` - Write file contents
   - `search_replace` - Search and replace in files
   - `apply_patch` - Apply git patches
   - `search` - Search codebase
   - `web_search` - Web search (if enabled)
   - And more...

2. **LocalShell** (`codex-core/src/shell.rs`)
   - Executes shell commands
   - Supports timeout, working directory
   - Can request escalated permissions

3. **UnifiedExec** (`codex-core/src/unified_exec/`)
   - Unified execution runtime
   - Handles complex execution scenarios
   - Manages execution sessions

4. **MCP Tools** (`codex-core/src/mcp/`)
   - Model Context Protocol tools
   - Connects to external MCP servers
   - Dynamic tool discovery

## Information Flow

### Conversation History Management

```
┌─────────────────────────────────────────────────────────────┐
│              Conversation History Flow                      │
└─────────────────────────────────────────────────────────────┘

1. User Input
   └─► Op::UserTurn { items: [ResponseItem::Message { role: "user", ... }] }

2. Model Response
   ├─► ResponseItem::Message { role: "assistant", ... }
   ├─► ResponseItem::FunctionCall { ... }
   └─► ResponseItem::FunctionCallOutput { ... }

3. Recording
   ├─► Session::record_conversation_items()
   ├─► ContextManager::append_items()
   └─► Stored in memory + persisted to disk

4. Next Turn
   ├─► ContextManager::build_prompt()
   ├─► Includes full conversation history
   ├─► May compact if token limit exceeded
   └─► Sent as "input" array to model
```

### Event Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Event System                              │
└─────────────────────────────────────────────────────────────┘

Session (Core)
    │
    ├─► send_event() / send_event_raw()
    │   └─► tx_event.send(Event { id, msg })
    │
    └─► Event types:
        ├─► TokenCountEvent (token usage)
        ├─► TurnDiffEvent (file changes)
        ├─► ExecApprovalRequestEvent (ask user)
        ├─► ErrorEvent (errors)
        ├─► AgentMessageContentDeltaEvent (streaming text)
        └─► ... many more

TUI (Frontend)
    │
    ├─► Receives events via rx_event
    │
    └─► Updates UI:
        ├─► Display streaming text
        ├─► Show approval dialogs
        ├─► Display errors
        └─► Update status
```

## Key Components

### Session (`codex-core/src/codex.rs`)

- **Purpose**: Manages a conversation session
- **Key responsibilities**:
  - Turn context management
  - Conversation history (via ContextManager)
  - Event emission
  - Task spawning and cancellation
  - MCP connection management

### TurnContext (`codex-core/src/codex.rs`)

- **Purpose**: Context for a single turn
- **Contains**:
  - ModelClient (for API calls)
  - Working directory
  - Approval/sandbox policies
  - Tool configuration
  - Instructions (base, user, developer)

### ModelClient (`codex-core/src/client.rs`)

- **Purpose**: Handles communication with LLM servers
- **Key methods**:
  - `stream(prompt)` - Streams responses from model
  - Supports Responses API and Chat Completions API
  - Handles authentication, retries, rate limiting

### ToolRouter (`codex-core/src/tools/router.rs`)

- **Purpose**: Routes tool calls to appropriate handlers
- **Key methods**:
  - `build_tool_call()` - Converts ResponseItem to ToolCall
  - `dispatch_tool_call()` - Executes tool call

### ToolOrchestrator (`codex-core/src/tools/orchestrator.rs`)

- **Purpose**: Coordinates tool execution with approvals and sandboxing
- **Key responsibilities**:
  - Approval workflow
  - Sandbox selection
  - Retry logic (sandbox → no sandbox)

### ToolRegistry (`codex-core/src/tools/registry.rs`)

- **Purpose**: Registry of available tools
- **Key methods**:
  - `dispatch()` - Routes to specific ToolRuntime

## Turn Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Turn Lifecycle                           │
└─────────────────────────────────────────────────────────────┘

1. User submits input
   └─► Op::UserTurn submitted to submission_loop

2. Create TurnContext
   ├─► New sub_id (submission ID)
   ├─► ModelClient with current config
   └─► Copy session settings (cwd, policies, etc.)

3. Spawn Task
   ├─► RegularTask::run()
   └─► Calls run_turn()

4. Execute Turn
   ├─► Build Prompt (history + input + tools)
   ├─► Stream from model
   ├─► Process ResponseEvents
   ├─► Execute tool calls (if any)
   ├─► Collect tool outputs
   └─► Process items

5. Record Results
   ├─► Record in conversation history
   ├─► Emit events (TurnDiff, TokenCount, etc.)
   └─► Return TurnRunResult

6. Next Turn (if tool outputs)
   └─► Tool outputs become input for next turn
       └─► Loop back to step 3
```

## Error Handling & Retries

### Stream Retries

- **Model stream disconnections**: Retry with exponential backoff
- **Max retries**: Configurable per provider (default: 3)
- **Fatal errors**: Context window exceeded, quota exceeded → no retry

### Tool Execution Errors

- **Sandbox denial**: Can retry without sandbox (if approved)
- **Tool failures**: Return error as tool output to model
- **Fatal errors**: Abort turn

## Sandboxing

### Sandbox Types

1. **None**: No sandboxing (full system access)
2. **Seatbelt** (macOS): Apple's sandbox framework
3. **Landlock + seccomp** (Linux): Linux security modules
4. **Restricted Token** (Windows): Windows security token

### Sandbox Policies

- **Never**: No sandboxing
- **WorkspaceWrite**: Can write to workspace, no network
- **DangerFullAccess**: No restrictions (dangerous)

## Summary

The Codex CLI main loop follows this pattern:

1. **Submission Loop**: Processes operations from UI (UserTurn, UserInput, etc.)
2. **Turn Execution**: For each turn:
   - Build prompt with history and tools
   - Stream from remote LLM server (SSE)
   - Process streaming events
   - Execute tool calls locally (with sandboxing/approvals)
   - Collect tool outputs
   - Record in conversation history
3. **Iteration**: Tool outputs become input for next turn, repeat

The architecture is designed for:
- **Real-time streaming**: Text and tool calls stream as they're generated
- **Safety**: Sandboxing and approval workflows for tool execution
- **Flexibility**: Supports multiple LLM providers and tool types
- **Extensibility**: MCP protocol for external tools
