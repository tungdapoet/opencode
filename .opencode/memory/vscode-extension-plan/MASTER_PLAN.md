# VS Code Extension for OpenCode - Master Plan

## Executive Summary

Build a **Webview-based VS Code Extension** that serves as a rich GUI frontend for OpenCode's existing headless server. The extension will connect to a running `opencode serve` instance, providing a chat-first interface for sessions, messages, tool execution monitoring, and session management—all without modifying core OpenCode.

**Key Strategy:** Headless server + rich webview UI = zero core changes, modular architecture, easy maintenance.

---

## Phase Breakdown & Deliverables

### PHASE 1: Research & Design (CURRENT)

**Duration:** 2-3 days | **Deliverables:** Architecture doc, UI mockups, API integration strategy

#### Tasks

1. **Architecture Design Document**
   - Component breakdown (extension host, webview, state management)
   - Data flow diagrams (server ↔ extension ↔ webview)
   - Streaming strategy (SSE vs WebSocket decision)
   - State management approach (Zustand? Jotai? Context?)
   - Error handling and resilience patterns
   - **Output:** `VSCODE_EXTENSION_ARCHITECTURE.md`

2. **UI/UX Design & Mockups**
   - Sidebar: Project/Session selector
   - Main chat view: Messages timeline, streaming display
   - Tool execution panel: Status, logs, duration, cost
   - Permission dialog: Accept/deny/ask interactions
   - Input bar: Message composer + model/agent selector
   - Settings panel: Server connection, preferences
   - **Output:** Figma/image mockups, component specs

3. **API Integration Strategy**
   - How to wrap @opencode-ai/sdk for extension use
   - Streaming patterns: SSE consumer, message assembly
   - Permission request handling (web UI feedback loop)
   - Event subscription model
   - **Output:** `API_INTEGRATION.md` with code examples

4. **Technical Spike (SSE vs WebSocket)**
   - Prototype SSE consumer in webview
   - Test message streaming assembly
   - Validate Bun WebSocket support on server side
   - **Outcome:** Clear decision + working code sample

5. **Dependencies & Build Strategy**
   - Review existing vscode extension setup (esbuild, webpack)
   - UI library choice: React, Vue, vanilla? → React (matches TUI patterns)
   - Markdown rendering: markdown-it, remark, or VS Code API?
   - Code highlighting: Shiki or highlight.js?
   - State management: Zustand (lightweight, Redux-like)
   - **Output:** `package.json` plan, build config decisions

6. **Risk Assessment**
   - Server availability/discovery mechanism
   - Webview CSP constraints on external resources
   - Connection lifecycle management
   - Message ordering under streaming
   - **Output:** Risk matrix + mitigation strategies

---

### PHASE 1.5: Critical Prerequisites (BLOCKING) ⚠️

**Duration:** 2-3 days | **Deliverable:** Validated foundations before implementation

> **Added post-review:** These critical gaps were identified during architecture review and MUST be completed before Phase 2 begins.

#### Tasks

1. **SSE Runtime Validation** (1 day) ⚠️ CRITICAL
   - Node.js doesn't have native `EventSource` - must validate polyfill works
   - Install `eventsource` npm package
   - Create proof-of-concept connecting to `opencode serve` from extension host
   - Validate reconnection semantics and buffering strategy
   - Document message parsing matches server format
   - **Output:** `src/streaming/sse-polyfill.ts`, integration test, documentation

2. **State Synchronization Architecture** (0.5 day) ⚠️ CRITICAL
   - Current plan broadcasts full state on every mutation → causes thrashing
   - Decide: Extension host owns state (recommended) OR webview owns state
   - Define state projection schema (what gets sent to webview)
   - Implement throttling (max 10 updates/sec)
   - Add message deduplication and backpressure
   - **Output:** Architecture decision in DECISIONS_AND_QA.md, `src/messaging/sync.ts`

3. **Security Hardening** (0.5 day) ⚠️ HIGH
   - Define Content Security Policy for webview
   - Configure DOMPurify for markdown sanitization
   - Audit Shiki for inline script/style injection risks
   - Plan sandboxing for tool output code blocks
   - Test XSS payloads don't execute
   - **Output:** CSP in `src/webview/index.html`, `src/utils/sanitize.ts`

4. **Message Part Coverage** (0.5 day) ⚠️ MEDIUM
   - Only ~6 of 16 MessageV2 part types have renderer specs
   - Document all missing types: subtask, retry, step-start/finish, compaction, patch, summary
   - Create placeholder components for unimplemented types
   - Add exhaustive type switch to prevent crashes on unknown types
   - **Output:** Updated DECISIONS_AND_QA.md, `src/webview/components/parts/UnknownPart.tsx`

#### Acceptance Criteria (Gate for Phase 2)

- [ ] SSE polyfill works in extension host context (tested with real server)
- [ ] State sync architecture documented with proof-of-concept
- [ ] CSP policy defined and XSS payloads blocked
- [ ] All 16 message part types have at least placeholder specs

---

### PHASE 2: Core Infrastructure

**Duration:** 3-4 days | **Deliverable:** Working extension skeleton with server connection

#### Tasks

1. **Extension Entry Point & Webview Setup**
   - Update `extension.ts` activation lifecycle
   - Create webview panel management (singleton or multi-window?)
   - HTML/CSS base for webview (dark/light theme support)
   - Resource URI handling for images, scripts
   - **Output:** `src/extension.ts`, `src/webview/index.html`, theme assets

2. **Server Discovery & Connection**
   - Config: Manual URL input or auto-discovery via `opencode serve --port`?
   - Status bar indicator (connected/disconnected)
   - Reconnection logic with exponential backoff
   - Connection validation endpoint
   - **Output:** `src/server/connection.ts`, status bar UI

3. **SDK Client Wrapper**
   - Create extension-friendly client factory
   - Request/response logging
   - Error normalization
   - Timeout handling
   - **Output:** `src/sdk/client.ts` (wraps @opencode-ai/sdk)

4. **State Management Setup (Zustand)**
   - Global app state: connection, current project, current session
   - Session state: messages, pending requests, permissions
   - UI state: sidebar collapsed, theme, selected message
   - Persistence layer (VS Code global/workspace storage)
   - **Output:** `src/state/store.ts`, `src/state/slices/*`

5. **Webview Communication Bridge**
   - Extension host ↔ Webview message passing
   - Type-safe message schema (Zod)
   - Request/response correlation
   - Event subscription forwarding
   - **Output:** `src/messaging/protocol.ts`, `src/messaging/bridge.ts`

6. **React Webview Setup**
   - Root component with providers (Zustand, theme context)
   - Layout skeleton: sidebar + main area
   - Error boundary component
   - **Output:** `src/webview/App.tsx`, basic layout

---

### PHASE 3: MVP UI Components

**Duration:** 4-5 days | **Deliverable:** Functional chat UI with message rendering

#### Tasks

1. **Sidebar Components**
   - ProjectList: Load projects, display with status
   - SessionList: For selected project, create/delete/switch
   - SessionDetail: Current session info, buttons
   - **Output:** `src/webview/components/sidebar/*`

2. **Message Timeline View**
   - MessageList component (virtualized scrolling for large histories)
   - MessageItem with role-based styling (user vs assistant)
   - Auto-scroll to latest
   - Timestamp + session markers
   - **Output:** `src/webview/components/chat/MessageList.tsx`

3. **Part Renderers**
   - TextPartRenderer: Markdown with syntax highlighting (use Shiki)
   - ToolPartRenderer: Collapsible tool execution with state badge
     - Pending: spinner
     - Running: progress indicator
     - Completed: output preview
     - Error: error message
   - FilePart: Link with icon, optional preview
   - ReasoningPart: Collapsible, styled differently
   - SnapshotPart: Link to snapshot
   - AgentPart: Agent name with link
   - **Output:** `src/webview/components/parts/*.tsx`

4. **Input Composer**
   - Textarea with auto-expand
   - Model/provider selector dropdown
   - Agent selector (build/plan/custom)
   - Send button (disabled while sending)
   - Clear/reset button
   - **Output:** `src/webview/components/composer/MessageInput.tsx`

5. **Tool Execution Display**
   - ToolGrid or ToolList view (sortable by status/time)
   - ToolCard: name, input summary, status, duration, cost
   - Click to expand full execution details
   - **Output:** `src/webview/components/tools/ToolPanel.tsx`

6. **Status & Loading States**
   - Connection status indicator
   - Message sending progress
   - Skeleton loaders for lists
   - **Output:** `src/webview/components/common/LoadingState.tsx`

---

### PHASE 4: Streaming & Interactions

**Duration:** 3-4 days | **Deliverable:** Live streaming, permissions, message sending

#### Tasks

1. **SSE Message Streaming**
   - SSE event consumer (fetch EventSource in extension host)
   - Message part assembly pipeline
   - Real-time UI updates via state mutations
   - Error recovery (reconnection)
   - **Output:** `src/streaming/sse-consumer.ts`, integrated into store

2. **Message Sending Flow**
   - POST message endpoint call
   - Optimistic UI update (show message immediately)
   - Stream parts as they arrive
   - Merge streamed parts into message
   - Handle API errors gracefully
   - **Output:** `src/api/messages.ts` with sendMessage handler

3. **Permission Request Handling**
   - Listen for permission events
   - Display permission dialog modal
   - Accept/deny/ask (custom prompt) buttons
   - POST permission response
   - Update session state post-permission
   - **Output:** `src/components/PermissionDialog.tsx`, handler in store

4. **Session Management Actions**
   - CreateSession: Form modal, POST /project/:id/session
   - DeleteSession: Confirm dialog, DELETE endpoint
   - ReverSession: Confirm + select snapshot, POST /revert
   - Share/UnshareSession: Toggle buttons
   - AbortSession: Stop button in header
   - **Output:** `src/api/sessions.ts`, modal components

5. **Event Subscription System**
   - Global event stream consumer
   - Event type dispatching to handlers
   - State mutations on relevant events
   - Cleanup on unmount
   - **Output:** `src/streaming/event-subscription.ts`

6. **Error Handling & Retry**
   - User-visible error toasts
   - Automatic retry with backoff for transient errors
   - Manual retry buttons for permanent errors
   - Error boundary for component crashes
   - **Output:** `src/utils/error-handler.ts`, Toast system

---

### PHASE 5: Advanced Features

**Duration:** 3-4 days | **Deliverable:** Session history, snapshots, file diffs, cost tracking

#### Tasks

1. **Session History & Search**
   - Load all messages for session (paginated?)
   - Search by text, tool, date range
   - Quick filters (user/assistant/tool messages)
   - Message deletion with confirmation
   - **Output:** `src/api/search.ts`, `SearchPanel.tsx`

2. **Snapshot & Revert UI**
   - Snapshot selector dropdown in session header
   - Timeline visualization of snapshots
   - Revert confirmation dialog with diff preview
   - Unrevert button if available
   - **Output:** `src/components/SnapshotSelector.tsx`, `RevertDialog.tsx`

3. **File Diff Visualization**
   - Display patch parts with side-by-side diffs
   - Use VS Code's diff API? Or embed solution?
   - Show affected files summary
   - Option to open in editor
   - **Output:** `src/components/DiffViewer.tsx`

4. **Cost & Token Tracking**
   - Extract from StepFinishPart
   - Display per-message breakdown
   - Cumulative session cost
   - Model comparison (show cost by model)
   - **Output:** `src/components/CostBreakdown.tsx`

5. **Session Compaction Support**
   - Show CompactionPart in timeline
   - Trigger manual compaction button
   - Display compaction progress
   - **Output:** `src/api/sessions.ts` compactSession, UI trigger

6. **File Explorer Integration**
   - List files in session directory (optional)
   - Open file in editor from message reference
   - Sync current editor file to session context
   - **Output:** `src/utils/file-integration.ts`

---

### PHASE 6: Polish & Integration

**Duration:** 2-3 days | **Deliverable:** Packaged extension, docs, testing

#### Tasks

1. **Theme Integration**
   - Detect VS Code theme (light/dark)
   - CSS-in-JS or CSS modules for consistent styling
   - Respect VS Code color tokens
   - Focus states and accessibility
   - **Output:** `src/styles/theme.ts`, updated components

2. **Keyboard Shortcuts & Accessibility**
   - Command palette integration
   - Quick session switcher (Ctrl+Shift+O?)
   - Send message (Ctrl+Enter)
   - Focus trap in modals
   - ARIA labels, semantic HTML
   - **Output:** `package.json` keybindings, a11y checklist

3. **Configuration & Settings**
   - VS Code settings for extension:
     - Server URL (fallback to localhost:auto-detect)
     - Auto-connect on startup
     - Theme preference override
     - Message limit per session
   - **Output:** `package.json` contributes.configuration

4. **Documentation**
   - README: Setup, connection, usage, features
   - Architecture overview for maintainers
   - Contributing guide
   - Troubleshooting section
   - **Output:** `README.md`, `ARCHITECTURE.md`, `TROUBLESHOOTING.md`

5. **Testing Strategy**
   - Unit tests for state mutations (Zustand store)
   - Integration tests for API calls (mock SDK)
   - Webview component tests (React Testing Library)
   - E2E tests (manual or Playwright?)
   - **Output:** `src/**/*.test.ts`, test configuration

6. **Build & Packaging**
   - Update esbuild config for webview bundling
   - Minification and tree-shaking
   - VSIX package generation
   - GitHub Actions workflow for auto-release
   - **Output:** Updated build scripts, CI/CD config

7. **Launch Checklist**
   - Extension gallery metadata (description, tags, keywords)
   - Marketplace icon and banner
   - Changelog generation
   - Version bump strategy (semver)
   - **Output:** `package.json` metadata, marketplace entry

---

## Technical Decisions & Rationale

### 1. Streaming Strategy: SSE Over WebSocket

**Decision:** Use SSE (Server-Sent Events) for message part streaming.

**Rationale:**

- Server already implements SSE for `/global/event` (proven pattern)
- Simpler for one-way server → client streaming
- Works through corporate proxies (WebSocket often blocked)
- Built on HTTP, no additional connection handshake
- Better error recovery (HTTP-native reconnection)

**Implementation:** Fetch EventSource API in extension host, parse JSON events, dispatch to state manager.

### 2. State Management: Zustand

**Decision:** Zustand for extension state, store subscriptions for webview updates.

**Rationale:**

- Lightweight (~2KB)
- Minimal boilerplate vs Redux
- React hooks integration (no provider hell)
- Middleware support for logging/persistence
- Scales well for extension's modest scope

**Store structure:**

```
connection/  - server URL, status, retry logic
project/     - current project, list
session/     - current session, messages, permissions
ui/          - sidebar state, theme, modals
```

### 3. UI Framework: React

**Decision:** React for webview components.

**Rationale:**

- TUI uses similar architecture patterns (component state, event handlers)
- Large ecosystem for VS Code extensions
- Strong TypeScript support
- Good performance with virtualization libraries

### 4. Markdown Rendering: Shiki + Remark

**Decision:** Markdown-it with Shiki code highlighting.

**Rationale:**

- Lightweight, no heavy AST processing needed
- Shiki has excellent theme support (matches VS Code)
- Security: sanitize HTML output
- Simplicity: one-pass rendering

### 5. Server Discovery: Manual + Auto-fallback

**Decision:**

1. First try: Get from VS Code workspace setting
2. Fallback: Try `localhost:42600` (common port for `opencode serve --port 0`)
3. Fallback: Show manual input dialog

**Rationale:**

- User control with sensible defaults
- Handles most common setups without config
- Explicit connection dialog for advanced users

### 6. Data Persistence: VS Code Storage API

**Decision:** Use `context.globalState` and `context.workspaceState`.

**Rationale:**

- No external databases needed
- Respects VS Code multi-instance model
- Automatic sync across machines (if user enables settings sync)
- Simple key-value interface

---

## Component Architecture

### Extension Host (Main Process)

```
extension.ts (activation, webview mgmt)
├── messaging/
│   ├── protocol.ts (message types)
│   └── bridge.ts (host ↔ webview IPC)
├── server/
│   ├── connection.ts (establish, reconnect)
│   └── discovery.ts (URL resolution)
├── sdk/
│   └── client.ts (wrapper around @opencode-ai/sdk)
├── streaming/
│   ├── sse-consumer.ts (SSE event stream)
│   └── event-subscription.ts (global events)
├── state/
│   ├── store.ts (Zustand store)
│   └── slices/ (connection, project, session, ui)
└── utils/
    ├── error-handler.ts
    └── logger.ts
```

### Webview (React App)

```
App.tsx (root provider setup)
├── components/
│   ├── sidebar/
│   │   ├── ProjectList.tsx
│   │   ├── SessionList.tsx
│   │   └── SessionDetail.tsx
│   ├── chat/
│   │   ├── MessageList.tsx
│   │   ├── MessageItem.tsx
│   │   └── MessageComposer.tsx
│   ├── parts/
│   │   ├── TextPartRenderer.tsx
│   │   ├── ToolPartRenderer.tsx
│   │   ├── FilePart.tsx
│   │   └── ... (other part types)
│   ├── tools/
│   │   ├── ToolPanel.tsx
│   │   └── ToolCard.tsx
│   ├── modals/
│   │   ├── PermissionDialog.tsx
│   │   ├── CreateSessionModal.tsx
│   │   ├── RevertDialog.tsx
│   │   └── SearchModal.tsx
│   └── common/
│       ├── StatusIndicator.tsx
│       ├── Toast.tsx
│       └── LoadingState.tsx
├── hooks/
│   ├── useStore.ts (Zustand integration)
│   ├── useSSE.ts (SSE consumption)
│   └── useAPI.ts (request wrapper)
├── utils/
│   ├── markdown.ts (rendering)
│   ├── format.ts (date, size, duration)
│   └── type-guards.ts
└── styles/
    ├── theme.ts
    └── globals.css
```

---

## Data Flow Diagrams

### Sending a Message

```
User Input → MessageComposer
    ↓
UI optimistically adds message to store
    ↓
POST /project/:id/session/:id/message → Server
    ↓
Server returns message ID
    ↓
SSE stream: Parts arrive → PartUpdated events
    ↓
Extension host assembles parts → store.messages[id].parts.push()
    ↓
Webview re-renders (Zustand subscriber)
    ↓
User sees streaming response in MessageItem
```

### Handling Permissions

```
Extension host listening to global events
    ↓
Receives PermissionRequested event (contains tool, action, description)
    ↓
Webview: PermissionDialog modal opens
    ↓
User selects: Allow / Deny / Ask again
    ↓
POST /project/:id/permission/:permissionId { decision: "allow" }
    ↓
Server resumes execution with granted permission
    ↓
Message continues streaming
```

### Session Switching

```
User clicks session in SessionList
    ↓
store.session.setCurrentSession(sessionId)
    ↓
useEffect in ChatView triggers: fetchMessages(sessionId)
    ↓
GET /project/:id/session/:id/message → returns message array
    ↓
store.messages = response
    ↓
MessageList re-renders with new session's messages
```

---

## Key Integration Points with OpenCode

### 1. API Contract (No changes needed)

- Uses existing `/project`, `/session`, `/message` endpoints
- Consumes `/global/event` SSE stream
- Calls `/permission/:id` for permission handling
- Calls `/revert` for snapshots (if supported)

### 2. Message Part Types

- Renderer for each part type (TextPart, ToolPart, FilePart, etc.)
- Tool execution display fed by ToolState transitions
- Markdown rendering for TextPart content

### 3. Server Events (from Bus system)

- Subscribe to all events published by server
- Parse by type discriminator
- Dispatch to appropriate handlers (permissions, updates, errors)

---

## Risk Mitigation

| Risk                                      | Likelihood | Impact | Mitigation                                                 |
| ----------------------------------------- | ---------- | ------ | ---------------------------------------------------------- |
| Server unavailable on startup             | Medium     | High   | Auto-reconnect logic, show status in UI, retry button      |
| SSE connection drops mid-stream           | Low        | Medium | Reconnect and refetch missing parts, deduplicate           |
| Webview CSP blocks external resources     | Low        | Medium | Use VS Code's `asWebviewUri()` for all resources           |
| Message part ordering issues              | Low        | Medium | Assemble by (messageId, partType, timestamp) tuple         |
| Permission dialog UX unclear              | Medium     | Low    | Clear description + reason, show tool documentation        |
| Performance with large sessions           | Medium     | Medium | Virtual scrolling (react-window), lazy-load older messages |
| Sync issues (extension ↔ server diverges) | Low        | High   | Event-sourced state, always fetch latest on switch         |

---

## Success Criteria

### Phase 1

- [x] Architecture document completed
- [x] UI mockups reviewed
- [x] API integration strategy documented
- [x] SSE vs WS decision made with prototype

### Phase 2

- [ ] Extension loads and connects to server
- [ ] Status bar shows connection state
- [ ] Webview renders with layout skeleton

### Phase 3

- [ ] Projects/sessions load and display
- [ ] Messages render with all part types
- [ ] Input composer sends messages
- [ ] Tool execution appears in UI

### Phase 4

- [ ] Message parts stream in real-time
- [ ] Permission dialog handles requests
- [ ] Session actions (create/delete/revert) work

### Phase 5

- [ ] Search across messages
- [ ] Snapshot selection and revert
- [ ] File diffs displayed
- [ ] Cost tracking shown

### Phase 6

- [ ] All tests passing
- [ ] Keyboard shortcuts functional
- [ ] Extension packaged and publishable
- [ ] Documentation complete

---

## Timeline Estimate

| Phase             | Duration       | Effort           |
| ----------------- | -------------- | ---------------- |
| 1. Design         | 2-3 days       | 1 dev            |
| 2. Infrastructure | 3-4 days       | 1-2 devs         |
| 3. MVP UI         | 4-5 days       | 1-2 devs         |
| 4. Streaming      | 3-4 days       | 1 dev            |
| 5. Advanced       | 3-4 days       | 1 dev            |
| 6. Polish         | 2-3 days       | 1 dev            |
| **Total**         | **18-23 days** | **6-9 dev-days** |

**Parallel opportunities:**

- Phase 2 + Phase 3 can overlap (infrastructure while designing UI)
- Phase 4 + Phase 5 can run concurrently (streaming independent of advanced features)

---

## Next Steps

1. **Validate Architecture** - Review with team, adjust if needed
2. **Finalize UI Mockups** - Get design buy-in
3. **Create Phase 1 Deliverables** - Architecture doc, API strategy, spike results
4. **Setup Development Environment** - Repository, build config, initial scaffolding
5. **Begin Phase 2** - Core infrastructure implementation
