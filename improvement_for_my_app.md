# AI System Audit
## Compared Against Production Agentic Architecture Patterns

---

## Executive Summary

Your AI system has the right instincts — a tool loop, read/write separation, context building — but several architectural decisions are causing reliability problems, token waste, and poor LLM behavior. The root issues cluster into **5 critical problems** and **8 improvement areas**. Many of the bandaid fixes in your code (temp=0 retries, mixed batch detection, `needs_confirmation` required param) exist because of upstream architectural mistakes that, if fixed, eliminate the need for the bandaid entirely.

---

## CRITICAL PROBLEMS (Fix These First)

---

### CRITICAL 1: `command-context.ts` Is Massive Dead Code — But Its Pattern Is Leaking

**File:** `lib/ai/command-context.ts`
**Severity:** High — 14 parallel DB queries that are never called in production

Your `command/route.ts` correctly uses `buildMiniContext`. But `command-context.ts` still exists — a 400-line monster that runs 14 parallel Supabase queries fetching tasks, projects, events, relationships, clients, team members, vault items, vault folders, chat rooms, messages, and LinkedIn posts all at once, then dumps it into the system prompt.

This was your old approach before you switched to mini-context + read tools. The problem: the pattern isn't fully dead. The architectural thinking behind it — "give the AI everything upfront" — still infects your system in subtle ways.

**The specific harm:**
- If this function ever gets called (e.g., someone imports it by accident or during a refactor), a single AI request hits 14 database queries simultaneously
- The output is thousands of tokens of context the AI mostly ignores
- The `console.log("[CMD-CTX] === FULL CONTEXT START ===")` + full context dump is in there, meaning if triggered it prints your entire workspace to Vercel logs

**Fix:**
```typescript
// Delete command-context.ts entirely.
// The mini-context + read tools pattern is correct. Commit to it.
```

---

### CRITICAL 2: The Agentic Loop Has Two Different Streaming Behaviors Based on Step Number

**File:** `app/api/ai/command/route.ts` lines ~155-185 and ~200-220

This is the most confusing bug in the system. Your loop behaves differently depending on whether it's the first step:

```typescript
// Step 0, no tools → streams live (good UX)
const directStream = await groq.chat.completions.create({ stream: true, ... })

// Step 1+, no tools → sends everything at once (broken UX)
const content = choice?.message?.content || ""
return createSSEResponse(content, actionEvents)  // NOT streaming
```

This means: if the AI uses one or more tools before responding (which is the normal case for any real request), the final response is delivered in a single chunk. The user sees nothing, then suddenly the full answer appears. The typing animation that users expect from AI is gone for 90% of requests.

**Why this happened:** You correctly avoided calling the API a second time when you already have a non-streaming response from the loop. But the fix was wrong — you should stream the pre-generated content word-by-word, not send it as a single SSE delta.

**Fix:**
```typescript
// When step > 0 and no tools, simulate streaming with the generated content
if (step > 0) {
  const content = choice?.message?.content || ""
  const encoder = new TextEncoder()
  const readable = new ReadableStream({
    async start(controller) {
      for (const event of actionEvents) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ type: "action_executed", ...event })}\n\n`))
      }
      // Stream word by word instead of dumping all at once
      const words = content.split(' ')
      for (const word of words) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ type: "delta", content: word + ' ' })}\n\n`))
        await new Promise(r => setTimeout(r, 20)) // ~50 words/sec
      }
      controller.enqueue(encoder.encode(`data: ${JSON.stringify({ type: "done" })}\n\n`))
      controller.close()
    }
  })
  return new Response(readable, { headers: SSE_HEADERS })
}
```

---

### CRITICAL 3: The `delete_task` Tool Schema Forces the LLM to Control Its Own Safety Gate

**File:** `lib/ai/tools.ts` and `lib/ai/mcp-read-tools.ts`

```typescript
// CURRENT (broken)
{
  name: "delete_task",
  parameters: {
    properties: {
      task_title: { type: "string" },
      needs_confirmation: { type: "boolean", description: "Must be true" },  // ← wrong
    },
    required: ["task_title", "needs_confirmation"],  // ← the LLM controls safety
  }
}
```

You're asking the LLM to pass `needs_confirmation: true` as a parameter. This is fundamentally wrong for two reasons:

1. **The LLM can pass `false`**. Any model, any temperature, any instruction drift can produce `needs_confirmation: false`. Your system then has no protection. You've made safety a matter of LLM compliance rather than system enforcement.

2. **It causes schema validation errors**. Groq's Llama 4 sometimes passes a string `"true"` instead of boolean `true`, triggering your 400 retry path. This is one of the main causes of your `tool_use_failed` errors.

**Fix:** Remove `needs_confirmation` from the schema entirely. Make it unconditional in the executor:

```typescript
// In action-executor.ts — always requires confirmation, never LLM's choice
async function executeDeleteTask(args, ctx): Promise<ActionResult> {
  const task = await findTaskByTitle(args.task_title, ctx.founder_id)
  if (!task) return { success: false, message: `Task "${args.task_title}" not found.` }

  // ALWAYS return confirmation requirement — not controlled by LLM
  return {
    success: true,
    needs_confirmation: true,  // System enforces this, not the LLM
    message: `Found: "${task.title}". Confirm deletion?`,
    confirmation_action: {
      tool: "delete_task_confirmed",
      resolved_id: task.id,
      description: `Delete "${task.title}"`,
    },
  }
}
```

```typescript
// Tool schema — simpler
{
  name: "delete_task",
  description: "Delete a task. Always requires user confirmation before executing.",
  parameters: {
    properties: {
      task_title: { type: "string", description: "Task title to find (fuzzy match)" }
    },
    required: ["task_title"]
  }
}
```

---

### CRITICAL 4: The Mixed Batch Detection Is Treating a Symptom, Not the Disease

**File:** `app/api/ai/command/route.ts` lines ~240-270

```typescript
// CURRENT — complex symptom treatment
const hasReadCalls = toolCalls.some((tc: any) => READ_TOOL_NAMES.has(tc.function.name))
const hasActionCalls = toolCalls.some((tc: any) => !READ_TOOL_NAMES.has(tc.function.name))
const isMixedBatch = hasReadCalls && hasActionCalls

if (isMixedBatch) {
  // defer action tools, return fake "deferred" tool result
  // hope the LLM re-calls the action tool with real data next step
}
```

This is a 30-line workaround for a system prompt problem. The LLM batches read+action tools together because it doesn't understand that it must complete reads first. The fix is making this constraint crystal clear in the system prompt and tool descriptions — not detecting and deferring at runtime.

**Why the current approach fails:**
- The "deferred" tool result message tells the LLM to "call the action tool again in the next step" — but after deferral, the conversation includes both the read result AND the deferred message, so the LLM now has two conflicting signals about what to do next
- It adds a full API round-trip for every mixed batch (extra latency, extra tokens, extra cost)
- It's invisible to you in monitoring — you see 4 steps instead of 3, with no clear signal that deferral happened

**Fix: Enforce sequencing in the system prompt and tool descriptions:**

```typescript
// In system prompt:
`## Tool Sequencing — MANDATORY
1. GATHER phase: Call ALL needed read tools (get_team_workload, get_projects, get_vault_files)
2. ACT phase: Call the action tool ONCE with ALL parameters resolved from step 1

NEVER call a read tool and an action tool in the same response. Always complete the GATHER phase first.`

// In create_task description — add at the start:
`Create a task. PREREQUISITE: Call get_team_workload before assigning to a team member. Call get_vault_files before attaching vault files. Call get_projects before linking to a project. Only call this tool AFTER all lookups are complete.`
```

Then remove the mixed batch detection entirely. The system prompt constraint does the same job without the complexity.

---

### CRITICAL 5: `action-executor.ts` Has Silent Auto-Fetch Fallbacks That Hide Missing Data

**File:** `lib/ai/action-executor.ts` — `resolveTeamMember` and `resolveProject` functions

```typescript
async function resolveTeamMember(name, team, founderId) {
  // Auto-fetch team data if not pre-loaded by read tools
  if (team.length === 0) {  // ← silent fallback
    const { data: members } = await supabaseAdmin
      .from("team_members")
      // ...fetches team members + task counts
  }
  // ...
}
```

When `ctx.team` is empty (because the AI didn't call `get_team_workload`), the executor silently runs the query itself. This looks like it's "helping" but it actually:

1. **Masks AI misbehavior**: The LLM should have called `get_team_workload` first. When the executor covers for it, you never know the LLM skipped a required step.
2. **Runs DB queries you can't monitor**: These queries happen inside the executor, not in your SSE loop logging. You lose visibility.
3. **Does it with less data**: The auto-fetch doesn't load the `actionContext` team data the same way the read tool does, so `ctx.team` stays empty for the duration of the request, meaning the next resolution attempt also auto-fetches.
4. **Adds latency silently**: Every `create_task` with an assignee that skipped `get_team_workload` now adds a hidden DB query inside `executeCreateTask`.

**Fix: Remove the auto-fetch. Return a clear error instead:**

```typescript
async function resolveTeamMember(name, team, founderId) {
  if (team.length === 0) {
    return null  // Don't auto-fetch — let the executor return an error
  }
  // ... fuzzy match against pre-loaded team
}

// In executeCreateTask:
if (assigned_to_name && ctx.team.length === 0) {
  return {
    success: false,
    message: `I need to look up team members first before assigning. Please wait.`
    // This causes the LLM to call get_team_workload on the next step
  }
}
```

---

## IMPORTANT IMPROVEMENTS (High-Value, Lower Urgency)

---

### IMPROVEMENT 1: Split Route Architecture Duplicates Logic

**Files:** `app/api/ai/chat/route.ts`, `app/api/ai/command/route.ts`

You have two separate routes with separate agentic loops. They differ only in:
- Chat: `READ_TOOLS` only, max 3 steps, saves to `chat_messages`
- Command: `ALL_TOOLS`, max 4 steps, emits `action_executed` events

This duplication means any bug fix to the loop logic must be applied to both files. The loop-exhausted path, the mixed-batch detection (in command), the step logging, and the SSE helpers are all duplicated.

**Recommended structure:**
```typescript
// lib/ai/agent-loop.ts — single loop implementation
export async function runAgentLoop({
  messages,
  tools,
  maxSteps,
  onToolCall,      // callback: (toolName, args) => result
  onProgress,      // callback: (update) => void  (SSE delta)
  onActionEvent,   // callback: (event) => void   (action_executed)
}): Promise<{ content: string; toolsCalled: string[] }>

// app/api/ai/chat/route.ts — thin wrapper
// app/api/ai/command/route.ts — thin wrapper
```

---

### IMPROVEMENT 2: Tool Definitions Have No Contract — They're Just JSON

**File:** `lib/ai/tools.ts`, `lib/ai/mcp-read-tools.ts`

Your tools are raw JSON objects passed to the Groq API. They have no lifecycle, no metadata for your own system, and no shared interface. Compare to Claude Code's pattern:

```typescript
// CURRENT — just a JSON blob
export const ACTION_TOOLS = [{
  type: "function",
  function: { name: "create_task", ... }
}]

// BETTER — a contract with behavioral metadata
interface Tool {
  // LLM-facing
  name: string
  description: string
  inputSchema: ZodSchema
  
  // System-facing (your code uses these)
  execute: (args, context) => Promise<ToolResult>
  checkPermissions?: (args, context) => Promise<PermissionResult>
  isConcurrencySafe?: (args) => boolean
  isReadOnly: boolean  // used to decide if parallel execution is safe
  requiresPrerequisites?: string[]  // ["get_team_workload"] for create_task
  
  // Observability
  onProgress?: (update: string) => void
}
```

This matters because right now your executor (`action-executor.ts`) is a separate 300-line switch statement with no connection to the tool definitions. The tool schema says `create_task` needs `assigned_to_name`, but the executor has its own resolution logic that doesn't share any code with the schema. If you add a parameter to the schema, you have to remember to handle it in the executor too — no compiler check enforces this.

---

### IMPROVEMENT 3: The System Prompt Is Defensive Documentation for LLM Failures

**File:** `app/api/ai/command/route.ts` — system prompt block

The system prompt contains:
```
## CRITICAL: Tool Parameter Names
When calling create_task, you MUST use EXACT parameter names:
- Task title → use "title" (NEVER "task_name", "name", "task_title")  
- Description/context → use "notes" (NEVER "description", "details", "content")
...
Wrong parameter names cause the tool call to fail entirely.
```

This section exists because the model uses wrong parameter names. But this is a signal that your tool descriptions are ambiguous, not a problem you should paper over with system prompt warnings. When the parameter is called `notes` but described as "Additional context or description", the LLM correctly infers it could be called `description` — because that's what it semantically is.

**Root cause fixes:**
1. Rename `notes` to `description` in the schema — use the name users and the LLM expect
2. Add `"NEVER use other names"` to the individual parameter descriptions, not the system prompt
3. Use constrained enums for everything that has a fixed set of values (you already do this well for `priority` and `status`)

The system prompt should describe *what the AI is* and *how it thinks*, not compensate for tool schema design flaws.

---

### IMPROVEMENT 4: History Capping Is Blunt — Missing Conversation Awareness

**File:** `app/api/ai/command/route.ts`

```typescript
const cappedHistory = history.slice(-6)  // last 6 messages only
```

6 messages is arbitrary. A conversation might have 6 messages of chitchat (0 context value) or 3 messages that established critical context ("assign all new tasks to Sarah since she's free this week"). Hard slicing at 6 loses the latter.

**Better approach — keep messages that contain tool results or action confirmations:**
```typescript
function selectRelevantHistory(history: Message[], maxMessages = 10): Message[] {
  // Always keep messages with tool call results (they contain resolved IDs)
  const important = history.filter(m => 
    m.role === 'tool' ||
    (m.role === 'assistant' && m.content?.includes('action_executed'))
  )
  
  // Fill remaining slots with most recent messages
  const recent = history.slice(-maxMessages)
  const combined = [...new Set([...important, ...recent])]
  return combined.slice(-maxMessages)
}
```

---

### IMPROVEMENT 5: No Observability Into What the AI Actually Did

**File:** Both route files

You log tool names: `console.log("[AI-CMD] Tools used: ${toolsCalled.join(", ")}")`. But you have no way to answer:
- Why did the AI call `get_team_workload` when I only asked about tasks?
- How many tokens did tool results consume vs the final response?
- Which tool calls took the longest?
- What was the AI's actual reasoning before calling each tool?

**Minimum viable observability:**

```typescript
interface ToolCallRecord {
  step: number
  toolName: string
  args: Record<string, any>
  durationMs: number
  outputTokens: number
  success: boolean
  error?: string
}

// Accumulated during the loop
const toolCallLog: ToolCallRecord[] = []

// After loop completes, save to DB or emit as SSE event
controller.enqueue(encoder.encode(
  `data: ${JSON.stringify({ type: "debug_trace", tools: toolCallLog })}\n\n`
))
```

This lets you build a debug view in your UI showing exactly what the AI did step by step — invaluable for diagnosing why a task got created with the wrong assignee.

---

### IMPROVEMENT 6: `buildMiniContext` Doesn't Include Names — Forcing Extra Read Tool Calls

**File:** `lib/ai/mini-context.ts`

Current mini-context output looks like:
```
Founder: John Smith | Wednesday, April 1, 2026 2:30 PM
Tasks: 12 active, 3 overdue, 1 blocked, 2 due today
Projects: 4 active / 6 total
Team: 3 members
CRM: 8 active deals | $45,000 pipeline
Calendar: 2 events this week
```

This is good for awareness, but it means for EVERY task creation, the AI must call `get_team_workload` to learn who's on the team before it can assign anyone. That's an extra round-trip (one extra API call, one extra DB query, ~300-500ms latency) for the most common operation.

**Simple fix — add name lists to mini-context:**
```typescript
// Add to buildMiniContext
const { data: team } = await supabaseAdmin
  .from("team_members")
  .select("user_id, position, profile:profiles!team_members_user_id_profiles_fkey(full_name)")
  .eq("founder_id", founderId)
  .eq("is_active", true)

// Include in output
if (team?.length) {
  parts.push(`Team members: ${team.map(m => `${m.profile?.full_name} (${m.position})`).join(', ')}`)
}

// Same for projects
const { data: projects } = await supabaseAdmin
  .from("projects")
  .select("name, status")
  .eq("founder_id", founderId)
  .eq("status", "active")

if (projects?.length) {
  parts.push(`Active projects: ${projects.map(p => p.name).join(', ')}`)
}
```

This eliminates `get_team_workload` and `get_projects` calls for simple task assignment — saving a full loop step for the most common AI request.

---

### IMPROVEMENT 7: Action Results Don't Flow Back Into the Conversation Meaningfully

**File:** `lib/ai/action-executor.ts` — return format

Your action results look like:
```typescript
return {
  success: true,
  message: "Task created successfully.",
  data: { task_id: "uuid", title: "...", assigned_to: "Sarah", ... }
}
```

But what gets pushed into the conversation messages is:
```typescript
toolResults.push({
  tool_call_id: toolCall.id,
  role: "tool",
  content: JSON.stringify(result),  // The whole object as a string
})
```

The AI then sees a JSON blob as a tool result and has to parse it mentally. This works but it's not optimal. Claude Code's approach: tool results should be **human-readable summaries** that the AI can reference naturally.

**Better tool result format:**
```typescript
// Instead of JSON blob, return a natural language summary
const toolResultContent = result.success
  ? `Task created: "${args.title}" assigned to ${assigneeName || 'unassigned'}, due ${args.due_date ? new Date(args.due_date).toLocaleDateString() : 'no date'}, linked to ${projectName || 'no project'}.`
  : `Failed to create task: ${result.message}`

toolResults.push({
  tool_call_id: toolCall.id,
  role: "tool",
  content: toolResultContent,  // Human-readable, not JSON
})
```

The AI produces more natural confirmations ("Created the task as requested — assigned to Sarah, due Friday") when the tool result reads like natural language rather than a JSON object.

---

### IMPROVEMENT 8: No Permission Layer — AI Can Take Any Action With No Audit Trail

**File:** `app/api/ai/command/route.ts`, `lib/ai/action-executor.ts`

Beyond the `delete_task` issue in Critical 3, there's a broader permission problem: there's no system-level control over what the AI is allowed to do for which users. A team member who has `can_create_tasks: false` could theoretically send requests to the `/api/ai/command` endpoint and the AI would create tasks for them — because the executor only checks `founder_id`, not the calling user's permissions.

```typescript
// CURRENT in executeCreateTask
const { data, error } = await supabaseAdmin
  .from("tasks")
  .insert({ user_id: ctx.founder_id, ... })  // Uses founder's ID
  // No check: can ctx.user_id actually create tasks?
```

**Fix — check permissions before executing any action:**
```typescript
// In executeAction, before the switch statement
async function checkUserPermissions(
  toolName: AIToolName,
  userId: string,
  founderId: string
): Promise<{ allowed: boolean; reason?: string }> {
  if (userId === founderId) return { allowed: true }  // Founder can do anything
  
  const { data: member } = await supabaseAdmin
    .from("team_members")
    .select("can_create_tasks, can_view_projects, is_active")
    .eq("user_id", userId)
    .eq("founder_id", founderId)
    .single()
  
  if (!member?.is_active) return { allowed: false, reason: "Account inactive" }
  
  const permissionMap: Record<AIToolName, keyof typeof member> = {
    create_task: "can_create_tasks",
    create_project: "can_create_projects",
    // ...
  }
  
  const requiredPerm = permissionMap[toolName]
  if (requiredPerm && !member[requiredPerm]) {
    return { allowed: false, reason: `You don't have permission to ${toolName.replace('_', ' ')}` }
  }
  
  return { allowed: true }
}
```

---

## Summary Table

| Issue | File(s) | Severity | Effort to Fix |
|---|---|---|---|
| `command-context.ts` dead code | `lib/ai/command-context.ts` | High | Delete the file |
| Streaming breaks after tool calls | `app/api/ai/command/route.ts` | High | Simulate streaming |
| LLM controls delete safety | `lib/ai/tools.ts`, `action-executor.ts` | High | Remove param from schema |
| Mixed batch detection complexity | `app/api/ai/command/route.ts` | High | Replace with system prompt constraint |
| Auto-fetch fallbacks in executor | `lib/ai/action-executor.ts` | High | Remove, return clear error |
| Duplicated route logic | Both route files | Medium | Extract `runAgentLoop` |
| Tools are untyped JSON blobs | `lib/ai/tools.ts` | Medium | Add Tool interface contract |
| System prompt compensates for bad schemas | `app/api/ai/command/route.ts` | Medium | Fix schemas, remove warnings |
| Blunt history capping | `app/api/ai/command/route.ts` | Medium | Semantic history selection |
| No observability | Both route files | Medium | Add `ToolCallRecord` logging |
| Mini-context missing names | `lib/ai/mini-context.ts` | Low-Medium | Add team/project name lists |
| Tool results as JSON blobs | `lib/ai/action-executor.ts` | Low | Return natural language |
| No permission check in executor | `lib/ai/action-executor.ts` | Medium | Add permission gate |

---

## The Core Architectural Insight You're Missing

Claude Code's agentic loop works because of one principle: **the system controls safety; the LLM controls strategy**.

Your system currently asks the LLM to:
- Control whether deletion needs confirmation (`needs_confirmation` param)
- Control the sequencing of read-before-write (deferred via mixed batch detection)
- Control parameter naming (warned against via system prompt)

All three of these should be system-enforced. The LLM should only decide *what to do*, not *how safely to do it*. When you move safety constraints from the system prompt into the executor, your system becomes dramatically more reliable — not because the LLM gets smarter, but because it can't accidentally break things even when it doesn't follow instructions perfectly.

The `tool_use_failed` 400 errors, the mixed batch deferral, and the `needs_confirmation` parameter problems are all symptoms of the same disease: trusting the LLM to follow rules it could easily break. Fix the underlying contracts and the bandaids become unnecessary.
