# OpenCode Event Types Reference

This document lists all SSE event types in OpenCode, their purposes, when they fire, and their payload structures.

---

## Table of Contents

1. [Event System Overview](#event-system-overview)
2. [Complete Event List](#complete-event-list)
3. [Session Events](#session-events)
4. [Message Events](#message-events)
5. [Permission Events](#permission-events)
6. [File Events](#file-events)
7. [System Events](#system-events)
8. [Event Payload Structures](#event-payload-structures)
9. [Event Flow Examples](#event-flow-examples)

---

## Event System Overview

### How Events Work

OpenCode uses an **Event Bus** pattern for real-time communication:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Server Code    │ ──→ │    Event Bus     │ ──→ │   SSE Clients    │
│  (TypeScript)    │     │  Bus.publish()   │     │  (TUI, IDE, etc) │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

**Source Files:**
- Event Bus: `packages/opencode/src/bus/index.ts`
- Event Definitions: `packages/opencode/src/bus/event.ts`
- SSE Endpoint: `packages/opencode/src/server/server.ts:1519`
- Go SDK Types: `packages/sdk/go/event.go`

### Event Format

Events are sent as JSON via Server-Sent Events:

```
data: {"type":"<event-type>","properties":{<event-data>}}

```

---

## Complete Event List

| # | Event Type | Category | Description |
|---|-----------|----------|-------------|
| 1 | `server.connected` | System | SSE connection established |
| 2 | `session.created` | Session | New session created |
| 3 | `session.updated` | Session | Session metadata changed |
| 4 | `session.deleted` | Session | Session removed |
| 5 | `session.idle` | Session | Session finished processing |
| 6 | `session.error` | Session | Error occurred in session |
| 7 | `session.compacted` | Session | Session was summarized |
| 8 | `message.updated` | Message | Message added or updated |
| 9 | `message.removed` | Message | Message deleted |
| 10 | `message.part.updated` | Message | Message part added/updated |
| 11 | `message.part.removed` | Message | Message part deleted |
| 12 | `permission.updated` | Permission | Permission request created/updated |
| 13 | `permission.replied` | Permission | User responded to permission |
| 14 | `file.edited` | File | File modified by agent |
| 15 | `file.watcher.updated` | File | External file change detected |
| 16 | `lsp.client.diagnostics` | System | Language Server diagnostics |
| 17 | `todo.updated` | System | Todo list changed |
| 18 | `installation.updated` | System | OpenCode CLI updated |
| 19 | `ide.installed` | System | IDE extension installed |

---

## Session Events

### `server.connected`

**When:** Immediately when SSE connection is established

**Purpose:** Confirms the SSE stream is working (heartbeat)

**Source:** `packages/opencode/src/server/server.ts:1533`

**Payload:**
```json
{
  "type": "server.connected",
  "properties": {}
}
```

**Usage:** Client can verify connection is alive. Not typically handled in UI.

---

### `session.created`

**When:** New session is created via `Session.New()`

**Purpose:** Notify clients that a new session exists

**Source:** `packages/opencode/src/session/index.ts:87`

**Payload:**
```json
{
  "type": "session.created",
  "properties": {
    "sessionID": "ses_01HXYZ..."
  }
}
```

**Usage:** Session list can update to show new session

---

### `session.updated`

**When:** Session metadata changes (title, share status, model, etc.)

**Purpose:** Sync session info across clients

**Source:** `packages/opencode/src/session/index.ts:93`

**Payload:**
```json
{
  "type": "session.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "session": {
      "id": "ses_01HXYZ...",
      "title": "New Title",
      "share": "share_abc123",
      "model": "claude-3-opus",
      "createdAt": 1699000000000,
      "updatedAt": 1699000001000
    }
  }
}
```

**Usage:** TUI updates session info display

---

### `session.deleted`

**When:** Session is deleted via `Session.Delete()`

**Purpose:** Remove session from all client UIs

**Source:** `packages/opencode/src/session/index.ts:99`

**Payload:**
```json
{
  "type": "session.deleted",
  "properties": {
    "sessionID": "ses_01HXYZ..."
  }
}
```

**Usage:** TUI clears session if it was the active one

---

### `session.idle`

**When:** Session finishes processing (AI response complete, tools done)

**Purpose:** Signal that session is no longer busy

**Source:** `packages/opencode/src/session/prompt.ts` (after processing completes)

**Payload:**
```json
{
  "type": "session.idle",
  "properties": {
    "sessionID": "ses_01HXYZ..."
  }
}
```

**Usage:**
- External clients know when to fetch final messages
- Can be used to hide "processing" indicator
- **Critical for SDK integrations** - this tells you the AI is done!

---

### `session.error`

**When:** An error occurs during session processing

**Purpose:** Display error to user

**Source:** `packages/opencode/src/session/index.ts:105`

**Payload:**
```json
{
  "type": "session.error",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "error": {
      "name": "ProviderAuthError",
      "message": "Invalid API key",
      "providerID": "anthropic"
    }
  }
}
```

**Error Types:**
- `ProviderAuthError` - Invalid API key
- `MessageOutputLengthError` - Response too long
- `MessageAbortedError` - User cancelled
- `UnknownError` - General error

**Usage:** TUI shows error toast

---

### `session.compacted`

**When:** Session is summarized to reduce token usage

**Purpose:** Notify that session history was compressed

**Source:** `packages/opencode/src/session/compaction.ts`

**Payload:**
```json
{
  "type": "session.compacted",
  "properties": {
    "sessionID": "ses_01HXYZ..."
  }
}
```

**Usage:** TUI shows "Session compacted" toast

---

## Message Events

### `message.updated`

**When:**
- New message created (user sends prompt)
- Message metadata changes
- AI starts responding (assistant message created)
- AI finishes responding (tokens counted)

**Purpose:** Add or update message in UI

**Source:** `packages/opencode/src/session/message-v2.ts`

**Payload:**
```json
{
  "type": "message.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "messageID": "msg_01ABC...",
    "message": {
      "id": "msg_01ABC...",
      "role": "assistant",
      "sessionID": "ses_01HXYZ...",
      "time": {
        "created": 1699000000000,
        "completed": 1699000005000
      },
      "modelID": "claude-3-opus",
      "providerID": "anthropic",
      "cost": 0.015,
      "tokens": {
        "input": 1500,
        "output": 500,
        "reasoning": 0,
        "cache": { "read": 1000, "write": 500 }
      }
    }
  }
}
```

**Usage:** TUI adds new message or updates existing message metadata

---

### `message.removed`

**When:** Message is deleted (e.g., during revert)

**Purpose:** Remove message from UI

**Source:** `packages/opencode/src/session/message-v2.ts`

**Payload:**
```json
{
  "type": "message.removed",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "messageID": "msg_01ABC..."
  }
}
```

**Usage:** TUI removes message from list

---

### `message.part.updated`

**When:**
- Text starts streaming from AI
- Each text chunk arrives
- Tool execution starts/updates/completes
- File attachment processed

**Purpose:** Real-time streaming of message content

**Source:** `packages/opencode/src/session/message-v2.ts`

**Payload (Text Part):**
```json
{
  "type": "message.part.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "messageID": "msg_01ABC...",
    "partID": "part_01DEF...",
    "part": {
      "id": "part_01DEF...",
      "type": "text",
      "text": "Hello! I can help you with",
      "time": {
        "start": 1699000001000,
        "end": 1699000002000
      }
    }
  }
}
```

**Payload (Tool Part):**
```json
{
  "type": "message.part.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "messageID": "msg_01ABC...",
    "partID": "part_01GHI...",
    "part": {
      "id": "part_01GHI...",
      "type": "tool",
      "tool": "Bash",
      "callID": "call_123",
      "state": {
        "status": "completed",
        "input": { "command": "ls -la" },
        "output": "total 48\ndrwxr-xr-x ...",
        "title": "List directory",
        "time": {
          "start": 1699000001000,
          "end": 1699000003000
        }
      }
    }
  }
}
```

**Part Types:**
- `text` - AI's text response
- `reasoning` - AI's reasoning/thinking
- `tool` - Tool execution (Bash, Edit, Read, Write, etc.)
- `file` - File attachment
- `step-start` - Agent step started
- `step-finish` - Agent step completed

**Usage:** TUI appends text chunks for streaming display, shows tool execution

---

### `message.part.removed`

**When:** Message part is deleted (during revert or edit)

**Purpose:** Remove specific part from message

**Source:** `packages/opencode/src/session/message-v2.ts`

**Payload:**
```json
{
  "type": "message.part.removed",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "messageID": "msg_01ABC...",
    "partID": "part_01DEF..."
  }
}
```

**Usage:** TUI removes part from message display

---

## Permission Events

### `permission.updated`

**When:** Agent needs user permission for an action

**Purpose:** Show permission dialog to user

**Source:** `packages/opencode/src/permission/index.ts`

**Payload:**
```json
{
  "type": "permission.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "permission": {
      "id": "perm_01XYZ...",
      "sessionID": "ses_01HXYZ...",
      "time": {
        "created": 1699000000000
      },
      "request": {
        "type": "file-write",
        "tool": "Write",
        "path": "/path/to/file.ts",
        "content": "new file content..."
      }
    }
  }
}
```

**Permission Types:**
- `file-write` - Write/create file
- `file-delete` - Delete file
- `bash-execute` - Run shell command
- `mcp-tool` - Use MCP tool

**Usage:** TUI shows permission dialog, blocks editor

---

### `permission.replied`

**When:** User responds to permission request (allow/deny)

**Purpose:** Dismiss permission dialog

**Source:** `packages/opencode/src/permission/index.ts`

**Payload:**
```json
{
  "type": "permission.replied",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "permissionID": "perm_01XYZ...",
    "response": "allow"
  }
}
```

**Response Values:** `"allow"`, `"deny"`, `"allowAll"`, `"denyAll"`

**Usage:** TUI removes permission dialog

---

## File Events

### `file.edited`

**When:** Agent modifies a file via Edit or Write tool

**Purpose:** Notify IDEs/editors to refresh

**Source:** `packages/opencode/src/file/index.ts`

**Payload:**
```json
{
  "type": "file.edited",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "path": "/absolute/path/to/file.ts",
    "content": "new file content..."
  }
}
```

**Usage:** IDE extensions refresh the file buffer

---

### `file.watcher.updated`

**When:** External file system change detected

**Purpose:** Notify about changes made outside OpenCode

**Source:** `packages/opencode/src/file/watcher.ts`

**Payload:**
```json
{
  "type": "file.watcher.updated",
  "properties": {
    "path": "/path/to/changed/file.ts",
    "event": "change"
  }
}
```

**Event Types:** `"create"`, `"change"`, `"delete"`

**Usage:** Background tracking, not typically displayed in TUI

---

## System Events

### `lsp.client.diagnostics`

**When:** Language Server reports errors/warnings

**Purpose:** Show code diagnostics

**Source:** `packages/opencode/src/lsp/client.ts`

**Payload:**
```json
{
  "type": "lsp.client.diagnostics",
  "properties": {
    "uri": "file:///path/to/file.ts",
    "diagnostics": [
      {
        "range": {
          "start": { "line": 10, "character": 5 },
          "end": { "line": 10, "character": 15 }
        },
        "message": "Property 'foo' does not exist",
        "severity": 1
      }
    ]
  }
}
```

**Severity:** 1=Error, 2=Warning, 3=Info, 4=Hint

**Usage:** IDE extensions show diagnostics inline

---

### `todo.updated`

**When:** AI creates or updates todo items

**Purpose:** Sync todo list across clients

**Source:** `packages/opencode/src/session/todo.ts`

**Payload:**
```json
{
  "type": "todo.updated",
  "properties": {
    "sessionID": "ses_01HXYZ...",
    "todos": [
      {
        "id": "todo_01ABC...",
        "content": "Implement login feature",
        "status": "in_progress"
      }
    ]
  }
}
```

**Status Values:** `"pending"`, `"in_progress"`, `"completed"`

**Usage:** Display todo progress in UI

---

### `installation.updated`

**When:** OpenCode CLI has been upgraded

**Purpose:** Prompt user to restart

**Source:** `packages/opencode/src/installation/index.ts`

**Payload:**
```json
{
  "type": "installation.updated",
  "properties": {
    "version": "0.15.32"
  }
}
```

**Usage:** TUI shows "Updated to X, restart to apply" toast

---

### `ide.installed`

**When:** IDE extension is installed

**Purpose:** Track extension installation

**Source:** `packages/opencode/src/ide/index.ts`

**Payload:**
```json
{
  "type": "ide.installed",
  "properties": {
    "ide": "vscode"
  }
}
```

**IDE Values:** `"vscode"`, `"cursor"`, `"windsurf"`

**Usage:** One-time notification, not typically displayed

---

## Event Payload Structures

### Common Properties

All events include:
```typescript
{
  type: string;           // Event type name
  properties: {
    sessionID?: string;   // Session this event relates to
    messageID?: string;   // Message this event relates to
    // ... event-specific data
  }
}
```

### TypeScript Types

```typescript
// From packages/opencode/src/bus/event.ts
export namespace Event {
  export const Session = {
    Created: "session.created",
    Updated: "session.updated",
    Deleted: "session.deleted",
    Idle: "session.idle",
    Error: "session.error",
    Compacted: "session.compacted",
  }

  export const Message = {
    Updated: "message.updated",
    Removed: "message.removed",
    PartUpdated: "message.part.updated",
    PartRemoved: "message.part.removed",
  }

  export const Permission = {
    Updated: "permission.updated",
    Replied: "permission.replied",
  }

  // ... etc
}
```

### Go SDK Types

```go
// From packages/sdk/go/event.go
const (
    EventListResponseTypeMessageUpdated         EventListResponseType = "message.updated"
    EventListResponseTypeMessageRemoved         EventListResponseType = "message.removed"
    EventListResponseTypeMessagePartUpdated     EventListResponseType = "message.part.updated"
    EventListResponseTypeMessagePartRemoved     EventListResponseType = "message.part.removed"
    EventListResponseTypeSessionCompacted       EventListResponseType = "session.compacted"
    EventListResponseTypePermissionUpdated      EventListResponseType = "permission.updated"
    EventListResponseTypePermissionReplied      EventListResponseType = "permission.replied"
    EventListResponseTypeFileEdited             EventListResponseType = "file.edited"
    EventListResponseTypeLspClientDiagnostics   EventListResponseType = "lsp.client.diagnostics"
    EventListResponseTypeInstallationUpdated    EventListResponseType = "installation.updated"
)
```

---

## Event Flow Examples

### Example 1: User Sends Prompt

```
User types: "Hello, explain Go interfaces"

Timeline:
─────────────────────────────────────────────────────────
T+0ms    HTTP POST /session/:id/prompt
         ↓
T+5ms    message.updated (user message created)
         {sessionID, messageID: "msg_user", role: "user"}
         ↓
T+10ms   message.part.updated (user text)
         {messageID: "msg_user", part: {type: "text", text: "Hello..."}}
         ↓
T+15ms   message.updated (assistant message created)
         {sessionID, messageID: "msg_asst", role: "assistant"}
         ↓
T+500ms  message.part.updated (AI starts streaming)
         {messageID: "msg_asst", part: {type: "text", text: "Go inter"}}
         ↓
T+600ms  message.part.updated (more text)
         {messageID: "msg_asst", part: {type: "text", text: "Go interfaces are"}}
         ↓
T+1500ms message.part.updated (complete)
         {messageID: "msg_asst", part: {type: "text", text: "Go interfaces are..."}}
         ↓
T+1550ms message.updated (tokens counted)
         {messageID: "msg_asst", tokens: {input: 10, output: 150}}
         ↓
T+1600ms session.idle
         {sessionID}
```

### Example 2: Agent Uses Tool

```
User: "Create a file called hello.go"

Timeline:
─────────────────────────────────────────────────────────
T+0ms    message.updated (user message)
T+10ms   message.part.updated (user text)
T+20ms   message.updated (assistant message)
         ↓
T+500ms  message.part.updated (AI thinking)
         {part: {type: "text", text: "I'll create..."}}
         ↓
T+600ms  message.part.updated (tool started)
         {part: {type: "tool", tool: "Write", state: {status: "pending"}}}
         ↓
T+650ms  permission.updated (needs approval)
         {permission: {type: "file-write", path: "hello.go"}}

         [User clicks "Allow"]

T+2000ms permission.replied
         {permissionID, response: "allow"}
         ↓
T+2100ms message.part.updated (tool running)
         {part: {type: "tool", state: {status: "running"}}}
         ↓
T+2200ms file.edited
         {path: "hello.go", content: "package main..."}
         ↓
T+2250ms message.part.updated (tool completed)
         {part: {type: "tool", state: {status: "completed", output: "Created"}}}
         ↓
T+2300ms message.part.updated (AI response)
         {part: {type: "text", text: "I've created hello.go..."}}
         ↓
T+2400ms session.idle
```

### Example 3: Error Handling

```
User: "Use GPT-4" (invalid API key)

Timeline:
─────────────────────────────────────────────────────────
T+0ms    message.updated (user message)
T+10ms   message.part.updated (user text)
T+20ms   message.updated (assistant message created)
         ↓
T+500ms  session.error
         {error: {name: "ProviderAuthError", message: "Invalid API key"}}
         ↓
T+550ms  session.idle
```

---

## Summary

### Event Categories

| Category | Events | Purpose |
|----------|--------|---------|
| **Session** | 7 events | Session lifecycle management |
| **Message** | 4 events | Message CRUD and streaming |
| **Permission** | 2 events | User approval workflow |
| **File** | 2 events | File change tracking |
| **System** | 4 events | Diagnostics, updates, todos |

### Most Important Events for SDK Integration

1. **`session.idle`** - Know when AI is done responding
2. **`message.updated`** - Track message creation
3. **`message.part.updated`** - Stream text in real-time
4. **`session.error`** - Handle errors gracefully
5. **`permission.updated`** - Handle permission requests

### File Locations

| Component | File |
|-----------|------|
| Event Definitions | `packages/opencode/src/bus/event.ts` |
| Event Bus | `packages/opencode/src/bus/index.ts` |
| SSE Endpoint | `packages/opencode/src/server/server.ts:1519` |
| Go SDK Events | `packages/sdk/go/event.go` |
| TUI Event Handlers | `packages/tui/internal/tui/tui.go:464` |
