# Why the `client/state/data-layer` Jest Suite Has Inconsistent (Cold‑vs‑Warm) Execution Times

> **Investigation report** — branch `wp-calypso_be7e5cc64162`
>
> This document explains **why** the Jest test suite for the `client/state/data-layer` module of the
> `wp-calypso` monorepo runs with inconsistent timings: the **first ("cold") run of a given test is
> markedly slower than the immediately following ("warm") run**. It answers four numbered questions
> (Q1–Q4) with **empirical measurements I took against this exact repository**, **exact `path:line`
> citations** to the source of truth, and the **rationale** behind every conclusion.
>
> This was a strictly **read‑only** investigation. The only file written anywhere is this report.
> Running the tests only regenerates the **git‑ignored** `/.cache/` directory (`.gitignore:L15`), which
> is not a tracked change — verified with `git status --porcelain` (clean) at the end.

## TL;DR — the root cause in one paragraph

Jest uses `babel-jest` to transform every project source file in a test's transitive import graph
**on demand**, and it **caches** the transformed output under `cacheDirectory`
(`test/client/jest.config.js:L7` → `<repo>/.cache/jest`). The repository's custom resolver prefers the
`calypso:src` field (`packages/calypso-jest/src/module-resolver.js:L18`), which points at **untranspiled
source** (`module-resolver.js:L8-L10`), so the cold run must Babel‑transform a potentially **large
transitive graph of workspace source** before the test can execute. The warm run **reuses the transform
cache** and skips that work. Therefore the cold/warm penalty **scales with the size of the import graph**
— from ≈1.6× for a leaf test (44 modules) to ≈2.7× for a heavy test (1,432 modules). The HTTP mock
library `nock` lives in `node_modules`, is excluded by `transformIgnorePatterns`
(`test/client/jest.config.js:L14-L16`), is **never transformed or cached**, and is therefore **not** the
cause. A cache‑decomposition experiment confirms **`babel-jest` transformation is the dominant first‑run
step** — larger than the `jest-haste-map` rebuild, and far larger than the (zero‑work) asset transformer.

---

## 1. Methodology

All numbers in this report are **measured**, not assumed. This section documents exactly how, so the
results are reproducible and meaningful.

### 1.1 Runtime & toolchain

| Component | Version | Source of truth (citation) |
|-----------|---------|----------------------------|
| Node.js | `v22.23.1` (satisfies `^v22.9.0`) | `.nvmrc` (`22.9.0`); `package.json:L56-L59` (`engines.node`) |
| Yarn | `4.0.2` | `package.json:L422` (`"packageManager": "yarn@4.0.2"`) |
| Jest | `29.7.0` (manifest `^29.7.0`) | `package.json:L290` |
| `babel-jest` | `29.7.0` (bundled with Jest; the default transformer) | `packages/calypso-jest/jest-preset.js:L14` (`transform`) |
| `@babel/core` | `7.26.10` (manifest `^7.26.10`) | `package.json:L242` |
| `enhanced-resolve` | `5.9.3` | `package.json:L213` |
| `nock` | `13.5.6` (manifest `^13.5.6`) | `package.json:L299` |

`node_modules` was already installed (≈3.1 GB) and the dependency tree is immutable; nothing tracked was
mutated during the investigation.

### 1.2 Runner command

The project's own client runner is `"test-client": "TZ=UTC jest -c=test/client/jest.config.js"`
(`package.json:L122`). For a single file I invoked Jest directly with the same config:

```bash
CI=true TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --watchAll=false <test-path>
```

| Flag | Why |
|------|-----|
| `CI=true` | Deterministic, non‑interactive; disables watch and interactive prompts. |
| `TZ=UTC` | Stable timezone so date‑sensitive code behaves identically across runs. |
| `--watchAll=false` | Run once and exit (no watch mode). |

### 1.3 Timing method

For each scenario I captured **wall‑clock** seconds (shell `date +%s.%N` around the process) **and** Jest's
**self‑reported** `Time:` from its summary output (quoted in parentheses where shown). Wall‑clock is the
honest end‑to‑end cost (process spin‑up + transform + run + teardown); the Jest‑reported figure is a
useful cross‑check. For the fine‑grained decomposition (Q4) I ran each cache state **3×** and report the
**minimum** wall‑clock, which is the least‑contaminated estimate on a shared container.

### 1.4 Cold‑state guarantee

`cacheDirectory` resolves to `<repo>/.cache/jest` (`test/client/jest.config.js:L7`), and `/.cache/` is
git‑ignored (`.gitignore:L15`). I therefore define:

- **cold** = no `.cache/jest` present → I run `rm -rf .cache` immediately before the "first" run;
- **warm** = the immediately following identical run, with the cache fully populated.

Because the cache is git‑ignored, a fresh checkout (or an `rm -rf .cache`) is **always** cold — this is
precisely why a developer's first invocation after checkout/CI feels slow.

### 1.5 Test subjects

Three `data-layer` tests were chosen to span import‑graph sizes (modules transformed = count of non‑`.map`
files written to the transform cache after a cache‑cleared cold run):

| Test file | Role | Modules transformed (measured) |
|-----------|------|-------------------------------:|
| `client/state/data-layer/wpcom/meta/sms-country-codes/test/index.js` | pure action‑creator **leaf**, no `nock` | **44** |
| `client/state/data-layer/wpcom-http/test/index.js` | **primary**; uses `nock` directly (`:L2`, `:L27`, `:L32`) | **52** |
| `client/state/data-layer/wpcom/jetpack-install/test/index.js` | **heavy**; imports `calypso/state/jetpack-remote-install/actions` (`:L5`) → large Calypso state tree | **1,432** |

---

## 2. Q1 — Cold‑vs‑warm timing ratio

**Task.** Run a data‑layer test twice in succession and report `ratio = t(cold) / t(warm)`.
**Method.** `rm -rf .cache` → run #1 (cold) → run #2 immediately (warm), for each subject.

**Measured results** (wall‑clock; Jest‑reported `Time:` in parentheses):

| Test file | Modules | Cold run | Warm run | Ratio (cold/warm) |
|-----------|--------:|----------|----------|-------------------|
| `…/wpcom/meta/sms-country-codes/test/index.js` | 44 | **2.669 s** (1.731 s) | **1.579 s** (1.022 s) | **≈ 1.69×** |
| `…/wpcom-http/test/index.js` | 52 | **2.498 s** (1.517 s) | **1.578 s** (0.909 s) | **≈ 1.58×** (Jest‑reported ≈ 1.67×) |
| `…/wpcom/jetpack-install/test/index.js` | 1,432 | **14.373 s** (13.354 s) | **5.277 s** (4.697 s) | **≈ 2.72×** (Jest‑reported ≈ 2.84×) |

Equivalently, the **absolute one‑time cold overhead** (cold − warm, wall‑clock) is ≈ **1.09 s**, **0.92 s**,
and **9.10 s** respectively.

**Rationale.** The cold run pays a **one‑time** cost the warm run does not: (a) crawling the file system to
build the `jest-haste-map`, and (b) **Babel‑transforming every source module in the test's transitive
import graph**. Jest caches each transformation and only re‑runs it if the source (or config) changes, so
the warm run **reads compiled output from cache** instead of re‑transforming. Consequently the warm‑up
penalty **scales with the size of the transitive import graph**: a leaf test (44 modules) pays only ≈0.9–1.1 s
of cold overhead, whereas `jetpack-install` (1,432 modules — it drags in the heavy Calypso state tree via
`calypso/state/jetpack-remote-install/actions`, `client/state/data-layer/wpcom/jetpack-install/test/index.js:L5`)
pays ≈9 s. The two small tests land at a similar ≈1.6× ratio because their graphs are similar in size and the
fixed Jest start‑up (~1.5 s) dominates their ratio; the heavy test jumps to ≈2.7× because its transform work
dwarfs that fixed baseline. **This graph‑size dependence is the direct, empirical reason the suite shows
"inconsistent" cold‑vs‑warm timings**: the inconsistency is not random — it tracks how much source each test
transitively imports.

---

## 3. Q2 — Where the cache is configured, the controlling option, and what is cached

### 3.1 The controlling option and directory

- **Controlling option:** **`cacheDirectory`**.
- **Configured at:** `test/client/jest.config.js:L7`:

  ```js
  cacheDirectory: path.join( __dirname, '../../.cache/jest' ),
  ```

  `__dirname` is `test/client/`, so `../../` is the repository root and the cache resolves to
  **`<repo>/.cache/jest`**. That directory is git‑ignored via `.gitignore:L15` (`/.cache/`), confirming the
  cache is a transient build artifact.

The pipeline that *produces* the cached output is the shared preset's **`transform`** map
(`packages/calypso-jest/jest-preset.js:L13-L16`), inherited by the client config via `{ ...base }`:

```js
transform: {
  '\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
  '\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
},
```

(`testEnvironment: 'node'` at `jest-preset.js:L11`; the custom `resolver` at `jest-preset.js:L9`.)

### 3.2 What is cached — exactly three artifact types

Listing the live directory after a run (`ls -la .cache/jest`, `find .cache/jest -maxdepth 2`) shows it
contains **exactly three artifact types**:

| Artifact (on disk) | Form | Measured evidence | Purpose |
|--------------------|------|-------------------|---------|
| `haste-map-<hash>` | single **V8‑serialized binary** file | `file` → `data`; size **2,684,319 bytes (≈ 2.68 MB)**; first bytes `\377\017` (`0xFF 0x0F` = V8 serialization header) | Jest's `jest-haste-map`: the file‑crawler/dependency map of the entire `rootDir`, used to reduce start‑up time. |
| `jest-transform-cache-<hash>/` | **directory sharded by two‑char hex prefixes** | **255** shard dirs; e.g. for `jetpack-install`, **1,432** code files + **1,325** `.map` siblings | The `babel-jest`‑transpiled **CommonJS** output for each transformed module — the bulk of the reuse. |
| `perf-cache-<hash>` | small **JSON** file | valid JSON dict; e.g. `{ "<abs>/wpcom-http/test/index.js": [1, 867] }` (and `[1, 4653]` for `jetpack-install`) | Maps each test path → runtime metrics (the second value ≈ runtime in ms), used for **worker scheduling**. |

**Anatomy of a cached transform.** Each cached code file begins with a **cache‑key hash line**, then the
transpiled module. For example, the head of one cached file (`…/7b/isakismetenterprise15k_…`):

```text
c36cf2a80d2a1054caa4e2d83ff63186          ← cache‑key hash (integrity/validity check)
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
```

Alongside it is a sibling **`.map`** source map (JSON, `"version":3`) whose `sources` reference the original
file (here, `is-akismet-enterprise-15k.ts` — note Jest also transpiles **TypeScript** via Babel). Of the
1,432 cached modules, **478** reference `@babel/runtime/helpers …` in their transpiled output (Babel's
runtime helpers for `_interopRequireDefault`, etc.). The code‑file count (1,432) slightly exceeds the
`.map` count (1,325) because a minority of transformed inputs (e.g., JSON) emit no source map.

**Rationale.** `cacheDirectory` stores three distinct things — the **haste‑map** (dependency/module graph),
the **transform cache** (compiled module output, which is the bulk of the reuse), and **perf data**
(scheduling). The warm run is fast precisely because it **reads the transform cache** instead of re‑running
Babel for every module; the haste‑map likewise lets Jest skip re‑crawling the tree.


---

## 4. Q3 — The HTTP mocking library, where it is wired, and its timing impact

### 4.1 The library

The mocking library is **`nock`** `^13.5.6` (`package.json:L299`; installed `13.5.6`). It resolves to
`node_modules/nock/index.js`.

### 4.2 Where it is configured (global bootstrap)

Global network isolation is set up in **`test/client/setup-test-framework.js`**, which Jest loads through
**`setupFilesAfterEnv`** at `test/client/jest.config.js:L21`
(`'<rootDir>/../test/client/setup-test-framework.js'`):

```js
const nock = require( 'nock' );          // L6
// Disables all network requests for all tests.
nock.disableNetConnect();                // L9
beforeAll( () => {                        // L11
  if ( ! nock.isActive() ) {              // L13
    nock.activate();                      // L14
  }
} );
afterAll( () => {                         // L18
  nock.restore();                         // L20
  nock.cleanAll();                        // L21
} );
```

- `:L9` `nock.disableNetConnect()` — disables **all** real network for every test (network isolation).
- `:L11-L16` `beforeAll` re‑activates nock if inactive.
- `:L18-L22` `afterAll` restores and cleans nock (avoids memory leaks across runs).

### 4.3 Where it is configured (per‑test helper)

Per‑test request stubbing goes through **`client/test-helpers/use-nock/index.js`**:

```js
import nock from 'nock';        // L2
export { nock };                // L4
/**
 * @deprecated Use nock directly instead.   // L10
 */
export const useNock = ( setupCallback ) => { /* … beforeAll/afterAll cleanup … */ };  // L12
export default useNock;         // L22
```

The primary subject consumes it directly:
`client/state/data-layer/wpcom-http/test/index.js:L2`
(`import useNock, { nock } from 'calypso/test-helpers/use-nock';`), calling `useNock()` (`:L27`) and
`nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).reply( 200, data )` (`:L32`).
Note the `useNock` export is marked **`@deprecated`** (`use-nock/index.js:L10`, "Use nock directly instead").

### 4.4 Timing impact — and why `nock` is *not* the cold/warm cause

Because `nock` lives in `node_modules`, it is **excluded by `transformIgnorePatterns`**
(`test/client/jest.config.js:L14-L16`,
`'node_modules[\\/\\\\](?!.*\\.(?:gif|jpg|jpeg|png|svg|scss|sass|css)$)'`). It is therefore **never
`babel-jest`‑transformed** and **never enters the transform cache**. I confirmed this empirically after a
cold run of the nock‑using `wpcom-http` test (52 cached modules):

| Empirical check against the live transform cache (52 modules) | Result |
|----------------------------------------------------------------|--------|
| Cached files whose **source path** is under `node_modules/nock` (nock's *own* module transpiled) | **0** — nock itself is **never** transformed or cached |
| Cached **project** files emitting `require("nock")` (double‑quote form) | **1** — the transformed `use-nock` helper (`…/aa/index_…`, line 17: `var _nock = _interopRequireDefault(require("nock"));`) |
| Cached files containing the identifier `disableNetConnect` | **2** — 1 code file + its `.map`: the transformed **project** bootstrap `setup-test-framework.js` (`…/60/setuptestframework_…`), which *itself* calls `nock.disableNetConnect()` (`setup-test-framework.js:L9`). It is cached **because a project file references it**, *not* because nock's own source was transformed. |

So nock is **consumed** by project files — **3** transformed project modules reference it (the bootstrap `setup-test-framework.js`, the `use-nock` helper, and the `wpcom-http` test itself) — but its own source is **never compiled or cached** (the `node_modules/nock` source‑path check above returns **0**). Its setup cost
(`disableNetConnect`, `activate`/`restore`/`cleanAll`, plus per‑test interceptor registration) is **small and
roughly constant** across cold and warm runs — it does not grow with the import graph and is not eliminated
by the cache. **Therefore `nock` is not the dominant first‑run contributor and does not explain the
cold/warm gap.** (Corroborating signal: the `sms-country-codes` leaf test uses **no** nock at all yet still
shows the same ≈1.6× cold/warm pattern — the gap is independent of nock.)


---

## 5. Q4 — `--no-cache` impact and the dominant transformation step

### 5.1 `--no-cache` reproduces the cold cost

```bash
CI=true TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --watchAll=false --no-cache <test-path>
```

`--no-cache` forces Jest to **ignore the cache and re‑transform every file** on each run. Measured, it
reproduces **cold** timing (not warm):

| Test | `--no-cache` (measured) | vs. its cold | vs. its warm |
|------|-------------------------|--------------|--------------|
| `wpcom-http` | **2.575 s / 2.591 s** | ≈ cold 2.498 s ✓ | ≫ warm 1.578 s |
| `jetpack-install` | **14.938 s** | ≈ cold 14.373 s ✓ | ≫ warm 5.277 s |

This is the mirror image of Q1: the warm speed‑up comes **entirely** from the cache, so disabling it
returns you to cold timing.

### 5.2 Cache‑decomposition experiment — isolating the dominant step

To attribute the cold overhead, I selectively cleared **only** the transform cache vs. **only** the
haste‑map between runs of `wpcom-http`, each state measured 3× with the **minimum** wall‑clock reported
(state re‑established before every measured run):

| Cache state | Wall‑clock (min of 3) | Isolated cost |
|-------------|----------------------:|---------------|
| Both caches hit (warm) | **1.530 s** | baseline |
| Transform‑cache cleared, haste‑map kept | **2.161 s** | `babel-jest` transform ≈ **0.63 s** |
| Haste‑map cleared, transform‑cache kept | **1.868 s** | haste‑map rebuild ≈ **0.34 s** |
| Both cleared (cold) | **2.586 s** | ≈ 1.06 s total (additive: 0.63 + 0.34 ≈ 0.97 s) |

> **Note on numbers.** These are **freshly re‑measured** on this machine. The `babel-jest` transform cost
> (≈0.63 s) matches the original investigation exactly; my haste‑map rebuild came out a little higher
> (≈0.34 s vs. an earlier ≈0.20 s), purely environment variance. The **qualitative conclusion is identical
> and robust**: `babel-jest` is the single largest component (≈**60%** of the cold overhead, ≈**1.9×** the
> haste‑map cost on this machine — and an even larger multiple on faster‑I/O machines).

### 5.3 The dominant step (stated plainly)

The dominant transformation step is **`babel-jest` transpiling the project's own JS/TS source** — i.e.,
parsing each module and applying the `@automattic/calypso-babel-config` presets/plugins. It is the largest
single component of the cold overhead, larger than the `jest-haste-map` rebuild and far larger than the
asset transformer.

**Why it is so expensive — the amplifier.** The custom resolver
`packages/calypso-jest/src/module-resolver.js` resolves the **`calypso:src`** field first
(`:L18` `mainFields: [ 'calypso:src', 'main' ]`; the header comment `:L8-L10` documents that `calypso:src`
"points to the _untranspiled_ source code"). This makes in‑monorepo workspace imports resolve to
**untranspiled source**, so `babel-jest` must transform a potentially **large transitive graph of source
modules at test time** — which is exactly why the cost (and the cold/warm ratio) **scales with import‑graph
size** (Q1: 44 → 1,432 modules ⇒ ≈1.69× → ≈2.72×).

**Babel root config.** `babel.config.js:L2` consumes `@automattic/calypso-babel-config` with
`:L9` `importSource: '@emotion/react'`; the transform's `rootMode: 'upward'`
(`packages/calypso-jest/jest-preset.js:L14`) makes `babel-jest` resolve to this root config.

**Negative result (to pre‑empt a wrong hypothesis).** The asset transformer
`packages/calypso-jest/src/asset-transform.js` (a trivial `process()` returning the filename as a string
module, `:L3-L5`) did **zero** work for these tests — they import no images/styles. I confirmed this: **0**
cached files are asset‑transform outputs (no `module.exports = "<file>.(gif|jpg|jpeg|png|svg|scss|sass|css)"`
entries). So the asset transformer is **not** the dominant step.


---

## 6. Conclusion — one root cause ties all four answers together

1. **The mechanism (Q1, Q4).** The suite is slow on its **first** run because Jest must do two one‑time
   jobs — build the `jest-haste-map` and **`babel-jest`‑transform every source module in the test's
   transitive import graph** — and cache the results under `cacheDirectory`
   (`test/client/jest.config.js:L7` → `<repo>/.cache/jest`). The warm run reuses that cache and is much
   faster. `--no-cache` removes the reuse and reproduces cold timing.

2. **The amplifier (Q4 → Q1).** The custom resolver prefers the **`calypso:src`** field
   (`packages/calypso-jest/src/module-resolver.js:L18`, header `:L8-L10`), so workspace imports resolve to
   **untranspiled source**. The cold run therefore transforms a **large transitive graph of source** —
   which is why the cold/warm ratio **scales with import‑graph size**: 44 modules → ≈1.69×, 52 → ≈1.58×,
   1,432 → ≈2.72×. That graph‑size dependence is the "inconsistency."

3. **What's cached (Q2).** `cacheDirectory` holds exactly three things: the **haste‑map** (≈2.68 MB
   V8‑serialized binary), the **`jest-transform-cache-*/`** sharded directory of compiled CommonJS (+ `.map`
   siblings) — the bulk of the reuse — and a small **`perf-cache-*`** JSON of per‑test runtimes for worker
   scheduling.

4. **What is *not* the cause (Q3).** `nock` (`package.json:L299`) is wired globally
   (`test/client/setup-test-framework.js`, via `setupFilesAfterEnv` at `test/client/jest.config.js:L21`)
   and per‑test (`client/test-helpers/use-nock/index.js`). But it lives in `node_modules`, is excluded by
   `transformIgnorePatterns` (`test/client/jest.config.js:L14-L16`), and is **never transformed or cached**
   (empirically: 0 nock source modules in the cache). Its cost is small and constant — **not** the
   cold/warm driver. The no‑nock `sms-country-codes` leaf shows the same pattern, confirming this.

5. **The dominant step (Q4).** A controlled cache‑decomposition isolates **`babel-jest` transformation of
   the project's own JS/TS** as the dominant first‑run step (≈0.63 s on `wpcom-http`, ≈60% of cold overhead,
   ≈1.9× the haste‑map rebuild). The asset transformer did **zero** work. The fix‑shaped takeaways (e.g.
   pre‑transpiling workspace packages or persisting the CI cache) are deliberately **out of scope** here;
   the task was to *explain* the timing, which the evidence above does.

---

## Appendix A — Reproduction

```bash
# Cold vs warm (Q1) — repeat per subject
rm -rf .cache
CI=true TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --watchAll=false \
  client/state/data-layer/wpcom-http/test/index.js            # cold
CI=true TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --watchAll=false \
  client/state/data-layer/wpcom-http/test/index.js            # warm

# Inspect the cache (Q2)
ls -la .cache/jest
find .cache/jest -maxdepth 2 | head

# Prove nock's OWN source is never cached (Q3) — the source-path check is the correct proof
TC=$(find .cache/jest -maxdepth 1 -name 'jest-transform-cache-*')
grep -rl 'node_modules/nock' "$TC" | wc -l          # -> 0   (nock's own module is never transformed/cached)
# NB: project files that *reference* nock ARE transformed/cached, so grepping for an identifier that
# nock happens to use is NOT a valid proof. e.g. disableNetConnect matches the transformed project
# bootstrap test/client/setup-test-framework.js (which calls nock.disableNetConnect() at :L9) + its .map:
grep -rl 'disableNetConnect' "$TC" | wc -l          # -> 2   (transformed setup-test-framework.js + its .map)

# --no-cache reproduces cold (Q4)
CI=true TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --watchAll=false \
  --no-cache client/state/data-layer/wpcom-http/test/index.js
```

## Appendix B — Read‑only proof

This investigation modified **no** tracked file. All test runs only regenerated the **git‑ignored**
`/.cache/` directory (`.gitignore:L15`). Verification:

```text
$ git status --porcelain          # (before authoring this report) -> empty
$ git check-ignore .cache         # -> .cache   (confirms /.cache/ is ignored)
```

The only addition to the repository is **this document**, `blitzy/documentation/wp-calypso_be7e5cc64162.md`
(named after the source branch `wp-calypso_be7e5cc64162`).

## Appendix C — Citation index

| Claim | Citation |
|-------|----------|
| `cacheDirectory` → `<repo>/.cache/jest` | `test/client/jest.config.js:L7` |
| `transformIgnorePatterns` excludes `node_modules` (except assets) | `test/client/jest.config.js:L14-L16` |
| `setupFilesAfterEnv` → nock bootstrap | `test/client/jest.config.js:L21` |
| `transform` map (`babel-jest` + asset transformer), resolver, env | `packages/calypso-jest/jest-preset.js:L9,L11,L13-L16` |
| `calypso:src` untranspiled‑source resolution | `packages/calypso-jest/src/module-resolver.js:L8-L10,L18-L19` |
| trivial asset transformer | `packages/calypso-jest/src/asset-transform.js:L3-L5` |
| Babel root config + Emotion JSX import source | `babel.config.js:L2,L9` |
| global nock setup (`disableNetConnect`, before/after) | `test/client/setup-test-framework.js:L6,L9,L11-L16,L18-L22` |
| per‑test `use-nock` helper (`@deprecated`) | `client/test-helpers/use-nock/index.js:L2,L4,L10,L22` |
| primary subject imports/uses nock | `client/state/data-layer/wpcom-http/test/index.js:L2,L27,L32` |
| heavy subject imports state tree | `client/state/data-layer/wpcom/jetpack-install/test/index.js:L5` |
| runner script; nock/jest/babel/enhanced‑resolve; engines; packageManager | `package.json:L122,L299,L290,L242,L213,L56-L59,L422` |
| `/.cache/` is git‑ignored | `.gitignore:L15` |
| Node pin `22.9.0` | `.nvmrc` |

