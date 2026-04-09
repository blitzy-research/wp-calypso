# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers specific behavioral questions about the `@automattic/explat-client` package — Automattic's standalone experiment assignment client used in the Calypso monorepo.

**Category:** Create new documentation
**Documentation Type:** Technical Q&A / Behavioral Reference Document

The user is onboarding onto the `wp-calypso` repository and integrating the ExPlat experiment assignment client into a feature. Before writing integration code, they require a verified understanding of four critical behavioral domains:

- **Failure Handling:** What actually happens when the ExPlat server is unavailable or takes too long to respond? What does the response object look like, and what variation is assigned to the user?
- **Concurrent Request Deduplication:** If multiple parts of the application request the same experiment assignment simultaneously, how many network calls are actually made?
- **Caching Behavior:** When the same experiment is requested multiple times in quick succession, does each request trigger a network call? What happens after the cache TTL expires?
- **Async/Sync API Interplay:** What happens if the synchronous getter (`dangerouslyGetExperimentAssignment`) is called before the async loader (`loadExperimentAssignment`) has completed? Does this break the application or is it handled gracefully?

Additionally, the user requested verification that the package's existing tests are all passing before relying on them as behavioral evidence.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL:** Do not modify any source files in the repository. The implementation rule `SWE-AtlasQnA-Repo` explicitly states: "Do not modify any existing files in the source repository."
- **Output format:** Create a new markdown document named `wp-calypso_be7e5cc64162.md` in the `blitzy/documentation` directory.
- **Evidence-based answers:** Do not make assumptions; base all answers on the code as the source of truth.
- **Provide rationale:** Include thinking and rationale behind all answers.
- **Temporary scripts:** The user permits creating temporary test scripts to observe behavior, but these must be cleaned up afterward.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer the failure handling questions**, we will analyze the `loadExperimentAssignment` method in `packages/explat-client/src/create-explat-client.ts` (lines 113–183), the `timeoutPromise` function in `packages/explat-client/src/internal/timing.ts` (lines 23–36), and the `createFallbackExperimentAssignment` factory in `packages/explat-client/src/internal/experiment-assignments.ts` (lines 29–38), cross-referencing with test evidence from `packages/explat-client/src/test/create-explat-client.ts` (lines 184–248).
- To **answer the concurrent request question**, we will analyze the `asyncOneAtATime` wrapper in `packages/explat-client/src/internal/timing.ts` (lines 44–54) and its application in `createWrappedExperimentAssignmentFetchAndStore` in `packages/explat-client/src/create-explat-client.ts` (lines 82–94), corroborated by tests at lines 292–474.
- To **answer the caching behavior question**, we will analyze the TTL-based caching via `isAlive` in `packages/explat-client/src/internal/experiment-assignments.ts` (lines 8–14) and the localStorage persistence layer in `packages/explat-client/src/internal/experiment-assignment-store.ts`, corroborated by multi-use tests at lines 292–368.
- To **answer the sync/async interplay question**, we will analyze `dangerouslyGetExperimentAssignment` in `packages/explat-client/src/create-explat-client.ts` (lines 184–224) and `dangerouslyGetMaybeLoadedExperimentAssignment` (lines 225–249), referencing test cases at lines 477–677.
- To **verify tests pass**, we will create the `blitzy/documentation` output based on the confirmed test run: **9 suites, 81 tests, all passing**.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation topics should be addressed to provide a complete understanding:

- **Fallback ExperimentAssignment shape:** The exact structure of the object returned during failures (including the `isFallbackExperimentAssignment` marker), which is critical for integration code to detect degraded states.
- **Minimum TTL enforcement:** The 60-second floor (`minimumTtl`) on all assignments, including fallbacks, which limits retry frequency during server outages.
- **The `EXPERIMENT_FETCH_TIMEOUT` constant and the A/B test on timeout duration:** The fetch timeout is either 5,000ms or 10,000ms (randomly chosen per request), which is relevant for understanding latency behavior.
- **SSR context safety:** The client automatically switches to a safe dummy implementation in SSR contexts, which is relevant for server-rendered feature code.
- **Stale assignment recovery:** During fetch failures, the client attempts to return any stale cached assignment from localStorage before generating a new fallback, which is important for offline-capable features.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals the following documentation structure relevant to the ExPlat client:

**Package-level documentation discovered:**

| File | Location | Coverage Status |
|------|----------|-----------------|
| `README.md` | `packages/explat-client/README.md` | Covers high-level API contract, `loadExperimentAssignment` and `dangerouslyGetExperimentAssignment` usage, `ExperimentAssignment` type semantics; does NOT cover failure behavior details, caching internals, or concurrency handling |
| `CHANGELOG.md` | `packages/explat-client/CHANGELOG.md` | Release history from 0.0.1 to 0.1.0; records key behavior changes (log instead of throw, localStorage storage, timeout shortening) |
| `README.md` | `packages/explat-client-react-helpers/README.md` | React helper documentation covering `useExperiment`, `<Experiment>`, `<ProvideExperimentData>` components and hooks |

**Repository-wide documentation infrastructure:**

- Documentation directory: `docs/` at repo root with Markdown files (no documentation generator configuration detected)
- No `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` found
- No automated API documentation generation tools detected for the explat-client
- Inline JSDoc-style comments present in source files but no doc generation pipeline
- Mermaid or PlantUML tooling: Not detected in the package
- Documentation hosting/deployment: Not configured for the package

**Documentation framework:** Plain Markdown files with no build tooling — documentation is read directly from the repository.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for identifying code to document:

- **Client factory:** `packages/explat-client/src/create-explat-client.ts` — the main `createExPlatClient` function (284 lines) containing `loadExperimentAssignment`, `dangerouslyGetExperimentAssignment`, and `dangerouslyGetMaybeLoadedExperimentAssignment`
- **Type contracts:** `packages/explat-client/src/types.ts` — `ExperimentAssignment` and `Config` interfaces
- **Internal modules:**
  - `packages/explat-client/src/internal/timing.ts` — `monotonicNow`, `timeoutPromise`, `asyncOneAtATime`
  - `packages/explat-client/src/internal/requests.ts` — `fetchExperimentAssignment`, `localStorageCachedGetAnonId`, response validation
  - `packages/explat-client/src/internal/experiment-assignments.ts` — `isAlive`, `minimumTtl`, `createFallbackExperimentAssignment`
  - `packages/explat-client/src/internal/experiment-assignment-store.ts` — localStorage persistence (`storeExperimentAssignment`, `retrieveExperimentAssignment`, `removeExpiredExperimentAssignments`)
  - `packages/explat-client/src/internal/local-storage.ts` — polyfilled localStorage for non-browser contexts
  - `packages/explat-client/src/internal/validations.ts` — runtime validation helpers (`isName`, `isExperimentAssignment`, `validateExperimentAssignment`)
- **Entry point:** `packages/explat-client/src/index.ts` — environment-aware factory selector (browser vs. SSR)

**Test files analyzed:**

| Test File | Scope | Tests |
|-----------|-------|-------|
| `src/test/create-explat-client.ts` | Client factory, loading, caching, fallback, dangerous getters | Core behavioral tests |
| `src/test/create-ssr-safe-dummy-explat-client.ts` | SSR dummy client fallback behavior | SSR safety |
| `src/test/index.ts` | Entry point environment routing | Module wiring |
| `src/internal/test/timing.ts` | Monotonic clock, timeouts, asyncOneAtATime | Timing primitives |
| `src/internal/test/requests.ts` | Fetch workflow, response validation, anonId caching | Request layer |
| `src/internal/test/experiment-assignments.ts` | TTL boundary, fallback creation | Assignment lifecycle |
| `src/internal/test/experiment-assignment-store.ts` | localStorage persistence, cleanup | Storage layer |
| `src/internal/test/local-storage.ts` | Polyfilled storage semantics | Storage polyfill |
| `src/internal/test/validations.ts` | Name and assignment validation | Validation rules |

### 0.2.3 Test Verification Results

All package tests were executed with the following results:

```
Test Suites: 9 passed, 9 total
Tests:       81 passed, 81 total
Snapshots:   23 passed, 23 total
Time:        3.755 s
```

**Environment:** Node v22.9.0, Jest 29.7.0, yarn 4.0.2

All 81 tests pass, confirming the behavioral contracts documented in the test suite are valid and can be relied upon as evidence for the Q&A documentation.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation targets four behavioral domains of the `@automattic/explat-client` package. Each question maps to specific source code functions and test evidence:

**Module: `packages/explat-client/src/create-explat-client.ts`**
- Public APIs: `loadExperimentAssignment`, `dangerouslyGetExperimentAssignment`, `dangerouslyGetMaybeLoadedExperimentAssignment`
- Current documentation: `README.md` covers API signatures and basic usage; does NOT document failure behavior, caching internals, or concurrency semantics
- Documentation needed: Detailed behavioral reference for failure handling, fallback response shape, concurrent request deduplication, TTL caching mechanics, and sync/async interplay

**Module: `packages/explat-client/src/internal/timing.ts`**
- Public APIs: `asyncOneAtATime`, `timeoutPromise`, `monotonicNow`
- Current documentation: Inline JSDoc only; no external documentation
- Documentation needed: Explanation of how `asyncOneAtATime` deduplicates concurrent requests and how `timeoutPromise` enforces fetch deadlines

**Module: `packages/explat-client/src/internal/experiment-assignments.ts`**
- Public APIs: `isAlive`, `createFallbackExperimentAssignment`, `minimumTtl`
- Current documentation: Inline JSDoc only
- Documentation needed: TTL boundary behavior (strict less-than comparison), fallback object structure, minimum TTL clamping

**Module: `packages/explat-client/src/internal/experiment-assignment-store.ts`**
- Public APIs: `storeExperimentAssignment`, `retrieveExperimentAssignment`, `removeExpiredExperimentAssignments`
- Current documentation: Inline JSDoc only
- Documentation needed: How localStorage caching works, stale assignment recovery during failures, race-condition protection

**Module: `packages/explat-client/src/internal/requests.ts`**
- Public APIs: `fetchExperimentAssignment`, `localStorageCachedGetAnonId`
- Current documentation: Inline JSDoc only
- Documentation needed: Response validation pipeline, TTL normalization, empty-variations fallback path

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the specific documentation gaps to be filled are:

- **Failure handling mechanics:** No existing documentation explains the multi-level error recovery chain in `loadExperimentAssignment` (try fetch → catch initial error → try stale cache → catch fallback error → last resort fallback)
- **Fallback ExperimentAssignment shape:** The `README.md` mentions fallback behavior conceptually but does not document the exact object shape returned during failures (including `variationName: null`, `isFallbackExperimentAssignment: true`, `ttl: 60`)
- **Concurrent request deduplication:** No documentation explains the `asyncOneAtATime` pattern that ensures a single in-flight fetch per experiment, or that subsequent callers share the same Promise
- **TTL-based caching behavior:** The `README.md` mentions "respects the server returned TTL" but does not explain the in-memory + localStorage caching layer, the `isAlive` predicate, or the 60-second minimum TTL floor
- **Sync getter before async load:** The `README.md` mentions the load-before-get requirement but does not document the specific graceful fallback behavior when this contract is violated
- **Timeout A/B experiment:** An undocumented runtime A/B test randomly selects between 5,000ms and 10,000ms fetch timeouts (line 136 of `create-explat-client.ts`)

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/wp-calypso_be7e5cc64162.md` will be structured as a comprehensive Q&A reference organized by the user's four behavioral questions:

```text
blitzy/documentation/wp-calypso_be7e5cc64162.md
  - Test Verification (confirming all 81 tests pass)
  - Q1: Failure Handling and Server Unavailability
  - Q2: Concurrent Request Deduplication
  - Q3: Caching Behavior and TTL
  - Q4: Sync Getter Before Async Load
  - Summary of Key Behavioral Guarantees
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract behavioral guarantees from the source code in `packages/explat-client/src/create-explat-client.ts`, specifically the error recovery chain in `loadExperimentAssignment` (lines 113–183)
- Extract concurrency semantics from `packages/explat-client/src/internal/timing.ts`, specifically `asyncOneAtATime` (lines 44–54)
- Extract TTL and caching logic from `packages/explat-client/src/internal/experiment-assignments.ts` (`isAlive`, `minimumTtl`) and `packages/explat-client/src/internal/experiment-assignment-store.ts` (`storeExperimentAssignment`, `retrieveExperimentAssignment`)
- Generate evidence by citing test assertions from the test suite in `packages/explat-client/src/test/create-explat-client.ts`

**Documentation Standards:**
- Markdown formatting with proper headers (# ## ###)
- Code examples using fenced code blocks with TypeScript syntax highlighting
- Source citations as inline references: `Source: packages/explat-client/src/create-explat-client.ts:LineNumber`
- Each answer section includes rationale/thinking behind the conclusion
- All claims traced to specific source code lines or test assertions

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams:

- **Failure handling recovery chain:** A flowchart showing the try/catch cascade from initial fetch through stale cache fallback to last resort fallback
- **Concurrent request deduplication:** A sequence diagram illustrating how `asyncOneAtATime` shares a single Promise among multiple callers
- **TTL caching lifecycle:** A flowchart showing the decision tree when `loadExperimentAssignment` is called (cache alive returns cached; cache expired triggers new fetch; fetch fails returns fallback)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `packages/explat-client/src/create-explat-client.ts`, `packages/explat-client/src/internal/timing.ts`, `packages/explat-client/src/internal/experiment-assignments.ts`, `packages/explat-client/src/internal/experiment-assignment-store.ts`, `packages/explat-client/src/internal/requests.ts`, `packages/explat-client/src/types.ts`, `packages/explat-client/src/test/create-explat-client.ts` | Comprehensive behavioral Q&A document answering all four user questions about failure handling, concurrent requests, caching, and sync/async interplay, with code evidence and test verification |

No existing documentation files are updated or deleted. No documentation configuration files require changes.

### 0.5.2 New Documentation File Detail

**File:** `blitzy/documentation/wp-calypso_be7e5cc64162.md`
**Type:** Technical Q&A / Behavioral Reference Document
**Source Code:** All files under `packages/explat-client/src/`

**Sections:**
- **Test Verification:** Report confirming all 81 tests pass across 9 suites under Node v22.9.0 and Jest 29.7.0
- **Question 1 — Failure Handling:**
  - The multi-level error recovery chain in `loadExperimentAssignment` (from `create-explat-client.ts` lines 113–183)
  - The `timeoutPromise` mechanism with the 5,000ms/10,000ms randomized timeout (from `timing.ts` lines 23–36 and `create-explat-client.ts` lines 134–145)
  - The fallback `ExperimentAssignment` object shape with `variationName: null` and `isFallbackExperimentAssignment: true` (from `experiment-assignments.ts` lines 29–38)
  - The stale cache recovery path (from `create-explat-client.ts` lines 160–165)
  - Test evidence: fetch failure test (lines 184–214), timeout test (lines 216–248), logError-throws test (lines 250–289)
- **Question 2 — Concurrent Request Deduplication:**
  - The `asyncOneAtATime` single-flight wrapper (from `timing.ts` lines 44–54)
  - Per-experiment deduplication map (from `create-explat-client.ts` lines 82–94)
  - Test evidence: multiple-use TTL test (lines 292–368), failed-request sharing test (lines 370–474)
- **Question 3 — Caching Behavior:**
  - TTL-based `isAlive` predicate using monotonic timestamps (from `experiment-assignments.ts` lines 8–14)
  - localStorage persistence via `storeExperimentAssignment`/`retrieveExperimentAssignment` (from `experiment-assignment-store.ts` lines 21–57)
  - The 60-second minimum TTL floor (from `experiment-assignments.ts` line 21)
  - Test evidence: TTL respect test (lines 293–368)
- **Question 4 — Sync Getter Before Async Load:**
  - `dangerouslyGetExperimentAssignment` returns a fallback with `variationName: null` when load has not completed (from `create-explat-client.ts` lines 184–224)
  - `dangerouslyGetMaybeLoadedExperimentAssignment` returns `null` when load has not completed (from `create-explat-client.ts` lines 225–249)
  - Neither method throws in production (errors are caught and logged in development mode)
  - Test evidence: not-loaded tests (lines 503–526), in-flight loading tests (lines 528–558, 642–658), loaded return tests (lines 560–574, 660–677)
- **Summary:** Key behavioral guarantees distilled into a quick-reference list

**Diagrams:**
- Failure handling flowchart (Mermaid)
- Concurrent request sequence diagram (Mermaid)
- TTL caching decision tree (Mermaid)

**Key Citations:**
- `packages/explat-client/src/create-explat-client.ts`
- `packages/explat-client/src/internal/timing.ts`
- `packages/explat-client/src/internal/experiment-assignments.ts`
- `packages/explat-client/src/internal/experiment-assignment-store.ts`
- `packages/explat-client/src/internal/requests.ts`
- `packages/explat-client/src/types.ts`
- `packages/explat-client/src/test/create-explat-client.ts`
- `packages/explat-client/README.md`

### 0.5.3 Documentation Configuration Updates

No documentation configuration files require updates. The `blitzy/documentation/` directory is created as part of this task and requires no build pipeline integration.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to the documentation exercise and were used to verify behavioral claims:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm | `@automattic/explat-client` | 0.1.0 | The package under documentation — standalone ExPlat experiment assignment client |
| npm | `jest` | ^29.7.0 | Test runner used to verify all 81 behavioral tests pass |
| npm | `jest-environment-jsdom` | 29.7.0 | jsdom test environment required by localStorage-dependent test suites |
| npm | `typescript` | ^5.8.2 | TypeScript compiler for the package source and test compilation |
| npm | `tslib` | ^2.3.0 | Runtime dependency of the explat-client package |
| npm | `@automattic/calypso-polyfills` | workspace:^ | Polyfills for `regeneratorRuntime` required by async test code |
| npm | `@automattic/calypso-jest` | workspace:^ | Shared Jest preset configuration for the monorepo |
| npm | `@automattic/calypso-babel-config` | workspace:^ | Babel configuration for transpiling TypeScript test files |
| npm | `@automattic/calypso-typescript-config` | workspace:^ | Shared TypeScript base configuration |
| system | Node.js | 22.9.0 | Runtime matching the project's `.nvmrc` and `package.json` engines field |
| system | Yarn | 4.0.2 | Package manager matching the project's `packageManager` field and `.yarnrc.yml` |

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The new document `blitzy/documentation/wp-calypso_be7e5cc64162.md` is a standalone file with no cross-links to maintain.

All internal references in the document use relative paths to the `packages/explat-client/` directory within the monorepo, and cite specific line numbers from the source files analyzed.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (before this task):**
- The `README.md` documents API signatures and basic usage: **partial** (~40% of the behavioral surface relevant to the user's questions)
- Failure handling documented: **0%** — no existing documentation covers the error recovery chain, fallback shape, or timeout behavior
- Concurrent request deduplication documented: **0%** — no existing documentation explains `asyncOneAtATime` or single-flight semantics
- Caching behavior documented: **~10%** — the README mentions "respects the server returned TTL" but does not explain the localStorage caching layer, `isAlive` predicate, or minimum TTL floor
- Sync/async interplay documented: **~20%** — the README mentions load-before-get requirements but does not document the specific graceful fallback when this contract is violated

**Target coverage (after this task):**
- All four user questions answered with full code-traced evidence: **100%**
- Each behavioral claim supported by specific source file and line number citations
- Each behavioral claim corroborated by specific passing test assertions

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every behavioral claim references the specific source code function and line number
- Every claim is corroborated by at least one passing test assertion
- The exact shape of the fallback `ExperimentAssignment` object is documented with all fields and their values
- The error recovery chain in `loadExperimentAssignment` is documented step by step

**Accuracy validation:**
- All behavioral claims derived from source code analysis (not assumptions)
- All test evidence confirmed by running the test suite in the correct environment (Node 22.9.0, Jest 29.7.0)
- Test suite results: 9 suites, 81 tests, 23 snapshots — all passing

**Clarity standards:**
- Technical accuracy with accessible language for a developer onboarding to the repo
- Progressive disclosure: brief answers first, then detailed code walkthroughs
- Consistent terminology matching the package's naming conventions (`ExperimentAssignment`, `variationName`, `loadExperimentAssignment`, etc.)

**Maintainability:**
- Source citations with file paths and line numbers for traceability
- Structured as a Q&A document that maps directly to the user's questions
- Self-contained with no external dependencies for rendering

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid diagram per major behavioral question (failure handling, concurrency, caching)
- Code snippets showing the key source code patterns (e.g., the `asyncOneAtATime` wrapper, the `createFallbackExperimentAssignment` factory, the `isAlive` predicate)
- Exact object literal examples showing the `ExperimentAssignment` shape in failure scenarios

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — the sole deliverable document

**Source files analyzed (read-only) for documentation content:**
- `packages/explat-client/src/create-explat-client.ts` — client factory, `loadExperimentAssignment`, dangerous getters, SSR dummy
- `packages/explat-client/src/index.ts` — environment-aware entry point
- `packages/explat-client/src/types.ts` — `ExperimentAssignment` and `Config` interfaces
- `packages/explat-client/src/internal/timing.ts` — `asyncOneAtATime`, `timeoutPromise`, `monotonicNow`
- `packages/explat-client/src/internal/experiment-assignments.ts` — `isAlive`, `minimumTtl`, `createFallbackExperimentAssignment`
- `packages/explat-client/src/internal/experiment-assignment-store.ts` — localStorage persistence layer
- `packages/explat-client/src/internal/requests.ts` — fetch workflow, response validation, anonId caching
- `packages/explat-client/src/internal/local-storage.ts` — storage polyfill
- `packages/explat-client/src/internal/validations.ts` — runtime validation helpers
- `packages/explat-client/src/internal/test-common.ts` — shared test fixtures and helpers
- `packages/explat-client/src/test/create-explat-client.ts` — main client behavioral tests
- `packages/explat-client/src/test/create-ssr-safe-dummy-explat-client.ts` — SSR dummy tests
- `packages/explat-client/src/test/index.ts` — entry point routing tests
- `packages/explat-client/src/internal/test/timing.ts` — timing primitive tests
- `packages/explat-client/src/internal/test/requests.ts` — request layer tests
- `packages/explat-client/src/internal/test/experiment-assignments.ts` — assignment lifecycle tests
- `packages/explat-client/src/internal/test/experiment-assignment-store.ts` — storage tests
- `packages/explat-client/src/internal/test/local-storage.ts` — localStorage polyfill tests
- `packages/explat-client/src/internal/test/validations.ts` — validation tests
- `packages/explat-client/README.md` — existing package documentation
- `packages/explat-client/CHANGELOG.md` — version history
- `packages/explat-client/package.json` — package manifest
- `packages/explat-client/jest.config.js` — test configuration

**Test execution:**
- Running the full `@automattic/explat-client` test suite to verify all tests pass

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing source files in the repository will be modified (per the user's explicit instruction and the `SWE-AtlasQnA-Repo` rule)
- **Test file modifications:** No test files will be modified
- **Feature additions or code refactoring:** No functional changes to the ExPlat client
- **Other packages:** The `@automattic/explat-client-react-helpers` package is not covered unless directly relevant to the four user questions
- **Redux integration layer:** The `client/state/explat-experiments/` Redux wiring is out of scope as it is a separate integration concern
- **Calypso-specific experiment implementations:** Individual experiment implementations like `client/lib/remove-duplicate-views-experiment/` are out of scope
- **Documentation for non-behavioral concerns:** Performance benchmarking, security analysis, or deployment documentation for the ExPlat client
- **Documentation tooling setup:** No documentation build pipelines, generators, or hosting configuration will be created or modified
- **Temporary test scripts:** If created for observation, they will be cleaned up as instructed

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Test execution command:** `cd packages/explat-client && npx jest --ci --watchAll=false --verbose`
- **Environment requirements:** Node v22.9.0 (from `.nvmrc` and `package.json` engines), Yarn 4.0.2 (from `packageManager` field)
- **Dependency installation:** `yarn workspaces focus @automattic/explat-client @automattic/calypso-jest @automattic/calypso-polyfills @automattic/calypso-typescript-config @automattic/calypso-babel-config` (from project root), plus `jest-environment-jsdom` for jsdom test suites
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every behavioral claim must reference the source file path and line number(s) from which it is derived
- **Style guide:** Follow the plain-Markdown documentation style established by the existing `packages/explat-client/README.md`, using TypeScript code blocks for examples and Mermaid for diagrams
- **Output location:** `blitzy/documentation/wp-calypso_be7e5cc64162.md` (per the `SWE-AtlasQnA-Repo` implementation rule)
- **Output naming:** `<source_branch_name>.md` where the source branch name is `wp-calypso_be7e5cc64162`

### 0.9.2 Build and Environment Setup Notes

The following setup steps were required to execute the test suite in this environment:

- Install Node v22.9.0 via nvm (the pre-installed v20.20.2 does not satisfy the `^v22.9.0` constraint)
- Enable corepack for Yarn 4.0.2 via `corepack enable` and `corepack prepare yarn@4.0.2 --activate`
- Run focused workspace install: `yarn workspaces focus` targeting the required packages
- Add `jest-environment-jsdom` which is no longer bundled with Jest 28+ but is required by several test suites annotated with `@jest-environment jsdom`
- Resolve transitive workspace dependencies: `@automattic/calypso-babel-config` (required by root `babel.config.js`), `@automattic/calypso-jest` (required by `test/packages/jest-preset.js`)

All tests passed successfully after this setup: **9 suites, 81 tests, 23 snapshots — all passing**.

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user or derived from the implementation rule `SWE-AtlasQnA-Repo`:

- **Do not modify any existing files in the source repository.** This is both a user instruction and an implementation rule requirement. The only new file created is the output document in `blitzy/documentation/`.
- **Do not make assumptions; base answers on the code as the truth.** Every behavioral claim in the output document must be traceable to a specific source code function, line number, or test assertion.
- **Provide thinking and rationale behind the answers.** Each answer section must explain the reasoning chain from source code to conclusion, not just state the conclusion.
- **Temporary test scripts must be cleaned up afterward.** If any temporary scripts are created to observe runtime behavior, they must be deleted before the task is complete.
- **Place the generated document in the `blitzy/documentation` directory.** The document is named `wp-calypso_be7e5cc64162.md` matching the source branch name convention.
- **Comprehensively answer the questions posed in the prompt.** All four behavioral questions (failure handling, concurrent requests, caching, sync/async interplay) must be answered completely with evidence.

## 0.11 References

### 0.11.1 Source Files Analyzed

The following files were read and analyzed to derive the behavioral documentation:

**Core implementation files:**

| File Path | Purpose | Key Functions/Exports |
|-----------|---------|----------------------|
| `packages/explat-client/src/create-explat-client.ts` | Client factory and main API methods | `createExPlatClient`, `createSsrSafeDummyExPlatClient`, `loadExperimentAssignment`, `dangerouslyGetExperimentAssignment`, `dangerouslyGetMaybeLoadedExperimentAssignment` |
| `packages/explat-client/src/index.ts` | Environment-aware entry point | `createExPlatClient` (dispatches to browser or SSR factory) |
| `packages/explat-client/src/types.ts` | TypeScript interfaces | `ExperimentAssignment`, `Config` |
| `packages/explat-client/src/internal/timing.ts` | Timing and concurrency primitives | `monotonicNow`, `timeoutPromise`, `asyncOneAtATime` |
| `packages/explat-client/src/internal/experiment-assignments.ts` | Assignment lifecycle helpers | `isAlive`, `minimumTtl`, `createFallbackExperimentAssignment` |
| `packages/explat-client/src/internal/experiment-assignment-store.ts` | localStorage persistence layer | `storeExperimentAssignment`, `retrieveExperimentAssignment`, `removeExpiredExperimentAssignments` |
| `packages/explat-client/src/internal/requests.ts` | Network request orchestration | `fetchExperimentAssignment`, `localStorageCachedGetAnonId`, `isFetchExperimentAssignmentResponse` |
| `packages/explat-client/src/internal/local-storage.ts` | Storage polyfill | `polyfilledLocalStorage`, default export |
| `packages/explat-client/src/internal/validations.ts` | Runtime validation | `isName`, `isExperimentAssignment`, `validateExperimentAssignment` |

**Test files:**

| File Path | Coverage Area |
|-----------|--------------|
| `packages/explat-client/src/test/create-explat-client.ts` | Client factory, loading, caching, fallback, dangerous getters (main behavioral evidence) |
| `packages/explat-client/src/test/create-ssr-safe-dummy-explat-client.ts` | SSR dummy client fallback behavior |
| `packages/explat-client/src/test/index.ts` | Entry point environment routing |
| `packages/explat-client/src/internal/test/timing.ts` | Monotonic clock, timeouts, asyncOneAtATime |
| `packages/explat-client/src/internal/test/requests.ts` | Fetch workflow, response validation, anonId caching |
| `packages/explat-client/src/internal/test/experiment-assignments.ts` | TTL boundary, fallback creation |
| `packages/explat-client/src/internal/test/experiment-assignment-store.ts` | localStorage persistence, cleanup |
| `packages/explat-client/src/internal/test/local-storage.ts` | Polyfilled storage semantics |
| `packages/explat-client/src/internal/test/validations.ts` | Name and assignment validation |

**Documentation and configuration files:**

| File Path | Purpose |
|-----------|---------|
| `packages/explat-client/README.md` | Existing package-level documentation |
| `packages/explat-client/CHANGELOG.md` | Version history and behavioral change log |
| `packages/explat-client/package.json` | Package manifest with version, dependencies, scripts |
| `packages/explat-client/jest.config.js` | Jest test configuration |
| `packages/explat-client/src/internal/test-common.ts` | Shared test fixtures and environment helpers |
| `test/packages/jest-preset.js` | Monorepo shared Jest preset |
| `.nvmrc` | Node version requirement (22.9.0) |
| `package.json` (root) | Root monorepo manifest with engines and packageManager fields |

**Folders explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (root) | 0 | Repository root with monorepo workspace configuration |
| `packages/explat-client/` | 1 | ExPlat client package root |
| `packages/explat-client/src/` | 2 | Source root with entry point, factory, and types |
| `packages/explat-client/src/internal/` | 3 | Private runtime helpers and shared utilities |
| `packages/explat-client/src/internal/test/` | 4 | Internal module test suites |
| `packages/explat-client/src/test/` | 3 | Package-level test suites |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 Tech Spec Sections Referenced

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Repository overview and project context (Calypso monorepo, WordPress.com) |
| 3.2 Frameworks and Libraries | Confirmed React and package ecosystem context |

