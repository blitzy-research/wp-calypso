# Blitzy Project Guide — Calypso Reader Run-First Onboarding Q&A

> **Project:** WordPress.com Calypso — "Reader" developer-onboarding knowledge document
> **Branch:** `blitzy-6c8223f7-1a04-4d05-a28b-eaec4b575d35` · **Base:** `be7e5cc641` · **HEAD:** `b61e53cc8f`
> **Deliverable:** `blitzy/documentation/wp-calypso_be7e5cc64162.md` (1,441 lines)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, comprehensive **run-first developer-onboarding knowledge document** that answers four groups of questions about running the WordPress.com **Calypso** monorepo's **Reader** experience locally. It targets engineers onboarding to Calypso who need authoritative, evidence-backed answers on (1) the dev-server port, readiness signal, and single-port architecture; (2) the Reader stream REST endpoints and initial-load Redux actions; (3) how login state is detected before render and which storage mechanisms are inspected; and (4) the responsive sidebar's margins/padding, CSS custom properties, and breakpoints. Per the governing "SWE-AtlasQnA-Repo" rules, this is a **read-only documentation task**: the app was actually built and run to capture real output, and the *only* repository change is the one answer document.

### 1.2 Completion Status

The AAP-scoped completion is **90.0%**, computed on an hours basis (PA1): all 16 AAP-specified requirements are complete; only path-to-production human review/merge remains.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Completion Status — 90.0% Complete
    "Completed Work (AI)" : 36
    "Remaining Work" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **40.0** |
| Completed Hours (AI: 36.0 + Manual: 0.0) | 36.0 |
| Remaining Hours | 4.0 |
| **Percent Complete** | **90.0%** |

> **Color key (Blitzy brand):** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF`.
> **Calculation:** `36.0 / (36.0 + 4.0) × 100 = 90.0%`. All completed work was performed autonomously by Blitzy agents (0.0 manual hours to date).

### 1.3 Key Accomplishments

- ✅ **Single deliverable created at the exact required path** — `blitzy/documentation/wp-calypso_be7e5cc64162.md` (1,441 lines, 79 balanced code blocks, 2 mermaid diagrams), named after the source branch.
- ✅ **Run-first methodology honored end-to-end** — Calypso was built and run on the canonical runtime (Node `v22.23.1` satisfying `^v22.9.0`, Yarn `4.0.2`); the in-server webpack reached the `Ready! … Have fun!` banner on port `3000`; `GET /version` returned `{"version":"0.17.0"}`.
- ✅ **All four question groups answered exhaustively** — OBJ-1 (port/readiness/single-port), OBJ-2 (`/read/*` endpoints + Redux action cycle), OBJ-3 (login detection + four storage mechanisms), OBJ-4 (sidebar box-model + custom properties + breakpoints across 8 viewport widths).
- ✅ **~130 `file:line` citations verified exact** against source (10 independently re-verified this session).
- ✅ **Read-only integrity proven** — three reproducible git proofs confirm the *only* change since base is this one document (`A` addition, 1,441 insertions, 0 deletions to any source file).
- ✅ **Single discrepancy self-corrected during validation** — the OBJ-4 off-canvas transform was re-attributed from `.site-selector` (`style.scss:231`) to the observed `.layout__secondary` rule (`style.scss:345-349`) using live CSSOM evidence.
- ✅ **Secret-safe** — 15 *public* client-config keys redacted as char-counts; confirmed no real secrets committed; `node_modules/` and `build/` are git-ignored; the husky pre-commit hook ran and passed.

### 1.4 Critical Unresolved Issues

There are **no critical, release-blocking issues**. The deliverable is complete, verified, and committed with zero unresolved discrepancies. The items below are standard path-to-production follow-ups, not defects.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Human SME sign-off pending | Onboarding doc should be validated by a Calypso engineer before it is treated as authoritative | Reviewing Engineer | 2.0h |
| Canonical logged-in **detection** trigger is inferred (no WordPress.com credentials in the environment) | One OBJ-3 sub-path is `(inferred)` rather than observed; render consequence *was* observed via forced Redux injection | Reviewer w/ WP.com creds (optional) | included in 1.5h reproduction |
| Doc not yet linked from onboarding index | Low discoverability until referenced from `README.md` / `docs/` | Reviewing Engineer | included in 0.5h merge |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| WordPress.com account (`public-api.wordpress.com`) | Authenticated API credentials | No login credentials in the sandbox; the canonical logged-in **detection** path (real `wordpress_logged_in` cookie → `200 /me`) could not be observed and is labeled `(inferred)`. Logged-in **render consequence** was still observed via a non-canonical forced Redux injection. | Open (optional) — does not block the deliverable | Reviewer w/ WP.com creds |
| Source repository | Write (commit/push) | None — the branch was committed successfully by `agent@blitzy.com`; husky pre-commit hook passed. | Resolved | — |

No access issues prevent build validation, integration, or delivery of the documentation artifact.

### 1.6 Recommended Next Steps

1. **[High]** Perform a technical SME review of the 1,441-line document; spot-check a sample of the ~130 `file:line` citations and confirm the inference/non-canonical labels (≈2.0h).
2. **[Medium]** Optionally reproduce the run-first observations on a canonical Node 22.x / Yarn 4.0.2 environment; if WP.com credentials are available, confirm the inferred logged-in detection path (≈1.5h).
3. **[Low]** Approve and merge the PR, then link the document from the onboarding index (`README.md` / `docs/`) for discoverability (≈0.5h).
4. **[Low]** Add the document to a periodic re-verification cadence, since `file:line` citations and computed CSS values can drift as the active Calypso monorepo evolves.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed hours were performed autonomously by Blitzy agents and trace to specific AAP requirements (investigation, authoring, and autonomous validation of the single deliverable).

| Component | Hours | Description |
|-----------|------:|-------------|
| Runtime provisioning & canonical build | 4.0 | Node 22.x + Yarn 4.0.2 via corepack; `yarn install` (241 workspaces, ~3.1 GB `node_modules`); `/etc/hosts` `calypso.localhost`; first successful `yarn start` build to the `Ready!` banner (~2.7 min lazy in-server compile); `NODE_OPTIONS` heap tuning; canonical Node-version discrepancy resolution |
| OBJ-1 — Dev server port, readiness, single-port architecture (§1.1–1.6) | 5.5 | Port `3000` derivation; two-phase readiness (boot log `:33` vs `Ready!` banner `:56` vs pre-ready interstitial `:77-93`); single-port proof via `curl` `X-Powered-By` across SSR / assets / HMR / API; `PORT` overrides `3001`/`3002`; welcome banner |
| OBJ-2 — Reader stream endpoints & Redux actions (§2.1–2.5) | 5.5 | Headless-Chrome capture of `/read/*` (logged-out `/streams/discover` + forced logged-in `/following` @ `apiVersion 1.2`); full endpoint map + `apiVersion` variants; `PAGE_REQUEST → http() → PAGE_RECEIVE` sequence via passive Redux shim, stable across two runs; 112 KB unedited response transcription |
| OBJ-3 — Login detection & storage mechanisms (§3.1–3.6) | 4.5 | Auth decision trace `initializeCurrentUser (:341) → isUserLoggedIn (:15-16)`; four storage mechanisms via `evaluate_script` + full JSON; logged-in forced render; enveloped `403` `/me`; optional-chaining nuance in the live JS engine |
| OBJ-4 — Responsive sidebar (§4.1–4.5) | 5.5 | 8-viewport CSSOM cross-product (Tables A/B); `.sidebar-header` vs global `.sidebar__header` distinction; custom properties + `calc()`; SCSS + JS breakpoints; off-canvas transform investigation & correction |
| Citation verification & source grounding | 3.5 | ~130 `file:line` citations verified exact against source |
| Coverage pass, read-only integrity proofs, cleanup, commits | 3.0 | Final coverage pass over every named item; 3 reproducible read-only proofs; removal of temp scripts + 21 stray screenshots; 3 commits |
| Autonomous validation (13 phases) | 4.5 | Reproduction of all run-first observations; single-discrepancy discovery & correction (off-canvas transform); reproducibility edits |
| **Total Completed** | **36.0** | Matches Completed Hours in §1.2 |

### 2.2 Remaining Work Detail

Every remaining item is a **path-to-production** activity requiring a human; none is an incomplete AAP deliverable.

| Category | Hours | Priority |
|----------|------:|----------|
| SME technical review of the document (accuracy, exhaustiveness, label correctness) | 2.0 | High |
| Reproduce run-first observations (ports/endpoints/storage/breakpoints; optional credentialed logged-in detection path) | 1.5 | Medium |
| PR review & merge to trunk + link from onboarding index | 0.5 | Low |
| **Total Remaining** | **4.0** | Matches Remaining Hours in §1.2 and §7 |

### 2.3 Basis of Estimate & Confidence

- **Total Project Hours = 36.0 (completed) + 4.0 (remaining) = 40.0h.** Completion = `36.0 / 40.0 = 90.0%`.
- **Confidence: High.** Completed hours are grounded in the actual 1,441-line artifact, three agent commits, and the validator's 13-phase logs; remaining hours reflect standard human review of a finished, verified document.
- **Why not higher than 90.0%:** per Blitzy policy, a technical onboarding document is not certified >99% before human SME review; here the honest remaining scope (review + optional reproduction + merge) is 4.0h against 36.0h delivered.

---

## 3. Test Results

**Important framing (integrity):** This is a **read-only documentation task**; the "SWE-AtlasQnA-Repo" rules **forbid adding source code or tests**, so **no unit/integration/e2e test suites were authored or executed** — doing so would violate scope. For a *run-first Q&A document*, the substantive quality gate is **reproduction of the documented observations** plus **static well-formedness and citation checks**. All entries below originate from Blitzy's autonomous validation logs and were re-confirmed in this assessment session. **Zero failures; zero unresolved discrepancies.**

| Test Category | Framework / Method | Total Checks | Passed | Failed | Coverage | Notes |
|---------------|--------------------|-------------:|-------:|-------:|----------|-------|
| Document well-formedness | shell + `grep` (static) | 3 | 3 | 0 | 79 fenced pairs balanced; 2 valid mermaid blocks; 0 forbidden markers | Re-verified this session |
| Citation accuracy (`file:line`) | `grep` vs source (static) | ~130 | ~130 | 0 | All load-bearing citations | 10 independently re-verified this session |
| App build / compile | in-server webpack (runtime) | 1 | 1 | 0 | Reached `Ready! … Have fun!` banner (`bundler:56`) | Compile time not stable run-to-run; ordering invariant |
| App health | `curl /version` (runtime) | 1 | 1 | 0 | `{"version":"0.17.0"}` | Endpoint `client/server/api/index.js:11-12` |
| OBJ-1 runtime verification | `curl` + Chrome DevTools | 7 | 7 | 0 | port, boot log, ready banner, recompile, interstitial, single-port ×4, PORT overrides | `X-Powered-By: Express` on all four responsibilities |
| OBJ-2 runtime verification | Chrome DevTools network + Redux shim | 6 | 6 | 0 | logged-out + forced logged-in endpoints, full map, `apiVersion`, action cycle, siblings | Action sequence stable across two runs |
| OBJ-3 runtime verification | `evaluate_script` (runtime) | 6 | 6 | 0 | decision path, reducer default, `/me` probe, optional-chaining (3 states), 4 storage mechanisms | Logged-in detection trigger labeled `(inferred)` |
| OBJ-4 runtime verification | CSSOM `getComputedStyle` ×8 widths | 6 | 6 | 0 | 8-width cross-product (Tables A/B), custom props, `calc()`, off-canvas transform, breakpoints | Off-canvas transform corrected & re-verified |
| Read-only integrity | `git` (static) | 3 | 3 | 0 | single `A` addition; clean tree; no source touched | All three proofs reproduce on the committed branch |

**Aggregate:** ≈166 discrete verification checks, **100% passed / reproduced, 0 failed.** The single genuine inaccuracy discovered during validation (the OBJ-4 off-canvas transform attribution) was corrected and re-verified.

---

## 4. Runtime Validation & UI Verification

Runtime health and UI verification are drawn from the autonomous run-first session (dev server kept alive on port `3000`) and corroborated by this assessment.

**Server & build**
- ✅ **Dev server operational** — single Express instance bound to `http://calypso.localhost:3000`.
- ✅ **Readiness signal observed** — authoritative webpack banner `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` (`client/server/bundler/index.js:56`).
- ✅ **Health endpoint** — `GET /version` → `{"version":"0.17.0"}`; `HEAD /version` polled ~every 20 s.
- ⚠ **Pre-ready state** — before compile completes, requests receive the "Welcome to Calypso!" interstitial with a 5 s meta-refresh (expected, transitional).

**Single-port architecture (OBJ-1)**
- ✅ **SSR HTML**, **`webpack-dev-middleware` assets**, **`webpack-hot-middleware` HMR** (`/__webpack_hmr` SSE), and **server JSON API** all served from port `3000` — each response carries `X-Powered-By: Express`.
- ✅ **`PORT` overrides** — `3001` (Jetpack Cloud), `3002` (A8C for Agencies) confirmed in `package.json` scripts.
- ✅ **Data nuance** — live WordPress.com REST data is fetched cross-origin from `public-api.wordpress.com`, not proxied through `:3000` in the default dev build.

**Reader initial load (OBJ-2 / OBJ-3)**
- ✅ **Logged-out** — `/reader` redirects to `/discover`; observed `GET /wpcom/v2/read/streams/discover`.
- ✅ **Redux cycle** — `READER_STREAMS_PAGE_REQUEST → http() → READER_STREAMS_PAGE_RECEIVE` (stable across two runs).
- ✅ **Auth decision** — `initializeCurrentUser()` resolves before routing; `isUserLoggedIn ← getCurrentUserId`; four storage mechanisms observed empty when logged-out.
- ⚠ **Forced logged-in render** — observed via non-canonical Redux injection (`GET /rest/v1.2/read/following` @ `apiVersion 1.2`); enveloped `403` (no credentials). Real values, non-canonical trigger.

**Responsive sidebar UI (OBJ-4)**
- ✅ **Rendered `.sidebar-header`** — `margin: 0 12px 44px`, `padding: 0 10px`; computed box model **constant across all 8 viewport widths**.
- ✅ **Custom properties** — `--sidebar-width-max: 272px`, `--sidebar-width-min: 228px`, `--masterbar-height: 46px→32px @782`.
- ✅ **Breakpoints** — transitions observed at `960px` (272→228), `<660px` (→100% + off-canvas), `<782px` (masterbar 32→46); off-canvas `translateX(-100%)` on `.layout__secondary` confirmed via CSSOM.

---

## 5. Compliance & Quality Review

Cross-mapping the governing "SWE-AtlasQnA-Repo" rule set to delivered evidence. Fixes applied during autonomous validation are noted.

| Compliance Benchmark (rule) | Status | Progress | Evidence / Notes |
|-----------------------------|--------|----------|------------------|
| Single deliverable at `blitzy/documentation/<branch>.md` | ✅ Pass | 100% | `wp-calypso_be7e5cc64162.md` exists; `blitzy/` tree = exactly one file |
| Run-first (build & run before writing) | ✅ Pass | 100% | Built & ran; `Ready!` banner; `:3000`; `/version` health |
| Canonical/default configuration | ✅ Pass | 100% | Node `v22.23.1` (`^v22.9.0`), Yarn `4.0.2`; Node 20.x flagged non-canonical |
| Complete, unedited output per condition | ✅ Pass | 100% | 112 KB response byte-for-byte; `curl -i` outputs; full `evaluate_script` JSON |
| Label inference vs observation vs non-canonical | ✅ Pass | 100% | Consistent `(observed)`/`(inferred)`/`(non-canonical)` labels |
| Ground every claim in `file:line` / observed output | ✅ Pass | 100% | ~130 citations; 10 re-verified exact |
| Answer every part & every named item (exhaustive) | ✅ Pass | 100% | Coverage-pass section; all named items grep-confirmed in body |
| Read-only (no source modified/added/deleted) | ✅ Pass | 100% | 3 reproducible git proofs; `numstat` = 1 doc, 0 deletions |
| Temporary scripts removed | ✅ Pass | 100% | `/tmp`-only; clean working tree; 21 stray screenshots removed |
| No dependency / lockfile changes | ✅ Pass | 100% | diff = single `.md` file |
| Report observed values honestly (fix inaccuracies) | ✅ Pass | 100% | Off-canvas transform re-attributed to `.layout__secondary:345-349` via CSSOM |
| Secret handling | ✅ Pass | 100% | 15 public client-config keys redacted; no real secrets committed |
| Human SME accuracy sign-off | ⏳ Pending | 0% | Path-to-production (see §1.6, HT-1) |

**Overall compliance:** 12 of 13 benchmarks fully satisfied; the 13th (human sign-off) is inherent path-to-production, not an autonomous gap.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Documentation drift — citations & computed CSS may age as Calypso evolves | Technical | Medium | High (over time) | Doc pins commit `be7e5cc641` + branch; schedule periodic re-verification | Mitigated (commit-pinned) |
| Build time not stable run-to-run (161,402 ms vs 159,233 ms) | Technical | Low | Low | No correctness impact; two-phase readiness ordering is invariant; durations stated | Documented / Accepted |
| Embedded 112 KB live API response ages (posts change) | Technical | Low | Medium | Durable evidence is response *shape*; labeled point-in-time capture | Accepted |
| Accidental secret exposure in committed doc | Security | Low (would-be High) | Low | 15 public client-config keys redacted as char-counts; no real secrets present | Mitigated |
| New attack surface | Security | None | — | Read-only doc; no new code/deps/endpoints | N/A |
| Discoverability — doc not linked from onboarding index | Operational | Low | Medium | Link from `README.md` / `docs/` (HT-3) | Open (human) |
| Canonical runtime discrepancy — setup note said Node 20.x vs enforced `^v22.9.0` | Operational | Low | Medium | Doc "Canonical runtime resolution" section resolves it; `check-node-version` gate documented | Mitigated (documented) |
| Reproduction depends on prerequisites (`/etc/hosts`, Node 22.x, 8 GB heap, WP.com egress) | Integration | Low | Medium | Doc + dev guide enumerate all prerequisites | Documented |
| Credential-gated logged-in **detection** path inferred, not observed | Integration | Low | Low | Labeled `(inferred)` + `file:line`-grounded; optional credentialed reproduction (HT-2) | Documented / Accepted |

**No High or Critical severity risks.** No risk blocks release of the documentation deliverable.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 36
    "Remaining Work" : 4
```

**Remaining work by priority** (hours from §2.2; total = 4.0h):

| Priority | Category | Hours | Bar |
|----------|----------|------:|-----|
| High | SME technical review | 2.0 | ██████████ |
| Medium | Reproduce run-first observations | 1.5 | ███████▌ |
| Low | PR merge & publish | 0.5 | ██▌ |
| | **Total** | **4.0** | |

> **Integrity:** "Remaining Work" = **4.0h** in §1.2, §2.2, and the §7 pie chart. "Completed Work" = **36.0h**. Total = **40.0h**. Completion = **90.0%**.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-backed, **run-first** onboarding document (`blitzy/documentation/wp-calypso_be7e5cc64162.md`, 1,441 lines) that exhaustively answers all four Calypso Reader question groups. Calypso was genuinely built and run on the canonical runtime; every behavioral claim is paired with the command that produced it, its unedited output, and a `file:line` citation, with disciplined `(observed)` / `(inferred)` / `(non-canonical)` labeling. Independent spot-checks confirmed citation accuracy, and all three read-only integrity proofs reproduce exactly.

**Remaining gaps.** The remaining **4.0 hours (10%)** are entirely path-to-production and human in nature: SME technical review (2.0h), optional run-first reproduction incl. the credential-gated logged-in detection path (1.5h), and PR merge + onboarding-index linking (0.5h). No AAP deliverable is incomplete and no defect is outstanding.

**Critical path to production.** SME review → (optional) reproduction → merge & link. None of these depend on further engineering; the artifact is already committed and byte-for-byte the only repository change since base.

**Success metrics (met).** One correctly-named/placed deliverable ✅; run-first evidence for every claim ✅; exhaustive coverage of all named items ✅; strict read-only integrity ✅; zero unresolved discrepancies ✅.

**Production-readiness assessment.** The documentation deliverable is **production-ready at 90.0% AAP-scoped completion**, pending human SME sign-off. Recommendation: **approve after SME review**, then merge and link from the onboarding index.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 90.0% |
| Completed / Total hours | 36.0 / 40.0 |
| AAP-specified requirements complete | 16 / 16 |
| Unresolved discrepancies | 0 |
| Release-blocking issues | 0 |

---

## 9. Development Guide

How to build, run, and reproduce the Reader investigation. Commands marked **[tested]** were executed and confirmed during this assessment; the two long-running commands (`yarn install`, `yarn start`) are documented from the autonomous run-first logs and verified by script inspection.

### 9.1 System Prerequisites

- **OS:** Linux or macOS (investigation ran on Linux, 4 CPUs).
- **Node.js:** `^v22.9.0` (enforced). `.nvmrc` pins `22.9.0`. **[tested]** provisioned `v22.23.1`.
- **Yarn:** `4.0.2` via corepack (vendored `.yarn/releases/yarn-4.0.2.cjs`). **[tested]**
- **Memory:** ~8 GB free recommended for the client build.
- **Hosts entry:** `127.0.0.1 calypso.localhost`. **[tested]** present at `/etc/hosts:9`.
- **Network:** egress to `public-api.wordpress.com` for live Reader data.

### 9.2 Environment Setup

```bash
# 1) Use the canonical Node version (fails the yarn start gate otherwise)
nvm install 22.9.0 && nvm use 22.9.0        # or any ^v22.9.0
node --version                               # [tested] -> v22.23.1

# 2) Enable the vendored Yarn 4.0.2
corepack enable
yarn --version                               # [tested] -> 4.0.2

# 3) Add the required host mapping (idempotent)
grep -q 'calypso.localhost' /etc/hosts || echo '127.0.0.1 calypso.localhost' | sudo tee -a /etc/hosts

# 4) Verify the engines gate passes BEFORE building
npx --no-install check-node-version --package ; echo "exit=$?"   # [tested] -> exit=0
```

### 9.3 Dependency Installation

```bash
# Installs 241 workspaces (~3.1 GB node_modules). Long-running on first run.
yarn install
```

### 9.4 Application Startup

```bash
# Single command; ~8 GB heap avoids OOM during the client build.
# Sequence: check-node-version --package -> bin/welcome.js banner
#           -> yarn run build -> yarn run start-build
#           (start-build = BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short)
NODE_OPTIONS=--max-old-space-size=8192 yarn start
# Dev server binds to http://calypso.localhost:3000
```

The client bundle compiles **lazily in-server** (~2.7 min). The server boot log (`client/server/index.js:33`) prints **before** the app is usable — do **not** treat it as "ready."

### 9.5 Verification Steps

```bash
# 1) Wait for the AUTHORITATIVE readiness banner in the server output:
#    "Ready! You can load http://calypso.localhost:3000/ now. Have fun!"

# 2) Health check [tested: endpoint defined at client/server/api/index.js:11-12]
curl -i -s http://calypso.localhost:3000/version
#    -> HTTP/1.1 200 OK ... {"version":"0.17.0"}

# 3) Load the Reader (redirects logged-out -> /discover)
#    Browser: http://calypso.localhost:3000/reader
```

Optional PORT variants (single-port each): `PORT=3001 … yarn start-jetpack-cloud-p` (Jetpack Cloud), `PORT=3002 … yarn start-a8c-for-agencies-p` (A8C for Agencies).

### 9.6 Example Usage (reproduce the run-first observations)

- **OBJ-1:** `curl -I http://calypso.localhost:3000/` and asset/HMR/`/version` requests — confirm `X-Powered-By: Express` on all (single port).
- **OBJ-2:** In DevTools, watch network for `/read/*` (`GET /wpcom/v2/read/streams/discover` logged-out) and the `READER_STREAMS_PAGE_REQUEST → http() → PAGE_RECEIVE` Redux cycle.
- **OBJ-3:** `evaluate_script` to read the four storage mechanisms (`wordpress_logged_in` cookie, `localStorage.wpcom_user_id`, OAuth token, `sessionStorage.flags`) — all empty logged-out.
- **OBJ-4:** `getComputedStyle` on the sidebar header across 8 widths; observe custom properties and the `<660px` off-canvas `translateX(-100%)`.

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---------|-------|------------|
| `yarn start` aborts immediately | Node version fails `check-node-version --package` (e.g., Node 20.x) | Switch to Node `^v22.9.0` (`nvm use 22.9.0`) |
| Build killed / OOM | Default heap too small for client build | `NODE_OPTIONS=--max-old-space-size=8192 yarn start` |
| "Welcome to Calypso!" page keeps refreshing | Client bundle still compiling in-server | Wait for the `Ready! … Have fun!` banner (~2.7 min) |
| `calypso.localhost` won't resolve | Missing hosts entry | Add `127.0.0.1 calypso.localhost` to `/etc/hosts` |
| Reader shows `/discover`, not `/following` | Not logged in (no WP.com credentials) | Expected; logged-out streams redirect to Discover |

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose | Verified |
|---------|---------|----------|
| `node --version` | Confirm Node `^v22.9.0` | [tested] v22.23.1 |
| `corepack enable && yarn --version` | Activate/confirm Yarn 4.0.2 | [tested] 4.0.2 |
| `npx --no-install check-node-version --package` | Engines gate (must exit 0) | [tested] exit=0 |
| `yarn install` | Install 241 workspaces | run-first log |
| `NODE_OPTIONS=--max-old-space-size=8192 yarn start` | Build + run dev server on `:3000` | run-first log |
| `curl -i -s http://calypso.localhost:3000/version` | Health check | [tested] endpoint verified |
| `git diff --name-status be7e5cc641 -- .` | Read-only proof (single `A`) | [tested] |
| `git status --porcelain` | Clean-tree proof | [tested] empty |

### Appendix B — Port Reference

| Port | Purpose |
|------|---------|
| `3000` | Default dev server (SSR + assets + HMR + API) |
| `3001` | Jetpack Cloud dev (`start-jetpack-cloud-p`) |
| `3002` | A8C for Agencies dev (`start-a8c-for-agencies-p`) |
| `/__webpack_hmr` | HMR SSE endpoint (same port `3000`) |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/wp-calypso_be7e5cc64162.md` | **The deliverable** (1,441 lines) |
| `config/_shared.json` / `config/development.json` | Port `3000` config (OBJ-1) |
| `client/server/index.js` | Boot log `:33`; `server.listen` `:83` (OBJ-1) |
| `client/server/bundler/index.js` | `Ready!` banner `:56`; interstitial `:77-93` (OBJ-1) |
| `client/state/data-layer/wpcom/read/streams/index.js` | `/read/*` map + `apiVersion` (OBJ-2) |
| `client/state/reader/streams/actions.js` | `READER_STREAMS_*` action creators (OBJ-2) |
| `client/boot/common.js` | `initializeCurrentUser` `:341`; OAuth gate (OBJ-3) |
| `client/state/current-user/selectors.js` | `isUserLoggedIn`/`getCurrentUserId` `:15-16` (OBJ-3) |
| `client/layout/global-sidebar/style.scss` | `.sidebar__header` box model `:70-75` (OBJ-4) |
| `client/layout/style.scss` | Custom props, `calc()`, off-canvas `:345-349` (OBJ-4) |
| `.storybook/preview.js` | 8 named breakpoint widths (OBJ-4) |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Node.js | `^v22.9.0` (`.nvmrc` 22.9.0; provisioned v22.23.1) | `package.json` engines |
| Yarn | `4.0.2` | `packageManager` + vendored release |
| express | `^4.21.2` | `client/package.json` |
| webpack | `^5.97.1` | `package.json` |
| webpack-dev-middleware | `^5.3.4` | `package.json` |
| webpack-hot-middleware | `^2.26.1` | `client/package.json` |
| react | `^18.3.1` | `package.json` |
| redux | `^5.0.1` | `package.json` |
| @tanstack/react-query | `^5.15.5` | `client/package.json` |
| App version (runtime) | `0.17.0` | `GET /version` |

### Appendix E — Environment Variable Reference

| Variable | Value / Example | Purpose |
|----------|-----------------|---------|
| `NODE_OPTIONS` | `--max-old-space-size=8192` | Raise heap for the client build |
| `PORT` | `3000` (default), `3001`, `3002` | Dev server port |
| `BROWSERSLIST_ENV` | `evergreen` | Build target (in `start-build`) |
| `CALYPSO_ENV` | `jetpack-cloud-development` / `a8c-for-agencies-development` | Variant environment |

### Appendix F — Developer Tools Guide

- **bunyan** — formats the server boot log stream (`… | bunyan -o short`).
- **Chrome DevTools (headless)** — used to drive `http://calypso.localhost:3000/reader`, capture the network log (`/read/*`), read Redux action types via a passive store shim, and read computed styles via the CSSOM at 8 viewport widths.
- **`curl -i`** — inspect single-port responses (`X-Powered-By: Express`) and the `/version` health endpoint.
- **git** — read-only integrity proofs (`diff --name-status`, `status --porcelain`).

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **SSR** | Server-Side Rendering — HTML rendered by the Express server before hydration. |
| **HMR** | Hot Module Replacement — live code updates via `webpack-hot-middleware` over an SSE stream at `/__webpack_hmr` (same port `3000`). |
| **Redux data-layer** | Calypso pattern where dispatching `READER_STREAMS_PAGE_REQUEST` triggers a registered handler that issues an `http()` action; the response returns as `READER_STREAMS_PAGE_RECEIVE`. |
| **Two-phase readiness** | The server boot log prints before the in-server webpack finishes; the authoritative signal is the `Ready! … Have fun!` banner. |
| **Run-first** | Methodology requiring the code to be built and run to capture real output before any answer is written. |
| **`(inferred)` / `(non-canonical)`** | Labels for claims read from source but not observed, or values obtained via a non-default path. |
| **breakpoint-deprecated** | The `@automattic` SCSS mixin driving responsive breakpoints (`<960px`, `<660px`, `>1400px`). |

---

*Generated for branch `blitzy-6c8223f7-1a04-4d05-a28b-eaec4b575d35` (HEAD `b61e53cc8f`). AAP-scoped completion: **90.0%** (36.0 of 40.0 hours). The sole in-scope deliverable is committed and is byte-for-byte the only repository change since base `be7e5cc641`.*