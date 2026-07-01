# Why Jest test-execution times differ between the first (cold) and second (warm) run in `client/state/data-layer`

> **Investigation type:** read-only, run-first Q&A. Every number below was produced by *actually running* the repository's own pinned toolchain and is quoted verbatim alongside the exact command that produced it. Every literal (config key, string value, file path, measured number) is cited with its `file:line` reference. Nothing in the repository was modified — the sole write is this document.

The subject under investigation is the Calypso **data layer**, confirmed by `client/state/data-layer/README.md:L1`:

```
# Calypso Data Layer
```

---

## Executive summary (the stable answer: ratios)

Absolute millisecond values vary with host load, so the **ratios** — not the raw milliseconds — are the stable, reportable quantities. All runs used the client Jest config and were measured on this host with Node `v22.23.1` and Jest `29.7.0`.

| Question | Measurement | Result |
|---|---|--:|
| **Q1** first/second (cold ÷ warm), single file | Jest `Time:` 1.567 s ÷ 0.896 s | **≈ 1.75×** |
| **Q1** first/second (cold ÷ warm), full `data-layer` suite | Jest `Time:` 23.325 s ÷ 12.875 s | **≈ 1.81×** |
| **Q4** `--no-cache` ÷ warm, single file | Jest `Time:` 1.539 s ÷ 0.896 s | **≈ 1.72×** |
| **Q4** `--no-cache` ÷ cold, single file | Jest `Time:` 1.539 s ÷ 1.567 s | **≈ 0.98×** (`--no-cache` ≈ cold) |

- **Q2 — cache:** the controlling option is **`cacheDirectory`** at `test/client/jest.config.js:L7`, resolving to **`<repo>/.cache/jest`**. It holds three artifact types: a `haste-map-*` module map, a `jest-transform-cache-*/` tree of `babel-jest` transformed code **plus sibling `.map` source maps**, and a single `perf-cache-*` JSON file.
- **Q3 — mocking:** the library is **`nock`** (`package.json:L299` = `"nock": "^13.5.6",`), configured globally in `test/client/setup-test-framework.js`. Because `nock` lives in `node_modules` (which Jest does not transform), it is **not** a material driver of first-run overhead — only a fixed per-run module-load cost.
- **Q4 — dominant step:** the dominant cost of an uncached run is **`babel-jest` transpiling first-party TypeScript/JSX** (`packages/calypso-jest/jest-preset.js:L14`). This aligns with Jest's documented guidance that disabling the cache makes it "at least two times slower" (Jest CLI Options, <https://jestjs.io/docs/cli#--cache>; see Answer 4).

---

## Investigation environment & methodology

### Runtime (intentional deviation from the setup script)

The repository pins Node and requires it via `engines`:

- `.nvmrc:L1` = `22.9.0`
- `package.json:L57` = `"node": "^v22.9.0",`

The environment's provided setup script installs Node `20.x`, which **would fail** the `engines.node` `^v22.9.0` requirement (and therefore break `yarn install` / Jest). The investigation therefore intentionally used a **Node v22.x** runtime that satisfies the pin. Verified verbatim:

```
$ node --version
v22.23.1
```

`v22.23.1` satisfies `^v22.9.0`. This deviation from the Node-20 setup script is deliberate and necessary; it is noted here for fidelity. The package manager is Yarn **4.0.2** (`package.json:L422` = `"packageManager": "yarn@4.0.2",`) with `nodeLinker: node-modules` (`.yarnrc.yml:L3` = `nodeLinker: node-modules`). Dependencies were already installed, so `node_modules/.bin/jest` and `node_modules/nock` were present; `yarn install` was not re-run.

### Why the `-c=test/client/jest.config.js` flag is required

The root `package.json` has **no top-level `jest` key** (verified — the command prints nothing):

```
$ grep -n '^  "jest"' package.json
$
```

Instead, tests are launched through per-area scripts. The relevant one is `package.json:L122`:

```
"test-client": "TZ=UTC jest -c=test/client/jest.config.js",
```

`client/state/data-layer` is governed by the **client** suite, so every measurement used that config explicitly, with `--ci` to prevent watch mode:

```
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js <path> --ci
```

Jest itself was confirmed functional and at the pinned major:

```
$ TZ=UTC node_modules/.bin/jest --version
29.7.0
```

`29.7.0` matches `package.json:L290` = `"jest": "^29.7.0",`.

### Run-first methodology

All timing, cache-anatomy, and mocking conclusions were produced by **running** the code first and capturing real output. Temporary measurement scripts were kept **outside** the repository (under `/tmp`) and removed afterward. Because `/.cache/` is git-ignored (`.gitignore:L15` = `/.cache/`), generating and clearing the transform cache never dirties the tracked tree — confirmed after all runs:

```
$ git status --porcelain
$
```

The tree was clean before and after every run (empty `git status --porcelain`).

### Representative test file

The single file measured is `client/state/data-layer/wpcom-http/test/index.js` (2 tests inside 1 `describe` suite). It was chosen because it is the one `data-layer` test that uses `nock` **directly**, so it serves both the timing question **and** the mocking question. Its import line is `client/state/data-layer/wpcom-http/test/index.js:L2`:

```
import useNock, { nock } from 'calypso/test-helpers/use-nock';
```

The suite header is at `L26` (`describe( '#queueRequest', () => {`), it calls `useNock()` at `L27`, and the two tests are at `L29` and `L43`.

---

## Answer 1 — First-run vs. second-run timing and the first/second ratio

**Question:** Run any `data-layer` test file twice; measure wall-clock for each; report the first-run ÷ second-run ratio.

**Method.** A `/tmp` script (kept outside the repo and removed afterward) wrapped each Jest invocation with `date +%s%N` deltas for wall-clock, writing Jest's full output (including its own `Time:` marker, printed to stderr) to a log via `>"$logfile" 2>&1`. The **first** run was made genuinely cold by deleting the transform cache first; the **second** run reused the warm cache. The wrapper, quoted verbatim:

```
$ cat /tmp/blitzy_timing/measure.sh
#!/usr/bin/env bash
# Timing wrapper: captures wall-clock (ms) + Jest's own Time: marker in one invocation.
# Usage: measure.sh <label> <logfile> -- <jest-args>
set -u
label="$1"; shift
logfile="$1"; shift
shift   # drop the literal --
start_ns=$(date +%s%N)
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js "$@" --ci >"$logfile" 2>&1
status=$?
end_ns=$(date +%s%N)
wall_ms=$(( (end_ns - start_ns) / 1000000 ))
echo "WALL_CLOCK_MS=${wall_ms}"
echo "EXIT_STATUS=${status}"
```

Every wall-clock figure below is the verbatim `WALL_CLOCK_MS=` line printed by this wrapper, shown next to the exact `bash measure.sh …` command that produced it, and the Jest `Time:` marker is read back verbatim from the same run's log via `grep`.

### Cold run (first) — cache deleted immediately before

```
$ rm -rf .cache/jest
$ bash /tmp/blitzy_timing/measure.sh single-cold /tmp/blitzy_timing/a1_cold.log -- client/state/data-layer/wpcom-http/test/index.js
WALL_CLOCK_MS=2581
EXIT_STATUS=0
$ grep -E "Test Suites:|Tests:|Snapshots:|Time:" /tmp/blitzy_timing/a1_cold.log
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.567 s
```

Measured wall-clock: **2581 ms** (the verbatim `WALL_CLOCK_MS=2581` above); Jest's own marker: `Time:        1.567 s`.

### Warm run (second) — same command, cache present

```
$ bash /tmp/blitzy_timing/measure.sh single-warm /tmp/blitzy_timing/a2_warm.log -- client/state/data-layer/wpcom-http/test/index.js
WALL_CLOCK_MS=1540
EXIT_STATUS=0
$ grep -E "Test Suites:|Tests:|Snapshots:|Time:" /tmp/blitzy_timing/a2_warm.log
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        0.896 s, estimated 2 s
```

Measured wall-clock: **1540 ms** (the verbatim `WALL_CLOCK_MS=1540` above); Jest's own marker: `Time:        0.896 s, estimated 2 s`.

### First/second ratio (single file)

- **Jest `Time:` ratio** = `1.567 s ÷ 0.896 s` = **≈ 1.75×**
- **Wall-clock ratio** = `2581 ms ÷ 1540 ms` = **≈ 1.68×**

### Full `data-layer` suite (to show the "significant difference" at scale)

Running the whole module (`state/data-layer/`, path relative to `rootDir` = `client` per `test/client/jest.config.js:L6` = `rootDir: '../../client',`). Cold run first (cache deleted), then warm — each through the same wrapper:

```
$ rm -rf .cache/jest
$ bash /tmp/blitzy_timing/measure.sh suite-cold /tmp/blitzy_timing/b1_cold.log -- state/data-layer/
WALL_CLOCK_MS=24219
EXIT_STATUS=0
$ grep -E "Test Suites:|Tests:|Snapshots:|Time:" /tmp/blitzy_timing/b1_cold.log
Test Suites: 92 passed, 92 total
Tests:       439 passed, 439 total
Snapshots:   1 passed, 1 total
Time:        23.325 s
```

```
$ bash /tmp/blitzy_timing/measure.sh suite-warm /tmp/blitzy_timing/b2_warm.log -- state/data-layer/
WALL_CLOCK_MS=13457
EXIT_STATUS=0
$ grep -E "Test Suites:|Tests:|Snapshots:|Time:" /tmp/blitzy_timing/b2_warm.log
Test Suites: 92 passed, 92 total
Tests:       439 passed, 439 total
Snapshots:   1 passed, 1 total
Time:        12.875 s, estimated 23 s
```

Both runs pass identically (92 suites, 439 tests, 1 snapshot). Measured wall-clock: cold **24219 ms**, warm **13457 ms** (the verbatim `WALL_CLOCK_MS=` lines above).

- **Full-suite Jest `Time:` ratio** = `23.325 s ÷ 12.875 s` = **≈ 1.81×**
- **Full-suite wall-clock ratio** = `24219 ms ÷ 13457 ms` = **≈ 1.80×**

### Why the second run is faster (rationale)

The first run must transform every first-party source file it loads; the second run reuses the transform outputs cached under `<repo>/.cache/jest` (see Answer 2). Jest re-runs a transformer for a file only when that file has changed, so a warm cache turns transpilation into a cheap cache read. This is exactly the ~1.75× (single file) to ~1.81× (full suite) cold/warm gap observed above.

> **Note on determinism.** Absolute milliseconds depend on host load and will differ from run to run; the **ratios** (~1.75× single file, ~1.81× full suite) are the stable answer the question asks for. These are in the same range as the reference expectation (~1.7–2.1×).

---

## Answer 2 — Where Jest's transformation cache is configured, its directory, the controlling option, and what it caches

**Question:** Find where Jest's transformation cache is configured, what directory it uses, and what option controls it; then inspect the directory and report the *types* of files cached.

### Controlling option and directory

The controlling option is **`cacheDirectory`**, set in the client suite config at `test/client/jest.config.js:L7`:

```
cacheDirectory: path.join( __dirname, '../../.cache/jest' ),
```

Because that file lives at `test/client/`, `path.join( __dirname, '../../.cache/jest' )` resolves to **`<repo>/.cache/jest`**. Surrounding context in the same config confirms how the suite is assembled:

- `test/client/jest.config.js:L2` = `const base = require( '@automattic/calypso-jest' );` — inherits the shared preset.
- `test/client/jest.config.js:L6` = `rootDir: '../../client',` — why the path argument `state/data-layer/` is relative to `client/`.
- `test/client/jest.config.js:L21` = `setupFilesAfterEnv: [ '<rootDir>/../test/client/setup-test-framework.js' ],` — loads the global `nock` setup (Answer 3).

`cacheDirectory` is the documented Jest option that controls the cache location; the transformer that writes into it (`babel-jest`) is defined in the inherited preset (Answer 4).

### The directory is git-ignored (so runs never dirty the tree)

`/.cache/` is ignored, proven verbatim:

```
$ git check-ignore -v .cache/jest
.gitignore:15:/.cache/	.cache/jest
```

That is `.gitignore:L15` = `/.cache/`. This is why clearing/regenerating the cache leaves `git status --porcelain` empty.

### What is actually cached — three artifact types

After running the full `data-layer` suite, the top level of `<repo>/.cache/jest` contained exactly three kinds of artifact. Listed with `find -printf` (deterministic: `%y` = entry type `d`/`f`, `%s` = size in bytes, `%p` = path — no timestamps, so the output is stable and fully verbatim):

```
$ find .cache/jest -maxdepth 1 -mindepth 1 -printf '%y %s %p\n' | sort
d 4096 .cache/jest/jest-transform-cache-015b28750f11b81093ccf79d1ca1a301-79ef2876fae7ca75eedb2aa53dc48338
f 13874 .cache/jest/perf-cache-015b28750f11b81093ccf79d1ca1a301-da39a3ee5e6b4b0d3255bfef95601890
f 2684319 .cache/jest/haste-map-015b28750f11b81093ccf79d1ca1a301-947c5144d86d896d5eee14b309e7917d-357458895ccd6b8e8f4bcd00096c5d2f
```

So: one directory (`jest-transform-cache-*`, 4096-byte dir entry), and two files — `perf-cache-*` (13874 bytes) and `haste-map-*` (2684319 bytes ≈ 2.68 MB). All three filenames embed the same Jest config id (`015b28750f11b81093ccf79d1ca1a301`).

**Type 1 — `haste-map-*` (a single Jest module map).** A ~2.68 MB binary file. It is **not** UTF-8 JSON; it is a **V8-serialized** `jest-haste-map` structure. `file` reports it as opaque data:

```
$ file .cache/jest/haste-map-*
.cache/jest/haste-map-015b28750f11b81093ccf79d1ca1a301-947c5144d86d896d5eee14b309e7917d-357458895ccd6b8e8f4bcd00096c5d2f: data
```

**Type 2 — `jest-transform-cache-*/<2-hex>/<name>_<hash>` (the `babel-jest` transformed module code, plus sibling `.map` source maps).** This is the directory that makes the warm run fast. It is sharded into 256 two-hex buckets; each transformed module is stored as `<name>_<hash>`, accompanied by a sibling `<name>_<hash>.map` source map. A representative bucket, listed by name (sorted, no timestamps — fully verbatim):

```
$ ls .cache/jest/jest-transform-cache-*/86 | sort | head
actions_866422276d3cd80e59c0965827b28f55
actions_866422276d3cd80e59c0965827b28f55.map
findthemefilterterm_860664356bcd068a5c6add6620ddd662
findthemefilterterm_860664356bcd068a5c6add6620ddd662.map
getthemerequesterrors_86ac55fde5ea681906806350dba23c10
getthemerequesterrors_86ac55fde5ea681906806350dba23c10.map
index_8653da430c4f6d5f2e315677efb9df08
index_8653da430c4f6d5f2e315677efb9df08.map
persistenceutils_86f93a4e937bea4366a92d61594822c5
persistenceutils_86f93a4e937bea4366a92d61594822c5.map
```

Each transformed code file is paired with a `.map` sibling, as the listing shows. Taking the `data-layer` test `index_8653da430c4f6d5f2e315677efb9df08` (a transformed first-party `data-layer` test module — its transpiled body is shown next) as the representative example, its two artifacts and their byte sizes are:

```
$ find .cache/jest/jest-transform-cache-*/86 -maxdepth 1 -name 'index_8653*' -printf '%s %f\n' | sort -n
4720 index_8653da430c4f6d5f2e315677efb9df08.map
8650 index_8653da430c4f6d5f2e315677efb9df08
```

The head of the transformed code file shows a leading integrity hash on the first line, then Babel's CommonJS output (ESM `import` lowered to `require`) — the first 300 bytes verbatim (a true byte-prefix via `head -c 300`, not an abridgement):

```
$ head -c 300 .cache/jest/jest-transform-cache-*/86/index_8653da430c4f6d5f2e315677efb9df08
ff7f1590c8779341e29cff2bfcd4bb4e
"use strict";

var _utils = require("calypso/state/data-layer/utils");
var _actions = require("calypso/state/data-layer/wpcom-http/actions");
var _actions2 = require("calypso/state/reader/follows/actions");
var _ = require("../");
describe('comment-email-subscription
```

This is unmistakably first-party `data-layer` source — its transpiled body calls `require("calypso/state/data-layer/utils")` and `require("calypso/state/data-layer/wpcom-http/actions")` (both visible in the block above), confirming that the cache holds transpiled workspace code — exactly the work the warm run skips. Its sibling `.map` is a source-map v3 JSON document — first 120 bytes verbatim:

```
$ head -c 120 .cache/jest/jest-transform-cache-*/86/index_8653da430c4f6d5f2e315677efb9df08.map
{"version":3,"names":["_utils","require","_actions","_actions2","_","describe","test","action","subscribeToNewPostEmail"
```

Counts across the whole transform cache after the full `data-layer` run:

```
$ find .cache/jest/jest-transform-cache-* -type f ! -name '*.map' | wc -l
2095
$ find .cache/jest/jest-transform-cache-* -type f -name '*.map' | wc -l
1981
$ find .cache/jest/jest-transform-cache-* -mindepth 1 -maxdepth 1 -type d | wc -l
256
```

So **2095 transformed code files + 1981 `.map` source maps** across **256** two-hex bucket directories.

**Type 3 — `perf-cache-*` (a single JSON file of per-file test-run performance).** One JSON file (13874 bytes) with 92 entries — one per test suite in the run. Each entry maps an absolute test path to a two-element array. The entry for the representative test, quoted verbatim (the exact `grep` command and its output):

```
$ grep -o '"[^"]*wpcom-http/test/index.js":\[[0-9]*,[0-9]*\]' .cache/jest/perf-cache-*
"/tmp/blitzy/wp-calypso/blitzy-0abb4dec-5b2f-4c98-a55c-fd66cf28682e_970ba2/client/state/data-layer/wpcom-http/test/index.js":[1,105]
```

That two-element array is **`[status, duration_ms]`**. The decisive verbatim evidence: across all 92 entries the **first** value is invariably `1`, while the **second** value ranges over three orders of magnitude (63 ms to 5382 ms). That pattern is consistent only with `[status, duration]` — a transform time would not be a constant `1` ms for 92 different files, and a file size would not fall in the `63`–`5382` range for these modules:

```
$ python3 -c "import json,glob; d=json.load(open(glob.glob('.cache/jest/perf-cache-*')[0])); print('entries=',len(d)); print('distinct first values=',sorted({v[0] for v in d.values()})); print('duration min/max=',min(v[1] for v in d.values()),'/',max(v[1] for v in d.values()))"
entries= 92
distinct first values= [1]
duration min/max= 63 / 5382
```

**Grounding in Jest's own source (installed `@jest/test-sequencer`, version `29.7.0`).** The sequencer defines the status constants and writes each entry as `[status, runtime]`:

```
$ sed -n '92,93p;268,270p' node_modules/@jest/test-sequencer/build/index.js
const FAIL = 0;
const SUCCESS = 1;
        cache[testResult.testFilePath] = [
          testResult.numFailingTests > 0 ? FAIL : SUCCESS,
          testRuntime || 0
```

and reads them back with the status at index `0` and the duration at index `1`:

```
$ sed -n '279,285p' node_modules/@jest/test-sequencer/build/index.js
    const cache = this._getCache(test);
    return cache[test.path]?.[0] === FAIL;
  }
  time(test) {
    const cache = this._getCache(test);
    return cache[test.path]?.[1];
  }
```

So `hasFailed` reads index `0` (the status) and `time` reads index `1` (the duration); the observed `[1,105]` therefore means **[SUCCESS, 105 ms]** for that test file.

> **Reconciliation with the planning assumption (`[transformTime, size]`).** An upstream planning note (AAP §0.3.4) described this array as `[transformTime, size]`, with the worked example `[1,874]`. Measured against the installed sequencer source and the live cache above, the array is actually **`[status, duration_ms]`** — a *test-run* record emitted by `@jest/test-sequencer` (used to order slow tests first), which is a **different cache from the `babel-jest` transform cache (Type 2)** that actually drives the warm-run speedup. The two framings reconcile on the AAP's own example: `[1,874]` reads identically under the correct semantics as **`[SUCCESS (=1), 874 ms]`** — i.e., the leading `1` that the planning note read as a "transform time" is in fact the `SUCCESS` status constant (`@jest/test-sequencer:L93`), and the second number is a run duration in milliseconds, not a byte size. The corrected wording is used here — rather than the planning phrase — because the deliverable's binding rule requires values to be *exact and grounded* and to "say so explicitly" when a prior assumption cannot be verified; both the observed data and Jest's installed source contradict `[transformTime, size]`.

### Summary for Answer 2

- **Option:** `cacheDirectory` (`test/client/jest.config.js:L7`).
- **Directory:** `<repo>/.cache/jest` (git-ignored via `.gitignore:L15` = `/.cache/`).
- **Cached file types:** (1) a `haste-map-*` V8-serialized module map; (2) a `jest-transform-cache-*/` tree of `babel-jest` transformed CommonJS code **with sibling `.map` source maps** (2095 + 1981 here); (3) a `perf-cache-*` JSON of per-file `[status, duration_ms]`. The warm-run speedup comes specifically from reuse of the **Type 2** transform outputs.

---

## Answer 3 — The HTTP-mocking library, where it is configured, and its effect on timing

**Question:** Identify the HTTP-mocking library, trace where it is configured in the test helpers, explain how the mock setup affects the observed timing, and determine whether the mock library contributes to first-run overhead.

### The library is `nock`

`nock` is declared in the root `package.json:L299`:

```
"nock": "^13.5.6",
```

and the installed/resolved version is `13.5.6` (`node_modules/nock/package.json` → `"version": "13.5.6",`).

### Global configuration (in the client test framework setup)

`nock` is wired up **globally** for every client test in `test/client/setup-test-framework.js`, which the client config loads via `setupFilesAfterEnv` (`test/client/jest.config.js:L21`). The relevant lines, verbatim:

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

- `test/client/setup-test-framework.js:L6` = `const nock = require( 'nock' );`
- `test/client/setup-test-framework.js:L9` = `nock.disableNetConnect();` — global network isolation for all tests.
- `L11-16` — a `beforeAll` that reactivates nock when inactive (`L13` = `if ( ! nock.isActive() ) {`, `L14` = `nock.activate();`).
- `L18-22` — an `afterAll` that restores/cleans (`L20` = `nock.restore();`, `L21` = `nock.cleanAll();`).

### Per-test helper (deprecated) and its use in the representative test

There is also a per-test helper at `client/test-helpers/use-nock/index.js`. It re-exports `nock` and provides a `useNock()` wrapper, and it is explicitly **deprecated**:

- `client/test-helpers/use-nock/index.js:L2` = `import nock from 'nock';`
- `L4` = `export { nock };`
- `L10` = `@deprecated Use nock directly instead.`
- `L12` = `export const useNock = ( setupCallback ) => {`

The representative test consumes this helper and mocks the WordPress.com REST API directly. The mock setup line is at **`client/state/data-layer/wpcom-http/test/index.js:L32`** (verbatim):

```
nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).reply( 200, data );
```

and the error-path mock at `L46`:

```
nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).replyWithError( error );
```

> **Citation correction.** The mock is at **L32**, not "L28–L31" as an upstream planning table stated. Re-verified against the live file at authoring time (the `describe` opens at `L26`, `useNock()` is `L27`, the first `test(...)` is `L29`, and the `nock(...).reply(200, data)` call is `L32`).

### How the mock setup affects the observed timing (rationale)

Two independent facts determine `nock`'s timing role:

1. **`nock` lives in `node_modules`, and Jest does not transform `node_modules`.** The client config's `transformIgnorePatterns` explicitly excludes it — `test/client/jest.config.js:L14-16`:

   ```
   transformIgnorePatterns: [
   	'node_modules[\\/\\\\](?!.*\\.(?:gif|jpg|jpeg|png|svg|scss|sass|css)$)',
   ],
   ```

   The negative-lookahead pattern means everything under `node_modules` **except** image/style assets is *not* passed through `babel-jest`. Therefore `nock` is **never** written into the `jest-transform-cache-*` tree (Answer 2, Type 2) and contributes **nothing** to the transform cost that differs between cold and warm runs.

2. **`nock` only intercepts in-process HTTP; it performs no real network I/O.** With `nock.disableNetConnect()` (`L9`) plus the per-test `.reply(...)`/`.replyWithError(...)` interceptors, requests are answered from memory. There is no socket latency to inflate either run.

**Verdict:** `nock` incurs only a **fixed, per-run module-load cost** (loading an already-compiled `node_modules` package) that is essentially identical on the cold and warm runs. It is therefore **not** a material driver of first-run (cold) overhead. The cold/warm gap in Answer 1 is explained by first-party **transform** work (Answer 2 / Answer 4), not by the mocking library. This conclusion is grounded in the exclusion pattern at `test/client/jest.config.js:L14-16` and the absence of any `nock` entry in the transform cache — not on assumption.


---

## Answer 4 — `--no-cache` vs. the cached run, and the dominant transformation step

**Question:** Re-run the same file with `--no-cache`, compare to the cached (warm) run, report the performance impact, and identify which transformation step consumes the most time during the uncached run.

### The `--no-cache` run

Measured through the same wrapper as Answer 1 (so the wall-clock value is verbatim command output, not prose):

```
$ bash /tmp/blitzy_timing/measure.sh single-nocache /tmp/blitzy_timing/a3_nocache.log -- client/state/data-layer/wpcom-http/test/index.js --no-cache
WALL_CLOCK_MS=2540
EXIT_STATUS=0
$ grep -E "Test Suites:|Tests:|Snapshots:|Time:" /tmp/blitzy_timing/a3_nocache.log
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.539 s
```

Measured wall-clock: **2540 ms** (the verbatim `WALL_CLOCK_MS=2540` above); Jest's own marker: `Time:        1.539 s`.

### Performance impact vs. the warm run

Using the warm baseline from Answer 1 (Jest `Time:` 0.896 s, wall 1540 ms):

- **`--no-cache` ÷ warm (Jest `Time:`)** = `1.539 s ÷ 0.896 s` = **≈ 1.72×**
- **`--no-cache` ÷ warm (wall-clock)** = `2540 ms ÷ 1540 ms` = **≈ 1.65×**

And critically, `--no-cache` ≈ the **cold** run:

- **`--no-cache` ÷ cold (Jest `Time:`)** = `1.539 s ÷ 1.567 s` = **≈ 0.98×**

The uncached time (1.539 s) is essentially the same as the cold-cache time (1.567 s), because both force a **full re-transform** of first-party source. `--no-cache` simply refuses to *read or write* the transform cache, so it behaves like a permanently cold cache. This ~1.7× penalty is directionally consistent with Jest's own documented guidance in the official Jest CLI documentation (Jest CLI Options, the `--cache` / `--no-cache` option, <https://jestjs.io/docs/cli#--cache>), which states that the cache "should only be disabled if you are experiencing caching related problems" and that "on average, disabling the cache makes Jest at least two times slower" (the measured ~1.7× here is a touch below their "at least two times" average because this suite's fixed per-run costs — Node/Jest/`nock` module load — dilute the transform-only savings on such a small file).

### The dominant transformation step: `babel-jest` transpiling first-party TS/JSX

The transform map that governs which step runs is defined in the shared preset, `packages/calypso-jest/jest-preset.js:L12-15`:

```
	testMatch: [ '<rootDir>/**/test/*.[jt]s?(x)', '!**/.eslintrc.*' ],
	transform: {
		'\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
		'\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
	},
```

There are exactly two transformers, and the dominant one is the first:

1. **`babel-jest` for `\.[jt]sx?$` (`packages/calypso-jest/jest-preset.js:L14`)** — this is where the time goes. It transpiles every first-party `.js/.jsx/.ts/.tsx` file the test graph loads. The `{ rootMode: 'upward' }` tuple makes Babel walk up to the repo-root `babel.config.js`, which composes the Calypso Babel preset:
   - `babel.config.js:L2` = `const babelConfig = require( '@automattic/calypso-babel-config' );`
   - `babel.config.js:L9` = `importSource: '@emotion/react',`

   The evidence that this is the costly step is the transform cache itself (Answer 2): after a run it holds **2095** `babel-jest`-produced code files and **1981** source maps — i.e., thousands of first-party modules had to be transpiled on a cold/uncached run, and reused on a warm run.

2. **The asset transform for images/styles (`packages/calypso-jest/jest-preset.js:L15` → `packages/calypso-jest/src/asset-transform.js`)** is trivial and therefore *not* the bottleneck. Its entire `process()` returns the file's basename as a string — `packages/calypso-jest/src/asset-transform.js:L5`:

   ```
   return { code: 'module.exports = ' + JSON.stringify( path.basename( filename ) ) + ';' };
   ```

   No parsing/transpilation occurs, so its cost is negligible.

**Why so much first-party source gets transformed.** The custom resolver deliberately resolves workspace packages to their **untranspiled source**, so that source is compiled at test time by `babel-jest` rather than loaded pre-built. See `packages/calypso-jest/src/module-resolver.js:L18`:

```
	mainFields: [ 'calypso:src', 'main' ],
```

Preferring `calypso:src` (source) over `main` (built output) means the workspace's own `.ts/.tsx/.js/.jsx` is fed through `babel-jest` — maximizing the transform work that the cache exists to amortize. Meanwhile `node_modules` is excluded (`test/client/jest.config.js:L14-16`), so third-party packages are *not* transpiled and do not contribute to this cost.

### Tooling versions behind the dominant step

- `packages/calypso-jest/package.json:L23` = `"@babel/core": "^7.26.10",`
- `packages/calypso-jest/package.json:L24` = `"babel-jest": "^29.7.0",`
- `packages/calypso-jest/package.json:L25` = `"enhanced-resolve": "^5.8.3",`
- `packages/calypso-jest/package.json:L26` = `"jest": "^29.7.0",`
- root `package.json:L290` = `"jest": "^29.7.0",`

### Summary for Answer 4

Disabling the cache costs **≈ 1.72×** vs. the warm run and reproduces the cold-run time (`--no-cache` ≈ cold), because both do a full re-transform. The single step consuming the most time is **`babel-jest` transpilation of first-party TypeScript/JavaScript/JSX** (`packages/calypso-jest/jest-preset.js:L14`, loading `babel.config.js` via `rootMode: 'upward'`) — corroborated by the ~2000 transformed modules in the cache, the triviality of the asset transform, the resolver's `calypso:src`-first policy, the `node_modules` exclusion, and Jest's documented "at least two times slower" guidance (Jest CLI Options, <https://jestjs.io/docs/cli#--cache>).


---

## Coverage pass — every sub-question answered

| # | Sub-question | Answer (with the value the question asks for) | Where |
|---|---|---|---|
| 1 | Run a `data-layer` test file twice; wall-clock each; first/second ratio? | Cold `Time:` **1.567 s** vs warm **0.896 s** → **≈ 1.75×** (single file); full suite **23.325 s** vs **12.875 s** → **≈ 1.81×**. Cause: warm run reuses cached `babel-jest` transforms. | Answer 1 |
| 2 | Where is the transform cache configured, which directory, which option? Cached file types? | Option **`cacheDirectory`** (`test/client/jest.config.js:L7`) → directory **`<repo>/.cache/jest`**. Types: `haste-map-*` module map; `jest-transform-cache-*/` transformed code **+ `.map` source maps** (2095 + 1981 here); `perf-cache-*` JSON of `[status, duration_ms]`. | Answer 2 |
| 3 | Which mocking library? Where configured? Timing effect? First-run driver? | Library **`nock`** (`package.json:L299` = `^13.5.6`); configured globally in `test/client/setup-test-framework.js` (`L9` `nock.disableNetConnect();`) with deprecated helper `client/test-helpers/use-nock/index.js`. Effect: fixed per-run module-load only; **not** a first-run overhead driver (excluded from transform via `transformIgnorePatterns`). | Answer 3 |
| 4 | `--no-cache` vs cached run — impact? Which transformation step dominates? | `--no-cache` **1.539 s** ≈ cold; **≈ 1.72×** vs warm. Dominant step: **`babel-jest` transpiling first-party TS/JSX** (`packages/calypso-jest/jest-preset.js:L14`). | Answer 4 |

### Caveats and grounding notes (per the "say so explicitly" rule)

- **Absolute ms are host-dependent.** The reported milliseconds reflect this host's load at run time; re-running will produce slightly different absolute numbers. The **ratios** are the stable answer and are what the question requests.
- **Node runtime deviation is intentional.** The repo requires `engines.node` `^v22.9.0` (`package.json:L57`; `.nvmrc:L1` = `22.9.0`). A Node-20 setup script would fail that gate, so a **Node v22.x** runtime (`v22.23.1`) was used. This is a deliberate, documented deviation, not an accident.
- **Two upstream-planning assumptions were corrected by direct verification:** (a) the `nock` mock line is **`L32`** (not L28–31) in `client/state/data-layer/wpcom-http/test/index.js`; (b) the `perf-cache-*` value is **`[status, duration_ms]`** (a `@jest/test-sequencer` record), not the `[transformTime, size]` described in the planning note (AAP §0.3.4). Both corrections were confirmed by reading the live files / installed sequencer source (`@jest/test-sequencer` `29.7.0`, `L92`–`L93`, `L268`–`L270`, `L280`, `L284`) and the live cache (first value `1` across all 92 entries; durations 63–5382 ms). The two framings reconcile on the AAP's own worked example — `[1,874]` = **`[SUCCESS, 874 ms]`** — so the correction refines the semantics rather than discarding the example. Both are called out where they appear (per the "be exact and grounded; say so explicitly" rule).
- **Read-only guarantee.** No tracked file was modified, created-over, or deleted. The only repository write is this document. `.cache/jest` is git-ignored (`.gitignore:L15`), so the transform-cache generation/clearing performed during measurement leaves the tracked tree unchanged (`git status --porcelain` empty).

### Command reference (exact forms used)

```
# invocation form (no top-level jest key; use the client config)
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js <path> --ci [--no-cache]

# true cold start before a cold measurement
rm -rf .cache/jest

# single representative file
client/state/data-layer/wpcom-http/test/index.js

# full data-layer module (path is relative to rootDir = client)
state/data-layer/
```

