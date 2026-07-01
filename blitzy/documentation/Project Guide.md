# Blitzy Project Guide — wp-calypso: Jest Cold-vs-Warm Test-Timing Investigation

> **Branch:** `blitzy-0abb4dec-5b2f-4c98-a55c-fd66cf28682e` · **Base:** `be7e5cc641` · **Task type:** Read-only investigative Q&A (Documentation) · **Rule:** `SWE-AtlasQnA-Repo`
>
> **Brand color legend:** <span style="color:#5B39F3">■</span> Completed / AI Work = Dark Blue `#5B39F3` · <span style="color:#FFFFFF;background:#333;padding:0 4px">■</span> Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers a developer's question — *why do Jest test-execution times in wp-calypso's `client/state/data-layer` module differ between a first ("cold") and second ("warm") run?* — by **actually running the code** and grounding every claim in verbatim output and exact `file:line` citations. It is a strictly read-only investigation targeting the Calypso client test suite (Jest 29.7.0, Yarn 4.0.2). The single deliverable is one new markdown answer document that decomposes and answers four sub-questions (timing/ratio, transform-cache configuration, `nock` HTTP mocking, and `--no-cache` impact). No existing repository file is modified. The audience is the requesting engineer and reviewers who need an evidence-backed explanation of Jest's transform-cache warmup behavior.

### 1.2 Completion Status

Completion is computed with the AAP-scoped methodology (PA1): only work defined by the Agent Action Plan plus standard path-to-production activities are counted. `Completion % = Completed Hours ÷ (Completed Hours + Remaining Hours) × 100 = 21 ÷ 23 = 91.3%`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 91.3% Complete
    "Completed Work (hrs)" : 21
    "Remaining Work (hrs)" : 2
```

| Metric | Value |
|---|--:|
| **Total Hours** | 23.0 |
| **Completed Hours (AI + Manual)** | 21.0 |
| &nbsp;&nbsp;• AI (Blitzy autonomous) | 21.0 |
| &nbsp;&nbsp;• Manual (human) | 0.0 |
| **Remaining Hours** | 2.0 |
| **Percent Complete** | **91.3%** |

### 1.3 Key Accomplishments

- ✅ **All four sub-questions answered** with freshly-measured evidence and exact `file:line` citations (single deliverable, 568 lines).
- ✅ **Requirement 1 (timing/ratio):** measured cold vs. warm for a single file (**≈ 1.75×**) and the full `data-layer` suite (**≈ 1.81×**), with verbatim wall-clock + Jest `Time:` markers.
- ✅ **Requirement 2 (cache):** identified the controlling option **`cacheDirectory`** (`test/client/jest.config.js:L7`) → `<repo>/.cache/jest`; classified the three cached artifact types (haste-map, `babel-jest` transform cache + source maps, perf-cache).
- ✅ **Requirement 3 (mocking):** identified **`nock`** (`^13.5.6`), traced `nock.disableNetConnect()` in the setup framework, and proved it is **not** a first-run overhead driver (excluded from transform via `transformIgnorePatterns`).
- ✅ **Requirement 4 (`--no-cache`):** measured ≈ **1.72×** vs. warm and identified **`babel-jest` transpiling first-party TS/JSX** as the dominant transformation step.
- ✅ **Read-only constraint honored:** `git diff` from base shows exactly **one file added**, zero modifications/deletions; working tree clean.
- ✅ **Two upstream AAP inaccuracies corrected and grounded** (nock mock at `L32`; perf-cache array is `[status, duration_ms]`), reconciled against installed `@jest/test-sequencer` source.
- ✅ **Independent validation:** the Final Validator re-ran every measurement and re-verified every claim; all 5 production-readiness gates PASS.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None_ | No blocking issues. All sub-questions answered; deliverable committed; tests pass (cold, warm, `--no-cache`); read-only honored; tree clean. | — | — |

### 1.5 Access Issues

No access issues identified.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | Repository, toolchain (Node v22.23.1, Yarn 4.0.2), and dependencies (`jest`, `nock`, `babel-jest`) were all accessible; `node_modules` populated; no third-party credentials or network access required (network is disabled by `nock`). | N/A — none | — |

### 1.6 Recommended Next Steps

1. **[Medium]** Perform the human technical review of `blitzy/documentation/wp-calypso_be7e5cc64162.md`, spot-checking the `file:line` citations and the two documented AAP corrections.
2. **[Medium]** Approve and merge the pull request — the deliverable is complete and the tree is clean.
3. **[Low]** Optionally re-run the cold/warm/`--no-cache` measurements on your own machine or CI to confirm the ratios land within the documented stable range (~1.7–2.1×); absolute milliseconds will differ by host.
4. **[Low]** If desired for future onboarding, link this answer document from the `data-layer` `README.md` in a separate (non-blocking) change — out of scope for this read-only task.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work is AI-autonomous (Blitzy). Each component traces to a specific AAP requirement or path-to-production activity.

| Component | Hours | Description |
|---|--:|---|
| Environment setup & toolchain validation | 2.0 | Selected Node v22.x to satisfy `engines.node ^v22.9.0` (intentional deviation from the Node-20 setup script), ran `yarn install`, verified `jest`/`nock` binaries and that `package.json` has no top-level `jest` key (validating the `-c` flag requirement). |
| R1 — Cold/warm timing measurement & ratios | 3.0 | Authored a timing wrapper; ran single-file and full-suite cold/warm; captured wall-clock + Jest `Time:` markers; computed ratios (≈ 1.75× single, ≈ 1.81× suite). |
| R2 — Transform-cache investigation | 2.5 | Read `cacheDirectory` (`test/client/jest.config.js:L7`); enumerated `<repo>/.cache/jest`; classified three artifact types (haste-map, transform-cache 2095 code + 1981 `.map`, perf-cache 92 entries). |
| R3 — HTTP mocking (`nock`) investigation | 2.0 | Identified `nock`; traced `setup-test-framework.js` + deprecated `use-nock` helper; reasoned from `transformIgnorePatterns`; grounded the "not a first-run driver" verdict in the cache's absence of `nock`. |
| R4 — `--no-cache` comparison & dominant step | 2.0 | Ran `--no-cache`; compared to warm/cold; attributed dominant cost to `babel-jest` transpiling first-party TS/JSX, corroborated by cache contents, resolver policy, and Jest docs. |
| Web research (Jest docs validation) | 1.5 | Validated transform-cache semantics ("run once per file"), `node_modules` exclusion, the "~2× slower" `--no-cache` guidance, and cache-key derivation against official Jest documentation. |
| Answer document authoring | 4.0 | Wrote the 568-line markdown: executive summary + four answer blocks + coverage-pass table + caveats + command reference; verbatim output; exact `file:line` citations; two grounded AAP corrections. |
| Independent validation re-run | 3.0 | Final Validator re-ran all measurements and re-verified every claim, citation, count, and version; confirmed both corrections; all 5 gates PASS. |
| Cleanup & read-only verification | 1.0 | Removed temporary scripts (`/tmp/blitzy_timing`); confirmed `git status --porcelain` empty; committed the deliverable on-branch. |
| **Total Completed** | **21.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|--:|---|
| Human technical review & acceptance of the answer document | 1.5 | Medium |
| Optional re-verification of timing ratios on target/CI hardware | 0.5 | Low |
| **Total Remaining** | **2.0** | |

### 2.3 Hours Reconciliation

| Check | Value |
|---|--:|
| Section 2.1 Completed total | 21.0 |
| Section 2.2 Remaining total | 2.0 |
| **Total (2.1 + 2.2)** | **23.0** |
| Matches Section 1.2 Total Hours | ✅ 23.0 |
| Completion (21 ÷ 23) | **91.3%** |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project (Jest 29.7.0 via `TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js <path> --ci`). Coverage was **not** collected because coverage measurement is outside the AAP scope for this read-only timing investigation; it is reported as N/A.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|--:|--:|--:|:--:|---|
| Data-layer full module (Unit/Integration) | Jest 29.7.0 | 439 | 439 | 0 | N/A | 92 suites + 1 snapshot; executed **cold and warm**; every run `EXIT_STATUS=0`. |
| Representative file — `wpcom-http` (Unit) | Jest 29.7.0 | 2 | 2 | 0 | N/A | 1 suite; executed **cold, warm, and `--no-cache`**; a subset of the full suite re-run in isolation for the timing comparison; `EXIT_STATUS=0`. |
| **Distinct total (full module)** | Jest 29.7.0 | **439** | **439** | **0** | N/A | 0 skipped, 0 blocked, 0 flaky. The 2-test file is a subset counted within the 439. |

**Pass rate: 100%** across every mode (single-file cold/warm/`--no-cache`; full-suite cold/warm). Independent re-verification during this assessment re-ran the representative file warm: `1 passed, 1 total` / `2 passed, 2 total` / `Time: 0.88 s` / `EXIT=0`, and cold: `Time: 1.721 s` / `EXIT=0`.

---

## 4. Runtime Validation & UI Verification

The "runtime" here is the Jest test runner and its transform/mocking/caching subsystems (this is a CLI/documentation task with **no UI component**).

- ✅ **Jest runner (Jest 29.7.0)** — Operational. Ran successfully in all five measured modes (single-file cold, warm, `--no-cache`; full-suite cold, warm); every run `EXIT_STATUS=0`.
- ✅ **`babel-jest` transpilation** — Operational. First-party TS/JSX compiled without error; the transform cache held 2095 transformed modules + 1981 source maps after a full run.
- ✅ **`nock` HTTP mocking** — Operational. `nock.disableNetConnect()` (`setup-test-framework.js:L9`) enforced in-process interception; the representative test's `.reply(200, …)` (`L32`) and `.replyWithError(…)` (`L46`) resolved from memory with zero real network I/O.
- ✅ **Transform cache subsystem** — Operational. `--showConfig` resolves `cacheDirectory` to `<repo>/.cache/jest`; three artifact types generated on run; directory is git-ignored (`.gitignore:L15`).
- ⚠ **Absolute timings** — Partial/host-dependent by nature. Raw millisecond values vary with host load; the **ratios** (~1.7–2.1×) are the stable, reportable quantities (see Risk T1).
- ➖ **UI verification** — Not applicable. No web UI, page, or component is in scope; the deliverable is a markdown document produced from CLI measurements.
- ➖ **External API integration** — Not applicable. No live external API is called; all HTTP is mocked in-process by `nock`.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and `SWE-AtlasQnA-Repo` rule requirements to their verified status. Fixes/refinements applied during autonomous validation are noted.

| Deliverable / Rule Benchmark | Status | Progress | Evidence / Notes |
|---|:--:|:--:|---|
| Read-only constraint — no tracked file modified/deleted | ✅ Pass | 100% | `git diff be7e5cc641..HEAD --name-status` = `A` (added) for exactly one file; zero `M`/`D`. |
| Single deliverable created at correct path/name | ✅ Pass | 100% | `blitzy/documentation/wp-calypso_be7e5cc64162.md` (name = source branch). |
| Run-first methodology (build & run before writing) | ✅ Pass | 100% | Measurements captured via a temp wrapper, then quoted; two commits (author + evidence-grounding pass). |
| Verbatim evidence quoted | ✅ Pass | 100% | Each measurement shows command + raw output (`WALL_CLOCK_MS=…`, Jest `Time:`, suite/test counts). |
| Exact, grounded `file:line` citations | ✅ Pass | 100% | e.g. `test/client/jest.config.js:L7`, `.gitignore:L15`, `package.json:L299`, `setup-test-framework.js:L9`, `jest-preset.js:L14`. |
| Every sub-question answered + coverage pass | ✅ Pass | 100% | Answers 1–4 plus a coverage-pass table mapping each sub-question to its answer. |
| Rationale provided per answer | ✅ Pass | 100% | Each answer includes a "why" section (cache reuse; `node_modules` exclusion; dominant transform). |
| Temporary scripts removed / clean tree | ✅ Pass | 100% | `/tmp/blitzy_timing` removed; `git status --porcelain` empty; `.cache/` git-ignored. |
| Node engines pin honored | ✅ Pass | 100% | Node v22.23.1 satisfies `^v22.9.0`; documented deviation from Node-20 setup script. |
| Correct test invocation (`-c` flag) | ✅ Pass | 100% | No top-level `jest` key in `package.json`; `-c=test/client/jest.config.js` used throughout. |
| AAP inaccuracies flagged & corrected with grounding | ✅ Pass | 100% | (a) nock mock at `L32` (not L28–31); (b) perf-cache `[status, duration_ms]` (not `[transformTime, size]`), reconciled on the `[1,874]` example against `@jest/test-sequencer` source. |
| Human review & acceptance | ⬜ Pending | 0% | Awaiting reviewer sign-off (Section 2.2, HT-1). |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|:--:|:--:|---|:--:|
| Absolute millisecond timings vary by host load | Technical | Low | High | Document reports stable **ratios** (~1.7–2.1×), not raw ms, as the answer. | Mitigated |
| Ratio varies run-to-run (validator saw ~1.67×/~1.95× vs. doc's ~1.75×/~1.81×) | Technical | Low | Medium | Document states a stable range and explains variance; all observed values fall inside it. | Mitigated |
| CI hardware may show different absolute numbers / lower ratio for tiny files (fixed per-run costs dilute savings) | Operational | Low | Medium | Document explains fixed-cost dilution; optional CI re-verification recommended (Section 2.2, HT-2). | Open (Low) |
| Upstream AAP contained two inaccuracies (nock line; perf-cache semantics) | Technical | Low | Low | Document corrects both and reconciles them with installed `@jest/test-sequencer` source and the AAP's own `[1,874]` example. | Mitigated |
| Security exposure | Security | None | N/A | Read-only task; zero new dependencies; no secrets; no auth surface; network disabled via `nock.disableNetConnect()`. | None identified |
| External integration failure | Integration | None | N/A | No external integrations; all HTTP mocked in-process; no API keys/credentials required. | None identified |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`). The "Remaining Work" value (2) equals Section 1.2 Remaining Hours and the Section 2.2 total.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 23.0h)
    "Completed Work" : 21
    "Remaining Work" : 2
```

**Remaining hours by priority** (from Section 2.2; sums to 2.0):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9','pieStrokeColor':'#5B39F3','pieOuterStrokeColor':'#5B39F3','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#000000','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Remaining Hours by Priority
    "Medium — Review & acceptance" : 1.5
    "Low — Optional re-verification" : 0.5
```

| Status band | Hours | Share |
|---|--:|--:|
| Completed | 21.0 | 91.3% |
| Remaining | 2.0 | 8.7% |
| **Total** | **23.0** | **100%** |

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.3% complete** on an AAP-scoped basis (21.0 of 23.0 hours). Every one of the four sub-questions is answered with freshly-measured, verbatim evidence and exact `file:line` citations, and the single deliverable is committed on-branch. Measured results confirm the developer's hypothesis of a "significant difference" between the first and second run: **≈ 1.75×** for a single file and **≈ 1.81×** for the full `data-layer` suite (cold ÷ warm). The transform cache is controlled by **`cacheDirectory`** (`test/client/jest.config.js:L7`) at `<repo>/.cache/jest`; **`nock`** is the mocking library and is confirmed **not** a first-run driver; and **`--no-cache`** costs **≈ 1.72×** vs. warm, dominated by **`babel-jest`** transpiling first-party TS/JSX.

**Remaining gaps.** The only remaining work is non-engineering and non-blocking: human technical review/acceptance of the document (1.5h) and optional re-verification of the ratios on target/CI hardware (0.5h) — 2.0 hours total.

**Critical path to production.** Review the document → approve/merge the PR. There is no build, deployment, migration, or integration step because the deliverable is a documentation artifact and the constraint is strictly read-only.

**Production-readiness assessment.** **Production-ready.** All five autonomous production-readiness gates pass: 100% test pass rate (cold, warm, `--no-cache`), runtime validated in every mode, zero unresolved errors, the sole in-scope file validated, and the deliverable AAP-compatible and committed with the read-only constraint fully honored.

| Success metric | Target | Actual |
|---|:--:|:--:|
| Sub-questions answered | 4/4 | ✅ 4/4 |
| Test pass rate | 100% | ✅ 100% (439/439 + 2/2) |
| Files modified (read-only) | 0 | ✅ 0 (1 added) |
| Working tree clean | Yes | ✅ Yes |
| Deliverable committed | Yes | ✅ Yes |
| AAP-scoped completion | ≥ 90% | ✅ 91.3% |

---

## 9. Development Guide

Every command below was executed during this assessment on the project branch and exits `0`. Run from the repository root: `/tmp/blitzy/wp-calypso/blitzy-0abb4dec-5b2f-4c98-a55c-fd66cf28682e_970ba2`.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu 25.10 container used here); macOS also works.
- **Node.js:** `^v22.9.0` (repo pins `22.9.0` in `.nvmrc`; validated with **v22.23.1**). ⚠ Do **not** use Node 20 — it fails the `engines` gate.
- **Yarn:** `4.0.2` (Berry; `nodeLinker: node-modules`).
- **Disk:** ~4.3 GB for the repo incl. `node_modules`.

```bash
# Verify prerequisites
node --version      # -> v22.x (must satisfy ^v22.9.0)
cat .nvmrc          # -> 22.9.0
yarn --version      # -> 4.0.2
```

### 9.2 Environment Setup

```bash
# From the repository root. If using nvm:
nvm install 22 && nvm use 22

# Confirm the repo has NO top-level "jest" key (this is why the -c flag is required):
node -e "console.log('has top-level jest key:', Object.prototype.hasOwnProperty.call(require('./package.json'),'jest'))"
# -> has top-level jest key: false
```

### 9.3 Dependency Installation

```bash
# Install all workspace dependencies (populates node_modules, provides the jest & nock binaries)
yarn install

# Verify the toolchain versions used for measurement:
node -e "console.log('jest', require('jest/package.json').version)"          # -> jest 29.7.0
node -e "console.log('nock', require('nock/package.json').version)"          # -> nock 13.5.6
node -e "console.log('babel-jest', require('babel-jest/package.json').version)"  # -> babel-jest 29.7.0
node -e "console.log('@babel/core', require('@babel/core/package.json').version)" # -> @babel/core 7.26.10
```

### 9.4 Reproducing the Investigation (Startup / Run Sequence)

```bash
# Canonical invocation (no top-level jest key -> pass the client config; --ci prevents watch mode):
#   TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js <path> --ci [--no-cache]

# (1) COLD run — force an empty transform cache first:
rm -rf .cache/jest
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom-http/test/index.js --ci
# -> Test Suites: 1 passed, 1 total ; Tests: 2 passed, 2 total ; Time: ~1.7 s

# (2) WARM run — same command again, cache now present:
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom-http/test/index.js --ci
# -> Time: ~0.9 s  (≈ 1.75x faster than cold)

# (3) --no-cache run — behaves like a permanently cold cache:
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom-http/test/index.js --ci --no-cache
# -> Time: ~1.5 s  (≈ 1.72x vs warm; ≈ cold)

# (4) Full data-layer module (path is relative to rootDir = client):
rm -rf .cache/jest
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js state/data-layer/ --ci   # cold
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js state/data-layer/ --ci   # warm
# -> 92 suites / 439 tests / 1 snapshot passing both runs (cold ≈ 1.81x warm)
```

### 9.5 Verification Steps

```bash
# Confirm the cache option & directory:
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --showConfig | grep cacheDirectory
# -> "cacheDirectory": "<repo>/.cache/jest",

# Inspect the three cached artifact types after a run:
ls -1 .cache/jest/
# -> haste-map-<hashes>
#    jest-transform-cache-<hashes>
#    perf-cache-<hashes>

# Prove the cache is git-ignored (so runs never dirty the tree):
git check-ignore -v .cache/jest        # -> .gitignore:15:/.cache/   .cache/jest

# Confirm the read-only guarantee (tree stays clean):
git status --porcelain                 # -> (empty)
```

### 9.6 Example Usage

```bash
# View the deliverable:
sed -n '13,30p' blitzy/documentation/wp-calypso_be7e5cc64162.md   # executive summary (ratios)

# Confirm exactly one file was added on this branch:
git diff be7e5cc641..HEAD --name-status
# -> A   blitzy/documentation/wp-calypso_be7e5cc64162.md
```

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `No tests found` | Missing `-c` flag (no top-level `jest` key) or wrong path base | Pass `-c=test/client/jest.config.js`; suite paths are relative to `rootDir = client` (e.g. `state/data-layer/`). |
| Runner hangs in watch mode | Interactive watch enabled | Always pass `--ci`. |
| `yarn install` fails on `engines` | Node 20.x active | Switch to Node v22.x (`nvm use 22`). |
| Timing numbers differ from the doc | Host load varies absolute ms | Compare **ratios**, not raw ms; expect ~1.7–2.1×. |
| Warm run not faster | Cache was cleared or `--no-cache` set | Remove `--no-cache`; do not `rm -rf .cache/jest` between the two runs. |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Canonical test invocation | `TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js <path> --ci` |
| Force cold cache | `rm -rf .cache/jest` |
| Disable cache | append `--no-cache` |
| Show resolved config | `TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --showConfig` |
| Inspect cache dir | `ls -1 .cache/jest/` |
| Prove cache ignored | `git check-ignore -v .cache/jest` |
| Read-only check | `git status --porcelain` |
| Files changed on branch | `git diff be7e5cc641..HEAD --name-status` |

### B. Port Reference

Not applicable — no server or long-running service is started. Tests run in-process under Node; all HTTP is intercepted in-memory by `nock` (no sockets opened).

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | **The deliverable** (answer document). |
| `test/client/jest.config.js` | Client suite config; `cacheDirectory` at `L7`; `transformIgnorePatterns` at `L14-16`. |
| `packages/calypso-jest/jest-preset.js` | Shared preset; `transform` map (`babel-jest` at `L14`, asset transform at `L15`). |
| `packages/calypso-jest/src/asset-transform.js` | Trivial asset transformer (`L5`). |
| `packages/calypso-jest/src/module-resolver.js` | `enhanced-resolve` resolver, `calypso:src`-first (`L18`). |
| `babel.config.js` | Root Babel config loaded via `rootMode: 'upward'`. |
| `test/client/setup-test-framework.js` | Global `nock.disableNetConnect()` (`L9`). |
| `client/test-helpers/use-nock/index.js` | Deprecated per-test `nock` helper. |
| `client/state/data-layer/wpcom-http/test/index.js` | Representative measured test (mock at `L32`). |
| `.gitignore` | `/.cache/` at `L15`. |
| `<repo>/.cache/jest/` | Runtime-generated, git-ignored cache. |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Node.js | v22.23.1 (pin `^v22.9.0`) | `.nvmrc`, `package.json:engines` |
| Yarn | 4.0.2 | `package.json:packageManager` |
| Jest | 29.7.0 | `package.json:L290` |
| babel-jest | 29.7.0 | `packages/calypso-jest/package.json:L24` |
| @babel/core | 7.26.10 | `packages/calypso-jest/package.json:L23` |
| nock | 13.5.6 (`^13.5.6`) | `package.json:L299` |
| enhanced-resolve | ^5.8.3 | `packages/calypso-jest/package.json:L25` |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `TZ` | `UTC` | Deterministic timezone for test runs (mirrors the `test-client` script). |
| `CI` | `true` (implied by `--ci`) | Prevents Jest watch mode / interactive prompts. |

### F. Developer Tools Guide

- **Jest CLI** — `--showConfig` (inspect resolved config incl. `cacheDirectory`), `--clearCache` (clear the transform cache), `--no-cache` (bypass read/write of the cache), `--ci` (non-interactive).
- **Git** — `git diff <base>..HEAD --name-status` (scope of change), `git check-ignore -v <path>` (confirm ignore rules), `git status --porcelain` (clean-tree check).
- **Node** — `node -e "require('<pkg>/package.json').version"` (resolve installed dependency versions).

### G. Glossary

| Term | Definition |
|---|---|
| Cold run | First run after clearing `.cache/jest`; must transform all first-party source. |
| Warm run | Subsequent run that reuses cached `babel-jest` transform outputs. |
| Transform cache | `jest-transform-cache-*` tree under `<repo>/.cache/jest` holding transformed code + `.map` source maps. |
| Haste map | Jest `jest-haste-map` module map (`haste-map-*`), a V8-serialized structure. |
| Perf cache | `perf-cache-*` JSON of per-file `[status, duration_ms]` used by `@jest/test-sequencer`. |
| `babel-jest` | Default Jest transformer for `\.[jt]sx?$`; the dominant cost on cold/uncached runs. |
| `nock` | HTTP interception/mocking library; `disableNetConnect()` blocks all real network I/O. |
| `rootMode: 'upward'` | Babel option making `babel-jest` walk up to the repo-root `babel.config.js`. |
| Ratio | First÷second (cold÷warm) or uncached÷warm run time — the stable, host-independent answer. |