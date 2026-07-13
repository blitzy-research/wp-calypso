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
command and the complete, unedited terminal output preserved. Each value was produced by repeating the **same
unchanged run** at least twice: the **warm** timings are tightly **stable** across five runs, whereas the
**cold** and **`--no-cache`** timings genuinely vary run-to-run and are therefore reported as an **honest
distribution** (a range/median) rather than a single "stable" figure. Values labeled **OBSERVED** were measured
directly; values labeled **INFERRED** are reasoned conclusions grounded in observed evidence and `file:line`
references. Any proposed _cause_ that was not itself measured — for example filesystem/JIT warmup or machine
contention affecting the cold/`--no-cache` spread — is labeled **INFERRED**, never OBSERVED.

## Methodology and Environment

**Representative test file (OBSERVED).** All timing runs use the same file so the cold, warm, and `--no-cache`
numbers are directly comparable:

```
client/state/data-layer/wpcom/jetpack-install/test/index.js
```

It is a small, low-variance suite of **5 tests across 3 `describe` blocks** — `installJetpackPlugin` (×1,
[`client/state/data-layer/wpcom/jetpack-install/test/index.js:L41`]), `handleSuccess` (×1, [L48]), and
`handleError` (×3, [L55]) — and it satisfies the "run any test file from the data-layer module" instruction.
The broader surface available for a larger-scale confirmation run is **80 `test/` directories** under
`client/state/data-layer/`, containing **92** files that Jest's `testMatch` (`*/test/*.[jt]s?(x)`, files
directly inside a `test/` directory) would match and **93** `.js/.jsx/.ts/.tsx` files if counted recursively
(the one extra file, `client/state/data-layer/wpcom/sites/atomic/test/transfers/index.js`, lives in a nested
`test/transfers/` subdirectory and is not matched by `testMatch`). Exact commands and complete output
(OBSERVED):

```
$ find client/state/data-layer -type d -name test | wc -l
80
$ find client/state/data-layer -type f -regextype posix-extended -regex '.*/test/[^/]+\.[jt]sx?$' | wc -l
92
$ find client/state/data-layer -type f -path '*/test/*' \( -name '*.js' -o -name '*.jsx' -o -name '*.ts' -o -name '*.tsx' \) | wc -l
93
```

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

**Runtime and dependency versions (OBSERVED).** The repository pins Node `22.9.0` ([`.nvmrc:L1`]) and requires
Node `^v22.9.0` ([`package.json:L56–L57`], `engines.node`) with Yarn `4.0.2` ([`package.json:L422`],
`packageManager`; the bundled release is pinned at [`.yarnrc.yml:L3–L5`], `nodeLinker: node-modules` /
`enableGlobalCache: true` / `yarnPath: .yarn/releases/yarn-4.0.2.cjs`). The provided setup instructions mention
a Node 20.x install step; that discrepancy is **resolved in favor of the repository-canonical Node 22**. The
running versions were confirmed (not assumed) with exact commands:

```
$ node --version
v22.23.1
$ yarn --version
4.0.2
$ TZ=UTC node_modules/.bin/jest --version
29.7.0
```

`v22.23.1` satisfies `^v22.9.0`, so this is documented here purely so the Node-version discrepancy is not
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

**Browserslist warning (disclosed and assessed, OBSERVED).** Every timing run in this document prints the same
Browserslist notice before the test result:

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
```

It appears **identically across all conditions** (cold, warm, and `--no-cache`) and is emitted by Babel's
Browserslist integration, not by the test runner. It is **not** a test failure — every run reports
`Tests: 5 passed, 5 total` — and because it appears in every run alike it does not affect the cold-vs-warm or
cached-vs-uncached comparison. Its suggested remediation (`npx update-browserslist-db@latest`) was
**intentionally not executed**: refreshing the `caniuse-lite` data would modify dependency/lockfile state,
which is out of scope for this read-only investigation.

**Cache-handling protocol (followed exactly).**

- The cache directory `.cache/jest` is **cleared** (`rm -rf .cache/jest`) **before** each cold run.
- The cache is **not** cleared between warm runs.
- The uncached comparison uses **`--no-cache`** (not `--clearCache`). In Jest 29.7 `--no-cache` does **not**
  disable cache _writes_; it only prevents Jest from **reading/reusing** previously cached transforms, so
  every transform-eligible file is transformed again on each invocation (see Q4 for the source-level and
  empirical evidence). It is the forced re-transformation — not any change to writing — that makes the run
  behave like a cold run.

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
**cold**: Jest must transpile the full **transform-eligible** application/workspace source graph (every file
matched by the `transform` map and not excluded by `transformIgnorePatterns` — so workspace/app
`.js`/`.jsx`/`.ts`/`.tsx`, but not JavaScript under `node_modules`) from scratch and write the results to disk.
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

**Cached file types (OBSERVED).** The cache is populated by a cold run and then inspected. The specific cold
run that produced the snapshot below reported:

```
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (15.729 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        15.776 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
```

The top level of `.cache/jest` then contains **three artifact families**. The on-disk names embed
config hashes (not secrets); to keep the listing deterministic and free of the non-reproducible timestamp
column, names are listed sorted:

```
$ ls -1 .cache/jest | sort
haste-map-5568e276d280a883cabe1d03482f7f50-7a07445e3e4ee1308b08068f5fc09fb5-dc06dbddc59ab91ef53eed8f2d3326fd
jest-transform-cache-5568e276d280a883cabe1d03482f7f50-79ef2876fae7ca75eedb2aa53dc48338
perf-cache-5568e276d280a883cabe1d03482f7f50-da39a3ee5e6b4b0d3255bfef95601890
```

**1. `haste-map-<hash>` — a binary module-resolution map (OBSERVED).** Jest's Haste map stores the scanned
dependency/module graph so it does not have to re-crawl the filesystem on every run. `file` reports it as
binary (`data`); in this snapshot it is 2,565,419 bytes (~2.57 MB):

```
$ file .cache/jest/haste-map-*
.cache/jest/haste-map-5568e276d280a883cabe1d03482f7f50-7a07445e3e4ee1308b08068f5fc09fb5-dc06dbddc59ab91ef53eed8f2d3326fd: data
$ stat -c '%s' .cache/jest/haste-map-*
2565419
```

**2. `perf-cache-<hash>` — a JSON test-timing map (OBSERVED).** This small JSON file records each test file's
outcome and duration so Jest can distribute suites across workers by expected duration on later runs. The
`cat` output is piped through `sed "s|$PWD/||g"` so the absolute test path is emitted repository-relative — a
transform applied by the command shown, not a manual redaction:

```
$ file .cache/jest/perf-cache-*
.cache/jest/perf-cache-5568e276d280a883cabe1d03482f7f50-da39a3ee5e6b4b0d3255bfef95601890: JSON text data
$ cat .cache/jest/perf-cache-* | sed "s|$PWD/||g"
{"client/state/data-layer/wpcom/jetpack-install/test/index.js":[1,15729]}
```

The value `[1, 15729]` is `[status, runtimeMs]`. The status `1` is `SUCCESS` (the module defines
`const FAIL = 0` / `const SUCCESS = 1` at [`@jest/test-sequencer/build/index.js:L92-L93`], written by
`cacheResults()` at [`@jest/test-sequencer/build/index.js:L258-L276`]). The runtime `15729` ms matches the
cold run's reported test duration (`15.729 s`) above, confirming this is the per-file timing record.

**3. `jest-transform-cache-<hash>/<2-hex>/<name>_<hash>` — transformed module output + `.map` sidecars
(OBSERVED).** This is the bulk of the cache and the family responsible for the warm speedup. It is organized
into **256** two-hex-character subdirectories (`00`–`ff`) — a real child-directory count (the `258` a plain
`ls -la` shows for this directory is its hard-link count: 256 children plus `.` and `..`):

```
$ find .cache/jest/jest-transform-cache-* -mindepth 1 -maxdepth 1 -type d | wc -l
256
```

After the cold run it held **2757 files** total — **1432** transformed-code files and **1325** `.map`
source-map sidecars:

```
$ find .cache/jest/jest-transform-cache-* -type f | wc -l
2757
$ find .cache/jest/jest-transform-cache-* -type f ! -name '*.map' | wc -l
1432
$ find .cache/jest/jest-transform-cache-* -type f -name '*.map' | wc -l
1325
```

Each cached module is a file whose **first line is a cache-key hash**, followed by the Babel-transformed
CommonJS output. For example, the transpiled `scheme-utils.ts` module (`head -n 12` bounds the excerpt):

```
$ head -n 12 .cache/jest/jest-transform-cache-*/7b/schemeutils_7b25f2c88fc00f429c8d353f3d8f9538
78678b27d0d9ffc66dc7d1a23fbb995d
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.addSchemeIfMissing = addSchemeIfMissing;
exports.setUrlScheme = setUrlScheme;
const schemeRegex = /^\w+:\/\//;
function addSchemeIfMissing(url, scheme) {
  if (false === schemeRegex.test(url)) {
    return scheme + '://' + url;
```

It has an adjacent **`.map` source-map sidecar** whose `sources` field is the original `.ts` file, proving a
TypeScript source was transpiled to CommonJS. The map is a single long line; `head -c 400` bounds the excerpt
to its first 400 bytes:

```
$ head -c 400 .cache/jest/jest-transform-cache-*/7b/schemeutils_7b25f2c88fc00f429c8d353f3d8f9538.map
{"version":3,"names":["schemeRegex","addSchemeIfMissing","url","scheme","test","setUrlScheme","schemeWithSlashes","startsWith","newUrl","replace"],"sources":["scheme-utils.ts"],"sourcesContent":["import { URL as URLString, Scheme } from 'calypso/types';\n\nconst schemeRegex = /^\\w+:\\/\\//;\n\nexport function addSchemeIfMissing( url: URLString, scheme: Scheme ): URLString {\n\tif ( false === sche
```

The remaining code files are **trivial asset stubs**: an image/style import is replaced by a one-line
`module.exports` string, with no `.map`. The complete cached `style.scss` stub is only two lines (the cache-key
hash plus the emitted string), so `cat` shows all of it:

```
$ cat .cache/jest/jest-transform-cache-*/7b/style_7bf204ecd26b12c08b316a9d3dad2ad9
f32d2749fad568fdaa09a1bb1dff6381
module.exports = "style.scss";
```

To classify the 1432 code files, the script below (run from the repository root) checks whether each file's
second line is exactly the asset-stub form `module.exports = "<file>";`. It is the exact classifier used, and
its complete output follows:

```
$ python3 - <<'PY'
import os, re, glob
trc = glob.glob('.cache/jest/jest-transform-cache-*')[0]
asset_re = re.compile(r'^module\.exports = "([^"]*)";$')
code = [os.path.join(r, n) for r, _, fs in os.walk(trc) for n in fs if not n.endswith('.map')]
maps = {os.path.join(r, n) for r, _, fs in os.walk(trc) for n in fs if n.endswith('.map')}
stubs, transpiled, no_map = {}, [], []
for p in code:
    with open(p, errors='replace') as fh:
        fh.readline(); line2 = fh.readline().rstrip('\n')
    m = asset_re.match(line2)
    if m:
        ext = os.path.splitext(m.group(1))[1].lstrip('.'); stubs[ext] = stubs.get(ext, 0) + 1
    else:
        transpiled.append(p)
        if p + '.map' not in maps: no_map.append(p)
print('asset stubs:', sum(stubs.values()), dict(sorted(stubs.items(), key=lambda kv: -kv[1])))
print('transpiled modules:', len(transpiled))
print('transpiled WITHOUT an adjacent .map:', len(no_map))
for p in sorted(no_map): print('   ', os.path.relpath(p, trc))
PY
asset stubs: 104 {'scss': 72, 'svg': 29, 'png': 2, 'jpg': 1}
transpiled modules: 1328
transpiled WITHOUT an adjacent .map: 3
    9e/jsonschemadraft04_9e099ffabbf28d94647ebb2083192a23
    e4/wpcommultileveltlds_e436b243aa2d15be81747675f577a172
    fd/languagesmeta_fdb566697eda9a10454550ecd7403ca6
```

So the transform cache is dominated by **1328 transpiled modules** (almost all Babel-transpiled JS/TS/JSX,
plus a few JSON data modules) versus only **104 trivial asset stubs** (72 `scss`, 29 `svg`, 2 `png`, 1 `jpg`).
Of the 1328 transpiled modules, **1,325 have a `.map` source-map sidecar**; the **3 without a map** are JSON
data modules (`json-schema-draft-04`, `wpcom-multi-level-tlds`, `languages-meta`), which are cached as raw JSON
with no transpilation and therefore no source map. This reconciles the counts exactly: 104 asset stubs + 1328
transpiled = 1432 code files, and 1432 code files + 1325 maps = 2757 total. (These counts reflect the import
graph reachable from this single test file; a larger test selection would cache more modules.)

**Rationale (OBSERVED + INFERRED).** Jest scans the dependency tree once (the `haste-map`) and caches each
transformed module in `jest-transform-cache-*`. A transformer runs **once per file unless that file changes**,
so on the second run Jest reads the transformed output from disk instead of re-running Babel. This on-disk
transform cache — populated on the cold run and read on warm runs — is (INFERRED) the direct cause of the ~3×
warmup overhead measured in **Q1**, an inference confirmed by the `--no-cache` experiment in **Q4**. The
`jest-transform-cache-*` family is precisely what the `babel-jest` transform (Q4) populates.

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
- Crucially, `nock` is **not** transformed or stored in Jest's transform cache. It resolves to
  `node_modules/nock/index.js`, and the client config's `transformIgnorePatterns`
  [`test/client/jest.config.js:L14–L16`] excludes JavaScript under `node_modules` from transformation (only
  asset extensions under `node_modules` remain transformable). So `nock` is loaded by Node's `require` from
  `node_modules` on **every** run — a common per-run baseline present on cold and warm runs alike — rather than
  being served from `.cache/jest`. This is directly OBSERVED in the populated cache, which holds the
  transformed **setup helper** but **no `nock` module and no `nock` source map**:

  ```
  $ TRC=$(ls -d .cache/jest/jest-transform-cache-*)
  $ grep -rl '"sources":\["setup-test-framework.js"\]' "$TRC" | wc -l   # setup helper IS cached
  1
  $ grep -rlE '"sources":\[[^]]*nock' "$TRC" | wc -l                    # no nock-sourced map
  0
  $ grep -rlF 'node_modules/nock' "$TRC" | wc -l                        # no reference to nock's package path
  0
  ```

  Because this per-run `require`/setup cost is the same on cold and warm runs, it cannot account for the
  cold-vs-warm delta.

- The measured ~3× cold-vs-warm difference (Q1) tracks the presence/absence of the transform cache (Q2) and is
  reproduced by `--no-cache` (Q4). If mock setup were the driver, forcing re-transformation with `--no-cache`
  would not change the timing — but it does, dramatically. It is therefore an **INFERENCE** (from this
  cache-sensitive timing delta, not from direct per-call profiling of `nock`) that the first-run overhead is
  dominated by **transformation** (Q4), not by the `nock` mock setup.

In short: `nock` provides network isolation with a small constant per-suite cost, and it is not a material
contributor to the first-run overhead.

## Q4 — `--no-cache` comparison and dominant transformation step

**Command (canonical entry point, `--no-cache` appended).** The same representative file is re-run with
`--no-cache` appended:

```
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --no-cache client/state/data-layer/wpcom/jetpack-install/test/index.js
```

**What `--no-cache` actually does (OBSERVED + source-grounded).** In Jest 29.7, `--no-cache` sets
`config.cache = false`. This makes the transformer **skip reading/reusing** previously cached transforms —
`const code = this._config.cache ? readCodeCacheFile(cacheFilePath) : null;`
[`node_modules/@jest/transform/build/ScriptTransformer.js:L527–L528`] — so every transform-eligible file is
processed again on each invocation. It does **not** disable cache _writes_: `_buildTransformResult` still
calls `writeCacheFile`/`writeCodeCacheFile` unconditionally [`ScriptTransformer.js:L503–L514`], the Haste map
is still persisted [`jest-haste-map/build/index.js:L366–L380`, `L713–L717`], and the performance cache is
still written [`@jest/test-sequencer/build/index.js:L258–L276`]. This is directly OBSERVED — deleting the
cache and then running with `--no-cache` **re-creates** `.cache/jest` and repopulates all three families:

```
$ rm -rf .cache/jest; test -e .cache/jest && echo present || echo absent
absent
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --no-cache client/state/data-layer/wpcom/jetpack-install/test/index.js 2>&1 | tail -1
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.
$ test -e .cache/jest && echo present || echo absent
present
$ find .cache/jest/jest-transform-cache-* -type f | wc -l
2757
$ ls -1 .cache/jest | sed -E 's/-[0-9a-f].*$//' | sort -u
haste-map
jest-transform
perf
```

So the relevant effect of `--no-cache` is that prior cached transforms are **not read/reused**, forcing
re-transformation of the full **transform-eligible** application/workspace source graph (the files matched by
the `transform` map and not excluded by `transformIgnorePatterns` — workspace/app `.js`/`.jsx`/`.ts`/`.tsx`,
but **not** JavaScript under `node_modules`). That forced re-transformation is what reproduces the cold run's
cost, even though the cache is still written.

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
run time. This is consistent with Jest's documented behavior for the `--cache`/`--no-cache` option, which
states that "the cache should only be disabled if you are experiencing caching related problems. On average,
disabling the cache makes Jest at least two times slower" (official Jest CLI Options documentation,
`--cache` / `--no-cache`, `https://jestjs.io/docs/cli#--cache`; the installed runner is Jest 29.7.0 per
`TZ=UTC node_modules/.bin/jest --version` above, and this guidance is unchanged in the v29.7 line —
this is EXTERNAL documentation, not a runtime observation). Note that the `--no-cache` times (~17–29 s) are in the **same regime as the cold-run times**
(~15–18 s from Q1) and far above the warm times (~5 s): both the cold run and every `--no-cache` run must
transform the full transform-eligible source graph, whereas warm runs read it from disk. This directly
confirms the Q1 rationale — the first-run overhead is the transformation step, not the mock setup (Q3).

### Which transformation step dominates?

**The dominant transformation step is `babel-jest` (OBSERVED composition + INFERRED cost attribution).** The
transform map in the shared preset binds file patterns to transformers [`packages/calypso-jest/jest-preset.js:L13–L16`]:

```
13: transform: {
14:     '\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
15:     '\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
16: },
```

- **`babel-jest`** [L14] handles every matching `.js`/`.jsx`/`.ts`/`.tsx` file that is not excluded by the
  client config's `transformIgnorePatterns` [`test/client/jest.config.js:L14–L16`] — i.e. workspace/app
  source, but not JavaScript under `node_modules` (`rootMode: 'upward'` makes it pick up the root
  `babel.config.js`).
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

  It emits a single short JavaScript string (`module.exports = "<basename>";`) but performs **no source
  parsing and no AST transpilation**, so its cost is negligible and it cannot be the bottleneck.

Because the asset transform is effectively free, the `babel-jest` step on JS/TS/JSX modules is **unambiguously
the most expensive** transformation. Two independent pieces of evidence confirm this:

1. **Why the graph is large (OBSERVED).** The custom resolver
   [`packages/calypso-jest/src/module-resolver.js`] uses `enhanced-resolve` with
   `mainFields: [ 'calypso:src', 'main' ]` [L18] and `conditionNames: [ 'calypso:src', 'node', 'require' ]`
   [L19], so monorepo packages resolve to their **untranspiled source** (`calypso:src`) rather than a
   pre-built artifact. There is no separate build step, so `babel-jest` must transpile the whole
   transform-eligible imported graph (workspace/app source, excluding `node_modules` JavaScript) on the
   cold/uncached run. The representative test imports its source
   [`client/state/data-layer/wpcom/jetpack-install/index.js`], which in turn pulls in `calypso/state/*` —
   action-types [L2], analytics actions [L3], the data-layer handler registry [L4], `wpcom-http` actions and
   utils [L5–L6], and `jetpack-remote-install` actions [L7–L10] — a large tree that all flows through
   `babel-jest`. `babel-jest` applies the Babel configuration in [`babel.config.js`], which delegates to
   `@automattic/calypso-babel-config` [`babel.config.js:L2,L6`].

2. **Cache composition corroborates it (OBSERVED, from Q2).** The `jest-transform-cache-*` directory is
   dominated by **1328 transpiled modules** (1,325 of them carry a `.map` sidecar; the 3 without one are JSON
   data modules) versus only **104 trivial asset stubs**. The overwhelming majority of transform work — and
   therefore of the uncached run time — is `babel-jest` transpilation.

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
  timing effect is a **small constant per-suite** cost; it is (INFERRED) **NOT** a material contributor to
  first-run overhead — `nock` is excluded from transformation by `transformIgnorePatterns`
  [`test/client/jest.config.js:L14–L16`], so it is loaded from `node_modules` via `require` on every run rather
  than served from the transform cache, and that per-run cost is identical cold and warm.
- **Q4 — `--no-cache` impact and slowest transformation step:** `--no-cache` is **≈ 4× slower** than warm
  (OBSERVED median 4.08×; cache saves ~75%), matching the cold-run regime; the dominant transformation step is
  **`babel-jest`** [`packages/calypso-jest/jest-preset.js:L14`], grounded in the trivial asset transform
  [`packages/calypso-jest/src/asset-transform.js:L3–L6`], the `calypso:src` resolver
  [`packages/calypso-jest/src/module-resolver.js:L18–L19`], and the observed cache composition.

**Repository state.** This investigation was **read-only**. No existing repository file was modified, added, or
deleted; the only new file is this document. Temporary observation artifacts were kept outside the working tree
and removed afterward, and `.cache/` and `node_modules/` remain gitignored and uncommitted. While the
investigation was in progress (before this document was committed), `git status --porcelain` listed only this
new document as a single untracked entry, confirming that no existing tracked file had been changed. Once this
document is committed, the working tree is left clean — `git status --porcelain` produces no output — with
`.cache/` and `node_modules/` still gitignored and untracked.
