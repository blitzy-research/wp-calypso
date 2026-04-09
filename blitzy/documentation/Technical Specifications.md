# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a behavioural investigation into the Calypso multi-step onboarding/signup flow's back-navigation system — specifically, why the Back button sometimes jumps to the first step or exits the flow entirely instead of retreating one step at a time.

- **Documentation Type:** Technical Q&A / Architecture Investigation Document
- **Category:** Create new documentation
- **Target artifact:** A markdown file named `wp-calypso_be7e5cc64162.md` placed in `blitzy/documentation/`, per the project rule "SWE-AtlasQnA-Repo"

The user's questions decompose into five discrete investigation objectives:

| # | Question | Technical Translation |
|---|----------|----------------------|
| Q1 | What actually decides the destination for the Back button at a given step? | Document the complete precedence chain governing back-navigation target resolution across both the classic signup system (`/start/`) and the declarative stepper system (`/setup/`). |
| Q2 | Which inputs win when flow position, component props, and query-string arguments disagree? | Identify and rank every input — `ownProps.backUrl`, `back_to` query parameter, computed previous step, flow-defined `goBack`, browser `history.back()` — and describe the resolution order. |
| Q3 | Where does the external back-target override come from? | Trace the `back_to` query parameter from URL entry through the signup controller dispatch, Redux dependency store, step consumption, and `StepWrapper` connect HOC. |
| Q4 | What precedence rule lets the override take control, and what code path handles the expected step-by-step navigation that is being bypassed? | Explain the nullish coalescing in `StepWrapper`'s `connect` (`ownProps.backUrl ?? backTo`), the early-return in `NavigationLink.getBackUrl()`, and the `getPreviousStep()` method that is skipped when `backUrl` is truthy. |
| Q5 | Observe the computed destination for each step position to confirm the pattern. | Provide a diagnostic observation strategy (temporary script or manual trace) and a per-step destination table for representative flows, without modifying the repository. |

### 0.1.2 Special Instructions and Constraints

- **No repository modifications:** The repository itself must remain unchanged. Temporary scripts used for observation must be cleaned up afterward.
- **Provide rationale:** Per the project rule, answers must include thinking/rationale and be based on the code as truth, with no assumptions.
- **Documentation style:** The output is a standalone markdown document with headings, code excerpts, tables, and Mermaid diagrams.
- **Code citations required:** Every claim must reference a specific file path and, where useful, line numbers.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To answer Q1–Q4, we will **create** `blitzy/documentation/wp-calypso_be7e5cc64162.md` containing an architectural walkthrough of back-navigation resolution across the two onboarding systems, citing every file in the decision chain.
- To answer Q5, we will describe a diagnostic approach (and optionally include a temporary Node script pattern) that logs the resolved back-navigation target per step position for a chosen flow, without leaving any trace in the repository.

### 0.1.4 Inferred Documentation Needs

Based on code analysis the following implicit documentation needs surface:

- **Dual-system explanation:** The repository has two independent multi-step navigation architectures (classic signup at `client/signup/` and declarative stepper at `client/landing/stepper/`). The document must distinguish these clearly so the reader understands which system applies.
- **`back_to` lifecycle diagram:** The `back_to` query parameter is the central override mechanism. Its lifecycle from URL → controller dispatch → Redux dependency store → step prop → StepWrapper connect HOC → NavigationLink requires a dedicated Mermaid sequence diagram.
- **Per-step back-URL computation in the domains step:** The domains step (`client/signup/steps/domains/index.jsx`) has an unusually complex back-URL computation chain (flow-specific branches, `source` query param, `getExternalBackUrl`, playground ID). This must be documented as a special case.
- **`allowBackFirstStep` side-effect:** When `backUrl` is truthy, `StepWrapper` forces `allowBackFirstStep` to `true`, which makes the back button appear even on position 0 in the flow — this is a key part of the override behaviour the user is observing.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature docs directory at `docs/` containing contributor-oriented markdown documentation. There is no dedicated documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx` configuration). Documentation is authored as standalone Markdown files.

- **Current documentation framework:** Plain Markdown (no generator)
- **Documentation generator configuration location:** N/A
- **API documentation tools in use:** `eslint-plugin-jsdoc` (v46.10.1) for enforcement of JSDoc comments in source; no API-doc-site generation
- **Diagram tools detected:** Mermaid (used in tech spec and inline README files)
- **Documentation hosting/deployment setup:** GitHub-rendered Markdown; docs served directly from the repository

Existing documentation reviewed for relevance:

| Path | Relevance |
|------|-----------|
| `docs/routing.md` | Describes the section/controller routing pattern used by the classic signup system |
| `docs/isomorphic-routing.md` | Covers SSR-capable route registration that underpins `/start` |
| `docs/our-approach-to-data.md` | Explains the Redux state management approach used by signup progress and dependency stores |
| `docs/development-workflow.md` | Contributor workflow; establishes coding and documentation norms |
| `client/landing/stepper/README.md` | Stepper-framework guide: explains `useStepNavigation`, `initialize`, and reusability contract |
| `client/landing/stepper/declarative-flow/flows/onboarding/README.md` | Minimal: points to `/setup/onboarding` for manual testing; ownership metadata only |

No existing documentation specifically addresses back-navigation resolution logic, precedence rules, or the `back_to` override lifecycle.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to discover the navigation decision chain:

| Search target | Pattern | Key results |
|---------------|---------|-------------|
| Back-button component (classic) | `client/signup/navigation-link/index.jsx` | `getPreviousStep()`, `getBackUrl()`, early-return on `backUrl` prop |
| Step wrapper connecting query param | `client/signup/step-wrapper/index.jsx` | `connect()` HOC reading `back_to` from query, nullish coalescing with `ownProps.backUrl` |
| External back-URL overrides | `client/signup/steps/domains/utils.js` | `getExternalBackUrl()`, `backUrlSourceOverrides`, `backUrlExternalSourceStepsOverrides` |
| Declarative stepper tracking hook | `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts` | `canUserGoBack` flag, flow-defined `goBack` override, `history.back()` fallback |
| Stepper step container | `packages/onboarding/src/step-container/index.tsx` | `renderBackButton()`: renders only if `goBack` or `backUrl` is truthy |
| Onboarding flow definition | `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts` | Returns only `{ submit }` — no `goBack`, so stepper falls through to browser history |
| Flow configurations declaring `back_to` | `client/signup/config/flows-pure.js` (lines 410, 446, 478) | Flows `difm`, `website-design-services`, `woocommerce-install` declare `back_to` in `providesDependenciesInQuery` |
| Controller dispatching `back_to` | `client/signup/controller.js` (lines 227–229) | Force-dispatches `back_to` into Redux dependency store for `woocommerce-install` |
| Steps consuming `back_to` as `backUrl` | `client/signup/steps/new-or-existing-site/index.tsx`, `client/signup/steps/difm-site-picker/index.tsx`, `client/signup/steps/site-options/index.tsx`, `client/signup/steps/woocommerce-install/step-store-address/index.tsx` | Destructure `back_to` from `signupDependencies`, pass as `backUrl` prop to `StepWrapper` |
| Signup utilities | `client/signup/utils.js` | `getFilteredSteps()`, `getStepUrl()`, `isFirstStepInFlow()`, `getPreviousStepName()` |

### 0.2.3 Web Search Research Conducted

No external web search was required. The investigation is entirely code-based, per the user's directive to base answers on the code as truth.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The documentation must cover the following modules and their roles in back-navigation:

**Classic Signup System (`/start/...`)**

- **Module:** `client/signup/navigation-link/index.jsx`
  - Public APIs: `getPreviousStep()`, `getBackUrl()`, `handleClick()`, `recordClick()`
  - Current documentation: No dedicated documentation; inline JSDoc and PropTypes only
  - Documentation needed: Full method-level walkthrough with precedence explanation

- **Module:** `client/signup/step-wrapper/index.jsx`
  - Public APIs: `renderBack()`, `renderSkip()`, `renderNext()`, `connect()` HOC
  - Current documentation: None
  - Documentation needed: Explain `connect()` mapStateToProps where `backUrl = ownProps.backUrl ?? backTo`, and `renderBack()` where `allowBackFirstStep` is forced true when `backUrl` is truthy

- **Module:** `client/signup/utils.js`
  - Public APIs: `getFilteredSteps()`, `getStepUrl()`, `isFirstStepInFlow()`, `getPreviousStepName()`
  - Current documentation: Inline comments only
  - Documentation needed: Explain role as fallback destination computer when no override is active

- **Module:** `client/signup/steps/domains/utils.js`
  - Public APIs: `getExternalBackUrl()`, `backUrlSourceOverrides`, `backUrlExternalSourceStepsOverrides`
  - Current documentation: Inline comments only
  - Documentation needed: Document the source-parameter override chain and per-step-section gating

- **Module:** `client/signup/config/flows-pure.js`
  - Key configuration: `providesDependenciesInQuery: ['back_to']` declarations
  - Current documentation: None
  - Documentation needed: Document which flows declare `back_to` as a dependency

- **Module:** `client/signup/controller.js`
  - Key logic: Lines 227–229 — force-dispatch of `back_to` into Redux
  - Current documentation: None
  - Documentation needed: Explain the controller-level override injection

**Declarative Stepper System (`/setup/...`)**

- **Module:** `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts`
  - Public APIs: `useStepNavigationWithTracking()`, `canUserGoBack` computation
  - Current documentation: `client/landing/stepper/README.md` (high-level only)
  - Documentation needed: Detail the precedence: flow-defined `goBack` > browser `history.back()` > no button

- **Module:** `packages/onboarding/src/step-container/index.tsx`
  - Public APIs: `renderBackButton()` — conditional on `goBack || backUrl` truthiness
  - Current documentation: None
  - Documentation needed: Explain back-button visibility conditions

- **Module:** `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts`
  - Key observation: `useStepNavigation` returns only `{ submit }` — no `goBack`
  - Current documentation: `README.md` in same directory (minimal)
  - Documentation needed: Explain that this flow relies entirely on the stepper's fallback back-navigation

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No architecture documentation** for the back-navigation precedence chain across either system
- **No explanation** of how the `back_to` query parameter propagates from URL to Redux to step props to rendered button destination
- **No documentation** of the interaction between `ownProps.backUrl`, `back_to` query, and the computed `getPreviousStep()` target
- **No documentation** of the `allowBackFirstStep` side-effect when `backUrl` is truthy
- **No per-step destination table** for representative flows showing what Back resolves to at each position
- **No diagnostic strategy documentation** for observing back-navigation targets at runtime

These gaps are precisely what the new document will address.


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The new document will follow this structure:

```
blitzy/documentation/
└── wp-calypso_be7e5cc64162.md
    ├── 1. Summary
    ├── 2. Two Navigation Systems Overview
    │   ├── 2.1 Classic Signup System (/start/)
    │   └── 2.2 Declarative Stepper System (/setup/)
    ├── 3. What Decides the Back Destination (Q1)
    │   ├── 3.1 Classic System Precedence Chain
    │   ├── 3.2 Stepper System Precedence Chain
    │   └── 3.3 Precedence Comparison Table
    ├── 4. Input Conflict Resolution (Q2)
    │   ├── 4.1 StepWrapper connect() Resolution
    │   ├── 4.2 NavigationLink.getBackUrl() Early Return
    │   └── 4.3 getPreviousStep() Fallback Path
    ├── 5. The External Override: back_to (Q3)
    │   ├── 5.1 Lifecycle Diagram
    │   ├── 5.2 Flow Configurations Declaring back_to
    │   ├── 5.3 Controller Dispatch
    │   └── 5.4 Step Consumption Pattern
    ├── 6. Precedence Rule and Bypassed Code Path (Q4)
    │   ├── 6.1 The Nullish Coalescing Gate
    │   ├── 6.2 allowBackFirstStep Side-Effect
    │   ├── 6.3 The Bypassed getPreviousStep()
    │   └── 6.4 Domains Step Special Cases
    ├── 7. Per-Step Destination Observation (Q5)
    │   ├── 7.1 Diagnostic Approach
    │   ├── 7.2 Expected Destination Table
    │   └── 7.3 Cleanup Checklist
    └── 8. Source Files Referenced
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the back-navigation precedence chain from `client/signup/step-wrapper/index.jsx` (connect HOC, lines 273–283) and `client/signup/navigation-link/index.jsx` (getBackUrl, lines 78–115)
- Extract the stepper precedence from `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts` (lines 54–58 for `canUserGoBack`, lines 91–165 for priority ordering)
- Extract `back_to` lifecycle from `client/signup/config/flows-pure.js`, `client/signup/controller.js`, and consumer steps
- Generate examples by analyzing the `woocommerce-install` flow as a representative case where `back_to` is declared, and the `onboarding` flow as a case where no `goBack` is defined

**Documentation Standards:**
- Markdown formatting with `#` / `##` / `###` headers
- Mermaid diagrams for the `back_to` lifecycle and precedence chain flowcharts
- Code excerpts using fenced blocks with language annotations and `Source:` citations
- Tables for per-step destination maps and precedence comparison
- Consistent terminology: "classic signup system", "declarative stepper system", "back-target", "override"

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the document:

| Diagram | Type | Purpose |
|---------|------|---------|
| Back-navigation precedence chain (classic) | Flowchart | Shows decision tree from backUrl prop → back_to query → getPreviousStep() |
| Back-navigation precedence chain (stepper) | Flowchart | Shows decision tree from flow-defined goBack → canUserGoBack → no button |
| `back_to` lifecycle | Sequence diagram | Traces back_to from URL through controller, Redux, step props, StepWrapper, NavigationLink |
| Domains step back-URL branching | Flowchart | Shows the extensive if/else chain in the domains step's back-URL computation |


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | `client/signup/navigation-link/index.jsx`, `client/signup/step-wrapper/index.jsx`, `client/signup/utils.js`, `client/signup/steps/domains/utils.js`, `client/signup/config/flows-pure.js`, `client/signup/controller.js`, `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts`, `packages/onboarding/src/step-container/index.tsx`, `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts`, `client/signup/steps/new-or-existing-site/index.tsx`, `client/signup/steps/woocommerce-install/step-store-address/index.tsx` | Complete investigation document answering all five user questions with rationale, code citations, Mermaid diagrams, and per-step destination tables |
| `client/signup/navigation-link/index.jsx` | REFERENCE | — | Used as the primary reference for `getPreviousStep()`, `getBackUrl()`, and the early-return override pattern |
| `client/signup/step-wrapper/index.jsx` | REFERENCE | — | Used as the primary reference for the `connect()` HOC that reads `back_to` and the `renderBack()` method that forces `allowBackFirstStep` |
| `client/signup/utils.js` | REFERENCE | — | Used as reference for `getFilteredSteps()`, `getStepUrl()`, `isFirstStepInFlow()` |
| `client/signup/steps/domains/utils.js` | REFERENCE | — | Used as reference for `getExternalBackUrl()` and override allowlists |
| `client/signup/config/flows-pure.js` | REFERENCE | — | Used as reference for flows declaring `back_to` in `providesDependenciesInQuery` |
| `client/signup/controller.js` | REFERENCE | — | Used as reference for the `back_to` force-dispatch at lines 227–229 |
| `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts` | REFERENCE | — | Used as reference for stepper back-navigation precedence, `canUserGoBack`, and flow-override priority |
| `packages/onboarding/src/step-container/index.tsx` | REFERENCE | — | Used as reference for stepper back-button visibility condition |
| `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts` | REFERENCE | — | Used as reference for the onboarding flow's lack of a `goBack` handler |
| `client/landing/stepper/README.md` | REFERENCE | — | Used as reference for stepper framework architecture guidance |
| `client/signup/navigation-link/test/index.jsx` | REFERENCE | — | Used as reference for expected back-navigation behaviour under test |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/wp-calypso_be7e5cc64162.md
Type: Technical Q&A / Architecture Investigation
Source Code: 11 source files (listed above)
Sections:
    - Summary (purpose and scope of investigation)
    - Two Navigation Systems Overview (classic vs stepper)
    - What Decides the Back Destination (precedence chain)
    - Input Conflict Resolution (who wins when inputs disagree)
    - The External Override: back_to (lifecycle and propagation)
    - Precedence Rule and Bypassed Code Path (the gate and the skip)
    - Per-Step Destination Observation (diagnostic approach and table)
    - Source Files Referenced (comprehensive list)
Diagrams:
    - Flowchart: Classic system back-navigation precedence
    - Flowchart: Stepper system back-navigation precedence
    - Sequence diagram: back_to lifecycle from URL to rendered destination
    - Flowchart: Domains step back-URL branching chain
Key Citations:
    - client/signup/step-wrapper/index.jsx:273-283
    - client/signup/navigation-link/index.jsx:78-115
    - client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts:54-58, 91-165
    - client/signup/steps/domains/utils.js:6-31
    - client/signup/controller.js:227-229
```

### 0.5.3 Cross-Documentation Dependencies

- The new document is self-contained. It does not require navigation updates or table-of-contents changes to any existing documentation.
- The `blitzy/documentation/` directory must be created if it does not already exist.
- No documentation configuration files need updating (no generator config to modify).


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

No additional documentation tools or packages are required for this task. The output is a standalone Markdown file authored directly without a documentation generator. The only tooling dependency is the Mermaid diagram syntax embedded in the Markdown, which is natively rendered by GitHub.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| npm | mermaid (render-time) | N/A (GitHub-native) | Diagrams embedded in markdown are rendered by GitHub's built-in Mermaid support |

### 0.6.2 Runtime and Framework Context

The following runtime and framework versions are documented for context, as they define the environment in which the investigated code operates:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| nvm | Node.js | 22.9.0 | Runtime specified in `.nvmrc` and `package.json` engines |
| npm | Yarn | 4.x | Workspace-aware package manager specified in `package.json` engines |
| npm | react | (workspace) | UI framework for all signup/stepper components |
| npm | react-redux | (workspace) | State management connector used by `StepWrapper` and `NavigationLink` |
| npm | @automattic/onboarding | (workspace) | Shared onboarding package containing `StepContainer`, `StepNavigationLink`, `ActionButtons` |
| npm | @automattic/data-stores | (workspace) | Data stores for onboarding (`ONBOARD_STORE`), stepper internals (`STEPPER_INTERNAL_STORE`) |
| npm | @automattic/components | (workspace) | Shared `Button`, `Gridicon` components used by navigation controls |
| npm | i18n-calypso | (workspace) | Localisation framework used across all signup/stepper modules |
| npm | @wordpress/data | (workspace) | WordPress data layer used by declarative stepper hooks |
| npm | @wordpress/url | (workspace) | URL manipulation utilities (`addQueryArgs`, `getQueryArg`, `removeQueryArgs`) |
| npm | lodash | (workspace) | Utility functions (`get`, `filter`, `find`, `sortBy`) used in signup utilities |
| npm | valid-url | (workspace) | URL validation used in `getExternalBackUrl()` |

### 0.6.3 Documentation Reference Updates

No link updates are required. The new document is the only artifact being created and it contains no outbound links to other documentation files that would need transformation.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

- **User questions addressed:** 5/5 (100%)
  - Q1 (What decides the destination): Covered via precedence chain analysis across both systems
  - Q2 (Which input wins): Covered via conflict resolution section with priority ranking
  - Q3 (Where does the override come from): Covered via `back_to` lifecycle trace
  - Q4 (What precedence rule, what bypassed path): Covered via gate analysis and `getPreviousStep()` walkthrough
  - Q5 (Observe computed destination per step): Covered via diagnostic approach and per-step table
- **Navigation systems documented:** 2/2 (100%) — Classic signup system and Declarative stepper system
- **Source files cited:** 11 primary source files spanning the complete decision chain
- **Diagrams planned:** 4 Mermaid diagrams covering precedence flows, lifecycle, and domains special case

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim in the document references a specific file path and, where appropriate, line numbers
- The precedence chain is documented exhaustively — no intermediate step is omitted
- Both navigation systems are covered, not just the one the user may be experiencing
- The `back_to` lifecycle is traced end-to-end from URL to rendered button destination
- A per-step destination table is provided for at least one representative flow

**Accuracy validation:**
- All code excerpts are taken directly from the repository source at the analysed commit (`wp-calypso_be7e5cc64162`)
- The precedence ordering is verified against the actual control flow in the source (nullish coalescing, early returns, conditional spreads)
- The `canUserGoBack` conditions are verified against the stepper tracking hook source
- No assumptions are made — every statement is grounded in code

**Clarity standards:**
- Technical accuracy with accessible explanation of each decision gate
- Progressive disclosure: summary first, then detailed walkthroughs
- Consistent terminology: "classic signup system", "declarative stepper system", "back-target", "override", "precedence chain"
- Each section begins with a direct answer to the user's question before providing supporting detail

**Maintainability:**
- Source citations formatted as `Source: <file_path>:<line_range>` for traceability
- Mermaid diagrams embedded for easy future updates
- Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 (two precedence flowcharts, one sequence diagram, one branching flowchart)
- **Minimum tables:** 3 (precedence comparison, flows declaring `back_to`, per-step destination map)
- **Code example testing:** Code excerpts are read-only citations; no executable examples required
- **Diagnostic approach:** A temporary observation script pattern is described in the document; the actual script is run and cleaned up during the documentation task


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/wp-calypso_be7e5cc64162.md` — the complete investigation document

**Source files analysed for documentation content (REFERENCE only, no modifications):**
- `client/signup/navigation-link/index.jsx` — back-navigation component with `getPreviousStep()` and `getBackUrl()`
- `client/signup/step-wrapper/index.jsx` — step wrapper with `connect()` HOC reading `back_to`
- `client/signup/utils.js` — signup utilities: `getFilteredSteps()`, `getStepUrl()`, `isFirstStepInFlow()`
- `client/signup/steps/domains/utils.js` — `getExternalBackUrl()` and override allowlists
- `client/signup/steps/domains/index.jsx` — domains step back-URL computation chain
- `client/signup/config/flows-pure.js` — flow definitions declaring `back_to`
- `client/signup/controller.js` — controller-level `back_to` dispatch
- `client/signup/steps/new-or-existing-site/index.tsx` — step consuming `back_to` as `backUrl`
- `client/signup/steps/woocommerce-install/step-store-address/index.tsx` — step consuming `back_to` as `backUrl`
- `client/signup/steps/difm-site-picker/index.tsx` — step consuming `back_to` as `backUrl`
- `client/signup/steps/site-options/index.tsx` — step consuming `back_to` as `backUrl`
- `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts` — stepper navigation tracking hook
- `packages/onboarding/src/step-container/index.tsx` — stepper step container
- `packages/onboarding/src/step-navigation-link/index.tsx` — stepper navigation link component
- `packages/onboarding/src/step-container-v2/components/buttons/BackButton/BackButton.tsx` — V2 back button
- `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts` — onboarding flow (no `goBack`)
- `client/landing/stepper/declarative-flow/internals/index.tsx` — stepper FlowRenderer
- `client/landing/stepper/README.md` — stepper framework documentation
- `client/signup/navigation-link/test/index.jsx` — test suite validating back-navigation behaviour

**Temporary diagnostic artifacts (created and cleaned up):**
- Any temporary script used to log back-navigation destinations per step position

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No changes to any file in the repository. The user explicitly requires the repository to remain unchanged.
- **Bug fixes** — The document explains the behaviour but does not propose or implement fixes.
- **Test modifications** — No changes to test files.
- **Feature additions or refactoring** — Not requested.
- **Deployment configuration changes** — Not applicable.
- **Documentation for unrelated flows** — Flows not relevant to back-navigation (e.g., checkout, import, site-setup design) are mentioned only when they provide comparative context.
- **A8C-for-Agencies signup flow** — This is a separate multi-step form system with its own `goBack` callbacks; it does not use the `back_to` override mechanism and is out of scope unless the user clarifies otherwise.
- **StepContainerV2 back button** — The V2 system uses analytics decoration but follows a simpler pattern (direct `goBack` callback) without the override chain. It is mentioned for completeness but not deeply investigated.


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — standalone Markdown; no build step required
- **Documentation preview command:** Any Markdown previewer or `npx grip blitzy/documentation/wp-calypso_be7e5cc64162.md` for GitHub-flavoured preview
- **Diagram generation command:** N/A — Mermaid diagrams are embedded in the Markdown and rendered by GitHub natively
- **Documentation deployment command:** N/A — the file is committed to the repository and rendered by GitHub
- **Default format:** GitHub-Flavoured Markdown with Mermaid diagram blocks
- **Citation requirement:** Every technical claim must reference a specific source file path. Line numbers are included where the claim depends on a specific code construct.
- **Style guide:** Follow the conventions in the existing `docs/` directory — plain markdown, no generator, `#`/`##`/`###` heading hierarchy
- **Documentation validation:** Visual review of Mermaid diagram rendering; link integrity is trivial (no outbound doc links)

### 0.9.2 Temporary Script Policy

Per the user's instruction:

- Temporary scripts may be used for observation (e.g., logging the computed back-navigation destination for each step in a flow)
- Such scripts must not modify any existing repository file
- All temporary artifacts must be removed after observation is complete
- The observation results are captured in the documentation file; the script itself is not committed


## 0.10 Rules for Documentation


The following rules are derived from the user's explicit instructions and the project-level implementation rule "SWE-AtlasQnA-Repo":

- **Create a new markdown document named `wp-calypso_be7e5cc64162.md`** — matching the source branch name
- **Place the document in the `blitzy/documentation` directory** in the destination repo
- **Provide thinking / rationale behind the answers** — every conclusion must be explained, not merely stated
- **Do not make assumptions; base answers on the code as the truth** — every claim must cite a specific file and, where applicable, line numbers
- **Do not modify any existing files in the source repository** — the investigation is read-only; temporary scripts are allowed but must be cleaned up
- **Temporary scripts used for observation must be cleaned up afterward** — no artifacts remain after the task completes
- **Use Mermaid diagrams** for all flowcharts and sequence diagrams to maintain readability and editability
- **Include per-step destination tables** to confirm the back-navigation pattern the user is observing
- **Cover both navigation systems** (classic signup and declarative stepper) so the document is comprehensive regardless of which flow the user is experiencing
- **Document the `back_to` override lifecycle end-to-end** from URL query parameter through Redux dispatch to rendered button destination


## 0.11 References


### 0.11.1 Source Files and Folders Searched

The following files and folders were examined during context gathering to derive the conclusions in this Agent Action Plan:

**Primary navigation files (read in full):**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `client/signup/navigation-link/index.jsx` | Core back-navigation component: `getPreviousStep()`, `getBackUrl()`, early-return on `backUrl` prop, `handleClick()` dispatch |
| `client/signup/step-wrapper/index.jsx` | Step wrapper HOC: `connect()` mapStateToProps reading `back_to` from query, nullish coalescing with `ownProps.backUrl`, `renderBack()` forcing `allowBackFirstStep` |
| `client/signup/utils.js` | Signup flow utilities: `getFilteredSteps()`, `getStepUrl()`, `isFirstStepInFlow()`, `getPreviousStepName()`, `getFlowSteps()` |
| `client/signup/steps/domains/utils.js` | External back-URL overrides: `getExternalBackUrl()`, `backUrlSourceOverrides`, `backUrlExternalSourceStepsOverrides` |
| `client/signup/steps/domains/index.jsx` (lines 1350–1450) | Domains step back-URL computation: flow-specific branching, source-parameter handling, playground ID handling |
| `client/signup/config/flows-pure.js` (lines 395–500) | Flow definitions declaring `back_to` in `providesDependenciesInQuery` (difm, website-design-services, woocommerce-install) |
| `client/signup/controller.js` (lines 215–260) | Controller pipeline: force-dispatch of `back_to` into Redux dependency store for woocommerce-install |
| `client/signup/steps/new-or-existing-site/index.tsx` | Step consuming `back_to` from `signupDependencies` as `backUrl` prop to `StepWrapper` |
| `client/signup/steps/woocommerce-install/step-store-address/index.tsx` (lines 55–100) | Step consuming `back_to` from `signupDependencies` with path validation fallback |
| `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts` | Stepper tracking hook: `canUserGoBack` computation, flow-defined `goBack` priority over `history.back()`, conditional spread pattern |
| `packages/onboarding/src/step-container/index.tsx` | Stepper step container: `renderBackButton()` visibility gated on `goBack \|\| backUrl` truthiness |
| `packages/onboarding/src/step-navigation-link/index.tsx` | Stepper navigation link: `StepNavigationLink` component rendering back/forward controls |
| `packages/onboarding/src/step-container-v2/components/buttons/BackButton/BackButton.tsx` | V2 back button with analytics decoration |
| `client/landing/stepper/declarative-flow/flows/onboarding/onboarding.ts` | Onboarding flow: returns only `{ submit }` — no `goBack`, triggering stepper fallback |
| `client/landing/stepper/declarative-flow/internals/index.tsx` | FlowRenderer: assembles step routes and passes tracked navigation to step components |
| `client/landing/stepper/README.md` | Stepper framework documentation: `useStepNavigation`, non-linearity, reusability contract |
| `client/landing/stepper/declarative-flow/flows/onboarding/README.md` | Onboarding flow documentation (minimal) |

**Supplementary files (searched or summarised):**

| File / Folder Path | Purpose in Investigation |
|---------------------|------------------------|
| `client/signup/navigation-link/test/index.jsx` | Test suite verifying back-navigation URL computation, `getPreviousStep()` edge cases, and `backUrl` override precedence |
| `client/landing/stepper/declarative-flow/flows/entrepreneur-flow/entrepreneur-flow.ts` | Comparative reference: flow that defines `goBack` only for `trial-acknowledge` step |
| `client/landing/stepper/declarative-flow/flows/domain-upsell/domain-upsell.ts` | Comparative reference: flow with step-specific `goBack` that exits to return URL |
| `client/landing/stepper/declarative-flow/flows/update-options/update-options.ts` | Comparative reference: flow with async `goBack` redirecting to launchpad |
| `client/landing/stepper/declarative-flow/internals/types.ts` | Type definitions for `Flow`, `FlowV2`, navigation contracts |
| `client/landing/stepper/utils/path.ts` | Route/login URL utility for the stepper |
| `client/signup/steps/difm-site-picker/index.tsx` | Step consuming `back_to` as `backUrl` |
| `client/signup/steps/site-options/index.tsx` | Step consuming `back_to` as `backUrl` |
| `client/state/signup/steps/` | Redux state composition for signup step data |
| `client/lib/signup/step-actions/` | Imperative action layer for signup step orchestration |
| Root `package.json` | Runtime versions: Node ^22.9.0, Yarn ^4.0.0 |
| `.nvmrc` | Node version: 22.9.0 |
| `docs/` directory listing | Existing documentation survey |

**Tech spec sections retrieved:**

| Section | Purpose |
|---------|---------|
| 4.7 SIGNUP AND ONBOARDING FLOW | Architecture context for signup controller pipeline, flow/step architecture, state persistence |
| 4.3 AUTHENTICATION WORKFLOWS | Context for authentication handoffs during multi-step flows |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma URLs are referenced.


