# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **produce a comprehensive investigative analysis document** answering a series of interrelated questions about how the Calypso Reader manages "logged-out intent" — specifically, what happens to an unauthenticated user's action (such as a "like" click) as it passes through the authentication boundary, and why the intent appears to be lost once the user completes sign-up or log-in and returns to the authenticated view.

- **Category:** Create new documentation
- **Documentation type:** Technical investigation / Architecture analysis document (Q&A format with rationale)
- **Primary question set to answer:**
  - Where does a logged-out intent (e.g., a like click) go after it triggers a "requires login" decision?
  - What is the system's source of truth for this intent — in-memory Redux state, persisted storage, or a handoff token?
  - What exact condition causes the replay path to skip after the user authenticates?
  - Is the skip caused by timing, initialization order, or cleanup that clears the pending action before it can be applied?

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint (CRITICAL):** The user has explicitly stated that the repository itself should remain unchanged. Temporary scripts may be used for observation, but must be cleaned up afterward. Per the implementation rule, no existing files in the source repository may be modified.
- **Implementation rule:** Create a new markdown document named `wp-calypso_be7e5cc64162.md` placed in the `blitzy/documentation` directory. The document must comprehensively answer the questions posed in the prompt, provide thinking/rationale behind the answers, and base all answers on the code as the source of truth — no assumptions.
- **Analytical depth:** The user expects a trace through the actual code paths, identifying the exact state containers, reducers, components, and lifecycle hooks involved, and pinpointing where the breakdown occurs.
- **Tone and style:** Technical, evidence-based, citing specific file paths and line numbers from the codebase.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To answer where the logged-out intent goes, we will trace the dispatch of `registerLastActionRequiresLogin` from interaction components (`client/blocks/like-button/index.jsx`, `client/blocks/comments/comment-likes.jsx`, `client/blocks/follow-button/index.jsx`, etc.) through the Redux action/reducer chain in `client/state/reader-ui/` and into the `LayoutLoggedOut` component at `client/layout/logged-out.jsx`.
- To identify the source of truth, we will examine the `lastActionRequiresLogin` reducer in `client/state/reader-ui/reducer.js`, determine its persistence characteristics by checking for `withPersistence` or `withSchemaValidation` wrappers, and compare it against persisted reducers like `lastPath`.
- To explain why the replay path is skipped, we will trace the `onLoginSuccess` handler in `LayoutLoggedOut` (which calls `window.location.reload()`), document how the page reload destroys in-memory Redux state, and show that no corresponding consumer exists in the logged-in layout to read or replay the stored action.
- To determine the root cause category (timing, initialization order, or cleanup), we will demonstrate that it is a combination of a **persistence gap** (the reducer lacks persistence) and an **initialization order issue** (the state is cleared before any replay logic could execute, and no replay logic exists on the authenticated side).

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The `lastActionRequiresLogin` sub-reducer at `client/state/reader-ui/reducer.js:45-54` stores the intent but is not wrapped with `withPersistence`, unlike `lastPath` at line 19 — this is a critical architectural observation that must be documented.
- The `ReaderJoinConversationDialog` at `client/blocks/reader-join-conversation/dialog.jsx` coordinates the popup login flow via `useLoginWindow` (`client/data/reader/use-login-window.ts`), but the `onLoginSuccess` callback in `LayoutLoggedOut` at line 307-312 performs a full page reload without first persisting or forwarding the intent.
- There are **eight distinct action types** registered through this mechanism (`like`, `unlike`, `comment-like`, `comment-unlike`, `reply`, `comment`, `comment-submit`, `follow-site`, `follow-tag`, `sidebar-signup`, `sidebar-link`) spanning seven source files, indicating this is a cross-cutting concern rather than an isolated bug.
- The `reader/login-window` feature flag gates the popup-based auth flow in `client/reader/like-button/index.jsx:47` and `client/blocks/comments/post-comment.jsx:123`, but is absent from all static config files in `config/*.json`, suggesting runtime or server-side control.
- A Mermaid diagram illustrating the complete lifecycle from click → registration → dialog → popup → reload → state loss will be valuable for the reader's understanding.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature documentation structure in the `docs/` directory with extensive coverage of development workflows, state management patterns, and component conventions, but no existing documentation specifically addressing the Reader's logged-out intent preservation mechanism or the authentication handoff lifecycle.

- **Documentation framework:** The repository uses plain Markdown files within the `docs/` directory. There is no dedicated documentation site generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` detected at root level).
- **Relevant existing documentation found:**
  - `docs/data-persistence.md` — Describes the Redux persistence model using `withPersistence` / `withSchemaValidation` wrappers and IndexedDB storage. This is directly relevant to understanding why `lastActionRequiresLogin` is not persisted.
  - `docs/our-approach-to-data.md` — Documents Calypso's five-era state management evolution, the dual Redux + React Query model, and the convention for data flow.
  - `docs/routing.md` and `docs/isomorphic-routing.md` — Explain section routing, middleware chains, and layout composition, which are relevant to how the logged-out layout is selected.
  - `docs/modularized-state.md` — Describes the dynamically registered reducer pattern (via `registerReducer`) used by `client/state/reader-ui/init.js`.
- **In-code documentation:**
  - `client/reader/like-button/README.md` — Documents the Reader like-button as a specialization of `blocks/like-button` that "sends stats," but does not mention the logged-out intent flow.
  - `client/blocks/reader-join-conversation/dialog.jsx` — No README or dedicated documentation for the join-conversation dialog's lifecycle.
  - `client/state/reader-ui/` — Action creators have JSDoc comments explaining their purpose, but no architectural documentation exists for the `lastActionRequiresLogin` flow end-to-end.
- **Diagram tools:** Mermaid is the standard for the repository's tech spec.
- **Destination directory:** The implementation rule specifies `blitzy/documentation/` for the output document, which does not yet exist and will be created.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns employed and key directories examined:

- **Intent registration sites:** Searched for all usages of `registerLastActionRequiresLogin` across the codebase. Found dispatches in seven files:
  - `client/blocks/like-button/index.jsx` — post like/unlike
  - `client/blocks/comments/comment-likes.jsx` — comment like/unlike
  - `client/blocks/comments/post-comment.jsx` — reply to comment
  - `client/blocks/comments/form.jsx` — write/submit comment
  - `client/blocks/follow-button/index.jsx` — follow site
  - `client/blocks/reader-subscription-list-item/index.jsx` — sidebar link
  - `client/reader/stream/reader-tag-sidebar/index.jsx` — sidebar signup / follow tag
  - `client/reader/tag-stream/main.jsx` — follow tag
  - `client/reader/stream/reader-list-followed-sites/item.jsx` — follow site from list
- **Intent consumption sites:** Found `getLastActionRequiresLogin` and `clearLastActionRequiresLogin` used in:
  - `client/layout/logged-out.jsx` — the sole consumer that reads the pending action, shows the dialog, and handles login success
- **State management layer:** Examined `client/state/reader-ui/` including `action-types.js`, `actions.js`, `reducer.js`, `selectors.js`, `init.js`, and all test files.
- **Authentication popup flow:** Read `client/data/reader/use-login-window.ts` and `client/blocks/reader-join-conversation/dialog.jsx` in full.
- **Persistence analysis:** Grepped for `withPersistence` in `client/state/reader-ui/` and confirmed only `lastPath` and sidebar sub-reducers are persisted.
- **Feature flag search:** Searched for `reader/login-window` across all config files (`config/*.json`) — flag not found in any static configuration, only referenced in two code files.

### 0.2.3 Web Search Research Conducted

No external web search was required for this investigation. The questions posed are entirely answerable from the codebase itself, and the implementation rule explicitly directs that answers must be based on the code as truth. The architecture follows standard React/Redux patterns documented in the repository's own `docs/` directory.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation document must trace the logged-out intent lifecycle through the following modules and their public interfaces:

- **Module: `client/state/reader-ui/`** (State management layer)
  - Public APIs: `registerLastActionRequiresLogin(lastAction)`, `clearLastActionRequiresLogin()`, `viewStream(streamKey, path)`, `getLastActionRequiresLogin(state)`, `getLastPath(state)`
  - Current documentation: JSDoc comments on action creators; no architectural doc
  - Documentation needed: Full lifecycle explanation covering the reducer's non-persistent nature, the action shape contract, and the selector's null-safety guard

- **Module: `client/blocks/like-button/index.jsx`** (Like intent registration)
  - Public APIs: `LikeButtonContainer` component with `handleLikeToggle(liked)` method
  - Current documentation: No README in `client/blocks/like-button/`
  - Documentation needed: Explanation of the `!isLoggedIn` branch that dispatches `registerLastActionRequiresLogin({ type: 'like', siteId, postId })`

- **Module: `client/blocks/reader-join-conversation/dialog.jsx`** (Authentication gate dialog)
  - Public APIs: `ReaderJoinConversationDialog` with props `onClose`, `isVisible`, `loggedInAction`, `onLoginSuccess`
  - Current documentation: No architectural documentation for the dialog's lifecycle or its relationship to `useLoginWindow`
  - Documentation needed: The dialog's role as the bridge between intent storage and authentication, its popup-based login flow, and the `onLoginSuccess` → `window.location.reload()` path

- **Module: `client/data/reader/use-login-window.ts`** (Popup authentication hook)
  - Public APIs: `useLoginWindow({ onLoginSuccess, onWindowClose })` returning `{ login, createAccount, close }`
  - Current documentation: None
  - Documentation needed: How the `postMessage` listener detects authentication success and why control returns to the parent page, which then triggers a full reload

- **Module: `client/layout/logged-out.jsx`** (Layout consumer of pending actions)
  - Public APIs: `LayoutLoggedOut` component
  - Current documentation: None specific to intent handling
  - Documentation needed: How `loggedInAction` drives dialog visibility, the `onLoginSuccess` handler's reload behavior, and the `onClose` handler's action clearing

- **Module: `client/state/reader-ui/reducer.js`** (Reducer with persistence gap)
  - Public APIs: `lastActionRequiresLogin` reducer, `lastPath` reducer (persisted), `combinedReducer` with `withStorageKey('readerUi', ...)`
  - Current documentation: Inline comments only
  - Documentation needed: Explicit comparison of `lastActionRequiresLogin` (not persisted) vs. `lastPath` (wrapped with `withPersistence`), explaining the architectural consequence

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the critical documentation gap is:

- **No end-to-end documentation** exists for the logged-out intent flow from click → Redux registration → dialog display → popup authentication → page reload → intent loss.
- **No documentation** explains why `lastActionRequiresLogin` is not persisted while other reader-ui state slices are.
- **No documentation** describes the layout swap from `LayoutLoggedOut` to the logged-in layout and its effect on pending actions.
- **No documentation** catalogs the full set of action types registered through `registerLastActionRequiresLogin` across the eight dispatching components.
- **The `reader/login-window` feature flag** is undocumented — its absence from config files and its runtime behavior are not explained anywhere.
- **No replay mechanism documentation** exists because no replay mechanism exists in code — this gap in the code is itself undocumented.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be a single Markdown file placed at `blitzy/documentation/wp-calypso_be7e5cc64162.md`, structured as a comprehensive technical Q&A with full rationale. The planned hierarchy:

```
blitzy/
└── documentation/
    └── wp-calypso_be7e5cc64162.md
        ├── Overview (question restatement and executive answer)
        ├── The Intent Registration Mechanism
        │   ├── Where the click goes: registerLastActionRequiresLogin
        │   ├── The action shape contract (all action types)
        │   └── The Redux state location and reducer behavior
        ├── The Source of Truth
        │   ├── In-memory Redux state (confirmed)
        │   ├── Persistence analysis (not persisted)
        │   └── Comparison with persisted slices (lastPath, sidebar)
        ├── The Authentication Boundary
        │   ├── ReaderJoinConversationDialog lifecycle
        │   ├── useLoginWindow popup flow
        │   ├── postMessage handshake
        │   └── The onLoginSuccess handler
        ├── Where the Intent Is Lost
        │   ├── window.location.reload() destroys state
        │   ├── Layout swap: LayoutLoggedOut → logged-in layout
        │   ├── No consumer on the authenticated side
        │   └── The clearLastActionRequiresLogin on dialog close
        ├── Root Cause Analysis
        │   ├── Persistence gap (reducer not wrapped)
        │   ├── Initialization order (state cleared before replay)
        │   ├── Absence of replay mechanism
        │   └── Feature flag ambiguity (reader/login-window)
        ├── Lifecycle Diagram (Mermaid)
        └── File Reference Index
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract the intent registration flow by tracing `registerLastActionRequiresLogin` dispatches from `client/blocks/like-button/index.jsx:34`, `client/blocks/comments/comment-likes.jsx:23`, and all other dispatch sites
  - Extract the reducer behavior from `client/state/reader-ui/reducer.js:45-54`, comparing against `lastPath` at line 19
  - Extract the dialog lifecycle from `client/blocks/reader-join-conversation/dialog.jsx` and `client/layout/logged-out.jsx:302-315`
  - Extract the popup authentication flow from `client/data/reader/use-login-window.ts:52-78`
  - Confirm persistence behavior by cross-referencing `docs/data-persistence.md` against the actual `withPersistence` usage in `client/state/reader-ui/`

- **Documentation Standards:**
  - Markdown formatting with proper heading hierarchy (`# ## ### ####`)
  - Mermaid diagrams for the lifecycle flow and state flow
  - Code path citations using format: `Source: /path/to/file.py:LineNumber`
  - Tables for the action type catalog and file reference index
  - Evidence-based narrative: every claim references a specific file and line

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Intent Lifecycle Flowchart:** A complete flow from user click → `registerLastActionRequiresLogin` → Redux store → `LayoutLoggedOut` → `ReaderJoinConversationDialog` → `useLoginWindow` popup → `postMessage` success → `window.location.reload()` → state loss. This will visually answer where the intent goes and where it is lost.
- **State Persistence Comparison Table:** A visual comparison of which `reader-ui` sub-reducers are persisted vs. not, highlighting `lastActionRequiresLogin` as the gap.
- **Sequence Diagram:** Showing the temporal interaction between the parent page, the popup window, the `postMessage` event, the `onLoginSuccess` callback, and the page reload, to illustrate the timing that causes the intent to be destroyed.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `client/state/reader-ui/actions.js`, `client/state/reader-ui/reducer.js`, `client/state/reader-ui/selectors.js`, `client/state/reader-ui/action-types.js`, `client/blocks/like-button/index.jsx`, `client/blocks/reader-join-conversation/dialog.jsx`, `client/data/reader/use-login-window.ts`, `client/layout/logged-out.jsx`, `client/reader/like-button/index.jsx`, `client/blocks/comments/comment-likes.jsx`, `client/blocks/comments/post-comment.jsx`, `client/blocks/comments/form.jsx`, `client/blocks/follow-button/index.jsx`, `client/blocks/reader-subscription-list-item/index.jsx`, `client/reader/stream/reader-tag-sidebar/index.jsx`, `client/reader/tag-stream/main.jsx`, `client/reader/stream/reader-list-followed-sites/item.jsx`, `client/state/reader-ui/test/actions.js`, `client/state/reader-ui/test/reducer.js`, `client/state/reader-ui/test/selectors.js`, `docs/data-persistence.md` | Complete investigative Q&A document answering how logged-out intent is registered, stored, consumed, and ultimately lost during the authentication boundary crossing in the Reader. Includes Mermaid lifecycle diagram, state persistence comparison, action-type catalog, and file reference index. |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Technical Investigation / Architecture Analysis (Q&A)
Source Code:
    - client/state/reader-ui/actions.js (intent registration action creators)
    - client/state/reader-ui/reducer.js (non-persisted reducer for lastActionRequiresLogin)
    - client/state/reader-ui/selectors.js (selector with null-safety guard)
    - client/state/reader-ui/action-types.js (Redux action type constants)
    - client/blocks/like-button/index.jsx (like intent dispatch at line 34)
    - client/blocks/reader-join-conversation/dialog.jsx (auth gate dialog)
    - client/data/reader/use-login-window.ts (popup login hook)
    - client/layout/logged-out.jsx (intent consumer and onLoginSuccess at line 307)
    - client/reader/like-button/index.jsx (Reader like-button with feature flag check)
    - client/blocks/comments/comment-likes.jsx (comment-like intent dispatch)
    - client/blocks/comments/post-comment.jsx (reply intent dispatch)
    - client/blocks/comments/form.jsx (comment/comment-submit intent dispatch)
    - client/blocks/follow-button/index.jsx (follow-site intent dispatch)
    - client/blocks/reader-subscription-list-item/index.jsx (sidebar-link intent dispatch)
    - client/reader/stream/reader-tag-sidebar/index.jsx (sidebar-signup intent dispatch)
    - client/reader/tag-stream/main.jsx (follow-tag intent dispatch)
    - client/reader/stream/reader-list-followed-sites/item.jsx (follow-site intent dispatch)
Sections:
    - Overview and Executive Answer
    - The Intent Registration Mechanism (registerLastActionRequiresLogin)
    - The Complete Action-Type Catalog (all action shapes)
    - The Source of Truth (in-memory Redux, not persisted)
    - The Authentication Boundary (dialog → popup → postMessage)
    - Where the Intent Is Lost (reload, layout swap, no replay)
    - Root Cause Analysis (persistence gap, initialization order, no replay code)
    - The Feature Flag Ambiguity (reader/login-window)
    - Intent Lifecycle Diagram (Mermaid flowchart)
    - Authentication Sequence Diagram (Mermaid sequence)
    - File Reference Index (table of all files examined)
Diagrams:
    - Mermaid flowchart: Complete intent lifecycle from click to loss
    - Mermaid sequence diagram: Popup authentication timing
    - State persistence comparison table
Key Citations:
    - client/state/reader-ui/reducer.js:19 (withPersistence on lastPath)
    - client/state/reader-ui/reducer.js:45-54 (lastActionRequiresLogin without persistence)
    - client/layout/logged-out.jsx:91 (getLastActionRequiresLogin selector read)
    - client/layout/logged-out.jsx:302-315 (ReaderJoinConversationDialog rendering)
    - client/layout/logged-out.jsx:307-312 (onLoginSuccess handler with reload)
    - client/data/reader/use-login-window.ts:52-60 (postMessage listener)
    - client/blocks/like-button/index.jsx:32-38 (like intent dispatch)
    - docs/data-persistence.md (persistence architecture explanation)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone Markdown document placed in the `blitzy/documentation/` directory per the implementation rule. There are no documentation generators, navigation configs, or build scripts to update.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tooling is required for this task. The output is a plain Markdown file (`.md`) that requires no build step, no documentation generator, and no additional packages.

The following project dependencies are relevant to the investigation as they form the runtime stack through which the logged-out intent flows:

| Registry | Package Name | Version | Relevance to Investigation |
|----------|-------------|---------|---------------------------|
| npm (workspace) | react | ^18.3.1 | Component lifecycle (useEffect, useState) in dialog and hook |
| npm | redux | ^5.0.1 | Redux store where `lastActionRequiresLogin` is held in memory |
| npm (workspace) | react-redux | workspace dep | `connect`, `useSelector`, `useDispatch` for state access |
| npm (workspace) | @automattic/calypso-config | workspace dep | Feature flag check for `reader/login-window` |
| npm (workspace) | @automattic/calypso-analytics | workspace dep | Event tracking in dialog (`recordTracksEvent`) |
| npm (workspace) | @automattic/components | workspace dep | `Dialog` component for the join-conversation modal |
| npm (workspace) | @wordpress/components | ^29.7.0 | `Button` component used in dialog UI |
| npm (workspace) | i18n-calypso | workspace dep | Translation hook (`useTranslate`) in dialog |
| npm (workspace) | @automattic/state-utils | workspace dep | `withStorageKey` used in reducer composition |
| npm (workspace) | calypso/state/utils | internal | `combineReducers`, `withPersistence` for state persistence |

### 0.6.2 Runtime and Toolchain Versions

| Tool | Version | Source |
|------|---------|--------|
| Node.js | 22.9.0 | `.nvmrc` and `package.json` engines field |
| Yarn | ^4.0.0 | `package.json` engines field |
| Jest | ^29.7.0 | `package.json` devDependencies (for test reference) |

### 0.6.3 Documentation Reference Updates

Not applicable. This is a new standalone document and no existing documentation files require link updates.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **Questions posed by the user:** 4 primary questions, each with sub-dimensions
  - Where does the logged-out intent go? → **Must be answered**
  - What is the source of truth (memory, persisted, handoff token)? → **Must be answered**
  - What exact condition causes the replay path to skip? → **Must be answered**
  - Is the skip caused by timing, initialization order, or cleanup? → **Must be answered**
- **Code paths requiring documentation coverage:**
  - Intent registration dispatch sites: 9 files identified → **100% coverage target**
  - State management layer (actions, reducer, selectors, action-types, init): 5 files → **100% coverage target**
  - Authentication dialog and popup flow: 2 files → **100% coverage target**
  - Layout consumer: 1 file → **100% coverage target**
  - Persistence comparison: `reducer.js` + `docs/data-persistence.md` → **100% coverage target**
  - Test files validating the state contract: 3 files → **Referenced for evidence**
- **Action types documented:** All action shapes dispatched through `registerLastActionRequiresLogin` must be cataloged — 10+ distinct types across 9 files → **100% coverage target**

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every question posed by the user must receive a direct, evidence-based answer
  - All answers must cite specific file paths and line numbers
  - The full lifecycle from click to intent loss must be traced without gaps
  - All action types registered through the mechanism must be cataloged with their source file
  - The persistence gap must be explained with a direct comparison to persisted reducers

- **Accuracy validation:**
  - Every claim must reference a specific file and line in the codebase
  - Code flow descriptions must match the actual branching logic in the source
  - The reducer persistence analysis must be verified by the presence/absence of `withPersistence` wrappers
  - The feature flag behavior must reflect the actual config file state (absent from all static configs)

- **Clarity standards:**
  - Begin with a concise executive answer before diving into details
  - Use Mermaid diagrams to illustrate the lifecycle and timing
  - Organize answers in the order the user asked them
  - Progressive disclosure: summary → mechanism → evidence → root cause

- **Maintainability:**
  - All source citations include file paths for traceability
  - Diagrams use stable component names from the codebase
  - The document can be updated if the underlying code changes

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 2 Mermaid diagrams (lifecycle flowchart + authentication sequence)
- **Tables required:** Action-type catalog table, persistence comparison table, file reference table
- **Code snippet references:** Inline citations to specific lines (e.g., `client/state/reader-ui/reducer.js:45-54`) rather than full code blocks, to maintain focus on the narrative

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/wp-calypso_be7e5cc64162.md` — The complete investigative analysis document

- **Code modules to analyze and document (read-only, no modifications):**
  - `client/state/reader-ui/actions.js` — Action creators for intent registration and clearing
  - `client/state/reader-ui/action-types.js` — Redux action type constants
  - `client/state/reader-ui/reducer.js` — Reducer with non-persisted `lastActionRequiresLogin`
  - `client/state/reader-ui/selectors.js` — Selector for reading the pending action
  - `client/state/reader-ui/init.js` — Store registration via `registerReducer`
  - `client/blocks/like-button/index.jsx` — Like intent dispatch (primary example)
  - `client/blocks/comments/comment-likes.jsx` — Comment-like intent dispatch
  - `client/blocks/comments/post-comment.jsx` — Reply intent dispatch
  - `client/blocks/comments/form.jsx` — Comment/comment-submit intent dispatch
  - `client/blocks/follow-button/index.jsx` — Follow-site intent dispatch
  - `client/blocks/reader-subscription-list-item/index.jsx` — Sidebar-link intent dispatch
  - `client/reader/stream/reader-tag-sidebar/index.jsx` — Sidebar-signup intent dispatch
  - `client/reader/tag-stream/main.jsx` — Follow-tag intent dispatch
  - `client/reader/stream/reader-list-followed-sites/item.jsx` — Follow-site intent dispatch
  - `client/reader/like-button/index.jsx` — Reader-specific like-button with feature flag check
  - `client/blocks/reader-join-conversation/dialog.jsx` — Authentication gate dialog
  - `client/data/reader/use-login-window.ts` — Popup login hook
  - `client/layout/logged-out.jsx` — Layout consumer and onLoginSuccess handler
  - `client/state/reader-ui/test/*.js` — Test files validating the state contract
  - `docs/data-persistence.md` — Persistence architecture reference

- **Analysis scope:**
  - Complete lifecycle trace of `registerLastActionRequiresLogin` → Redux store → dialog → authentication → page reload → intent loss
  - Persistence comparison across all `reader-ui` sub-reducers
  - Feature flag status determination for `reader/login-window`
  - All action-type shapes registered through the mechanism
  - The `onLoginSuccess` and `onClose` handler behavior in `LayoutLoggedOut`

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** The repository must remain unchanged per user instruction. No existing files will be modified.
- **Bug fixes or patches:** This is a documentation/analysis task, not an implementation task. No code changes to fix the intent loss are in scope.
- **Test file modifications:** No test files will be created or changed.
- **Other Reader features:** Reader stream loading, card expansion, sidebar state, unseen posts — these are unrelated to the authentication intent flow.
- **Non-Reader authentication flows:** Desktop login, OAuth token capture, Jetpack social auth, magic link login — these are separate authentication paths not involved in the Reader logged-out intent mechanism.
- **Documentation for other sections:** Only the `blitzy/documentation/wp-calypso_be7e5cc64162.md` file is in scope. Existing docs in `docs/` will not be modified.
- **Deployment or configuration changes:** No changes to feature flags, config files, or infrastructure.
- **Temporary observation scripts:** The user mentions these may be used but must be cleaned up. The Agent Action Plan does not plan for any temporary scripts as the code analysis is completed through repository inspection tools.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file requiring no build step.
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/wp-calypso_be7e5cc64162.md`
- **Diagram generation:** Mermaid diagrams are embedded inline within the Markdown file using fenced mermaid code blocks. They render in any Mermaid-compatible Markdown viewer (GitHub, VS Code with Mermaid extension, etc.).
- **Default format:** Markdown (`.md`) with embedded Mermaid diagrams and standard GitHub-flavored Markdown tables.
- **Citation requirement:** Every technical claim must reference its source file path and, where relevant, line numbers. Format: `Source: client/path/to/file.ext:LineNumber`
- **Style guide:** The document follows the repository's existing documentation tone (technical, precise, developer-facing) as observed in `docs/data-persistence.md` and `docs/our-approach-to-data.md`.
- **Documentation validation:** The Markdown file must be syntactically valid — all code fences closed, all table columns aligned, all Mermaid blocks parseable.

### 0.9.2 Directory Setup

The `blitzy/documentation/` directory does not currently exist in the repository. It must be created as part of the documentation task before the output file can be placed. This is the only filesystem operation required beyond writing the document itself.

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

- **Do not modify any existing files in the source repository.** The document is a read-only investigation. All source files are examined but never changed.
- **Do not make assumptions; base answers on the code as the truth.** Every conclusion must be traceable to a specific file, line, or structural observation in the codebase. Speculation about what the code "might" do is not permitted — only what it demonstrably does.
- **Provide thinking and rationale behind the answers.** The document must not merely state conclusions but explain the reasoning chain: what was observed, what it implies, and why the conclusion follows.
- **Create the output document at `blitzy/documentation/wp-calypso_be7e5cc64162.md`.** The filename matches the source branch name as required by the implementation rule.
- **Temporary scripts may be used for observation but must be cleaned up afterward.** In practice, no temporary scripts are needed — the investigation is conducted entirely through static code analysis via repository inspection tools.

### 0.10.2 Derived Documentation Standards

- **Evidence chain for every claim:** Each answer section must follow the pattern: Observation (what the code does) → Implication (what that means for the intent lifecycle) → Conclusion (why the intent is lost / preserved / skipped).
- **No placeholder content:** The document must be complete in a single pass. No sections marked as "TODO" or "to be investigated later."
- **Mermaid diagrams for complex flows:** The authentication popup lifecycle and the intent registration flow are sufficiently complex to warrant visual representation.
- **Consistent citation format:** `Source: path/to/file.ext:LineNumber` for specific lines, `Source: path/to/file.ext` for general file references.
- **Action-type catalog completeness:** Every distinct action type dispatched through `registerLastActionRequiresLogin` must be listed with its source file and the shape of the action object.

## 0.11 References

### 0.11.1 Files and Folders Searched

The following files and folders were searched, retrieved, and analyzed to derive the conclusions in this Agent Action Plan:

**State Management Layer (Intent Storage)**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `client/state/reader-ui/actions.js` | Defines `registerLastActionRequiresLogin` and `clearLastActionRequiresLogin` action creators |
| `client/state/reader-ui/action-types.js` | Defines `READER_REGISTER_LAST_ACTION_REQUIRES_LOGIN` and `READER_CLEAR_LAST_ACTION_REQUIRES_LOGIN` constants |
| `client/state/reader-ui/reducer.js` | Contains the `lastActionRequiresLogin` reducer (NOT persisted) and `lastPath` reducer (persisted with `withPersistence`) |
| `client/state/reader-ui/selectors.js` | Defines `getLastActionRequiresLogin` selector with null-safety guard |
| `client/state/reader-ui/init.js` | Registers the `readerUi` reducer key with the Redux store via `registerReducer` |
| `client/state/reader-ui/package.json` | Contains `sideEffects` declaration protecting `init.js` from tree-shaking |
| `client/state/reader-ui/test/actions.js` | Test file validating action creator shape with `{ type: 'like', siteId: 123, postId: 456 }` fixture |
| `client/state/reader-ui/test/reducer.js` | Test file validating reducer stores and clears `lastAction` correctly |
| `client/state/reader-ui/test/selectors.js` | Test file validating selector returns null for empty/missing state |

**Intent Registration Dispatch Sites**

| File Path | Action Type(s) Registered |
|-----------|--------------------------|
| `client/blocks/like-button/index.jsx` | `like`, `unlike` |
| `client/blocks/comments/comment-likes.jsx` | `comment-like`, `comment-unlike` |
| `client/blocks/comments/post-comment.jsx` | `reply` |
| `client/blocks/comments/form.jsx` | `comment`, `comment-submit` |
| `client/blocks/follow-button/index.jsx` | `follow-site` |
| `client/blocks/reader-subscription-list-item/index.jsx` | `sidebar-link` (with `redirectTo`) |
| `client/reader/stream/reader-tag-sidebar/index.jsx` | `sidebar-signup` |
| `client/reader/tag-stream/main.jsx` | `follow-tag` |
| `client/reader/stream/reader-list-followed-sites/item.jsx` | `follow-site` |

**Authentication Boundary Components**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `client/blocks/reader-join-conversation/dialog.jsx` | The authentication gate dialog that shows login/signup options and coordinates with `useLoginWindow` |
| `client/data/reader/use-login-window.ts` | The popup login hook that opens a browser popup, listens for `postMessage`, and triggers `onLoginSuccess` |
| `client/layout/logged-out.jsx` | The logged-out layout that reads `getLastActionRequiresLogin`, renders the dialog, and handles `onLoginSuccess` with `window.location.reload()` |
| `client/reader/like-button/index.jsx` | Reader-specific like-button wrapper with `reader/login-window` feature flag check |

**Reference Documentation**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `docs/data-persistence.md` | Explains the `withPersistence` / `withSchemaValidation` model for Redux state persistence to IndexedDB |
| `docs/our-approach-to-data.md` | Documents the dual Redux + React Query state management architecture |
| `docs/modularized-state.md` | Describes dynamic reducer registration via `registerReducer` |

**Configuration and Infrastructure**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `package.json` | Root monorepo manifest — engines (Node 22.9.0, Yarn 4), dependencies |
| `.nvmrc` | Confirms Node.js version 22.9.0 |
| `config/*.json` | Searched for `reader/login-window` feature flag — NOT found in any static config file |

**Folders Explored**

| Folder Path | Purpose |
|-------------|---------|
| `` (root) | Repository structure assessment |
| `client/state/reader-ui/` | Complete state management layer for Reader UI |
| `client/blocks/reader-join-conversation/` | Authentication dialog block |
| `client/reader/like-button/` | Reader-specific like-button specialization |
| `client/blocks/reader-post-actions/` | Reader post actions bar (like, comment, share) |
| `client/reader/full-post/` | Reader full-post route and controller |
| `docs/` | Existing documentation hub |

### 0.11.2 Attachments and External Resources

- **Attachments:** No attachments were provided by the user.
- **Figma URLs:** None provided.
- **External URLs:** None required — all analysis is based on codebase inspection.

### 0.11.3 Tech Spec Sections Referenced

The following sections from the existing Technical Specification document were retrieved and used for context:

| Section | Relevance |
|---------|-----------|
| 1.1 Executive Summary | Project overview and architecture context (Calypso as WordPress.com SPA) |
| 4.3 Authentication Workflows | Multi-method authentication flow, session management, redirect guards |
| 4.4 Root Landing Page Decision Tree | Route resolution logic showing Reader as a landing page option |
| 4.9 State Management Lifecycle | Redux initialization, hydration, persistence hooks, and the dual-model data flow |
| 4.12 Error Handling and Recovery Flows | Error monitoring patterns for context on how other intents (e.g., payment) are preserved |

