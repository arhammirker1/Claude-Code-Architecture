# Claude Code — Deep Architecture Research
## How Anthropic Built a Production AI Agent with Action-Taking Tools

---

## Table of Contents

1. [The Big Picture: What Kind of System Is This?](#1-the-big-picture)
2. [The Core Loop: QueryEngine — The Brain](#2-the-core-loop-queryengine)
3. [The Tool System: How Actions Are Defined](#3-the-tool-system)
4. [The Permission System: Safety Layer](#4-the-permission-system)
5. [The Context System: How the AI "Sees" the World](#5-the-context-system)
6. [Multi-Agent Architecture: Swarms and Coordinators](#6-multi-agent-architecture)
7. [The MCP Protocol: Extensible External Tools](#7-the-mcp-protocol)
8. [State Management in an Agentic System](#8-state-management)
9. [The Command System vs. Tool System: Key Distinction](#9-commands-vs-tools)
10. [Memory and Persistence](#10-memory-and-persistence)
11. [The Bridge System: IDE Integration Pattern](#11-the-bridge-system)
12. [Startup Architecture: Parallelism as a First-Class Concern](#12-startup-architecture)
13. [Feature Flags and Dead Code Elimination](#13-feature-flags)
14. [Key Design Patterns to Copy for Your Own AI](#14-design-patterns-to-copy)
15. [Consolidated Architecture Diagram](#15-architecture-diagram)

---

## 1. The Big Picture

Claude Code is not a simple chatbot wrapper. It is a **full agentic system** — one where the AI can plan, execute, observe results, and iterate through multiple tool calls before producing a final answer. This is the ReAct (Reasoning + Acting) pattern at production scale.

The system has three conceptual layers:

```
┌─────────────────────────────────────────────┐
│           USER INTERFACE LAYER              │
│  React + Ink components, REPL, Commands     │
├─────────────────────────────────────────────┤
│           AGENTIC LOOP LAYER                │
│  QueryEngine ↔ Tools ↔ Permissions          │
├─────────────────────────────────────────────┤
│           INFRASTRUCTURE LAYER              │
│  State, Memory, MCP, Bridge, Services       │
└─────────────────────────────────────────────┘
```

The key insight: **tools are not functions that humans call — they are functions the LLM calls**. The AI decides when, how, and in what sequence to invoke them. The system's job is to make that safe, observable, and extensible.

---

## 2. The Core Loop: QueryEngine

`src/QueryEngine.ts` (~46,000 lines) is the single most important file. It contains the entire agentic loop. Here is what it does conceptually:

### The Agentic Loop in Pseudocode

```
function agenticLoop(userMessage, tools, context):
  messages = [systemPrompt, ...conversationHistory, userMessage]
  
  while true:
    response = callAnthropicAPI(messages, tools, streamingCallback)
    
    if response.stopReason == "end_turn":
      return response.text          // AI is done
    
    if response.stopReason == "tool_use":
      toolCalls = extractToolUses(response)
      toolResults = []
      
      for each toolCall in toolCalls:
        permission = checkPermission(toolCall, context)
        
        if permission.denied:
          toolResults.push(buildErrorResult(toolCall, permission.reason))
          continue
        
        if permission.needsUserApproval:
          approved = await promptUser(toolCall)
          if !approved:
            toolResults.push(buildRejectionResult(toolCall))
            continue
        
        result = await executeTool(toolCall)
        toolResults.push(result)
      
      // Append AI's tool calls + results to message history
      messages.push(response.content)           // assistant turn
      messages.push(buildToolResultMessage(toolResults))  // user turn (tool results)
      
      // Loop again — AI sees results and decides what to do next
      continue
    
    if response.stopReason == "max_tokens":
      handleTokenLimit()
      break
```

### Why This Design Matters

1. **The loop is the intelligence.** The AI doesn't get one shot — it can use dozens of tools in sequence, observing each result before deciding the next action. This is what makes it capable of complex multi-step tasks.

2. **Tool results re-enter as "user" turns.** In the Anthropic API, tool results are sent back in a `user` role message. This means the AI's conversation history grows with every tool call — which is why context compression (`services/compact/`) is critical for long sessions.

3. **Parallel tool calls are supported.** The API can return multiple `tool_use` blocks in a single response. Claude Code handles these concurrently when `isConcurrencySafe()` returns true, using `p-map` with configurable concurrency.

4. **Streaming is mandatory for UX.** The QueryEngine uses streaming so the UI can display text and tool calls as they arrive, not after the full response. This requires careful state management as partial tool-use blocks stream in.

### The Message Format (Anthropic API)

```typescript
// What goes INTO the API
messages = [
  { role: "user",      content: "Fix the bug in auth.ts" },
  { role: "assistant", content: [
    { type: "text",     text: "I'll look at auth.ts first." },
    { type: "tool_use", id: "tu_123", name: "FileRead", input: { path: "auth.ts" } }
  ]},
  { role: "user",      content: [
    { type: "tool_result", tool_use_id: "tu_123", content: "...file contents..." }
  ]},
  // Loop continues...
]
```

The system prompt is sent as a separate `system` parameter, not as a message. This is important because tools can inject text into the system prompt via their `prompt()` method.

---

## 3. The Tool System

This is the core of what makes Claude Code capable of taking actions. Every action the AI can perform is defined as a **Tool** — a self-contained module with a strict interface.

### The `buildTool` Factory

All tools are created with a factory function (`buildTool`) that enforces a consistent interface:

```typescript
const MyTool = buildTool({
  // IDENTITY
  name: 'BashTool',
  aliases: ['bash', 'shell'],          // Alternative names the LLM can use
  description: `Executes a shell command...`,  // THIS IS WHAT THE LLM READS
  
  // SCHEMA (Zod) — What input the LLM must provide
  inputSchema: z.object({
    command: z.string().describe('The shell command to run'),
    timeout: z.number().optional().describe('Timeout in ms, default 30000'),
  }),
  
  // EXECUTION — What actually happens when the LLM calls this tool
  async call(input, context, canUseTool, parentMessage, onProgress) {
    // input is fully typed from inputSchema
    const result = await executeShellCommand(input.command, input.timeout)
    return {
      data: result.stdout,
      // Optional: inject new messages into the conversation
      newMessages: result.stderr ? [{ type: 'text', text: result.stderr }] : undefined,
    }
  },
  
  // PERMISSION — Gate every invocation
  async checkPermissions(input, context) {
    if (context.permissionMode === 'bypass') return { granted: true }
    if (isDangerous(input.command)) return { granted: false, reason: 'Dangerous command' }
    return { granted: true, requiresApproval: true, prompt: `Run: ${input.command}?` }
  },
  
  // CONCURRENCY — Can multiple instances run simultaneously?
  isConcurrencySafe(input) {
    return isReadOnlyCommand(input.command)  // 'ls' yes, 'rm' no
  },
  
  // READ-ONLY — Does this modify state?
  isReadOnly(input) {
    return false  // BashTool can modify anything
  },
  
  // SYSTEM PROMPT INJECTION — Each tool can add context to the system prompt
  prompt(options) {
    return `You can run shell commands with BashTool. Working directory: ${options.cwd}`
  },
  
  // UI — How to display a call to this tool in the terminal
  renderToolUseMessage(input, options) {
    return <BashToolUI command={input.command} />  // React component
  },
  
  // UI — How to display the tool's result
  renderToolResultMessage(content, progressMessages, options) {
    return <BashResultUI output={content} />
  },
})
```

### What the LLM Actually Sees

When the API is called with tools, Anthropic converts each tool's schema into a JSON Schema description. The LLM is essentially given a function signature and decides when to call it. The **description field is the most important part** — it's what the LLM uses to decide which tool to invoke.

Here is the actual structure sent to the API:

```json
{
  "name": "BashTool",
  "description": "Executes a shell command in the current working directory...",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {
        "type": "string",
        "description": "The shell command to run"
      }
    },
    "required": ["command"]
  }
}
```

### The Full Tool Catalog (40+ tools)

**File System Tools:**
- `FileReadTool` — Reads files. Handles images (base64), PDFs (via file API), Jupyter notebooks
- `FileWriteTool` — Creates/overwrites files, with confirmation on overwrites
- `FileEditTool` — Precise partial edits via string replacement (avoids full rewrites)
- `NotebookEditTool` — Jupyter-specific editing (cell manipulation)

**Search Tools:**
- `GlobTool` — File pattern matching (e.g., `**/*.ts`)
- `GrepTool` — Content search using ripgrep under the hood (much faster than native JS)
- `WebSearchTool` — Live web search
- `WebFetchTool` — Fetch and parse URL content

**Execution Tools:**
- `BashTool` — Shell command execution with timeout, working directory, env vars
- `SkillTool` — Runs predefined "skill" workflows
- `MCPTool` — Dynamically calls tools from connected MCP servers

**Agent/Coordination Tools:**
- `AgentTool` — Spawns a sub-agent with its own conversation, tools, and context
- `SendMessageTool` — Inter-agent messaging in multi-agent setups
- `TeamCreateTool` / `TeamDeleteTool` — Creates agent teams for parallel work

**Mode Control Tools:**
- `EnterPlanModeTool` / `ExitPlanModeTool` — Switches to plan-only mode (no execution)
- `EnterWorktreeTool` / `ExitWorktreeTool` — Isolates work in a git worktree
- `SleepTool` — Waits in proactive/daemon mode
- `CronCreateTool` — Creates scheduled triggers

**Infrastructure Tools:**
- `LSPTool` — Language Server Protocol (hover info, go-to-definition, diagnostics)
- `ToolSearchTool` — Deferred tool discovery (loads tools lazily)
- `SyntheticOutputTool` — Generates structured output to a schema

### Tool Registration and Presets

Tools are registered in `src/tools.ts` and organized into **presets** — named collections of tools appropriate for different contexts:

```typescript
// Conceptual structure
const TOOL_PRESETS = {
  'readonly':      [FileReadTool, GlobTool, GrepTool, WebSearchTool],
  'standard':      [FileReadTool, FileWriteTool, FileEditTool, BashTool, GlobTool, GrepTool, ...],
  'agent':         [...standard, AgentTool, SendMessageTool, TaskCreateTool, ...],
  'plan-mode':     [FileReadTool, GlobTool, GrepTool, EnterPlanModeTool],  // No write tools!
}
```

The active preset changes based on permission mode, user configuration, and which features are enabled.

---

## 4. The Permission System

Every tool invocation goes through a permission check before execution. This is the safety layer that prevents the AI from doing destructive things without user consent.

### Permission Modes

```
bypassPermissions → auto-approve everything (dangerous, for automation)
plan              → show plan first, ask once, then execute
default           → prompt per operation (most common)
auto              → ML classifier decides what's safe
```

### How a Permission Check Works

```
LLM decides to call BashTool("rm -rf /tmp/test")
       ↓
QueryEngine calls tool.checkPermissions(input, context)
       ↓
Tool returns { requiresApproval: true, prompt: "Delete /tmp/test?" }
       ↓
QueryEngine checks permission rules (from config)
  - If rule matches "Bash(rm *)" → auto-approve or auto-deny
  - Else → show UI prompt to user
       ↓
User approves/denies via terminal UI
       ↓
If approved: execute tool, return result
If denied:   return { type: "tool_result", content: "User denied" }
       ↓
AI sees the denial and decides what to do next
```

### Permission Rules: Wildcard Patterns

Users configure rules like:
```
Bash(git *)      → auto-approve any git command
FileEdit(/src/*) → auto-approve edits to files under /src
FileEdit(*)      → auto-approve all file edits (dangerous)
Bash(rm *)       → always require approval
```

The pattern `ToolName(argument_pattern)` is matched against the tool name and its primary argument. This gives users fine-grained control.

### Why Returning Denials to the AI is Smart

Instead of crashing when a tool is denied, Claude Code returns the denial as a tool result. The AI then **adapts** — it might try an alternative approach, ask the user for clarification, or give up gracefully. This makes the system much more robust than hard-stopping on permission failures.

---

## 5. The Context System

`src/context.ts` is responsible for collecting everything the AI needs to understand the environment before it starts working.

### What Context Is Collected

```typescript
interface SystemContext {
  // Environment
  cwd: string                    // Current working directory
  platform: string               // 'darwin', 'linux', 'win32'
  shell: string                  // '/bin/zsh', '/bin/bash'
  
  // Git information
  gitRoot: string | null         // Root of git repo
  gitBranch: string | null       // Current branch
  gitStatus: string | null       // Modified/staged files
  gitLog: string | null          // Recent commits
  
  // Project context
  directoryListing: string       // Files in cwd
  memoryContents: string[]       // Contents of CLAUDE.md files
  projectInstructions: string    // .claude/INSTRUCTIONS.md
  
  // User/session context
  userName: string               // System username
  sessionId: string              // Unique session ID
  conversationHistory: Message[] // Previous turns
  
  // Runtime
  availableTools: Tool[]         // Currently active tools
  permissionMode: PermissionMode // Current permission setting
  featureFlags: Record<string, boolean>
}
```

### How Context is Injected

All of this becomes part of the system prompt. The AI reads the git status, the directory listing, and the memory files before deciding what to do. Each tool can also contribute to the system prompt via its `prompt()` method:

```typescript
// BashTool might add to system prompt:
"Working directory: /home/user/myproject
Available shell: /bin/zsh  
Note: prefer 'fd' over 'find' when available"

// FileEditTool might add:
"When editing files, always use the exact string replacement format.
Never output the entire file — only the changed section."
```

This per-tool system prompt contribution is a powerful pattern: the AI gets exactly the right instructions for each capability it has available.

---

## 6. Multi-Agent Architecture

This is the most sophisticated part of the system. Claude Code supports spawning sub-agents that run their own independent agentic loops.

### AgentTool: Spawning Sub-Agents

```typescript
// The LLM can call this to spawn a specialized sub-agent
AgentTool.inputSchema = z.object({
  task: z.string().describe('The task to give the sub-agent'),
  tools: z.array(z.string()).optional().describe('Which tools to give the sub-agent'),
  context: z.string().optional().describe('Additional context for the sub-agent'),
})

// Under the hood, AgentTool creates a new QueryEngine instance:
async call(input, parentContext) {
  const subAgent = new QueryEngine({
    systemPrompt: buildSubAgentPrompt(input.task, input.context),
    tools: selectTools(input.tools),
    maxIterations: 50,
    parentContext,  // Sub-agent can report back to parent
  })
  
  const result = await subAgent.run(input.task)
  return { data: result.finalAnswer }
}
```

### Team Architecture

For complex parallelizable tasks, Claude Code can create **teams** of agents:

```
TeamCreateTool(task="refactor codebase")
       ↓
Coordinator creates N sub-agents:
  Agent A: "Handle authentication module"
  Agent B: "Handle database layer"  
  Agent C: "Handle API endpoints"
       ↓
Agents work in parallel using SendMessageTool to coordinate
       ↓
Coordinator aggregates results
```

### Coordinator Module

`src/coordinator/` contains the supervisor logic:
- Assigns tasks to agents
- Monitors agent progress
- Handles agent failures (retry, reassign)
- Merges results from parallel workstreams

### Why This Matters for Your AI

The key insight: **recursive agent spawning is how you get "deep" intelligence**. A top-level agent that can only make ~20 tool calls is limited. But a top-level agent that can spawn sub-agents, each making their own tool calls, can tackle arbitrarily complex tasks. The challenge is coordination and preventing loops.

---

## 7. The MCP Protocol

MCP (Model Context Protocol) is how Claude Code gets **external tools** at runtime, without modifying the core codebase.

### How MCP Works

```
MCP Server (external process)
├── Exposes: tools, resources, prompts
├── Transport: stdio, SSE, or WebSocket
└── Protocol: JSON-RPC over stdio

Claude Code (MCP Client)
├── Discovers tools from connected servers
├── Dynamically adds them to the available tool set
└── Calls them via MCPTool
```

### MCPTool: The Dynamic Dispatcher

```typescript
// MCPTool is a meta-tool that wraps any MCP tool
// The LLM sees each MCP tool as if it were a native tool
// But under the hood, they all route through MCPTool

async call(input, context) {
  const { serverName, toolName, toolInput } = input
  const server = getMCPServer(serverName)
  const result = await server.callTool(toolName, toolInput)
  return { data: result }
}
```

### MCP Configuration

```json
// .mcp.json in the project
{
  "mcpServers": {
    "my-database-tool": {
      "command": "node",
      "args": ["./mcp-servers/database/index.js"],
      "env": { "DB_URL": "postgres://..." }
    },
    "my-api-tool": {
      "command": "python",
      "args": ["./mcp-servers/api.py"]
    }
  }
}
```

This is genius: external developers can add tools to Claude Code without touching Anthropic's codebase. The LLM sees them as native tools. You can do this in your own system.

---

## 8. State Management

State in an agentic system is more complex than in a regular web app because:
- State changes during tool execution (file system, git, etc.)
- Multiple agents may run concurrently
- State must be observable by the UI in real-time

### AppState Structure

```typescript
interface AppState {
  // Conversation
  messages: Message[]             // Full conversation history
  currentStreamingMessage: string // Partial message being streamed
  
  // Tool execution
  pendingToolCalls: ToolUse[]     // Tools waiting for permission
  runningToolCalls: ToolUse[]     // Tools currently executing  
  completedToolCalls: ToolUse[]   // Tools that finished
  
  // Progress
  progressMessages: ProgressMessage[]  // Real-time progress updates
  
  // Cost tracking
  inputTokens: number
  outputTokens: number
  costUSD: number
  
  // Session
  sessionId: string
  permissionMode: PermissionMode
  isAgentRunning: boolean
  
  // UI state
  isCompact: boolean             // Whether context was compressed
  theme: Theme
}
```

### The `onProgress` Callback

During long-running tool calls, tools can emit progress updates:

```typescript
async call(input, context, canUseTool, parentMessage, onProgress) {
  onProgress({ type: 'text', text: 'Reading file...' })
  const content = await readFile(input.path)
  
  onProgress({ type: 'text', text: 'Parsing...' })
  const parsed = parse(content)
  
  return { data: parsed }
}
```

These progress updates appear in the UI in real-time, even before the tool finishes. This is critical for UX — users need to know the system is alive during slow operations.

---

## 9. Commands vs. Tools: Key Distinction

This distinction is often confused. Here's what each one is:

### Tools: What the AI Calls

- Defined in `src/tools/`
- Invoked by the **LLM** during an agentic loop
- Take structured JSON input (validated by Zod)
- Return structured results that re-enter the conversation
- Examples: `BashTool`, `FileReadTool`, `FileEditTool`

### Commands: What the User Calls

- Defined in `src/commands/`
- Invoked by the **human** with `/command-name` in the REPL
- Are shorthand for complex operations
- May trigger an agentic loop with specific tools and prompts
- Examples: `/commit`, `/review`, `/compact`

### Three Command Types

```typescript
// 1. PromptCommand — Most common. Packages a user message + tool set, launches agentic loop
{
  type: 'prompt',
  name: 'commit',
  allowedTools: ['Bash(git *)', 'FileRead(*)'],
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: 'Generate a commit message and commit staged changes' }]
  }
}

// 2. LocalCommand — Runs immediately in-process, no LLM call
{
  type: 'local',
  name: 'clear',
  async call(args, context) {
    return 'Conversation cleared.'
  }
}

// 3. LocalJSXCommand — Returns React UI component
{
  type: 'local-jsx',
  name: 'cost',
  async call(args, context) {
    return <CostDisplay tokens={context.totalTokens} cost={context.totalCost} />
  }
}
```

---

## 10. Memory and Persistence

### CLAUDE.md Files: The Memory System

Claude Code uses a hierarchical file-based memory system. Before every session, it reads:

```
~/.claude/CLAUDE.md              → User-global preferences
/project-root/CLAUDE.md          → Project-specific instructions
/project-root/.claude/CLAUDE.md  → Additional project config
```

These files are plain markdown that humans write. They inject persistent context into every session. Example:

```markdown
# My Project Instructions

## Coding Style
- Always use TypeScript strict mode
- Prefer `const` over `let`
- Use Zod for all external data validation

## Architecture
- Keep business logic in src/services/
- Database queries only in src/db/
- Never use `any` type

## Common Tasks
- To run tests: bun test
- To build: bun run build
```

The AI reads this every session, so it "remembers" project conventions.

### Automatic Memory Extraction

`services/extractMemories/` watches conversations and automatically extracts facts to save:
- Project conventions discovered during work
- User preferences stated in conversation
- Useful patterns found during debugging

These get written back to CLAUDE.md files, creating a self-improving memory system.

### Session Resumption

Claude Code can save and restore sessions:
- Full conversation history is serialized to disk
- Tool call results are stored
- Sessions can be resumed with `/resume`

---

## 11. The Bridge System

`src/bridge/` handles bidirectional communication with IDE extensions (VS Code, JetBrains).

### Architecture

```
VS Code Extension                  Claude Code CLI
      │                                  │
      │  ←── JSON-RPC over stdio ───→   │
      │                                  │
[File operations]          [Execute agentic loop]
[Show diffs in editor]     [Send tool results]
[User approvals in UI]     [Stream progress]
```

### Why This Matters

The bridge solves a fundamental UX problem: terminal tool calls are opaque. The bridge lets IDE users:
- See diffs in the actual editor (not raw terminal output)
- Approve/deny file changes with inline UI
- Navigate to files being edited
- Track agent progress in the sidebar

### Bridge Protocol

```typescript
// Messages going TO the IDE
type OutboundMessage = 
  | { type: 'tool_use',    tool: string, input: unknown }
  | { type: 'progress',    message: string }
  | { type: 'file_edit',   path: string, diff: string }
  | { type: 'ask_user',    question: string, options: string[] }
  | { type: 'result',      content: string }

// Messages coming FROM the IDE
type InboundMessage =
  | { type: 'user_input',  text: string }
  | { type: 'approve',     toolUseId: string }
  | { type: 'deny',        toolUseId: string, reason: string }
  | { type: 'cancel' }
```

---

## 12. Startup Architecture

Claude Code's startup sequence reveals sophisticated engineering for performance:

### Parallel Initialization

```typescript
// src/main.tsx — These fire immediately, before any heavy modules load
startMdmRawRead()       // MDM policy (enterprise device management)
startKeychainPrefetch() // API key from OS keychain

// These happen in parallel:
await Promise.all([
  initTelemetry(),      // OpenTelemetry
  initOAuth(),          // Auth check
  loadFeatureFlags(),   // GrowthBook
  prefetchAPIConnection() // TCP handshake to Anthropic API
])

// Only THEN do heavy modules load
const { startREPL } = await import('./replLauncher.tsx')
```

### Lazy Loading Heavy Modules

```typescript
// OpenTelemetry is ~400KB and gRPC is ~700KB
// These are only imported when actually needed:
const getTelemetry = () => import('./services/analytics/telemetry.js')
```

### Why This Matters for Your AI

Cold start time matters enormously for developer tools. Users measure trust in milliseconds. Every 100ms of startup delay is felt.

---

## 13. Feature Flags and Dead Code Elimination

One of the most elegant engineering decisions in the codebase:

### Bun Bundle Feature Flags

```typescript
import { feature } from 'bun:bundle'

// At BUILD TIME, these conditions are evaluated
// Dead code is eliminated entirely from the bundle
if (feature('VOICE_MODE')) {
  const { VoiceInput } = await import('./voice/index.js')
  tools.push(VoiceTool)
}

if (feature('PROACTIVE')) {
  tools.push(SleepTool, CronCreateTool, RemoteTriggerTool)
}

if (feature('COORDINATOR_MODE')) {
  tools.push(TeamCreateTool, TeamDeleteTool, SendMessageTool)
}
```

### What Flags Control

| Flag | What It Enables |
|---|---|
| `PROACTIVE` | Daemon mode, cron triggers, sleep tool |
| `KAIROS` | Unknown (internal codename, likely scheduling) |
| `BRIDGE_MODE` | IDE integration tools |
| `VOICE_MODE` | Voice input/output |
| `COORDINATOR_MODE` | Multi-agent teams |
| `DAEMON` | Background running mode |
| `WORKFLOW_SCRIPTS` | Automated workflow support |

### The `USER_TYPE === 'ant'` Gate

Many experimental features are gated behind `process.env.USER_TYPE === 'ant'` — only Anthropic employees see them. This lets them ship features incrementally without user-facing flags.

---

## 14. Design Patterns to Copy for Your Own AI

Here are the most valuable patterns from Claude Code, ordered by impact:

### Pattern 1: The Tool Interface Contract

Every tool must answer these questions:
1. **What am I?** (name, description for LLM)
2. **What do I accept?** (Zod input schema)
3. **What do I do?** (`call` function)
4. **Do I need permission?** (`checkPermissions`)
5. **Am I safe to parallelize?** (`isConcurrencySafe`)
6. **Do I modify state?** (`isReadOnly`)
7. **What should the LLM know about me?** (`prompt` contribution)
8. **How do I look in the UI?** (`renderToolUseMessage`, `renderToolResultMessage`)

This forces clarity. Before adding a tool, you must think through all eight dimensions.

### Pattern 2: Deny-as-Result, Not Deny-as-Crash

When permission is denied, return a tool result saying so. Don't throw an exception or stop the loop. The AI can handle "I wasn't allowed to do that" and choose an alternative.

### Pattern 3: Progressive Disclosure via onProgress

Long-running tools should emit progress updates continuously. Users need to know the system is alive. Silence for >2 seconds feels like a crash.

### Pattern 4: System Prompt Injection per Tool

Each tool knows what context it needs the LLM to have. Let tools contribute to the system prompt. This keeps tool-specific instructions co-located with the tool code, not scattered in a monolithic system prompt.

### Pattern 5: Context-First Startup

Before the first LLM call, collect everything:
- Directory structure
- Git status
- Relevant config files
- Memory files
- Environment details

An LLM that knows its context makes far fewer unnecessary tool calls.

### Pattern 6: Recursive Agent Spawning

Build your top-level agent to be capable of creating sub-agents. This is how you scale beyond what a single context window can handle. The key requirements:
- Sub-agents need isolated contexts
- Sub-agents need a way to return results to parent
- The parent needs a way to combine partial results

### Pattern 7: MCP for Extensibility

Don't bake all tools into your core. Use MCP (or your own equivalent protocol) to allow external tool servers. This separates concerns: your AI core stays stable, while capabilities grow via plugins.

### Pattern 8: Permission Modes, Not Binary On/Off

Don't make permission a yes/no. Have modes:
- `bypass`: trust everything (CI/automation)
- `plan`: show plan, approve once
- `interactive`: approve each action
- `readonly`: no state modification at all

### Pattern 9: Memory Files as Persistent Context

Use human-readable files (like CLAUDE.md) as the memory system. They're:
- Editable by humans
- Version-controllable in git
- Inspectable without special tools
- Hierarchical (global → project → session)

### Pattern 10: Tool Concurrency Metadata

Mark tools as concurrency-safe or not. When the LLM emits multiple tool calls in one response, run the safe ones in parallel. This can cut execution time dramatically for read-heavy tasks.

---

## 15. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  REPL Input  │  │  Command Bar │  │  Tool Progress Display │ │
│  │  (Ink/React) │  │  /commit etc │  │  (streaming updates)   │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────────────────┘ │
└─────────┼─────────────────┼────────────────────────────────────┘
          │                 │
          ▼                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                        QUERY ENGINE                              │
│                   (src/QueryEngine.ts)                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   AGENTIC LOOP                           │    │
│  │                                                          │    │
│  │  [User Message] ──→ [API Call] ──→ [Response]            │    │
│  │       ↑                                  │              │    │
│  │       │           stop_reason == tool_use│              │    │
│  │       │                    ↓              │              │    │
│  │  [Tool Results] ←── [Execute Tools] ←────┘              │    │
│  │                            │                             │    │
│  │                    ┌───────┴────────┐                    │    │
│  │                    │  Permission    │                    │    │
│  │                    │  Check         │                    │    │
│  │                    └───────┬────────┘                    │    │
│  │                            │                             │    │
│  │            ┌───────────────┤                             │    │
│  │            │               │                             │    │
│  │         [Deny]          [Approve]                        │    │
│  │            │               │                             │    │
│  │     return error    execute tool                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                      TOOL LAYER                                  │
│                                                                  │
│  File System      Execution       Search        Agent           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐  ┌──────────┐    │
│  │FileRead  │    │BashTool  │    │GrepTool  │  │AgentTool │    │
│  │FileWrite │    │SkillTool │    │GlobTool  │  │TeamCreate│    │
│  │FileEdit  │    │          │    │WebSearch │  │SendMsg   │    │
│  └──────────┘    └──────────┘    └──────────┘  └────┬─────┘    │
│                                                       │          │
│  External (MCP)   LSP         Mode Control           │          │
│  ┌──────────┐    ┌──────────┐  ┌──────────┐          │          │
│  │MCPTool   │    │LSPTool   │  │PlanMode  │          │          │
│  │(dynamic) │    │          │  │Worktree  │          │          │
│  └──────────┘    └──────────┘  └──────────┘          │          │
└──────────────────────────────────────────────────────┼──────────┘
                                                        │
                                                        ▼
                                            ┌───────────────────┐
                                            │   SUB-AGENT        │
                                            │  (New QueryEngine) │
                                            │  own tools/context │
                                            └───────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE LAYER                          │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  AppState   │  │   Memory    │  │  MCP Client │             │
│  │  (React ctx)│  │  CLAUDE.md  │  │  (external) │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Bridge    │  │   Context   │  │   Services  │             │
│  │  (IDE sync) │  │  (git/env)  │  │  (API/OAuth)│             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary: The Five Core Insights

1. **The loop is the intelligence.** The LLM calls tools, observes results, and loops. Everything else is infrastructure supporting this loop.

2. **Tools are contracts, not functions.** Every tool must declare its identity, schema, permissions, concurrency safety, and UI. This forces clarity and enables safe automation.

3. **Deny gracefully, never crash.** Permission denials return as tool results. The AI adapts. This is what makes the system robust in the real world.

4. **Context is collected before the first word.** Git status, directory listing, memory files, env details — all of it goes into the system prompt before the LLM says anything. An LLM that knows its context makes far fewer unnecessary calls.

5. **Extensibility via protocol, not modification.** MCP lets external developers add tools without touching the core. Your AI's capabilities should grow via plugins, not PRs.

---

*Research based on analysis of the leaked Claude Code source (2026-03-31). ~1,900 files, 512K+ lines of TypeScript.*
