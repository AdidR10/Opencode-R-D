# OpenCode TUI Architecture: How It Works with SSE Events and Go SDK

This document explains how the OpenCode TUI (`packages/tui/`) communicates with the OpenCode server using Server-Sent Events (SSE) and the Go SDK.

---

## Table of Contents

1. [Overview](#overview)
2. [Communication Architecture](#communication-architecture)
3. [TUI Initialization Flow](#tui-initialization-flow)
4. [SSE Event Streaming](#sse-event-streaming)
5. [Event Handlers in TUI](#event-handlers-in-tui)
6. [HTTP API Calls (Beyond SSE)](#http-api-calls-beyond-sse)
7. [Control API (Bidirectional)](#control-api-bidirectional)
8. [Complete Event Flow Diagram](#complete-event-flow-diagram)
9. [Key Architectural Insights](#key-architectural-insights)

---

## Overview

**Does the TUI fully depend on SSE events?**

**No.** The TUI uses **two primary communication channels**:

| Channel | Direction | Purpose |
|---------|-----------|---------|
| **SSE Events** | Server → TUI | Real-time push updates (messages, permissions, errors) |
| **HTTP REST API** | TUI → Server | Commands, queries, and actions (send prompt, create session, etc.) |

Additionally, there's a **Control API** for IDE integration that provides bidirectional communication.

---

## Communication Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode TUI (Go)                        │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Bubble Tea   │  │   Go SDK     │  │    Control API       │  │
│  │   Program    │←─│   Client     │←─│  (IDE Integration)   │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│         ↑                ↑ ↓                    ↑ ↓             │
└─────────│────────────────│─│────────────────────│─│─────────────┘
          │                │ │                    │ │
     ┌────┴────┐      ┌────┴─┴────┐         ┌────┴─┴────┐
     │   SSE   │      │   HTTP    │         │  Control  │
     │ Stream  │      │   REST    │         │    API    │
     │ (push)  │      │  (pull)   │         │  (poll)   │
     └────┬────┘      └────┬──────┘         └────┬──────┘
          │                │                     │
          ↓                ↓                     ↓
┌─────────────────────────────────────────────────────────────────┐
│                   OpenCode Server (TypeScript)                   │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Event Bus   │  │  REST API    │  │    TUI Control       │  │
│  │  (publish)   │  │  Handlers    │  │    Endpoints         │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## TUI Initialization Flow

**File:** `packages/tui/cmd/opencode/main.go`

When the TUI starts, it performs these steps:

### Step 1: Connect to Server
```go
// Read server URL from environment
url := os.Getenv("OPENCODE_SERVER")

// Create HTTP client with Go SDK
httpClient = opencode.NewClient(option.WithBaseURL(url))
```

### Step 2: Parallel Initialization
```go
// Fetch required data in parallel
eg.Go(func() error {
    project, err = httpClient.Project.Current(ctx)
    return err
})

eg.Go(func() error {
    agentList, err = httpClient.Agent.List(ctx)
    return err
})

eg.Go(func() error {
    paths, err = httpClient.Path.Get(ctx)
    return err
})
```

### Step 3: Start Bubble Tea Program
```go
program := tea.NewProgram(model, tea.WithAltScreen())
```

### Step 4: Start SSE Event Stream (Background)
```go
go func() {
    stream := httpClient.Event.ListStreaming(ctx, opencode.EventListParams{})
    for stream.Next() {
        evt := stream.Current().AsUnion()
        program.Send(evt)  // Send event to Bubble Tea
    }
}()
```

### Step 5: Start Control API (Background)
```go
go api.Start(app, program)  // For IDE integration
```

---

## SSE Event Streaming

### How SSE Works

**Server Side** (`packages/opencode/src/server/server.ts:1519-1562`):

```typescript
// GET /event endpoint
app.get("/event", async (c) => {
    return streamSSE(c, async (stream) => {
        // Send initial connection event
        await stream.writeSSE({
            data: JSON.stringify({ type: "server.connected", properties: {} })
        })

        // Subscribe to ALL events from the Bus
        const unsub = await Bus.subscribeAll((event, data) => {
            stream.writeSSE({
                data: JSON.stringify({ type: event, properties: data })
            })
        })

        // Clean up on disconnect
        stream.onAbort(() => unsub())
    })
})
```

**Go SDK** (`packages/sdk/go/packages/ssestream/ssestream.go`):

```go
type Stream[T any] struct {
    decoder *sse.Decoder
    // ...
}

func (s *Stream[T]) Next() bool {
    // Read next SSE event from stream
    // Parse "event:" and "data:" fields
    // Unmarshal JSON into type T
    return true
}

func (s *Stream[T]) Current() T {
    return s.cur
}
```

**TUI Consumer** (`packages/tui/internal/tui/tui.go:146-155`):

```go
// Background goroutine listens for events
stream := httpClient.Event.ListStreaming(ctx, opencode.EventListParams{})
for stream.Next() {
    evt := stream.Current().AsUnion()
    program.Send(evt)  // Send to Bubble Tea's Update() loop
}
```

### SSE Event Format

Events are sent as Server-Sent Events (text/event-stream):

```
data: {"type":"message.updated","properties":{"sessionID":"ses_123","messageID":"msg_456"}}

data: {"type":"message.part.updated","properties":{"sessionID":"ses_123","messageID":"msg_456","partID":"part_789"}}

data: {"type":"session.idle","properties":{"sessionID":"ses_123"}}

```

---

## Event Handlers in TUI

**File:** `packages/tui/internal/tui/tui.go:464-680`

The TUI's `Update()` method handles incoming SSE events:

```go
func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {

    case opencode.EventListResponseEventInstallationUpdated:
        // Show toast: "opencode updated to X, restart to apply"

    case opencode.EventListResponseEventSessionDeleted:
        // Clear session and messages

    case opencode.EventListResponseEventSessionUpdated:
        // Update app.Session with new info

    case opencode.EventListResponseEventSessionCompacted:
        // Show success toast

    case opencode.EventListResponseEventMessagePartUpdated:
        // Find message by ID, update or insert part

    case opencode.EventListResponseEventMessagePartRemoved:
        // Find message by ID, remove part

    case opencode.EventListResponseEventMessageRemoved:
        // Remove entire message from list

    case opencode.EventListResponseEventMessageUpdated:
        // Add new message or update existing

    case opencode.EventListResponseEventPermissionUpdated:
        // Show permission popup, blur editor

    case opencode.EventListResponseEventPermissionReplied:
        // Remove from permission queue

    case opencode.EventListResponseEventSessionError:
        // Show error toast with details
    }
}
```

### Events Handled vs Not Handled

| Event | Handled in TUI? | Action |
|-------|-----------------|--------|
| `installation.updated` | ✅ Yes | Show update toast |
| `session.deleted` | ✅ Yes | Clear UI state |
| `session.updated` | ✅ Yes | Update session info |
| `session.compacted` | ✅ Yes | Show success toast |
| `message.updated` | ✅ Yes | Add/update message |
| `message.removed` | ✅ Yes | Remove message |
| `message.part.updated` | ✅ Yes | Stream text chunks |
| `message.part.removed` | ✅ Yes | Remove part |
| `permission.updated` | ✅ Yes | Show permission dialog |
| `permission.replied` | ✅ Yes | Clear dialog |
| `session.error` | ✅ Yes | Show error toast |
| `server.connected` | ❌ No | (Heartbeat only) |
| `session.idle` | ❌ No | (Not needed in TUI) |
| `session.created` | ❌ No | (TUI creates sessions) |
| `file.edited` | ❌ No | (Editor extension handles) |
| `file.watcher.updated` | ❌ No | (Background) |
| `lsp.client.diagnostics` | ❌ No | (Editor extension handles) |
| `todo.updated` | ❌ No | (Background) |
| `ide.installed` | ❌ No | (One-time) |

---

## HTTP API Calls (Beyond SSE)

The TUI doesn't only use SSE. It makes many HTTP calls for actions and queries:

### Session Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `Session.New()` | `POST /session` | Create new session |
| `Session.List()` | `GET /session` | Fetch all sessions |
| `Session.Get()` | `GET /session/:id` | Get single session |
| `Session.Delete()` | `DELETE /session/:id` | Delete session |
| `Session.Update()` | `PATCH /session/:id` | Rename session |
| `Session.Share()` | `POST /session/:id/share` | Create share link |
| `Session.Unshare()` | `DELETE /session/:id/share` | Remove share link |
| `Session.Messages()` | `GET /session/:id/message` | Get message history |
| `Session.Children()` | `GET /session/:id/children` | Get child sessions |

### Message Operations

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `Session.Prompt()` | `POST /session/:id/prompt` | Send user message |
| `Session.Command()` | `POST /session/:id/command` | Execute slash command |
| `Session.Shell()` | `POST /session/:id/shell` | Execute shell command |
| `Session.Init()` | `POST /session/:id/init` | Set model/provider |
| `Session.Summarize()` | `POST /session/:id/summarize` | Compact session |
| `Session.Abort()` | `POST /session/:id/abort` | Cancel operation |
| `Session.Revert()` | `POST /session/:id/revert` | Undo changes |
| `Session.Unrevert()` | `POST /session/:id/unrevert` | Redo changes |
| `Session.Permissions.Respond()` | `POST /session/:id/permission/:pid/respond` | Answer permission |

### Configuration & Discovery

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `Config.Get()` | `GET /config` | Get keybinds, theme, settings |
| `Project.Current()` | `GET /project/current` | Get project metadata |
| `Path.Get()` | `GET /path` | Get filesystem paths |
| `Agent.List()` | `GET /agent` | List available agents |
| `App.Providers()` | `GET /app/providers` | Get AI providers/models |
| `Command.List()` | `GET /command` | Get custom commands |

### Search & File Operations

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `Find.Files()` | `POST /find/files` | Ripgrep file search |
| `Find.Symbols()` | `POST /find/symbols` | Code symbol search |
| `File.Status()` | `GET /file/status` | Get file diff status |

---

## Control API (Bidirectional)

**File:** `packages/tui/internal/api/api.go`

The Control API allows external tools (like IDE extensions) to interact with the TUI:

```go
func Start(app *app.App, program *tea.Program) {
    // Poll for commands from server
    for {
        res, _ := app.Client.Tui.Control.Next(ctx)

        switch res.Command {
        case "focus":
            // Focus TUI window
        case "input":
            // Insert text into input
        case "prompt":
            // Send a prompt
        }

        // Send response back
        app.Client.Tui.Control.Response(ctx, ...)
    }
}
```

---

## Complete Event Flow Diagram

Here's what happens when a user sends a prompt:

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER ACTION                              │
│                    User types "Hello" and presses Enter          │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     TUI: Bubble Tea Update()                     │
│                                                                 │
│  case tea.KeyMsg:                                               │
│      if key == "enter":                                         │
│          return app.SendPrompt(input)                           │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     TUI: App.SendPrompt()                        │
│                                                                 │
│  client.Session.Prompt(ctx, sessionID, params)                  │
│  → HTTP POST /session/:id/prompt                                │
│  → Returns immediately (async processing)                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     SERVER: Process Prompt                       │
│                                                                 │
│  1. Create user message                                         │
│     → Bus.publish("message.updated", {...})                     │
│                                                                 │
│  2. Create assistant message placeholder                        │
│     → Bus.publish("message.updated", {...})                     │
│                                                                 │
│  3. Call AI provider (stream response)                          │
│     → Bus.publish("message.part.updated", {...})  [×N chunks]   │
│                                                                 │
│  4. Complete response                                           │
│     → Bus.publish("message.updated", {...})                     │
│     → Bus.publish("session.idle", {...})                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     SSE Stream to TUI                            │
│                                                                 │
│  GET /event (persistent connection)                             │
│                                                                 │
│  data: {"type":"message.updated","properties":{...}}            │
│  data: {"type":"message.part.updated","properties":{...}}       │
│  data: {"type":"message.part.updated","properties":{...}}       │
│  data: {"type":"message.updated","properties":{...}}            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     TUI: Event Handler                           │
│                                                                 │
│  for stream.Next() {                                            │
│      evt := stream.Current()                                    │
│      program.Send(evt)  // → Bubble Tea Update()                │
│  }                                                              │
│                                                                 │
│  case EventMessagePartUpdated:                                  │
│      // Find message, append text chunk                         │
│      // UI re-renders with new text                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Insights

### 1. Asymmetric Communication
- **SSE (push)**: Server pushes events to TUI in real-time
- **REST (pull)**: TUI sends commands to server
- This creates a responsive, decoupled UI

### 2. Event-Driven UI Updates
- TUI never polls for message updates
- Server pushes `message.part.updated` as AI generates text
- Enables real-time streaming display

### 3. Discriminated Union Types
- Go SDK defines `EventListResponse` as a union of all 19 event types
- TUI uses type assertions to handle each event type
- Type-safe event handling

### 4. Streaming Optimization
- Message parts are streamed individually
- `message.part.updated` fires for each text chunk
- Allows character-by-character display

### 5. Large Event Support
- Custom unbounded SSE decoder (>32MB)
- Handles large file contents, long outputs
- Defined in `decoders/decoder.go`

### 6. Graceful Degradation
- If SSE disconnects, TUI can still fetch via HTTP
- `Session.Messages()` provides full message history
- Events are for real-time; HTTP is for recovery

---

## Summary

| Question | Answer |
|----------|--------|
| **Does TUI depend only on SSE?** | No - uses SSE + HTTP REST + Control API |
| **What is SSE used for?** | Real-time push updates (messages, permissions, errors) |
| **What is HTTP used for?** | Commands, queries, actions (send prompt, create session) |
| **What is Control API for?** | IDE integration, external tool communication |
| **How does streaming work?** | `message.part.updated` events fire for each text chunk |
| **Where are events handled?** | `packages/tui/internal/tui/tui.go:464-680` |

---

## File Locations

| Component | File |
|-----------|------|
| TUI Entry Point | `packages/tui/cmd/opencode/main.go` |
| TUI Event Handlers | `packages/tui/internal/tui/tui.go` |
| TUI App Logic | `packages/tui/internal/app/app.go` |
| TUI Control API | `packages/tui/internal/api/api.go` |
| Go SDK Events | `packages/sdk/go/event.go` |
| Go SDK SSE Stream | `packages/sdk/go/packages/ssestream/ssestream.go` |
| Server SSE Endpoint | `packages/opencode/src/server/server.ts:1519` |
| Event Bus | `packages/opencode/src/bus/index.ts` |
