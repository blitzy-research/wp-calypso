# Why `client/state/data-layer` Jest tests run inconsistently between a cold and a warm run

_An empirical investigation of the `Automattic/wp-calypso` monorepo at HEAD `be7e5cc641622d153040491fd5625c6cb83e12eb`._

> **Methodology note (read first).** Every number, listing, and behavioural claim in this document was produced by **actually running the code first** and pasting the **complete, unedited** output next to the exact command that produced it. Values are the ones observed on this machine; where they diverge from commonly cited figures, the observed value is reported as-is and the divergence is called out explicitly. Facts that were read from source rather than observed at runtime are labelled **(inferred from code)**. Every factual claim carries an actual value **and** a `file:line` reference.

---

## 1. TL;DR — the direct answer

The first ("cold") run of a `client/state/data-layer` test is **~2.5× slower in wall-clock time** (2.51–2.58× measured; ~2.9× on Jest's own reported `Time`) than an immediately following identical ("warm") run. Concretely, the chosen test `client/state/data-layer/wpcom/jetpack-install/test/index.js` took **`real ~15.2–15.5s`** cold versus **`real ~6.0–6.1s`** warm across three back-to-back pairs.

The cause of the warm-up overhead is **`babel-jest` transpilation combined with Jest's transform cache**:

1. Jest's transform cache is controlled by the **`cacheDirectory`** option, set at **`test/client/jest.config.js:L7`** to resolve to **`<repo-root>/.cache/jest`**. On a cold run this directory is empty; on a warm run it is fully populated.
2. Because the custom resolver (`packages/calypso-jest/src/module-resolver.js:L16-20`) maps in-repo `@automattic/*` packages via their **`calypso:src`** field to **untranspiled source**, a cold run must Babel-transpile the _entire imported source graph_ — **1432 modules** for this single test — and write the results (transformed JS + source maps + a haste map) into `.cache/jest`. The warm run reads those cached transforms and skips transpilation, which is why it is dramatically faster.
3. **HTTP mocking is performed by `nock`** (`package.json:L299`), wired globally in `test/client/setup-test-framework.js`. It is a **negligible, fixed** cost: `require('nock')` measured **~20 ms** one-time, versus the **~9.3s** cold−warm delta (≈0.2%). It is **not** a meaningful contributor to first-run overhead.
4. Running with **`--no-cache`** forces transpilation on _every_ run: **`real ~15.7–16.1s`**, i.e. **~2.6× the warm run** and essentially **equal to the cold run**. The transformation step that consumes the most time is unambiguously **`babel-jest` transpilation** (`packages/calypso-jest/jest-preset.js:L14`).

**In one sentence:** the cold/warm inconsistency is Jest's transform cache doing its job — the first run pays the full `babel-jest` transpilation cost of ~1400+ untranspiled source modules and persists it under `<repo-root>/.cache/jest`; the second run reuses it.

---

## 2. Environment & method

### 2.1 Repository, runtime, and toolchain

| Item               | Value                                                                | Evidence / reference                                                                                             |
| ------------------ | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Repo / HEAD        | `Automattic/wp-calypso` @ `be7e5cc641622d153040491fd5625c6cb83e12eb` | `git rev-parse HEAD`                                                                                             |
| Node               | `v22.23.1` (satisfies canonical `node "^v22.9.0"`)                   | `node --version`; `package.json:L57`; `.nvmrc` = `22.9.0`                                                        |
| Yarn               | `4.0.2` via Corepack                                                 | `.yarnrc.yml:L5` `yarnPath: .yarn/releases/yarn-4.0.2.cjs`; `package.json:L422` `"packageManager": "yarn@4.0.2"` |
| Node linker        | `node-modules`                                                       | `.yarnrc.yml:L3` `nodeLinker: node-modules`                                                                      |
| Jest               | `29.7.0`                                                             | `yarn jest --version`; `package.json:L290` `"jest": "^29.7.0"`                                                   |
| `@babel/core`      | `7.26.10`                                                            | `package.json:L242` `"@babel/core": "^7.26.10"`                                                                  |
| `nock`             | `13.5.6`                                                             | `package.json:L299` `"nock": "^13.5.6"`                                                                          |
| `enhanced-resolve` | `5.9.3`                                                              | `packages/calypso-jest/package.json:L25` `"enhanced-resolve": "^5.8.3"`                                          |

`node_modules` was already installed in this working tree (dependencies restored via `corepack enable && yarn install --immutable`, `nodeLinker: node-modules`). Verification:

```
$ node --version
v22.23.1
$ yarn --version
4.0.2
$ yarn jest --version
29.7.0
```

### 2.2 The canonical entry point and the exact commands

The tests are exercised through the **real** client Jest configuration `test/client/jest.config.js`, which is what the `test-client` npm script uses. That script is defined at `package.json:L122`:

```
"test-client": "TZ=UTC jest -c=test/client/jest.config.js",
```

Note what the npm script is and is **not**: it sets `TZ=UTC`, selects the client config, and takes **no** `time` wrapper, **no** `yarn` wrapper, and **no** explicit test path. This investigation wraps that same invocation with the shell's `time` keyword and `yarn`, and appends a specific data-layer test path. The canonical commands are therefore:

- **Cold / warm run:** `TZ=UTC time yarn jest -c=test/client/jest.config.js <data-layer test>`
- **Uncached run:** the same command with **`--no-cache`** appended
- **Cache inspection:** `ls -R .cache/jest`
- **Force a cold state:** `rm -rf .cache/jest` (this is the gitignored transform cache — `.gitignore:L15` `/.cache/`)

### 2.3 A required caveat about `time`

This environment has **no `/usr/bin/time` binary**, and `time` in bash is only a _reserved keyword_, not a command. Prefixing a variable assignment (`TZ=UTC`) directly before `time` makes bash try to execute `time` as an external program, which fails. Observed:

```
$ ls -la /usr/bin/time
ls: cannot access '/usr/bin/time': No such file or directory
$ TZ=UTC time echo hello
/bin/bash: line 226: time: command not found
```

The **functionally identical** form used for every timed run in this document keeps `TZ=UTC` in the environment and lets the bash `time` keyword measure wall-clock (`real`/`user`/`sys`):

```
export TZ=UTC; time yarn jest -c=test/client/jest.config.js <data-layer test> [--no-cache]
```

`TZ` remains `UTC`; the only change from the literal prompt command is that `TZ` is exported first so `time` is parsed as the keyword. The reported `real` values are thus fully reproducible.

### 2.4 Target test files

| Target                                                        | Size                                                        | Role                                                                                                                                                                                                                            |
| ------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client/state/data-layer/wpcom/jetpack-install/test/index.js` | 99 lines, 3 `describe` blocks / **5 tests**, **1 snapshot** | **Primary** target for all timing (Parts 1, 2, 4). A pure-function/snapshot test; imports `calypso/state/action-types` (L1), `calypso/state/jetpack-remote-install/actions` (L2-5), and `../` (L6-11). Does **not** use `nock`. |
| `client/state/data-layer/wpcom-http/test/index.js`            | 56 lines, **2 tests**                                       | Used for **Part 3** because it directly exercises `nock` interceptors: `useNock()` (L27), `nock( … ).get( '/rest/v1.1/me' ).reply(…)` (L32) / `.replyWithError(…)` (L46).                                                       |

Either file is a valid choice: there are **80** `test/` directories under `client/state/data-layer/**`, and the preset's `testMatch` glob `<rootDir>/**/test/*.[jt]s?(x)` (`packages/calypso-jest/jest-preset.js:L12`) makes any of them eligible.

---

## 3. Part 1 — The cold/warm timing ratio

**Question:** run a data-layer test twice in a row; what is the ratio of first-run time to second-run time?

**Direct answer:** the first (cold) run is **~2.5× slower in wall-clock time** than the second (warm) run — measured range **2.51–2.58×** on `real`, and **2.89–2.94×** on Jest's own reported `Time`. The value is stable across three independently-cooled pairs.

### 3.1 Method

The cold/warm pair was run **three times** for stability. Before _each_ cold run the transform cache was deleted so run #1 of every pair reflects a genuine first run:

```
rm -rf .cache/jest        # guarantee a cold cache
export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js   # COLD (run #1)
export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js   # WARM (run #2, identical)
```

### 3.2 Results (3 pairs + an extra back-to-back warm run)

| Pair | COLD `real` | COLD Jest `Time` | WARM `real` | WARM Jest `Time` | ratio `real` | ratio Jest |
| ---- | ----------- | ---------------- | ----------- | ---------------- | ------------ | ---------- |
| 1    | 15.523s     | 13.684 s         | 6.011s      | 4.657 s          | 2.58×        | 2.94×      |
| 2    | 15.344s     | 13.525 s         | 6.117s      | 4.686 s          | 2.51×        | 2.89×      |
| 3    | 15.237s     | 13.454 s         | 6.081s      | 4.662 s          | 2.51×        | 2.89×      |

Extra back-to-back warm run (cache still populated): `real 6.092s` / Jest `4.696 s`.

- **COLD range:** `real` 15.24–15.52s; Jest `Time` 13.45–13.68s.
- **WARM range:** `real` 6.01–6.12s; Jest `Time` 4.66–4.70s.
- **Ratio:** **2.51–2.58× wall-clock (≈2.5×)**, **2.89–2.94× on Jest `Time` (≈2.9×)** — tightly stable across all three pairs.
- **Cold−warm delta:** ~9.2–9.5s (`real`) / ~8.8–9.0s (Jest `Time`). This delta is the transpilation cost the cache later eliminates (see Parts 2 and 4).

**Divergence from community figures (reported as observed):** public reports for large TypeScript suites frequently cite **20–30×** cold-vs-warm slowdowns. This small data-layer test does **not** reproduce that magnitude — the measured ratio is **~2.5×**. The observed value is reported as-is and **not** adjusted toward the community figure. The likely reason for the smaller ratio (inferred from code) is that a single 99-line test spends a fixed ~6s on process/worker startup, the Node test-environment (`jest-environment-node`) setup (`packages/calypso-jest/jest-preset.js:L11` sets `testEnvironment: 'node'`, with no override in the client config), and the `setupFilesAfterEnv` framework, so the variable transpilation component (~9s) yields ~2.5× rather than the 20–30× seen when transpilation dominates a very large suite.

### 3.3 Verbatim output

**COLD — Pair 1, run #1** (`rm -rf .cache/jest` first; then `export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js`):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (13.641 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        13.684 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m15.523s
user	0m20.976s
sys	0m2.135s
```

**WARM — Pair 1, run #2** (identical command; cache populated):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        4.657 s, estimated 14 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m6.011s
user	0m7.312s
sys	0m0.931s
```

**COLD — Pair 2:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (13.478 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        13.525 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m15.344s
user	0m20.712s
sys	0m1.894s
```

**WARM — Pair 2:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        4.686 s, estimated 14 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m6.117s
user	0m7.429s
sys	0m0.953s
```

**COLD — Pair 3:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (13.407 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        13.454 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m15.237s
user	0m20.972s
sys	0m1.939s
```

**WARM — Pair 3:**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        4.662 s, estimated 14 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m6.081s
user	0m7.505s
sys	0m0.972s
```

**Extra back-to-back WARM run** (immediately after Pair 3's warm run; note `estimated 5 s`):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        4.696 s, estimated 5 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m6.092s
user	0m7.449s
sys	0m1.032s
```

### 3.4 Two observations worth noting

1. **The warm run's `estimated` figure comes from the `perf-cache`.** The first warm run of each pair reports `estimated 14 s` (seeded from the preceding cold run), while the _extra_ back-to-back warm run reports `estimated 5 s` (re-seeded from the previous warm run). This is direct evidence that Jest persists per-test timing between runs — see the `perf-cache-*` file in Part 2.
2. **Cold `user` time (~21s) exceeds cold `real` time (~15.3s).** This indicates the transpilation work is parallelised across more than one thread/worker during the cold run; the warm run's `user` (~7.3s) is much closer to its `real` (~6.0s) because little CPU-bound transpilation remains.

---

## 4. Part 2 — Warm-up overhead: where the transform cache lives, and what it stores

**Question:** where is Jest's transformation cache configured, which directory does it use, which configuration option controls it, and after a run what types of files are cached?

**Direct answers:**

- **Controlling option:** Jest's **`cacheDirectory`**.
- **Where it is configured:** `test/client/jest.config.js:L7`.
- **Directory it uses:** `<repo-root>/.cache/jest`.
- **Cached file types (after a run):** (1) hash-named **transformed JS modules**, (2) `.map` **source maps**, (3) a single **`haste-map-*`** module map, and (4) a small **`perf-cache-*`** timing JSON.

### 4.1 The controlling option and directory

`test/client/jest.config.js:L7` sets `cacheDirectory` explicitly:

```
$ sed -n '1,8p' test/client/jest.config.js
const path = require( 'path' );
const base = require( '@automattic/calypso-jest' );

module.exports = {
	...base,
	rootDir: '../../client',
	cacheDirectory: path.join( __dirname, '../../.cache/jest' ),
	testPathIgnorePatterns: [ '<rootDir>/server/' ],
```

`__dirname` is `test/client/`, so `path.join( __dirname, '../../.cache/jest' )` resolves to **`<repo-root>/.cache/jest`**. That path is gitignored, so cache writes never touch tracked files:

```
$ sed -n '15p' .gitignore
/.cache/
```

### 4.2 Empty-before / populated-after (the transitional state)

**Before** a cold run the directory does not exist (this is what makes run #1 genuinely cold):

```
$ rm -rf .cache/jest
$ ls -la .cache/jest
ls: cannot access '.cache/jest': No such file or directory   # ABSENT
$ ls -la .cache
total 8
drwxr-sr-x  2 root root 4096 .
drwxr-sr-x 25 root root 4096 ..                               # .cache/ is empty
```

**After** a single cold run of the primary test, `.cache/jest` contains exactly **three** top-level entries:

```
$ yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js   # (populates cache)
$ ls -1 .cache/jest
haste-map-e11c1efbbe9931cf9309ad64a565f476-c2e14e547c851c8ab671b5187972f8c9-7e97cf74e6d7c24298140bc7bc186e4b
jest-transform-cache-e11c1efbbe9931cf9309ad64a565f476-79ef2876fae7ca75eedb2aa53dc48338
perf-cache-e11c1efbbe9931cf9309ad64a565f476-da39a3ee5e6b4b0d3255bfef95601890
```

_(The hash prefix `e11c1efbbe…` is derived from the absolute repository path, so it will differ between checkouts; the three entry **types** are invariant.)_

### 4.3 The cached file types

Counts and sizes from a single, isolated cold run of the primary test:

```
$ TC=$(ls -d .cache/jest/jest-transform-cache-*)
$ find "$TC" -type f ! -name '*.map' | wc -l     # transformed JS modules
1432
$ find "$TC" -type f -name '*.map' | wc -l       # source maps
1325
$ find .cache/jest -type f | wc -l               # total files
2759
$ du -sh .cache/jest
29M	.cache/jest
```

**Type 1 — hash-named transformed JS modules (1432 files).** These are `babel-jest`'s transpiled CommonJS output, stored under `jest-transform-cache-*/` and sharded into two-hex-character subdirectories (`00`…`ff`). Each file begins with a **32-hex integrity-hash line**, then the transpiled code. Sample (`7b/Heading_7b647c533507ed88ee17a3d2be98ec38`):

```
$ head -6 .cache/jest/jest-transform-cache-*/7b/Heading_7b647c533507ed88ee17a3d2be98ec38
ea9c704ba56cd2f4b48635fafb92fcdf
"use strict";

var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault").default;
Object.defineProperty(exports, "__esModule", {
  value: true
```

**Type 2 — `.map` source maps (1325 files).** JSON v3 source maps written alongside each transformed module (Jest transformers are expected to emit a source map for correct stack traces). Sample:

```
$ head -c 220 .cache/jest/jest-transform-cache-*/7b/Heading_7b647c533507ed88ee17a3d2be98ec38.map
{"version":3,"names":["_clsx","_interopRequireDefault","require","_jsxRuntime","Heading","text","subText","align","size","jsxs","className","clsx","left","center","children","jsx","small","exports"],"sources":["Heading.t
```

Representative shard `00` — note each code file has a sibling `.map`, **except** asset-stub outputs (e.g. `style_*`) which have none:

```
$ ls -1 .cache/jest/jest-transform-cache-*/00 | head -12
isdifmproduct_00f343c33d59c3a8f97907db77557dff
isdifmproduct_00f343c33d59c3a8f97907db77557dff.map
isjetpackproductslug_00caf917d0ececaf060709df7f1348de
isjetpackproductslug_00caf917d0ececaf060709df7f1348de.map
plans_005b87a3d8004f55165dd18f879c3882
plans_005b87a3d8004f55165dd18f879c3882.map
style_0077af9b21ef289b6c5d28b92499f152
types_00f8b760fcb676b2b4d2c69d9b3e72e8
types_00f8b760fcb676b2b4d2c69d9b3e72e8.map
```

The 1432-vs-1325 discrepancy (107 files) is explained by the **asset transformer**: `.scss/.css/.svg/.png/…` files are handled by the stub at `packages/calypso-jest/src/asset-transform.js:L3-6`, which returns only `{ code: … }` with **no** source map. The cached body of such a file is just the basename:

```
$ sed -n '2p' .cache/jest/jest-transform-cache-*/7b/style_7bc9af1692a827e378c3a23c9ea7719b
module.exports = "style.scss";
```

**Type 3 — `haste-map-*` (1 file, ~2.6 MB).** Jest's serialized module/haste map (module name → file path lookup), rebuilt when absent:

```
$ stat -c '%s bytes  %n' .cache/jest/haste-map-*
2565499 bytes  .cache/jest/haste-map-e11c1efbbe9931cf9309ad64a565f476-c2e14e547c851c8ab671b5187972f8c9-7e97cf74e6d7c24298140bc7bc186e4b
```

**Type 4 — `perf-cache-*` (1 file, ~147 bytes).** A tiny JSON mapping each test file to its last-observed pass count and duration in milliseconds; this is what drives Jest's `estimated Ns` display (see Part 1 §3.4). The `14131` below is the cold run's Jest time (14.131s):

```
$ stat -c '%s bytes  %n' .cache/jest/perf-cache-*
147 bytes  .cache/jest/perf-cache-e11c1efbbe9931cf9309ad64a565f476-da39a3ee5e6b4b0d3255bfef95601890
$ cat .cache/jest/perf-cache-*
{"/tmp/blitzy/wp-calypso/blitzy-bc99994b-43dc-413a-9432-87e7c9eff4d6_42227b/client/state/data-layer/wpcom/jetpack-install/test/index.js":[1,14131]}
```

### 4.4 Why `node_modules` is (mostly) excluded from the cache

Most third-party packages are **not** transformed and therefore never appear in the transform cache. This is governed by `transformIgnorePatterns` at `test/client/jest.config.js:L14-16`:

```
$ sed -n '14,16p' test/client/jest.config.js
	transformIgnorePatterns: [
		'node_modules[\\/\\\\](?!.*\\.(?:gif|jpg|jpeg|png|svg|scss|sass|css)$)',
	],
```

The negative-lookahead pattern excludes everything in `node_modules` **except** the listed asset extensions. This is directly relevant to Part 3 (nock is in `node_modules`, so it is never transpiled) and to Part 4 (the transpiled graph consists of in-repo source, not third-party packages). What _does_ get transpiled from the monorepo — despite living under `node_modules` as symlinks — is the `@automattic/*` workspace packages, because they resolve to their untranspiled `calypso:src` (see Part 4).

---

## 5. Part 3 — HTTP mocking: the library, its wiring, and its (bounded) first-run contribution

**Question:** which library performs the HTTP mocking, where is it configured in the test helpers, how does the mock setup affect observed timing, and does the mock library contribute to first-run overhead?

**Direct answers:**

- **Library:** **`nock`**, `^13.5.6` (`package.json:L299`; installed `13.5.6`).
- **Where configured:** globally in `test/client/setup-test-framework.js` (registered via `setupFilesAfterEnv`), plus a deprecated per-test helper `client/test-helpers/use-nock/index.js`.
- **Timing effect:** essentially a **fixed, one-time ~20 ms module-load cost**; it does **not** scale with the suite and is **not** the dominant first-run contributor.
- **Contribution to first-run overhead:** **marginal** — ~20 ms out of a ~9.3s cold−warm delta (≈0.2%).

### 5.1 The library

```
$ node -e "console.log(require.resolve('nock'))"
<repo-root>/node_modules/nock/index.js
$ node -e "console.log(require('nock/package.json').version)"
13.5.6
```

Declared at `package.json:L299`:

```
"nock": "^13.5.6",
```

### 5.2 Where and how it is wired (global setup)

`nock` is configured **once, globally** in `test/client/setup-test-framework.js`, which the client config registers via `setupFilesAfterEnv` at `test/client/jest.config.js:L21` (`'<rootDir>/../test/client/setup-test-framework.js'`). The relevant lines:

```
$ sed -n '6,22p' test/client/setup-test-framework.js
const nock = require( 'nock' );

// Disables all network requests for all tests.
nock.disableNetConnect();

beforeAll( () => {
	// reactivate nock on test start
	if ( ! nock.isActive() ) {
		nock.activate();
	}
} );

afterAll( () => {
	// helps clean up nock after each test run and avoid memory leaks
	nock.restore();
	nock.cleanAll();
} );
```

- `L6` `const nock = require( 'nock' );` — the single load site for the whole client suite.
- `L9` `nock.disableNetConnect();` — blocks _all_ real network for every test (which is why the suite needs no network access).
- `L11-16` `beforeAll(…)` — reactivates nock at the start of a run if it is not already active.
- `L18-22` `afterAll(…)` — `nock.restore()` + `nock.cleanAll()` to prevent interceptor/memory leakage between runs.

### 5.3 The deprecated per-test helper

`client/test-helpers/use-nock/index.js` re-exports `nock` and offers a convenience wrapper:

```
$ cat -n client/test-helpers/use-nock/index.js
     1	import debug from 'debug';
     2	import nock from 'nock';
     3
     4	export { nock };
     5
     6	const log = debug( 'calypso:test:use-nock' );
     7
     8	/**
     9	 * @param {Function} setupCallback Function executed before all tests are run.
    10	 * @deprecated Use nock directly instead.
    11	 */
    12	export const useNock = ( setupCallback ) => {
    13		if ( setupCallback ) {
    14			beforeAll( () => setupCallback( nock ) );
    15		}
    16		afterAll( () => {
    17			log( 'Cleaning up nock' );
    18			nock.cleanAll();
    19		} );
    20	};
    21
    22	export default useNock;
```

- `L2` imports `nock`; `L4` re-exports it; `L10` marks the helper `@deprecated` ("Use nock directly instead").
- `L12-20` `useNock()` registers a `beforeAll` (running the caller's setup) at `L14` and an `afterAll` cleanup calling `nock.cleanAll()` at `L18`.

### 5.4 Observed working example

`client/state/data-layer/wpcom-http/test/index.js` uses this helper and sets interceptors — `useNock()` (L27), `nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).reply( 200, data )` (L32), and `.replyWithError( error )` (L46). It passes:

```
$ export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom-http/test/index.js
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom-http/test/index.js

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.072 s
Ran all test suites matching /client\/state\/data-layer\/wpcom-http\/test\/index.js/i.

real	0m2.563s
user	0m3.052s
sys	0m0.591s
```

### 5.5 How the mock setup affects timing, and its first-run contribution (bounded, with evidence)

`nock` is loaded exactly once via the setup file and is a single node in the module graph. Its timing contribution is **fixed and small**, established by three facts:

**(a) `nock` is never transpiled.** It lives in `node_modules`, which `transformIgnorePatterns` (`test/client/jest.config.js:L14-16`) excludes from transformation. Searching the populated transform cache for every nock library module confirms **zero** were transpiled/cached:

```
$ TC=$(ls -d .cache/jest/jest-transform-cache-*)
$ for base in back common intercept interceptor scope recorder socket \
      playback_interceptor match_body global_emitter intercepted_request_router; do
    printf '  %-28s %s cached\n' "${base}_*" "$(find "$TC" -type f -name "${base}_*" ! -name '*.map' | wc -l)"
  done
  back_*                       0 cached
  common_*                     0 cached
  intercept_*                  0 cached
  interceptor_*                0 cached
  scope_*                      0 cached
  recorder_*                   0 cached
  socket_*                     0 cached
  playback_interceptor_*       0 cached
  match_body_*                 0 cached
  global_emitter_*             0 cached
  intercepted_request_router_* 0 cached
```

**(b) `nock` is tiny.** Its runtime is 11 JS files totalling ~136 KB — a small module graph to `require()`:

```
$ find node_modules/nock/lib -name '*.js' | wc -l
11
$ du -sh node_modules/nock/lib
136K	node_modules/nock/lib
```

**(c) Loading it is a one-time ~20 ms cost.** Measured in isolation (three trials), and contrasted with the Babel compiler's own load cost:

```
$ for i in 1 2 3; do node -e "const t0=performance.now(); require('nock'); const t1=performance.now(); console.log('trial $i  require(nock): '+(t1-t0).toFixed(1)+' ms');"; done
trial 1  require(nock): 20.1 ms
trial 2  require(nock): 20.6 ms
trial 3  require(nock): 19.3 ms
$ node -e "const t0=performance.now(); require('@babel/core'); const t1=performance.now(); console.log('require(@babel/core): '+(t1-t0).toFixed(1)+' ms');"
require(@babel/core): 76.6 ms
```

**Conclusion:** `nock`'s first-run contribution is the ~20 ms it takes to `require()` it once. Against the ~9.3s cold−warm delta measured in Part 1, that is **≈0.2%**. `nock` therefore contributes to first-run module-load cost only **marginally** and is **not** the dominant contributor — the dominant cost is `babel-jest` transpilation of the in-repo source graph (Part 4). The mock _setup_ itself (`disableNetConnect`, `beforeAll`/`afterAll`) is behavioural wiring with no measurable per-run transpilation cost, since nock is never transformed.

---

## 6. Part 4 — `--no-cache`: performance impact, and the dominant transformation step

**Question:** run the same test file with `--no-cache`; what is the performance impact of disabling the cache, and which specific transformation step consumes the most time during the uncached run?

**Direct answers:**

- **Impact:** the uncached run takes **`real ~15.7–16.1s`** — **~2.6× the warm run** (~6.0s) and **essentially equal to the cold run** (~15.4s). Disabling the cache re-imposes the full first-run cost on _every_ run.
- **Dominant step:** **`babel-jest` transpilation** of the imported source graph (`packages/calypso-jest/jest-preset.js:L14`), amplified by the `calypso:src` resolver pulling in untranspiled monorepo source.

### 6.1 `--no-cache` timing (2 runs)

```
$ export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js --no-cache
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (14.165 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        14.236 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m16.075s
user	0m22.165s
sys	0m1.995s
```

```
$ export TZ=UTC; time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js --no-cache
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/data-layer/wpcom/jetpack-install/test/index.js (13.775 s)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   1 passed, 1 total
Time:        13.82 s
Ran all test suites matching /client\/state\/data-layer\/wpcom\/jetpack-install\/test\/index.js/i.

real	0m15.652s
user	0m21.335s
sys	0m2.262s
```

### 6.2 Impact of disabling the cache

| Condition          | `real`          | Jest `Time`     | vs warm (`real`) |
| ------------------ | --------------- | --------------- | ---------------- |
| Warm (cached)      | ~6.0s           | ~4.66s          | 1.0×             |
| **`--no-cache`**   | **~15.7–16.1s** | **~13.8–14.2s** | **~2.6×**        |
| Cold (empty cache) | ~15.2–15.5s     | ~13.45–13.68s   | ~2.5×            |

- **`--no-cache` vs warm:** `real` **~2.64×** slower (+~9.8s); on Jest `Time` **~3.0×** (~14.0s vs ~4.66s).
- **`--no-cache` vs cold:** **comparable** — the uncached run lands right on top of the cold run (in these measurements it is marginally higher, within noise). Both must transpile the full source graph; the difference is that `--no-cache` disables cache **reads** (forcing full re-transpilation on every run — confirmed by consecutive `--no-cache` runs never speeding up) but still **writes** the transform cache and haste-map to disk. The tiny `--no-cache`-vs-cold gap is run-to-run noise, not write-avoidance.

This is the key confirmation: the warm speedup is **entirely** the reuse of cached transforms. Remove the cache (`--no-cache`) and the run reverts to cold-run cost.

### 6.3 The dominant step is `babel-jest` transpilation

**(1) The transform map makes `babel-jest` the only meaningful transformer.** `packages/calypso-jest/jest-preset.js:L13-16`:

```
$ sed -n '13,16p' packages/calypso-jest/jest-preset.js
	transform: {
		'\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
		'\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
	},
```

Every `.js/.jsx/.ts/.tsx` file is transformed by **`babel-jest`** (L14). The only other transformer is the asset stub (L15), which — as shown in Part 2 §4.3 — merely returns the file's basename (`packages/calypso-jest/src/asset-transform.js:L3-6`) at negligible cost. So essentially **all** real transform work is `babel-jest`.

**(2) The `calypso:src` resolver amplifies the cost by pulling in untranspiled source.** `packages/calypso-jest/src/module-resolver.js:L16-20`:

```
$ sed -n '16,20p' packages/calypso-jest/src/module-resolver.js
const resolver = enhancedResolve.create.sync( {
	extensions: [ '.json', '.js', '.jsx', '.ts', '.tsx' ],
	mainFields: [ 'calypso:src', 'main' ],
	conditionNames: [ 'calypso:src', 'node', 'require' ],
} );
```

`calypso:src` is **first** in `mainFields` (L18) and `conditionNames` (L19). The resolver's own comment (`module-resolver.js:L8-10`) states that `calypso:src` "points to the _untranspiled_ source code." Verified against `@automattic/state-utils` (imported by the wpcom-http test at L1) — its `main` target does **not** exist, so only the untranspiled `.ts` source is resolvable:

```
$ node -e "const p=require('@automattic/state-utils/package.json'); console.log('calypso:src=',p['calypso:src'],'| main=',p.main)"
calypso:src= src/index.ts | main= dist/cjs/index.js
$ ls node_modules/@automattic/state-utils/dist/cjs/index.js 2>&1
ls: cannot access 'node_modules/@automattic/state-utils/dist/cjs/index.js': No such file or directory
$ ls node_modules/@automattic/state-utils/src/index.ts
node_modules/@automattic/state-utils/src/index.ts
```

_(The package's `dist/` directory exists but contains only `dist/esm/` and `dist/types/` — there is no `dist/cjs/`, so the `main` field target is absent and `calypso:src` = `src/index.ts` is what resolves. The package is a symlink to `<repo-root>/packages/state-utils`.)_ Consequently, in-repo `@automattic/*` packages enter Jest as **raw TypeScript/JSX source** and must be Babel-compiled on every cold/uncached run.

**(3) The scale: ~1432 modules, and per-file Babel cost.** A single cold run transpiles **1432** modules (Part 2 §4.3). A direct measurement of one representative transform — `@automattic/state-utils/src/index.ts` compiled via the repo's `babel.config.js` with `envName='test'` and the `babel-jest` caller — shows the shape of the cost (first call pays one-time preset/plugin warm-up; subsequent calls are the amortized per-file cost):

```
$ node -e "
const babel=require('@babel/core'), fs=require('fs'), path=require('path');
const file=path.resolve('packages/state-utils/src/index.ts');
const src=fs.readFileSync(file,'utf8');
const opts={filename:file,envName:'test',configFile:path.resolve('babel.config.js'),caller:{name:'babel-jest',supportsStaticESM:false,supportsDynamicImport:false}};
for(let i=1;i<=3;i++){const t0=performance.now();const out=babel.transformSync(src,opts);const t1=performance.now();console.log('transform #'+i+'  '+(t1-t0).toFixed(1)+' ms  (in='+src.length+'B out='+out.code.length+'B)');}
"
transform #1  261.6 ms  (in=252B out=1019B)
transform #2  6.8 ms  (in=252B out=1019B)
transform #3  6.2 ms  (in=252B out=1019B)
```

The first transform is **261.6 ms** (it triggers one-time compilation of the preset/plugin stack); the amortized per-file cost thereafter is **~6–7 ms**. At **~6–7 ms × 1432 modules ≈ 9–10 s**, this squarely accounts for the observed ~9.2–9.5 s cold−warm delta — i.e. the cold/uncached overhead **is** the aggregate `babel-jest` transpilation of the resolved source graph.

**(4) The Babel stack that runs per file.** `babel.config.js:L2,L6-10` delegates to `@automattic/calypso-babel-config`. Its default preset (`packages/calypso-babel-config/presets/default.js:L15-36`) composes:

```
$ sed -n '15,36p' packages/calypso-babel-config/presets/default.js
	presets: [
		[
			require.resolve( '@babel/preset-env' ),
			{
				corejs: opts.corejs ? opts.corejs : 3.6,
				debug: opts.debug ? opts.debug : false,
				bugfixes: opts.bugfixes ? opts.bugfixes : false,
				modules: modulesOption( opts ),
				useBuiltIns: opts.useBuiltIns ? opts.useBuiltIns : 'entry',
				// Exclude transforms that make all code slower, see https://github.com/facebook/create-react-app/pull/5278
				exclude: [ 'transform-typeof-symbol' ],
			},
		],
		[
			require.resolve( '@babel/preset-react' ),
			{
				runtime: 'automatic',
				importSource: opts.importSource ?? process.env.IMPORT_SOURCE,
			},
		],
		[ require.resolve( '@babel/preset-typescript' ), { allowDeclareFields: true } ],
	],
```

- `@babel/preset-env` (L17), `@babel/preset-react` (L29, `runtime: 'automatic'` at L31), `@babel/preset-typescript` (L35).
- Plugins (`default.js:L37-53`): `@babel/plugin-proposal-class-properties` (L38), `@babel/plugin-transform-runtime` (L40), `@automattic/babel-plugin-preserve-i18n` (L51), `@emotion/babel-plugin` (L52).
- In the **`test`** environment, `packages/calypso-babel-config/config.js:L22-25` adds a `@babel/preset-env` targeting `{ node: 'current' }` and the `babel-plugin-dynamic-import-node` plugin:

```
$ sed -n '22,25p' packages/calypso-babel-config/config.js
		test: {
			presets: [ [ '@babel/preset-env', { targets: { node: 'current' } } ] ],
			plugins: [ 'babel-plugin-dynamic-import-node' ],
		},
```

Running `preset-typescript` + `preset-env` + `preset-react` + this plugin chain over **every** file in a ~1400-module source graph is precisely the work that the cache eliminates on warm runs and that `--no-cache` re-imposes.

---

## 7. Causal model — why the cold run is slow

Putting the four parts together, the cold/warm inconsistency follows a single, evidenced chain:

```
custom resolver (module-resolver.js:L16-20)
      │  mainFields: [ 'calypso:src', 'main' ]  → resolves @automattic/* to UNTRANSPILED source
      ▼
imported source graph is raw .ts/.tsx/.jsx (e.g. state-utils/src/index.ts; dist/cjs absent)
      │
      ▼
babel-jest (jest-preset.js:L14) must transpile EVERY .[jt]sx? file
      │  preset-typescript + preset-env + preset-react + plugins
      │  (calypso-babel-config presets/default.js:L15-53, config.js:L22-25)
      ▼
~1432 modules compiled on a cold run  (≈ 6–7 ms each ≈ 9–10 s aggregate)
      │
      ▼
results written to <repo-root>/.cache/jest  (cacheDirectory, jest.config.js:L7)
      │  1432 transformed JS + 1325 source maps + haste-map + perf-cache  (2759 files, 29 MB)
      ▼
WARM run reads the cache → transpilation skipped → ~6 s   (≈2.5× faster)
```

Supporting/negative results that complete the picture:

- **`nock`** (Part 3) sits in this graph as one small `node_modules` node that is **excluded from transformation** (`transformIgnorePatterns`, `jest.config.js:L14-16`) and costs a fixed ~20 ms to load — so it is explicitly **not** part of the dominant cost.
- **`--no-cache`** (Part 4) short-circuits the "read the cache" step, so it reverts to full transpilation (~2.6× the warm run), proving the warm speedup is cache reuse.
- The specific function doing the expensive work is **`babel-jest`**'s transform (registered at `jest-preset.js:L14`), invoking **`@babel/core`**'s `transformSync` with the `test`-env preset/plugin stack.

---

## 8. Coverage pass

A re-read of the four-part question, confirming each named item is answered with a value, a `file:line` reference, observed evidence, and a causal reason.

- [x] **Part 1 — timing ratio.** Reported **2.51–2.58× wall-clock** (≈2.5×) and **2.89–2.94× Jest `Time`** (≈2.9×) across **3** cold/warm pairs + an extra warm run (§3.2); run scale and per-run `real`/`user`/`sys` stated; **verbatim** cold and warm output for all pairs shown (§3.3); divergence from the community 20–30× figure reported honestly as observed (§3.2).
- [x] **Part 2 — warm-up overhead / cache.**
  - **Where configured / which option:** `cacheDirectory` at **`test/client/jest.config.js:L7`** (§4.1).
  - **Which directory:** **`<repo-root>/.cache/jest`** (§4.1), gitignored (`.gitignore:L15`).
  - **File types cached:** (1) 1432 hash-named transformed JS modules, (2) 1325 `.map` source maps, (3) one ~2.6 MB `haste-map-*`, (4) one ~147 B `perf-cache-*` — with `ls`/`find`/`cat` evidence and empty-before/populated-after states (§4.2–4.3).
  - **`transformIgnorePatterns`** at `test/client/jest.config.js:L14-16` explaining the `node_modules` exclusion (§4.4).
- [x] **Part 3 — HTTP mocking.**
  - **Library:** **`nock`** (`package.json:L299`) (§5.1).
  - **Where configured:** `test/client/setup-test-framework.js:L6/L9/L11-16/L18-22` and the deprecated helper `client/test-helpers/use-nock/index.js:L2/L4/L10/L12-20` (§5.2–5.3), with a passing `wpcom-http` run (§5.4).
  - **How it affects timing / first-run contribution:** fixed ~20 ms one-time `require('nock')`, never transpiled (0 cached lib files), **≈0.2%** of the ~9.3 s delta → **not** dominant (§5.5).
- [x] **Part 4 — `--no-cache` & dominant step.**
  - **Impact:** **~2.6× slower than warm**, **≈ cold** — verbatim `--no-cache` output ×2 and a comparison table (§6.1–6.2).
  - **Dominant step:** **`babel-jest` transpilation** (`jest-preset.js:L14`), grounded in the transform map, the `calypso:src`→untranspiled resolver (`module-resolver.js:L16-20`), the 1432-module scale, a per-file Babel-cost measurement (261.6 ms warm-up → ~6–7 ms amortized), and the full preset/plugin stack (`presets/default.js:L15-53`, `config.js:L22-25`) (§6.3).
- [x] **Reproducibility artifacts:** exact commands (§2.2), the `time` caveat (§2.3), environment/versions (§2.1), and the causal model (§7).

### Reproduce it yourself

```
corepack enable && yarn install --immutable          # if node_modules is absent
rm -rf .cache/jest                                    # force a cold cache
export TZ=UTC
time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js   # COLD  (~15s)
time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js   # WARM  (~6s)
ls -R .cache/jest                                     # inspect cached artifacts
time yarn jest -c=test/client/jest.config.js client/state/data-layer/wpcom/jetpack-install/test/index.js --no-cache   # UNCACHED (~16s)
```
