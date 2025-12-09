# VS Code Extension Plan - Executive Summary & Quick Reference

## The Big Picture

**Goal:** Build a webview-based VS Code Extension that provides a rich GUI for OpenCode, connecting to the existing headless server (`opencode serve`) without modifying core.

**Strategy:** Leverage proven patterns from the existing TUI + server architecture. Extension = GUI frontend only, server = unchanged.

---

## One-Page Quick Start

### Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                    VS Code Extension                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Extension Host (TypeScript)        Webview (React)             │
│  ├── extension.ts                   ├── App.tsx                │
│  ├── server/connection.ts           ├── components/             │
│  ├── sdk/client.ts                  │   ├── Chat                │
│  ├── streaming/sse-consumer.ts      │   ├── Sidebar             │
│  ├── state/store.ts (Zustand)       │   ├── Parts               │
│  └── messaging/bridge.ts            │   └── Modals              │
│                                      └── hooks/store             │
│                                                                  │
│         ↓↓↓ HTTP + SSE + IPC ↓↓↓                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
        ↓
    OpenCode Server (Hono)
    • /project, /session, /message
    • /global/event (SSE stream)
    • /permission (handling)
```

### Tech Stack

- **Language:** TypeScript
- **Extension UI:** React 18 + Zustand + CSS Modules
- **Server Communication:** Fetch API + EventSource (SSE)
- **Markdown:** Markdown-it + Shiki (syntax highlighting)
- **Build:** esbuild (existing setup)
- **Testing:** Vitest + React Testing Library

### Key Decisions (Final)

| Decision             | Choice              | Why                                   |
| -------------------- | ------------------- | ------------------------------------- |
| **Streaming**        | SSE                 | Already used, simpler, proxy-friendly |
| **State Mgmt**       | Zustand             | Lightweight, minimal boilerplate      |
| **UI Framework**     | React               | Ecosystem, TypeScript support         |
| **Markdown**         | Markdown-it + Shiki | Fast, VS Code color themes            |
| **Server Discovery** | 3-tier fallback     | Config → auto-detect → manual         |

---

## Six Phases at a Glance

### Phase 1: Design ✅ (COMPLETE)

**What:** Architecture, UI specs, API strategy, technical spikes
**Outputs:**

- MASTER_PLAN.md (this doc)
- DECISIONS_AND_QA.md (detailed Q&A)
- architecture-research.md (codebase analysis)

### Phase 2: Infrastructure (3-4 days)

**What:** Connection layer, state management, webview IPC
**Key deliverables:**

- `src/extension.ts` (main entry, webview mgmt)
- `src/server/connection.ts` (discovery + health checks)
- `src/state/store.ts` (Zustand store)
- `src/webview/App.tsx` (React root)

### Phase 3: MVP UI (4-5 days)

**What:** Chat interface, message rendering, basic UI
**Key deliverables:**

- Message list with virtualization
- Part renderers (text, tool, file)
- Input composer
- Sidebar (projects, sessions)

### Phase 4: Streaming (3-4 days)

**What:** Live updates, permissions, message sending
**Key deliverables:**

- SSE message stream consumer
- Permission dialog + handling
- Message send flow
- Real-time state updates

### Phase 5: Advanced Features (3-4 days)

**What:** History, snapshots, diffs, cost tracking
**Key deliverables:**

- Session history + search
- Snapshot selector & revert
- Diff viewer
- Cost breakdown

### Phase 6: Polish (2-3 days)

**What:** Testing, docs, packaging, launch
**Key deliverables:**

- Unit & integration tests
- Documentation (README, arch guide)
- VS Code Marketplace prep
- CI/CD workflow

---

## Critical Success Factors

### Must Have

1. ✅ Connects to running `opencode serve` instance
2. ✅ Sends messages and displays streamed responses live
3. ✅ Shows tool execution with status (pending→running→done)
4. ✅ Handles permission requests with user dialog
5. ✅ Zero changes to OpenCode core

### Should Have

- Agent selection (build/plan)
- Model/provider selection
- Session history & search
- File diffs
- Cost tracking

### Nice to Have

- Theme integration
- Inline code suggestions
- Multi-session tabs
- Custom agents

---

## File Structure (Extension Only)

```
sdks/vscode/
├── src/
│   ├── extension.ts              # Main entry point
│   ├── server/
│   │   ├── connection.ts         # Server discovery, health
│   │   └── discovery.ts          # URL resolution
│   ├── sdk/
│   │   └── client.ts             # @opencode-ai/sdk wrapper
│   ├── streaming/
│   │   ├── sse-consumer.ts       # EventSource subscription
│   │   └── event-subscription.ts # Global events
│   ├── state/
│   │   ├── store.ts              # Main Zustand store
│   │   └── slices/               # Store slices
│   │       ├── connection.ts
│   │       ├── project.ts
│   │       ├── session.ts
│   │       └── ui.ts
│   ├── messaging/
│   │   ├── protocol.ts           # Message types (Zod)
│   │   └── bridge.ts             # Host ↔ webview IPC
│   ├── utils/
│   │   ├── error-handler.ts
│   │   ├── markdown.ts
│   │   └── logger.ts
│   ├── webview/                  # React app
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── sidebar/
│   │   │   ├── chat/
│   │   │   ├── parts/
│   │   │   ├── tools/
│   │   │   ├── modals/
│   │   │   └── common/
│   │   ├── hooks/
│   │   │   ├── useStore.ts
│   │   │   ├── useSSE.ts
│   │   │   └── useAPI.ts
│   │   ├── styles/
│   │   │   ├── theme.ts
│   │   │   ├── globals.css
│   │   │   └── markdown.css
│   │   └── index.tsx
│   └── index.html                # Webview template
├── test/
│   ├── store.test.ts
│   ├── components/
│   └── integration/
├── esbuild.js                    # Updated build config
├── package.json                  # Updated with React deps
└── tsconfig.json
```

---

## API Endpoints Used

**No new endpoints needed.** Uses existing OpenCode API:

```
GET  /project
POST /project/init
GET  /project/current
GET  /project/:id/session
GET  /project/:id/session/:id
POST /project/:id/session
DELETE /project/:id/session/:id

GET  /project/:id/session/:id/message
POST /project/:id/session/:id/message        (streaming via SSE)
GET  /project/:id/session/:id/message/:mid
DELETE /project/:id/session/:id/message/:mid

GET  /global/event                           (SSE stream)

POST /project/:id/session/:id/abort
POST /project/:id/session/:id/share
POST /project/:id/session/:id/compact
POST /project/:id/session/:id/revert
POST /project/:id/permission/:id
```

---

## Answers to 6 Key Questions

### 1. Server Discovery

**Answer:** Config setting → localhost auto-detect (port 42600) → manual input dialog

### 2. Streaming Protocol

**Answer:** SSE (Server-Sent Events). Already proven in codebase, simpler than WebSocket.

### 3. State Management

**Answer:** Zustand with slices: connection, project, session, ui, messages

### 4. Markdown Rendering

**Answer:** Markdown-it (parsing) + Shiki (code highlighting) + DOMPurify (sanitization)

### 5. Permission UX

**Answer:** Toast notification → modal dialog → Allow/Deny/Ask again buttons + remember checkbox

### 6. File Integration

**Answer:** Open files in editor via URI, fetch content from session, apply patches to workspace

---

## Timeline & Effort

| Phase                    | Duration       | 1 Dev       | 2 Devs      |
| ------------------------ | -------------- | ----------- | ----------- |
| 1. Design                | 2-3 days       | ✅          | ✅          |
| **1.5 Prerequisites** ⚠️ | **2-3 days**   | **3d**      | **2d**      |
| 2. Infrastructure        | 4-5 days       | 5d          | 3d          |
| 3. MVP UI                | 5-6 days       | 6d          | 4d          |
| 4. Streaming             | 3-4 days       | 4d          | 2d          |
| 5. Advanced              | 3-4 days       | 4d          | 2d          |
| 6. Polish                | 2-3 days       | 3d          | 2d          |
| **TOTAL**                | **22-28 days** | **28 days** | **18 days** |

**Parallelization:** Phases 2+3 can overlap, phases 4+5 can overlap.
**Realistic:** 2 devs, 4-5 weeks to MVP (revised with buffer).

> ⚠️ **Timeline revised post-review:** Added Phase 1.5 for critical prerequisites and increased Phase 2/3 estimates to account for VSCode-specific complexity.

---

## Risks & Mitigations

| Risk                         | Severity     | Mitigation                                                     |
| ---------------------------- | ------------ | -------------------------------------------------------------- |
| Server not running           | High         | Auto-reconnect + status UI + helpful error messages            |
| SSE drops mid-stream         | Medium       | Reconnect + refetch missing parts + deduplicate                |
| Webview CSP blocks resources | Medium       | Use `asWebviewUri()` for all external resources                |
| Large session performance    | Medium       | Virtual scrolling + lazy loading + pagination/retention policy |
| Permission dialog confusing  | Low          | Clear descriptions + examples                                  |
| Message ordering issues      | Low          | Assemble by (messageId, partType, timestamp)                   |
| **SSE runtime unavailable**  | **Critical** | **Validate `eventsource` polyfill in VSCode Node env**         |
| **State sync thrashing**     | **High**     | **Implement diffing/throttling in IPC bridge**                 |
| **XSS via markdown/tools**   | **High**     | **Define CSP policy + sandbox untrusted content**              |
| **Auth/TLS in enterprise**   | **Medium**   | **Add proxy/auth config to connection layer**                  |
| **SDK version skew**         | **Medium**   | **Pin SDK version, validate on connect**                       |

---

## ⚠️ Critical Pre-Phase 2 Actions (BLOCKING)

**Status: MUST COMPLETE before starting Phase 2**

These critical gaps were identified during architecture review and must be addressed first:

### 1. SSE Runtime Validation (1 day)

**Problem:** Plan assumes native `EventSource` in VSCode's Node18 environment, but this is not guaranteed.

**Actions:**

- [ ] Install `eventsource` npm package as polyfill
- [ ] Create proof-of-concept in VSCode extension host context
- [ ] Validate reconnection semantics work correctly
- [ ] Document buffering strategy for dropped connections
- [ ] Test with actual `opencode serve` instance

**Validation:**

```typescript
// Test this works in extension.ts activation
import EventSource from "eventsource"
const es = new EventSource("http://localhost:42600/global/event")
es.onmessage = (e) => console.log("SSE works:", e.data)
```

### 2. State Synchronization Design (0.5 day)

**Problem:** Current approach broadcasts entire Zustand state on every mutation, causing webview thrashing and data races.

**Actions:**

- [ ] Decide: Single source of truth in host OR webview-owned UI store?
- [ ] Document selective projection strategy (what state goes to webview)
- [ ] Implement throttling/debouncing for high-frequency updates
- [ ] Define serialization format and costs
- [ ] Add backpressure mechanism for slow webviews

**Recommended Pattern:**

```typescript
// Extension host owns canonical state, webview gets projections
const projection = {
  connectionStatus: store.connection.status,
  currentSession: store.session.id,
  messages: store.messages.slice(-50), // Last 50 only
  pendingParts: store.streaming.pending,
}
// Throttle to max 10 updates/sec
```

### 3. Security Hardening (0.5 day)

**Problem:** DOMPurify mentioned but no CSP plan or sandboxing for untrusted content.

**Actions:**

- [ ] Define Content Security Policy for webview
- [ ] Document DOMPurify configuration for markdown
- [ ] Plan sandboxing for tool output code blocks
- [ ] Audit Shiki for inline script/style injection
- [ ] Add CSP headers to webview HTML template

**CSP Template:**

```html
<meta
  http-equiv="Content-Security-Policy"
  content="
  default-src 'none';
  style-src ${webview.cspSource} 'unsafe-inline';
  script-src ${webview.cspSource};
  img-src ${webview.cspSource} https: data:;
  font-src ${webview.cspSource};
"
/>
```

### 4. Complete Message Part Coverage (0.5 day)

**Problem:** Only subset of 16 MessageV2 part types have renderer specs.

**Actions:**

- [ ] Document renderer spec for: Subtask, Retry, StepStart, StepFinish
- [ ] Document renderer spec for: Compaction, Patch, Summary
- [ ] Add placeholder components for unimplemented types
- [ ] Define fallback rendering for unknown part types

**Missing Parts Checklist:**
| Part Type | Status | Renderer Spec |
|-----------|--------|---------------|
| text | ✅ | Markdown with syntax highlighting |
| tool-call | ✅ | Collapsible panel with args |
| tool-result | ✅ | Output display with status |
| file | ✅ | File path with open action |
| **subtask** | ❌ | TODO: Nested task display |
| **retry** | ❌ | TODO: Retry indicator with count |
| **step-start** | ❌ | TODO: Step header with timing |
| **step-finish** | ❌ | TODO: Step completion status |
| **compaction** | ❌ | TODO: Compaction notice |
| **patch** | ❌ | TODO: Diff viewer component |

---

## Next Steps

1. **Review & Approval** (1 day)
   - Team review of architecture
   - Stakeholder sign-off on approach
   - Adjust if needed

2. **Phase 1 Final Deliverables** (1 day)
   - Finalize architecture doc
   - Create UI mockups (Figma or wireframes)
   - Document API integration patterns

3. **Environment Setup** (1 day)
   - Add React, Zustand, Markdown-it, Shiki to package.json
   - Update esbuild config for webview bundling
   - Create initial component scaffold

4. **Begin Phase 2** (3-4 days)
   - Implement connection discovery
   - Setup Zustand store
   - Create webview IPC bridge
   - Verify server communication works

---

## Key References

- **OpenCode Server:** `packages/opencode/src/server/server.ts`
- **Message Types:** `packages/opencode/src/session/message-v2.ts`
- **Existing VS Code Extension:** `sdks/vscode/src/extension.ts` (terminal launcher)
- **SDK:** `packages/sdk/js/` (@opencode-ai/sdk)
- **API Spec:** `packages/sdk/openapi.json`

---

## Questions?

Refer to:

- **MASTER_PLAN.md** - Detailed phase breakdown, component architecture, data flows
- **DECISIONS_AND_QA.md** - Detailed answers to all 6 key questions with code examples
- **architecture-research.md** - Analysis of existing OpenCode codebase

All docs are in `.opencode/memory/vscode-extension-plan/`
