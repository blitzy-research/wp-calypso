# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new onboarding Q&A document** that comprehensively answers five interconnected clusters of questions about the Calypso codebase's Reader section, grounded entirely in evidence from the source code. The request is categorized as:

- **Category:** Create new documentation
- **Documentation Type:** Onboarding Q&A / Architecture walkthrough (investigative reference for a developer new to the codebase)

The user is onboarding on the wp-calypso monorepo and seeks code-backed answers to the following questions:

- **Development Server Binding:** What port does the development server bind to, and what signals indicate readiness? (Answered from `config/development.json`, `client/server/index.js`, `client/server/bundler/index.js`)
- **Multi-Port Architecture:** Does the architecture use separate ports for HMR, API calls, and asset serving, or is everything unified? (Answered from the bundler middleware and Express boot pipeline)
- **Reader Stream API Endpoints:** What REST API endpoints populate the Reader "Following" stream during initial load? (Answered from `client/state/data-layer/wpcom/read/streams/index.js` and its `streamApis` configuration table)
- **Redux Actions During Initial Load:** Which Redux actions fire when the Reader section bootstraps and fetches its first page? (Answered from `client/state/reader/action-types.ts`, `client/state/reader/streams/actions.js`, and the route middleware chain in `client/reader/index.ts`)
- **Authentication / Login Detection:** How does the app determine whether a user is logged in before deciding what to render? What storage mechanisms does it check? (Answered from `client/state/current-user/selectors.js`, `client/lib/user/store.js`, `client/lib/user/shared-utils/initialize-current-user.js`, and `client/server/user-bootstrap/index.js`)
- **Sidebar Responsive Design:** What specific margin/padding values, CSS custom properties, and viewport breakpoints drive the Reader sidebar layout? (Answered from `client/reader/sidebar/style.scss`, `client/reader/style.scss`, and `client/assets/stylesheets/shared/_variables.scss`)

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No repository modification.** The user explicitly stated: "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." The generated document must be placed in `blitzy/documentation/` per the implementation rules, which is a net-new directory in the destination repo—not a modification to existing source.
- **Implementation Rule Compliance:** The file must be named `wp-calypso_be7e5cc64162.md` (the source branch name), placed in `blitzy/documentation/`.
- **Evidence-Based Answers:** The user requested "Do not make assumptions, base your answers on the code as the truth."
- **Provide Rationale:** "Provide thinking / rationale behind the answers."
- **Do not modify existing files:** Only a new markdown file is created.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the development server port and readiness**, we will extract configuration values from `config/development.json` (port: `3000`, hostname: `calypso.localhost`, protocol: `http`) and trace the boot lifecycle through `client/server/index.js` → `server.listen()` → `sendBootStatus('ready')` and `client/server/bundler/index.js` → `compiler.hooks.done` → console `Ready!` message.
- To **document the multi-port architecture**, we will analyze how `webpack-dev-middleware` and `webpack-hot-middleware` are mounted on the same Express app in `client/server/bundler/index.js`, confirming a single-port architecture where all traffic (HTML, compiled assets, HMR WebSocket upgrades, and proxied API calls) flows through port 3000.
- To **document Reader stream API endpoints**, we will extract the `streamApis` configuration table from `client/state/data-layer/wpcom/read/streams/index.js`, documenting every stream type, its REST path, date property, API version, and query parameters.
- To **document Redux actions during Reader initial load**, we will trace the middleware chain from `client/reader/index.ts` through the controller functions in `client/reader/controller.js`, identifying the actions from `client/state/reader/action-types.ts` that fire during the stream page request lifecycle.
- To **document authentication detection**, we will trace the full login-state resolution path from `client/boot/common.js` → `initializeCurrentUser()` → `window.currentUser` bootstrap / `/me` fetch fallback, through `client/state/current-user/selectors.js` → `isUserLoggedIn()`, and identify the storage mechanisms: `localStorage` (`wpcom_user_id` via the `store` library), `IndexedDB` (persisted Redux state), `window.currentUser` (SSR bootstrap), and the `wordpress_logged_in` cookie (server-side bootstrap).
- To **document the sidebar responsive design**, we will catalog the CSS custom properties (`--masterbar-height`, `--sidebar-width-max`, `--content-padding-top`, `--content-padding-bottom`) from `client/assets/stylesheets/shared/_variables.scss` and `client/my-sites/sidebar/style.scss`, then document the specific pixel values, media query breakpoints (600px, 781px, 782px, 1300px), and `calc()` expressions from `client/reader/sidebar/style.scss` and `client/reader/style.scss`.

### 0.1.4 Inferred Documentation Needs

- **Boot lifecycle sequencing:** The `yarn start` flow involves multiple coordinated steps (webpack compile → bundler readiness gate → Express listen → IPC boot status). This sequence is critical for understanding readiness and should be documented as a Mermaid diagram.
- **Data-layer handler registration:** The Reader stream data layer uses a side-effect-based handler registration pattern (`registerHandlers`) that is non-obvious to newcomers and should be called out explicitly.
- **State persistence and hydration:** The dual-storage strategy (IndexedDB for Redux persistence, localStorage for `wpcom_user_id`, cookies for server-side bootstrap) is a common source of confusion and warrants a dedicated explanation with the exact key names and storage backends.
- **Lasagna (WebSocket) middleware:** The Reader conditionally loads `lasagna` middleware for real-time updates, gated behind feature flags. This should be documented as it affects the runtime behavior of the Reader stream.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **mature, Markdown-first documentation structure** with no external documentation generator framework (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` found). All documentation is authored as standalone `.md` files organized under the `docs/` directory and per-module `README.md` files throughout the `client/` tree.

- **Documentation framework:** Plain Markdown, no static site generator
- **Documentation entry point:** `README.md` (root) and `docs/install.md` for onboarding
- **Diagram support:** Mermaid is used natively (confirmed in `docs/yarn-start.md`)
- **API documentation tools:** None detected (no JSDoc/TypeDoc generation pipeline)
- **In-app docs browser:** `client/devdocs/` provides an in-app developer documentation browser with Lunr-based search indexing

**Key existing documentation files relevant to this task:**

| File | Relevance | Coverage Status |
|------|-----------|----------------|
| `docs/install.md` | Installation and port/host setup | Covers port 3000 and `calypso.localhost` but lacks internal architecture detail |
| `docs/yarn-start.md` | Build pipeline Mermaid diagram | Shows build flow but not readiness signals |
| `docs/development-workflow.md` | SECTION_LIMIT, debugging, limited builds | Covers build filtering but not Reader-specific workflow |
| `docs/routing.md` | Section-based routing architecture | Covers the routing system conceptually |
| `docs/server-side-rendering.md` | SSR constraints and cache behavior | Covers SSR but not dev-server specifics |
| `docs/data-persistence.md` | IndexedDB persistence, schema validation | Covers persistence architecture in detail |
| `docs/our-approach-to-data.md` | Redux → React Query evolution | Historical data-layer context |
| `client/reader/README.md` | Reader module purpose and routes | Lightweight; lacks API/state detail |
| `client/server/README.md` | Server architecture and Express middleware | Covers boot/pages/template split conceptually |
| `client/boot/README.md` | Boot module startup responsibilities | Documents startup chronology |
| `client/state/README.md` | Redux architecture conventions | Modular reducers, dynamic loading |

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to locate code relevant to the user's questions:

- **Server entry and port binding:** `client/server/index.js` — reads `protocol`, `hostname`, `port` from `@automattic/calypso-config` and calls `server.listen({ port, host })`
- **Bundler middleware and HMR:** `client/server/bundler/index.js` — mounts `webpack-dev-middleware` and `webpack-hot-middleware` on the Express app, prints `Ready!` message via `compiler.hooks.done`
- **Configuration manifests:** `config/development.json` — defines `port: 3000`, `hostname: "calypso.localhost"`, `protocol: "http"`
- **Reader route controllers:** `client/reader/index.ts`, `client/reader/controller.js` — define route middleware chains and the `following()` controller that renders the initial stream
- **Reader stream data layer:** `client/state/data-layer/wpcom/read/streams/index.js` — contains `streamApis` table mapping stream types to REST endpoints (`/read/following`, `/read/streams/following`, `/read/search`, etc.)
- **Redux action types:** `client/state/reader/action-types.ts` — canonical action vocabulary for the Reader feature (110 action type constants)
- **Stream actions:** `client/state/reader/streams/actions.js` — defines `requestPage`, `receivePage`, `showUpdates`, `receiveUpdates`, `selectItem`, etc.
- **Current user selectors:** `client/state/current-user/selectors.js` — exports `isUserLoggedIn()` which checks `state.currentUser.id !== null`
- **User store persistence:** `client/lib/user/store.js` — manages `wpcom_user_id` in localStorage via the `store` library
- **User initialization:** `client/lib/user/shared-utils/initialize-current-user.js` — checks `window.currentUser` (SSR bootstrap) then falls back to fetching `/me`
- **Server-side user bootstrap:** `client/server/user-bootstrap/index.js` — reads `wordpress_logged_in` cookie, signs with HMAC, fetches `/rest/v1/me?meta=flags`
- **Boot orchestration:** `client/boot/common.js` — calls `initializeCurrentUser()`, creates Redux store, hydrates persisted state, starts page.js routing
- **Sidebar styles:** `client/reader/sidebar/style.scss` — sidebar header margin/padding, menu link styling, tag list spacing
- **Reader section styles:** `client/reader/style.scss` — responsive breakpoints at `$break-small` (600px), 781px, 782px, 1300px; CSS custom property usage for layout calculations
- **Global CSS variables:** `client/assets/stylesheets/shared/_variables.scss` — defines `--masterbar-height: 46px` (32px at ≥782px), `--sidebar-width-max: 272px`
- **Content padding:** `client/my-sites/sidebar/style.scss` — sets `--content-padding-top: 16px`, `--content-padding-bottom: 16px` within `.is-global-sidebar-visible`

### 0.2.3 Web Search Research Conducted

No web search was needed for this documentation task. All answers are derived directly from the source code, as the user explicitly requested code-grounded answers.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's questions span six distinct architectural domains within the Calypso codebase. Each domain requires documentation that traces through multiple modules:

**Domain 1: Development Server Configuration**
- Module: `client/server/index.js`
  - Public APIs: `createServer()`, `sendBootStatus()`, `loadSslCert()`
  - Current documentation: Partially covered in `docs/install.md` (port 3000, `calypso.localhost`)
  - Documentation needed: Complete port binding lifecycle, readiness signals, IPC boot status
- Module: `client/server/bundler/index.js`
  - Public APIs: `middleware(app)` — installs webpack-dev-middleware, webpack-hot-middleware, and a `waitForCompiler` gate
  - Current documentation: `client/server/bundler/README.md` (conceptual overview only)
  - Documentation needed: Readiness message format, asset gating behavior, HMR pipeline
- Configuration: `config/development.json`
  - Documented values: `port: 3000`, `hostname: "calypso.localhost"`, `protocol: "http"`
  - Documentation needed: How these values flow into the server entry point

**Domain 2: Multi-Port Architecture**
- Module: `client/server/boot/` — Express app composition and middleware stack
  - Current documentation: `client/server/README.md` covers it at a conceptual level
  - Documentation needed: Confirmation that all traffic (HTML, assets, HMR, API proxy) shares a single port

**Domain 3: Reader Stream API Endpoints**
- Module: `client/state/data-layer/wpcom/read/streams/index.js`
  - `streamApis` config table: maps 16+ stream types to REST endpoint paths
  - Current documentation: None specific to the endpoint mapping
  - Documentation needed: Complete endpoint reference table with path, API version, date property, and query behavior
- Module: `client/reader/controller.js`
  - `following()` function: creates `StreamComponent` with `streamKey: 'following'`
  - Documentation needed: How the stream key maps to the `/read/following` endpoint

**Domain 4: Redux Actions During Initial Load**
- Module: `client/state/reader/action-types.ts` — 110 action type constants
  - Current documentation: None (the file is self-documenting as a constant registry)
  - Documentation needed: Subset of actions that fire during initial Reader load
- Module: `client/state/reader/streams/actions.js`
  - `requestPage()`, `receivePage()`, `showUpdates()`, `receiveUpdates()`
  - Current documentation: JSDoc comments on `requestPage`
  - Documentation needed: Action lifecycle sequence during initial stream fetch
- Module: `client/reader/index.ts`
  - `lazyLoadDependencies()`: conditionally loads Lasagna WebSocket middleware
  - Route registration: middleware chain `[redirectLoggedOutToDiscover, sidebar, setSelectedSiteIdByOrigin, following, makeLayout, clientRender]`
  - Documentation needed: Middleware chain execution order and the actions each step dispatches

**Domain 5: Authentication Detection**
- Module: `client/state/current-user/selectors.js`
  - `isUserLoggedIn(state)`: checks `state.currentUser.id !== null`
  - Current documentation: `client/state/current-user/README.md`
  - Documentation needed: Complete login-detection flow from storage to Redux selector
- Module: `client/lib/user/store.js`
  - `getStoredUserId()`: reads `wpcom_user_id` from localStorage
  - Current documentation: None
  - Documentation needed: Storage key names and backend identification
- Module: `client/lib/user/shared-utils/initialize-current-user.js`
  - Checks `window.currentUser` (SSR bootstrap), falls back to `rawCurrentUserFetch()` → `/me` endpoint
  - Current documentation: None
  - Documentation needed: Decision tree for user initialization
- Module: `client/server/user-bootstrap/index.js`
  - Reads `wordpress_logged_in` cookie, HMAC-signs, fetches `https://public-api.wordpress.com/rest/v1/me?meta=flags`
  - Current documentation: None
  - Documentation needed: Cookie → HMAC → API fetch pipeline

**Domain 6: Sidebar Responsive Design**
- Module: `client/reader/sidebar/style.scss`
  - Sidebar header: `margin: 0 12px 44px`, `padding: 0 10px`
  - Tag list: `margin-bottom: 8px`, form text input `padding: 0 8px`, `margin: 4px 4px 4px 0`
  - Current documentation: None
  - Documentation needed: Complete margin/padding inventory with CSS custom property references
- Module: `client/reader/style.scss`
  - Breakpoints: `$break-small` (600px), 781px, 782px, 1300px
  - CSS custom properties: `--masterbar-height`, `--sidebar-width-max`, `--content-padding-top`, `--content-padding-bottom`
  - Current documentation: None
  - Documentation needed: Breakpoint table with layout changes at each threshold
- Module: `client/assets/stylesheets/shared/_variables.scss`
  - `--masterbar-height: 46px` (mobile), `32px` (≥782px)
  - `--sidebar-width-max: 272px` (default), `295px` (global sidebar visible)
  - Current documentation: None
  - Documentation needed: Variable definitions and their responsive overrides

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internals:** The `streamApis` endpoint table in the data-layer is not documented anywhere outside the source file itself. No existing doc maps stream keys to REST paths.
- **No onboarding walkthrough for Reader:** `client/reader/README.md` is a lightweight overview. No existing document answers "how does the Reader section boot, fetch data, detect auth, and render responsively?"
- **Auth storage mechanisms are scattered:** The login detection pipeline spans four modules and three storage backends (cookie, localStorage, IndexedDB) with no consolidated reference.
- **CSS custom property inventory absent:** No existing document catalogs the layout-critical CSS variables or their responsive overrides for the Reader section.
- **No existing Q&A format docs:** The `docs/` folder contains guides and reference material but no Q&A-style developer onboarding documents.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive markdown document answering all six question clusters. It will live at `blitzy/documentation/wp-calypso_be7e5cc64162.md` in the destination repository.

```
blitzy/
└── documentation/
    └── wp-calypso_be7e5cc64162.md
        ├── Introduction & Context
        ├── Q1: Development Server Port & Readiness
        │   ├── Port Configuration
        │   ├── Readiness Signals
        │   └── Evidence & Rationale
        ├── Q2: Multi-Port Architecture
        │   ├── Single-Port Confirmation
        │   ├── Middleware Stack Walkthrough
        │   └── Evidence & Rationale
        ├── Q3: Reader Stream API Endpoints
        │   ├── Endpoint Reference Table
        │   ├── Request Lifecycle
        │   └── Evidence & Rationale
        ├── Q4: Redux Actions During Initial Load
        │   ├── Middleware Chain Sequence
        │   ├── Action Dispatch Timeline
        │   └── Evidence & Rationale
        ├── Q5: Authentication Detection
        │   ├── Storage Mechanisms
        │   ├── Decision Tree
        │   └── Evidence & Rationale
        ├── Q6: Sidebar Responsive Design
        │   ├── CSS Custom Properties Inventory
        │   ├── Breakpoint Table
        │   ├── Margin & Padding Values
        │   └── Evidence & Rationale
        └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach**

- Extract port and host values directly from `config/development.json` (lines 2–4) and trace their consumption in `client/server/index.js`.
- Extract readiness signals by analyzing `client/server/index.js` (IPC `sendBootStatus('ready')`) and `client/server/bundler/index.js` (`compiler.hooks.done` callback printing "Ready!" message).
- Build the API endpoint reference table by parsing the `streamApis` object literal in `client/state/data-layer/wpcom/read/streams/index.js`.
- Reconstruct the Redux action sequence by tracing the middleware chain in `client/reader/index.ts`, the controller functions in `client/reader/controller.js`, and the stream data-layer handlers.
- Document auth detection by tracing `client/boot/common.js` → `client/lib/user/shared-utils/initialize-current-user.js` → `client/lib/user/store.js` → `client/server/user-bootstrap/index.js`.
- Extract responsive design values from `client/reader/style.scss`, `client/reader/sidebar/style.scss`, and `client/assets/stylesheets/shared/_variables.scss`.

**Documentation Standards**

- Markdown formatting with `#`, `##`, `###` hierarchy and horizontal rules between major sections.
- Mermaid diagrams for:
  - Server startup and readiness sequence (`sequenceDiagram`)
  - Auth detection decision tree (`flowchart TD`)
  - Redux action dispatch timeline (`sequenceDiagram`)
- Code snippets limited to 2–3 lines for precision, always accompanied by the source file path and line numbers.
- Tables for structured data (endpoint mapping, CSS variable inventory, breakpoint thresholds).
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`.
- Consistent terminology aligned with Calypso's own naming: "stream", "streamKey", "masterbar", "sidebar", "boot status".

### 0.4.3 Diagram and Visual Strategy

**Mermaid Diagrams to Create**

- **Server Startup Sequence Diagram:** Illustrates the flow from `yarn start` through webpack compilation to the "Ready!" console message and IPC boot status.
- **Authentication Decision Flowchart:** Shows the branching logic from support-session check → `window.currentUser` → `rawCurrentUserFetch()` → Redux state hydration.
- **Reader Initial Load Sequence Diagram:** Maps the middleware chain (`redirectLoggedOutToDiscover` → `sidebar` → `following` → `makeLayout` → `clientRender`) to the Redux actions dispatched at each step.
- **Stream Data Fetch Sequence Diagram:** Traces `READER_STREAMS_PAGE_REQUEST` through the data-layer handler to the REST API call and back through `READER_STREAMS_PAGE_RECEIVE` and `READER_POSTS_RECEIVE`.

All diagrams will use `mermaid` code blocks with concise labels and will avoid nesting triple backticks.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

Per the implementation rule **SWE-AtlasQnA-Repo**, the deliverable is a single new markdown document in the destination repository. No existing repository files are modified or deleted.

| Target Documentation File | Transformation | Source Code / Docs | Content / Changes |
|---------------------------|----------------|--------------------|--------------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `config/development.json`, `client/server/index.js`, `client/server/bundler/index.js`, `client/reader/index.ts`, `client/reader/controller.js`, `client/state/data-layer/wpcom/read/streams/index.js`, `client/state/reader/streams/actions.js`, `client/state/reader/action-types.ts`, `client/state/current-user/selectors.js`, `client/state/current-user/actions.js`, `client/lib/user/store.js`, `client/lib/user/shared-utils/initialize-current-user.js`, `client/server/user-bootstrap/index.js`, `client/boot/common.js`, `client/reader/style.scss`, `client/reader/sidebar/style.scss`, `client/assets/stylesheets/shared/_variables.scss`, `client/my-sites/sidebar/style.scss`, `docs/install.md`, `docs/yarn-start.md` | Comprehensive Q&A document covering all six question clusters with Mermaid diagrams, reference tables, code citations, and rationale |

**Source files used as REFERENCE for documentation style:**

| Reference File | Role |
|----------------|------|
| `docs/install.md` | REFERENCE — documentation tone, heading style, and inline command formatting conventions |
| `docs/yarn-start.md` | REFERENCE — Mermaid diagram integration patterns and build-flow narrative structure |
| `docs/development-workflow.md` | REFERENCE — developer onboarding narrative style |
| `docs/our-approach-to-data.md` | REFERENCE — technical explanation depth and Redux data-flow documentation conventions |
| `docs/routing.md` | REFERENCE — route-to-controller documentation patterns |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Developer Onboarding Q&A Reference
Source Code: 20 files across client/server, client/reader,
             client/state, client/boot, client/lib/user,
             client/assets/stylesheets, config/, docs/
Sections:
  - Introduction & Context
      Purpose statement and codebase orientation
  - Q1: Development Server Port & Readiness
      Port 3000 from config/development.json
      Readiness via IPC sendBootStatus('ready') and console "Ready!" message
      Source: config/development.json:2-4,
              client/server/index.js,
              client/server/bundler/index.js
  - Q2: Multi-Port Architecture
      Single-port confirmation — Express serves HTML, webpack-dev-middleware
      serves assets, webpack-hot-middleware handles HMR, all on port 3000
      Source: client/server/bundler/index.js,
              client/server/index.js
  - Q3: Reader Stream API Endpoints
      Complete streamApis endpoint reference table (16+ stream types)
      Request lifecycle with INITIAL_FETCH=4, PER_FETCH=7, PER_POLL=40
      Source: client/state/data-layer/wpcom/read/streams/index.js
  - Q4: Redux Actions During Initial Load
      Middleware chain sequence diagram
      Action dispatch timeline: route match → controller →
      READER_STREAMS_PAGE_REQUEST → API fetch → READER_POSTS_RECEIVE +
      READER_STREAMS_PAGE_RECEIVE
      Source: client/reader/index.ts,
              client/reader/controller.js,
              client/state/reader/streams/actions.js,
              client/state/reader/action-types.ts
  - Q5: Authentication Detection
      Storage mechanisms: wordpress_logged_in cookie (server),
      wpcom_user_id in localStorage, window.currentUser (SSR),
      IndexedDB (Redux persistence)
      Decision tree flowchart
      Source: client/state/current-user/selectors.js,
              client/lib/user/store.js,
              client/lib/user/shared-utils/initialize-current-user.js,
              client/server/user-bootstrap/index.js,
              client/boot/common.js
  - Q6: Sidebar Responsive Design
      CSS custom property inventory table
      Breakpoint threshold table (600px, 781px, 782px, 1300px)
      Margin/padding values for sidebar header and content area
      Source: client/reader/style.scss,
              client/reader/sidebar/style.scss,
              client/assets/stylesheets/shared/_variables.scss,
              client/my-sites/sidebar/style.scss
  - Summary
      Consolidated findings and further reading pointers
Diagrams:
  - Server startup sequence (Mermaid sequenceDiagram)
  - Authentication decision tree (Mermaid flowchart)
  - Reader initial load sequence (Mermaid sequenceDiagram)
  - Stream data fetch sequence (Mermaid sequenceDiagram)
Key Citations: All 20 source files listed in the transformation table
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration files need modification. The deliverable is a standalone markdown file placed in `blitzy/documentation/` and does not integrate into any existing documentation build system (mkdocs, Docusaurus, or Sphinx). No navigation, sidebar, or table-of-contents configuration changes are required.

### 0.5.4 Cross-Documentation Dependencies

- **No shared includes:** The new document is self-contained with no imports from or exports to other documentation files.
- **Navigation links:** The document will contain internal anchor links between its own sections (e.g., from the Introduction to each Q&A cluster) but no outbound links to other repository docs that would require maintenance.
- **Index and glossary:** Not applicable — no existing index or glossary files are affected.
- **Relationship to existing docs:** The document references patterns and conventions found in `docs/install.md`, `docs/yarn-start.md`, and `docs/our-approach-to-data.md` but does not modify or depend on them at build time.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The deliverable is a plain Markdown file with embedded Mermaid diagrams. It requires no additional documentation framework installation. The only rendering dependency is Mermaid support, which GitHub and most modern Markdown viewers provide natively.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm (workspace) | `@automattic/calypso-config` | workspace:* | Provides the `port`, `hostname`, `protocol` values referenced in documentation |
| npm (devDependency) | `webpack-dev-middleware` | ^5.x | Referenced in single-port architecture explanation |
| npm (devDependency) | `webpack-hot-middleware` | ^2.x | Referenced in HMR pipeline explanation |
| npm (dependency) | `store` | ^2.0.12 | localStorage wrapper referenced in auth storage documentation |
| npm (dependency) | `page` | ^1.11.6 | Client-side router referenced in route middleware chain documentation |
| npm (peer) | `node` | ^22.9.0 | Runtime documented as prerequisite in setup context |
| npm (peer) | `yarn` | ^4.0.0 | Package manager documented as prerequisite |
| Mermaid | `mermaid` (rendering) | 11.x | Diagram rendering in Markdown viewers — no installation needed for authored `.md` files |

No packages need to be added, upgraded, or removed. All dependencies listed above are already present in the repository and are cited only as documentation references to explain architectural behavior.

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file `blitzy/documentation/wp-calypso_be7e5cc64162.md` is a standalone addition that does not alter or depend on any existing link structure in the repository's `docs/` folder or `README.md`.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis per question cluster:**

| Question Cluster | Existing Coverage | Source of Existing Coverage | Target |
|------------------|-------------------|-----------------------------|--------|
| Q1: Dev Server Port & Readiness | Partial (~40%) | `docs/install.md` mentions port 3000 and `calypso.localhost` but does not document readiness signals or IPC boot status | 100% — port source, binding lifecycle, both readiness signals |
| Q2: Multi-Port Architecture | Minimal (~10%) | `client/server/bundler/README.md` mentions webpack middleware conceptually but does not confirm single-port design | 100% — explicit single-port confirmation with middleware stack walkthrough |
| Q3: Reader Stream API Endpoints | None (0%) | No existing documentation maps stream keys to REST endpoints | 100% — full `streamApis` table with paths, API versions, date properties, and fetch constants |
| Q4: Redux Actions During Initial Load | None (0%) | No existing document traces the action dispatch sequence during Reader boot | 100% — middleware chain, controller dispatches, data-layer request/response cycle |
| Q5: Authentication Detection | Minimal (~15%) | `docs/data-persistence.md` mentions IndexedDB; `client/state/current-user/README.md` covers selectors | 100% — all four storage mechanisms, decision tree, SSR bootstrap pipeline |
| Q6: Sidebar Responsive Design | None (0%) | No CSS variable inventory or breakpoint reference exists for Reader | 100% — variable definitions, responsive overrides, breakpoint thresholds, margin/padding values |

**Overall target:** 100% coverage of all six question clusters, with every claim traced to a specific source file and line range.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer includes the specific file path(s) and line numbers that substantiate it.
- Every API endpoint entry includes the stream type, REST path, API version or namespace, and any special query parameters.
- Every CSS value includes the property name, value, file path, and the condition under which it applies (breakpoint or class name).
- Every Redux action includes the action type constant, the module that dispatches it, and the trigger condition.

**Accuracy validation:**

- Code references cite exact file paths verified during repository exploration (no assumed paths).
- Configuration values (`port: 3000`, `hostname: "calypso.localhost"`) are quoted directly from `config/development.json`.
- CSS custom property values and breakpoints are extracted from the actual SCSS source files, not inferred from browser behavior.
- The `streamApis` table is reproduced from the literal object in `client/state/data-layer/wpcom/read/streams/index.js`, not from external API documentation.

**Clarity standards:**

- Each question cluster opens with a direct, concise answer before providing supporting detail.
- Progressive disclosure: answer first, then evidence, then rationale.
- Consistent terminology: use "stream key" (not "stream type" or "stream name" interchangeably), "masterbar" (not "top bar"), "sidebar" (not "nav panel").
- Mermaid diagrams accompany any multi-step process (server startup, auth detection, Redux dispatch chain).

**Maintainability:**

- Source citations use the format `Source: path/to/file.ext:LineRange` for traceability.
- The document is structured as independent Q&A clusters so individual sections can be updated without affecting others.

### 0.7.3 Example and Diagram Requirements

| Diagram | Type | Purpose |
|---------|------|---------|
| Server Startup Sequence | Mermaid `sequenceDiagram` | Shows `yarn start` → webpack compile → `waitForCompiler` gate → `compiler.hooks.done` → "Ready!" message → `server.listen` → IPC `sendBootStatus('ready')` |
| Authentication Decision Tree | Mermaid `flowchart TD` | Branches: support session check → `window.currentUser` present? → `rawCurrentUserFetch()` → Redux state hydration |
| Reader Initial Load Sequence | Mermaid `sequenceDiagram` | Middleware chain → `following()` controller → `StreamComponent` mount → `READER_STREAMS_PAGE_REQUEST` → API call → response actions |
| Stream Data Fetch Sequence | Mermaid `sequenceDiagram` | `requestPage()` → data-layer handler → REST GET `/read/following` → `handlePage()` → `receivePosts` + `receivePage` |

- **Minimum code snippets per answer:** 1–2 short excerpts (2–3 lines each) showing the key line(s) from source.
- **Table count:** At least 3 reference tables (streamApis endpoint mapping, CSS custom property inventory, breakpoint thresholds).
- **Code example testing:** Not applicable — code snippets are read-only citations from the existing codebase, not executable examples.

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — the sole deliverable

**Source files read for documentation content (read-only reference):**

- `config/development.json` — port, hostname, protocol values
- `client/server/index.js` — server creation, `sendBootStatus('ready')`, listen callback
- `client/server/bundler/index.js` — webpack-dev-middleware, webpack-hot-middleware, `waitForCompiler`, "Ready!" message
- `client/server/bundler/README.md` — bundler conceptual overview
- `client/server/user-bootstrap/index.js` — cookie-based auth, HMAC signing, `/me` API fetch
- `client/boot/common.js` — app boot sequence, `initializeCurrentUser` call, Redux store creation
- `client/reader/index.ts` — Reader route registration, middleware chain, `lazyLoadDependencies`
- `client/reader/controller.js` — `sidebar()`, `following()`, `feedDiscovery()` and all Reader controllers
- `client/reader/utils.ts` — `getStreamType()` stream key parsing
- `client/reader/style.scss` — Reader layout CSS, breakpoints, padding, box-shadow, custom properties
- `client/reader/sidebar/style.scss` — sidebar header margins, tag list padding, icon states
- `client/reader/sidebar/index.jsx` — sidebar component structure
- `client/state/data-layer/wpcom/read/streams/index.js` — `streamApis` endpoint table, `requestPage`, `handlePage`
- `client/state/reader/streams/actions.js` — `requestPage()`, `receivePage()`, `showUpdates()`, stream action creators
- `client/state/reader/action-types.ts` — all Reader Redux action type constants
- `client/state/reader/reducer.ts` — `withStorageKey('reader', ...)` persistence configuration
- `client/state/current-user/selectors.js` — `isUserLoggedIn()`, `getCurrentUserId()`, `getCurrentUser()`
- `client/state/current-user/actions.js` — `fetchCurrentUser()`, `redirectToLogout()`
- `client/lib/user/store.js` — `getStoredUserId()`, `setStoredUserId()`, `clearStore()`
- `client/lib/user/shared-utils/initialize-current-user.js` — `window.currentUser` check, `rawCurrentUserFetch()` fallback
- `client/assets/stylesheets/shared/_variables.scss` — `--masterbar-height`, `--sidebar-width-max`, `--sidebar-width-min`
- `client/my-sites/sidebar/style.scss` — `.is-global-sidebar-visible` and `.is-global-sidebar-collapsed` overrides
- `client/sections.js` — Reader section registration, `enableLoggedOut: true`
- `docs/install.md` — existing install documentation (style reference)
- `docs/yarn-start.md` — existing build flow documentation (style reference)
- `docs/our-approach-to-data.md` — data layer documentation conventions (style reference)
- `docs/routing.md` — routing documentation conventions (style reference)
- `package.json` — Node/Yarn version constraints, workspace configuration

**Documentation topics covered:**

- Development server port binding and readiness detection
- Single-port vs. multi-port architecture confirmation
- Reader stream REST API endpoint mapping
- Redux action dispatch sequence during Reader initial load
- Authentication detection storage mechanisms and decision tree
- Sidebar responsive layout CSS custom properties, breakpoints, and spacing values

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the Calypso repository will be modified, created, or deleted. The implementation rule "Do not modify any existing files in the source repository" is strictly observed.
- **Test file modifications:** No test files will be altered or created.
- **Feature additions or refactoring:** No functional changes to Reader, authentication, or layout code.
- **Deployment configuration changes:** No CI/CD, Docker, or hosting configuration modifications.
- **Documentation outside the six question clusters:** Topics such as Calypso's commerce pipeline, Jetpack integration, desktop app, or A4A product surface are not covered.
- **Non-Reader sections:** The document focuses exclusively on the Reader section; other Calypso sections (My Sites, Stats, Plans, etc.) are out of scope.
- **Temporary observation scripts:** The user mentioned that "temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward." Since the deliverable is a static markdown document and all analysis was performed via read-only inspection, no temporary scripts were created and none require cleanup.
- **Documentation build system integration:** The deliverable is not wired into any existing docs build pipeline (mkdocs, Docusaurus, etc.).

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file that does not require a build step.
- **Documentation preview command:** Any Markdown viewer with Mermaid support (e.g., VS Code with the Markdown Preview Mermaid extension, or GitHub's native renderer) can preview `blitzy/documentation/wp-calypso_be7e5cc64162.md`.
- **Diagram generation command:** Not applicable — Mermaid diagrams are embedded inline and rendered by the viewer at display time. No pre-rendering step is needed.
- **Documentation deployment command:** Not applicable — the file is committed to the destination repository and requires no separate deployment.
- **Default format:** Markdown with embedded Mermaid diagram blocks.
- **Citation requirement:** Every section must reference the specific source file(s) and line range(s) that substantiate its claims, using the format `Source: path/to/file.ext:LineRange`.
- **Style guide:** Follow conventions observed in the repository's existing `docs/` folder — specifically the heading hierarchy, inline code formatting, and Mermaid integration patterns from `docs/install.md` and `docs/yarn-start.md`.
- **Documentation validation:** Manual review for:
  - All Mermaid blocks parse without syntax errors (validate by rendering in a Mermaid-compatible viewer).
  - All file path citations correspond to files that exist in the repository.
  - All configuration values match the values found in the cited source files.
  - No broken internal anchor links between Q&A sections.

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the implementation rule **SWE-AtlasQnA-Repo**:

- **Do not modify any existing files in the source repository.** The Calypso codebase must remain byte-identical after documentation generation. All output goes exclusively to `blitzy/documentation/wp-calypso_be7e5cc64162.md` in the destination repository.
- **Do not make assumptions — base all answers on the code as the truth.** Every claim in the document must be substantiated by a specific file, line range, or configuration value found during repository inspection. If the codebase does not contain evidence for a particular assertion, the assertion must not be made.
- **Provide thinking and rationale behind the answers.** Each Q&A cluster must include not only the factual answer but also the reasoning chain: which files were examined, what patterns were observed, and why the conclusion follows.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.** Since all analysis was performed via read-only file inspection, no temporary scripts were created. This rule is satisfied by default.
- **Create the document as `<source_branch_name>.md` in `blitzy/documentation/`.** The source branch is `wp-calypso_be7e5cc64162`, so the file name is `wp-calypso_be7e5cc64162.md`.
- **Comprehensive answers across all six question clusters.** No question may be left unanswered or partially addressed. The six clusters are: (1) dev server port and readiness, (2) multi-port architecture, (3) Reader stream API endpoints, (4) Redux actions during initial load, (5) authentication detection storage mechanisms, (6) sidebar responsive design with specific CSS values and breakpoints.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were retrieved and analyzed during the context-gathering phase. Each file contributed evidence to one or more of the six question clusters.

**Server and Configuration:**

| File Path | Relevance |
|-----------|-----------|
| `package.json` | Node ^22.9.0, Yarn ^4.0.0, workspace scripts, project metadata |
| `config/development.json` | `port: 3000`, `hostname: "calypso.localhost"`, `protocol: "http"`, feature flags |
| `client/server/index.js` | Server creation, `sendBootStatus('ready')` IPC, `server.listen()` callback |
| `client/server/bundler/index.js` | webpack-dev-middleware, webpack-hot-middleware, `waitForCompiler` gate, "Ready!" console message |
| `client/server/bundler/README.md` | Bundler conceptual overview |
| `client/server/user-bootstrap/index.js` | Cookie-based server-side auth, HMAC signing, `/me?meta=flags` API fetch |

**Reader Feature:**

| File Path | Relevance |
|-----------|-----------|
| `client/reader/index.ts` | Route registration, middleware chains, `lazyLoadDependencies()` for Lasagna |
| `client/reader/controller.js` | All Reader controllers: `sidebar()`, `following()`, `feedDiscovery()`, `feedListing()`, etc. |
| `client/reader/utils.ts` | `getStreamType()` stream key parsing utility |
| `client/reader/style.scss` | Reader layout CSS: breakpoints at 600px, 781px, 782px, 1300px; `--masterbar-height`, `--sidebar-width-max`, padding values |
| `client/reader/sidebar/style.scss` | Sidebar header `margin: 0 12px 44px`, `padding: 0 10px`, tag list spacing, form input padding |
| `client/reader/sidebar/index.jsx` | Sidebar component structure |
| `client/sections.js` | Reader section registration with `enableLoggedOut: true` |

**State Management:**

| File Path | Relevance |
|-----------|-----------|
| `client/state/data-layer/wpcom/read/streams/index.js` | `streamApis` endpoint table (16+ stream types), `requestPage`, `handlePage`, fetch constants |
| `client/state/reader/streams/actions.js` | `requestPage()`, `receivePage()`, `showUpdates()`, `receiveUpdates()`, stream action creators |
| `client/state/reader/action-types.ts` | Complete list of 110 Reader Redux action type constants |
| `client/state/reader/reducer.ts` | `withStorageKey('reader', ...)` IndexedDB persistence configuration |
| `client/state/current-user/selectors.js` | `isUserLoggedIn()`, `getCurrentUserId()`, `getCurrentUser()` |
| `client/state/current-user/actions.js` | `fetchCurrentUser()`, `rawCurrentUserFetch()`, `redirectToLogout()` |

**Authentication and User Libraries:**

| File Path | Relevance |
|-----------|-----------|
| `client/lib/user/store.js` | `getStoredUserId()`, `setStoredUserId()`, `clearStore()` — localStorage `wpcom_user_id` |
| `client/lib/user/shared-utils/initialize-current-user.js` | `window.currentUser` check, `rawCurrentUserFetch()` fallback, support session guard |
| `client/boot/common.js` | Boot sequence: `initializeCurrentUser`, Redux store creation, page.js routing start |

**Stylesheets and Layout:**

| File Path | Relevance |
|-----------|-----------|
| `client/assets/stylesheets/shared/_variables.scss` | `--masterbar-height: 46px / 32px`, `--sidebar-width-max: 272px`, `--sidebar-width-min: 228px` |
| `client/my-sites/sidebar/style.scss` | `.is-global-sidebar-visible` overrides (`--sidebar-width-max: 295px`), `.is-global-sidebar-collapsed` (`--sidebar-width-max: 69px`) |

**Existing Documentation (style reference):**

| File Path | Relevance |
|-----------|-----------|
| `docs/install.md` | Installation guide — documents port 3000, `calypso.localhost`, `SECTION_LIMIT` usage |
| `docs/yarn-start.md` | Build flow with Mermaid diagrams — documentation style reference |
| `docs/our-approach-to-data.md` | Redux data-flow documentation conventions |
| `docs/routing.md` | Route-to-controller documentation patterns |
| `docs/development-workflow.md` | Developer onboarding narrative style |
| `docs/data-persistence.md` | Data persistence documentation (IndexedDB mention) |

**Folders explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (root) | 0 | Repository structure overview |
| `client/` | 1 | Browser app root — identified Reader, server, state, boot, lib, assets |
| `client/reader/` | 2 | Reader feature root — controllers, routes, styles, sub-features |
| `client/reader/sidebar/` | 3 | Sidebar component and styles |
| `client/server/` | 2 | Express server — index, boot, bundler, user-bootstrap |
| `client/server/bundler/` | 3 | Webpack middleware and HMR |
| `client/state/` | 2 | Redux state tree — identified reader, current-user, streams slices |
| `client/state/reader/` | 3 | Reader state slices — streams, feeds, posts, follows, etc. |
| `client/state/reader/streams/` | 4 | Stream actions, reducers, selectors |
| `client/state/current-user/` | 3 | Current user actions, selectors, reducer |
| `client/boot/` | 2 | App boot sequence |
| `client/lib/user/` | 3 | User store and shared utilities |
| `config/` | 1 | Environment configs (development, production, stage) |
| `docs/` | 1 | Documentation hub |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma URLs or screens were provided for this project.

### 0.11.4 Tech Spec Sections Referenced

| Section Heading | Purpose |
|-----------------|---------|
| 1.1 Executive Summary | Confirmed Calypso as a REST-API-powered SPA covering WordPress.com, Jetpack Cloud, and A4A |
| 1.3 Scope | Confirmed Reader streams are within the `reader` product group scope |

