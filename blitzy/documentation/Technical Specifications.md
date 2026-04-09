# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of investigative questions about the selector memoization and caching behavior within the wp-calypso monorepo's state management system.

**Category:** Create new documentation
**Documentation Type:** Technical investigation / Q&A reference document

The user is experiencing unexpected caching behavior where stale results are returned from a memoized selector even after underlying state has changed. The investigation requires deep code analysis and empirical validation through test execution — not source code modification. The resulting document must be placed at `blitzy/documentation/wp-calypso_be7e5cc64162.md` per the project's implementation rules.

The user's questions decompose into the following discrete documentation requirements:

- **Comparison Mechanism:** Document the precise comparison logic that determines cache hits versus cache misses in both `createSelector` (`@automattic/state-utils`) and `treeSelect` (`@automattic/tree-select`)
- **Selector Invocation Counts:** Provide concrete numbers from actual test runs showing how many times the underlying selector function is called when dependent state changes between successive invocations
- **Cache Entry Strategy:** Determine whether the cache maintains separate entries for each unique filter argument or if calling with different arguments invalidates the previous cached result
- **Programmatic Cache Clearing:** Document whether and how a selector's entire cache can be cleared programmatically at runtime
- **Nullish vs. Primitive Dependants:** Explain the behavioral differences when dependency getter functions return `null`/`undefined` versus primitive values like numbers or booleans
- **Custom Cache Key Generation:** Describe the mechanism for customizing how cache keys are generated, specifically for complex query objects passed as arguments

### 0.1.2 Special Instructions and Constraints

- **CRITICAL CONSTRAINT:** The user explicitly stated: "Please don't modify any source files during your investigation." All analysis must be read-only.
- **Implementation Rule (SWE-AtlasQnA-Repo):** The deliverable must be a new markdown document named `wp-calypso_be7e5cc64162.md` placed in the `blitzy/documentation` directory. The document must provide thinking and rationale behind all answers. All answers must be grounded in the code, not assumptions.
- **No existing files may be modified** in the source repository.
- **Style Preference:** Technical depth with concrete evidence from code and test runs. Answers should include code citations and empirical test output.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **comparison mechanism**, we will analyze and explain the implementation of `createSelector` at `packages/state-utils/src/create-selector/index.ts` (which uses `@wordpress/is-shallow-equal` for dependency comparison and `lodash.memoize` for argument-level caching) and `treeSelect` at `packages/tree-select/src/index.ts` (which uses a `WeakMap`-based dependency tree with `Map` leaf nodes keyed by `args.join()`)
- To provide **concrete invocation counts**, we will document test results from `packages/state-utils/src/create-selector/test/index.js` and `packages/tree-select/test/index.js`, supplemented by custom runtime experiments exercising the selector factories with tracked `jest.fn()` / manual call counters
- To explain **cache entry strategy**, we will document the internal caching data structures: `lodash.memoize`'s `MapCache` for `createSelector` and the `WeakMap→WeakMap→Map` tree for `treeSelect`, showing their distinct multi-key vs. single-global-cache behaviors
- To document **programmatic cache clearing**, we will explain the `memoizedSelector.cache.clear()` method for `createSelector` and the `clearCache()` method exposed on `treeSelect` selectors
- To document **nullish vs. primitive behavior**, we will explain how `createSelector` uses `isShallowEqual` on dependency arrays (where nullish values are compared by strict equality) versus `treeSelect`'s `insertDependentKey` function (which maps nullish values to a `NULLISH_KEY` sentinel object but throws a `TypeError` for non-nullish primitives)
- To document **custom cache key generation**, we will explain the third parameter of `createSelector` and the `getCacheKey` option of `treeSelect`, including development-mode warnings for complex object arguments

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Cache key collision risk:** The default cache key strategy (`args.join()`) produces identical keys for `null` and `undefined` arguments, and for arguments like `["1,2"]` vs. `[1, 2]`. This is a likely source of stale data and must be documented.
- **Dependency declaration pitfalls:** If a `getDependants` function does not capture all relevant state branches, mutations to uncaptured branches will not trigger cache invalidation — a common source of the "stale results" the user is observing.
- **createSelector vs. treeSelect architectural differences:** The user's description ("filtered data from a central store") matches both selectors' usage patterns, but the two have fundamentally different caching architectures. The document must clarify when to use each.
- **Direct state mutation anti-pattern:** If any reducer or middleware mutates state in-place rather than producing a new reference, shallow equality checks will report no change and cached results will be stale.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature documentation ecosystem with Markdown as the primary format and no dedicated documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` detected). Documentation is authored as standalone `.md` files organized within two primary locations:

- **`docs/` directory (35 files):** Central documentation hub covering contributor workflow, installation, monorepo conventions, routing, SSR, state management, accessibility, i18n, performance, component standards, and testing
- **Package-level READMEs:** Each workspace package (80+ under `packages/`) contains its own `README.md` documenting the package's API and usage

Key existing documentation files relevant to this investigation:

| File | Content Summary |
|------|----------------|
| `docs/our-approach-to-data.md` | History of data management eras (Emitter → Flux → Redux → Modularized Redux → React Query), current recommendations, Redux terminology, folder structure |
| `docs/modularized-state.md` | Modularized Redux architecture — `init` modules, `withStorageKey`, dynamic reducer registration |
| `docs/data-persistence.md` | Redux state persistence to IndexedDB, schema validation, serialization/deserialization |
| `docs/reactivity.md` | Reactive UI principles, data-poller approach, avoiding loading spinners |
| `packages/state-utils/src/create-selector/README.md` | `createSelector` API documentation — memoization, dependency tracking, cache key behavior, `memoizedSelector` access |
| `packages/tree-select/README.md` | `treeSelect` API documentation — cached selector factory, WeakMap dependency tree, garbage collection model |
| `client/state/selectors/README.md` | Selector architecture conventions — one selector per file, kebab-case naming, colocated tests, direct import paths |

**Documentation framework:** None (raw Markdown files served via GitHub)
**API documentation tools:** None formally configured (no JSDoc/TypeDoc generation pipeline detected)
**Diagram tools:** Mermaid is used within the tech spec; no dedicated diagram tooling in the repo itself

### 0.2.2 Repository Code Analysis for Documentation

The following code modules were analyzed to derive answers for the investigation:

**Primary Implementation Files:**

| File Path | Relevance |
|-----------|-----------|
| `packages/state-utils/src/create-selector/index.ts` | Core `createSelector` implementation — memoization, dependency tracking, cache invalidation, custom cache key support |
| `packages/tree-select/src/index.ts` | Core `treeSelect` implementation — WeakMap dependency tree, `NULLISH_KEY` sentinel, `clearCache()`, `getCacheKey` option |
| `packages/state-utils/src/create-selector/test/index.js` | 13 test cases validating `createSelector` behavior: cache reuse, invalidation, multi-arg caching, custom keys, warnings |
| `packages/tree-select/test/index.js` | 17 test cases validating `treeSelect` behavior: caching, multi-dependent caching, nullish handling, clearCache, getCacheKey |

**Dependency Files:**

| File Path | Relevance |
|-----------|-----------|
| `packages/state-utils/package.json` | Declares `@automattic/state-utils` v1.0.0-alpha.4, runtime deps: `lodash ^4.17.21`, `@wordpress/is-shallow-equal ^5.21.0`, `@wordpress/warning ^3.21.0` |
| `packages/tree-select/package.json` | Declares `@automattic/tree-select` v2.0.0, minimal deps: `tslib ^2.3.0` |

**Real-World Usage Examples (examined for pattern verification):**

| File Path | Pattern |
|-----------|---------|
| `client/state/posts/selectors/get-site-posts.js` | `createSelector` with `(state) => state.posts.queries` as dependant |
| `client/state/reader/posts/selectors.js` | `treeSelect` with `(state) => [state.reader.posts.items]` as dependents |
| `client/state/comments/selectors/get-post-comments-tree.js` | `treeSelect` usage for comments |
| `client/state/invites/selectors.js` | `treeSelect` usage for invites |

### 0.2.3 Empirical Test Execution Results

Both test suites were executed successfully and passed all assertions:

**`createSelector` test suite:** 13 tests passed in 1.235s
- Validates memoized result reuse, cache invalidation on state change, distinct cache entries for different arguments, development-mode warnings, dependency array support, default behavior, custom cache key support, and argument forwarding

**`treeSelect` test suite:** 17 tests passed in 1.239s
- Validates caching, multi-dependent caching, argument validation, production/development mode differences, primitive argument rejection, cache busting on state change, simultaneous cache entries for unique dependents, `clearCache()` method, nullish dependent handling, primitive dependent rejection, and custom `getCacheKey` support

Additional runtime experiments were conducted to produce concrete invocation counts across 8-step and 9-step scenarios, documenting exact call counts at each step for both selector factories.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation document must cover two distinct memoized selector factories and their surrounding ecosystem:

**Module: `packages/state-utils/src/create-selector/index.ts`**
- Public API: `createSelector(selector, getDependants?, getCacheKey?)`
- Current documentation: `README.md` exists but does not cover nullish dependant behavior, concrete call count expectations, cache clearing mechanics, or cache key collision risks
- Documentation needed: Deep behavioral analysis with concrete numbers, edge cases, and stale-data scenarios

**Module: `packages/tree-select/src/index.ts`**
- Public API: `treeSelect(getDependents, selector, options?)`
- Exported types: `CachedSelector`, `Options`
- Current documentation: `README.md` exists but does not document `clearCache()`, nullish vs. primitive dependant behavior, or `getCacheKey` option
- Documentation needed: Complete behavioral reference covering WeakMap tree mechanics, cache isolation, nullish handling, and programmatic clearing

**Supporting Module: `@wordpress/is-shallow-equal` (v5.21.0)**
- Used by `createSelector` to compare dependency snapshots
- Documentation needed: Explanation of shallow equality semantics for arrays (element-by-element strict equality)

**Supporting Module: `lodash.memoize` (v4.17.21)**
- Used by `createSelector` for argument-level caching via `MapCache`
- Documentation needed: Explanation of `MapCache` behavior — multiple entries per unique cache key, `cache.clear()` method, `cache.size` property

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented Behavioral Details:**
- Neither README covers the exact number of selector invocations across state change scenarios with concrete test data
- The `createSelector` README does not explain that `lodash.memoize` maintains **separate** entries per unique cache key (multiple arguments are cached simultaneously within the same dependency snapshot)
- The `createSelector` README does not explain that a dependency change triggers `cache.clear()`, which destroys **all** cached entries across all argument combinations
- The `tree-select` README does not document the `clearCache()` method at all
- The `tree-select` README does not document the `getCacheKey` option
- Neither README documents the behavior when `getDependants` returns nullish values

**Undocumented Edge Cases:**
- Cache key collisions: `null` and `undefined` produce the same cache key (`""`) via `args.join()`, and string arguments containing commas can collide with multi-argument keys (e.g., `["1,2"]` produces the same key as `[1, 2]`)
- `treeSelect` throws `TypeError` when `getDependents` returns non-nullish primitive values (numbers, booleans, strings) because `WeakMap` keys must be objects or nullish
- `createSelector` emits a `@wordpress/warning` in development mode when non-primitive arguments are passed, but silently uses `[object Object]` as the cache key in production — leading to incorrect cache hits across different objects

**Missing Consolidated Reference:**
- No single document in the repository compares `createSelector` and `treeSelect` side-by-side, explaining when to use each and their fundamental architectural differences (single-layer MapCache vs. WeakMap dependency tree)

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive Markdown document that answers all six investigation questions with code-grounded evidence. The document will be structured as follows:

```
blitzy/documentation/
└── wp-calypso_be7e5cc64162.md
    ├── Introduction (investigation context and scope)
    ├── Q1: Cache Hit/Miss Comparison Mechanism
    │   ├── createSelector: isShallowEqual + lodash.memoize
    │   └── treeSelect: WeakMap referential identity + Map key lookup
    ├── Q2: Selector Invocation Counts (with concrete numbers)
    │   ├── createSelector: 8-step scenario with exact call counts
    │   └── treeSelect: 9-step scenario with exact call counts
    ├── Q3: Cache Entry Strategy per Argument
    │   ├── createSelector: MapCache maintains separate entries per key
    │   └── treeSelect: WeakMap tree maintains entries per dependent+key
    ├── Q4: Programmatic Cache Clearing
    │   ├── createSelector: memoizedSelector.cache.clear()
    │   └── treeSelect: selector.clearCache()
    ├── Q5: Nullish vs. Primitive Dependant Behavior
    │   ├── createSelector: isShallowEqual treats null/undefined/primitives uniformly
    │   └── treeSelect: NULLISH_KEY sentinel for null/undefined, TypeError for primitives
    ├── Q6: Custom Cache Key Generation
    │   ├── createSelector: third argument getCacheKey
    │   └── treeSelect: options.getCacheKey
    ├── Common Pitfalls and Stale Data Causes
    └── Architectural Comparison Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract comparison mechanism from `packages/state-utils/src/create-selector/index.ts` lines 96–112 (the wrapper function with `isShallowEqual` check and `memoizedSelector.cache.clear()`)
- Extract WeakMap tree caching from `packages/tree-select/src/index.ts` lines 65–101 (the `cachedSelector` function with `dependents.reduce(insertDependentKey, cache)`)
- Generate invocation count tables from the runtime experiments executed during context gathering
- Extract cache clearing APIs from source code: `memoizedSelector` property at `create-selector/index.ts:111` and `clearCache` method at `tree-select/src/index.ts:96-99`
- Extract nullish handling from `tree-select/src/index.ts` lines 107–131 (`NULLISH_KEY` sentinel and `insertDependentKey` type check)
- Extract custom cache key from `create-selector/index.ts` line 88 (third parameter) and `tree-select/src/index.ts` lines 24–27 and 67 (`Options` interface with `getCacheKey`)

**Source Citations:**
- Every technical claim will cite the specific source file and line number
- Test assertions will reference the test file and test case name
- Runtime experiment results will be presented as reproducible step-by-step tables

### 0.4.3 Documentation Standards

- Markdown formatting with `#`, `##`, `###` heading hierarchy
- Code examples using fenced blocks with `js` or `ts` language tags
- Mermaid diagrams for the dependency comparison flow and cache data structures
- Tables for invocation count data and behavioral comparison matrices
- Source citations in the format: `Source: packages/state-utils/src/create-selector/index.ts:L96-L112`

### 0.4.4 Diagram and Visual Strategy

The following Mermaid diagrams will be included in the document:

- **createSelector cache flow:** Flowchart showing the decision path from invocation through dependency comparison, cache clear, cache key generation, and cache hit/miss
- **treeSelect WeakMap tree structure:** Diagram illustrating the nested WeakMap→WeakMap→Map cache hierarchy and how dependent objects serve as tree keys
- **Comparison matrix:** Side-by-side table comparing the two selectors across all six investigation dimensions

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `packages/state-utils/src/create-selector/index.ts`, `packages/tree-select/src/index.ts`, `packages/state-utils/src/create-selector/test/index.js`, `packages/tree-select/test/index.js`, `packages/state-utils/src/create-selector/README.md`, `packages/tree-select/README.md` | Comprehensive Q&A document answering all six caching behavior investigation questions with concrete test numbers, code citations, and architectural analysis |

**Note:** Only one file is created. Per the implementation rule `SWE-AtlasQnA-Repo`, the document is named after the source branch (`wp-calypso_be7e5cc64162`) and placed in the `blitzy/documentation` directory. No existing files in the source repository are modified.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Technical Investigation / Q&A Reference
Source Code:
  - packages/state-utils/src/create-selector/index.ts (primary implementation)
  - packages/tree-select/src/index.ts (primary implementation)
  - packages/state-utils/src/create-selector/test/index.js (test validation)
  - packages/tree-select/test/index.js (test validation)
  - packages/state-utils/src/create-selector/README.md (existing documentation context)
  - packages/tree-select/README.md (existing documentation context)
  - packages/state-utils/package.json (version and dependency info)
  - packages/tree-select/package.json (version and dependency info)
  - client/state/posts/selectors/get-site-posts.js (real-world createSelector usage)
  - client/state/reader/posts/selectors.js (real-world treeSelect usage)
  - client/state/selectors/README.md (selector architecture conventions)
  - docs/our-approach-to-data.md (data management history and architecture)
  - docs/modularized-state.md (modularized Redux architecture)
Sections:
  - Introduction (investigation scope, relevant packages, methodology)
  - Q1: Cache Comparison Mechanism (isShallowEqual, referential identity, args.join())
  - Q2: Selector Invocation Counts (8-step createSelector scenario, 9-step treeSelect scenario)
  - Q3: Cache Entry Strategy (MapCache multi-key vs. WeakMap tree per-dependent)
  - Q4: Programmatic Cache Clearing (memoizedSelector.cache.clear(), selector.clearCache())
  - Q5: Nullish vs. Primitive Dependants (isShallowEqual behavior, NULLISH_KEY, TypeError)
  - Q6: Custom Cache Key Generation (third param, options.getCacheKey, development warnings)
  - Common Pitfalls (stale data causes, cache key collisions, mutation anti-patterns)
  - Architectural Comparison (createSelector vs. treeSelect decision matrix)
Diagrams:
  - createSelector cache invalidation flowchart
  - treeSelect WeakMap tree structure diagram
  - Side-by-side comparison table
Key Citations:
  - packages/state-utils/src/create-selector/index.ts
  - packages/tree-select/src/index.ts
  - packages/state-utils/src/create-selector/test/index.js
  - packages/tree-select/test/index.js
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration files need to be updated, as the repository uses raw Markdown files without a formal documentation build system. The `blitzy/documentation/` directory will be created as a new directory to house the deliverable.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are directly relevant to this investigation and must be accurately referenced in the documentation:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm (workspace) | `@automattic/state-utils` | 1.0.0-alpha.4 | Exports `createSelector` — the primary memoized selector factory used across 90+ Redux state domains |
| npm (workspace) | `@automattic/tree-select` | 2.0.0 | Exports `treeSelect` — the WeakMap-based cached selector factory used for comments, reader posts, invites, stats |
| npm | `lodash` | ^4.17.21 | Provides `memoize` function used internally by `createSelector` for argument-level caching via `MapCache` |
| npm | `@wordpress/is-shallow-equal` | ^5.21.0 | Provides `isShallowEqual` used by `createSelector` to compare dependency snapshots between invocations |
| npm | `@wordpress/warning` | ^3.21.0 | Provides `warn` used by `createSelector` in development mode to warn about complex object arguments |
| npm | `tslib` | ^2.3.0 | TypeScript helper runtime used by both packages |
| npm | `redux` | ^5.0.1 | State management framework that provides the global store these selectors operate against |
| npm | `redux-thunk` | ^3.1.0 | Middleware for asynchronous Redux actions |

### 0.6.2 Runtime and Build Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | ^22.9.0 (per `.nvmrc` and `package.json` engines) | JavaScript runtime |
| Yarn | 4.0.2 (Berry) | Package manager with workspace support |
| TypeScript | ^5.8.2 | Both packages use TypeScript for source |
| Jest | (via `@automattic/calypso-jest` preset) | Test runner for executing selector test suites |

### 0.6.3 Documentation Reference Updates

No documentation link updates are required. The new document is a standalone investigation artifact placed in `blitzy/documentation/` and does not need to be integrated into existing documentation navigation structures.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the investigation scope:**

| Topic | Existing Coverage | Target Coverage | Gap |
|-------|-------------------|-----------------|-----|
| Cache comparison mechanism (`createSelector`) | Partial — README mentions immutable update model and `Array.prototype.join` | 100% — full code-level explanation with `isShallowEqual` and `lodash.memoize` internals | Concrete comparison flow missing |
| Cache comparison mechanism (`treeSelect`) | Partial — README mentions referential equality | 100% — full explanation of `WeakMap` tree traversal and `Map` key lookup | `insertDependentKey` logic undocumented |
| Selector invocation counts | 0% — no existing document provides concrete numbers | 100% — step-by-step tables with exact call counts from runtime experiments | Entirely missing |
| Cache entry strategy per argument | Partial — `createSelector` README alludes to `args.join()` cache key | 100% — explicit documentation of `MapCache` multi-entry behavior and `treeSelect` per-dependent isolation | Multi-entry behavior not explained |
| Programmatic cache clearing | Partial — `createSelector` README mentions `memoizedSelector` property | 100% — explicit API documentation for both selectors | `treeSelect.clearCache()` entirely undocumented in README |
| Nullish vs. primitive dependant behavior | 0% — neither README documents this | 100% — complete behavioral matrix with code evidence | Entirely missing |
| Custom cache key generation | Partial — `createSelector` README mentions optional third argument | 100% — full examples for both selectors with complex object use case | `treeSelect` `getCacheKey` option undocumented in README |

**Target:** 100% coverage of all six investigation questions with code-grounded evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**
- Every question posed by the user must receive a direct, unambiguous answer
- Every answer must cite the specific source file and line number that supports it
- Every behavioral claim must be validated by either a passing test case or a reproducible runtime experiment
- The document must cover both `createSelector` and `treeSelect` for each question, since both are used in the codebase for memoized selectors

**Accuracy Validation:**
- All test suites (`create-selector`: 13 tests, `tree-select`: 17 tests) must pass before claims are made about expected behavior
- Runtime experiments must be reproducible by executing the documented code against the repository's installed dependencies
- Code snippets in the document must match the actual source code in the repository

**Clarity Standards:**
- Each question gets its own clearly labeled section with a direct answer followed by detailed explanation
- Technical depth is appropriate for a developer investigating caching bugs — assume familiarity with Redux, selectors, and memoization concepts
- Concrete numbers and code references take priority over abstract explanations
- Potential pitfalls and edge cases are explicitly called out

### 0.7.3 Example and Diagram Requirements

- Minimum of 1 concrete code example per question demonstrating the described behavior
- Mermaid flowchart for the `createSelector` cache invalidation decision path
- Mermaid diagram for the `treeSelect` WeakMap tree structure
- Comparison table summarizing behavioral differences across all six dimensions
- Step-by-step invocation count tables with at least 8 steps for each selector factory

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — the sole deliverable

**Source code analyzed (read-only):**
- `packages/state-utils/src/create-selector/index.ts` — `createSelector` implementation
- `packages/state-utils/src/create-selector/test/index.js` — `createSelector` test suite
- `packages/state-utils/src/create-selector/README.md` — existing `createSelector` documentation
- `packages/state-utils/package.json` — package version and dependencies
- `packages/state-utils/src/index.ts` — barrel export verifying `createSelector` is the public API
- `packages/tree-select/src/index.ts` — `treeSelect` implementation
- `packages/tree-select/test/index.js` — `treeSelect` test suite
- `packages/tree-select/README.md` — existing `treeSelect` documentation
- `packages/tree-select/package.json` — package version and dependencies
- `client/state/posts/selectors/get-site-posts.js` — real-world `createSelector` usage example
- `client/state/reader/posts/selectors.js` — real-world `treeSelect` usage example
- `client/state/selectors/README.md` — selector conventions documentation
- `docs/our-approach-to-data.md` — data management architecture context
- `docs/modularized-state.md` — modularized Redux state documentation
- `docs/data-persistence.md` — state persistence documentation
- `docs/reactivity.md` — reactivity principles documentation

**Test execution (read-only, non-modifying):**
- `packages/state-utils/src/create-selector/test/index.js` — 13 test cases
- `packages/tree-select/test/index.js` — 17 test cases

**Runtime experiments (read-only, transient):**
- Custom Node.js scripts exercising `createSelector` and `treeSelect` behavior with call counting

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository may be modified, per the user's explicit instruction and the `SWE-AtlasQnA-Repo` implementation rule
- **Test file modifications:** No test files may be changed
- **Feature additions or code refactoring:** This is a documentation-only investigation
- **Deployment configuration changes:** No CI/CD, Docker, or infrastructure changes
- **Unrelated documentation:** Only the caching/memoization investigation questions are addressed; no other documentation is created or updated
- **React Query (`@tanstack/react-query`) analysis:** While the codebase is migrating toward React Query, the user's questions are specifically about Redux selector caching (`createSelector` and `treeSelect`), not React Query's caching model
- **Reselect library analysis:** Although `createSelector` README mentions `reselect` as a comparison point, the user's codebase does not use `reselect` — only the custom `createSelector` and `treeSelect` implementations
- **State reducer or action creator analysis:** The investigation focuses on selector memoization, not reducer logic or action dispatch patterns

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Test execution command (createSelector):**
  `CI=true npx jest --config packages/state-utils/jest.config.js --testPathPattern 'create-selector' --no-cache --watchAll=false --verbose`
- **Test execution command (treeSelect):**
  `CI=true npx jest --config packages/tree-select/jest.config.js --testPathPattern 'test/index' --no-cache --watchAll=false --verbose`
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with file path and line numbers
- **Style guide:** Follow the existing documentation patterns found in `packages/state-utils/src/create-selector/README.md` and `packages/tree-select/README.md` — concise prose with code examples, FAQ-style question framing, and practical usage guidance
- **Documentation validation:** Manual review to ensure all six questions are fully answered with code-grounded evidence; no automated link checking or linting is required for a standalone investigation document

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The investigation is strictly read-only. All analysis is performed through file reading, test execution, and runtime experiments — no code changes.
- **Create a new markdown document named `wp-calypso_be7e5cc64162.md`** that comprehensively answers the questions posed in the prompt.
- **Place the generated document in the `blitzy/documentation` directory** in the destination repo.
- **Provide thinking / rationale behind the answers.** Every answer must include the reasoning chain: what code was examined, what behavior was observed, and why the conclusion follows.
- **Do not make assumptions, base answers on the code as the truth.** Every claim must cite a specific file, line number, or test result. No speculation or documentation of "typical" behavior that isn't verified in this specific codebase.
- **Cover both `createSelector` and `treeSelect`** for each question, since the user's description of "filtered data from a central store" matches the usage pattern of both selector factories present in the codebase.
- **Include concrete numbers from test runs** for the selector invocation count question — not approximate or theoretical counts.
- **Document cache key collision risks** that may explain the user's "stale results" observation, including the `null`/`undefined` collision and comma-in-string collision via `args.join()`.
- **Document the development-mode warning** about complex object arguments, since the user asks about passing complex query objects as arguments.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were systematically examined to derive all conclusions in this Agent Action Plan:

**Primary Implementation Files (read in full):**

| File Path | Lines | Purpose |
|-----------|-------|---------|
| `packages/state-utils/src/create-selector/index.ts` | 1–113 | Complete `createSelector` implementation — memoization, dependency tracking, cache key logic, shallow equality comparison |
| `packages/tree-select/src/index.ts` | 1–132 | Complete `treeSelect` implementation — WeakMap dependency tree, `NULLISH_KEY` sentinel, `insertDependentKey`, `clearCache()`, `getCacheKey` option |
| `packages/state-utils/src/create-selector/test/index.js` | 1–291 | Full `createSelector` test suite — 13 test cases covering cache reuse, invalidation, multi-arg, warnings, dependency arrays, custom keys |
| `packages/tree-select/test/index.js` | 1–266 | Full `treeSelect` test suite — 17 test cases covering caching, multi-dependent, argument validation, clearCache, nullish handling, getCacheKey |
| `packages/state-utils/src/create-selector/README.md` | 1–70 | Existing `createSelector` documentation |
| `packages/tree-select/README.md` | 1–71 | Existing `treeSelect` documentation |
| `packages/state-utils/package.json` | 1–44 | Package metadata and dependency versions |
| `packages/tree-select/package.json` | 1–43 | Package metadata and dependency versions |
| `packages/state-utils/src/index.ts` | 1–4 | Barrel export confirming public API surface |

**Contextual Documentation Files (read in full or partially):**

| File Path | Purpose |
|-----------|---------|
| `docs/our-approach-to-data.md` | Data management history and current Redux architecture recommendations |
| `docs/modularized-state.md` | Modularized Redux state with `init` modules and `withStorageKey` |
| `docs/data-persistence.md` | State persistence to IndexedDB |
| `docs/reactivity.md` | Reactive UI principles |
| `client/state/selectors/README.md` | Selector architecture conventions |

**Real-World Usage Examples (read for pattern verification):**

| File Path | Pattern |
|-----------|---------|
| `client/state/posts/selectors/get-site-posts.js` | `createSelector` with `(state) => state.posts.queries` |
| `client/state/reader/posts/selectors.js` | `treeSelect` with `(state) => [state.reader.posts.items]` |

**Folders Explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (repository root) | 0 | Top-level structure assessment |
| `packages/state-utils/` | 1 | Package structure and contents |
| `packages/state-utils/src/create-selector/` | 2 | Implementation, README, and test folder |
| `packages/state-utils/src/create-selector/test/` | 3 | Test file |
| `packages/tree-select/` | 1 | Package structure and contents |
| `packages/tree-select/src/` | 2 | Implementation file |
| `packages/tree-select/test/` | 2 | Test file |
| `docs/` | 1 | Documentation directory listing |
| `client/state/selectors/` | 1 (via search) | Selector architecture and patterns |
| `client/state/posts/selectors/` | 2 (via search) | Real-world selector examples |

**Runtime Configuration Files:**

| File Path | Purpose |
|-----------|---------|
| `.nvmrc` | Node.js version requirement (22.9.0) |
| `package.json` | Root monorepo manifest with engines and workspace config |
| `.yarnrc.yml` | Yarn 4 configuration with node-modules linker |
| `test/packages/jest-preset.js` | Shared Jest preset for workspace packages |

### 0.11.2 Test Execution Results

| Test Suite | File | Tests Passed | Total | Time |
|------------|------|-------------|-------|------|
| `createSelector` | `packages/state-utils/src/create-selector/test/index.js` | 13 | 13 | 1.235s |
| `treeSelect` | `packages/tree-select/test/index.js` | 17 | 17 | 1.239s |

### 0.11.3 Runtime Experiment Summary

| Experiment | Steps | Key Finding |
|------------|-------|-------------|
| `createSelector` 8-step invocation count | 8 | Selector called 4 times total: state change clears ALL cached entries, but different args within same state maintain separate cache entries |
| `treeSelect` 9-step invocation count | 9 | Selector called 5 times total (including post-clearCache): different dependents maintain fully isolated cache subtrees via WeakMap |
| `isShallowEqual` behavioral matrix | 11 comparisons | Strict reference equality (`===`) for array elements; `null === null` is true, `null !== undefined` |
| `args.join()` cache key collision analysis | 10 keys + 3 collisions | `null` and `undefined` produce empty string; `["1,2"]` collides with `[1, 2]` |
| Nullish dependant behavior (createSelector) | 8 states | `null`, `undefined`, `0`, `false`, and `'hello'` all work as dependant values with correct strict equality comparison |
| Nullish/primitive dependant behavior (treeSelect) | 8 values | `null` and `undefined` accepted via `NULLISH_KEY` sentinel; `true`, `false`, `1`, `0`, `'a'`, `''` all throw `TypeError` |

### 0.11.4 Attachments

No attachments were provided by the user. No Figma URLs were specified.

