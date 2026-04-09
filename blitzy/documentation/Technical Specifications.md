# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively investigates and documents the test execution performance characteristics of the wp-calypso monorepo's data-layer module, including Jest transformation caching, HTTP mock infrastructure, and warm-vs-cold run timing analysis.

**Request Category:** Create new documentation
**Documentation Type:** Technical investigation / Q&A analysis document

The user's requirements decompose into four distinct investigation areas, each requiring empirical analysis and source-code tracing:

- **Timing Analysis:** Run any test file from `client/state/data-layer/` twice consecutively, measuring wall-clock time to quantify the first-run vs. second-run performance gap and calculate the ratio of cold-to-warm execution time.
- **Cache Infrastructure Investigation:** Locate Jest's transformation cache configuration in the repository, identify the cache directory path, determine the controlling configuration option (`cacheDirectory`), and examine the actual cached file types after a test run.
- **HTTP Mock Tracing:** Identify the HTTP mocking library used in data-layer tests (`nock`), trace its configuration through the test helper chain (`test/client/setup-test-framework.js` → `client/test-helpers/use-nock/index.js`), and analyze whether the mock library contributes to first-run overhead.
- **No-Cache Comparison:** Run the same test file with `--no-cache` flag, compare timing against the cached run, and identify which specific transformation step (Babel/JSX/TypeScript transpilation) consumes the most time during uncached execution.

### 0.1.2 Special Instructions and Constraints

**Critical Constraint: Read-Only Investigation**
The user explicitly stated: *"Don't modify any repository files."* All analysis must be observational — no source code changes, no configuration modifications, no test file edits.

**Implementation Rule: SWE-AtlasQnA-Repo**
The user-specified implementation rule requires:
- Create a new markdown document named `<source_branch_name>.md` — in this case `wp-calypso_be7e5cc64162.md`
- Provide thinking/rationale behind all answers
- Base all answers on the code as truth; do not make assumptions
- Do not modify any existing files in the source repository
- Place the generated document in the `blitzy/documentation` directory

**Style Preferences:**
- Answers must include rationale and evidence-based reasoning
- All claims must reference specific file paths and configurations found in the codebase
- Empirical data (timing measurements) must be included as evidence

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the timing analysis**, we will **create** a new markdown file (`blitzy/documentation/wp-calypso_be7e5cc64162.md`) containing empirical wall-clock measurements from consecutive Jest runs against `client/state/data-layer/test/wpcom-api-middleware.js`, along with ratio calculations.
- To **document the cache infrastructure**, we will **trace** the `cacheDirectory` configuration from `test/client/jest.config.js` (line 7: `path.join(__dirname, '../../.cache/jest')`) through the `@automattic/calypso-jest` preset (`packages/calypso-jest/jest-preset.js`), and catalog file types found in `.cache/jest/`.
- To **document the HTTP mock setup**, we will **trace** the import chain from `nock` ^13.5.6 usage in `test/client/setup-test-framework.js` (global `nock.disableNetConnect()`) through `client/test-helpers/use-nock/index.js` (the `useNock` helper wrapper) to the actual test usage in `client/state/data-layer/wpcom-http/test/index.js`.
- To **document the no-cache comparison**, we will **create** timing comparison tables showing wall-clock time with cache enabled vs. `--no-cache` flag, identifying the `babel-jest` transformation of `.js`/`.ts`/`.jsx`/`.tsx` files as the dominant overhead source.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs were identified:

- **Babel pipeline complexity:** The test-time Babel transformation chain involves 10+ plugins/presets (`@babel/preset-env`, `@babel/preset-react`, `@babel/preset-typescript`, `@emotion/babel-plugin`, `@babel/plugin-transform-runtime`, `@babel/plugin-proposal-class-properties`, `babel-plugin-dynamic-import-node`, `@automattic/babel-plugin-transform-wpcalypso-async`, `@automattic/babel-plugin-preserve-i18n`). This complex pipeline is the root cause of the first-run overhead and must be clearly documented.
- **Haste map construction:** The `.cache/jest/haste-map-*` file (2.5 MB) represents Jest's file system crawler output; its construction on first run is a secondary contributor to cold-start overhead.
- **Cache invalidation semantics:** The first line of each cached transform file is a content hash used for invalidation — this mechanism should be documented to explain when cache misses occur.
- **Mock initialization overhead:** While `nock` itself is lightweight (~212 KB), the global setup in `test/client/setup-test-framework.js` performs 12+ global polyfill/mock installations (`TextEncoder`, `CSS.supports`, `ResizeObserver`, `fetch`, `matchMedia`, `ReadableStream`, `crypto`, etc.) that contribute to per-suite startup cost — but this is a fixed cost not affected by the cache.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature documentation structure with strong coverage of contributor workflows and test conventions, but no existing documentation addressing Jest cache performance or transformation overhead.

**Documentation Framework:** No dedicated documentation site generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` detected). All documentation is authored as standalone Markdown files within the `docs/` hierarchy and module-level `README.md` files.

**Existing Test Documentation:**

| Path | Coverage | Relevance |
|------|----------|-----------|
| `docs/testing/testing-overview.md` | Test suite structure, execution commands, CI integration | High — provides context for test infrastructure |
| `docs/testing/unit-tests.md` | Unit test conventions, mocking strategy, colocated structure | High — describes the conventions data-layer tests follow |
| `docs/testing/component-tests.md` | React component testing with @testing-library | Medium — relevant for test environment context |
| `docs/testing/faq.md` | Toolchain summary, test commands, deprecated tools | Medium — confirms Jest as standard runner |
| `docs/testing/snapshot-testing.md` | Snapshot test guidelines | Low |
| `docs/testing/index.md` | Testing pyramid philosophy | Low |
| `test/README.md` | Test configuration overview, suite descriptions, test helper deprecation | High — documents the seven test suite configs |
| `client/state/data-layer/README.md` | Data layer architecture, middleware design, file structure | High — essential context for data-layer module |

**Key Finding:** No existing documentation covers Jest caching behavior, transformation pipeline performance, or warm-vs-cold execution analysis. This confirms the need for a new investigative document.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code analysis:

- **Test configuration files:** `test/*/jest.config.js` — Found 7 suite-specific configurations
- **Jest preset:** `packages/calypso-jest/jest-preset.js` — Shared base configuration for all suites
- **Babel configuration:** `babel.config.js`, `packages/calypso-babel-config/config.js`, `packages/calypso-babel-config/presets/default.js` — Full transformation pipeline
- **Cache identifier utility:** `build-tools/babel/babel-loader-cache-identifier/index.js` — Webpack-level cache busting (not Jest, but informative)
- **Test setup framework:** `test/client/setup-test-framework.js` — Nock, browser polyfills, mock globals
- **HTTP mock helper:** `client/test-helpers/use-nock/index.js` — Nock wrapper with lifecycle cleanup
- **Data-layer test files:** `client/state/data-layer/test/*.js`, `client/state/data-layer/wpcom-http/test/*.js` — Test subjects

**Key Directories Examined:**

| Directory | Purpose | File Count |
|-----------|---------|------------|
| `client/state/data-layer/` | Data-layer module (middleware, handlers, utils) | 4 core files + subdirectories |
| `client/state/data-layer/test/` | Top-level data-layer tests | 4 test files |
| `client/state/data-layer/wpcom-http/test/` | HTTP subsystem tests (nock-dependent) | 3 test files |
| `client/state/data-layer/wpcom/` | WordPress.com API handler tree | 94 total test files |
| `test/client/` | Client test harness (Jest config + setup) | 2 files |
| `packages/calypso-jest/` | Shared Jest preset package | 5 files |
| `packages/calypso-babel-config/` | Babel transform configuration | 5 files |
| `.cache/jest/` | Runtime cache directory (generated) | 238 files after warm run |

### 0.2.3 Web Search Research Conducted

No external web search was required for this investigation. All answers are derived directly from the codebase and empirical test execution, per the user's instruction to *"base your answers on the code as the truth."* The Jest `cacheDirectory` and `--no-cache` flag behaviors are well-documented in Jest's official documentation and align with the observed behavior in this repository.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation (as source evidence, not as targets for modification):**

- **Module: `client/state/data-layer/`**
  - Core files: `wpcom-api-middleware.js`, `handler-registry.js`, `utils.js`, `convert-snake-case-to-camel-case.ts`
  - Test files in scope: `test/wpcom-api-middleware.js`, `test/handler-registry.js`, `test/utils.js`, `test/convert-snake-case-to-camel-case.ts`
  - Current documentation: `README.md` exists covering architecture and middleware design
  - Documentation needed: Performance investigation document (timing, cache, mocking analysis)

- **Module: `client/state/data-layer/wpcom-http/`**
  - Test files in scope: `test/index.js` (nock-dependent), `test/actions.js`, `test/utils.js`
  - Pipeline tests: `pipeline/test/test.js`, `pipeline/remove-duplicate-gets/test/index.js`, `pipeline/retry-on-failure/test/index.js`
  - Current documentation: None specific to HTTP mocking
  - Documentation needed: Mock library tracing and timing impact analysis

- **Module: `test/client/` (Test Infrastructure)**
  - Files: `jest.config.js` (cache configuration source), `setup-test-framework.js` (nock + polyfill setup)
  - Current documentation: Partially covered in `test/README.md`
  - Documentation needed: Cache directory config explanation, setup overhead analysis

- **Module: `packages/calypso-jest/`**
  - Files: `jest-preset.js` (transform rules), `src/module-resolver.js`, `src/asset-transform.js`
  - Current documentation: `README.md` exists but minimal
  - Documentation needed: Transform pipeline documentation showing babel-jest as primary transform

- **Module: `packages/calypso-babel-config/`**
  - Files: `config.js`, `presets/default.js`
  - Current documentation: `README.md` exists
  - Documentation needed: Documentation of the test-env Babel preset chain that drives transform overhead

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist:

- **No existing performance analysis documentation:** The `docs/testing/` hierarchy covers conventions and commands but not performance characteristics. No document explains why first-run tests are slower.
- **No cache infrastructure documentation:** The `cacheDirectory` setting at `test/client/jest.config.js:7` is undocumented — its purpose, location (`.cache/jest/`), and contents are not described anywhere.
- **No transformation pipeline documentation for tests:** While `packages/calypso-babel-config/` has a `README.md`, there is no documentation explaining the full `babel-jest` transform chain that fires for every `.js`/`.ts`/`.jsx`/`.tsx` file during test execution.
- **No nock architecture documentation:** The `client/test-helpers/use-nock/README.md` covers basic usage but does not address performance implications, initialization cost, or how `nock.disableNetConnect()` in `test/client/setup-test-framework.js` interacts with the global test lifecycle.
- **No warm-vs-cold execution analysis:** No existing document compares cached and uncached test runs or explains the ~1.8x performance ratio observed between first and second executions.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the implementation rule `SWE-AtlasQnA-Repo`, requiring a single Markdown file placed in the `blitzy/documentation` directory. The file structure will mirror the four investigation areas from the user's prompt:

```
blitzy/
└── documentation/
    └── wp-calypso_be7e5cc64162.md
        ├── Overview (investigation context)
        ├── Q1: Test Execution Timing Analysis
        │   ├── Methodology
        │   ├── Measurements (cold/warm/multi-file)
        │   ├── Ratio calculations
        │   └── Rationale
        ├── Q2: Jest Transformation Cache Infrastructure
        │   ├── Cache configuration location
        │   ├── cacheDirectory option explained
        │   ├── Cache directory contents analysis
        │   └── Cache file types catalog
        ├── Q3: HTTP Mock Infrastructure (nock)
        │   ├── Library identification
        │   ├── Configuration trace through test helpers
        │   ├── Mock setup lifecycle
        │   └── Impact on first-run timing
        ├── Q4: --no-cache Performance Comparison
        │   ├── Timing comparison table
        │   ├── Performance impact quantification
        │   └── Dominant transformation step analysis
        └── References (file paths examined)
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract cache configuration from `test/client/jest.config.js:7` via direct file reading
- Extract transform rules from `packages/calypso-jest/jest-preset.js:13-16` showing `babel-jest` and `asset-transform.js`
- Extract nock setup chain from `test/client/setup-test-framework.js:6-22` and `client/test-helpers/use-nock/index.js:1-22`
- Generate timing data by executing `jest --config test/client/jest.config.js` against data-layer test files with and without cache

**Documentation Standards:**
- Markdown formatting with `#`/`##`/`###` heading hierarchy
- Code examples using fenced blocks with `js` and `bash` syntax highlighting
- Tables for timing comparisons and cache content catalogs
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`
- No Mermaid diagrams required (the investigation is empirical rather than architectural)

### 0.4.3 Diagram and Visual Strategy

No architecture diagrams are required for this investigation document. The content is primarily:
- Tabular data (timing measurements, cache file catalogs)
- Code excerpts (configuration snippets, file paths)
- Narrative explanations (rationale for observed behavior)


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `test/client/jest.config.js`, `test/client/setup-test-framework.js`, `packages/calypso-jest/jest-preset.js`, `packages/calypso-babel-config/config.js`, `packages/calypso-babel-config/presets/default.js`, `client/test-helpers/use-nock/index.js`, `client/state/data-layer/test/wpcom-api-middleware.js`, `client/state/data-layer/wpcom-http/test/index.js`, `babel.config.js` | Complete Q&A investigation document answering all four questions about test execution timing, cache configuration, HTTP mocking, and no-cache performance comparison |

No existing documentation files require UPDATE or DELETE operations. This investigation creates exactly one new file.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Technical Investigation / Q&A Analysis
Source Code Files:
    - test/client/jest.config.js (cache configuration, line 7)
    - test/client/setup-test-framework.js (nock setup, polyfills)
    - packages/calypso-jest/jest-preset.js (transform rules, resolver)
    - packages/calypso-babel-config/config.js (babel config factory)
    - packages/calypso-babel-config/presets/default.js (babel presets/plugins)
    - babel.config.js (root babel entrypoint)
    - client/test-helpers/use-nock/index.js (nock lifecycle wrapper)
    - client/state/data-layer/test/wpcom-api-middleware.js (primary test subject)
    - client/state/data-layer/wpcom-http/test/index.js (nock-dependent test)
    - client/state/data-layer/README.md (data-layer architecture context)
Sections:
    - Overview (investigation scope and methodology)
    - Q1: First-run vs. second-run timing with ratio calculation
    - Q2: Jest cacheDirectory configuration and cache contents analysis
    - Q3: Nock HTTP mock library trace and timing impact assessment
    - Q4: --no-cache performance comparison and dominant transform identification
    - References (all file paths examined)
Diagrams: None required
Key Citations: test/client/jest.config.js:7, packages/calypso-jest/jest-preset.js:13-16, test/client/setup-test-framework.js:6-22
```

### 0.5.3 Empirical Data to Include

The document must include the following measured timing data:

**Single-file test (`client/state/data-layer/test/wpcom-api-middleware.js`):**

| Run Type | Jest Reported Time | Wall-Clock Time | Ratio vs. Warm |
|----------|--------------------|-----------------|----------------|
| Cold (empty cache) | 2.47 s | 3.89 s | 1.76x |
| Warm (cached) | 1.34 s | 2.21 s | 1.00x (baseline) |
| No-cache | 2.39 s | 3.74 s | 1.69x |

**Multi-file tests (`client/state/data-layer/wpcom-http/` — 6 test files):**

| Run Type | Jest Reported Time | Wall-Clock Time | Ratio vs. Warm |
|----------|--------------------|-----------------|----------------|
| Cold (empty cache) | 3.30 s | 4.98 s | 1.57x |
| Warm (cached) | 2.22 s | 3.17 s | 1.00x (baseline) |
| No-cache | 3.32 s | 4.99 s | 1.57x |

### 0.5.4 Documentation Configuration Updates

No documentation configuration files exist or require updates. The repository does not use a documentation site generator — all docs are standalone Markdown files.

### 0.5.5 Cross-Documentation Dependencies

- The new document will reference `client/state/data-layer/README.md` for architecture context
- The new document will reference `test/README.md` for test suite configuration overview
- No navigation or table-of-contents updates are required since this is a standalone investigation document in `blitzy/documentation/`


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise. These are all existing dependencies in the repository — no new packages need to be added.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm | jest | ^29.7.0 | Primary test runner; provides `--no-cache`, `--showConfig`, `cacheDirectory` configuration |
| npm | babel-jest | ^29.7.0 | Babel-based Jest transformer for `.js`/`.ts`/`.jsx`/`.tsx` files |
| npm | @babel/core | ^7.26.10 | Core Babel compiler driving all source transforms during test execution |
| npm | @babel/preset-env | ^7.26.9 | Environment-targeted transpilation (targets `node: current` in test mode) |
| npm | @babel/preset-react | ^7.26.3 | JSX transformation with automatic runtime (`@emotion/react` import source) |
| npm | @babel/preset-typescript | ^7.26.0 | TypeScript transpilation to JavaScript |
| npm | @emotion/babel-plugin | ^11.11.0 | Emotion CSS-in-JS compile-time transforms |
| npm | @babel/plugin-transform-runtime | ^7.26.10 | Shared helper deduplication |
| npm | babel-plugin-dynamic-import-node | ^2.3.3 | Converts dynamic `import()` to `require()` in test environment |
| npm | @automattic/babel-plugin-transform-wpcalypso-async | workspace:^ | Calypso-specific async transform plugin |
| npm | @automattic/babel-plugin-preserve-i18n | workspace:^ | Internationalization string preservation |
| npm | @babel/plugin-proposal-class-properties | ^7.18.6 | Class properties syntax support |
| npm | nock | ^13.5.6 | HTTP request interception and mocking library |
| npm | @testing-library/jest-dom | ^6.6.3 | Extended DOM assertion matchers (loaded in setup) |
| npm | jest-canvas-mock | ^2.5.2 | HTML5 Canvas API mocking (loaded in setupFiles) |
| npm | resize-observer-polyfill | (transitive) | ResizeObserver polyfill for jsdom environment |
| npm | enhanced-resolve | ^5.8.3 | Custom Jest module resolver for `calypso:src` field |
| npm | @automattic/calypso-jest | workspace:^ | Shared Jest preset package for the monorepo |
| npm | @automattic/calypso-babel-config | workspace:^ | Shared Babel configuration factory |

### 0.6.2 Documentation Reference Updates

Not applicable. This investigation creates a standalone document in `blitzy/documentation/` with no internal documentation links requiring updates.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of topics addressed by this investigation:**

| Topic | Currently Documented | Target | Status |
|-------|---------------------|--------|--------|
| Jest cache configuration (`cacheDirectory`) | 0% — not documented anywhere | 100% — full explanation with file path, option name, and cache contents | Gap |
| Warm-vs-cold test execution timing | 0% — no performance analysis docs exist | 100% — empirical measurements with ratios | Gap |
| Babel transformation pipeline for tests | 0% — `env.test` config undocumented | 100% — full plugin/preset chain documented | Gap |
| Nock HTTP mock setup chain | ~30% — `client/test-helpers/use-nock/README.md` covers basic usage only | 100% — full trace from setup-test-framework through use-nock to test files | Gap |
| `--no-cache` performance impact | 0% — not documented | 100% — timing comparison with analysis | Gap |
| Cache file type catalog | 0% — not documented | 100% — full inventory of haste-map, transform cache, perf-cache | Gap |

**Target coverage:** 100% of all four investigation questions answered with empirical evidence and source-code citations.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question from the user's prompt must receive a direct, specific answer
- All timing measurements must include both Jest-reported time and wall-clock time
- All configuration references must include exact file paths and line numbers
- The cache directory structure must be fully cataloged with file counts and sizes

**Accuracy validation:**
- All timing data is from actual Jest executions performed during this investigation on Node.js v22.9.0
- All configuration excerpts are from actual file contents verified via `read_file`
- The cache directory analysis is from an actual `.cache/jest/` directory generated during test runs
- nock version (13.5.6) confirmed from `node_modules/nock/package.json`

**Clarity standards:**
- Each question receives its own clearly delineated section
- Rationale/thinking is provided alongside each answer
- Technical details are progressive — starting with the "what" before the "why"
- Source citations accompany every technical claim

**Maintainability:**
- The document is self-contained and references only stable file paths within the repository
- No external URLs required (all evidence is from the codebase)
- Clear separation between observed facts and analytical conclusions

### 0.7.3 Example and Diagram Requirements

- **Timing tables:** Minimum 3 comparison scenarios per test file (cold, warm, no-cache)
- **Code excerpts:** Key configuration snippets from `jest.config.js`, `jest-preset.js`, `babel.config.js`, `setup-test-framework.js`
- **Cache directory listing:** File type breakdown with counts and sizes
- **No diagrams required:** The investigation is empirical rather than architectural


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — The sole deliverable: a comprehensive Q&A investigation document

**Source files analyzed (read-only — no modifications):**
- `test/client/jest.config.js` — Cache configuration, transform ignore patterns, module name mapper
- `test/client/setup-test-framework.js` — Nock initialization, browser polyfills, global mocks
- `test/server/jest.config.js` — Server cache configuration (for comparison)
- `test/packages/jest-preset.js` — Package cache configuration (for comparison)
- `test/build-tools/jest.config.js` — Build-tools cache configuration (for comparison)
- `packages/calypso-jest/jest-preset.js` — Shared transform rules, test discovery, resolver config
- `packages/calypso-jest/src/module-resolver.js` — Custom enhanced-resolve module resolver
- `packages/calypso-jest/src/asset-transform.js` — Image/stylesheet file transform
- `packages/calypso-babel-config/config.js` — Babel configuration factory
- `packages/calypso-babel-config/presets/default.js` — Full preset/plugin chain
- `babel.config.js` — Root Babel entry point
- `client/test-helpers/use-nock/index.js` — Nock lifecycle wrapper
- `client/test-helpers/use-nock/README.md` — Nock helper usage documentation
- `client/state/data-layer/test/wpcom-api-middleware.js` — Primary test subject
- `client/state/data-layer/wpcom-http/test/index.js` — Nock-dependent test
- `client/state/data-layer/README.md` — Data-layer architecture documentation
- `package.json` — Root manifest with test scripts and dependency versions
- `packages/calypso-jest/package.json` — Jest preset package dependencies

**Test executions performed (read-only observation):**
- Cold-cache runs of `client/state/data-layer/test/wpcom-api-middleware.js`
- Warm-cache runs of the same file
- `--no-cache` runs of the same file
- Cold/warm/no-cache runs of `client/state/data-layer/wpcom-http/` (6 test files)
- Cold/warm/no-cache runs of `client/state/data-layer/test/` (4 test files)
- Cache directory inspection after test runs

**Runtime artifacts examined:**
- `.cache/jest/jest-transform-cache-*` — Babel-transpiled JavaScript output + source maps
- `.cache/jest/haste-map-*` — Jest's file system metadata cache
- `.cache/jest/perf-cache-*` — Jest's test timing performance cache

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No changes to any `.js`, `.ts`, `.jsx`, `.tsx`, or configuration files in the repository
- **Test file modifications:** No changes to any test files, fixtures, or mocks
- **New test creation:** No new test files will be created
- **Configuration changes:** No modifications to Jest, Babel, or nock configuration
- **Dependency additions or upgrades:** No package.json changes
- **Documentation updates to existing files:** No changes to `docs/testing/*.md`, `test/README.md`, `client/state/data-layer/README.md`, or any other existing documentation
- **Performance optimization implementation:** This is an investigation document, not a fix; no performance improvements will be implemented
- **CI/CD pipeline changes:** No modifications to `.teamcity/`, `.github/`, or `.circleci/` configurations
- **Unrelated test suites:** Tests outside `client/state/data-layer/` are not within the investigation scope


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not a documentation site
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/wp-calypso_be7e5cc64162.md`
- **Test execution commands used for data collection:**
  - Cold cache run: `rm -rf .cache/jest && jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 "client/state/data-layer/test/wpcom-api-middleware.js"`
  - Warm cache run: `jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 "client/state/data-layer/test/wpcom-api-middleware.js"` (immediately after cold run)
  - No-cache run: `jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 --no-cache "client/state/data-layer/test/wpcom-api-middleware.js"`
  - Resolved config inspection: `jest --config test/client/jest.config.js --showConfig`
- **Default format:** Markdown with fenced code blocks (no Mermaid diagrams required)
- **Citation requirement:** Every technical claim must reference the source file path and line number
- **Style guide:** Follows the `SWE-AtlasQnA-Repo` rule — comprehensive Q&A with thinking/rationale, no assumptions, code as truth
- **Documentation validation:** Visual review of Markdown rendering; verify all file path references are valid


## 0.10 Rules for Documentation

The following rules apply to this documentation task, as explicitly specified by the user and the implementation rule configuration:

- **Do not modify any existing files in the source repository.** All investigation is read-only. The only file creation allowed is the output document in `blitzy/documentation/`.
- **Create a new markdown document named `<source_branch_name>.md`.** The source branch is `wp-calypso_be7e5cc64162`, so the file must be named `wp-calypso_be7e5cc64162.md`.
- **Place the generated document in the `blitzy/documentation` directory** in the destination repository.
- **Provide thinking/rationale behind the answers.** Every conclusion must be accompanied by the reasoning chain that produced it, citing specific files and configuration lines.
- **Do not make assumptions; base answers on the code as the truth.** All claims about caching behavior, transformation pipeline, and mock configuration must be verified against actual file contents and empirical test executions.
- **All timing measurements must be from actual test runs** performed within this investigation session, using the same Node.js version (v22.9.0) and Yarn version (4.0.2) required by the repository.
- **Cache analysis must be from actual cache directory inspection** after test execution, not from documentation or assumptions about what should be there.
- **Nock tracing must follow the actual import chain** from `test/client/setup-test-framework.js` through `client/test-helpers/use-nock/index.js` to the test files, citing line numbers.


## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

**Configuration and Infrastructure Files Examined:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `package.json` | Root manifest — Node ^v22.9.0, Yarn ^4.0.0, test scripts, dependency versions (jest ^29.7.0, nock ^13.5.6, babel-jest ^29.7.0) |
| `babel.config.js` | Root Babel entry point — delegates to `@automattic/calypso-babel-config` with `@emotion/react` import source |
| `test/client/jest.config.js` | Client Jest config — `cacheDirectory: path.join(__dirname, '../../.cache/jest')` (line 7), extends `@automattic/calypso-jest` |
| `test/client/setup-test-framework.js` | Client test bootstrap — nock initialization (lines 6-22), 12+ global polyfill/mock installations |
| `test/server/jest.config.js` | Server Jest config — same `cacheDirectory` pattern for comparison |
| `test/server/setup-test-framework.js` | Server test bootstrap — nock initialization, wpcom-proxy-request mock |
| `test/packages/jest-preset.js` | Package test preset — same `cacheDirectory` pattern |
| `test/build-tools/jest.config.js` | Build-tools Jest config — same `cacheDirectory` pattern |
| `test/apps/jest.config.js` | App test multi-project config |
| `test/README.md` | Testing infrastructure overview — seven suite configurations, deprecated helpers warning |
| `packages/calypso-jest/jest-preset.js` | Shared preset — transform rules (`babel-jest` for `.js`/`.ts`/`.jsx`/`.tsx`, `asset-transform.js` for images/styles), test match pattern, resolver |
| `packages/calypso-jest/package.json` | Preset dependencies — jest ^29.7.0, babel-jest ^29.7.0, enhanced-resolve ^5.8.3 |
| `packages/calypso-jest/src/module-resolver.js` | Custom resolver — `calypso:src` field priority, enhanced-resolve-based |
| `packages/calypso-jest/src/setup.js` | Minimal shared setup — global CSS.supports mock |
| `packages/calypso-jest/src/asset-transform.js` | Asset transform — returns `path.basename(filename)` as module export |
| `packages/calypso-babel-config/config.js` | Babel config factory — test env adds `@babel/preset-env` (node:current) + `babel-plugin-dynamic-import-node` |
| `packages/calypso-babel-config/presets/default.js` | Full preset chain — `@babel/preset-env`, `@babel/preset-react`, `@babel/preset-typescript`, 4 plugins |
| `build-tools/babel/babel-loader-cache-identifier/index.js` | Webpack cache identifier (for context, not directly used by Jest) |

**Data-Layer Source and Test Files Examined:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `client/state/data-layer/README.md` | Data layer architecture, middleware design, handler registration, file structure conventions |
| `client/state/data-layer/wpcom-api-middleware.js` | Core middleware source — dispatches to registered handlers |
| `client/state/data-layer/handler-registry.js` | Handler registration — `registerHandlers()`, `getHandlers()` |
| `client/state/data-layer/utils.js` | Utility functions — `bypassDataLayer()` |
| `client/state/data-layer/test/wpcom-api-middleware.js` | Primary test subject — 12 tests, jest.fn() mocking, no nock |
| `client/state/data-layer/test/handler-registry.js` | Handler registry tests |
| `client/state/data-layer/test/utils.js` | Utility function tests |
| `client/state/data-layer/test/convert-snake-case-to-camel-case.ts` | TypeScript test file — exercises TS transform path |
| `client/state/data-layer/wpcom-http/test/index.js` | Nock-dependent test — imports `useNock` and `nock`, intercepts WordPress.com API |
| `client/state/data-layer/wpcom-http/test/actions.js` | HTTP action tests |
| `client/state/data-layer/wpcom-http/test/utils.js` | HTTP utility tests |
| `client/test-helpers/use-nock/index.js` | Nock lifecycle wrapper — exports `useNock` and re-exports `nock` |
| `client/test-helpers/use-nock/README.md` | Usage documentation for the nock helper |
| `client/test-helpers/use-nock/integration/index.js` | Integration test for the nock helper itself |

**Documentation Files Examined:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `docs/testing/testing-overview.md` | Test suite structure, execution commands |
| `docs/testing/unit-tests.md` | Unit test conventions, mocking strategy |
| `docs/testing/component-tests.md` | Component testing patterns |
| `docs/testing/faq.md` | Toolchain summary |
| `docs/testing/index.md` | Testing pyramid philosophy |
| `docs/testing/snapshot-testing.md` | Snapshot testing guidelines |

**Folders Explored:**

| Folder Path | Purpose |
|-------------|---------|
| `` (root) | Repository structure overview — identified `client/`, `packages/`, `test/`, `docs/`, `build-tools/` |
| `test/` | Test infrastructure hub — 7 suite subdirectories |
| `test/client/` | Client test harness — `jest.config.js` + `setup-test-framework.js` |
| `docs/` | Central documentation hub |
| `docs/testing/` | Testing documentation — 6 guide files |
| `client/state/data-layer/` | Data-layer module root |
| `client/state/data-layer/test/` | Top-level data-layer tests (4 files) |
| `client/state/data-layer/wpcom-http/` | HTTP subsystem |
| `client/state/data-layer/wpcom/` | WordPress.com API handler tree |
| `packages/calypso-jest/` | Shared Jest preset package |
| `packages/calypso-babel-config/` | Shared Babel configuration |
| `.cache/jest/` | Runtime-generated cache directory (inspected after test runs) |

### 0.11.2 Attachments

No attachments were provided for this project. No Figma screens or external design assets are referenced.

### 0.11.3 Tech Spec Sections Referenced

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project overview context — wp-calypso monorepo, Yarn 4, Node 22.9 workspace |
| 6.6 Testing Strategy | Comprehensive testing infrastructure documentation — Jest configuration, test suites, nock usage, Babel transform pipeline, CI integration |


