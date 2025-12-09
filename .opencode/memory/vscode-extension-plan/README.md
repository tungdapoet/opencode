# VS Code Extension Planning - Complete Documentation Index

## 📋 Document Overview

This planning package contains **4 comprehensive documents** that form a complete blueprint for building a VS Code Extension for OpenCode using the headless server approach.

### 1. **EXECUTIVE_SUMMARY.md** ⭐ START HERE

**Purpose:** One-page overview for leadership/stakeholders
**Contains:**

- Big picture architecture
- Tech stack summary
- Key decisions table
- Timeline & effort estimates
- Next steps
- Quick reference links

**Read this if:** You need to understand the plan in 5-10 minutes

---

### 2. **MASTER_PLAN.md** 📖 DETAILED ROADMAP

**Purpose:** Complete phase-by-phase breakdown
**Contains:**

- Executive summary
- 6 phases with detailed tasks (18-23 days total)
- Phase 1 (Design) - COMPLETE ✅
- Phase 2 (Infrastructure) - 3-4 days
- Phase 3 (MVP UI) - 4-5 days
- Phase 4 (Streaming) - 3-4 days
- Phase 5 (Advanced) - 3-4 days
- Phase 6 (Polish) - 2-3 days
- Technical decisions & rationale
- Component architecture (file structure)
- Data flow diagrams
- Risk matrix
- Success criteria
- Timeline estimates

**Read this if:** You're implementing the plan or presenting to the team

---

### 3. **DECISIONS_AND_QA.md** 🎯 ANSWERS TO KEY QUESTIONS

**Purpose:** Deep-dive answers to the 6 critical questions with code examples
**Contains:**

- Q1: Server discovery/connection (3-tier fallback strategy)
- Q2: WebSocket vs SSE (SSE decision with rationale)
- Q3: State management (Zustand with split stores)
- Q4: Markdown rendering (Markdown-it + Shiki)
- Q5: Permission dialog UX (modal + toast pattern)
- Q6: File system integration (open, sync, patch)
- Code examples for each
- Summary decision table

**Read this if:** You're implementing a specific component or need justification

---

### 4. **PHASE2_CHECKLIST.md** ✅ IMPLEMENTATION GUIDE

**Purpose:** Detailed task breakdown for Phase 2 (Core Infrastructure)
**Contains:**

- Pre-implementation setup
- Task 1: Server connection (discovery, manager, status bar)
- Task 2: SDK client wrapper
- Task 3: State management (Zustand store + persistence)
- Task 4: IPC bridge (protocol + bridge implementation)
- Task 5: React scaffold (entry point, root, styling)
- Task 6: Extension entry point
- Per-task checklists
- Testing checklist
- Deliverables summary

**Read this if:** You're starting implementation (especially Phase 2)

---

### 5. **architecture-research.md** 🔬 CODEBASE ANALYSIS

**Purpose:** Analysis of existing OpenCode architecture
**Contains:**

- Current VS Code extension role
- Server architecture (Hono, SSE, WebSocket)
- Message & Part types (16 types, streaming states)
- SDK integration details
- API routes (complete list)
- Key technical decisions found in codebase
- Constraints identified

**Read this if:** You need to understand OpenCode's internals

---

## 🎬 How to Use This Package

### For Project Managers

1. Read **EXECUTIVE_SUMMARY.md**
2. Share timeline & effort estimates with stakeholders
3. Use **MASTER_PLAN.md** Phase overview for tracking
4. Reference risk matrix for risk management

### For Architects/Tech Leads

1. Read **EXECUTIVE_SUMMARY.md** (overview)
2. Deep-dive **MASTER_PLAN.md** (all phases)
3. Review **DECISIONS_AND_QA.md** (rationale)
4. Check **architecture-research.md** (codebase fit)
5. Present to team + get buy-in

### For Developers (Starting Phase 2)

1. Skim **EXECUTIVE_SUMMARY.md** (context)
2. Read relevant Phase in **MASTER_PLAN.md**
3. Open **PHASE2_CHECKLIST.md** for implementation
4. Reference **DECISIONS_AND_QA.md** as needed for details
5. Review **architecture-research.md** for API details

### For Developers (Future Phases)

1. Refer to relevant Phase in **MASTER_PLAN.md**
2. Use **DECISIONS_AND_QA.md** for component patterns
3. Check architecture-research.md for API contracts

---

## 📊 Quick Stats

| Metric               | Value                                                         |
| -------------------- | ------------------------------------------------------------- |
| Total Duration       | **22-28 days** (revised)                                      |
| Single Dev Timeline  | ~28 days                                                      |
| Two Dev Timeline     | ~18 days (parallel)                                           |
| Number of Phases     | **7** (includes Phase 1.5)                                    |
| Component Categories | 8 (sidebar, chat, parts, tools, modals, common, hooks, utils) |
| Estimated Components | 25-30                                                         |
| Part Types Supported | 16                                                            |
| API Endpoints Used   | 15                                                            |
| Tech Stack Items     | 8 (React, Zustand, TypeScript, etc.)                          |

> ⚠️ **Revised post-review:** Timeline extended by 4-5 days to include critical prerequisites (Phase 1.5)

---

## 🏗️ Architecture at a Glance

```
VS Code Window
├── Extension Host (TypeScript)
│   ├── Connection Manager
│   ├── SDK Client Wrapper
│   ├── Zustand Store (global state)
│   ├── Event Listeners
│   ├── SSE Consumer
│   └── Webview Manager
│
└── Webview Panel (React App)
    ├── Components
    │   ├── Sidebar (Projects, Sessions)
    │   ├── ChatView (Messages, Input)
    │   ├── Parts (Text, Tool, File, etc.)
    │   ├── Modals (Permissions, Create Session, etc.)
    │   └── Common (Status, Toast, Loading)
    ├── State Management
    │   └── Zustand Store Consumer (via hooks)
    └── Styles (CSS Modules, theme tokens)

↓ Communication (IPC)

OpenCode Server (Hono)
├── /project - Project management
├── /session - Session management
├── /message - Message streaming (SSE)
├── /permission - Permission requests
├── /global/event - Global event stream (SSE)
└── File storage & VCS integration
```

---

## 🚀 Getting Started Checklist

- [ ] **Read EXECUTIVE_SUMMARY.md** (5 min)
- [ ] **Review MASTER_PLAN.md phases** (20 min)
- [ ] **Schedule team sync** to discuss DECISIONS_AND_QA.md (30 min)
- [ ] **Get stakeholder buy-in** on timeline & scope
- [ ] **Setup repository branch** for vscode-extension work
- [ ] **Configure development environment:**
  - [ ] Bun runtime (already installed)
  - [ ] VS Code + ESLint extension
  - [ ] TypeScript support
  - [ ] React DevTools browser extension (for testing)
- [ ] **Review existing codebase:**
  - [ ] Current extension: `sdks/vscode/src/extension.ts`
  - [ ] Server: `packages/opencode/src/server/server.ts`
  - [ ] SDK: `packages/sdk/js/`
  - [ ] Message types: `packages/opencode/src/session/message-v2.ts`
- [ ] **Kick off Phase 2** using PHASE2_CHECKLIST.md

---

## 📁 Where to Find Everything

**In VS Code repository:**

```
sdks/vscode/                    ← Extension location
├── src/extension.ts            ← Current entry point (to update)
├── package.json                ← Update with React deps
├── esbuild.js                  ← Update build config
└── [NEW] src/                  ← Create new structure:
    ├── server/
    ├── sdk/
    ├── streaming/
    ├── state/
    ├── messaging/
    ├── webview/
    └── utils/
```

**Documentation:**

```
.opencode/memory/vscode-extension-plan/
├── EXECUTIVE_SUMMARY.md        ← Start here
├── MASTER_PLAN.md              ← Full roadmap
├── DECISIONS_AND_QA.md         ← Technical deep-dive
├── PHASE2_CHECKLIST.md         ← Implementation guide
└── architecture-research.md    ← Codebase analysis
```

---

## 🎯 Success Metrics

### Phase 1: Design ✅

- [x] Architecture document complete
- [x] UI specs defined
- [x] API strategy documented
- [x] Technical decisions made

### Phase 2: Infrastructure

- [ ] Extension loads and connects
- [ ] Status bar shows state
- [ ] Webview renders with skeleton
- [ ] Build succeeds with no errors

### Phase 3: MVP UI

- [ ] Projects/sessions display
- [ ] Messages render with all types
- [ ] Input sends messages
- [ ] Tool execution shows in UI

### Phase 4: Streaming

- [ ] Parts stream in real-time
- [ ] Permissions dialog works
- [ ] Session actions functional

### Phase 5: Advanced

- [ ] Search across messages
- [ ] Snapshots + revert work
- [ ] Diffs display
- [ ] Cost tracking shown

### Phase 6: Polish

- [ ] All tests pass
- [ ] Keyboard shortcuts work
- [ ] Extension packaged
- [ ] Docs complete

---

## 🔗 Key Links & References

**OpenCode Codebase:**

- Server: `packages/opencode/src/server/server.ts`
- Messages: `packages/opencode/src/session/message-v2.ts`
- SDK: `packages/sdk/js/`
- API Spec: `packages/sdk/openapi.json`
- Existing ext: `sdks/vscode/src/extension.ts`

**External Resources:**

- Zustand docs: https://github.com/pmndrs/zustand
- Markdown-it: https://markdown-it.github.io/
- Shiki: https://shiki.style/
- VS Code API: https://code.visualstudio.com/api/references/vscode-api
- React: https://react.dev/

---

## 💬 Questions & Support

**Technical questions:** Refer to DECISIONS_AND_QA.md
**Implementation questions:** Refer to PHASE2_CHECKLIST.md
**Architecture questions:** Refer to MASTER_PLAN.md
**Codebase questions:** Refer to architecture-research.md

---

## 📝 Document Maintenance

**Last Updated:** December 2024
**Status:** Phase 1 Complete, Ready for Phase 2
**Author:** Planning Agent

**To update:**

1. For architectural changes → update MASTER_PLAN.md
2. For new decisions → update DECISIONS_AND_QA.md
3. For implementation changes → update PHASE2_CHECKLIST.md
4. Always update EXECUTIVE_SUMMARY.md for consistency

---

## Version History

| Version | Date     | Phase              | Status      |
| ------- | -------- | ------------------ | ----------- |
| 1.0     | Dec 2024 | Design (1)         | ✅ Complete |
| 2.0     | TBD      | Infrastructure (2) | Pending     |
| 3.0     | TBD      | MVP UI (3)         | Pending     |
| 4.0     | TBD      | Streaming (4)      | Pending     |
| 5.0     | TBD      | Advanced (5)       | Pending     |
| 6.0     | TBD      | Polish (6)         | Pending     |

---

**Ready to build! Follow the Getting Started Checklist above and reference the appropriate documents as you implement each phase.**
