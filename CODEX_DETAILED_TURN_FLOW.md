# Codex CLI: Detailed Turn Flow and ModelClient Architecture

## Table of Contents
1. [run_turn Flow - Step by Step](#run_turn-flow---step-by-step)
2. [ModelClient Architecture](#modelclient-architecture)
3. [From User Input to Tool Calls](#from-user-input-to-tool-calls)
4. [Tool Execution and Result Formulation](#tool-execution-and-result-formulation)
5. [All Built-in Tools](#all-built-in-tools)
6. [All Built-in Prompts](#all-built-in-prompts)

---

## run_turn Flow - Step by Step

### Overview

`run_turn()` is the core function that executes a single conversation turn. It orchestrates:
- Building the prompt with conversation history and tool specs
- Streaming responses from the model
- Processing tool calls as they arrive
- Collecting tool outputs
- Formulating the next turn's input

### Detailed Flow

```rust
async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    turn_diff_tracker: SharedTurnDiffTracker,
    input: Vec<ResponseItem>,  // Conversation history + user input
    cancellation_token: CancellationToken,
) -> CodexResult<TurnRunResult>
```

#### Step 1: Build ToolRouter

```rust
let mcp_tools = sess.services.mcp_connection_manager.list_all_tools();
let router = Arc::new(ToolRouter::from_config(
    &turn_context.tools_config,
    Some(mcp_tools),
));
```

- Collects all available tools (built-in + MCP tools)
- Creates `ToolRouter` that maps tool names to handlers
- Determines which tools support parallel execution

#### Step 2: Build Prompt

```rust
let prompt = Prompt {
    input,                    // Conversation history as ResponseItems
    tools: router.specs(),    // JSON schemas for all tools
    parallel_tool_calls,      // Based on model support
    base_instructions_override: turn_context.base_instructions.clone(),
    output_schema: turn_context.final_output_json_schema.clone(),
};
```

**Prompt Structure:**
- `input`: Array of `ResponseItem` representing full conversation history
- `tools`: Array of tool specifications (JSON schemas)
- `parallel_tool_calls`: Boolean indicating if model supports parallel calls
- `base_instructions_override`: Optional custom system instructions
- `output_schema`: Optional JSON schema for structured output

#### Step 3: Stream from Model

```rust
let mut stream = turn_context
    .client
    .clone()
    .stream(prompt)
    .or_cancel(&cancellation_token)
    .await??;
```

This calls `ModelClient::stream()` which:
1. Builds the HTTP request payload
2. POSTs to the LLM API (Responses API or Chat Completions)
3. Parses Server-Sent Events (SSE) stream
4. Converts SSE events to `ResponseEvent` enum

#### Step 4: Process Stream Events

The main loop processes events as they arrive:

```rust
loop {
    let event = stream.next().or_cancel(&cancellation_token).await?;
    
    match event {
        ResponseEvent::Created => {
            // Response created, continue
        }
        ResponseEvent::OutputTextDelta(delta) => {
            // Stream text to UI in real-time
            sess.send_event(AgentMessageContentDeltaEvent { ... }).await;
        }
        ResponseEvent::OutputItemAdded(item) => {
            // Item started (e.g., tool call initiated)
            sess.emit_turn_item_started(&turn_item).await;
        }
        ResponseEvent::OutputItemDone(item) => {
            // Item completed - process tool call or message
            if is_tool_call(item) {
                handle_tool_call(item).await;
            } else {
                record_message(item).await;
            }
        }
        ResponseEvent::Completed { token_usage } => {
            // Turn finished - collect all results
            let processed_items = output.try_collect().await?;
            return TurnRunResult { processed_items, token_usage };
        }
    }
}
```

#### Step 5: Handle Tool Calls

When `ResponseEvent::OutputItemDone` contains a tool call:

```rust
match ToolRouter::build_tool_call(sess.as_ref(), item.clone()) {
    Ok(Some(call)) => {
        // Spawn async task to execute tool
        let response = tool_runtime.handle_tool_call(call, cancellation_token).await?;
        
        // Store ProcessedResponseItem with tool output
        output.push_back(ProcessedResponseItem {
            item: original_tool_call_item,
            response: Some(response),  // ResponseInputItem with tool output
        });
    }
    Ok(None) => {
        // Not a tool call (e.g., assistant message)
        record_message(item).await;
    }
}
```

#### Step 6: Collect and Process Results

After `ResponseEvent::Completed`:

```rust
let processed_items = output.try_collect().await?;

// Process items: record in history, build next turn input
let (responses, items_to_record) = process_items(
    processed_items,
    sess,
    turn_context,
).await;
```

`process_items()`:
- Separates items to record in conversation history
- Builds `ResponseInputItem` array for next turn
- Records everything in `ContextManager`

#### Step 7: Return Results

```rust
TurnRunResult {
    processed_items: Vec<ProcessedResponseItem>,
    total_token_usage: Option<TokenUsage>,
}
```

---

## ModelClient Architecture

### Overview

`ModelClient` handles all communication with remote LLM servers. It supports:
- **Responses API** (OpenAI's experimental API)
- **Chat Completions API** (standard OpenAI API)
- **OSS models** (local/self-hosted)

### Request Building

#### Responses API Request Structure

```rust
pub struct ResponsesApiRequest<'a> {
    pub model: &'a str,                    // e.g., "gpt-5-codex"
    pub instructions: &'a str,              // System instructions
    pub input: &'a Vec<ResponseItem>,       // Conversation history
    pub tools: &'a [serde_json::Value],    // Tool specifications
    pub tool_choice: &'static str,         // "auto"
    pub parallel_tool_calls: bool,         // Model capability flag
    pub reasoning: Option<Reasoning>,      // Reasoning effort/summary
    pub store: bool,                        // Cache response
    pub stream: bool,                       // Always true
    pub include: Vec<String>,              // e.g., ["reasoning.encrypted_content"]
    pub prompt_cache_key: Option<String>,   // Conversation ID for caching
    pub text: Option<TextControls>,        // Verbosity/output schema
}
```

#### Building Instructions

```rust
pub fn get_full_instructions(&self, model: &ModelFamily) -> Cow<str> {
    let base = self.base_instructions_override
        .as_deref()
        .unwrap_or(model.base_instructions.deref());
    
    // Add apply_patch instructions if needed (for older models)
    if model.needs_special_apply_patch_instructions && !has_apply_patch_tool {
        Cow::Owned(format!("{base}\n{APPLY_PATCH_TOOL_INSTRUCTIONS}"))
    } else {
        Cow::Borrowed(base)
    }
}
```

#### Building Tools JSON

```rust
pub fn create_tools_json_for_responses_api(tools: &[ToolSpec]) -> Result<Vec<Value>> {
    tools.iter().map(|tool| serde_json::to_value(tool)).collect()
}
```

Each tool is serialized as:
```json
{
  "type": "function",
  "name": "read_file",
  "description": "Reads a local file...",
  "strict": false,
  "parameters": {
    "type": "object",
    "properties": {
      "file_path": { "type": "string", "description": "..." },
      "offset": { "type": "number", "description": "..." }
    },
    "required": ["file_path"]
  }
}
```

### HTTP Request Flow

```rust
async fn stream_responses(&self, prompt: &Prompt) -> Result<ResponseStream> {
    // 1. Build payload
    let payload = ResponsesApiRequest { ... };
    let payload_json = serde_json::to_value(&payload)?;
    
    // 2. Create HTTP request
    let mut req_builder = self.provider
        .create_request_builder(&self.client, &auth)
        .await?;
    
    req_builder = req_builder
        .header("conversation_id", self.conversation_id.to_string())
        .header("session_id", self.conversation_id.to_string())
        .header("Accept", "text/event-stream")
        .json(&payload_json);
    
    // 3. Send request with retries
    for attempt in 0..=max_attempts {
        match self.attempt_stream_responses(attempt, &payload_json).await {
            Ok(stream) => return Ok(stream),
            Err(retryable) => {
                tokio::time::sleep(retryable.delay(attempt)).await;
            }
        }
    }
}
```

### SSE Stream Processing

```rust
async fn process_sse<S>(stream: S, tx_event: mpsc::Sender<...>) {
    let mut stream = stream.eventsource();
    
    loop {
        let sse = timeout(idle_timeout, stream.next()).await?;
        let event: SseEvent = serde_json::from_str(&sse.data)?;
        
        match event.kind.as_str() {
            "response.created" => {
                tx_event.send(ResponseEvent::Created).await?;
            }
            "response.output_text.delta" => {
                tx_event.send(ResponseEvent::OutputTextDelta(delta)).await?;
            }
            "response.output_item.done" => {
                let item: ResponseItem = serde_json::from_value(event.item)?;
                tx_event.send(ResponseEvent::OutputItemDone(item)).await?;
            }
            "response.completed" => {
                let completed: ResponseCompleted = serde_json::from_value(event.response)?;
                tx_event.send(ResponseEvent::Completed {
                    response_id: completed.id,
                    token_usage: completed.usage.map(Into::into),
                }).await?;
                return;
            }
        }
    }
}
```

---

## From User Input to Tool Calls

### Example: User asks "What's in src/main.rs?"

#### Step 1: User Input Received

```rust
Op::UserTurn {
    items: vec![ResponseItem::Message {
        role: "user",
        content: vec![ContentItem::InputText { text: "What's in src/main.rs?" }],
    }],
}
```

#### Step 2: Build Prompt with History

```rust
Prompt {
    input: vec![
        // Previous conversation items...
        ResponseItem::Message { role: "user", content: [...] },
    ],
    tools: vec![
        ToolSpec::Function(ResponsesApiTool {
            name: "read_file",
            description: "Reads a local file...",
            parameters: JsonSchema::Object { ... },
        }),
        ToolSpec::Function(ResponsesApiTool {
            name: "shell",
            description: "Runs a shell command...",
            parameters: JsonSchema::Object { ... },
        }),
        // ... more tools
    ],
    parallel_tool_calls: true,
}
```

#### Step 3: Model Receives Request

**HTTP POST to `/responses`:**
```json
{
  "model": "gpt-5-codex",
  "instructions": "You are a coding agent...",
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [{"type": "input_text", "text": "What's in src/main.rs?"}]
    }
  ],
  "tools": [
    {
      "type": "function",
      "name": "read_file",
      "description": "Reads a local file...",
      "parameters": { ... }
    },
    ...
  ],
  "tool_choice": "auto",
  "parallel_tool_calls": true,
  "stream": true
}
```

#### Step 4: Model Streams Response

**SSE Events:**
```
event: response.created
data: {"type":"response.created","response":{...}}

event: response.output_item.added
data: {"type":"response.output_item.added","item":{"type":"function_call","name":"read_file","arguments":{"file_path":"src/main.rs"},"call_id":"call_123"}}

event: response.output_item.done
data: {"type":"response.output_item.done","item":{"type":"function_call","name":"read_file","arguments":{"file_path":"src/main.rs"},"call_id":"call_123"}}

event: response.completed
data: {"type":"response.completed","response":{"id":"resp_456","usage":{...}}}
```

#### Step 5: Codex Processes Tool Call

```rust
// In try_run_turn loop:
ResponseEvent::OutputItemDone(item) => {
    match ToolRouter::build_tool_call(sess.as_ref(), item.clone()) {
        Ok(Some(call)) => {
            // call = ToolCall {
            //     tool_name: "read_file",
            //     call_id: "call_123",
            //     payload: ToolPayload::Function {
            //         arguments: json!({"file_path": "src/main.rs"})
            //     }
            // }
            
            let response = tool_runtime.handle_tool_call(call).await?;
            // response = ResponseInputItem::FunctionCallOutput {
            //     call_id: "call_123",
            //     output: FunctionCallOutputPayload {
            //         content: "fn main() { ... }",
            //         success: Some(true)
            //     }
            // }
        }
    }
}
```

#### Step 6: Tool Executes Locally

```rust
// ToolRegistry::dispatch() routes to ReadFileHandler
ReadFileHandler::handle(invocation) => {
    let args: ReadFileArgs = serde_json::from_value(arguments)?;
    let content = std::fs::read_to_string(&args.file_path)?;
    
    Ok(ToolOutput::Function {
        content,
        success: true,
    })
}
```

#### Step 7: Result Formulated for Next Turn

```rust
// process_items() creates:
ResponseInputItem::FunctionCallOutput {
    call_id: "call_123",
    output: FunctionCallOutputPayload {
        content: "fn main() {\n    println!(\"Hello\");\n}",
        success: Some(true),
    }
}
```

This becomes part of the next turn's `input` array:

```json
{
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [{"type": "input_text", "text": "What's in src/main.rs?"}]
    },
    {
      "type": "function_call",
      "name": "read_file",
      "arguments": {"file_path": "src/main.rs"},
      "call_id": "call_123"
    },
    {
      "type": "function_call_output",
      "call_id": "call_123",
      "output": {
        "content": "fn main() {\n    println!(\"Hello\");\n}",
        "success": true
      }
    }
  ]
}
```

#### Step 8: Model Responds with Answer

The model receives the tool output and generates a final message:

```json
{
  "type": "message",
  "role": "assistant",
  "content": [{
    "type": "output_text",
    "text": "The file `src/main.rs` contains:\n\n```rust\nfn main() {\n    println!(\"Hello\");\n}\n```"
  }]
}
```

---

## Tool Execution and Result Formulation

### Tool Call Flow

```
Model Stream
    │
    ├─► ResponseEvent::OutputItemDone(FunctionCall { ... })
    │
    ▼
ToolRouter::build_tool_call()
    │
    ├─► Parses ResponseItem → ToolCall
    │   ├─► tool_name: "read_file"
    │   ├─► call_id: "call_123"
    │   └─► payload: ToolPayload::Function { arguments }
    │
    ▼
ToolCallRuntime::handle_tool_call()
    │
    ├─► Waits for tool_call_gate (readiness flag)
    ├─► Acquires parallel execution lock (if needed)
    │
    ▼
ToolRegistry::dispatch()
    │
    ├─► Looks up handler by tool_name
    │   └─► ReadFileHandler
    │
    ▼
ToolHandler::handle(ToolInvocation)
    │
    ├─► Parses arguments from JSON
    ├─► Executes tool logic
    │   └─► std::fs::read_to_string(file_path)
    │
    └─► Returns ToolOutput
        └─► ToolOutput::Function { content, success }
```

### Result Conversion

```rust
// ToolOutput → ResponseInputItem
impl ToolOutput {
    fn into_response(&self, call_id: &str, payload: &ToolPayload) -> ResponseInputItem {
        match (self, payload) {
            (ToolOutput::Function { content, success }, ToolPayload::Function { .. }) => {
                ResponseInputItem::FunctionCallOutput {
                    call_id: call_id.to_string(),
                    output: FunctionCallOutputPayload {
                        content: content.clone(),
                        success: Some(*success),
                        ..Default::default()
                    },
                }
            }
            (ToolOutput::Custom { content }, ToolPayload::Custom { .. }) => {
                ResponseInputItem::CustomToolCallOutput {
                    call_id: call_id.to_string(),
                    output: content.clone(),
                }
            }
            (ToolOutput::Mcp { result }, ToolPayload::Mcp { .. }) => {
                ResponseInputItem::McpToolCallOutput {
                    call_id: call_id.to_string(),
                    result: result.clone(),
                }
            }
        }
    }
}
```

### Parallel Tool Execution

When `parallel_tool_calls: true` and multiple tools are called:

```rust
// Model calls multiple tools simultaneously:
// - read_file("src/main.rs")
// - read_file("src/lib.rs")
// - grep_files("pattern")

// ToolCallRuntime uses RwLock for parallel tools:
let _guard = if supports_parallel {
    Either::Left(lock.read().await)  // Multiple readers allowed
} else {
    Either::Right(lock.write().await) // Exclusive access
};
```

All tool calls execute concurrently, results collected via `FuturesOrdered`.

---

## All Built-in Tools

### Core Tools (Always Available)

#### 1. `shell` / `local_shell` / `container.exec`
**Type:** Function  
**Description:** Runs a shell command and returns its output.

**Arguments:**
- `command` (array of strings, required): The command to execute
- `workdir` (string, optional): The working directory to execute the command in
- `timeout_ms` (number, optional): The timeout for the command in milliseconds
- `with_escalated_permissions` (boolean, optional): Whether to request escalated permissions
- `justification` (string, optional): 1-sentence explanation if escalated permissions needed

**Handler:** `ShellHandler`  
**Parallel Support:** No

#### 2. `update_plan`
**Type:** Function  
**Description:** Tracks steps and progress, renders them to the user.

**Arguments:**
- `steps` (array of objects, required): List of plan steps
  - `status` (string): "pending", "in_progress", or "completed"
  - `description` (string): Step description
- `explanation` (string, optional): Rationale for plan changes

**Handler:** `PlanHandler`  
**Parallel Support:** No

#### 3. `list_mcp_resources`
**Type:** Function  
**Description:** Lists resources provided by MCP servers.

**Arguments:**
- `server` (string, optional): MCP server name (omitted = all servers)
- `cursor` (string, optional): Opaque cursor from previous call

**Handler:** `McpResourceHandler`  
**Parallel Support:** Yes

#### 4. `list_mcp_resource_templates`
**Type:** Function  
**Description:** Lists resource templates provided by MCP servers.

**Arguments:**
- `server` (string, optional): MCP server name
- `cursor` (string, optional): Opaque cursor from previous call

**Handler:** `McpResourceHandler`  
**Parallel Support:** Yes

#### 5. `read_mcp_resource`
**Type:** Function  
**Description:** Read a specific resource from an MCP server.

**Arguments:**
- `server` (string, required): MCP server name
- `uri` (string, required): Resource URI to read

**Handler:** `McpResourceHandler`  
**Parallel Support:** Yes

#### 6. `view_image`
**Type:** Function  
**Description:** Attach a local image (by filesystem path) to the conversation context.

**Arguments:**
- `path` (string, required): Local filesystem path to an image file

**Handler:** `ViewImageHandler`  
**Parallel Support:** Yes

### Unified Exec Tools (When UnifiedExec Feature Enabled)

#### 7. `exec_command`
**Type:** Function  
**Description:** Runs a command in a PTY, returning output or a session ID for ongoing interaction.

**Arguments:**
- `cmd` (string, required): Shell command to execute
- `shell` (string, optional): Shell binary to launch (defaults to /bin/bash)
- `login` (boolean, optional): Whether to run shell with -l/-i semantics (defaults to true)
- `yield_time_ms` (number, optional): How long to wait (ms) for output before yielding
- `max_output_tokens` (number, optional): Maximum tokens to return (excess truncated)

**Handler:** `UnifiedExecHandler`  
**Parallel Support:** No

#### 8. `write_stdin`
**Type:** Function  
**Description:** Writes characters to an existing unified exec session and returns recent output.

**Arguments:**
- `session_id` (number, required): Identifier of the running unified exec session
- `chars` (string, optional): Bytes to write to stdin (may be empty to poll)
- `yield_time_ms` (number, optional): How long to wait (ms) for output before yielding
- `max_output_tokens` (number, optional): Maximum tokens to return

**Handler:** `UnifiedExecHandler`  
**Parallel Support:** No

### Experimental Tools (When Enabled)

#### 9. `read_file`
**Type:** Function  
**Description:** Reads a local file with 1-indexed line numbers, supporting slice and indentation-aware block modes.

**Arguments:**
- `file_path` (string, required): Absolute path to the file
- `offset` (number, optional): Line number to start reading from (1-indexed, ≥1)
- `limit` (number, optional): Maximum number of lines to return
- `mode` (string, optional): "slice" (default) or "indentation"
- `indentation` (object, optional): When mode="indentation"
  - `anchor_line` (number, optional): Anchor line (defaults to offset)
  - `max_levels` (number, optional): How many parent indentation levels to include
  - `include_siblings` (boolean, optional): Include blocks sharing anchor indentation
  - `include_header` (boolean, optional): Include doc comments/attributes above
  - `max_lines` (number, optional): Hard cap on lines returned

**Handler:** `ReadFileHandler`  
**Parallel Support:** Yes

#### 10. `list_dir`
**Type:** Function  
**Description:** Lists entries in a local directory with 1-indexed entry numbers and simple type labels.

**Arguments:**
- `dir_path` (string, required): Absolute path to the directory to list
- `offset` (number, optional): Entry number to start listing from (1-indexed, ≥1)
- `limit` (number, optional): Maximum number of entries to return
- `depth` (number, optional): Maximum directory depth to traverse (≥1)

**Handler:** `ListDirHandler`  
**Parallel Support:** Yes

#### 11. `grep_files`
**Type:** Function  
**Description:** Finds files whose contents match the pattern and lists them by modification time.

**Arguments:**
- `pattern` (string, required): Regular expression pattern to search for
- `include` (string, optional): Glob that limits which files are searched (e.g., "*.rs")
- `path` (string, optional): Directory or file path to search (defaults to session's working directory)
- `limit` (number, optional): Maximum number of file paths to return (defaults to 100)

**Handler:** `GrepFilesHandler`  
**Parallel Support:** Yes

#### 12. `test_sync_tool`
**Type:** Function  
**Description:** Internal synchronization helper used by Codex integration tests.

**Arguments:**
- `sleep_before_ms` (number, optional): Delay in milliseconds before any other action
- `sleep_after_ms` (number, optional): Delay in milliseconds after completing the barrier
- `barrier` (object, optional):
  - `id` (string, required): Identifier shared by concurrent calls
  - `participants` (number, required): Number of tool calls that must arrive before barrier opens
  - `timeout_ms` (number, optional): Maximum time in milliseconds to wait at the barrier

**Handler:** `TestSyncHandler`  
**Parallel Support:** Yes

### Conditional Tools

#### 13. `apply_patch`
**Type:** Function (or Freeform)  
**Description:** Applies a patch to files. Available in two variants:
- **Function variant:** Structured JSON patch format
- **Freeform variant:** Text-based unified diff format

**Arguments (Function variant):**
- `input` (string, required): JSON patch string

**Arguments (Freeform variant):**
- Uses freeform text format (unified diff)

**Handler:** `ApplyPatchHandler`  
**Parallel Support:** No  
**Availability:** When `ApplyPatchFreeform` feature enabled or model family requires it

#### 14. `web_search`
**Type:** WebSearch  
**Description:** Performs a web search query.

**Arguments:** (Handled by model, not structured)

**Handler:** Built into model (not local)  
**Parallel Support:** N/A  
**Availability:** When `WebSearchRequest` feature enabled

### MCP Tools

#### 15. MCP Server Tools (Dynamic)
**Type:** Function  
**Description:** Tools provided by external MCP servers. Names follow pattern `{server_name}/{tool_name}`.

**Arguments:** Defined by each MCP server's tool schema

**Handler:** `McpHandler`  
**Parallel Support:** Depends on tool  
**Availability:** When MCP servers are configured

---

## All Built-in Prompts

### 1. Base Prompt (`prompt.md`)

**Location:** `codex-rs/core/prompt.md`  
**Usage:** Default system instructions for all models

**Key Sections:**
- Personality and tone guidelines
- AGENTS.md file handling
- Responsiveness and preamble messages
- Planning tool usage
- Task execution guidelines
- Sandbox and approvals (skipped per requirements)
- Validating work
- Tool guidelines
- Final message formatting

**Excerpt:**
```
You are a coding agent running in the Codex CLI, a terminal-based coding assistant.
Codex CLI is an open source project led by OpenAI. You are expected to be precise, safe, and helpful.

Your capabilities:
- Receive user prompts and other context provided by the harness
- Communicate with the user by streaming thinking & responses
- Emit function calls to run terminal commands and apply patches
```

### 2. GPT-5 Codex Prompt (`gpt_5_codex_prompt.md`)

**Location:** `codex-rs/core/gpt_5_codex_prompt.md`  
**Usage:** Specific instructions for GPT-5 Codex models

**Key Sections:**
- General guidelines (bash commands, workdir usage, rg preference)
- Editing constraints (ASCII default, comments, apply_patch usage, git worktree handling)
- Plan tool usage
- Codex CLI harness configuration (skipped per requirements)
- Special user requests (time queries, review requests)
- Final message formatting

**Excerpt:**
```
You are Codex, based on GPT-5. You are running as a coding agent in the Codex CLI on a user's computer.

## General
- The arguments to `shell` will be passed to execvp(). Most terminal commands should be prefixed with ["bash", "-lc"].
- Always set the `workdir` param when using the shell function.
- When searching for text or files, prefer using `rg` or `rg --files` respectively.
```

### 3. Review Prompt (`review_prompt.md`)

**Location:** `codex-rs/core/review_prompt.md`  
**Usage:** Instructions for code review tasks

**Key Sections:**
- Review guidelines (bug identification criteria)
- Comment construction guidelines
- Finding prioritization (P0-P3)
- Output format (JSON schema)
- Formatting guidelines

**Excerpt:**
```
# Review guidelines:

You are acting as a reviewer for a proposed code change made by another engineer.

Below are some default guidelines for determining whether the original author would appreciate the issue being flagged.

These are not the final word in determining whether an issue is a bug. In many cases, you will encounter other, more specific guidelines.
```

**Output Schema:**
```json
{
  "findings": [
    {
      "title": "<≤ 80 chars, imperative>",
      "body": "<valid Markdown explaining why this is a problem>",
      "confidence_score": <float 0.0-1.0>,
      "priority": <int 0-3, optional>,
      "code_location": {
        "absolute_file_path": "<file path>",
        "line_range": {"start": <int>, "end": <int>}
      }
    }
  ],
  "overall_correctness": "patch is correct" | "patch is incorrect",
  "overall_explanation": "<1-3 sentence explanation>",
  "overall_confidence_score": <float 0.0-1.0>
}
```

### 4. Compact Prompt (`templates/compact/prompt.md`)

**Location:** `codex-rs/core/templates/compact/prompt.md`  
**Usage:** Instructions for conversation history compaction

**Purpose:** Guides the model on how to summarize conversation history when token limits are reached.

### 5. Sandboxing Assessment Prompt (`templates/sandboxing/assessment_prompt.md`)

**Location:** `codex-rs/core/templates/sandboxing/assessment_prompt.md`  
**Usage:** (Skipped per requirements - sandboxing related)

### 6. Review Exit Templates

**Location:** `codex-rs/core/templates/review/`  
**Usage:** Templates for review task completion

- `exit_success.xml`: Template for successful review completion
- `exit_interrupted.xml`: Template for interrupted review
- `history_message_completed.md`: Template for completed review history message
- `history_message_interrupted.md`: Template for interrupted review history message

### Prompt Selection Logic

```rust
pub fn get_full_instructions(&self, model: &ModelFamily) -> Cow<str> {
    // 1. Use override if provided
    if let Some(override) = self.base_instructions_override {
        return Cow::Borrowed(override);
    }
    
    // 2. Use model family's base instructions
    let base = model.base_instructions.deref();
    
    // 3. Add apply_patch instructions for older models if needed
    if model.needs_special_apply_patch_instructions && !has_apply_patch_tool {
        Cow::Owned(format!("{base}\n{APPLY_PATCH_TOOL_INSTRUCTIONS}"))
    } else {
        Cow::Borrowed(base)
    }
}
```

**Model Family Base Instructions:**
- Most models: `prompt.md`
- GPT-5 Codex: `gpt_5_codex_prompt.md`
- Custom models: Can specify custom prompt path

---

## Summary: Complete Turn Flow Diagram

```
User Input: "What's in src/main.rs?"
    │
    ▼
Op::UserTurn { items: [Message { role: "user", ... }] }
    │
    ▼
submission_loop() → spawn_task() → RegularTask::run()
    │
    ▼
run_turn(input: Vec<ResponseItem>)
    │
    ├─► Build ToolRouter (collects all tools)
    ├─► Build Prompt {
    │     input: [previous_history..., user_message],
    │     tools: [read_file, shell, ...],
    │     parallel_tool_calls: true
    │   }
    │
    ▼
ModelClient::stream(prompt)
    │
    ├─► Build ResponsesApiRequest {
    │     model: "gpt-5-codex",
    │     instructions: "You are a coding agent...",
    │     input: [conversation_history],
    │     tools: [tool_specs...],
    │     tool_choice: "auto",
    │     parallel_tool_calls: true,
    │     stream: true
    │   }
    │
    ├─► POST to /responses API
    │
    └─► Parse SSE stream
        │
        ├─► response.created
        ├─► response.output_item.added (FunctionCall started)
        ├─► response.output_item.done (FunctionCall ready)
        │   └─► ToolRouter::build_tool_call()
        │       └─► ToolCall { name: "read_file", call_id: "call_123", ... }
        │
        └─► response.completed
            │
            ▼
Process tool call:
    │
    ├─► ToolCallRuntime::handle_tool_call()
    │   └─► ToolRegistry::dispatch()
    │       └─► ReadFileHandler::handle()
    │           └─► std::fs::read_to_string("src/main.rs")
    │               └─► Returns: "fn main() { ... }"
    │
    └─► Convert to ResponseInputItem::FunctionCallOutput {
          call_id: "call_123",
          output: { content: "fn main() { ... }", success: true }
        }
    │
    ▼
process_items()
    │
    ├─► Record in conversation history:
    │   - FunctionCall { name: "read_file", call_id: "call_123", ... }
    │   - FunctionCallOutput { call_id: "call_123", output: "..." }
    │
    └─► Build next turn input:
        [
          previous_history...,
          user_message,
          FunctionCall { ... },
          FunctionCallOutput { ... }
        ]
    │
    ▼
Next turn (if model needs to respond):
    │
    └─► Model receives tool output
        └─► Generates final message: "The file contains: ..."
```

This completes the detailed analysis of Codex's turn flow and ModelClient architecture.
