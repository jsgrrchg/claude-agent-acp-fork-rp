# Plan: Collapsible Subagent Tool Calls in Zed Agent Panel

## Context

When Claude Code spawns subagents (via the Agent/Task tool), all their individual tool calls (Read, Edit, Bash, Grep, etc.) appear as flat sequential entries in the parent thread. With multiple subagents, this creates significant clutter — dozens of tool call cards that obscure the main conversation flow.

Zed already has a **collapsible subagent card UI** used by its native `SpawnAgent` tool. This plan describes how to make Claude Code's subagents use that same UI, so each subagent appears as a single collapsed card showing title + status, with optional expand to see details.

## How Zed's Native Subagent Card Works

### Rendering flow (Zed side — `crates/agent_ui/src/connection_view/thread_view.rs`)

1. `render_any_tool_call()` (line 5213) checks `tool_call.is_subagent()`.
2. If true, delegates to `render_subagent_tool_call()` (line 6552) → `render_subagent_card()` (line 6582).
3. The card shows: spinner/check icon, title, file change count, expand/collapse chevron.
4. Expanded content renders up to 8 recent entries from the subagent's `ThreadView` (line 6908, `render_subagent_expanded_content`).

### `is_subagent()` detection (`crates/acp_thread/src/acp_thread.rs`, line 411):
```rust
pub fn is_subagent(&self) -> bool {
    self.tool_name.as_ref().is_some_and(|s| s == "spawn_agent")
        || self.subagent_session_info.is_some()
}
```

Two triggers:
- `tool_name == "spawn_agent"` (Zed's native tool, read from `_meta.tool_name`)
- `subagent_session_info` is present (read from `_meta.subagent_session_info`)

### `SubagentSessionInfo` structure (`crates/acp_thread/src/acp_thread.rs`, line 57):
```rust
pub struct SubagentSessionInfo {
    pub session_id: acp::SessionId,       // Virtual session ID for the subagent
    pub message_start_index: usize,       // Start of subagent's entries
    pub message_end_index: Option<usize>, // End of subagent's entries (None = still running)
}
```

Read from `_meta.subagent_session_info` on tool_call / tool_call_update notifications.

### Subagent session loading (`crates/agent_ui/src/connection_view.rs`, line 1665):

When `AcpThreadEvent::SubagentSpawned(session_id)` fires, Zed calls `load_subagent_session()`, which calls the ACP agent's `loadSession(session_id)`. This creates a separate `AcpThread` + `ThreadView` for the subagent, enabling the expanded content view.

### Current gap: SubagentSpawned not emitted for external ACP agents

The `SubagentSpawned` event is only emitted by the native Thread struct (for Zed's built-in agent). When an external ACP agent (like Claude Code) sends a `tool_call` with `subagent_session_info` meta, the `AcpThread.upsert_tool_call_inner()` stores it but does NOT emit `SubagentSpawned`. This means `loadSession` is never called during live streaming — only during session restore.

**Impact**: During live streaming, the subagent card appears (title + status) but cannot be expanded. After reopening the session, expanded view works.

## Implementation Plan

### Part 1: ACP Extension Changes (`src/acp-agent.ts`, `src/tools.ts`)

These changes make Claude Code's subagents render as collapsible cards immediately.

#### 1.1 Track subagent sessions

In `acp-agent.ts`, add a `subagentSessions` map to the session object:

```typescript
// In session type
subagentSessions?: Map<string, {  // parentToolUseId → info
  virtualSessionId: string;
  toolCallId: string;             // The Agent/Task tool call ID in parent session
  notifications: SessionNotification[];  // Buffered notifications
  messageIndex: number;           // Running count of entries
}>;
```

#### 1.2 Detect new subagents

In `streamEventToAcpNotifications()` and the `prompt()` message loop, when `message.parent_tool_use_id` is non-null and not yet tracked:

1. Generate a virtual session ID: `subagent-${uuid()}`
2. Find the parent Agent/Task tool call ID (from `toolUseCache` — the tool_use that spawned this subagent, which has `parent_tool_use_id === null` and tool name "Agent" or "Task")
3. Store in `subagentSessions` map

#### 1.3 Set `subagent_session_info` on Agent/Task tool calls

When emitting `tool_call` or `tool_call_update` for Agent/Task tools, include:

```typescript
_meta: {
  tool_name: "Agent",  // Top-level key for Zed's tool_name_from_meta()
  subagent_session_info: {
    session_id: virtualSessionId,
    message_start_index: 0,
  },
  claudeCode: { toolName: "Agent" }
}
```

When the subagent completes (tool result arrives), update with `message_end_index`.

#### 1.4 Route subagent notifications to virtual session

Instead of emitting subagent tool calls/messages to the parent session:

1. **Suppress** notifications where `parentToolUseId` is non-null from the parent session's output
2. **Buffer** them in `subagentSessions[parentToolUseId].notifications`
3. **If a virtual session is active** (Zed called `loadSession`), forward directly via `client.sessionUpdate()` with the virtual session ID

#### 1.5 Handle `loadSession` for virtual sessions

In `loadSession()`, check if the requested `sessionId` matches a virtual subagent session:

```typescript
async loadSession(params: LoadSessionRequest): Promise<LoadSessionResponse> {
  // Check if this is a virtual subagent session
  const subagentInfo = this.findSubagentSession(params.sessionId);
  if (subagentInfo) {
    // Replay buffered notifications
    for (const notification of subagentInfo.notifications) {
      await this.client.sessionUpdate({
        ...notification,
        sessionId: params.sessionId,
      });
    }
    return { modes: [], models: [], configOptions: [] };
  }

  // Normal session loading...
  return this.createSession(...);
}
```

#### 1.6 Set `_meta.tool_name` on all tool calls

Currently the extension sets `_meta.claudeCode.toolName` but Zed reads `_meta.tool_name` (top-level key). Add `tool_name` to the top-level `_meta`:

```typescript
_meta: {
  tool_name: chunk.name,      // For Zed's tool_name_from_meta()
  claudeCode: {
    toolName: chunk.name,     // Keep existing for our own use
  }
}
```

This is a separate improvement — not required for subagent collapse but makes tool names available to Zed.

### Part 2: Zed-Side Change (upstream PR to `zed-industries/zed`)

This enables expanded subagent content during live streaming.

#### 2.1 Emit `SubagentSpawned` from `upsert_tool_call_inner`

In `crates/acp_thread/src/acp_thread.rs`, after creating/updating a tool call, check if `subagent_session_info` was newly set:

```rust
// In upsert_tool_call_inner(), after the upsert logic:
if let AgentThreadEntry::ToolCall(call) = &self.entries[ix_or_new] {
    if let Some(info) = &call.subagent_session_info {
        // Check if this is a newly discovered subagent
        if !self.known_subagent_sessions.contains(&info.session_id) {
            self.known_subagent_sessions.insert(info.session_id.clone());
            cx.emit(AcpThreadEvent::SubagentSpawned(info.session_id.clone()));
        }
    }
}
```

Add `known_subagent_sessions: HashSet<acp::SessionId>` to `AcpThread`.

This is a small, surgical change that makes external ACP agents' subagents work the same as native agents.

### Files to Modify

| File | Changes |
|------|---------|
| `src/acp-agent.ts` | Subagent tracking, notification routing, virtual session handling, `loadSession` for virtual sessions, `_meta.tool_name` |
| `src/tools.ts` | Add `tool_name` to top-level `_meta` in `toAcpNotifications` |
| `src/lib.ts` | Export any new types if needed |

### Files NOT to Modify

The Zed-side change (Part 2) would be a separate upstream PR to `zed-industries/zed`.

## Behavior Summary

### Before (current)
```
[User Message]
[Agent: "Research the codebase"]  ← Agent tool call, kind: "think"
[Read src/foo.ts]                  ← Subagent tool call, flat
[Grep "pattern"]                   ← Subagent tool call, flat
[Read src/bar.ts]                  ← Subagent tool call, flat
[Edit src/foo.ts]                  ← Subagent tool call, flat
[Bash: npm test]                   ← Subagent tool call, flat
[Agent: "Fix the bug"]            ← Another agent tool call
[Read src/baz.ts]                  ← More flat subagent calls...
[Edit src/baz.ts]
[Bash: npm test]
[Assistant response]
```

### After (with this change)
```
[User Message]
[▶ Research the codebase ✓ — 3 files changed +12 -4]  ← Collapsed subagent card
[▶ Fix the bug ✓ — 1 file changed +5 -2]              ← Collapsed subagent card
[Assistant response]
```

Click to expand shows last 8 entries from the subagent's thread (with a gradient overlay and "Make Full Screen" button — Zed's existing UI).

## Edge Cases

1. **Subagent with no tool calls** (just text output): Card still shows with title and status; no expanded content.
2. **Multiple concurrent subagents**: Each gets its own virtual session ID and card.
3. **Nested subagents** (subagent spawns subagent): The inner subagent's notifications are routed to the outer subagent's virtual session. The inner subagent appears as a nested card within the outer's expanded view.
4. **File edit interceptor**: The `fileEditInterceptor` is wired to the main session. Subagent Edit/Write operations are intercepted by the PostToolUse hook in the main session (since all SDK calls go through the same session). Virtual sessions just receive the resulting notifications.
5. **Session restore**: `loadSession` replays buffered notifications, so expanded view works after reopening.

## Verification

1. `npm run build` — TypeScript compilation
2. `npm run test:run` — Unit tests
3. Manual testing in Zed:
   - Trigger a Claude Code session with subagents (e.g., a task that spawns Agent tools)
   - Verify subagent tool calls appear as collapsed cards, not flat entries
   - Verify the card shows title, status spinner (while running), and completion state
   - After Part 2 (Zed PR): verify expand shows subagent entries
   - Verify file edit interceptor still works for subagent edits (Review Changes UI)
   - Verify session restore shows the cards correctly
