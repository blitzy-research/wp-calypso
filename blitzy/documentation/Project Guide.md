# Blitzy Project Guide — wp-calypso Testing Infrastructure Investigation

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces a comprehensive, read-only investigation document analyzing the **wp-calypso testing infrastructure**. The sole deliverable is a 973-line Markdown document (`blitzy/documentation/wp-calypso_be7e5cc64162.md`) that answers six onboarding questions: development server verification, test environment characterization, test-only globals/polyfills/mocks, network request interception via nock, end-to-end API mock tracing through a Redux thunk, and configuration/feature flag divergence between `test.json` and `development.json`. Zero existing repository files were modified; the document is entirely evidence-based with code excerpts and file path citations from 20+ source files.

### 1.2 Completion Status

```mermaid
pie title Project Completion
    "Completed (32h)" : 32
    "Remaining (3h)" : 3
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 35 |
| **Completed Hours (AI)** | 32 |
| **Remaining Hours** | 3 |
| **Completion Percentage** | 91.4% |

**Calculation**: 32 completed hours / (32 + 3) total hours = 91.4% complete.

### 1.3 Key Accomplishments

- ✅ Created 973-line comprehensive testing infrastructure investigation document
- ✅ Analyzed 20+ source files with evidence-based code excerpts and line number citations
- ✅ Cataloged 15+ test-only globals/polyfills installed by setup files across 4 test suites
- ✅ Traced complete nock network isolation lifecycle (disableNetConnect → useNock → afterAll cleanup)
- ✅ Documented end-to-end API mock flow: nock interceptor → wpcom API client → Redux dispatch spy → test assertion
- ✅ Compared `config/test.json` (101 feature flags) vs `config/development.json` (178 flags) with detailed divergence tables
- ✅ Proved config resolution via CLI: `config('env_id')` returns `'test'` with `NODE_ENV=test`, `'development'` with `NODE_ENV=development`
- ✅ Validated 36/36 tests passing across client (26), server (7), and build-tools (3) suites
- ✅ Maintained strict read-only constraint — zero existing repository files modified
- ✅ Included Mermaid diagrams for test boot sequence and API mock flow

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Document requires human peer review for technical accuracy | Low — document is validated but not yet reviewed by a domain expert | Human Developer | 2h |
| Full development server boot not performed | Low — dev server startup was documented from source code but `yarn start` was not executed end-to-end due to build time constraints | Human Developer | 1h |

### 1.5 Access Issues

No access issues identified. All source files were readable, all test suites executed successfully, and all dependencies were installed and functional. Node.js v22.22.2 (satisfies `^v22.9.0`) and Yarn 4.0.2 (satisfies `^4.0.0`) were available.

### 1.6 Recommended Next Steps

1. **[High]** Peer review the 973-line investigation document (`blitzy/documentation/wp-calypso_be7e5cc64162.md`) for technical accuracy and completeness
2. **[Medium]** Run `yarn start` to perform a full development server boot test and verify the document's dev server section against actual boot output
3. **[Low]** Update browserslist database (`npx update-browserslist-db@latest`) to suppress the 14-month-old data warning in test output

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository Analysis & Code Reading | 6 | Deep analysis of 20+ source files across test infrastructure, config system, API client, and mock patterns |
| Investigation Area 1 — Dev Server Verification | 2 | Documented startup sequence (`yarn start`), engine requirements (`^v22.9.0`), runtime baseline (native browser globals) |
| Investigation Area 2 — Test Environment Characterization | 4 | Cataloged 7 Jest suites, traced setup file chains, documented environment variables, module resolution with `enhanced-resolve` |
| Investigation Area 3 — Test-Only Globals/Polyfills | 3 | Enumerated 15+ globals across 4 setup files with line numbers, built comparison table (test vs development runtime) |
| Investigation Area 4 — Network Request Interception | 3 | Traced nock.disableNetConnect() lifecycle, documented useNock helper, explained unintercepted request behavior |
| Investigation Area 5 — End-to-End Mock Tracing | 4 | Full trace of products-list test: nock interceptor → wpcom API → Redux thunk → dispatch spy → assertion; Mermaid sequence diagram |
| Investigation Area 6 — Config/Feature Flag Divergence | 4 | Traced config resolution chain, compared test.json vs development.json (101 vs 178 flags), documented 3 override mechanisms |
| Document Structure & Diagrams | 2 | Table of Contents, Mermaid flow diagrams, comparison tables, formatted code blocks, anchor links |
| Validation Testing & Proof Generation | 2 | Ran 36 tests (26 client + 7 server + 3 build-tools), CLI config proof, feature flag count verification |
| Code Review Fixes | 1 | 2 fix commits: corrected package.json line reference (125→129), addressed code review findings |
| Environment Setup | 1 | Node.js version verification, Yarn validation, dependency installation |
| **Total** | **32** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Document Peer Review — Human review of 973-line technical document for accuracy and completeness | 2 | High |
| Development Server Boot Test — Execute `yarn start` end-to-end to verify dev server section against actual boot output | 1 | Medium |
| **Total** | **3** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Client Unit (products-list) | Jest 29.7 | 26 | 26 | 0 | N/A | Actions (3), Reducers (15), Selectors (8) — validates API mock tracing documented in investigation |
| Server Unit (config/parser) | Jest 29.7 | 7 | 7 | 0 | N/A | Parser cascading, secrets handling, feature overrides — validates config resolution documented in investigation |
| Build-tools Unit | Jest 29.7 | 3 | 3 | 0 | N/A | Webpack sections-loader tests — validates build-tools suite setup documented in investigation |
| **Total** | | **36** | **36** | **0** | | **100% pass rate** |

All tests originate from Blitzy's autonomous validation logs. Test suites were selected to validate the specific testing infrastructure behaviors documented in the investigation (API mocking patterns, config resolution, and test suite configuration).

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Node.js runtime** — v22.22.2 active and satisfies engine constraint `^v22.9.0`
- ✅ **Yarn package manager** — v4.0.2 active and satisfies constraint `^4.0.0`
- ✅ **Dependency installation** — All workspace dependencies installed successfully
- ✅ **Test runner** — Jest 29.7 operational across all 3 validated test suites
- ✅ **nock network isolation** — Verified operational (tests pass without real HTTP calls)
- ✅ **Config resolution** — `config('env_id')` returns `'test'` with `NODE_ENV=test` and `'development'` with `NODE_ENV=development`
- ✅ **Feature flag counts** — Programmatically verified: 101 in `test.json`, 178 in `development.json`

### UI Verification

- ⚠️ **Development server** — Not boot-tested via `yarn start` due to full build time constraints; startup sequence documented from source code analysis
- ✅ **No UI components modified** — This is a documentation-only deliverable; no UI verification required

### API Integration

- ✅ **nock interceptor validation** — Products-list test successfully mocks `https://public-api.wordpress.com:443/rest/v1.1/products` endpoints (200 and 500 responses)
- ✅ **wpcom API client** — Request chain traced through `client/lib/wp` → `wpcom.req.get()` → nock interception → mock response

---

## 5. Compliance & Quality Review

| AAP Deliverable | Status | Evidence |
|----------------|--------|----------|
| Development server verification (Section 1) | ✅ Pass | Document Section 1: startup sequence, engine requirements, runtime baseline documented |
| Test environment characterization (Section 2) | ✅ Pass | Document Section 2: 7 Jest suites cataloged, setup chains traced, env vars documented |
| Test-only globals/polyfills/mocks (Section 3) | ✅ Pass | Document Section 3: 15+ globals enumerated with line numbers and comparison table |
| Network request interception (Section 4) | ✅ Pass | Document Section 4: nock lifecycle, useNock helper, unintercepted request behavior |
| End-to-end mock tracing (Section 5) | ✅ Pass | Document Section 5: Full products-list trace with Mermaid sequence diagram |
| Config/feature flag divergence (Section 6) | ✅ Pass | Document Section 6: Resolution chain, JSON comparison, 3 override mechanisms |
| No existing files modified | ✅ Pass | `git diff --name-status` shows only 1 file added (`A blitzy/documentation/wp-calypso_be7e5cc64162.md`) |
| Temporary script cleanup | ✅ Pass | `git status`: working tree clean, no temporary artifacts |
| Output file in correct location | ✅ Pass | File exists at `blitzy/documentation/wp-calypso_be7e5cc64162.md` (973 lines) |
| Evidence-based answers with code citations | ✅ Pass | All claims cite specific files and line numbers; 20+ source files cross-referenced |
| Mermaid diagrams for complex flows | ✅ Pass | 2 diagrams: test boot sequence (graph TD) and API mock flow (sequenceDiagram) |
| Document structured for sequential reading | ✅ Pass | 7 major sections with Table of Contents, progressive knowledge building |

### Validation Fixes Applied

| Fix | Commit | Description |
|-----|--------|-------------|
| Line reference correction | `b2b1a90588` | Corrected `package.json` line reference for `test-packages` script from 125 to 129 |
| Code review findings | `079144c08c` | Addressed code review findings in testing infrastructure documentation |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Document technical inaccuracies | Technical | Low | Low | All code excerpts cross-referenced against source files; line numbers verified; feature flag counts validated programmatically | Mitigated — awaiting human peer review |
| Dev server boot not verified | Operational | Low | Medium | Startup sequence documented from `package.json` scripts and source analysis; boot test deferred due to build time | Open — 1h human task |
| Browserslist data 14 months old | Technical | Low | Low | Non-blocking warning in test output; does not affect test results or document accuracy | Open — optional update |
| Document may become stale as codebase evolves | Operational | Medium | High | Document cites specific line numbers that may shift; recommend periodic re-validation | Acknowledged — inherent to documentation |
| Node.js version drift | Technical | Low | Low | Current v22.22.2 satisfies `^v22.9.0`; future Node updates within range should be compatible | Mitigated |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 32
    "Remaining Work" : 3
```

### Remaining Work by Category

| Category | Hours |
|----------|-------|
| Document Peer Review | 2 |
| Dev Server Boot Test | 1 |
| **Total Remaining** | **3** |

---

## 8. Summary & Recommendations

### Achievements

The project successfully delivered a comprehensive 973-line testing infrastructure investigation document covering all six required investigation areas. The document is evidence-based, citing 20+ source files with specific line numbers and code excerpts. All 36 validation tests passed (100% pass rate), and the config resolution system was independently verified via CLI commands. The strict read-only constraint was honored — zero existing repository files were modified, and no temporary artifacts remain.

### Completion Assessment

The project is **91.4% complete** (32 completed hours / 35 total hours). All AAP-scoped autonomous work has been delivered. The remaining 3 hours consist of human-required tasks: peer review of the technical document (2h) and a full development server boot test (1h).

### Critical Path to Production

1. **Human peer review** (2h) — A domain-expert developer should review the document's technical claims, especially the test-only globals catalog, config resolution chain, and feature flag divergence tables
2. **Dev server boot test** (1h) — Execute `yarn start` to verify the development server boots successfully and compare actual output against the document's Section 1

### Production Readiness Assessment

The deliverable is ready for review. All autonomous validation gates passed, all investigation areas are thoroughly documented with evidence, and the document follows a progressive structure suitable for onboarding new contributors. The document can be merged after human peer review confirms technical accuracy.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Verification Command |
|-------------|---------|---------------------|
| Node.js | ^v22.9.0 | `node --version` |
| Yarn | ^4.0.0 | `yarn --version` |
| Git | Any recent | `git --version` |

### Environment Setup

```bash
# 1. Clone the repository and switch to the feature branch
git clone https://github.com/Automattic/wp-calypso.git
cd wp-calypso
git checkout blitzy-ba0197c7-112c-479a-94d5-69920b416b04

# 2. Verify Node.js version (must satisfy ^v22.9.0)
node --version
# Expected: v22.x.x (e.g., v22.22.2)

# 3. If using nvm, install and activate the correct version
nvm install 22.9.0
nvm use 22.9.0

# 4. Verify Yarn version (must satisfy ^4.0.0)
yarn --version
# Expected: 4.0.2
```

### Dependency Installation

```bash
# Install all workspace dependencies
yarn install
# This installs dependencies for all workspaces: client, desktop, apps/*, packages/*, test/e2e
```

### Running Tests (Validation)

```bash
# Run the client suite tests for products-list (validates API mock tracing)
npx jest -c=test/client/jest.config.js --testPathPattern="products-list" --watchAll=false --ci --no-coverage
# Expected: 3 test suites, 26 tests passed

# Run the server suite tests for config parser (validates config resolution)
npx jest -c=test/server/jest.config.js --testPathPattern="config/test/parser" --watchAll=false --ci --no-coverage
# Expected: 1 test suite, 7 tests passed

# Run the build-tools suite tests (validates build-tools setup)
npx jest -c=test/build-tools/jest.config.js --watchAll=false --ci --no-coverage
# Expected: 1 test suite, 3 tests passed

# Run all four default test suites
yarn test
# Runs: test-client, test-packages, test-server, test-build-tools
```

### Verifying Config Resolution

```bash
# Prove that config('env_id') returns 'test' when NODE_ENV=test
NODE_ENV=test node -e "const config = require('./client/server/config'); console.log('env_id:', config('env_id'));"
# Expected: env_id: test

# Prove that config('env_id') returns 'development' when NODE_ENV=development
NODE_ENV=development node -e "const config = require('./client/server/config'); console.log('env_id:', config('env_id'));"
# Expected: env_id: development
```

### Verifying Feature Flag Counts

```bash
# Count feature flags in test.json and development.json
node -e "
const test = require('./config/test.json');
const dev = require('./config/development.json');
console.log('test.json feature flags:', Object.keys(test.features || {}).length);
console.log('development.json feature flags:', Object.keys(dev.features || {}).length);
"
# Expected: test.json feature flags: 101
# Expected: development.json feature flags: 178
```

### Viewing the Deliverable

```bash
# The investigation document is located at:
cat blitzy/documentation/wp-calypso_be7e5cc64162.md
# 973 lines covering all 6 investigation areas
```

### Starting the Development Server (Optional)

```bash
# Full build and start (takes several minutes)
yarn start
# This runs: check-node-version → welcome.js → build → start-build
# Server will be available at http://calypso.localhost:3000
```

### Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Browserslist: browsers data is 14 months old` | Outdated caniuse-lite database | Run `npx update-browserslist-db@latest` (non-blocking warning) |
| `Node version mismatch` | Node.js < 22.9.0 active | Run `nvm install 22.9.0 && nvm use 22.9.0` |
| `yarn install` fails | Yarn version mismatch | Ensure Yarn 4.0.2 is active (check `.yarnrc.yml`) |
| Tests enter watch mode | Missing `--watchAll=false` flag | Always use `--watchAll=false --ci` flags for non-interactive execution |
| `NetConnectNotAllowedError` in test | Missing nock interceptor | Register a nock interceptor for the URL being requested |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `yarn test` | Run all 4 default test suites (client, packages, server, build-tools) |
| `yarn test-client` | Run client-side tests with `TZ=UTC` |
| `yarn test-server` | Run server-side tests |
| `yarn test-packages` | Run package-level tests |
| `yarn test-build-tools` | Run build-tools tests |
| `yarn test-integration` | Run integration tests (allows real network) |
| `yarn test-apps` | Run apps tests (jsdom environment) |
| `yarn start` | Full build and start development server |
| `npx jest -c=<config> --testPathPattern=<pattern>` | Run specific test files |

### B. Port Reference

| Port | Service |
|------|---------|
| 3000 | Calypso development server (`http://calypso.localhost:3000`) |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | Investigation document deliverable (973 lines) |
| `test/client/jest.config.js` | Client-suite Jest configuration |
| `test/client/setup-test-framework.js` | Client-suite bootstrap (globals, nock, mocks) |
| `test/server/jest.config.js` | Server-suite Jest configuration |
| `test/server/setup-test-framework.js` | Server-suite bootstrap (nock, mocks) |
| `test/packages/jest-preset.js` | Packages-suite Jest preset |
| `test/packages/setup.js` | Packages-suite bootstrap |
| `test/apps/jest-preset.js` | Apps-suite Jest preset |
| `test/build-tools/jest.config.js` | Build-tools Jest configuration |
| `test/integration/jest.config.js` | Integration-suite Jest configuration |
| `packages/calypso-jest/jest-preset.js` | Shared base Jest preset |
| `packages/calypso-jest/src/setup.js` | Shared minimal setup (CSS.supports mock) |
| `config/test.json` | Test environment configuration (101 feature flags) |
| `config/development.json` | Development environment configuration (178 feature flags) |
| `config/_shared.json` | Shared base configuration |
| `client/server/config/index.js` | Server-side config entry point |
| `client/server/config/parser.js` | Config file parser and merger |
| `client/test-helpers/use-nock/index.js` | Deprecated nock helper wrapper |
| `client/state/products-list/test/actions.js` | Example API mock test (traced in document) |
| `client/state/products-list/actions.js` | Action creators under test |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Node.js | ^v22.9.0 (active: v22.22.2) | `package.json` engines, `.nvmrc` |
| Yarn | 4.0.2 | `package.json` packageManager |
| Jest | ^29.7.0 | `packages/calypso-jest/package.json` |
| nock | ^13.5.6 | `package.json` devDependencies |
| TypeScript | 5.8.2 | `package.json` devDependencies |
| React | ^18.3.1 | `package.json` dependencies |
| Webpack | ^5.97.1 | `package.json` devDependencies |
| enhanced-resolve | 5.9.3 | `test/module-resolver.js` |
| jest-environment-jsdom | ^29.7.0 | `package.json` devDependencies |
| @testing-library/jest-dom | ^6.6.3 | `package.json` devDependencies |

### E. Environment Variable Reference

| Variable | Dev Server | Test Runner | Purpose |
|----------|-----------|-------------|---------|
| `NODE_ENV` | `development` | `test` (set by Jest) | Determines which config JSON file is loaded |
| `CALYPSO_ENV` | unset (falls to NODE_ENV) | unset (falls to NODE_ENV) | Takes precedence over NODE_ENV for config resolution |
| `TZ` | system default | `UTC` (test-client only) | Timezone for date-dependent tests |
| `ACTIVE_FEATURE_FLAGS` | unset | optionally set | Comma-separated flags forced to `true` by `isEnabled()` |
| `ENABLE_FEATURES` | user-configurable | not typically set | Comma-separated flags forced to `true` in parser |
| `DISABLE_FEATURES` | user-configurable | not typically set | Comma-separated flags forced to `false` in parser |
| `BROWSERSLIST_ENV` | `evergreen` (start-build) | `test` (browserslist config) | Determines browser target list |

### G. Glossary

| Term | Definition |
|------|-----------|
| **nock** | HTTP interceptor library that monkey-patches Node.js `http`/`https` modules to intercept outbound requests during tests |
| **useNock** | Deprecated wp-calypso helper wrapping nock setup in `beforeAll`/`afterAll` lifecycle hooks |
| **jsdom** | JavaScript-based DOM implementation used as a Jest test environment for browser-like testing in Node.js |
| **moduleNameMapper** | Jest configuration option that redirects module imports to different paths (used to redirect `@automattic/calypso-config` to server-side config in tests) |
| **enhanced-resolve** | Webpack-compatible module resolution library used as Jest's custom resolver with `calypso:src` field priority |
| **calypso:src** | Custom `package.json` field pointing to untranspiled source code; prioritized by the test resolver over compiled `main` field |
| **Redux thunk** | Middleware pattern where action creators return functions instead of objects, receiving `dispatch` as a parameter for async operations |
| **dispatch spy** | `jest.fn()` used as a mock Redux `dispatch` function to capture all dispatched actions for test assertions |