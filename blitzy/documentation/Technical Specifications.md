# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** that comprehensively answers a set of targeted questions about test runtime environments, module resolution behavior, and initialization order across the Calypso monorepo's multiple Jest execution contexts.

- **Category:** Create new documentation
- **Documentation Type:** Technical investigation / Q&A analysis document
- **Output:** A markdown file named `wp-calypso_be7e5cc64162.md` placed in the `blitzy/documentation` directory

The user's requirements translate into the following specific investigation objectives:

- **Test command inventory**: Identify every test command available in the codebase and characterize the runtime environment each produces — including the Jest test environment setting, available global APIs, setup file chain, and module resolution configuration
- **Environment comparison**: Systematically compare what globals and browser-like APIs exist in each test context, identifying capabilities that are present in one context but absent in another
- **Internal package dependency tracing**: Locate a monorepo workspace package that depends on another internal package, run its tests, and determine the actual file loaded when that internal dependency is imported — verifying whether this changes based on execution context
- **Import redirection discovery**: Identify where the test infrastructure overrides import paths (via `moduleNameMapper` or custom resolvers), trace where a redirected import actually resolves at runtime, and determine whether the same import resolves to different file locations depending on context
- **Initialization order verification**: Investigate what loads first when a test runs, identify what provides browser-like APIs (such as `matchMedia`, `ResizeObserver`, `fetch`), determine when those providers become available, and observe what is accessible at module-load time vs. `beforeAll` vs. test execution
- **No permanent file modifications**: All investigation uses temporary probe files cleaned up after observation

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule — SWE-AtlasQnA-Repo**: The user specifies that the output must be a new markdown document named after the source branch (`wp-calypso_be7e5cc64162.md`) placed in `blitzy/documentation`, providing thinking and rationale behind answers, basing all conclusions on the code as truth, and not modifying any existing repository files
- **Temporary probe files are acceptable**: The user explicitly permits creating temporary test files to observe runtime behavior, provided they are cleaned up when done
- **Evidence-based analysis**: All answers must be grounded in actual code inspection and runtime observation, not assumptions

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer all questions about test runtime environments**, we will create a comprehensive markdown document `blitzy/documentation/wp-calypso_be7e5cc64162.md` that presents findings from:
  - Running each of the six test commands (`test-client`, `test-server`, `test-packages`, `test-apps`, `test-build-tools`, `test-integration`) and characterizing their environments
  - Deploying temporary probe tests across five contexts (client, server, packages, apps, build-tools) to capture global API availability, module resolution paths, and initialization timing
  - Tracing the `@automattic/calypso-config` import across all contexts to document where it resolves, what module shape it returns, and where it fails
  - Examining `@automattic/data-stores` as the internal package with workspace dependencies on `@automattic/calypso-config`, `@automattic/calypso-analytics`, `i18n-calypso`, and nine other internal packages
  - Verifying that browser-like APIs (`matchMedia`, `ResizeObserver`, `CSS.supports`, `fetch`, `Worker`) are available from module-load time onward, because `setupFilesAfterEnv` executes before test modules are parsed

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation requirements are surfaced:

- The `calypso:src` custom resolver mechanism (`packages/calypso-jest/src/module-resolver.js`) is a critical piece of infrastructure that makes all workspace packages resolve to untranspiled source — this needs clear explanation
- The `moduleNameMapper` entries in client, server, and integration configs that redirect `@automattic/calypso-config` to `client/server/config/index.js` represent an intentional import override that diverges from the default `calypso:src` resolution — this pattern needs documentation
- The difference between the `calypso-config` browser source (`packages/calypso-config/src/index.ts`, which requires `window`) and the server source (`client/server/config/index.js`, which reads JSON from disk) is the root cause of divergent behavior across test contexts
- The four distinct setup file chains (`test/client/setup-test-framework.js`, `test/server/setup-test-framework.js`, `test/packages/setup.js`, `packages/calypso-jest/src/setup.js`) provide different levels of browser API simulation, which directly impacts which tests can access which globals

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **well-structured documentation directory** (`docs/`) alongside a **sophisticated, multi-layered test infrastructure** (`test/`) that is partially documented but leaves critical cross-context behavior undocumented.

**Documentation framework:** No dedicated documentation generator (e.g., MkDocs, Docusaurus, Sphinx) is configured at the repository root. Documentation is plain Markdown files in the `docs/` directory, organized by topic.

**Existing test documentation found:**
- `test/README.md` — Brief overview listing four test suite groups (client, integration, server, e2e) with pointers to detailed guides
- `docs/testing/testing-overview.md` — Documents test commands and mentions four test modes (server, client, integration, e2e) but omits packages, apps, and build-tools suites
- `docs/testing/unit-tests.md` — Conventions for unit test naming, structure, and mocking strategy
- `docs/testing/component-tests.md` — Component testing patterns with `@testing-library/react`
- `docs/testing/snapshot-testing.md` — Snapshot testing guidelines
- `docs/testing/faq.md` — Testing FAQ
- `docs/testing/index.md` — Testing section entry point

**Diagram tools detected:** Mermaid (used extensively in tech spec sections; supported by Markdown renderers)

**API documentation tools:** None dedicated; JSDoc comments exist in some files but no generator is configured

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to locate code relevant to the investigation:

- **Test configuration files:** `test/*/jest.config.js`, `test/*/jest-preset.js`, `test/*/setup*.js` — all seven suite configurations inspected
- **Shared test preset:** `packages/calypso-jest/jest-preset.js`, `packages/calypso-jest/src/module-resolver.js`, `packages/calypso-jest/src/setup.js`, `packages/calypso-jest/src/asset-transform.js`
- **Module resolver variants:** `test/module-resolver.js` (duplicate of calypso-jest resolver)
- **Package manifests with `calypso:src`:** Inspected `packages/calypso-config/package.json`, `packages/calypso-analytics/package.json`, `packages/data-stores/package.json`, `packages/calypso-products/package.json`, `packages/i18n-calypso/package.json`, `packages/load-script/package.json`, `packages/components/package.json`
- **Config module variants:** `packages/calypso-config/src/index.ts` (browser version), `client/server/config/index.js` (server version), `client/server/config/parser.js` (JSON config file reader), `packages/create-calypso-config/src/index.ts` (config factory)
- **Tests using calypso-config:** `packages/calypso-products/test/plan-lookups.js` (mocks it), `packages/i18n-utils/src/test/utils.js` (mocks it), `packages/data-stores/src/onboard/test/utils.ts` (mocks it with `@jest-environment jsdom`)
- **Babel configuration:** `babel.config.js` (root), confirming `@automattic/calypso-babel-config` usage with browser/server target switching via `BROWSERSLIST_ENV`
- **Root package.json:** All test scripts, workspace definitions, engine constraints (Node ≥22.9.0, Yarn ≥4.0.0)

### 0.2.3 Runtime Investigation Conducted

Temporary probe test files were deployed across five execution contexts (client, server, packages via data-stores, apps via notifications, build-tools) to capture:

- **Global API availability:** `window`, `document`, `matchMedia`, `ResizeObserver`, `CSS.supports`, `fetch`, `Worker`, `TextEncoder`, `ReadableStream`, `structuredClone`, `crypto.randomUUID`, `__i18n_text_domain__`
- **Module resolution paths:** `require.resolve('@automattic/calypso-config')`, `require.resolve('@automattic/calypso-analytics')`, `require.resolve('i18n-calypso')`
- **Module shape analysis:** Exported keys, module type (CJS vs ESM-like), presence of `default` export
- **Initialization timing:** State captured at module-parse time, `beforeAll` hooks, and test execution time to verify when APIs become available

All probe files were created in standard test directories, verified to pass, and then removed — confirmed via `git status` showing a clean working tree.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation spans the following modules and infrastructure files, each of which requires documentation coverage in the output document:

- **Module: `test/client/jest.config.js`**
  - Key configuration: `rootDir: '../../client'`, `testEnvironment: node` (inherited), `moduleNameMapper` redirecting `@automattic/calypso-config` to `client/server/config/index.js`, `setupFiles: ['jest-canvas-mock']`, `setupFilesAfterEnv: ['test/client/setup-test-framework.js']`, `globals: { google: {}, __i18n_text_domain__: 'default' }`, `testEnvironmentOptions: { url: 'https://example.com' }`
  - Documentation needed: Full environment characterization including what each setting provides

- **Module: `test/client/setup-test-framework.js`**
  - Provides: `@testing-library/jest-dom`, `nock.disableNetConnect()`, `TextEncoder`/`TextDecoder`, `CSS.supports` mock, `ResizeObserver` polyfill, `fetch` mock, `wpcom-proxy-request` mock (full: `canAccessWpcomApis`, `reloadProxy`, `requestAllBlogsAccess`), `crypto.randomUUID` (Node implementation), `matchMedia` mock, `ReadableStream`/`TransformStream`, `Worker` (Node `worker_threads`), `structuredClone` fallback, `crypto.subtle`
  - Documentation needed: Complete API inventory table

- **Module: `test/server/jest.config.js`**
  - Key configuration: `rootDir: '../../client/server'`, `testEnvironment: node` (inherited), `moduleNameMapper` redirecting `@automattic/calypso-config` to `calypso/server/config`, `setupFilesAfterEnv: ['test/server/setup-test-framework.js']`
  - Documentation needed: Differences from client config

- **Module: `test/server/setup-test-framework.js`**
  - Provides: `nock.disableNetConnect()`, `wpcom-proxy-request` mock (minimal: `__esModule: true` only)
  - Documentation needed: What is NOT provided compared to client

- **Module: `test/packages/jest.config.js`** and **`test/packages/jest-preset.js`**
  - Key configuration: Multi-project coordinator discovering `packages/*/jest.config.js`, `testEnvironment: node` (inherited), `moduleNameMapper` for `react-markdown` only (NO calypso-config redirect), `setupFilesAfterEnv: ['test/packages/setup.js']`, `globals: { __i18n_text_domain__: 'default' }`
  - Documentation needed: Why calypso-config behaves differently here

- **Module: `test/packages/setup.js`**
  - Provides: `@testing-library/jest-dom`, `crypto.randomUUID` (returns `'fake-uuid'`), `ResizeObserver` polyfill, `matchMedia` mock
  - Documentation needed: Reduced API surface compared to client

- **Module: `test/apps/jest-preset.js`**
  - Key configuration: `testEnvironment: 'jsdom'` (explicitly set), `setupFiles: ['jest-canvas-mock']`, `setupFilesAfterEnv: ['test/client/setup-test-framework.js']` (reuses client setup)
  - Documentation needed: Only context with jsdom, reuses client setup providing all browser APIs

- **Module: `test/build-tools/jest.config.js`**
  - Key configuration: `rootDir: '../../build-tools'`, `testEnvironment: node` (inherited), inherits base calypso-jest preset only
  - `setupFilesAfterEnv: ['packages/calypso-jest/src/setup.js']` (provides `CSS.supports` only)
  - Documentation needed: Most minimal context

- **Module: `test/integration/jest.config.js`**
  - Key configuration: `rootDir: '../..'`, `testEnvironment: 'node'` (explicitly set), `moduleNameMapper` redirecting `@automattic/calypso-config` to `client/server/config/index.js`, custom resolver via `@automattic/calypso-jest/src/module-resolver.js`, `modulePaths: ['<rootDir>/client/extensions']`, NO `setupFilesAfterEnv` override
  - Documentation needed: Network access allowed (no nock), unique resolver reference

- **Module: `packages/calypso-jest/src/module-resolver.js`**
  - The custom Jest resolver using `enhanced-resolve` with `mainFields: ['calypso:src', 'main']` and `conditionNames: ['calypso:src', 'node', 'require']`
  - Documentation needed: How `calypso:src` field enables untranspiled source resolution

- **Module: `packages/calypso-config/src/index.ts`** (browser version)
  - Requires `window` at line 17, reads `window.configData`, exports: `default`, `isEnabled`, `enabledFeatures`, `enable`, `disable`, `isCalypsoLive`
  - Documentation needed: Why it fails in non-jsdom contexts

- **Module: `client/server/config/index.js`** (server version)
  - Uses `parser.js` to read JSON config files from `config/` directory, exports CJS: `isEnabled`, `enabledFeatures`, `enable`, `disable`, `clientData`
  - Documentation needed: Why this is the redirect target for client/server/integration tests

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist:

- **No existing document** compares runtime environments across the six test suites
- **No existing document** traces the `calypso-config` import redirection mechanism and its divergent behavior
- **No existing document** catalogs which browser-like APIs are available in which test context
- **No existing document** explains the `calypso:src` resolver's interaction with `moduleNameMapper` overrides
- The `test/README.md` mentions only four test groups, omitting `test-packages`, `test-apps`, and `test-build-tools`
- `docs/testing/testing-overview.md` similarly mentions only server, client, integration, and e2e — not the full seven suites
- No documentation explains why `@automattic/calypso-config` must be mocked in package tests but resolves automatically in client tests

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the `SWE-AtlasQnA-Repo` rule: a single markdown file answering the user's questions with reasoning grounded in the codebase.

```
blitzy/
└── documentation/
    └── wp-calypso_be7e5cc64162.md
        ├── Overview and Methodology
        ├── Q1: Test Commands and Their Runtime Environments
        │   ├── Command Inventory Table
        │   ├── Per-Context Deep Dive (client, server, packages, apps, build-tools, integration)
        │   └── Environment Comparison Matrix
        ├── Q2: Global API Availability Differences
        │   ├── Browser-Like API Comparison Table
        │   ├── What Provides Each API
        │   └── What Exists in One Context But Not Another
        ├── Q3: Internal Package Dependencies and Import Resolution
        │   ├── @automattic/data-stores as Case Study
        │   ├── calypso:src Resolution Mechanism
        │   └── Does the Loaded File Differ by Execution Context?
        ├── Q4: Import Redirection and Override Tracing
        │   ├── The calypso-config Redirect Pattern
        │   ├── moduleNameMapper vs calypso:src Resolution
        │   └── Same Import, Different Files
        ├── Q5: Initialization Order and Browser API Timing
        │   ├── Jest Lifecycle: setupFiles → Environment → setupFilesAfterEnv → Module Load
        │   ├── Probe Results: Module-Load vs beforeAll vs Test-Time
        │   └── When Browser APIs Become Available
        └── Summary: Root Causes of Context-Dependent Test Behavior
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract Jest configuration details from `test/*/jest.config.js` and `test/*/jest-preset.js` files using direct file reading"
- "Generate environment comparison data by deploying probe tests that capture `typeof` checks for each global API across five contexts"
- "Trace module resolution by calling `require.resolve()` inside probe tests running under each Jest configuration"
- "Verify initialization order by capturing global state at module-parse time, `beforeAll`, and test execution time"
- "Validate calypso-config behavior by importing the module in each context and observing whether it resolves, what file it points to, and what exports it provides"

**Documentation Standards:**

- Markdown tables for comparison matrices (environment features, API availability, module resolution paths)
- Mermaid diagrams for the setup file chain and resolution flow
- Source citations as inline references: `Source: test/client/jest.config.js:11`
- Code snippets limited to 2-3 lines showing key configuration patterns

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams planned for the output document:

- **Jest setup lifecycle flowchart**: Showing `setupFiles` → Test Environment initialization → `setupFilesAfterEnv` → Test module loading → `beforeAll` → Test execution, annotated with what becomes available at each stage
- **Module resolution decision tree**: Showing how `@automattic/calypso-config` import flows through `moduleNameMapper` (if present) vs. `calypso:src` custom resolver
- **Setup file inheritance diagram**: Showing which setup files are loaded by each test context and what APIs each provides

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `test/client/jest.config.js`, `test/client/setup-test-framework.js`, `test/server/jest.config.js`, `test/server/setup-test-framework.js`, `test/packages/jest.config.js`, `test/packages/jest-preset.js`, `test/packages/setup.js`, `test/apps/jest.config.js`, `test/apps/jest-preset.js`, `test/build-tools/jest.config.js`, `test/integration/jest.config.js`, `packages/calypso-jest/jest-preset.js`, `packages/calypso-jest/src/module-resolver.js`, `packages/calypso-jest/src/setup.js`, `packages/calypso-config/package.json`, `packages/calypso-config/src/index.ts`, `client/server/config/index.js`, `client/server/config/parser.js`, `packages/data-stores/package.json`, `packages/data-stores/jest.config.js`, `packages/calypso-products/package.json`, `packages/calypso-products/jest.config.js`, `packages/calypso-products/test/plan-lookups.js`, `packages/i18n-utils/src/test/utils.js`, `packages/data-stores/src/onboard/test/utils.ts`, `package.json`, `babel.config.js` | Complete investigative Q&A document answering all user questions about test runtime environments, module resolution, import redirection, initialization order, and cross-context differences |

No existing files are modified or deleted. The `SWE-AtlasQnA-Repo` rule prohibits modifying existing repository files.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Investigative Q&A / Technical Analysis
Source Code: 27 files across test/, packages/, client/ directories
Sections:
    - Overview and Methodology
    - Test Commands and Runtime Environments (6 commands, 5 probed contexts)
    - Global API Availability Comparison (13 APIs across 5 contexts)
    - Internal Package Dependency: @automattic/data-stores case study
    - Import Redirection: @automattic/calypso-config resolution tracing
    - Initialization Order: setupFiles → environment → setupFilesAfterEnv → modules
    - Root Causes and Summary
Diagrams:
    - Jest lifecycle flowchart (setup file chain)
    - Module resolution decision tree (calypso-config)
    - Setup file inheritance diagram (which context gets what)
Key Citations:
    test/client/jest.config.js, test/client/setup-test-framework.js,
    test/server/jest.config.js, test/server/setup-test-framework.js,
    test/packages/jest.config.js, test/packages/jest-preset.js, test/packages/setup.js,
    test/apps/jest-preset.js, test/build-tools/jest.config.js,
    test/integration/jest.config.js,
    packages/calypso-jest/jest-preset.js, packages/calypso-jest/src/module-resolver.js,
    packages/calypso-jest/src/setup.js,
    packages/calypso-config/src/index.ts, client/server/config/index.js,
    packages/data-stores/package.json, package.json
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration updates are needed — the repository uses plain Markdown documentation without a static site generator. The output file is placed in a new `blitzy/documentation/` directory as specified by the implementation rules.

### 0.5.4 Cross-Documentation Dependencies

- The output document references findings from `docs/testing/testing-overview.md` for context but does not modify it
- The output document references `test/README.md` for the documented test suite inventory but does not modify it
- No navigation, table of contents, or index updates are required since the output is a standalone investigation document in the `blitzy/documentation/` directory

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following key packages are directly relevant to this documentation exercise — all are already installed in the repository:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm | jest | ^29.7.0 | Primary test runner for all six non-e2e test suites |
| npm | jest-environment-jsdom | ^29.7.0 | jsdom environment used by apps suite and opt-in via docblock |
| npm | @automattic/calypso-jest | 1.0.0 (workspace) | Shared Jest preset: custom resolver, base config, asset transforms |
| npm | enhanced-resolve | 5.9.3 | Powers the `calypso:src` custom module resolver in calypso-jest |
| npm | @testing-library/jest-dom | ^6.6.3 | DOM assertion matchers loaded in client, packages, and apps setup |
| npm | nock | ^13.5.6 | HTTP request interception, network isolation in client and server |
| npm | jest-canvas-mock | ^2.5.2 | Canvas API mock loaded via `setupFiles` in client and apps |
| npm | resize-observer-polyfill | ^1.5.1 | ResizeObserver polyfill for client and packages setup |
| npm | @automattic/calypso-config | workspace (calypso:src → src/index.ts) | Config module whose resolution differs across contexts |
| npm | @automattic/create-calypso-config | workspace (calypso:src → src/index.ts) | Config factory used by both browser and server config variants |
| npm | @automattic/data-stores | workspace (calypso:src → src/index.ts) | Case study package with 12 workspace dependencies |
| npm | @automattic/calypso-products | workspace (calypso:src → src/index.ts) | Package that demonstrates jest.mock pattern for calypso-config |
| npm | @babel/core | ^7.26.10 | Babel transpilation underpinning the `babel-jest` transform |
| npm | babel-jest | ^29.7.0 | Jest transform using Babel with `rootMode: 'upward'` |
| npm | typescript | 5.8.2 | TypeScript compiler for `.ts`/`.tsx` file handling |

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are applicable since this exercise creates a standalone document without modifying existing files. The `SWE-AtlasQnA-Repo` rule explicitly prohibits modifying existing repository files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The investigation must produce documented answers for the following coverage areas:

- **Test commands characterized:** 7/7 (100%) — `test`, `test-client`, `test-server`, `test-packages`, `test-apps`, `test-build-tools`, `test-integration`
- **Execution contexts probed at runtime:** 5/5 (100%) — client, server, packages (data-stores), apps (notifications), build-tools
- **Browser-like APIs inventoried:** 13/13 (100%) — `window`, `document`, `matchMedia`, `ResizeObserver`, `CSS.supports`, `fetch`, `Worker`, `TextEncoder`, `ReadableStream`, `structuredClone`, `crypto.randomUUID`, `__i18n_text_domain__`, `crypto.subtle`
- **Module resolution paths traced:** 3/3 (100%) — `@automattic/calypso-config`, `@automattic/calypso-analytics`, `i18n-calypso`
- **Import redirection mechanisms documented:** 2/2 (100%) — `moduleNameMapper` overrides, `calypso:src` custom resolver
- **Initialization lifecycle stages verified:** 3/3 (100%) — module-parse time, `beforeAll`, test execution
- **Internal package dependency case study:** 1/1 (100%) — `@automattic/data-stores` with its workspace dependencies

### 0.7.2 Documentation Quality Criteria

- **Completeness:** Every question posed by the user must receive a direct, specific answer with supporting evidence from the codebase
- **Accuracy:** All module resolution paths, API availability claims, and behavioral differences must be verified by actual runtime probe results, not inferred from configuration alone
- **Traceability:** Every claim must cite the specific source file(s) that support it (e.g., `Source: test/client/jest.config.js:11`)
- **Clarity:** Comparison tables must make it immediately obvious which API exists in which context; diagrams must make the resolution flow self-explanatory
- **No assumptions:** Per the implementation rule, the document must not make assumptions — all answers must be grounded in the code as truth

### 0.7.3 Example and Diagram Requirements

- **Minimum comparison tables:** 3 (environment features, API availability, module resolution paths)
- **Minimum Mermaid diagrams:** 2 (Jest lifecycle flow, module resolution decision tree)
- **Code example style:** Configuration excerpts kept to 2-3 lines showing the specific setting under discussion
- **Verification method:** All claims about runtime behavior were verified via temporary probe tests that were run, observed, and then cleaned up

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/wp-calypso_be7e5cc64162.md` — The sole output artifact

- **Investigation targets (read-only analysis):**
  - `test/client/jest.config.js` — Client Jest configuration
  - `test/client/setup-test-framework.js` — Client browser API setup
  - `test/server/jest.config.js` — Server Jest configuration
  - `test/server/setup-test-framework.js` — Server network isolation setup
  - `test/packages/jest.config.js` — Packages multi-project coordinator
  - `test/packages/jest-preset.js` — Packages shared Jest preset
  - `test/packages/setup.js` — Packages browser API setup (reduced)
  - `test/apps/jest.config.js` — Apps multi-project coordinator
  - `test/apps/jest-preset.js` — Apps shared Jest preset (jsdom)
  - `test/build-tools/jest.config.js` — Build-tools Jest configuration
  - `test/integration/jest.config.js` — Integration Jest configuration
  - `packages/calypso-jest/jest-preset.js` — Shared base Jest preset
  - `packages/calypso-jest/src/module-resolver.js` — Custom `calypso:src` resolver
  - `packages/calypso-jest/src/setup.js` — Base setup (CSS.supports only)
  - `packages/calypso-jest/src/asset-transform.js` — Asset file transform
  - `packages/calypso-config/package.json` — Package manifest with `calypso:src` field
  - `packages/calypso-config/src/index.ts` — Browser-side config module
  - `packages/calypso-config/src/desktop.ts` — Desktop override config
  - `client/server/config/index.js` — Server-side config module (redirect target)
  - `client/server/config/parser.js` — JSON config file parser
  - `packages/create-calypso-config/src/index.ts` — Config factory
  - `packages/data-stores/package.json` — Case study package manifest
  - `packages/data-stores/jest.config.js` — Case study Jest config
  - `packages/data-stores/src/onboard/test/utils.ts` — Example test using calypso-config mock
  - `packages/calypso-products/package.json` — Package with calypso-config dependency
  - `packages/calypso-products/jest.config.js` — Package Jest config
  - `packages/calypso-products/test/plan-lookups.js` — Test demonstrating jest.mock pattern
  - `packages/i18n-utils/src/test/utils.js` — Test demonstrating calypso-config mock
  - `package.json` — Root monorepo manifest (test scripts, engines, workspaces)
  - `babel.config.js` — Root Babel configuration
  - `test/README.md` — Testing infrastructure README
  - `docs/testing/testing-overview.md` — Testing overview documentation

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No existing repository files will be modified per the `SWE-AtlasQnA-Repo` rule
- **Test file modifications** — No existing test files will be changed
- **Feature additions or refactoring** — This is a documentation-only exercise
- **E2E test infrastructure** — The `test/e2e/` directory uses Playwright and is architecturally separate from the Jest-based suites; the user's questions focus on unit/package test contexts
- **Deployment configuration changes** — Not applicable
- **Documentation outside `blitzy/documentation/`** — The existing `docs/testing/` files will not be updated
- **Build output analysis** — The investigation focuses on test-time behavior, not production webpack builds
- **CI/CD pipeline configuration** — TeamCity, GitHub Actions, and CircleCI configs are out of scope for this investigation

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone Markdown file, no static site generator
- **Documentation preview command:** Any Markdown renderer (e.g., GitHub, VS Code Markdown Preview)
- **Diagram generation:** Mermaid diagrams embedded directly in Markdown using fenced code blocks with the `mermaid` language tag; rendered natively by GitHub and most Markdown viewers
- **Default format:** Markdown with Mermaid diagrams, comparison tables, and inline source citations
- **Citation requirement:** Every behavioral claim must reference the specific source file and line where the configuration or code is defined
- **Style guide:** Follow existing `docs/testing/` Markdown style — headers, tables, code fences, concise prose
- **Documentation validation:** Manual review of Markdown rendering; verify all file path citations point to real files in the repository

### 0.9.2 Environment Verified During Investigation

| Aspect | Value |
|--------|-------|
| Node.js version used | v22.9.0 (matching `package.json` engines: `^v22.9.0`) |
| Yarn version | 4.0.2 (matching `.yarnrc.yml` yarnPath) |
| Jest version | ^29.7.0 |
| Package installation | `yarn install --mode=skip-build` (3,176 packages) |
| Packages built | No (untranspiled source used via `calypso:src`) |
| Test execution method | `npx jest -c [config] --testPathPattern=[pattern] --verbose --no-cache` |
| Temporary files | Created, verified, and removed — `git status` confirms clean tree |

## 0.10 Rules for Documentation

The following rules apply to this documentation exercise, derived from user-specified implementation rules and constraints:

- **SWE-AtlasQnA-Repo**: Create a new markdown document named `wp-calypso_be7e5cc64162.md` that comprehensively answers the questions posed in the prompt. Provide thinking and rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.
- **No permanent file modifications**: All investigation must use temporary probe files that are cleaned up after observation. The final state of the repository must have no modified existing files.
- **Evidence-based answers only**: Every behavioral claim (e.g., "matchMedia is available in client tests but not server tests") must be supported by either direct code inspection or actual runtime probe results.
- **Comprehensive coverage**: All questions in the user prompt must be answered — no question may be left unaddressed or deferred.
- **Source citations required**: All technical details must cite the originating source file for traceability.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were directly read and analyzed to derive conclusions for this Agent Action Plan:

**Test Infrastructure (Configuration and Setup):**
- `test/README.md` — Testing infrastructure overview (lists 4 of 7 suites)
- `test/module-resolver.js` — Root-level copy of the custom Jest resolver
- `test/client/jest.config.js` — Client Jest config (rootDir, moduleNameMapper, setupFiles, setupFilesAfterEnv, globals, testEnvironmentOptions)
- `test/client/setup-test-framework.js` — Client setup: nock, browser API mocks/polyfills (TextEncoder, CSS, ResizeObserver, fetch, matchMedia, ReadableStream, Worker, structuredClone, crypto)
- `test/server/jest.config.js` — Server Jest config (rootDir, moduleNameMapper, transformIgnorePatterns)
- `test/server/setup-test-framework.js` — Server setup: nock, wpcom-proxy-request mock (minimal)
- `test/packages/jest.config.js` — Packages multi-project coordinator (projects glob, react-markdown mapper)
- `test/packages/jest-preset.js` — Packages preset (calypso-jest base, cacheDirectory, __i18n_text_domain__, setupFilesAfterEnv)
- `test/packages/setup.js` — Packages setup: @testing-library/jest-dom, crypto.randomUUID ('fake-uuid'), ResizeObserver, matchMedia
- `test/apps/jest.config.js` — Apps multi-project coordinator (projects glob)
- `test/apps/jest-preset.js` — Apps preset (jsdom environment, jest-canvas-mock, client setup-test-framework reuse)
- `test/build-tools/jest.config.js` — Build-tools config (calypso-jest base, rootDir)
- `test/integration/jest.config.js` — Integration config (node environment, moduleNameMapper, modulePaths, custom resolver)

**Shared Test Preset Package:**
- `packages/calypso-jest/package.json` — Package manifest (dependencies: enhanced-resolve, babel-jest, jest-config)
- `packages/calypso-jest/jest-preset.js` — Base preset (node environment, calypso:src resolver, testMatch, transform, snapshotFormat)
- `packages/calypso-jest/src/module-resolver.js` — Custom resolver using enhanced-resolve with mainFields: ['calypso:src', 'main']
- `packages/calypso-jest/src/setup.js` — Base setup providing only CSS.supports mock
- `packages/calypso-jest/src/asset-transform.js` — Asset file transformer (returns basename)

**Configuration Module Variants:**
- `packages/calypso-config/package.json` — Package manifest (calypso:src: src/index.ts, main: dist/cjs/index.js)
- `packages/calypso-config/src/index.ts` — Browser-side config: requires window, reads window.configData, exports default/isEnabled/enable/disable/enabledFeatures/isCalypsoLive
- `packages/calypso-config/src/desktop.ts` — Desktop override config
- `client/server/config/index.js` — Server-side config: reads JSON files via parser.js, exports CJS isEnabled/enabledFeatures/enable/disable/clientData
- `client/server/config/parser.js` — Config JSON parser reading _shared.json, {env}.json, {env}.local.json
- `packages/create-calypso-config/src/index.ts` — Config factory function

**Case Study Packages:**
- `packages/data-stores/package.json` — 12 workspace dependencies including calypso-config, calypso-analytics, i18n-calypso
- `packages/data-stores/jest.config.js` — Uses packages jest-preset, no moduleNameMapper for calypso-config
- `packages/data-stores/src/onboard/test/utils.ts` — Example: @jest-environment jsdom docblock + jest.mock calypso-config
- `packages/calypso-products/package.json` — 7 workspace dependencies including calypso-config
- `packages/calypso-products/jest.config.js` — Uses packages jest-preset, jsdom environment, configData global
- `packages/calypso-products/test/plan-lookups.js` — Example: jest.mock calypso-config with mock implementation
- `packages/i18n-utils/src/test/utils.js` — Example: jest.mock calypso-config with key-value stub
- `packages/calypso-analytics/package.json` — Workspace dependency on load-script
- `packages/i18n-calypso/package.json` — Workspace dependency on interpolate-components
- `packages/load-script/package.json` — calypso:src: src/index.js
- `packages/components/package.json` — 11 workspace dependencies, calypso:src: src/index.ts
- `client/package.json` — Workspace package named "calypso" (main: server/index.js)

**Root Configuration:**
- `package.json` — Monorepo manifest: test scripts, workspace definitions, engines, dependencies, browserslist
- `babel.config.js` — Root Babel config using @automattic/calypso-babel-config
- `.yarnrc.yml` — Yarn 4 config (nodeLinker: node-modules, yarnPath)

**Existing Documentation:**
- `docs/testing/testing-overview.md` — Testing overview (4 suites documented)
- `docs/testing/unit-tests.md` — Unit test conventions (referenced for context)

**Folders explored:**
- `test/` — Top-level test infrastructure root (7 suite subdirectories)
- `test/client/`, `test/server/`, `test/packages/`, `test/apps/`, `test/build-tools/`, `test/integration/` — Each suite's configuration directory
- `packages/calypso-jest/` — Shared Jest preset package
- `packages/calypso-config/` — Config module with browser and server variants
- `packages/data-stores/` — Case study for internal package dependencies
- `packages/calypso-products/` — Example of jest.mock pattern for calypso-config
- `apps/notifications/` — App used for apps-context probe testing
- `client/server/config/` — Server-side config directory
- `config/` — JSON configuration files directory (referenced by parser.js)

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 External References

No external URLs, Figma screens, or third-party documentation were required for this investigation. All findings are derived exclusively from the repository codebase and runtime observation.

