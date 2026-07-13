# Data-Layer Jest Test Timing Investigation

This document investigates and explains the **run-to-run timing behavior** of the Jest test suite in the
`wp-calypso` `client/state/data-layer/` module. It answers four questions about why the first execution of a
test file is much slower than subsequent executions:

1. **Q1 — Timing ratio:** run a `data-layer` test file twice in a row, measure each run, and report the
   ratio of first-run time to second-run time.
2. **Q2 — Warmup overhead / cache configuration:** find where Jest's transformation cache is configured,
   which directory it uses, which configuration option controls it, and what types of files are cached there.
3. **Q3 — HTTP mocking infrastructure:** identify the HTTP-mocking library, trace where it is configured in
   the test helpers, explain how it affects timing, and determine whether it contributes to first-run overhead.
4. **Q4 — `--no-cache` comparison:** re-run the same file with `--no-cache`, quantify the performance impact
   of disabling the cache, and identify the transformation step that consumes the most time during the
   uncached run.

Every headline value below was **observed at runtime** through the canonical Jest entry point, with the exact
command and the complete, unedited terminal output preserved, and confirmed stable across **at least two runs**
(warm timings across five runs; cold and `--no-cache` timings across multiple runs, with the honest
distribution reported where variance exists). Values labeled **OBSERVED** were measured directly; values
labeled **INFERRED** are reasoned conclusions grounded in observed evidence and `file:line` references.

## Methodology and Environment

**Representative test file (OBSERVED).** All timing runs use the same file so the cold, warm, and `--no-cache`
numbers are directly comparable:

```
client/state/data-layer/wpcom/jetpack-install/test/index.js
```

It is a small, low-variance suite of **5 tests across 3 `describe` blocks** — `installJetpackPlugin` (×1,
[`client/state/data-layer/wpcom/jetpack-install/test/index.js:L41`]), `handleSuccess` (×1, [L48]), and
`handleError` (×3, [L55]) — and it satisfies the "run any test file from the data-layer module" instruction.
The broader surface available for a larger-scale confirmation run is **80 `test/` directories / 93 test files**
under `client/state/data-layer/` (OBSERVED via `find`).

**Canonical entry point (mandatory).** The `test-client` script defines the exact command
[`package.json:L122`]:

```
"test-client": "TZ=UTC jest -c=test/client/jest.config.js",
```

Because `jest` is not on `PATH`, the installed binary is invoked directly (equivalent and still canonical);
`TZ=UTC` is set on every run exactly as the script requires:

```
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js
```

**Runtime and dependency versions (OBSERVED).** The repository pins Node `^v22.9.0`
([`package.json`] `engines`) and Yarn `4.0.2` (`packageManager`). The provided setup instructions mention a
Node 20.x install step; that discrepancy is **resolved in favor of the repository-canonical Node 22**
(Node `v22.23.1` is present and satisfies `^v22.9.0`). This is documented here so the discrepancy is not
mistaken for an error. Dependencies were installed offline via the bundled Yarn (`.yarn/releases/yarn-4.0.2.cjs`,
`--mode=skip-build`); native build steps (electron/playwright/swc/esbuild) are unnecessary for the
`data-layer` Jest suite. The actually-resolved tool versions were confirmed (not assumed):

| Tool               | Resolved version (OBSERVED) | Manifest range | Locator                                                                                               |
| ------------------ | --------------------------- | -------------- | ----------------------------------------------------------------------------------------------------- |
| `jest`             | 29.7.0                      | `^29.7.0`      | [`package.json:L290`]                                                                                 |
| `babel-jest`       | 29.7.0                      | `^29.7.0`      | [`packages/calypso-jest/package.json:L24`] (declared by the shared preset, **not** the root manifest) |
| `@babel/core`      | 7.26.10                     | `^7.26.10`     | [`package.json:L242`], [`packages/calypso-jest/package.json:L23`]                                     |
| `nock`             | 13.5.6                      | `^13.5.6`      | [`package.json:L299`]                                                                                 |
| `enhanced-resolve` | 5.9.3                       | `^5.8.3`       | [`packages/calypso-jest/package.json:L25`]                                                            |

The canonical binary reports its version as `29.7.0` (`TZ=UTC node_modules/.bin/jest --version`).

**Cache-handling protocol (followed exactly).**

- The cache directory `.cache/jest` is **cleared** (`rm -rf .cache/jest`) **before** each cold run.
- The cache is **not** cleared between warm runs.
- The uncached comparison uses **`--no-cache`** (not `--clearCache`), so the run neither reads nor writes the
  cache.

Clearing the cache before the first run is what makes it a genuine _cold_ run; without this step a
pre-existing cache would make even the "first" run warm. This faithfully reproduces the user's
"run it twice in a row" scenario, where the first invocation populates the cache and the second reads it.

## Q1 — Timing ratio (first run ÷ second run)

**Command (canonical entry point).** Each cold run is preceded by clearing the cache; each warm run reuses the
populated cache:

```
# Cold run
rm -rf .cache/jest
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js

# Warm run (same command, WITHOUT clearing the cache)
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js
```

### Cold-run raw output

**Cold run A** (first cold run of the session):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (18.375 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        18.421 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Cold run B** (cache cleared, then run):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (15.658 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        15.706 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Cold run C** (cache cleared, then run):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (15.043 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        15.095 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

### Warm-run raw output

**Warm run #1** (immediately after cold run A, cache populated):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (5.009 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        5.058 s, estimated 19 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Warm run #2:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (5.482 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        5.542 s, estimated 6 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Warm run #3:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        5.047 s, estimated 6 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Warm run B** (immediately after cold run B):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (5.008 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        5.06 s, estimated 16 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**Warm run C** (immediately after cold run C):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (5.016 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        5.067 s, estimated 16 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

### Observed timings and the ratio

| Run     | Cold `Time:` (s) | Warm `Time:` (s) | first ÷ second |
| ------- | ---------------- | ---------------- | -------------- |
| Cycle A | 18.421           | 5.058            | **3.64×**      |
| Cycle B | 15.706           | 5.060            | **3.10×**      |
| Cycle C | 15.095           | 5.067            | **2.98×**      |

- **Cold runs (OBSERVED):** 18.421 s, 15.706 s, 15.095 s. The two clean consecutive cycles (B, C) cluster
  tightly at ~15.1–15.7 s; cycle A's 18.421 s is a mild first-of-session outlier (filesystem / JIT warmup).
- **Warm runs (OBSERVED):** 5.058, 5.542, 5.047, 5.060, 5.067 s across **five** runs — a range of only
  ~0.5 s, i.e. **very stable**.

**Answer (OBSERVED).** The ratio of first-run time to second-run time is **≈ 3.0×**. The clean
back-to-back cold→warm pairs (B and C) give **3.10×** and **2.98×** (mean **3.04×**); including the
first-of-session cold run, the full observed range is **2.98×–3.64×**. In plain terms, the first execution
takes roughly **three times as long** as the second.

Arithmetic (clean pairs):

```
Cycle B: 15.706 / 5.060 = 3.10x
Cycle C: 15.095 / 5.067 = 2.98x
mean(3.10, 2.98) = 3.04x
```

**Rationale (INFERRED, confirmed in Q2 and Q4).** The first run is slow because the transform cache is
**cold**: Jest must transpile the entire imported module graph from scratch and write the results to disk.
The second run is fast because the cache is **warm**: Jest reads the already-transformed modules from
`.cache/jest` instead of re-transpiling them. The mechanism (the `cacheDirectory` transform cache) is
demonstrated in **Q2**, and confirmed by the `--no-cache` experiment in **Q4**.

## Q2 — Warmup overhead and cache configuration

**Configuration option and directory (OBSERVED).** The option that controls the cache is **`cacheDirectory`**,
set in the client Jest config [`test/client/jest.config.js:L7`]:

```
cacheDirectory: path.join( __dirname, '../../.cache/jest' ),
```

Because that file lives in `test/client/`, `path.join( __dirname, '../../.cache/jest' )` resolves to
**`.cache/jest` at the repository root**. The client config extends the shared preset
`@automattic/calypso-jest` [`test/client/jest.config.js:L2`], which supplies the transform map and resolver
(see Q4).

The directory is **gitignored** [`.gitignore:L15`]:

```
/.cache/
```

which is why `.cache/jest` does not exist on a fresh checkout and is **created on the first run** — exactly the
warmup step that makes the first run slow.

**Cached file types (OBSERVED).** After a warm run, the top level of `.cache/jest` contains **three artifact
families**:

```
$ ls -la .cache/jest
total 2640
drwxr-sr-x   3 root root    4096 .
drwxr-sr-x   3 root root    4096 ..
-rw-r--r--   1 root root 2684319 haste-map-5568e276d280a883cabe1d03482f7f50-7a07445e3e4ee1308b08068f5fc09fb5-dc06dbddc59ab91ef53eed8f2d3326fd
drwxr-sr-x 258 root root    4096 jest-transform-cache-5568e276d280a883cabe1d03482f7f50-79ef2876fae7ca75eedb2aa53dc48338
-rw-r--r--   1 root root     146 perf-cache-5568e276d280a883cabe1d03482f7f50-da39a3ee5e6b4b0d3255bfef95601890
```

**1. `haste-map-<hash>` — a binary module-resolution map (OBSERVED).** Jest's Haste map stores the scanned
dependency/module graph so it does not have to re-crawl the filesystem on every run. It is a binary file
(~2.68 MB here):

```
$ file .cache/jest/haste-map-*
.cache/jest/haste-map-5568e276d280a883cabe1d03482f7f50-...: data
```

**2. `perf-cache-<hash>` — a JSON test-timing map (OBSERVED).** This small JSON file records how long each
test file took, so Jest can distribute suites across workers by expected duration on later runs:

```
$ file .cache/jest/perf-cache-*
.cache/jest/perf-cache-5568e276d280a883cabe1d03482f7f50-...: JSON text data

$ cat .cache/jest/perf-cache-*
{"/tmp/.../client/state/data-layer/wpcom/jetpack-install/test/index.js":[1,5016]}
```

The value `5016` (ms) matches the warm run C test duration (`5.016 s`), confirming this is the per-file timing
record.

**3. `jest-transform-cache-<hash>/<2-hex>/<name>_<hash>` — transformed module output + `.map` sidecars
(OBSERVED).** This is the bulk of the cache and the family responsible for the warm speedup. It is organized
into 258 two-hex-character subdirectories. After the run it held **2757 files** total:

```
$ find .cache/jest/jest-transform-cache-* -type f | wc -l
2757
$ find .cache/jest/jest-transform-cache-* -type f ! -name '*.map' | wc -l
1432
$ find .cache/jest/jest-transform-cache-* -type f -name '*.map' | wc -l
1325
```

Each cached module is a file whose **first line is a cache-key hash**, followed by the Babel-transformed
CommonJS output. For example, the transpiled `scheme-utils.ts` module:

```
78678b27d0d9ffc66dc7d1a23fbb995d
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.addSchemeIfMissing = addSchemeIfMissing;
exports.setUrlScheme = setUrlScheme;
const schemeRegex = /^\w+:\/\//;
function addSchemeIfMissing(url, scheme) {
  ...
```

with an adjacent **`.map` source-map sidecar** (the `sources` field shows the original `.ts` file, proving a
TypeScript source was transpiled to CommonJS):

```
{"version":3,"names":["schemeRegex","addSchemeIfMissing","url","scheme","test","setUrlScheme",...],
 "sources":["scheme-utils.ts"],"sourcesContent":["import { URL as URLString, Scheme } from 'calypso/types';\n\n..."}
```

The remaining files are **trivial asset stubs**. An image/style import is replaced by a one-line
`module.exports`; e.g. the cached `style.scss` stub (63 bytes, no `.map`):

```
f32d2749fad568fdaa09a1bb1dff6381
module.exports = "style.scss";
```

Classifying the 1432 non-`.map` code files by their second line gives the composition (OBSERVED):

```
ASSET STUBS: 104          (72 scss, 29 svg, 2 png, 1 jpg)
TRANSPILED MODULES: 1328  (92.7% of code files)
```

So the transform cache is dominated by **1328 Babel-transpiled JS/TS/JSX modules** versus only **104 trivial
asset stubs** — plus **1325 `.map` source-map sidecars** for the transpiled modules. (These counts reflect the
import graph reachable from this single test file; a larger test selection would cache more modules.)

**Rationale (OBSERVED + INFERRED).** Jest scans the dependency tree once (the `haste-map`) and caches every
transformed module in `jest-transform-cache-*`. A transformer runs **once per file unless that file changes**,
so on the second run Jest reads the transformed output from disk instead of re-running Babel. This on-disk
transform cache — populated on the cold run and read on warm runs — is the direct cause of the ~3× warmup
overhead measured in **Q1**. The `jest-transform-cache-*` family is precisely what the `babel-jest` transform
(Q4) populates.

## Q3 — HTTP mocking infrastructure

**Library (OBSERVED).** The HTTP-mocking library is **`nock`** (v13.5.6), declared at [`package.json:L299`]:

```
"nock": "^13.5.6",
```

**Where it is configured (OBSERVED).** `nock` is set up in the shared setup helper
**`test/client/setup-test-framework.js`**, which the client config registers via `setupFilesAfterEnv`
[`test/client/jest.config.js:L21`]:

```
setupFilesAfterEnv: [ '<rootDir>/../test/client/setup-test-framework.js' ],
```

The relevant lines of the helper are:

```
6:  const nock = require( 'nock' );
...
8:  // Disables all network requests for all tests.
9:  nock.disableNetConnect();
10:
11: beforeAll( () => {
12:     // reactivate nock on test start
13:     if ( ! nock.isActive() ) {
14:         nock.activate();
15:     }
16: } );
17:
18: afterAll( () => {
19:     // helps clean up nock after each test run and avoid memory leaks
20:     nock.restore();
21:     nock.cleanAll();
22: } );
```

- `require( 'nock' )` [`test/client/setup-test-framework.js:L6`] loads the library once per suite.
- `nock.disableNetConnect()` [L9] disables **all** real network requests for every test (belt-and-suspenders
  network isolation; combined with the `node` test environment, no `data-layer` test can reach the network).
- `beforeAll` [L11–L16] reactivates `nock` at suite start if it is not already active.
- `afterAll` [L18–L22] calls `nock.restore()` and `nock.cleanAll()` to clean up interceptors and avoid memory
  leaks.

The same helper also installs sibling mocks that are loaded once per suite: a `global.fetch` mock
[`test/client/setup-test-framework.js:L36–L40`]:

```
36: global.fetch = jest.fn( () =>
37:     Promise.resolve( {
38:         json: () => Promise.resolve(),
39:     } )
40: );
```

and a module mock for `wpcom-proxy-request` [L44–L49]:

```
44: jest.mock( 'wpcom-proxy-request', () => ( {
45:     __esModule: true,
46:     canAccessWpcomApis: jest.fn(),
47:     reloadProxy: jest.fn(),
48:     requestAllBlogsAccess: jest.fn(),
49: } ) );
```

**Is the mock library contributing to the first-run overhead? No — `nock` is NOT the dominant driver
(INFERRED, grounded in the OBSERVED cache behavior).**

- `nock` is loaded **once per suite** through `setupFilesAfterEnv` [`test/client/jest.config.js:L21`], and its
  setup work (`disableNetConnect`, the `activate`/`restore` lifecycle) is a small, **constant per-suite** cost
  that does not scale with the size of the module graph.
- Crucially, `nock` is itself a JavaScript module and is therefore subject to the **same transform cache** as
  any other module (Q2). It is transpiled once on the cold run and read from `.cache/jest` on warm runs — so it
  cannot explain the cold-vs-warm delta, because it is cached just like everything else.
- The measured ~3× cold-vs-warm difference (Q1) tracks exactly with the presence/absence of the transform
  cache (Q2) and is reproduced by `--no-cache` (Q4). If mock setup were the driver, disabling the transform
  cache would not change the timing — but it does, dramatically. Therefore the first-run overhead is dominated
  by **transformation** (Q4), not by the `nock` mock setup.

In short: `nock` provides network isolation with a small constant per-suite cost, and it is not a material
contributor to the first-run overhead.

## Q4 — `--no-cache` comparison and dominant transformation step

**Command (canonical entry point, `--no-cache` appended).** With `--no-cache`, Jest neither reads from nor
writes to `.cache/jest`, so the full module graph is re-transformed on every run:

```
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --no-cache client/state/data-layer/wpcom/jetpack-install/test/index.js
```

### `--no-cache` raw output (five runs)

**`--no-cache` run #1:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (16.784 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        16.829 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**`--no-cache` run #2:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (29.163 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        29.213 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**`--no-cache` run #3:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (18.118 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        18.166 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**`--no-cache` run #4:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (20.591 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        20.639 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

**`--no-cache` run #5:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (25.173 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        25.228 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

### Performance impact of disabling the cache

- **`--no-cache` timings (OBSERVED, five runs):** 16.829, 18.166, 20.639, 25.228, 29.213 s — mean **≈ 22.0 s**,
  median **≈ 20.6 s**. Unlike the warm runs, these are **not tightly stable**: because every run re-transpiles
  the entire graph, the timing is dominated by a heavy, CPU-bound transformation workload that is sensitive to
  machine contention. The honest distribution is reported here rather than a single controlled figure.
- **Warm baseline (OBSERVED):** ~5.06 s (from Q1, stable across five runs).

Ratio and savings (arithmetic):

```
median --no-cache / warm = 20.639 / 5.06 = 4.08x
mean   --no-cache / warm = 22.015 / 5.06 = 4.35x
range                    = 16.829/5.06 .. 29.213/5.06 = 3.33x .. 5.77x

cache savings (median) = (20.639 - 5.06) / 20.639 = 75.5%
cache savings (range)  = 69.9% .. 82.7%
```

**Answer (OBSERVED).** Disabling the cache makes the run roughly **4× slower** than the warm run
(median 4.08×; observed range 3.33×–5.77×), i.e. the transform cache **saves on the order of ~75%** of the
run time. This is consistent with Jest's documented behavior that running with the cache disabled is "at least
two times slower." Note that the `--no-cache` times (~17–29 s) are in the **same regime as the cold-run times**
(~15–18 s from Q1) and far above the warm times (~5 s): both the cold run and every `--no-cache` run must
transform the full module graph, whereas warm runs read it from disk. This directly confirms the Q1 rationale —
the first-run overhead is the transformation step, not the mock setup (Q3).

### Which transformation step dominates?

**The dominant transformation step is `babel-jest` (OBSERVED composition + INFERRED cost attribution).** The
transform map in the shared preset binds file patterns to transformers [`packages/calypso-jest/jest-preset.js:L13–L16`]:

```
13: transform: {
14:     '\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
15:     '\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
16: },
```

- **`babel-jest`** [L14] handles every `.js`/`.jsx`/`.ts`/`.tsx` module (`rootMode: 'upward'` makes it pick up
  the root `babel.config.js`).
- The **asset transform** [L15] handles image/style extensions, and it is a **trivial one-line stub** that
  returns `module.exports = "<basename>";` [`packages/calypso-jest/src/asset-transform.js:L3–L6`, emission at
  L5]:

  ```
  3: module.exports = {
  4:     process( src, filename ) {
  5:         return { code: 'module.exports = ' + JSON.stringify( path.basename( filename ) ) + ';' };
  6:     },
  7: };
  ```

  Its cost is negligible (it does no parsing or code generation), so it cannot be the bottleneck.

Because the asset transform is effectively free, the `babel-jest` step on JS/TS/JSX modules is **unambiguously
the most expensive** transformation. Two independent pieces of evidence confirm this:

1. **Why the graph is large (OBSERVED).** The custom resolver
   [`packages/calypso-jest/src/module-resolver.js`] uses `enhanced-resolve` with
   `mainFields: [ 'calypso:src', 'main' ]` [L18] and `conditionNames: [ 'calypso:src', 'node', 'require' ]`
   [L19], so monorepo packages resolve to their **untranspiled source** (`calypso:src`) rather than a
   pre-built artifact. There is no separate build step, so `babel-jest` must transpile the whole imported graph
   on the cold/uncached run. The representative test imports its source
   [`client/state/data-layer/wpcom/jetpack-install/index.js`], which in turn pulls in `calypso/state/*` —
   action-types [L2], analytics actions [L3], the data-layer handler registry [L4], `wpcom-http` actions and
   utils [L5–L6], and `jetpack-remote-install` actions [L7–L10] — a large tree that all flows through
   `babel-jest`. `babel-jest` applies the Babel configuration in [`babel.config.js`], which delegates to
   `@automattic/calypso-babel-config` [`babel.config.js:L2,L6`].

2. **Cache composition corroborates it (OBSERVED, from Q2).** The `jest-transform-cache-*` directory is
   dominated by **1328 Babel-transpiled JS/TS/JSX modules** (each with a `.map` sidecar) versus only **104
   trivial asset stubs**. The overwhelming majority of transform work — and therefore of the uncached run time
   — is `babel-jest` transpilation.

**Conclusion (Q4).** Disabling the cache costs roughly **4×** relative to the warm run (~75% of the time is
saved by the cache), and the transformation step consuming the most time during the uncached run is
**`babel-jest`** transpiling the untranspiled `calypso:src` module graph.

## Coverage pass

Every named sub-part of the four questions is addressed above:

- **Q1 — data-layer test file used:** `client/state/data-layer/wpcom/jetpack-install/test/index.js` (5 tests /
  3 `describe` blocks). **First ÷ second ratio ≈ 3.0×** (OBSERVED; clean pairs 3.10× and 2.98×, full range
  2.98×–3.64×); warm runs stable across five repeats.
- **Q2 — cache configuration option, directory, and cached file types:** option **`cacheDirectory`**
  [`test/client/jest.config.js:L7`]; directory **`.cache/jest`** at the repo root (gitignored,
  [`.gitignore:L15`]); **three cached file types** — `haste-map-<hash>` (binary module-resolution map),
  `perf-cache-<hash>` (JSON per-file timing map), and `jest-transform-cache-<hash>/…` (Babel-transformed module
  output + `.map` source-map sidecars; OBSERVED composition **1328 transpiled modules vs 104 asset stubs**,
  1325 `.map` sidecars).
- **Q3 — mocking library, helper location, timing effect, first-run contribution:** library **`nock`**
  ([`package.json:L299`], v13.5.6); configured in **`test/client/setup-test-framework.js`** (L6, L9, L11–L16,
  L18–L22; sibling mocks L36–L40 and L44–L49) via `setupFilesAfterEnv` [`test/client/jest.config.js:L21`];
  timing effect is a **small constant per-suite** cost; it is **NOT** a material contributor to first-run
  overhead (it is itself cached like any other module).
- **Q4 — `--no-cache` impact and slowest transformation step:** `--no-cache` is **≈ 4× slower** than warm
  (OBSERVED median 4.08×; cache saves ~75%), matching the cold-run regime; the dominant transformation step is
  **`babel-jest`** [`packages/calypso-jest/jest-preset.js:L14`], grounded in the trivial asset transform
  [`packages/calypso-jest/src/asset-transform.js:L3–L6`], the `calypso:src` resolver
  [`packages/calypso-jest/src/module-resolver.js:L18–L19`], and the observed cache composition.

**Repository state.** This investigation was **read-only**. No existing repository file was modified, added, or
deleted; the only new file is this document. Temporary observation artifacts were kept outside the working tree
and removed afterward, and `.cache/` and `node_modules/` remain gitignored and uncommitted, so
`git status --porcelain` shows only this document.
