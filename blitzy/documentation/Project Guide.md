# Blitzy Project Guide — Logged-Out Reader "Like" Intent Investigation (wp-calypso)

> **Repository:** `Automattic/wp-calypso` &nbsp;|&nbsp; **Branch:** `blitzy-86b269e0-17ba-4075-9310-6c345cfb19c5` &nbsp;|&nbsp; **HEAD:** `9434faf066` &nbsp;|&nbsp; **Citation base:** `be7e5cc641622d153040491fd5625c6cb83e12eb`
> **Task type:** Read-only investigative Q&A / root-cause documentation &nbsp;|&nbsp; **Brand palette:** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-backed investigative document that explains — with real runtime output and exact `file:line` citations — how a "like" clicked in the WordPress.com Reader **while logged out** is *intended* to survive the authentication boundary, and precisely *why* that intent is lost after the user completes signup or login. The audience is Automattic Calypso/Reader engineers and technical stakeholders. It is a **read-only** code investigation against the `Automattic/wp-calypso` monorepo: no source file is modified, created, or deleted; the sole artifact is one Markdown answer document. The technical scope spans the Reader like button, the `reader-ui` Redux slice, the logged-out layout, the login-window hook, and the state-persistence utilities.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'17px'}}}%%
pie showData title Completion Status — 95.6% Complete
    "Completed Work (AI)" : 43
    "Remaining Work" : 2
```

<sub>Slice colors: **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`** (Violet-Black `#B23AF2` outline). Center/label reads **95.6% Complete**.</sub>

| Metric | Value |
|---|---|
| **Total Hours** | **45.0 h** |
| **Completed Hours (AI + Manual)** | **43.0 h** |
| &nbsp;&nbsp;• AI / Autonomous | 43.0 h |
| &nbsp;&nbsp;• Manual | 0.0 h |
| **Remaining Hours** | **2.0 h** |
| **Percent Complete** | **95.6 %** |

> **Calculation (PA1, AAP-scoped):** `Completion % = Completed ÷ (Completed + Remaining) = 43.0 ÷ 45.0 = 95.6%`. 100% of the AAP-scoped autonomous work (investigation + document) is delivered; the 2.0 h remaining is human path-to-production (review/acceptance) only — there is no shippable code to deploy because the repository is intentionally unchanged.

### 1.3 Key Accomplishments

- ✅ **Sole deliverable authored & committed** — `blitzy/documentation/wp-calypso_be7e5cc64162.md` (635 lines / 50,780 bytes), 13 structured sections, answer-first.
- ✅ **All five sub-questions answered directly (R1–R5)** with a lead-with-answer TL;DR table.
- ✅ **Run-code-first evidence captured** — the real `readerUi` reducer plus `serialize`/`deserialize` were executed under the repo's own Jest runner; before/during/after state transitions recorded.
- ✅ **Source of truth proven at runtime** — in-memory Redux state is operative; persisted and handoff-token candidates ruled out empirically.
- ✅ **Determinism established** — observation output byte-identical across two runs (excluding the volatile `Time:` line), refuting the timing/race hypothesis.
- ✅ **Exhaustive named-item coverage** — 3 source-of-truth candidates + 3 skip-cause candidates (by name), 8 sibling requires-login intents, 3 edge/transitional branches, 2 decisive negatives.
- ✅ **Every claim grounded** — 55 `file:line` anchors pinned to the citation commit, with consistent `[observed-at-runtime]` / `[inferred-from-reading]` labeling.
- ✅ **Read-only constraint honored** — temporary observation spec deleted; `git diff` shows only the deliverable; working tree clean.
- ✅ **Autonomous validation passed** — 9 validation phases and 5 production-readiness gates all PASS; zero edits required.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None — no blocking issues._ The deliverable is complete, runtime-validated, citation-accurate, read-only-compliant, and committed. | None | — | — |

> Note: the investigation *documents* a real product UX defect (a logged-out like is silently lost after login). Fixing it is **explicitly out of scope** per AAP §0.5.2 (explain-not-repair) and is therefore recorded as a handoff item in §6 (risk O1), not an unresolved issue of this task.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| `Automattic/wp-calypso` repository | Read/write (branch) | None — repository cloned; branch `blitzy-86b269e0-…` checked out at HEAD `9434faf066`. | ✅ Resolved | Blitzy |
| Node / Yarn toolchain | Local execution | None — Node `v22.23.1` (satisfies `engines ^v22.9.0`), Yarn `4.0.2` via Corepack, `node_modules` populated. | ✅ Resolved | Blitzy |
| Live WordPress.com OAuth session / login popup | External auth service | Cannot be exercised in the sandbox (real session + popup-based OAuth). Bounded by design: the decisive state model is validated at the unit level; the E2E popup/reload is documented from code and labeled `[inferred-from-reading]`. | ⚠ Accepted (documented constraint) | Human SME |

### 1.6 Recommended Next Steps

1. **[Medium]** SME technical review of the R1–R5 findings — confirm the in-memory-source-of-truth, reload-not-replay, and cleanup-by-teardown conclusions against Calypso Reader/Redux domain knowledge (~1.0 h).
2. **[Medium]** Independently reproduce the runtime observation — recreate the Appendix §A.4 spec, run the client Jest command, diff against §A.1, then delete the spec (~0.5 h).
3. **[Low]** Accept the deliverable and merge the branch / close the Q&A task (~0.5 h).
4. **[Low / optional, separate initiative]** If remediation is later desired, open a follow-up to persist or replay the pending like after login (out of scope here).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement (investigation clusters) or the run-code-first / read-only rules.

| Component | Hours | Description |
|---|---:|---|
| Environment & Toolchain Setup | 2.0 | Node `^22.9.0` via Corepack, Yarn `4.0.2`, `yarn install` (git-ignored `node_modules`), Jest client-config discovery (`rootDir: client/`). |
| Repository Scope Discovery (6 clusters / 29 files) | 5.0 | Traversed the 18,879-file monorepo; identified the exact capture path, intent store, replay surface, persistence primitives, supporting paths, and sibling callers participating in the logged-out like → auth → replay flow. |
| R3 — Run-Code-First Runtime Observation | 5.0 | Authored + ran a temporary Jest spec exercising the **real** `readerUi` reducer with **real** `serialize`/`deserialize`; captured before/during/after and the serialized blob; established the persistence asymmetry. |
| R1 — Capture-Path Trace | 2.5 | Traced `handleLikeToggle` logged-out early-return dispatching `registerLastActionRequiresLogin`; documented the Reader-wrapper bypass nuance. |
| R2 + R4 — Replay-Surface Trace & Skip Condition | 4.0 | Traced the sole consumer `logged-out.jsx` → join-conversation dialog → login popup → `onLoginSuccess` reload/redirect branch selection. |
| R5 — Skip-Cause Classification & Decisive Negatives | 3.0 | Ruled out timing/race and initialization-order; proved single selector consumer and no middleware/saga consumes the action types (grep). |
| Sibling-Intent Exhaustive Enumeration (8) | 3.0 | like/unlike, comment-like/unlike, reply, comment, comment-submit, follow-site, follow-tag, sidebar-link — payload shapes, `redirectTo` presence, cause→effect. |
| Edge / Secondary / Transitional Branches (3) | 2.0 | Reader-tag-embed popup branch, `sidebar-link` navigate-on-success variant, dialog-cancel cleanup. |
| SPA-Pattern Web-Research Framing | 1.5 | OAuth `state` parameter, stored return-URL, popup-preserves-state — framing only, never substituting for code evidence. |
| Deliverable Authoring (635 lines / 13 sections) | 6.5 | Structured Markdown: TL;DR table, methodology, R1–R5, sibling table, edges, decisive negatives, end-to-end Mermaid trace, Appendix A.1–A.6 with verbatim output. |
| Citation Grounding & Observed/Inferred Labeling (55 anchors) | 2.5 | Pinned `file:line` references to the citation commit; consistent `[observed-at-runtime]` / `[inferred-from-reading]` tags. |
| Autonomous Final Validation (9 phases / 5 gates) | 5.0 | Environment, dependencies, citation bounds check, runtime reproduction, completeness, documentation, pre-commit, commit, clean-tree — all gates PASS. |
| Read-Only Cleanup & Clean-Tree Verification | 1.0 | Deleted temp spec + Jest cache; `git status` clean; diff = only the deliverable. |
| **Total Completed** | **43.0** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production for a documentation deliverable (no code to deploy). Each item traces to acceptance of the AAP deliverable.

| Category | Hours | Priority |
|---|---:|---|
| SME technical review of investigative findings (R1–R5) | 1.0 | Medium |
| Independent runtime reproduction (§A.4 → §A.1) | 0.5 | Medium |
| Stakeholder acceptance & branch merge | 0.5 | Low |
| **Total Remaining** | **2.0** | **Matches Remaining Hours in §1.2 & §7 pie** |

### 2.3 Total Project Hours & Completion Calculation

| Quantity | Hours |
|---|---:|
| Completed (§2.1 total) | 43.0 |
| Remaining (§2.2 total) | 2.0 |
| **Total Project Hours** | **45.0** |

> **Formula:** `Completion % = 43.0 ÷ (43.0 + 2.0) = 43.0 ÷ 45.0 = 95.5556% ≈ 95.6%`.
> **Integrity check:** §2.1 (43.0) + §2.2 (2.0) = §1.2 Total (45.0) ✅ — and §2.2 total (2.0) equals §1.2 Remaining and the §7 pie "Remaining Work" value.

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project and were independently reproduced this session using `CI=true yarn test-client …` (Jest `29.7.0`, `@automattic/calypso-jest` preset, `TZ=UTC`, `rootDir: client/`).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Existing `reader-ui` unit suite | Jest 29.7.0 | 7 | 7 | 0 | N/A (targeted) | `reducer.js`, `actions.js`, `selectors.js` — 3 suites; register→store & clear→null. |
| Run-code-first observation spec | Jest 29.7.0 | 2 | 2 | 0 | N/A (targeted) | Temporary spec (created, run, **deleted**). Combined-reducer serialize/deserialize + sub-reducer facts; before/during/after transitions. Reproduced across **2 runs**, byte-identical output (excl. `Time:`). |
| **Total** | — | **9** | **9** | **0** | — | **100% pass rate; deterministic & stable.** |

> **Coverage note:** this is a read-only investigation, not a feature build, so line/branch coverage targets are not applicable. The tests are *targeted observations* of the pure state functions that decide the outcome (the canonical unit-level entry point). No coverage instrumentation was in scope.

---

## 4. Runtime Validation & UI Verification

**Runtime health (state-model — observed at runtime):**

- ✅ **Operational** — Client Jest runner (`test-client`) executes the `reader-ui` suite and the observation spec.
- ✅ **Operational** — Real production `readerUi` reducer + real `serialize`/`deserialize` utilities exercised (canonical unit-level path).
- ✅ **Operational** — Before/during/after transition observed: `lastActionRequiresLogin` `null` → `{ type:'like', siteId:123, postId:456 }` → `null` after serialize/deserialize.
- ✅ **Operational** — Control sibling `lastPath` (persistence-wrapped) survives the round-trip (`/reader/feeds/123`), confirming the opt-in-persistence asymmetry.
- ✅ **Operational** — Determinism/stability: output byte-identical across 2 runs (excluding the volatile `Time:` line).
- ✅ **Operational** — Read-only invariant: working tree clean; `git diff be7e5cc641..HEAD` = only the added deliverable.

**End-to-end auth flow (inferred from code — sandbox constraint):**

- ⚠ **Partial (by design)** — The full sequence *click → join-conversation dialog → WordPress.com login popup → `postMessage` success → `window.location.reload()`* is documented from the code paths and labeled `[inferred-from-reading]`. A live logged-in WordPress.com OAuth session and its popup cannot be driven in the sandbox; the deterministic in-memory state model that governs the outcome **is** observed.

**UI verification:**

- ▫ **Not applicable** — No UI was built or modified. Reader UI components (`like-button`, `reader-join-conversation/dialog`) were read as evidence only. No screenshots or visual regression apply to a read-only code investigation.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and the "SWE-AtlasQnA-Repo" ruleset to autonomous validation outcomes. Fixes applied during validation: **zero edits were required** — the deliverable was already accurate (the prior agent even tightened citations to be *more* precise than the AAP).

| Benchmark / AAP Rule | Requirement | Status | Progress | Notes |
|---|---|---|---|---|
| Deliverable location & name | `blitzy/documentation/wp-calypso_be7e5cc64162.md` | ✅ Pass | 100% | Committed at HEAD `9434faf066`. |
| Run-code-first methodology | Run code & capture real output first | ✅ Pass | 100% | §2 / §A.1; reproduced live this session. |
| Stability ≥ 2 runs | Confirm value stable across ≥ 2 runs | ✅ Pass | 100% | §A.2 — byte-identical except `Time:`. |
| Real entry point | Canonical path; label non-canonical | ✅ Pass | 100% | Real reducer + utils; E2E labeled `[inferred-from-reading]`. |
| Every condition (edge/secondary/transitional) | Beyond happy path | ✅ Pass | 100% | §9 — tag-embed popup, sidebar-link, dialog-cancel. |
| Before / during / after | Observe state change | ✅ Pass | 100% | §2.3 — `null` → populated → `null`. |
| Cover every named item | 3+3 candidates, 8 siblings | ✅ Pass | 100% | §5 (source-of-truth), §7 (skip-cause), §8 (siblings). |
| Actual unedited output + commands | No paraphrase / no `// …` elision | ✅ Pass | 100% | §A.1 (107 lines) + exact command in §2.1. |
| Ground every claim; label observed vs inferred | `file:line` or observed output | ✅ Pass | 100% | 55 anchors + consistent labels. |
| Lead with the direct answer | Answer-first | ✅ Pass | 100% | §1 TL;DR table. |
| Read-only (source unchanged; temp removed) | Byte-for-byte unchanged | ✅ Pass | 100% | Diff = only deliverable; tree clean. |
| Explain-not-repair | No remediation | ✅ Pass | 100% | §A.5 scope statement. |
| Citation accuracy | Anchors in-range at pinned commit | ✅ Pass | 100% | 68 anchors, 0 out-of-range; R1/R3/R4 spot-verified live. |
| Human SME sign-off | Independent acceptance | ⏳ Pending | 0% | The 2.0 h remaining (§2.2). |

---

## 6. Risk Assessment

Because this is a read-only documentation task with **no shipped code, dependencies, credentials, or deployment**, most classic software risks are not applicable. The genuine residual risks are documentation-integrity and acceptance risks, plus a handoff note for the documented (unremediated) product defect.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| **T1** — Citation `file:line` drift if the document is read against a different commit | Technical | Low | Low | Anchors pinned to `be7e5cc641…`; 68 verified in-range; R1/R3/R4 spot-checked live | Mitigated |
| **T2** — E2E popup/reload behavior is inferred, not observed (sandbox cannot run live OAuth) | Technical | Low–Medium | Low | Clearly labeled `[inferred-from-reading]`; the decisive R3 state model **is** observed & reproducible; exact `file:line` cited | Accepted / Documented |
| **T3** — Reproduction depends on the full Calypso monorepo + Node `^22.9` | Technical | Low | Low | Exact versions, commands, and full spec provided (§2.1 / §A.1 / §A.4); reproduced live | Mitigated |
| **O1** — Documented product defect remains unremediated (logged-out like lost after login) | Operational | Medium | N/A | Remediation is **out of scope** per AAP §0.5.2 (explain-not-repair); flagged for a possible follow-up initiative | Out-of-scope by design; handed off |
| **A1** — SME may request additional coverage or contest a finding | Acceptance / Process | Low | Low | Exhaustive named-item coverage; grounded citations; reproducible runtime evidence; answer-first format | Low residual (= 2.0 h remaining) |
| **S1** — Security exposure | Security | None | N/A | No code/dependencies/credentials/attack surface introduced; temporary spec deleted | N/A — no security-relevant change |
| **I1** — Integration failure | Integration | None | N/A | Nothing integrated or changed; the login popup + `postMessage` is investigated as evidence only | N/A — no integration |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 43
    "Remaining Work" : 2
```

**Remaining work by priority** (2.0 h total):

| Priority | Hours | Items |
|---|---:|---|
| High (blocking) | 0.0 | None — nothing blocks |
| Medium | 1.5 | SME technical review (1.0) + runtime reproduction (0.5) |
| Low | 0.5 | Acceptance & merge |
| **Total** | **2.0** | **Equals §1.2 Remaining and §2.2 total** |

> **Integrity:** the pie chart "Remaining Work" (2) equals the §1.2 Remaining Hours (2.0) and the sum of the §2.2 "Hours" column (2.0). "Completed Work" (43) equals the §1.2 Completed Hours (43.0) and the §2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-backed answer to the user's question. All five sub-questions (R1–R5) are answered directly and grounded in code and captured runtime output. The decisive finding — that the pending like intent lives **only** in non-persisted, in-memory Redux state (`readerUi.lastActionRequiresLogin`) and is destroyed by the `window.location.reload()` taken on login success when the payload carries no `redirectTo` — is proven at the unit level and reproduced deterministically across two runs. The three "worlds" the user named (in-memory / persisted / handoff token) and the three skip causes (timing / initialization-order / cleanup) are each resolved **by name**, and the eight sibling requires-login intents plus three edge branches are enumerated exhaustively.

**Remaining gaps & critical path.** The project is **95.6% complete (43.0 h of 45.0 h)**. There are no blocking issues and no incomplete AAP work; the remaining **2.0 h** is human path-to-production: SME technical review (1.0 h), independent runtime reproduction (0.5 h), and acceptance/merge (0.5 h). The critical path is simply human sign-off.

**Production-readiness assessment.** For a read-only documentation deliverable, "production" means acceptance and merge of the answer document. The document is committed, the source tree is byte-for-byte unchanged, the runtime evidence is reproducible, and every claim is grounded — so it is ready for review. Two items are surfaced for stakeholders: (1) the E2E popup/reload behavior is `[inferred-from-reading]` due to the sandbox's inability to run live OAuth (the state model that decides the outcome is observed); and (2) the investigation documents a genuine, still-unremediated product UX defect whose fix is explicitly out of scope and would be a separate initiative.

**Success metrics.**

| Metric | Target | Result |
|---|---|---|
| AAP requirements completed | 100% of scoped items | 17 / 17 line items ✅ |
| Autonomous tests passing | 100% | 9 / 9 ✅ |
| Runtime observation stable | ≥ 2 identical runs | 2 / 2 (byte-identical) ✅ |
| Citations in-range at pinned commit | 100% | 68 / 68 ✅ |
| Read-only invariant | Source unchanged | Diff = only deliverable ✅ |
| Completion | ≤ 99% pre-review | 95.6% ✅ |

---

## 9. Development Guide

A read-only investigation: "build & run" here means **reproducing the runtime evidence** and **verifying the read-only invariant**. All commands below were tested live this session.

### 9.1 System Prerequisites

- **OS:** Linux or macOS (Windows via WSL2).
- **Node.js:** `^22.9.0` (repo-canonical; `.nvmrc` pins `22.9.0`; the sandbox uses `v22.23.1`, which satisfies the constraint).
- **Corepack:** bundled with Node (`v0.34.6`) — used to provision Yarn.
- **Yarn:** `4.0.2` (the repo's `packageManager` pin; provisioned by Corepack — do not install a different Yarn).
- **Git** + **Git LFS**.
- **Disk:** ~5 GB free (repository ≈ 4.2 GB plus `node_modules`).

### 9.2 Environment Setup

```bash
# From the repository root, on branch blitzy-86b269e0-17ba-4075-9310-6c345cfb19c5 (HEAD 9434faf066)
node --version          # expect v22.x (>= 22.9.0)
corepack enable         # provisions Yarn 4.0.2 from the packageManager pin
yarn --version          # expect 4.0.2
# TZ=UTC is applied automatically by the test-client script — no manual export needed
```

### 9.3 Dependency Installation

```bash
# Populates the git-ignored node_modules; does NOT modify any tracked file
yarn install
# Verify (should print a path, i.e. node_modules exists):
test -d node_modules && echo "node_modules OK"
```

*Expected:* `yarn.lock` and `package.json` remain unchanged (no drift); installation only creates the git-ignored `node_modules`.

### 9.4 Reproduce the Runtime Evidence (verification)

```bash
# (1) Run the existing reader-ui unit suite — expect 3 suites / 7 tests passing
CI=true yarn test-client state/reader-ui/test
```

*Expected tail:*

```
Test Suites: 3 passed, 3 total
Tests:       7 passed, 7 total
```

```bash
# (2) Reproduce the run-code-first observation:
#     - Recreate the temporary spec from Appendix §A.4 of the deliverable at:
#         client/state/reader-ui/test/blitzy_adhoc_test_persistence_observation.js
#       (it MUST live under a client/**/test/ path so Jest, rootDir=client/, discovers it)
#     - Then run:
CI=true yarn test-client state/reader-ui/test/blitzy_adhoc_test_persistence_observation
#     - Compare console output against the deliverable's Appendix §A.1 (byte-identical except the Time: line)
#     - Finally, DELETE the temporary spec to preserve the read-only invariant:
rm -f client/state/reader-ui/test/blitzy_adhoc_test_persistence_observation.js
```

*Expected:* `Tests: 2 passed, 2 total`, and the decisive console lines:

```
BEFORE  lastActionRequiresLogin = null
AFTER REGISTER lastActionRequiresLogin = { type: 'like', siteId: 123, postId: 456 }
SERIALIZED root has lastActionRequiresLogin key? = false
AFTER DESERIALIZE lastActionRequiresLogin = null
AFTER DESERIALIZE lastPath                = /reader/feeds/123
```

### 9.5 Verify the Read-Only Invariant

```bash
git status --porcelain                                             # expect: (empty) — clean tree
git diff be7e5cc641622d153040491fd5625c6cb83e12eb --name-status    # expect exactly: A  blitzy/documentation/wp-calypso_be7e5cc64162.md
```

### 9.6 Example Usage (reading the deliverable)

The document is the product; "usage" is answer lookup:

- **R1** (where the intent goes) → §3 · **R2** (replay mechanism) → §4 · **R3** (source of truth, runtime proof) → §5 + Appendix §A.1 · **R4** (exact skip condition) → §6 · **R5** (classification) → §7.
- Sibling requires-login intents → §8 · edge/transitional branches → §9 · decisive negatives (grep proofs) → §10 · end-to-end trace diagram → §11.

### 9.7 Troubleshooting

- **`yarn: command not found`** → run `corepack enable` (Yarn is provisioned from the `packageManager` pin).
- **Node version mismatch** → use Node `^22.9.0` (e.g., `nvm use`); older majors are not supported by the repo engines.
- **Jest doesn't discover the temp spec** → it must live under a `client/**/test/` path because the client config sets `rootDir: client/`.
- **`Time:` line differs run-to-run** → expected; it is the only volatile line. All other observation lines are deterministic.
- **Temp spec appears in `git status`** → delete it (`rm …`); it must never be committed (read-only invariant).
- **Browserslist "caniuse-lite is N months old" warning** → harmless; safe to ignore for this observation.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Node version | `node --version` |
| Enable Yarn (Corepack) | `corepack enable` |
| Yarn version | `yarn --version` |
| Install dependencies | `yarn install` |
| Run reader-ui suite | `CI=true yarn test-client state/reader-ui/test` |
| Run observation spec | `CI=true yarn test-client state/reader-ui/test/blitzy_adhoc_test_persistence_observation` |
| Delete temp spec | `rm -f client/state/reader-ui/test/blitzy_adhoc_test_persistence_observation.js` |
| Clean-tree check | `git status --porcelain` |
| Read-only diff check | `git diff be7e5cc641622d153040491fd5625c6cb83e12eb --name-status` |

### B. Port Reference

Not applicable — no server, service, or listening port is started. The investigation runs pure unit-level Jest observations only.

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | **The deliverable** (answer document). |
| `client/blocks/like-button/index.jsx` | Capture point — `handleLikeToggle` dispatches the pending intent (R1). |
| `client/state/reader-ui/reducer.js` | Intent store — plain `lastActionRequiresLogin` vs persisted `lastPath` (R3). |
| `client/state/reader-ui/{actions,selectors,action-types,init}.js` | Action creators, selector, constants, slice registration. |
| `client/layout/logged-out.jsx` | The only consumer — dialog + `onLoginSuccess` reload/redirect (R2/R4). |
| `client/blocks/reader-join-conversation/dialog.jsx` | Post-click dialog (analytics/visibility only). |
| `client/data/reader/use-login-window.ts` | WordPress.com login popup + `postMessage` success. |
| `client/state/utils/{serialize.ts,with-persistence.ts,reducer-utils.ts,index.ts}` | Persistence primitives (opt-in). |
| `test/client/jest.config.js` | Client Jest config (`rootDir: client/`, `TZ=UTC`, `calypso-jest`). |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Node.js (engine) | `^22.9.0` (sandbox `v22.23.1`) | `package.json` engines / `.nvmrc` |
| Yarn (package manager) | `4.0.2` | `package.json` `packageManager` |
| Corepack | `0.34.6` | Bundled with Node |
| Jest | `^29.7.0` | `package.json` devDependencies |
| `@automattic/calypso-jest` | `1.0.0` | Jest preset (workspace) |
| `@automattic/state-utils` | `1.0.0-alpha.4` | `withStorageKey` / `getInitialState` (workspace) |
| redux | `^5.0.1` | `package.json` dependencies |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `CI` | `true` | Forces non-interactive Jest (no watch mode). |
| `TZ` | `UTC` | Applied by the `test-client` script for deterministic time handling. |

### F. Developer Tools Guide

- **Client Jest runner** — `test-client` resolves to `TZ=UTC jest -c=test/client/jest.config.js`; pass a path/pattern to scope discovery (e.g., `state/reader-ui/test`).
- **Read-only diff verification** — `git diff <citation-commit> --name-status` confirms the source tree is unchanged apart from the single deliverable.
- **Determinism check** — run the observation spec twice and diff the outputs; only the `Time:` line should differ.

### G. Glossary

| Term | Definition |
|---|---|
| **Pending intent** | The logged-out action captured for post-login handling: `readerUi.lastActionRequiresLogin`. |
| **Opt-in persistence** | Calypso's `combineReducers` persists a reducer only if wrapped with `withPersistence` (or given a schema); plain reducers are session-only. |
| **`serialize` / `deserialize`** | State-utils primitives that write/read the persisted store blob; a reducer without a `.serialize` method yields `undefined` and rehydrates to its initial state. |
| **`redirectTo`** | Optional payload field; when present, `onLoginSuccess` navigates via `window.location = redirectTo` — only the `sidebar-link` intent sets it. |
| **Cleanup-by-teardown** | The root cause: a full-page `window.location.reload()` destroys the in-memory store, discarding the non-persisted intent. |
| **`[observed-at-runtime]` / `[inferred-from-reading]`** | Evidence labels: runtime-captured output vs. conclusions drawn from reading code at the pinned commit. |
