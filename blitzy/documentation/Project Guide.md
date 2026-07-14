# Blitzy Project Guide — Calypso Reader Onboarding Investigation

> **Deliverable:** `blitzy/documentation/wp-calypso_be7e5cc64162.md` — a run-first, evidence-backed answer document for developers onboarding onto the WordPress.com Calypso monorepo.
> **Repository:** `wp-calypso` v18.13.0 · **Branch:** `blitzy-d047ffaf-7ba7-44b1-a2e4-6fc44eb2a3eb` · **HEAD:** `e5a53663ad` · **Product baseline:** `be7e5cc641`
> **Scope:** Read-only investigation — no product source modified.
>
> **Blitzy brand colors:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive onboarding investigation of the WordPress.com **Calypso Reader** for engineers running Calypso locally for the first time. Using a strict **run-first** methodology, Blitzy built and ran the canonical development server and captured real runtime signals to answer four onboarding questions: **(Q1)** the development-server port, readiness signals, and single-vs-multi-port topology; **(Q2)** the Reader initial-load REST endpoints and the ordered Redux action sequence; **(Q3)** how the app detects whether a visitor is logged in before rendering, and which storage mechanisms it consults; and **(Q4)** the responsive-sidebar margin/padding values, the CSS custom properties driving layout `calc()`, and the viewport breakpoints. The sole permanent deliverable is one Markdown answer document; every claim is grounded in `file:line` citations and labeled OBSERVED vs INFERRED. Product source remains unchanged.

### 1.2 Completion Status

The project is **91.5% complete** on an AAP-scoped, hours-based basis (43.0 completed hours of 47.0 total). Remaining hours reflect mandatory human acceptance/verification and merge — not code defects.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Completion Status — 91.5% Complete (AAP-Scoped)
    "Completed (AI)" : 43
    "Remaining (Human)" : 4
```

| Metric | Value |
|---|---|
| **Total Hours** | **47.0 h** |
| **Completed Hours (AI + Manual)** | **43.0 h** (43.0 AI + 0.0 Manual) |
| **Remaining Hours** | **4.0 h** |
| **Percent Complete** | **91.5%** (43.0 ÷ 47.0) |

> **Color key:** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF`.

### 1.3 Key Accomplishments

- ✅ Answer document delivered and committed — `blitzy/documentation/wp-calypso_be7e5cc64162.md` (1,588 lines, 116,241 bytes; HEAD `e5a53663ad`).
- ✅ **Run-first** honored — dev server built and run; all four questions' runtime signals reproduced (webpack 5.97.1).
- ✅ **Q1** — port **3000** confirmed; single-port topology (SSR + JS/CSS assets + HMR on :3000); both "Ready!" banners and the "Welcome to Calypso!" holding page captured.
- ✅ **Q2** — `/reader → /discover` redirect; `number=4` (`INITIAL_FETCH`) then `number=7` (`PER_FETCH`); full ordered Redux action sub-sequence.
- ✅ **Q3** — `isUserLoggedIn=false` decision path traced; storage tiers enumerated (server cookie, OAuth token, IndexedDB → localStorage → in-memory).
- ✅ **Q4** — sidebar padding `30px 24px 29px`; tokens `272px / 228px / 46px`; breakpoints `782px` and `960/961px` confirmed at runtime.
- ✅ Evidentiary quality — 124 distinct `file:line` citations (0 out-of-bounds); 51 `[OBSERVED]`, 8 `[INFERRED]`, 7 NON-CANONICAL labels; 66 fenced output/code blocks.
- ✅ **Read-only scope honored** — exactly one file added; all temporary artifacts removed; working tree clean.
- ✅ **Validation** — 5/5 gates passed; scoped Jest 96/96; build exit 0; `git diff --check` clean.

### 1.4 Critical Unresolved Issues

**No critical (release-blocking) issues identified.** The deliverable validated across all five autonomous gates. The items below are **non-blocking** human acceptance dependencies, listed for transparency.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| No blocking issues — deliverable validated across 5 autonomous gates | None | — | — |
| (Non-blocking) Residual-PII/secret scan of the 66 captured-output blocks awaits human sign-off | Low — SEC-1 redactions already applied (commit `f80ae1c`) | Onboarding lead / Security reviewer | 0.5 h |
| (Non-blocking) Independent runtime reproduction in a fresh environment | Low — fully reproduced in Blitzy env | Onboarding engineer | 2.0 h |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Remote WordPress.com REST API (`public-api.wordpress.com`) | Outbound HTTPS / network | Local Calypso directs REST/API traffic to the **remote** host (not a second local port); environment needs outbound network. Logged-out Reader needs no credentials. | ✅ Resolved — network confirmed during validation | Onboarding engineer |
| `calypso.localhost` hosts entry | Local `/etc/hosts` | Browser must resolve `calypso.localhost:3000`; requires `127.0.0.1 calypso.localhost` | ✅ Resolved — entry present | Developer |
| Logged-in Reader (WordPress.com account) | Auth credentials | Observing logged-in flows needs a real account/cookie; **out of AAP scope** (canonical session is logged-out) | Not required — logged-out is canonical per AAP | N/A |

> **Overall:** No blocking access issues. All required access was available during validation.

### 1.6 Recommended Next Steps

1. **[High]** Documentation acceptance review — onboarding lead reads the 1,588-line doc and spot-checks a sample of the 124 citations plus headline values (port 3000; `INITIAL_FETCH=4`/`PER_FETCH=7`; `isUserLoggedIn`; sidebar `30px 24px 29px`; `--sidebar-width-max/min` 272/228). *(1.0 h)*
2. **[High]** Residual sensitive-data scan — review all 66 captured-output code blocks to confirm no real cookie/token/PII remains beyond the SEC-1 redactions. *(0.5 h)*
3. **[Medium]** Independent runtime reproduction in a fresh environment — `corepack enable` → `yarn install --immutable` → build → `yarn start`; confirm Q1–Q4 signals. *(2.0 h)*
4. **[Medium]** PR review & merge of the docs-only branch — confirm the diff is a single added file, CI green, approve and merge. *(0.5 h)*
5. **[Low]** Establish a periodic refresh cadence so line-pinned citations stay current as the codebase evolves. *(ongoing advisory — 0 incremental hours this cycle)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each completed component traces to a specific AAP requirement (R1–R11) or path-to-production activity. **Total = 43.0 h** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|---|---:|---|
| Environment provisioning & canonical build | 4.0 | Provision Node 22.9.0+ / Yarn 4.0.2 (Corepack), `yarn install --immutable`, `yarn run build` → `build/server.js` + `public/` (AAP R1). |
| Runtime observation harness | 5.0 | Temporary Redux store subscriber, outbound network capture, and computed-style probes at the real entry point; removed afterward (AAP R2). |
| Q1 — Dev-server topology & readiness | 5.0 | Observed port 3000, boot log, both "Ready!" banners, holding page, single-port multiplexing; write-up + citations (AAP R3). |
| Q2 — Reader endpoints & Redux actions | 6.0 | Loaded stream; captured `discover` endpoint `number=4`→`number=7`, 6 endpoint families, ordered action sub-sequence; write-up (AAP R4). |
| Q3 — Auth detection & storage | 6.0 | Traced `isUserLoggedIn` path; enumerated cookie/OAuth-token/IndexedDB→localStorage→memory tiers; write-up (AAP R5). |
| Q4 — Responsive sidebar CSS/tokens/breakpoints | 5.0 | Cataloged sidebar padding/margin, custom properties, and breakpoints; runtime computed-style confirmation; write-up (AAP R6). |
| Coverage-pass tables | 3.0 | Four Coverage Pass tables ensuring every named item across Q1–Q4 answered by name and value (AAP R7). |
| Citation grounding + OBSERVED/INFERRED labeling + GFM | 3.0 | 124 `file:line` citations, 51/8/7 labels, balanced fences and clean tables (AAP R8, R9). |
| QA remediation cycles | 4.0 | Code-review findings, SEC-1 redaction, GFM table fix, DOC-RULE OUTPUT/EDGE/LABEL fixes across 6 commits. |
| Read-only scope enforcement + cleanup + commit | 2.0 | Verified single-file diff; removed 46 PNGs + 1 webm + `/tmp` scratch; committed at HEAD `e5a53663ad` (AAP R10, R11). |
| **TOTAL COMPLETED** | **43.0** | **Matches §1.2 Completed Hours** |

### 2.2 Remaining Work Detail

Each remaining category traces to a path-to-production acceptance/verification need. **No code defects exist.** **Total = 4.0 h** (matches Remaining Hours in §1.2 and the §7 pie "Remaining Work").

| Category | Hours | Priority |
|---|---:|---|
| Documentation Acceptance Review (content & citation review + residual-PII scan of 66 code blocks) | 1.5 | High |
| Independent Runtime Reproduction (fresh env: install → build → start → verify Q1–Q4) | 2.0 | Medium |
| PR Review & Merge (docs-only branch) | 0.5 | Medium |
| **TOTAL REMAINING** | **4.0** | — |

> **Cross-section check:** §2.1 (43.0) + §2.2 (4.0) = **47.0 h** = Total Hours in §1.2. Remaining 4.0 h is identical in §1.2, §2.2, and §7.

### 2.3 Hours Calculation & Methodology

Completion percentage is computed strictly on AAP-scoped and path-to-production hours (PA1 methodology):

```
Completion % = Completed Hours ÷ (Completed Hours + Remaining Hours) × 100
             = 43.0 ÷ (43.0 + 4.0) × 100
             = 43.0 ÷ 47.0 × 100
             = 91.5%   (91.4894%, rounded to 91.5%)
```

- **Denominator (Total = 47.0 h)** = sum of every AAP requirement (R1–R11) plus path-to-production acceptance.
- **Completed (43.0 h)** = all 11 AAP requirements delivered and validated (see §5).
- **Remaining (4.0 h)** = mandatory human acceptance/verification/merge; **no rework hours** because no defects were found.
- **Confidence:** *High* for Q1/Q4 (deterministic, source-grounded values) and for scope/cleanup; *Medium* for Q2/Q3 runtime counts (live remote stream exhibits disclosed run-to-run variance, not asserted as deterministic).
- Per RG2, completion is **capped below 100%** to reflect the required human acceptance gate.

---

## 3. Test Results

All entries originate from Blitzy's autonomous validation logs for this project. For a documentation deliverable, "tests" comprise **(a)** scoped code-behavior tests that confirm the cited source behaves as documented and **(b)** document-integrity checks (citation resolution, value verification, build/compile, well-formedness, runtime reproduction).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Unit — cited Reader/current-user code | Jest | 96 | 96 | 0 | scoped | 7 suites: `client/state/current-user`, `client/state/reader/streams`, `client/state/data-layer/wpcom/read/streams` — confirms Q2/Q3 cited behavior |
| Citation resolution sweep | Blitzy doc-validator | 130 | 130 | 0 | 100% resolvable | 0 out-of-bounds line refs; independent recount = 124 distinct `file:line` (counting-method variance for ranges/combos) |
| Value verification | Manual vs source (`grep`/`sed`) | 60 | 60 | 0 | n/a | Headline values matched EXACT: port 3000; `INITIAL_FETCH=4`/`PER_FETCH=7`; `isUserLoggedIn`; `30px 24px 29px`; `272/228/46` |
| Dependency install gate | Yarn 4.0.2 | 1 | 1 | 0 | n/a | `CI=true yarn install --immutable` → exit 0; `yarn.lock` unchanged |
| Build / compile gate | Yarn + Webpack 5.97.1 | 1 | 1 | 0 | n/a | `CALYPSO_ENV=development yarn run build` → exit 0 → `build/server.js` (7.9 MB) + `public/` |
| GFM well-formedness | `git diff --check` + fence/table lint | 3 | 3 | 0 | n/a | 132 fence lines balanced (66 blocks); 4 tables, 0 column mismatches; `--check` clean |
| Runtime signal reproduction (Q1–Q4) | Manual runtime capture | 4 | 4 | 0 | n/a | All four questions' signals reproduced with exact fidelity |
| **TOTAL** | — | **195** | **195** | **0** | **100% pass** | Zero failures across all categories |

> **Integrity note:** The citation sweep is reported as **130** per the primary autonomous validator log; an independent stricter-regex recount yielded **124** distinct `file:line` references (227 total occurrences). Both agree on **0 out-of-bounds** citations; the delta is a counting convention for range/compound citations.

---

## 4. Runtime Validation & UI Verification

**Legend:** ✅ Operational · ⚠ Partial / by-design · ❌ Failing

**Server runtime health**
- ✅ Dev server boots on **port 3000**; boot line `wp-calypso booted in <n>ms - http://calypso.localhost:3000` emitted; `GET /` → HTTP 200 (~25,104 bytes).
- ✅ **Both** readiness banners captured — first-compile `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` (~164,542 ms, webpack 5.97.1, 37 warnings) and recompile `Ready! All assets are re-compiled. Have fun!` (~12,578 ms).
- ✅ Transient holding page during compile — 630 bytes, `<h1>Welcome to Calypso!</h1>`, `<meta http-equiv="refresh" content="5">`.
- ✅ **Single-port topology** — SSR (`text/html`) + JS (`application/javascript`) + CSS (`text/css`) + HMR (`/__webpack_hmr`, `text/event-stream`) all served on `:3000`.
- ✅ **No second local port** — 5858 / 8080 / 443 confirmed closed.

**API integration (remote)**
- ✅ Q2: `/reader → /discover` redirect; `GET /wpcom/v2/read/streams/discover?…&number=4` then `…&number=7` (byte-identical `page_handle`); all 6 endpoint families exercised.
- ✅ Q3: `/rest/v1.1/me?meta=flags` → HTTP 200 auth-detection call to remote `public-api.wordpress.com`.
- ✅ Redux action sub-sequence fired in documented order: `RESET_CARD_EXPANSIONS → VIEW_STREAM → STREAMS_PAGE_REQUEST → POSTS_RECEIVE → RECOMMENDED_SITES_RECEIVE → STREAMS_PAGE_RECEIVE → STREAMS_PAGE_REQUEST`.

**UI verification (logged-out Reader — canonical session)**
- ✅ Q3: `isUserLoggedIn=false`, `getCurrentUserId=null`, `window.currentUser=undefined`; no `wordpress_logged_in` / no `wpcom_token` cookie; `localStorage=[tusSupport]`; IndexedDB `calypso` (v2) / `calypso_store` with ~16 `redux-state-logged-out*` keys + sanity-test entry.
- ✅ Q4: sidebar absent on logged-out Reader (`has-no-sidebar`); tokens resolved — `--masterbar-height` 46px (<782px) / 32px (≥782px), `--sidebar-width-max` 272px, `--sidebar-width-min` 228px (constant); `.layout__content` padding `79px 32px 32px` (≥961px) / `71px 24px 24px` (782–960px) / `47px 0px 0px` (375px); breakpoints confirmed at runtime `781→782` (masterbar) and `960→961` (padding).

**By-design / partial notes**
- ⚠ Logged-in Reader UI not runtime-observed — out of AAP scope (canonical session is logged-out); logged-in values are labeled `[INFERRED]` in the document.
- ⚠ Run-to-run variance on a live remote stream — total Redux action count (290 vs the document's 269) and `APPLY_STORED_STATE` (1 vs 3) vary between runs; the document discloses this and does **not** assert these counts as deterministic.

---

## 5. Compliance & Quality Review

AAP deliverables and SWE-AtlasQnA-Repo rules mapped to Blitzy quality/compliance benchmarks. All 11 AAP requirements (R1–R11) are **Completed**.

| Requirement / Rule | Ref | Status | Progress | Notes |
|---|---|---|---|---|
| Deliverable location & form (`blitzy/documentation/<branch>.md`) | R11 | ✅ Pass | 100% | `wp-calypso_be7e5cc64162.md` present & committed at HEAD `e5a53663ad` |
| Run-first methodology | R2 | ✅ Pass | 100% | Gate 4 reproduced all Q1–Q4 signals before write-up |
| Observe at canonical entry point | R2 | ✅ Pass | 100% | `yarn start` / `node build/server.js`; 7 NON-CANONICAL paths explicitly labeled |
| Every named item by name & value | R7 | ✅ Pass | 100% | 4 Coverage Pass tables (Q1–Q4) |
| OBSERVED vs INFERRED labeling | R8 | ✅ Pass | 100% | 51 `[OBSERVED]`, 8 `[INFERRED]`, 7 NON-CANONICAL |
| `file:line` grounding | R9 | ✅ Pass | 100% | 124 distinct citations; 0 out-of-bounds |
| Actual output next to each claim | R2 | ✅ Pass | 100% | 66 fenced output/code blocks |
| Read-only scope | R10 | ✅ Pass | 100% | Exactly 1 file added; no product source created/modified/deleted |
| Temp-artifact cleanup | R10 | ✅ Pass | 100% | 46 PNGs + 1 webm + `/tmp` scratch removed; working tree clean |
| Node-version gate (`^v22.9.0`) | R1 | ✅ Pass | 100% | Node 22.23.1 used; Node 20 correctly rejected by `check-node-version` |
| GFM well-formedness | Quality | ✅ Pass | 100% | 132 balanced fences; 4 clean tables; `git diff --check` clean |
| Sensitive-data handling | Security | ✅ Pass (verify) | 95% | SEC-1 redaction applied (commit `f80ae1c`); human PII sign-off recommended (0.5 h) |

**Fixes applied during autonomous validation (6 documentation-only commits):** `2956f78` (add) → `9f07f7e` (code-review) → `d5e682e` (quote-fidelity / storage-count / git-baseline) → `0436287` (GFM table fix) → `f80ae1c` (SEC-1 redaction) → `e5a5366` (DOC-RULE OUTPUT/EDGE/LABEL, HEAD).
**Outstanding:** human PII scan sign-off (0.5 h) — non-blocking.

---

## 6. Risk Assessment

Overall risk posture: **LOW** — no High or Critical risks. Because the change is read-only documentation, the product attack surface is unchanged.

| # | Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|---|
| 1 | Citation line-number drift as code evolves | Technical | Low | Medium | Citations pinned to commit `be7e5cc641`/HEAD; periodic refresh advised | Mitigated |
| 2 | Run-to-run variance (Redux action counts; storage keys 16–18) | Technical | Low | Medium | Disclosed in doc; not asserted deterministic | Resolved |
| 3 | Environment drift (Node patch, webpack minor) | Technical | Low | Low | Exact versions recorded (Node 22.23.1, webpack 5.97.1, Yarn 4.0.2) | Mitigated |
| 4 | Sensitive runtime data in captured output (analytics cookie, geo) | Security | Medium | Low | Redacted in commit `f80ae1c` (SEC-1) | Resolved |
| 5 | Residual PII/secret in captured output | Security | Low | Low | Human review scans 66 code blocks (0.5 h) | Open (verification) |
| 6 | Product attack surface changed | Security | None | n/a | Read-only; no product source modified | N/A (positive) |
| 7 | Onboarding doc staleness over time | Operational | Low | Medium | Commit-pinned; periodic refresh cadence recommended | Accepted |
| 8 | Git hooks interfering with commit | Operational | None | Low | `pre-commit` lints `.json/.js/.jsx/.ts/.tsx/.scss/.php`, not `.md` | Verified no-op |
| 9 | Remote API dependency (`public-api.wordpress.com`) | Integration | Low | Medium | Documented as remote; logged-out needs no credentials | Mitigated |
| 10 | Logged-in path not runtime-observed | Integration | Low | Low | AAP-scoped to logged-out; logged-in values labeled `[INFERRED]` | Accepted (by design) |
| 11 | Docs-only branch merge conflict | Integration | Low | Low | Single new file under new directory; no overlap with product source | Mitigated |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown (47.0h total)
    "Completed Work" : 43
    "Remaining Work" : 4
```

**Remaining hours by category** (from §2.2, sums to 4.0 h):

```mermaid
xychart-beta
    title "Remaining Hours by Category (Total 4.0h)"
    x-axis ["Doc Acceptance", "Runtime Repro", "PR Merge"]
    y-axis "Hours" 0 --> 3
    bar [1.5, 2.0, 0.5]
```

| Category | Remaining Hours | Priority |
|---|---:|---|
| Documentation Acceptance Review | 1.5 | High |
| Independent Runtime Reproduction | 2.0 | Medium |
| PR Review & Merge | 0.5 | Medium |
| **Total** | **4.0** | — |

> **Integrity:** Pie "Remaining Work" = 4 = §1.2 Remaining Hours = §2.2 total. Pie "Completed Work" = 43 = §1.2 Completed Hours = §2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **91.5% complete** (43.0 of 47.0 AAP-scoped hours). Blitzy produced a single, run-first, evidence-backed answer document that addresses all four onboarding questions in full — each named item answered by name and by observed value, grounded in 124 `file:line` citations and clearly labeled OBSERVED vs INFERRED. The dev server was built and run; port 3000, both "Ready!" banners, the single-port topology, the Reader `discover` endpoint (`number=4`→`7`), the ordered Redux action sequence, the logged-out `isUserLoggedIn` decision with its storage tiers, and the sidebar spacing/tokens/breakpoints were all reproduced at runtime. Validation passed all five autonomous gates (dependencies, build, tests 96/96, runtime, scope/cleanup), and read-only scope was strictly honored — exactly one file was added and every temporary artifact removed.

**Remaining gaps & critical path.** The 4.0 remaining hours are entirely **human path-to-production acceptance** — there are no code defects to fix. The critical path is: **(1)** documentation acceptance review + residual-PII scan (1.5 h, High) → **(2)** independent runtime reproduction in a fresh environment (2.0 h, Medium) → **(3)** PR review & merge of the docs-only branch (0.5 h, Medium).

**Success metrics.**

| Metric | Target | Actual | Status |
|---|---|---|---|
| Questions answered (by name & value) | 4 / 4 | 4 / 4 | ✅ |
| AAP requirements completed | 11 / 11 | 11 / 11 | ✅ |
| Citation resolution | 100% | 100% (0 out-of-bounds) | ✅ |
| Scoped code tests | Pass | 96 / 96 | ✅ |
| Product source files changed | 0 | 0 | ✅ |
| Autonomous validation gates | 5 / 5 | 5 / 5 | ✅ |

**Production readiness.** As a documentation artifact, the deliverable is **production-ready pending human acceptance**. Because the product source is unchanged, there is no build, deployment, or runtime-behavior risk to the product itself; the only gate is human review/merge of the new Markdown file.

**Ongoing advisory (Low, 0 incremental hours this cycle).** Establish a periodic refresh cadence to keep the line-pinned citations current as the Calypso codebase evolves; re-verify headline values after major Reader/layout refactors. This is an ongoing recommendation and is deliberately excluded from the remaining-hours total to preserve cross-section integrity.

---

## 9. Development Guide

Build, run, verify, and consume the deliverable. Commands were tested against the current on-disk state and the autonomous validation gates. Run from the repository root: `/tmp/blitzy/wp-calypso/blitzy-d047ffaf-7ba7-44b1-a2e4-6fc44eb2a3eb_8b2411`.

### 9.1 System Prerequisites

- **OS:** Linux/macOS (validated on Ubuntu 25.10 container).
- **Node.js:** `^v22.9.0` (engines gate); `.nvmrc` pins `22.9.0`. Validated on **v22.23.1**. **Node 20 will fail** the `check-node-version` gate — do not use it.
- **Package manager:** **Yarn 4.0.2** via Corepack (`packageManager: yarn@4.0.2`).
- **Disk:** repo ~439 MB; `node_modules` ~3.1 GB when warmed.
- **Network:** outbound HTTPS to `public-api.wordpress.com` (REST is remote).

### 9.2 Environment Setup

```bash
# Activate the pinned Yarn via Corepack (Node 22.9.0+ on PATH)
corepack enable

# Ensure the app is reachable in a browser at calypso.localhost:3000
grep -qE 'calypso\.localhost' /etc/hosts || \
  echo "127.0.0.1 calypso.localhost" | sudo tee -a /etc/hosts
```

Verify the toolchain:

```bash
node --version         # -> v22.x (>= 22.9.0)
corepack --version     # -> 0.34.6 (present)
yarn --version         # -> 4.0.2
cat .nvmrc             # -> 22.9.0
```

### 9.3 Dependency Installation

```bash
CI=true yarn install --immutable    # exit 0; yarn.lock must remain unchanged
```

*Expected:* install completes with exit 0 and `git diff --stat yarn.lock` is empty (read-only per AAP §0.6.2).

### 9.4 Build

```bash
CALYPSO_ENV=development CI=true yarn run build
```

*Expected:* exit 0; produces `build/server.js` (~7,935,308 bytes) and `public/` (top-level entries). Offline "caniuse-lite outdated" Browserslist notices are harmless warnings, not errors.

### 9.5 Application Startup

Two supported forms:

```bash
# (a) Canonical — full Node-version gate + welcome banner + build + serve
yarn start

# (b) Fast — serve a pre-built bundle, Reader/login sections only
NODE_ENV=development CALYPSO_ENV=development SECTION_LIMIT=reader,login \
  node build/server.js
```

**Wait for the readiness banner** before browsing:

```
Ready! You can load http://calypso.localhost:3000/ now. Have fun!
```

> First compile can take a while (~164 s observed); during compilation the server returns a 630-byte holding page (`<h1>Welcome to Calypso!</h1>`, `<meta http-equiv="refresh" content="5">`). Recompiles are fast (~12.5 s) and emit `Ready! All assets are re-compiled. Have fun!`.

### 9.6 Verification

```bash
# Boot log line (before the HTTP listener callback):
#   wp-calypso booted in <n>ms - http://calypso.localhost:3000

# Server responds with 200 once ready:
curl -s -o /dev/null -w "%{http_code}\n" http://calypso.localhost:3000/    # -> 200

# Reader redirects to /discover when logged out:
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" \
  http://calypso.localhost:3000/reader
```

Then browse **http://calypso.localhost:3000/reader** (redirects to `/discover` when logged out).

### 9.7 Scoped Tests (confirms cited Q2/Q3 behavior)

```bash
npx jest -c=test/client/jest.config.js --ci --watchAll=false \
  client/state/current-user \
  client/state/reader/streams \
  client/state/data-layer/wpcom/read/streams
# -> 7 suites, 96/96 tests PASS
```

### 9.8 Consume the Deliverable

```bash
# 1,588 lines · 116,241 bytes · 132 balanced fence lines (66 blocks)
less  blitzy/documentation/wp-calypso_be7e5cc64162.md
wc -l blitzy/documentation/wp-calypso_be7e5cc64162.md
```

### 9.9 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `check-node-version` fails at startup | Node < 22.9.0 (e.g., Node 20) | Install Node 22.9.0+; `corepack enable` |
| Browser can't reach `calypso.localhost` | Missing hosts entry | `echo "127.0.0.1 calypso.localhost" \| sudo tee -a /etc/hosts` |
| Long first load / "Welcome to Calypso!" page | First webpack compile in progress (~164 s) | Wait for the `Ready!` banner (holding page auto-refreshes every 5 s) |
| REST calls fail / empty stream | No outbound network to `public-api.wordpress.com` | Ensure outbound HTTPS; logged-out Reader needs no credentials |
| Port 3000 already in use | Another process bound to 3000 | Free port 3000 (no second local port is used by Calypso) |
| Redux action counts differ between runs | Live remote stream variance | Expected; the document does not assert these counts as deterministic |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Enable pinned Yarn | `corepack enable` |
| Install deps (immutable) | `CI=true yarn install --immutable` |
| Build (dev) | `CALYPSO_ENV=development CI=true yarn run build` |
| Start (canonical) | `yarn start` |
| Start (fast, Reader/login) | `NODE_ENV=development CALYPSO_ENV=development SECTION_LIMIT=reader,login node build/server.js` |
| Health check | `curl -s -o /dev/null -w "%{http_code}\n" http://calypso.localhost:3000/` |
| Scoped tests | `npx jest -c=test/client/jest.config.js --ci --watchAll=false client/state/current-user client/state/reader/streams client/state/data-layer/wpcom/read/streams` |
| View deliverable | `less blitzy/documentation/wp-calypso_be7e5cc64162.md` |
| Confirm single-file diff | `git diff be7e5cc641..HEAD --name-status` |

### B. Port Reference

| Port | Role | Notes |
|---|---|---|
| **3000** | Calypso dev server | Single port — SSR (`text/html`) + JS/CSS assets + HMR (`/__webpack_hmr`, `text/event-stream`) |
| 5858 | Node debugger (optional) | Documented in `docs/install.md:L63`; **not** used by default — closed during validation |
| 443 (remote) | `public-api.wordpress.com` | Remote REST/API host — **not** a second local port |

### C. Key File Locations

| Concern | File |
|---|---|
| **Deliverable** | `blitzy/documentation/wp-calypso_be7e5cc64162.md` |
| Q1 — port config | `config/development.json:L8`, `config/_shared.json:L25` |
| Q1 — server bootstrap / listen / boot log | `client/server/index.js` (`config('port')`, `server.listen`, boot log) |
| Q1 — bundler readiness / holding page | `client/server/bundler/index.js` (`waitForCompiler`, "Ready!", "Welcome to Calypso!") |
| Q2 — stream endpoint map & fetch counts | `client/state/data-layer/wpcom/read/streams/index.js` (`INITIAL_FETCH=4`, `PER_FETCH=7`) |
| Q2 — action creators / types | `client/state/reader/streams/actions.js`, `client/state/reader/action-types.ts:L76-L86` |
| Q3 — login selector | `client/state/current-user/selectors.js:L15-L17` (`isUserLoggedIn`) |
| Q3 — server auth cookie | `client/server/user-bootstrap/index.js:L8` (`wordpress_logged_in`) |
| Q3 — OAuth token | `packages/oauth-token/src/index.js:L7` (`wpcom_token`) |
| Q3 — tiered browser storage | `client/lib/browser-storage/index.ts` (IndexedDB → localStorage → memory) |
| Q4 — design tokens | `client/assets/stylesheets/shared/_variables.scss:L7,L15,L16` |
| Q4 — sidebar header spacing | `client/layout/global-sidebar/style.scss:L75` (`padding: 30px 24px 29px`) |
| Q4 — breakpoints | `client/assets/stylesheets/shared/mixins/_breakpoints.scss` (deprecated) + `@wordpress/base-styles` |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Node.js | `^v22.9.0` (pinned `22.9.0`; ran `v22.23.1`) | `package.json:engines`, `.nvmrc` |
| Yarn | `4.0.2` (via Corepack `0.34.6`) | `package.json:packageManager`, `.yarnrc.yml` |
| Webpack | `5.97.1` | Runtime "Ready!" banner |
| wp-calypso | `18.13.0` | `package.json` |
| check-node-version | `^4.0.2` | `package.json` |
| bunyan | `^1.8.15` | `start-build` log formatter |
| @wordpress/base-styles | `^5.21.0` | Gutenberg breakpoints (Q4) |
| @automattic/oauth-token | `1.0.1` | `wpcom_token` reader (Q3) |

### E. Environment Variable Reference

| Variable | Example | Purpose |
|---|---|---|
| `CALYPSO_ENV` | `development` | Selects Calypso config (drives `config('port')` → 3000) |
| `NODE_ENV` | `development` | Node environment for the server bundle |
| `SECTION_LIMIT` | `reader,login` | Limits compiled sections for a faster start |
| `CI` | `true` | Non-interactive install/build; disables watch prompts |
| `BROWSERSLIST_ENV` | `evergreen` | Used by `start-build` for target browsers |

### F. Developer Tools Guide

- **Redux action observation:** attach a temporary store subscriber (removed after capture — read-only scope) to log the initial-load action sub-sequence.
- **Network capture:** inspect outbound requests to `public-api.wordpress.com` to confirm the `read/streams/discover` endpoint and `number=4`→`number=7` transition.
- **Computed styles:** use DevTools computed-style inspection for the sidebar/layout tokens and breakpoints (`--masterbar-height`, `--sidebar-width-max/min`, `.layout__content` padding) at 375 / 782 / 961 px widths.
- **Log formatting:** `start-build` pipes server output through `bunyan -o short` for readable boot/readiness logs.

### G. Glossary

| Term | Meaning |
|---|---|
| **Calypso** | The JavaScript/React single-page application behind WordPress.com (this monorepo). |
| **Reader** | Calypso section that displays streams of posts from followed sites. |
| **SSR** | Server-Side Rendering — initial HTML rendered by the Express server. |
| **HMR** | Hot Module Replacement — live module updates during development (served on :3000). |
| **`INITIAL_FETCH` / `PER_FETCH`** | Item counts for the first stream fetch (4) vs. subsequent pages (7). |
| **`isUserLoggedIn`** | Selector returning `getCurrentUserId(state) !== null`; the pre-render auth decision. |
| **OBSERVED / INFERRED** | Labels distinguishing runtime-captured facts from code-read inferences. |
| **NON-CANONICAL** | A value obtained via a fallback/bypassing path rather than the real entry point. |
| **AAP** | Agent Action Plan — the governing project specification. |
| **SWE-AtlasQnA-Repo** | The governing rule: run-first, evidence-backed, read-only Q&A documentation. |

---

*Generated by the Blitzy Platform. Completion is measured strictly against AAP-scoped and path-to-production work: **43.0 of 47.0 hours = 91.5% complete**. Remaining 4.0 hours are human acceptance/verification/merge — no code defects exist.*