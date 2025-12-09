# VS Code Extension Architecture Research

## Current State Analysis (as of Dec 2025)

### Existing VS Code Extension (sdks/vscode/)
- **Current role**: Terminal launcher + context helper
- **Approach**: Spawns opencode TUI in terminal with file context passing
- **Limitations**: Not a full GUI, just enhances existing TUI workflow

### Server Architecture (packages/opencode/src/server/server.ts)
- **Framework**: Hono-based HTTP server with OpenAPI support
- **Streaming**: SSE for streaming responses (`/global/event`, message streaming)
- **WebSocket**: Available via `upgradeWebSocket` from Hono/Bun
- **Auth**: Uses Instance-based directory context
- **Error handling**: Custom error schema with status code mapping

### Message & Part Architecture (message-v2.ts)
**Message types:**
- `User` - user prompts with optional summary/diffs
- `Assistant` - AI responses with parts
- `System` - system messages

**Part types (16 total):**
1. TextPart - regular text
2. ReasoningPart - thinking/analysis (time tracked)
3. FilePart - file references with mime type + source location
4. ToolPart - tool execution (pending/running/completed/error states)
5. SnapshotPart - git snapshot references
6. PatchPart - file changes
7. AgentPart - agent invocations
8. CompactionPart - session compaction markers
9. SubtaskPart - delegated tasks
10. RetryPart - retry attempts with error details
11. StepStartPart - step markers
12. StepFinishPart - step completion with tokens/cost

**Tool states:** pending → running → completed/error
**Token tracking:** input, output, reasoning, cache read/write

### SDK Integration (packages/sdk/js/)
- Auto-generated from OpenAPI spec
- Exports: default client, v2 types, server utilities
- Published to npm: @opencode-ai/sdk

### API Routes (from specs/project.md)
```
/project - list, create, current
/project/:id/session - CRUD operations
/project/:id/session/:id/message - send, list, get
/project/:id/session/:id/message/:id - delete specific message
/project/:id/session/:id/abort - abort session
/project/:id/session/:id/share - session sharing
/project/:id/session/:id/compact - compaction
/project/:id/session/:id/revert - revert to snapshot
/project/:id/session/:id/file - list files, get status
/project/:id/permission/:id - handle permission requests
/global/event - global event stream (SSE)
```

### Key Technical Decisions Found
1. **SSE for streaming** - Used for global events and message parts
2. **Bus system** - Event pub/sub with type-safe schemas
3. **Instance scoping** - Per-directory context isolation
4. **Zod validation** - All schemas defined with Zod
5. **LSP integration** - Has LSP server for code intelligence

### Constraints Identified
1. Server spawns per-project (can have multiple instances)
2. No authentication layer - relies on file system access
3. Storage: Not directly visible, likely file-based per project
4. Permissions: Ask/allow/deny system for tool execution
