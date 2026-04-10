# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive developer onboarding Q&A document for the Calypso Reader section within the wp-calypso monorepo. The deliverable is a single 773-line markdown file (`blitzy/documentation/wp-calypso_be7e5cc64162.md`) that answers six architectural question clusters — development server configuration, multi-port architecture, Reader stream API endpoints, Redux action lifecycle, authentication detection mechanisms, and sidebar responsive design — grounded entirely in source code evidence with 45 inline citations referencing 18 verified source files. The document targets developers new to the Calypso codebase who need a code-backed architectural reference for the Reader feature. No existing repository files were modified.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (AI)" : 33
    "Remaining" : 4
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 37 |
| **Completed Hours (AI)** | 33 |
| **Remaining Hours** | 4 |
| **Completion Percentage** | 89.2% |

**Calculation:** 33 completed hours / (33 + 4 remaining hours) = 33 / 37 = **89.2% complete**

### 1.3 Key Accomplishments

- ✅ Created `blitzy/documentation/wp-calypso_be7e5cc64162.md` (773 lines, 40,513 bytes) — the sole AAP deliverable
- ✅ All 6 Q&A clusters fully documented with progressive disclosure structure (Direct Answer → Evidence → Rationale)
- ✅ 45 inline source citations using `Source: file:line` format, all verified against actual repository files
- ✅ 4 Mermaid diagrams created: server startup sequence, reader initial load sequence, stream data fetch sequence, authentication decision flowchart
- ✅ 6 reference tables: streamApis endpoint mapping (23 variants), CSS custom property inventory, breakpoint thresholds, storage mechanism summary, action types, and consolidated findings
- ✅ 18 unique source files referenced — all confirmed to exist in the repository
- ✅ 3 code review fix iterations applied (oversized code blocks trimmed, internal anchor links added, optional chaining explanation corrected, fetch constants order fixed)
- ✅ 15 visual verification screenshots captured confirming correct Mermaid rendering and table formatting
- ✅ Zero existing repository files modified — all 16 changes are additions (`A` status) in the `blitzy/` directory
- ✅ Zero placeholders, TODOs, or stubs in the document
- ✅ Clean working tree — no temporary files remain

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Line number citations may drift as source files are modified upstream | Low — citations reference current commit; future edits could make line numbers inaccurate | Human Developer | Ongoing maintenance |

### 1.5 Access Issues

No access issues identified. The deliverable is a standalone markdown file that does not require any external service credentials, API keys, or special repository permissions beyond standard read access to the source code.

### 1.6 Recommended Next Steps

1. **[High]** Conduct human technical review of all 6 Q&A sections to verify architectural accuracy and completeness
2. **[Medium]** Verify Mermaid diagram rendering in the target documentation viewer (GitHub, VS Code, internal wiki)
3. **[Medium]** Validate all internal anchor links function correctly across browsers
4. **[Low]** Establish a periodic citation freshness audit process to keep line number references current as the codebase evolves

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository Analysis & Source Code Investigation | 6 | Analyzed 20+ source files across client/server, client/reader, client/state, client/boot, client/lib/user, client/assets, config/, and docs/ to extract evidence for all 6 question clusters |
| Q1: Development Server Port & Readiness | 3 | Documented port 3000 binding from config/development.json, traced IPC sendBootStatus('ready') and console "Ready!" readiness signals, created server startup Mermaid sequence diagram |
| Q2: Multi-Port Architecture | 2 | Confirmed single-port architecture by analyzing webpack-dev-middleware, webpack-hot-middleware, and waitForCompiler gate mounted on same Express app instance |
| Q3: Reader Stream API Endpoints | 4 | Extracted complete streamApis configuration table (23 endpoint variants across 20 stream keys), documented fetch constants (INITIAL_FETCH=4, PER_FETCH=7, PER_POLL=40), handler registration pattern, and stream key parsing |
| Q4: Redux Actions During Initial Load | 4 | Traced middleware chain from client/reader/index.ts, documented action dispatch timeline (REQUEST→API→POSTS_RECEIVE+PAGE_RECEIVE), created 2 Mermaid sequence diagrams, documented Lasagna WebSocket middleware |
| Q5: Authentication Detection | 4 | Documented all 4 storage mechanisms (cookie, localStorage, window.currentUser, IndexedDB), traced full initialization pipeline across 4 modules, created authentication decision flowchart |
| Q6: Sidebar Responsive Design | 3 | Cataloged CSS custom property inventory (8 properties with overrides), documented 4 breakpoints (600/781/782/1300px), extracted specific margin/padding values from 3 SCSS files |
| Document Structure, Navigation & Summary | 1.5 | Created introduction with terminology conventions, internal anchor navigation, progressive disclosure structure, consolidated summary table, key architectural insights, and further reading pointers |
| Source Citation Verification | 2 | Verified all 45 inline citations against actual source code, confirmed 18 referenced files exist in repository |
| Code Review Fix Iterations (3 commits) | 2 | Trimmed oversized code blocks to AAP 2–3 line limit, added internal anchor links, corrected Q5 optional chaining explanation, fixed Q3 fetch constants order |
| Visual Rendering Verification | 1.5 | Captured 15 screenshots verifying Mermaid diagram rendering, table formatting, and document structure across all sections |
| **Total Completed** | **33** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Technical Accuracy Review | 2 | High |
| Mermaid & Anchor Link Rendering Verification | 1 | Medium |
| Minor Corrections from Review | 1 | Medium |
| **Total Remaining** | **4** | |

### 2.3 Hours Calculation Verification

- **Completed Hours:** 33 (sum of Section 2.1)
- **Remaining Hours:** 4 (sum of Section 2.2)
- **Total Project Hours:** 33 + 4 = **37** (matches Section 1.2)
- **Completion Percentage:** 33 / 37 = **89.2%** (matches Section 1.2)

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Source File Existence Verification | Bash (file system check) | 18 | 18 | 0 | 100% | All 18 referenced source files confirmed to exist in repository |
| Source Citation Accuracy | Manual code inspection | 45 | 45 | 0 | 100% | All 45 inline citations verified against actual source code at cited line numbers |
| Mermaid Diagram Syntax | Visual rendering via browser | 4 | 4 | 0 | 100% | Server startup, reader initial load, stream data fetch, auth decision flowchart all render correctly |
| Repository Integrity | Git diff analysis | 1 | 1 | 0 | 100% | Confirmed all 16 file changes are additions only — zero existing files modified |
| Document Completeness | Content audit | 6 | 6 | 0 | 100% | All 6 Q&A clusters present with Direct Answer, Evidence, and Rationale subsections |
| Placeholder/TODO Audit | Grep scan | 1 | 1 | 0 | 100% | Zero placeholders, TODOs, or stub content found in deliverable |

**Note:** This is a documentation-only project. No traditional unit tests, integration tests, or compilation steps apply. Validation consisted of content accuracy verification, source citation cross-referencing, visual rendering checks, and repository integrity confirmation.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Repository Integrity** — Working tree is clean with no uncommitted changes
- ✅ **Git Status** — All changes committed across 4 commits on the correct branch (`blitzy-278d7045-9ccb-41de-9dd0-814cf1136420`)
- ✅ **File System** — Deliverable file exists at `blitzy/documentation/wp-calypso_be7e5cc64162.md` (773 lines, 40,513 bytes)
- ✅ **No Temporary Files** — Zero temporary observation scripts or intermediate files remain

### UI / Document Rendering Verification

- ✅ **Document Preview** — Document renders correctly with proper heading hierarchy (H1, H2, H3)
- ✅ **Mermaid Diagrams** — All 4 diagrams (sequenceDiagram x3, flowchart x1) render correctly in browser-based Markdown viewers
- ✅ **Reference Tables** — All 6 tables render with proper column alignment and data integrity
- ✅ **Internal Navigation** — Anchor links from table of contents connect to corresponding Q&A sections
- ✅ **Code Blocks** — 27 code blocks (JSON, JS, TS, SCSS) display with proper syntax highlighting markers
- ✅ **15 Screenshots Captured** — Visual evidence saved in `blitzy/screenshots/` confirming rendering quality

### API / Integration Verification

- ✅ **Not Applicable** — This is a documentation-only deliverable with no API endpoints, database connections, or external service integrations

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|-----------------|--------|----------|
| Create `blitzy/documentation/wp-calypso_be7e5cc64162.md` | ✅ Pass | File exists, 773 lines, 40,513 bytes |
| Q1: Dev Server Port & Readiness documented | ✅ Pass | Lines 32–137: port 3000, IPC boot status, console Ready message, Mermaid diagram |
| Q2: Multi-Port Architecture documented | ✅ Pass | Lines 140–183: single-port confirmation with middleware stack evidence |
| Q3: Reader Stream API Endpoints documented | ✅ Pass | Lines 186–294: 23-row endpoint table, fetch constants, handler registration, stream key parsing |
| Q4: Redux Actions During Initial Load documented | ✅ Pass | Lines 297–466: middleware chain, action dispatch timeline, 2 Mermaid diagrams, Lasagna docs |
| Q5: Authentication Detection documented | ✅ Pass | Lines 469–629: 4 storage mechanisms, 5-step detection flow, Mermaid flowchart |
| Q6: Sidebar Responsive Design documented | ✅ Pass | Lines 633–743: CSS property inventory, breakpoint table, margin/padding values |
| Summary section with consolidated findings | ✅ Pass | Lines 747–773: summary table, key insights, further reading links |
| Source citations with file paths and line numbers | ✅ Pass | 45 inline citations using `Source: file:line` format |
| Mermaid diagrams (4 required) | ✅ Pass | Server startup, reader initial load, stream data fetch, auth decision flowchart |
| Reference tables (3+ required) | ✅ Pass | 6 tables: endpoints, CSS properties, breakpoints, storage mechanisms, action types, summary |
| No existing repository files modified | ✅ Pass | Git diff confirms all 16 changes are additions only (`A` status) |
| Evidence-based answers (no assumptions) | ✅ Pass | Every claim backed by cited source file and line number |
| Provide thinking/rationale behind answers | ✅ Pass | Each Q&A section includes a "Rationale" subsection explaining reasoning |
| Progressive disclosure structure | ✅ Pass | Direct Answer → Evidence → Rationale format in all 6 sections |
| Consistent terminology (stream key, masterbar, sidebar) | ✅ Pass | Terminology conventions defined in introduction, used consistently |

### Fixes Applied During Validation

| Fix | Commit | Description |
|-----|--------|-------------|
| Code block trimming | `f880c89` | Oversized code blocks reduced to AAP-mandated 2–3 line limit |
| Internal anchor links | `f880c89` | Added table of contents with anchor navigation to all 6 Q&A sections |
| Code review findings | `4d94317` | Addressed 6 findings: heading hierarchy, source format consistency, table alignment |
| Q5 optional chaining & Q3 constants | `52e2c51` | Corrected Q5 optional chaining edge case explanation; fixed Q3 fetch constants order |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source file line numbers drift as upstream code evolves | Technical | Low | High | Include file path + function name alongside line numbers; establish periodic audit | Open — requires human maintenance process |
| Mermaid diagrams may render differently across viewers | Technical | Low | Medium | Tested in browser-based renderer; use simple diagram syntax without advanced features | Mitigated — all 4 diagrams use standard Mermaid syntax |
| Document accuracy depends on current commit snapshot | Operational | Low | Medium | Pin citations to commit hash or add "last verified" date header | Open — recommended for human review |
| No automated freshness checking for citations | Operational | Low | Medium | Could add CI script to verify cited files/lines still match | Open — enhancement opportunity |
| Reader architecture may change without document update | Operational | Medium | Medium | Add document to team wiki with ownership assignment; set quarterly review cadence | Open — requires team process setup |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 33
    "Remaining Work" : 4
```

**Completed:** 33 hours (89.2%) — Dark Blue (#5B39F3)
**Remaining:** 4 hours (10.8%) — White (#FFFFFF)

### Remaining Hours by Category

| Category | Hours |
|----------|-------|
| Human Technical Accuracy Review | 2 |
| Mermaid & Anchor Link Rendering Verification | 1 |
| Minor Corrections from Review | 1 |
| **Total** | **4** |

---

## 8. Summary & Recommendations

### Achievements

The Blitzy autonomous agents successfully delivered a comprehensive 773-line developer onboarding Q&A document covering all six architectural question clusters specified in the Agent Action Plan. The document provides code-backed answers to questions about development server configuration, port architecture, Reader stream API endpoints, Redux action lifecycle, authentication detection, and sidebar responsive design — all grounded in 45 verified source citations across 18 repository files. Four Mermaid diagrams and six reference tables provide visual and tabular summaries of complex architectural flows.

The project is **89.2% complete** (33 of 37 total hours). All AAP-specified content has been authored, validated through 3 code review fix iterations, and visually verified with 15 screenshots. The remaining 4 hours consist of human technical review, rendering verification in the target environment, and minor corrections.

### Remaining Gaps

1. **Human technical review (2h):** A senior developer familiar with the Calypso Reader codebase should review all 6 Q&A sections for architectural accuracy, especially the authentication pipeline (Q5) and the streamApis endpoint table (Q3) which have the most complex mappings.
2. **Rendering verification (1h):** Mermaid diagrams and internal anchor links should be tested in the team's actual documentation viewer (GitHub, internal wiki, or VS Code) to confirm cross-platform rendering.
3. **Minor corrections (1h):** Budget for any accuracy fixes discovered during human review.

### Critical Path to Production

This is a documentation-only deliverable with no runtime impact on the Calypso application. The critical path to "production" (making the document available to the development team) requires only:
1. Human review approval
2. PR merge to the target branch

### Production Readiness Assessment

| Criterion | Status |
|-----------|--------|
| All AAP deliverables authored | ✅ Complete |
| Source citations verified | ✅ Complete |
| No existing files modified | ✅ Confirmed |
| Visual rendering verified | ✅ 15 screenshots |
| Human review completed | ⏳ Pending |
| Rendering verified in target viewer | ⏳ Pending |

**Recommendation:** Approve for human review. The document is content-complete and ready for senior developer validation before merge.

---

## 9. Development Guide

### System Prerequisites

| Software | Required Version | Purpose |
|----------|-----------------|---------|
| Node.js | ^22.9.0 | Runtime for Calypso (per `package.json` engines) |
| Yarn | 4.0.2 | Package manager (per `package.json` packageManager) |
| Git | Latest | Version control |
| Markdown viewer with Mermaid support | Any | Rendering the documentation deliverable |

### Environment Setup

**1. Clone the repository:**

```bash
git clone https://github.com/Automattic/wp-calypso.git
cd wp-calypso
```

**2. Switch to the feature branch:**

```bash
git checkout blitzy-278d7045-9ccb-41de-9dd0-814cf1136420
```

**3. Verify the deliverable exists:**

```bash
ls -la blitzy/documentation/wp-calypso_be7e5cc64162.md
# Expected: 773 lines, ~40KB file
wc -l blitzy/documentation/wp-calypso_be7e5cc64162.md
# Expected output: 773 blitzy/documentation/wp-calypso_be7e5cc64162.md
```

### Viewing the Document

**Option A: GitHub (recommended)**

Navigate to the file in the GitHub PR diff view. GitHub natively renders Mermaid diagrams and markdown tables.

**Option B: VS Code**

```bash
code blitzy/documentation/wp-calypso_be7e5cc64162.md
```

Install the "Markdown Preview Mermaid Support" extension for diagram rendering. Use `Ctrl+Shift+V` (or `Cmd+Shift+V` on macOS) to open the preview pane.

**Option C: Command line preview**

```bash
# View the table of contents and introduction
head -30 blitzy/documentation/wp-calypso_be7e5cc64162.md

# Count sections
grep -c "^## " blitzy/documentation/wp-calypso_be7e5cc64162.md
# Expected: 8 (Intro + Q1-Q6 + Summary)

# Count source citations
grep -c "Source:" blitzy/documentation/wp-calypso_be7e5cc64162.md
# Expected: 45

# Count Mermaid diagrams
grep -c "mermaid" blitzy/documentation/wp-calypso_be7e5cc64162.md
# Expected: 4
```

### Verifying Source Citations

To spot-check that source citations in the document still match the codebase:

```bash
# Verify Q1: Port configuration
sed -n '6,8p' config/development.json
# Expected: protocol, hostname, port values matching doc

# Verify Q5: Auth cookie name
sed -n '8,8p' client/server/user-bootstrap/index.js
# Expected: const AUTH_COOKIE_NAME = 'wordpress_logged_in';

# Verify: isUserLoggedIn selector
sed -n '6,17p' client/state/current-user/selectors.js
# Expected: getCurrentUserId and isUserLoggedIn functions
```

### Running the Calypso Dev Server (for context)

If you want to see the Reader section live (context for the documentation):

```bash
# Add hosts entry (one-time)
echo "127.0.0.1 calypso.localhost" | sudo tee -a /etc/hosts

# Install dependencies
yarn

# Start with Reader section only (faster build)
SECTION_LIMIT=reader,login yarn start

# Open in browser when "Ready!" appears
# URL: http://calypso.localhost:3000/reader
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Mermaid diagrams show as code blocks | Install a Mermaid-compatible Markdown viewer (VS Code extension or use GitHub) |
| Internal anchor links don't navigate | Ensure the Markdown viewer supports GitHub-style anchor IDs (lowercase, hyphens) |
| Source citation line numbers don't match | The document references the codebase at the time of writing; line numbers may shift with upstream changes |
| `yarn start` fails with Node version error | Ensure Node.js ^22.9.0 is installed (use `nvm install 22` to manage versions) |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `wc -l blitzy/documentation/wp-calypso_be7e5cc64162.md` | Verify document line count (expected: 773) |
| `grep -c "Source:" blitzy/documentation/wp-calypso_be7e5cc64162.md` | Count source citations (expected: 45) |
| `grep -c "mermaid" blitzy/documentation/wp-calypso_be7e5cc64162.md` | Count Mermaid diagrams (expected: 4) |
| `git diff --name-status origin/wp-calypso_be7e5cc64162...HEAD` | Verify only additions, no modifications |
| `SECTION_LIMIT=reader,login yarn start` | Start Calypso dev server with Reader section only |

### B. Port Reference

| Port | Service | Source |
|------|---------|--------|
| 3000 | Calypso development server (HTML, assets, HMR, API proxy — all unified) | `config/development.json:8` |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | **Deliverable** — Developer onboarding Q&A document |
| `config/development.json` | Dev server configuration (port, hostname, protocol) |
| `client/server/index.js` | Server entry point (boot, listen, IPC) |
| `client/server/bundler/index.js` | Webpack middleware and HMR setup |
| `client/reader/index.ts` | Reader route registration and middleware chains |
| `client/reader/controller.js` | Reader route controllers (sidebar, following, etc.) |
| `client/state/data-layer/wpcom/read/streams/index.js` | streamApis endpoint table and data-layer handlers |
| `client/state/reader/streams/actions.js` | Stream Redux action creators |
| `client/state/reader/action-types.ts` | Reader Redux action type constants |
| `client/state/current-user/selectors.js` | Login state selectors (isUserLoggedIn) |
| `client/lib/user/shared-utils/initialize-current-user.js` | User initialization pipeline |
| `client/server/user-bootstrap/index.js` | Server-side cookie-based auth bootstrap |
| `client/boot/common.js` | App boot sequence orchestration |
| `client/reader/style.scss` | Reader section responsive styles |
| `client/reader/sidebar/style.scss` | Reader sidebar layout styles |
| `client/assets/stylesheets/shared/_variables.scss` | Global CSS custom property definitions |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Node.js | ^22.9.0 | `package.json` engines |
| Yarn | 4.0.2 | `package.json` packageManager |
| React | (workspace dependency) | `package.json` |
| Redux | (workspace dependency) | `package.json` |
| Express | (workspace dependency) | `client/server/index.js` |
| webpack | (devDependency) | `client/server/bundler/index.js` |
| webpack-dev-middleware | ^5.x | `client/server/bundler/index.js:5` |
| webpack-hot-middleware | ^2.x | `client/server/bundler/index.js:6` |
| Mermaid | 11.x (rendering) | Embedded in markdown, rendered by viewer |

### E. Environment Variable Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `SECTION_LIMIT` | Restrict build to specific sections for faster startup | `SECTION_LIMIT=reader,login` |
| `NODE_OPTIONS` | Pass options to Node runtime (debugging) | `NODE_OPTIONS="--inspect=5858"` |
| `CALYPSO_IS_FORK` | Indicates server runs as forked child process | Set by build orchestrator |

### G. Glossary

| Term | Definition |
|------|-----------|
| **stream key** | Identifier string passed to the data layer to select a stream (e.g., `'following'`, `'site:12345'`) |
| **streamApis** | Configuration object in `client/state/data-layer/wpcom/read/streams/index.js` mapping stream keys to REST API endpoints |
| **masterbar** | Top navigation bar rendered across all Calypso sections |
| **sidebar** | Left-hand navigation panel within the Reader section |
| **boot status** | IPC message (`{ boot: 'ready' }`) sent to parent process when server is ready |
| **waitForCompiler** | Express middleware gate that blocks requests until webpack finishes initial compilation |
| **Lasagna** | WebSocket middleware for real-time Reader updates, gated behind feature flags |
| **registerHandlers** | Side-effect-based handler registration pattern connecting Redux action types to data-layer API calls |
| **withStorageKey** | Redux persistence utility enabling IndexedDB-based state persistence for specific state slices |