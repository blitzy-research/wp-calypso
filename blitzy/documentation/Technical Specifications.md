# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that serves as a comprehensive onboarding guide to the Calypso monorepo's testing infrastructure. The user is a new contributor seeking to understand, through investigation and evidence-based documentation, how the test environment differs from the development environment, what test-only runtime artifacts exist, how network isolation works during tests, and how configuration (feature flags) resolves differently across environments.

**Documentation Category:** Create new documentation
**Documentation Type:** Technical investigation guide / Onboarding reference for testing infrastructure

The specific documentation requirements, restated with enhanced clarity, are:

- **Development server verification** — Confirm the dev server starts successfully, documenting the boot process and output
- **Test environment anatomy** — Document the complete test environment bootstrap compared to normal development, including all setup files, globals, polyfills, and environment variables that exist only during test execution
- **Network isolation mechanics** — Explain what happens when code attempts network requests during tests, tracing the `nock.disableNetConnect()` enforcement, the `beforeAll`/`afterAll` lifecycle hooks, and the `wpcom-proxy-request` mock
- **API mock tracing** — Identify and trace through an actual test that mocks an API call, showing exactly how the mocked response flows through the action creator back to the test assertion
- **Configuration/feature flag divergence** — Document how `@automattic/calypso-config` resolves differently in tests vs. development, proving with evidence that `config/test.json` provides different values than `config/development.json`
- **Test-controlled configuration** — Show how tests control what config returns, including `jest.mock('@automattic/calypso-config')` and module name mapper redirections
- **No file modifications** — All findings must be obtained through read-only investigation; temporary scripts must be cleaned up

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: Read-only constraint** — "Don't modify any repository files." The user explicitly forbids modifying any existing repository files. All investigation must be passive.
- **Temporary script policy** — "If you need to create temporary scripts to investigate behavior, clean them up when done." Temporary files are permitted for investigation but must be removed before completion.
- **Implementation rule: SWE-AtlasQnA-Repo** — The user's project rule requires creating a markdown document named `wp-calypso_be7e5cc64162.md` in the `blitzy/documentation` directory that comprehensively answers the posed questions, provides thinking and rationale behind answers, bases all answers on the code as truth, and does not modify any existing source repository files.
- **Evidence-based answers** — "Do not make assumptions, base your answers on the code as the truth." Every claim in the generated documentation must reference specific source files and line numbers.
- **Proof requirement** — The user explicitly asks to "show me proof" and "trace how" — the document must include concrete code excerpts and traced execution paths, not abstract descriptions.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document dev server verification**, we will create a section in `blitzy/documentation/wp-calypso_be7e5cc64162.md` that runs and captures output from `yarn start` (or the appropriate start command) and records boot diagnostics
- To **document test environment anatomy**, we will create a detailed comparison section analyzing `test/client/jest.config.js`, `test/client/setup-test-framework.js`, `test/server/jest.config.js`, `test/server/setup-test-framework.js`, `test/packages/setup.js`, and `test/packages/jest-preset.js` against the development boot path
- To **document network isolation**, we will trace the `nock.disableNetConnect()` call chain in `test/client/setup-test-framework.js` (line 9) and `test/server/setup-test-framework.js` (line 4), the `beforeAll`/`afterAll` lifecycle hooks, and the `wpcom-proxy-request` mock
- To **trace an API mock example**, we will select `client/state/user-suggestions/test/actions.js` as the reference test, tracing through the nock interceptor setup, the `requestUserSuggestions` thunk invocation, and the dispatch spy assertions
- To **document configuration divergence**, we will create a side-by-side comparison of `config/test.json` vs `config/development.json`, analyze the `moduleNameMapper` entries that redirect `@automattic/calypso-config` to `client/server/config/index.js`, and document the `client/server/config/parser.js` environment selection logic
- To **show test-controlled config**, we will document how `jest.mock('@automattic/calypso-config')` is used in tests and how the parser selects the `test` environment profile via `CALYPSO_ENV` or `NODE_ENV`

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- **Module resolver documentation** — The custom `test/module-resolver.js` using `enhanced-resolve` with `calypso:src` field priority is essential context for understanding how tests resolve monorepo package imports differently than production builds
- **Suite-specific setup differences** — The client test setup (`test/client/setup-test-framework.js`) installs significantly more globals and polyfills than the server setup (`test/server/setup-test-framework.js`) or the packages setup (`test/packages/setup.js`); this asymmetry needs documenting
- **Shared Jest preset chain** — The `@automattic/calypso-jest` preset in `packages/calypso-jest/jest-preset.js` is the shared base for all suites, and understanding it is prerequisite context for the environment comparison
- **`useNock` helper pattern** — The `client/test-helpers/use-nock/index.js` helper wraps nock lifecycle management and is used by many action creator tests; its deprecation status and recommended replacement pattern should be noted
- **Feature flag environment variable overrides** — The `ENABLE_FEATURES`, `DISABLE_FEATURES`, and `ACTIVE_FEATURE_FLAGS` environment variable mechanisms in the config parser (`client/server/config/parser.js`) and in `create-calypso-config` (`packages/create-calypso-config/src/index.ts`) form part of the test-vs-development divergence story

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature documentation structure with significant existing coverage of testing concepts, but notable gaps in the specific areas the user is asking about (test-vs-dev environment comparison, network isolation mechanics, and config divergence evidence).

**Documentation framework:** Static Markdown files in `docs/` directory — no documentation generator (mkdocs, Docusaurus, Sphinx) detected
**API documentation tools:** None detected at the project level; JSDoc comments exist inline but no generation pipeline
**Diagram tools:** Mermaid is used throughout the tech spec; no PlantUML or other diagram tooling detected
**Documentation hosting:** GitHub-rendered Markdown (no `.readthedocs.yml`, `mkdocs.yml`, or `docusaurus.config.js` found)

**Existing testing documentation inventory:**

| File | Path | Coverage Status |
|------|------|----------------|
| Testing Guide (entry point) | `docs/testing/index.md` | Links to sub-guides; philosophy overview |
| Testing Overview | `docs/testing/testing-overview.md` | Suite organization, run commands, CI context |
| Unit Tests | `docs/testing/unit-tests.md` | File organization, naming, mocking strategy |
| Component Tests | `docs/testing/component-tests.md` | @testing-library/react patterns, connected components |
| Snapshot Testing | `docs/testing/snapshot-testing.md` | Snapshot usage, update workflow, best practices |
| FAQ | `docs/testing/faq.md` | Toolchain summary, run commands, `.only()` warning |
| Test README | `test/README.md` | Suite directory layout, legacy helper deprecation |
| Config README | `config/README.md` | Config loading, feature flags, environment progression |

**Gaps identified in existing documentation:**
- No document explains what globals, polyfills, and environment variables exist only during test execution
- No document traces the flow of a mocked API response through an action creator
- No document provides a side-by-side comparison of test vs. development configuration values
- No document explains how `nock.disableNetConnect()` works at the framework level or what happens when code tries to make a network request
- No document shows how tests control configuration/feature flag resolution

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code relevant to the investigation:

- **Test setup files:** `test/*/setup-test-framework.js`, `test/*/setup.js`, `test/*/jest.config.js`
- **Config system:** `client/server/config/parser.js`, `client/server/config/index.js`, `packages/create-calypso-config/src/index.ts`, `packages/calypso-config/src/index.ts`
- **Config environment files:** `config/test.json`, `config/development.json`, `config/_shared.json`
- **Test helpers:** `client/test-helpers/use-nock/index.js`
- **Representative action creator tests:** `client/state/user-suggestions/test/actions.js`, `client/state/products-list/test/actions.js`, `client/state/sharing/publicize/test/actions.js`
- **Shared Jest preset:** `packages/calypso-jest/jest-preset.js`
- **Module resolver:** `test/module-resolver.js`

Key directories examined:
- `test/` — Top-level testing infrastructure (7 suite subdirectories: client, server, packages, apps, integration, build-tools, e2e)
- `config/` — Runtime configuration JSON manifests (18 environment-specific files)
- `docs/testing/` — Existing testing documentation (6 Markdown files)
- `packages/calypso-jest/` — Shared Jest preset package
- `packages/create-calypso-config/` — Configuration factory package
- `client/server/config/` — Server-side config parser and entry point

### 0.2.3 Web Search Research Conducted

No external web searches are required for this documentation task. All answers are derived directly from the codebase, as mandated by the user's instruction: "Do not make assumptions, base your answers on the code as the truth." The repository contains complete source code for every component under investigation.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation:**

- **Module: `test/client/setup-test-framework.js`**
  - Public APIs / Key exports: Side-effect-only setup (no exports)
  - Current documentation: Partially described in `docs/testing/unit-tests.md` and `docs/testing/component-tests.md`, but no line-by-line explanation
  - Documentation needed: Complete catalog of every global, polyfill, and mock injected; comparison against server and packages setups

- **Module: `test/server/setup-test-framework.js`**
  - Public APIs / Key exports: Side-effect-only setup
  - Current documentation: Mentioned in `test/README.md` overview
  - Documentation needed: Network isolation mechanics, `wpcom-proxy-request` mock rationale

- **Module: `test/packages/setup.js`**
  - Public APIs / Key exports: Side-effect-only setup
  - Current documentation: None beyond folder summary
  - Documentation needed: Deterministic UUID stub, ResizeObserver polyfill, matchMedia mock

- **Module: `test/client/jest.config.js`**
  - Key configuration: `moduleNameMapper`, `globals`, `testEnvironmentOptions`, `setupFiles`, `setupFilesAfterEnv`
  - Current documentation: Not documented beyond code comments
  - Documentation needed: How `@automattic/calypso-config` is remapped, what `google` and `__i18n_text_domain__` globals mean

- **Module: `client/server/config/parser.js`**
  - Public APIs: `parser(configPath, opts)` → `{ serverData, clientData }`
  - Current documentation: Referenced in `config/README.md` but mechanics not explained
  - Documentation needed: Layered merge logic (`_shared.json` → `{env}.json` → `{env}.local.json`), feature flag override mechanism

- **Module: `client/server/config/index.js`**
  - Public APIs: Default export = `createConfig(serverData)`, `module.exports.clientData`
  - Current documentation: Referenced in `config/README.md`
  - Documentation needed: How `CALYPSO_ENV`/`NODE_ENV` selects the environment profile, why tests see `test.json` values

- **Module: `packages/create-calypso-config/src/index.ts`**
  - Public APIs: `createConfig(data)` → `ConfigApi`, `config.isEnabled()`, `config.enable()`, `config.disable()`
  - Current documentation: `packages/create-calypso-config/README.md`
  - Documentation needed: `ACTIVE_FEATURE_FLAGS` env var override, `NODE_ENV`-dependent missing-key behavior

- **Module: `client/state/user-suggestions/test/actions.js`**
  - Purpose: Representative test demonstrating API mock → action creator → dispatch assertion flow
  - Current documentation: None
  - Documentation needed: Annotated walkthrough tracing nock interceptor → thunk execution → dispatch spy assertions

- **Module: `client/test-helpers/use-nock/index.js`**
  - Public APIs: `useNock(setupCallback)`, re-exported `nock`
  - Current documentation: `client/test-helpers/use-nock/README.md`
  - Documentation needed: Deprecation status, lifecycle hook behavior

**Configuration options requiring documentation:**

- Config file: `config/test.json` — `env_id: "test"`, feature flags differing from development (e.g., `checkout/checkout-version: false` in test vs `true` in development)
- Config file: `config/development.json` — `env_id: "development"`, `favicon_url`, additional service keys not present in test

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented test-only globals:** No existing doc catalogs `global.fetch`, `global.CSS`, `global.ResizeObserver`, `global.matchMedia`, `global.ReadableStream`, `global.TransformStream`, `global.Worker`, `global.structuredClone`, `global.crypto.subtle`, `global.crypto.randomUUID`, `global.TextEncoder`, `global.TextDecoder` as test-only artifacts
- **Missing network isolation trace:** No doc explains the full `nock.disableNetConnect()` → `beforeAll(nock.activate)` → `afterAll(nock.restore + nock.cleanAll)` lifecycle
- **Missing config divergence proof:** No doc demonstrates with concrete flag values that `config/test.json` and `config/development.json` resolve differently
- **Missing API mock trace:** No doc walks through a complete test showing how a nock interceptor feeds a mocked response into an action creator thunk and how the dispatch spy captures and asserts the resulting Redux actions
- **Missing Jest config explanation:** No doc explains the `moduleNameMapper` redirect from `@automattic/calypso-config` to `client/server/config/index.js` and its implications for test config resolution

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the project rule `SWE-AtlasQnA-Repo`: a single comprehensive Markdown file placed in `blitzy/documentation/`:

```
blitzy/
└── documentation/
    └── wp-calypso_be7e5cc64162.md
```

**Internal document structure:**

```
wp-calypso_be7e5cc64162.md
├── 1. Development Server Verification
│   ├── Boot command and expected output
│   └── How the dev server resolves config (config/development.json)
├── 2. Test Environment Anatomy
│   ├── 2.1 How Jest boots the test environment (jest.config.js chain)
│   ├── 2.2 Shared preset: @automattic/calypso-jest
│   ├── 2.3 Client test setup vs. Server test setup vs. Packages setup
│   └── 2.4 Complete catalog: test-only globals, polyfills, and env vars
├── 3. Network Isolation During Tests
│   ├── 3.1 nock.disableNetConnect() enforcement
│   ├── 3.2 Lifecycle hooks: beforeAll / afterAll
│   ├── 3.3 wpcom-proxy-request mock
│   ├── 3.4 Global fetch mock
│   └── 3.5 What happens when code makes a network request
├── 4. API Mock Trace: Action Creator Test Walkthrough
│   ├── 4.1 Test under analysis: user-suggestions/test/actions.js
│   ├── 4.2 Step 1: nock interceptor setup
│   ├── 4.3 Step 2: Thunk invocation and dispatch spy
│   ├── 4.4 Step 3: Request lifecycle dispatch assertions
│   └── 4.5 Flow diagram: mock → thunk → dispatch → assertion
├── 5. Configuration and Feature Flag Divergence
│   ├── 5.1 How config resolution works (parser.js + index.js)
│   ├── 5.2 moduleNameMapper: how tests see config/test.json
│   ├── 5.3 Side-by-side proof: test vs. development values
│   ├── 5.4 Feature flag divergence table
│   └── 5.5 How tests control config (jest.mock, env vars, ACTIVE_FEATURE_FLAGS)
└── 6. Summary: Test Environment vs. Development at a Glance
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- "Extract test-only globals from `test/client/setup-test-framework.js` lines 25–79 by identifying every `global.*` assignment"
- "Extract config divergence by programmatically comparing `config/test.json` and `config/development.json` feature flags"
- "Generate the API mock trace by reading `client/state/user-suggestions/test/actions.js` line-by-line and annotating each step"
- "Create the network isolation narrative by following the nock initialization chain across `setup-test-framework.js` files"

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for the API mock flow trace
- Code examples using fenced code blocks with syntax highlighting and `Source:` citations
- Tables for comparison data (globals, feature flags, config values)
- Consistent terminology aligned with existing `docs/testing/` conventions

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Sequence diagram:** API mock flow from nock interceptor → thunk dispatch → spy assertion (for Section 4)
- **Flowchart:** Config resolution pipeline showing how `NODE_ENV`/`CALYPSO_ENV` selects `test.json` vs `development.json` (for Section 5)
- **Comparison table diagram:** Test environment vs. development environment side-by-side boot paths (for Section 2)

### 0.4.4 Investigation Approach

The documentation agent will:

- **Start the development server** using `yarn start` (or the most lightweight start command) to confirm it works, capturing output
- **Run a subset of tests** using `yarn test-client -- --watchAll=false --ci --maxWorkers=2` to observe the test environment in action
- **Create temporary investigation scripts** (if needed) in `/tmp/` for any programmatic comparison, and clean them up upon completion
- **Read and annotate source files** to produce the evidence-based content that forms the document body
- **Never modify** any file in the repository source tree

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `test/client/jest.config.js`, `test/client/setup-test-framework.js`, `test/server/jest.config.js`, `test/server/setup-test-framework.js`, `test/packages/setup.js`, `test/packages/jest-preset.js`, `test/module-resolver.js`, `packages/calypso-jest/jest-preset.js`, `client/server/config/parser.js`, `client/server/config/index.js`, `packages/create-calypso-config/src/index.ts`, `config/test.json`, `config/development.json`, `config/_shared.json`, `config/README.md`, `client/state/user-suggestions/test/actions.js`, `client/test-helpers/use-nock/index.js`, `package.json` | Comprehensive Q&A document answering all user questions about testing infrastructure: dev server verification, test environment anatomy, network isolation, API mock tracing, and config/feature flag divergence |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Technical Investigation / Onboarding Q&A
Source Code:
  - test/client/jest.config.js (client Jest configuration)
  - test/client/setup-test-framework.js (client test bootstrap: globals, polyfills, nock)
  - test/server/jest.config.js (server Jest configuration)
  - test/server/setup-test-framework.js (server test bootstrap: nock, wpcom-proxy mock)
  - test/packages/setup.js (package test bootstrap: deterministic UUID, matchMedia)
  - test/packages/jest-preset.js (package Jest preset composition)
  - test/module-resolver.js (custom enhanced-resolve Jest resolver)
  - packages/calypso-jest/jest-preset.js (shared Jest preset: transforms, env, matcher)
  - client/server/config/parser.js (config file layering and feature flag override logic)
  - client/server/config/index.js (config entry point: env selection, createConfig call)
  - packages/create-calypso-config/src/index.ts (config factory: isEnabled, feature flags)
  - config/test.json (test environment config and feature flags)
  - config/development.json (development environment config and feature flags)
  - config/_shared.json (shared default configuration)
  - config/README.md (config system documentation)
  - client/state/user-suggestions/test/actions.js (API mock trace example)
  - client/state/user-suggestions/test/sample-response.json (fixture data)
  - client/test-helpers/use-nock/index.js (nock lifecycle helper)
  - package.json (test commands, dependencies, engines)
Sections:
  - Development server verification
  - Test environment anatomy and comparison
  - Test-only globals, polyfills, and environment variables catalog
  - Network isolation mechanics
  - API mock trace walkthrough (user-suggestions actions)
  - Configuration and feature flag divergence with evidence
  - How tests control config values
Diagrams:
  - Sequence diagram: API mock flow (nock → thunk → dispatch → assertion)
  - Flowchart: Config resolution pipeline (NODE_ENV → parser → createConfig)
Key Citations:
  - test/client/setup-test-framework.js (lines 1-79)
  - test/server/setup-test-framework.js (lines 1-23)
  - test/packages/setup.js (lines 1-16)
  - client/server/config/parser.js (lines 1-88)
  - client/server/config/index.js (lines 1-12)
  - packages/create-calypso-config/src/index.ts (lines 1-140)
  - client/state/user-suggestions/test/actions.js (lines 1-57)
  - config/test.json (env_id, feature flags)
  - config/development.json (env_id, feature flags)
```

### 0.5.3 Documentation Files to Update Detail

No existing documentation files will be modified. The user explicitly states: "Don't modify any repository files." All output is confined to the newly created `blitzy/documentation/wp-calypso_be7e5cc64162.md`.

### 0.5.4 Documentation Configuration Updates

No documentation configuration files require updates. The repository does not use a documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, `.readthedocs.yml`, or `sphinx/conf.py` detected). The output document is a standalone Markdown file.

### 0.5.5 Cross-Documentation Dependencies

- The generated document references existing docs `config/README.md` and `docs/testing/testing-overview.md` for background context but does not modify them
- No navigation links, table of contents updates, or index updates are required since the output is a standalone file in a separate `blitzy/documentation/` directory

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are directly relevant to the testing infrastructure under investigation and will be referenced in the generated documentation:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm | jest | ^29.7.0 | Primary test runner for all unit, component, and integration suites |
| npm | jest-environment-jsdom | ^29.7.0 | Browser-like DOM test environment for client and app tests |
| npm | @testing-library/jest-dom | ^6.6.3 | Extended DOM assertion matchers (`.toBeInTheDocument()`, etc.) |
| npm | nock | ^13.5.6 | HTTP request interception and mocking for network isolation |
| npm | jest-canvas-mock | ^2.5.2 | HTML5 Canvas API mocking for client and app test environments |
| npm | resize-observer-polyfill | ^1.5.1 | ResizeObserver polyfill used in test setup for browser API simulation |
| npm | enhanced-resolve | 5.9.3 | Module resolution engine powering the custom Jest resolver |
| npm | @automattic/calypso-jest | workspace:^ | Shared Jest preset providing resolver, transforms, and setup |
| npm | @automattic/calypso-config | workspace:^ | Client-side config API (`config()`, `config.isEnabled()`) |
| npm | @automattic/create-calypso-config | workspace:^ | Config factory function that produces the ConfigApi object |
| npm | babel-jest | (via @automattic/calypso-build) | Babel transform integration for Jest test transpilation |
| npm | debug | ^4.4.0 | Namespaced debug logging used in `useNock` and config parser |
| npm | lodash | ^4.17.21 | `assignWith` used in config parser for feature-aware merging |
| npm | deep-freeze | (transitive) | Used in test fixtures to prevent accidental mutation of mock data |

### 0.6.2 Runtime and Toolchain Requirements

| Runtime/Tool | Required Version | Source |
|-------------|-----------------|--------|
| Node.js | ^v22.9.0 | `package.json` `engines` field |
| Yarn | ^4.0.0 | `package.json` `engines` field, `.yarnrc.yml` |
| TypeScript | 5.8.2 | `package.json` `devDependencies` |

### 0.6.3 Documentation Reference Updates

No existing documentation reference updates are required, as no links or cross-references need to be modified. The generated document is a standalone artifact in `blitzy/documentation/`.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the user's questions:**

| Question Area | Current Documentation Coverage | Target |
|--------------|-------------------------------|--------|
| Dev server boot verification | Partially covered in `docs/yarn-start.md` and `docs/install.md` | Full boot evidence with output |
| Test environment anatomy | Partially in `docs/testing/testing-overview.md` (suite listing only) | Complete globals/polyfills catalog |
| Network isolation mechanics | Not documented | Full nock lifecycle trace |
| API mock tracing | Not documented | Step-by-step annotated walkthrough |
| Config divergence proof | Partially in `config/README.md` (describes system, not evidence) | Side-by-side value comparison |
| Test-controlled config | Not documented | jest.mock patterns and env var mechanisms |

**Target coverage:** 100% of user questions answered with evidence-based responses citing specific source files and line numbers.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must be answered with a dedicated section
- All claims must cite source files with line references (e.g., `Source: test/client/setup-test-framework.js:9`)
- All test-only globals must be cataloged exhaustively (no partial lists)
- Feature flag divergence must include concrete flag names and boolean values from both config files
- The API mock trace must follow the complete flow from nock setup through final assertion

**Accuracy validation:**
- All code examples must be copied verbatim from source files (not paraphrased)
- All file paths must be verified to exist in the repository
- All config values cited must match the actual JSON content
- The API mock trace must reflect the actual execution order observable in Jest

**Clarity standards:**
- Technical accuracy with accessible language suitable for an onboarding developer
- Progressive disclosure: overview first, then detailed trace
- Consistent terminology aligned with Calypso's existing documentation vocabulary (e.g., "setup-test-framework" not "test bootstrap")
- Each section opens with a concise answer, then provides supporting evidence

**Maintainability:**
- Source citations use format `Source: path/to/file.js:LineNumber` for traceability
- The document identifies itself as based on a specific branch (`wp-calypso_be7e5cc64162`) for freshness tracking

### 0.7.3 Example and Diagram Requirements

- **Minimum code examples:** At least one annotated code excerpt per question area (6 minimum across the document)
- **Diagram types:** 1 sequence diagram (API mock flow), 1 flowchart (config resolution)
- **Tables:** At least 3 comparison tables (globals catalog, feature flag divergence, environment comparison)
- **Code example sourcing:** All examples sourced from actual repository files, not synthesized

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — The sole output document containing all investigation findings

**Source files to analyze and reference (read-only):**
- `test/client/jest.config.js` — Client Jest configuration
- `test/client/setup-test-framework.js` — Client test globals, polyfills, network isolation
- `test/server/jest.config.js` — Server Jest configuration
- `test/server/setup-test-framework.js` — Server test network isolation, wpcom-proxy mock
- `test/packages/jest-preset.js` — Package test preset composition
- `test/packages/setup.js` — Package test globals (UUID, ResizeObserver, matchMedia)
- `test/apps/jest-preset.js` — App test preset (jsdom, canvas mock, shared setup)
- `test/integration/jest.config.js` — Integration test Node.js environment config
- `test/module-resolver.js` — Custom enhanced-resolve Jest module resolver
- `packages/calypso-jest/jest-preset.js` — Shared Jest preset baseline
- `client/server/config/parser.js` — Config file layering and override logic
- `client/server/config/index.js` — Config entry point and environment selection
- `packages/create-calypso-config/src/index.ts` — Config factory (isEnabled, feature flags)
- `config/test.json` — Test environment configuration
- `config/development.json` — Development environment configuration
- `config/_shared.json` — Shared default configuration
- `config/README.md` — Config system documentation
- `client/state/user-suggestions/test/actions.js` — API mock trace example
- `client/state/user-suggestions/test/sample-response.json` — Fixture data for trace
- `client/test-helpers/use-nock/index.js` — Nock lifecycle helper
- `package.json` — Root manifest (test scripts, deps, engines)

**Investigation activities:**
- Starting the development server to confirm it works
- Running a subset of tests to observe test environment behavior
- Programmatic comparison of `config/test.json` vs `config/development.json`
- Creating temporary scripts in `/tmp/` for investigation (cleaned up after use)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly prohibits modifying any repository files
- **Test file modifications** — No existing test files will be changed
- **Feature additions or code refactoring** — This is a documentation-only task
- **E2E test infrastructure** — The user's questions focus on unit/component test environments, not Playwright/E2E
- **Build tool tests** — `test/build-tools/` is not part of the user's investigation focus
- **Deployment configuration changes** — No deployment configs will be touched
- **New test creation** — No new tests will be written; existing tests are analyzed as-is
- **Desktop app testing** — `desktop/` test infrastructure is not in scope
- **Storybook** — Visual component testing in `.storybook/` is not in the user's question set
- **Documentation for unrelated topics** — Only the specific questions about test infrastructure, network isolation, API mock tracing, and config divergence are addressed
- **CI/CD pipeline documentation** — TeamCity, GitHub Actions, and CircleCI configurations are not part of the user's questions

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Development server start command:** `yarn start` (full build + server) or `yarn run build-server && yarn run start-build` for the SSR server portion
- **Test execution commands (read-only investigation):**
  - Client tests: `TZ=UTC yarn test-client -- --watchAll=false --ci --maxWorkers=2`
  - Server tests: `yarn test-server -- --watchAll=false --ci --maxWorkers=2`
  - Packages tests: `yarn test-packages -- --watchAll=false --ci --maxWorkers=2`
  - All core tests: `yarn test` (runs client, packages, server, build-tools sequentially)
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with line numbers
- **Style guide:** Follow the existing Calypso documentation conventions observed in `docs/testing/` (Markdown headers, code blocks with language tags, description-first structure)
- **Documentation validation:** Manual review for completeness against user questions; no automated doc linting tool detected in the repository

### 0.9.2 Investigation Sequence

The documentation agent should follow this investigation sequence to gather evidence:

- **Phase 1:** Attempt to start the development server; capture boot output or failure diagnostics
- **Phase 2:** Run a minimal subset of client tests to observe and confirm the test environment bootstrap
- **Phase 3:** Read and annotate all test setup files to catalog globals, polyfills, and mocks
- **Phase 4:** Select `client/state/user-suggestions/test/actions.js` as the API mock trace example; read the test, the action creator, and the fixture to trace the full flow
- **Phase 5:** Compare `config/test.json` and `config/development.json` programmatically; document concrete divergences
- **Phase 6:** Read `client/server/config/parser.js` and `client/server/config/index.js` to document config resolution mechanics
- **Phase 7:** Identify tests that use `jest.mock('@automattic/calypso-config')` to show how tests override config values
- **Phase 8:** Compile all findings into the output document with diagrams, tables, and annotated code excerpts
- **Phase 9:** Clean up any temporary scripts created during investigation

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** All investigation is read-only. The only file created is `blitzy/documentation/wp-calypso_be7e5cc64162.md`.
- **If temporary scripts are needed for investigation, clean them up when done.** Any helper scripts created in `/tmp/` during the investigation phase must be deleted before the task is marked complete.
- **Base all answers on the code as the truth.** Do not make assumptions. Every claim, value, and path cited in the documentation must be verified against actual source files.
- **Provide thinking and rationale behind the answers.** The document should explain not just "what" but "why" — why test environments differ, why certain globals are mocked, why nock is used.
- **Create the output document as `wp-calypso_be7e5cc64162.md`** in the `blitzy/documentation` directory, as specified by the `SWE-AtlasQnA-Repo` implementation rule.
- **Include source citations for all technical details.** Use the format `Source: path/to/file.js:LineNumber` to trace every claim back to its origin in the codebase.
- **Show concrete proof for configuration divergence.** The user explicitly says "show me proof that a test actually uses a different value than the dev server would resolve." The document must include actual JSON values from both config files.
- **Trace the complete API mock flow.** The user asks to "trace how the mocked response flows through the action creator back to the test assertion." The documentation must provide a step-by-step walkthrough with code excerpts and a flow diagram.
- **Follow existing documentation style and structure.** Align with the conventions observed in `docs/testing/` — Markdown headers, fenced code blocks with language tags, tables for structured data, and description-first narratives.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

**Test Infrastructure Files:**
- `test/README.md` — Testing configuration overview
- `test/client/jest.config.js` — Client Jest configuration (moduleNameMapper, globals, setupFiles)
- `test/client/setup-test-framework.js` — Client test bootstrap (nock, globals, polyfills, mocks)
- `test/server/jest.config.js` — Server Jest configuration (rootDir, moduleNameMapper)
- `test/server/setup-test-framework.js` — Server test bootstrap (nock, wpcom-proxy-request mock)
- `test/packages/jest.config.js` — Package test multi-project coordinator
- `test/packages/jest-preset.js` — Package test preset (calypso-jest base, i18n global, setup hook)
- `test/packages/setup.js` — Package test globals (randomUUID, ResizeObserver, matchMedia)
- `test/apps/jest.config.js` — App test multi-project coordinator
- `test/apps/jest-preset.js` — App test preset (jsdom, canvas mock, shared setup)
- `test/integration/jest.config.js` — Integration test config (Node.js env, network allowed)
- `test/module-resolver.js` — Custom enhanced-resolve Jest resolver (calypso:src priority)
- `packages/calypso-jest/jest-preset.js` — Shared Jest preset (resolver, transforms, test match, snapshot config)

**Configuration System Files:**
- `config/README.md` — Config system documentation (env selection, feature flags, overrides)
- `config/test.json` — Test environment config (env_id: "test", feature flags)
- `config/development.json` — Development environment config (env_id: "development", feature flags)
- `config/_shared.json` — Shared default configuration base
- `config/client.json` — Client-exposed config key manifest
- `config/empty-secrets.json` — Placeholder secrets for non-secret environments
- `client/server/config/parser.js` — Config parser: layered JSON merging, feature overrides, secrets handling
- `client/server/config/index.js` — Config entry point: env selection via CALYPSO_ENV/NODE_ENV, createConfig call
- `packages/create-calypso-config/src/index.ts` — Config factory: ConfigApi, isEnabled, enable, disable, ACTIVE_FEATURE_FLAGS
- `packages/calypso-config/src/index.ts` — Browser-side config bootstrap (window.configData, URL/cookie flag overrides)
- `packages/calypso-config/README.md` — Config package usage guide

**Test Example and Helper Files:**
- `client/state/user-suggestions/test/actions.js` — Representative API mock test (nock → thunk → dispatch)
- `client/state/user-suggestions/test/sample-response.json` — Fixture data for mock response
- `client/test-helpers/use-nock/index.js` — Nock lifecycle helper (deprecated, wraps beforeAll/afterAll)
- `client/test-helpers/use-nock/README.md` — useNock helper documentation

**Existing Documentation Files:**
- `docs/testing/index.md` — Testing guide entry point
- `docs/testing/testing-overview.md` — Test suite organization and run commands
- `docs/testing/unit-tests.md` — Unit test conventions and mocking strategy
- `docs/testing/component-tests.md` — Component testing with @testing-library/react
- `docs/testing/snapshot-testing.md` — Snapshot testing guidelines
- `docs/testing/faq.md` — Testing FAQ (tools, commands, .only() warning)

**Root Configuration Files:**
- `package.json` — Root manifest (test scripts, dependencies, engines, browserslist)
- `babel.config.js` — Shared Babel entry for browser/server/Emotion transforms

**Folders Explored:**
- `test/` — Top-level testing infrastructure hub
- `test/client/` — Client test harness (2 files)
- `test/server/` — Server test harness (2 files)
- `test/packages/` — Package test infrastructure (3 files)
- `test/apps/` — App test infrastructure (2 files)
- `test/integration/` — Integration test config (1 file)
- `config/` — Runtime configuration hub (18 files)
- `docs/` — Central documentation hub
- `docs/testing/` — Testing documentation (6 files)
- `packages/calypso-jest/` — Shared Jest preset package
- `packages/create-calypso-config/` — Config factory package
- `packages/calypso-config/` — Browser-side config package
- `client/server/config/` — Server-side config implementation
- `client/test-helpers/use-nock/` — Nock test helper

### 0.11.2 Technical Specification Sections Retrieved

- **Section 6.6 Testing Strategy** — Comprehensive testing architecture, suite organization, tool versions, CI/CD integration, and test environment documentation

### 0.11.3 Attachments

No attachments were provided for this project. All information was derived from the repository source code and the technical specification document.

