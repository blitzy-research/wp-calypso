# Why some `wp-calypso` tests pass in isolation yet can fail in the full suite

> A runtime-observed investigation of the Jest test-runner environments and the monorepo's custom module-resolution layer.
>
> **Source branch:** `wp-calypso_be7e5cc64162`  •  **HEAD:** `be7e5cc641`  •  **Runner:** Jest `29.7.0`

---

## TL;DR

Every one of the six `test-*` commands starts from the **same default `testEnvironment: 'node'`** provided by the shared preset `@automattic/calypso-jest` [packages/calypso-jest/jest-preset.js:L11], but the _effective_ runtime a given test file sees is **decided per file, not per command**: a `/** @jest-environment jsdom */` docblock (present in **498** `client/` files) flips that one file into a browser-like `jsdom` world with `window`/`document`/`localStorage`, while its docblock-less neighbours run under plain Node. On top of that, the **same import specifier resolves to different on-disk files depending on which suite runs it** — `@automattic/calypso-config` loads the untranspiled TypeScript `packages/calypso-config/src/index.ts` under the _packages_ suite (via the custom `calypso:src`-first `enhanced-resolve` resolver) but is redirected to `client/server/config/index.js` under the _client_ suite (via that config's `moduleNameMapper`). Because environment selection, global provisioning (fetch/matchMedia/ResizeObserver come from _setup files_, not the environment), network-isolation state (`nock`), and module targets all vary **within a single invocation and across suites**, a file that quietly depends on one of those conditions can pass when run alone and fail when the full suite runs it in a different order or context. Every claim below is backed by the exact command, its complete unedited output, and a `file:line` reference.

---

## Environment & Methodology

### Legend — evidence tags

| Tag | Meaning |
| --- | --- |
| **[OBSERVED]** | Directly produced by running the adjacent command; the complete unedited output is embedded next to the claim. |
| **[INFERRED]** | Not directly printable (e.g. Jest's internal lifecycle ordering, taken from the docs); where feasible, confirmed by an observation probe — noted as _inferred + confirmed_. |
| **[NON-CANONICAL]** | A value produced by an instrumentation config, a bypassing interface, or a build-state-dependent artifact — not the repository's default canonical behavior. Flagged explicitly. |

### Reproducibility of the commands in this document

Every fenced command below is **self-contained and runnable as written** from the repository root. Commands that need a probe test **create it under a uniquely-named throwaway directory (`__blitzy_probe__` / `__blitzy_probe_int__`), run Jest, and delete the directory in the same block** — so nothing is left behind and each block reproduces on its own. The direct-resolver command runs inline through a `node - <<'NODE'` heredoc and writes no file at all. The two magnitude claims (the packages `jsdom`/`node` split and the `calypso-products` suite/test counts) are shown across **two consecutive runs** to demonstrate stability. Volatile fields — the wall-clock `Time:`/`estimated` line, the per-test `(N ms)` durations printed next to individual `✓` test names, and the order of `PASS` lines — naturally vary between runs and are timing artifacts only; the counts, resolved paths, `typeof` fields, and stack-frame `file:line` references the claims actually rely on do not vary. Where an embedded transcript shows a `(N ms)` duration or a `Time:` value, treat that number as illustrative of one run, not as a claim.

### Versions actually observed

**COMMAND**

```
echo "--- node --version ---"; node --version
echo "--- yarn --version ---"; node .yarn/releases/yarn-4.0.2.cjs --version
echo "--- jest --version ---"; CI=true node_modules/.bin/jest --version
echo "--- jsdom version ---"; node -e "console.log(require('jsdom/package.json').version)"
echo "--- jest-environment-jsdom version ---"; node -e "console.log(require('jest-environment-jsdom/package.json').version)"
echo "--- enhanced-resolve version ---"; node -e "console.log(require('enhanced-resolve/package.json').version)"
echo "--- symlink proof ---"; ls -la node_modules/@automattic/calypso-jest
```

**OBSERVED [OBSERVED]**

```
--- node --version ---
v22.23.1
--- yarn --version ---
4.0.2
--- jest --version ---
29.7.0
--- jsdom version ---
20.0.3
--- jest-environment-jsdom version ---
29.7.0
--- enhanced-resolve version ---
5.9.3
--- symlink proof ---
lrwxrwxrwx 1 root root 27 Jul  6 21:59 node_modules/@automattic/calypso-jest -> ../../packages/calypso-jest
```

- Node **`v22.23.1`** satisfies `engines.node: "^v22.9.0"` [package.json:L57]; `.nvmrc` pins `22.9.0` [.nvmrc:L1].
- Yarn **`4.0.2`** [package.json:L422].
- Jest **`29.7.0`**, jsdom **`20.0.3`**, jest-environment-jsdom **`29.7.0`**, enhanced-resolve **`5.9.3`**.
- Internal `@automattic/*` packages are **symlinked** into `node_modules` (`nodeLinker: node-modules`, not PnP), e.g. `node_modules/@automattic/calypso-jest -> ../../packages/calypso-jest`. This is what lets Jest resolve in-repo source.

### Canonical invocation

All suites are exercised through their real entry points — the exact `jest -c=test/<context>/jest.config.js` invocations declared in `package.json` `scripts` — run non-interactively with `CI=true`, `--runInBand`, and (for the client suite) `TZ=UTC`, mirroring `test-client` [package.json:L122]. A positional path filter (e.g. `__blitzy_probe__`) restricts a run to a probe **without modifying any config file** — this is a standard, canonical Jest CLI argument.

### Read-only approach (see Q6 for proof)

Establishing the environment (`yarn install`) and running Jest write **only to gitignored locations**: `node_modules/` [.gitignore:L17], the `packages/*/dist/` build outputs produced by each package's `prepare` script [.gitignore:L69], and Jest's cache under `/.cache/` [.gitignore:L15]. No tracked file is touched. Temporary probe files are created under uniquely-named throwaway directories and **deleted in the same command block**; the Q6 proof confirms the source tree is unchanged.

---

## Q1 — Test-command runtime environments: what each command actually uses, and how they differ

### The six commands (by name)

Each command invokes Jest with a distinct config [package.json:scripts]:

| Command | Definition | `file:line` |
| --- | --- | --- |
| `test-build-tools` | `jest -c=test/build-tools/jest.config.js` | [package.json:L121] |
| `test-client` | `TZ=UTC jest -c=test/client/jest.config.js` | [package.json:L122] |
| `test-integration` | `jest -c=test/integration/jest.config.js` | [package.json:L125] |
| `test-apps` | `jest -c=test/apps/jest.config.js` | [package.json:L127] |
| `test-packages` | `jest -c=test/packages/jest.config.js` | [package.json:L129] |
| `test-server` | `jest -c=test/server/jest.config.js` | [package.json:L131] |

The aggregate `test` script is `run-s -s test-client test-packages test-server test-build-tools` [package.json:L120] — note it **excludes** `test-apps` and `test-integration`.

### How the suites relate to the shared preset (the inheritance chain)

The root configs do **not** all import the preset the same way — this distinction matters for the rest of Q1:

- **client, server, build-tools** each `require` the preset and **spread** it: `const base = require( '@automattic/calypso-jest' ); module.exports = { ...base, ... }` [test/client/jest.config.js:L2,L5; test/server/jest.config.js:L2,L5; test/build-tools/jest.config.js:L2,L5]. The package `main` is the preset [packages/calypso-jest/package.json:L12].
- **packages, apps** root configs do **not** import the preset at all; they **delegate** to per-package / per-app child configs via `projects: [ '<rootDir>/packages/*/jest.config.js' ]` [test/packages/jest.config.js:L4] and `projects: [ '<rootDir>/apps/*/jest.config.js' ]` [test/apps/jest.config.js:L4]. The child preset files are what spread `...base` [test/packages/jest-preset.js:L9; test/apps/jest-preset.js:L5].
- **integration** is **standalone** — it imports no preset and instead sets `testEnvironment: 'node'` [test/integration/jest.config.js:L7] and `resolver` [test/integration/jest.config.js:L8] directly.

So the preset's default `testEnvironment: 'node'` [packages/calypso-jest/jest-preset.js:L11] reaches client/server/build-tools **by spread**, packages/apps **by child-config delegation**, and integration matches it **by an explicit setting** — not by one uniform `require` in every root config.

### Effective `testEnvironment` for the single-project suites

**COMMAND**

```
for ctx in client server build-tools integration; do
  echo "--- test/$ctx/jest.config.js ---"
  TZ=UTC CI=true node_modules/.bin/jest -c=test/$ctx/jest.config.js --showConfig 2>/dev/null | grep '"testEnvironment"' | head -1
done
```

**OBSERVED [OBSERVED]**

```
--- test/client/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/node_modules/jest-environment-node/build/index.js",
--- test/server/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/node_modules/jest-environment-node/build/index.js",
--- test/build-tools/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/node_modules/jest-environment-node/build/index.js",
--- test/integration/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/node_modules/jest-environment-node/build/index.js",
```

**Reasoning.** All four single-project suites resolve `testEnvironment` to `jest-environment-node`. Client, server, and build-tools inherit it from the preset's `testEnvironment: 'node'` via `...base` [test/client/jest.config.js:L5; test/server/jest.config.js:L5; test/build-tools/jest.config.js:L5]; integration sets `'node'` directly [test/integration/jest.config.js:L7]. So the config-level default for these commands is uniformly Node.

### The packages suite is multi-project — environments differ **within one command**

`test-packages` delegates to every `packages/*/jest.config.js` [test/packages/jest.config.js:L4]; each child config picks its own environment.

**COMMAND**

```
for run in 1 2; do
  echo "===== RUN $run ====="
  TZ=UTC CI=true node_modules/.bin/jest -c=test/packages/jest.config.js --showConfig 2>/dev/null \
    | grep '"testEnvironment"' \
    | sed -E 's#.*/node_modules/(jest-environment-[a-z]+)/build/index.js.*#\1/build/index.js#' \
    | sort | uniq -c
done
echo "--- total project count ---"
ls -1 packages/*/jest.config.js | wc -l
```

**OBSERVED [OBSERVED]** (two consecutive runs — stable)

```
===== RUN 1 =====
     22 jest-environment-jsdom/build/index.js
     36 jest-environment-node/build/index.js
===== RUN 2 =====
     22 jest-environment-jsdom/build/index.js
     36 jest-environment-node/build/index.js
--- total project count ---
58
```

**Reasoning.** Within a **single** `test-packages` invocation, the 58 package projects split **22 `jsdom` / 36 `node`** — most run under Node, but 22 opt into `jsdom` in their own `jest.config.js` (e.g. `packages/calypso-products/jest.config.js:L3` sets `testEnvironment: 'jsdom'`). The split is identical across both runs, and `22 + 36 = 58` equals the number of `packages/*/jest.config.js` files, so the observed split is internally consistent.

### The apps suite is jsdom across the board

**COMMAND**

```
TZ=UTC CI=true node_modules/.bin/jest -c=test/apps/jest.config.js --showConfig 2>/dev/null \
  | grep '"testEnvironment"' \
  | sed -E 's#.*/node_modules/(jest-environment-[a-z]+)/build/index.js.*#\1/build/index.js#' \
  | sort | uniq -c
echo "--- total apps project count ---"
ls -1 apps/*/jest.config.js | wc -l
```

**OBSERVED [OBSERVED]**

```
      3 jest-environment-jsdom/build/index.js
--- total apps project count ---
3
```

**Reasoning.** `test-apps` delegates to `apps/*/jest.config.js` [test/apps/jest.config.js:L4], and the apps preset sets `testEnvironment: 'jsdom'` [test/apps/jest-preset.js:L7]; all 3 app projects therefore run under jsdom.

### Per-suite runtime globals (canonical probe under each command)

A node-default probe (no docblock) was run under each suite; the parseable `BLITZY_PROBE_JSON` line reports `typeof` of each global. Summary of the decisive fields:

| Suite (command) | `userAgent` | `window` | `matchMedia` | `ResizeObserver` | `CSS` | `localStorage` |
| --- | --- | --- | --- | --- | --- | --- |
| `test-client` (node file) | `Node.js/22` | `undefined` | `function` | `function` | `object` | `undefined` |
| `test-client` (jsdom file) | `…jsdom/20.0.3` | `object` | `function` | `function` | `object` | `object` |
| `test-server` | `Node.js/22` | `undefined` | `undefined` | `undefined` | `undefined` | `undefined` |
| `test-build-tools` | `Node.js/22` | `undefined` | `undefined` | `undefined` | `object` | `undefined` |
| `test-integration` | `Node.js/22` | `undefined` | `undefined` | `undefined` | `undefined` | `undefined` |
| `test-packages` (default node) | `Node.js/22` | `undefined` | `function` | `function` | `object` | `undefined` |
| `test-apps` (default) | `…jsdom/20.0.3` | `object` | `function` | `function` | `object` | `object` |

**COMMAND (server)**

```
mkdir -p client/server/__blitzy_probe__/test
cat > client/server/__blitzy_probe__/test/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/server/jest.config.js __blitzy_probe__ --runInBand
rm -rf client/server/__blitzy_probe__
```

**OBSERVED [OBSERVED]**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/server/__blitzy_probe__/test/env-node.js
  ● Console

    console.log
      BLITZY_PROBE_JSON={"hasWindow":"undefined","hasDocument":"undefined","hasNavigator":"object","userAgent":"Node.js/22","hasSelf":"undefined","hasProcess":"object","hasSetImmediate":"function","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"undefined","hasResizeObserver":"undefined","hasCSS":"undefined","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"undefined"}

      at Object.log (__blitzy_probe__/test/env-node.js:19:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.573 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__/i.
```

**Reasoning.** Server is Node (`userAgent: "Node.js/22"`, `window: undefined`) and provides **none** of `matchMedia`/`ResizeObserver`/`CSS`, because `test/server/setup-test-framework.js` only wires `nock` [test/server/setup-test-framework.js:L4] and a `wpcom-proxy-request` mock [test/server/setup-test-framework.js:L21-L23] — no browser polyfills.

**COMMAND (build-tools)**

```
mkdir -p build-tools/__blitzy_probe__/test
cat > build-tools/__blitzy_probe__/test/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/build-tools/jest.config.js __blitzy_probe__ --runInBand
rm -rf build-tools/__blitzy_probe__
```

**OBSERVED [OBSERVED]**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS build-tools/__blitzy_probe__/test/env-node.js
  ● Console

    console.log
      BLITZY_PROBE_JSON={"hasWindow":"undefined","hasDocument":"undefined","hasNavigator":"object","userAgent":"Node.js/22","hasSelf":"undefined","hasProcess":"object","hasSetImmediate":"function","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"undefined","hasResizeObserver":"undefined","hasCSS":"object","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"undefined"}

      at Object.log (__blitzy_probe__/test/env-node.js:19:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.564 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__/i.
```

**Reasoning.** Build-tools is Node, but `CSS` is `object` here. Build-tools uses `...base` without overriding `setupFilesAfterEnv`, so it inherits the preset default `packages/calypso-jest/src/setup.js`, which sets `global.CSS = { supports: jest.fn() }` [packages/calypso-jest/src/setup.js:L3-L5]. It still lacks `matchMedia`/`ResizeObserver` (nothing adds them here).

**COMMAND (integration)**

```
mkdir -p client/__blitzy_probe_int__/integration
cat > client/__blitzy_probe_int__/integration/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/integration/jest.config.js __blitzy_probe_int__ --runInBand
rm -rf client/__blitzy_probe_int__
```

**OBSERVED [OBSERVED]** _(the two `jest-haste-map` blocks are build-state stderr noise — see Q4/Synthesis; kept unedited)_

```
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/packages/plans-grid-next/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/packages/plans-grid-next/dist/cjs/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/packages/plans-grid-next/dist/cjs/__mocks__/wpcom-proxy-request.js
    * <rootDir>/packages/plans-grid-next/dist/esm/__mocks__/wpcom-proxy-request.js

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/__blitzy_probe_int__/integration/env-node.js
  ● Console

    console.log
      BLITZY_PROBE_JSON={"hasWindow":"undefined","hasDocument":"undefined","hasNavigator":"object","userAgent":"Node.js/22","hasSelf":"undefined","hasProcess":"object","hasSetImmediate":"function","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"undefined","hasResizeObserver":"undefined","hasCSS":"undefined","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"undefined"}

      at Object.log (client/__blitzy_probe_int__/integration/env-node.js:19:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.546 s, estimated 1 s
Ran all test suites matching /__blitzy_probe_int__/i.
```

**Reasoning.** Integration is Node and provides **none** of the browser-ish globals (including `CSS: undefined`), because its config is standalone — it does **not** spread `...base` and declares **no** `setupFilesAfterEnv` [test/integration/jest.config.js:L1-L16], so neither the preset's `src/setup.js` nor any polyfill file runs.

**COMMAND (packages — default node project)**

```
mkdir -p packages/__blitzy_probe__/test
printf "module.exports = { preset: '../../test/packages/jest-preset.js' };\n" > packages/__blitzy_probe__/jest.config.js
cat > packages/__blitzy_probe__/test/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/packages/jest.config.js __blitzy_probe__ --runInBand
rm -rf packages/__blitzy_probe__
```

**OBSERVED [OBSERVED]**

```
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/esm/__mocks__/wpcom-proxy-request.js

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
  console.log
    BLITZY_PROBE_JSON={"hasWindow":"undefined","hasDocument":"undefined","hasNavigator":"object","userAgent":"Node.js/22","hasSelf":"undefined","hasProcess":"object","hasSetImmediate":"function","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"function","hasResizeObserver":"function","hasCSS":"object","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"undefined"}

      at Object.log (test/env-node.js:19:10)

PASS packages/__blitzy_probe__/test/env-node.js
  ✓ blitzy env probe (14 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.741 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__/i.
```

**Reasoning.** The default packages project is Node, yet it has `matchMedia`/`ResizeObserver` (`function`) and `CSS` (`object`). The packages preset replaces `setupFilesAfterEnv` with `test/packages/setup.js` [test/packages/jest-preset.js:L14], which sets `ResizeObserver` [test/packages/setup.js:L5] and `matchMedia` [test/packages/setup.js:L7-L16]. `CSS` becomes `object` as a **require-time side effect** of `import '@testing-library/jest-dom'` [test/packages/setup.js:L1] (proven below), not from the environment.

**COMMAND (apps — default project)**

```
mkdir -p apps/__blitzy_probe__/test
printf "module.exports = { preset: '../../test/apps/jest-preset.js' };\n" > apps/__blitzy_probe__/jest.config.js
cat > apps/__blitzy_probe__/test/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/apps/jest.config.js __blitzy_probe__ --runInBand
rm -rf apps/__blitzy_probe__
```

**OBSERVED [OBSERVED]**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
  console.log
    BLITZY_PROBE_JSON={"hasWindow":"object","hasDocument":"object","hasNavigator":"object","userAgent":"Mozilla/5.0 (linux) AppleWebKit/537.36 (KHTML, like Gecko) jsdom/20.0.3","hasSelf":"object","hasProcess":"object","hasSetImmediate":"undefined","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"function","hasResizeObserver":"function","hasCSS":"object","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"object"}

      at Object.log (test/env-node.js:19:10)

PASS apps/__blitzy_probe__/test/env-node.js
  ✓ blitzy env probe (11 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.983 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__/i.
```

**Reasoning.** Apps is jsdom (`userAgent: "…jsdom/20.0.3"`, `window: object`, `localStorage: object`) because the apps preset forces `testEnvironment: 'jsdom'` [test/apps/jest-preset.js:L7] and reuses the client setup file [test/apps/jest-preset.js:L13].

### `CSS`/`matchMedia`/`ResizeObserver` come from **setup files**, not the environment (grounding sub-probe)

**COMMAND**

```
node -e "console.log('typeof CSS =', typeof CSS)"
node -e "try { require('@testing-library/jest-dom'); } catch(e){ console.log('require threw (expected, needs jest expect):', e.message.split('\n')[0]); } console.log('after require typeof CSS =', typeof CSS)"
```

**OBSERVED [OBSERVED]**

```
typeof CSS = undefined
require threw (expected, needs jest expect): expect is not defined
after require typeof CSS = object
```

**Reasoning [OBSERVED].** Plain Node 22 has **no** global `CSS`; requiring `@testing-library/jest-dom` defines `global.CSS` as a side effect. This confirms the causal split behind the Q1 table: `window`/`document`/`self`/`localStorage` track the **environment** (jsdom vs node), whereas `CSS`/`matchMedia`/`ResizeObserver`/`fetch` track the **setup files** — and those setup files run for **every** file in a suite regardless of that file's own `@jest-environment`.

**Answer to Q1.** The six commands all start from `testEnvironment: 'node'` [packages/calypso-jest/jest-preset.js:L11]. They differ in three observable ways: (1) whether files opt into `jsdom` — never (server/build-tools/integration), per-file (client), or wholesale (apps `jsdom`; packages a `22/36` mix within one run); (2) which setup file augments globals — client `test/client/setup-test-framework.js`, packages `test/packages/setup.js`, build-tools the preset `src/setup.js`, server the nock-only file, integration none; and (3) module-resolution overrides (Q4). The environment fingerprint is `navigator.userAgent`: `"Node.js/22"` for `jest-environment-node`, `"…jsdom/20.0.3"` for `jest-environment-jsdom`.

---

## Q2 — Global-availability contrast: a capability that exists in one context but not another

The sharpest contrast happens **under the identical `test-client` command**: two files differing only by a one-line docblock see different globals.

**COMMAND**

```
mkdir -p client/__blitzy_probe__/test
cat > client/__blitzy_probe__/test/env-node.js <<'PROBE'
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
PROBE
{ echo '/** @jest-environment jsdom */'; cat client/__blitzy_probe__/test/env-node.js; } > client/__blitzy_probe__/test/env-jsdom.js
TZ=UTC CI=true node_modules/.bin/jest -c=test/client/jest.config.js __blitzy_probe__ --runInBand
rm -rf client/__blitzy_probe__
```

**OBSERVED [OBSERVED]** (complete, unedited)

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/__blitzy_probe__/test/env-jsdom.js
  ● Console

    console.log
      BLITZY_PROBE_JSON={"hasWindow":"object","hasDocument":"object","hasNavigator":"object","userAgent":"Mozilla/5.0 (linux) AppleWebKit/537.36 (KHTML, like Gecko) jsdom/20.0.3","hasSelf":"object","hasProcess":"object","hasSetImmediate":"undefined","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"function","hasResizeObserver":"function","hasCSS":"object","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"object"}

      at Object.log (__blitzy_probe__/test/env-jsdom.js:20:10)

PASS client/__blitzy_probe__/test/env-node.js
  ● Console

    console.log
      BLITZY_PROBE_JSON={"hasWindow":"undefined","hasDocument":"undefined","hasNavigator":"object","userAgent":"Node.js/22","hasSelf":"undefined","hasProcess":"object","hasSetImmediate":"function","hasTextEncoder":"function","hasFetch":"function","hasMatchMedia":"function","hasResizeObserver":"function","hasCSS":"object","hasReadableStream":"function","hasStructuredClone":"function","hasLocalStorage":"undefined"}

      at Object.log (__blitzy_probe__/test/env-node.js:19:10)


Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.313 s
Ran all test suites matching /__blitzy_probe__/i.
```

### Concrete capability difference

| Global | jsdom-docblock file | node-default file | Provided by |
| --- | --- | --- | --- |
| `window` | `object` | **`undefined`** | the environment (jsdom) |
| `document` | `object` | **`undefined`** | the environment (jsdom) |
| `self` | `object` | **`undefined`** | the environment (jsdom) |
| `localStorage` | `object` | **`undefined`** | the environment (jsdom) |
| `setImmediate` (reverse case) | **`undefined`** | `function` | the environment (Node) |

**Answer to Q2 [OBSERVED].** Under the _same_ `test-client` command, **`window`, `document`, `self`, and `localStorage` exist only in the file carrying `/** @jest-environment jsdom */`**; they are `undefined` in the docblock-less (node-default) file. The config-level `testEnvironment` is `jest-environment-node` (Q1); the per-file docblock is what overrides it. The **reverse / edge case** is equally real: `setImmediate` is `function` under Node but `undefined` under jsdom — a Node-only global that jsdom does not provide. (Note that `matchMedia`, `ResizeObserver`, `CSS`, and `fetch` are `function` / `object` in *both* files, because the client `setupFilesAfterEnv` runs for every file irrespective of its environment — see Q5.)

### How widespread is the per-file opt-in?

**COMMAND**

```
echo "--- files containing docblock ---"
grep -rl '@jest-environment jsdom' client/ --include='*.js' --include='*.jsx' --include='*.ts' --include='*.tsx' | wc -l
echo "--- value histogram ---"
grep -rHo '@jest-environment [a-z]*' client/ --include='*.js' --include='*.jsx' --include='*.ts' --include='*.tsx' | sed -E 's#.*:(@jest-environment [a-z]*)#\1#' | sort | uniq -c
echo "--- RUN 2 (stability) ---"
grep -rl '@jest-environment jsdom' client/ --include='*.js' --include='*.jsx' --include='*.ts' --include='*.tsx' | wc -l
```

**OBSERVED [OBSERVED]** (with a second run for stability)

```
--- files containing docblock ---
498
--- value histogram ---
    498 @jest-environment jsdom
--- RUN 2 (stability) ---
498
```

**Reasoning [OBSERVED].** **498** files under `client/` carry the docblock, each exactly once, and **all 498 use the value `jsdom`** (no other value appears); the count is identical on the second run. So under the single `test-client` command, 498 files run in a browser-like world and the remainder run in Node — the per-file opt-in is the crux of Q2 and a pillar of the Synthesis. (The environment default is `node` [packages/calypso-jest/jest-preset.js:L11]; there is no config-level `jsdom` for the client suite — it is purely per file.)

---

## Q3 — Internal dependency resolution: which file actually loads, and does it differ by how you run the tests?

### The named example

`@automattic/calypso-products` declares an internal dependency on `@automattic/calypso-config` [packages/calypso-products/package.json:L39], under `dependencies` at [packages/calypso-products/package.json:L38]. Its source imports it directly, e.g. `import { isEnabled } from '@automattic/calypso-config'`. That package also transitively pulls in `@automattic/create-calypso-config`.

### Running its tests through the canonical entry point

**COMMAND**

```
CI=true node_modules/.bin/jest -c=test/packages/jest.config.js packages/calypso-products --runInBand
```

**OBSERVED [OBSERVED]** (complete, unedited; the `jest-haste-map` blocks are build-state stderr noise — see Q4/Synthesis. `PASS`-line order and `Time:` vary between runs; the suite/test counts do not — see the stability transcript below)

```
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/esm/__mocks__/wpcom-proxy-request.js

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS packages/calypso-products/test/plan-lookups.js
PASS packages/calypso-products/test/product-values.js
PASS packages/calypso-products/test/get-difm-tiered-price-details.ts
PASS packages/calypso-products/test/get-jetpack-item-term-variants.js
PASS packages/calypso-products/test/choose-default-customer-type.js
PASS packages/calypso-products/test/plan-other.js
PASS packages/calypso-products/test/plan-levels-match.js
PASS packages/calypso-products/test/products-list.js
PASS packages/calypso-products/test/get-feature-difference.ts
PASS packages/calypso-products/test/is-plan.js
PASS packages/calypso-products/test/plans-link.js
PASS packages/calypso-products/test/is-superseding-jetpack-item.js
PASS packages/calypso-products/test/get-popular-plan-spec.js
PASS packages/calypso-products/test/has-marketplace-product.js
PASS packages/calypso-products/test/get-interval-type-for-term.js
PASS packages/calypso-products/test/is-jetpack-legacy-item.js
PASS packages/calypso-products/test/is-jetpack-purchasable-item.js

Test Suites: 17 passed, 17 total
Tests:       243 passed, 243 total
Snapshots:   0 total
Time:        7.423 s, estimated 12 s
Ran all test suites matching /packages\/calypso-products/i.
```

**Stability of the magnitude claim (two consecutive runs)**

**COMMAND**

```
for run in 1 2; do
  echo "===== RUN $run ====="
  CI=true node_modules/.bin/jest -c=test/packages/jest.config.js packages/calypso-products --runInBand 2>&1 \
    | grep -E '^(Test Suites|Tests|Snapshots):'
done
```

**OBSERVED [OBSERVED]**

```
===== RUN 1 =====
Test Suites: 17 passed, 17 total
Tests:       243 passed, 243 total
Snapshots:   0 total
===== RUN 2 =====
Test Suites: 17 passed, 17 total
Tests:       243 passed, 243 total
Snapshots:   0 total
```

**Reasoning.** All **17** test files under `packages/calypso-products/test/` pass (**243** tests), and the counts are identical across both runs. Note the suite runs under `jsdom` — `packages/calypso-products/jest.config.js:L3` sets `testEnvironment: 'jsdom'` — extending `test/packages/jest-preset.js` [packages/calypso-products/jest.config.js:L2], which itself spreads the base preset [test/packages/jest-preset.js:L9].

### Which file does the internal import actually load?

An in-suite `require.resolve` probe (Jest's own module resolver, its canonical path) was run under the packages suite:

**COMMAND**

```
mkdir -p packages/__blitzy_probe__/test
printf "module.exports = { preset: '../../test/packages/jest-preset.js' };\n" > packages/__blitzy_probe__/jest.config.js
cat > packages/__blitzy_probe__/test/resolve.js <<'PROBE'
test( 'blitzy resolve probe', () => {
	console.log( 'BLITZY_RESOLVE_CONFIG=' + require.resolve( '@automattic/calypso-config' ) );
	try {
		console.log( 'BLITZY_RESOLVE_CCC=' + require.resolve( '@automattic/create-calypso-config' ) );
	} catch ( e ) {
		console.log( 'BLITZY_RESOLVE_CCC_ERR=' + e.message.split( '\n' )[ 0 ] );
	}
	expect( true ).toBe( true );
} );
PROBE
CI=true node_modules/.bin/jest -c=test/packages/jest.config.js __blitzy_probe__/test/resolve --runInBand
rm -rf packages/__blitzy_probe__
```

**OBSERVED [OBSERVED]** (resolve lines; the full run also passes the probe assertion)

```
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/esm/__mocks__/wpcom-proxy-request.js

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
  console.log
    BLITZY_RESOLVE_CONFIG=/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/packages/calypso-config/src/index.ts

      at Object.log (test/resolve.js:2:10)

  console.log
    BLITZY_RESOLVE_CCC=/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/packages/create-calypso-config/src/index.ts

      at Object.log (test/resolve.js:4:11)

PASS packages/__blitzy_probe__/test/resolve.js
  ✓ blitzy resolve probe (15 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.744 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__\/test\/resolve/i.
```

**Reasoning [OBSERVED].** Inside the packages suite, `@automattic/calypso-config` loads the **untranspiled TypeScript source `packages/calypso-config/src/index.ts`** — not the built `main` `dist/cjs/index.js` [packages/calypso-config/package.json:L9]. That happens because the package declares `"calypso:src": "src/index.ts"` [packages/calypso-config/package.json:L11] and the custom resolver prefers `calypso:src` over `main` (Q4). `babel-jest` with `rootMode: 'upward'` [packages/calypso-jest/jest-preset.js:L14] transpiles that TS on the fly — which is exactly why the 243 tests importing from a `.ts` file pass with **no build step**. Transitively, `@automattic/create-calypso-config` also resolves to its `src/index.ts` [packages/create-calypso-config/package.json:L11].

### Does it differ based on how you execute the tests? — **Yes**

The **same specifier** resolved inside the **client** suite instead:

**COMMAND**

```
mkdir -p client/__blitzy_probe__/test
cat > client/__blitzy_probe__/test/resolve.js <<'PROBE'
test( 'blitzy resolve probe', () => {
	console.log( 'BLITZY_RESOLVE_CONFIG=' + require.resolve( '@automattic/calypso-config' ) );
	try {
		console.log( 'BLITZY_RESOLVE_CCC=' + require.resolve( '@automattic/create-calypso-config' ) );
	} catch ( e ) {
		console.log( 'BLITZY_RESOLVE_CCC_ERR=' + e.message.split( '\n' )[ 0 ] );
	}
	expect( true ).toBe( true );
} );
PROBE
TZ=UTC CI=true node_modules/.bin/jest -c=test/client/jest.config.js __blitzy_probe__/test/resolve --runInBand
rm -rf client/__blitzy_probe__
```

**OBSERVED [OBSERVED]**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/__blitzy_probe__/test/resolve.js
  ● Console

    console.log
      BLITZY_RESOLVE_CONFIG=/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/client/server/config/index.js

      at Object.log (__blitzy_probe__/test/resolve.js:2:10)

    console.log
      BLITZY_RESOLVE_CCC=/tmp/blitzy/wp-calypso/blitzy-30b27a23-6c85-47da-9f81-5acf9717cb20_bb9c37/packages/create-calypso-config/src/index.ts

      at Object.log (__blitzy_probe__/test/resolve.js:4:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.766 s, estimated 1 s
Ran all test suites matching /__blitzy_probe__\/test\/resolve/i.
```

**Answer to Q3 [OBSERVED].** The identical import `@automattic/calypso-config` loads a **different file** depending on the suite: `packages/calypso-config/src/index.ts` under `test-packages` (custom resolver → `calypso:src`), but `client/server/config/index.js` under `test-client` — because the client config remaps that specifier via `moduleNameMapper` [test/client/jest.config.js:L11] (detailed in Q4). Notably, the **unmapped** transitive dependency `@automattic/create-calypso-config` resolves to `packages/create-calypso-config/src/index.ts` in **both** suites, isolating the mapping as the cause of the divergence.

---

## Q4 — Where imports are overridden, and where they resolve at runtime

Two mechanisms decide where a specifier lands: a per-config **`moduleNameMapper`** (highest precedence) and, when nothing maps, the shared **custom resolver**.

### The custom resolver

`packages/calypso-jest/src/module-resolver.js` builds an `enhanced-resolve` resolver [packages/calypso-jest/src/module-resolver.js:L16] with `mainFields: [ 'calypso:src', 'main' ]` [packages/calypso-jest/src/module-resolver.js:L18] and `conditionNames: [ 'calypso:src', 'node', 'require' ]` [packages/calypso-jest/src/module-resolver.js:L19] — so `calypso:src` wins over `main`. The exported function [packages/calypso-jest/src/module-resolver.js:L22-L24] resolves `options.basedir` + `request`. The preset wires it as the Jest `resolver` [packages/calypso-jest/jest-preset.js:L9], and the integration config points at it explicitly [test/integration/jest.config.js:L8].

**COMMAND (exercise the resolver directly, contrasted with Node's default)**

```
REPO="$(pwd)"
node - "$REPO" <<'NODE'
const REPO = process.argv[2];
const resolver = require( REPO + '/node_modules/@automattic/calypso-jest/src/module-resolver.js' );
const reqs = [ '@automattic/calypso-config', '@automattic/create-calypso-config' ];
const rel = ( p ) => p.replace( REPO + '/', '' );
console.log( '=== CUSTOM RESOLVER (mainFields [calypso:src, main]) from basedir packages/calypso-products ===' );
for ( const r of reqs )
	console.log( r + '  ->  ' + rel( resolver( r, { basedir: REPO + '/packages/calypso-products' } ) ) );
console.log( '\n=== NODE DEFAULT require.resolve (main-first) from REPO root ===' );
for ( const r of reqs ) {
	try { console.log( r + '  ->  ' + rel( require.resolve( r, { paths: [ REPO ] } ) ) ); }
	catch ( e ) { console.log( r + '  ->  ERROR ' + e.message.split( '\n' )[ 0 ] ); }
}
NODE
```

**OBSERVED [OBSERVED]** (complete, unedited)

```
=== CUSTOM RESOLVER (mainFields [calypso:src, main]) from basedir packages/calypso-products ===
@automattic/calypso-config  ->  packages/calypso-config/src/index.ts
@automattic/create-calypso-config  ->  packages/create-calypso-config/src/index.ts

=== NODE DEFAULT require.resolve (main-first) from REPO root ===
@automattic/calypso-config  ->  packages/calypso-config/dist/cjs/index.js
@automattic/create-calypso-config  ->  packages/create-calypso-config/dist/cjs/index.js
```

**Reasoning.** The custom resolver returns the `calypso:src` TypeScript (`…/src/index.ts`) for both packages, while Node's `require.resolve` (which honours `main` first) returns the built `…/dist/cjs/index.js`. **Build-state caveat [NON-CANONICAL]:** the Node-default result is `dist/cjs/index.js` **only because `packages/*/dist` is present** (built by `yarn install`'s `prepare`). In a fresh, unbuilt clone `main` points at a file the resolver's own comment says "usually _does not_ exist" [packages/calypso-jest/src/module-resolver.js:L11-L12], so plain Node resolution would error there — the reason the monorepo needs the `calypso:src`-first resolver at all.

### `moduleNameMapper` overrides the resolver — per-suite matrix

For the specifier `^@automattic/calypso-config$`:

| Suite | `moduleNameMapper` target | `file:line` | Resolves to |
| --- | --- | --- | --- |
| client | `<rootDir>/server/config/index.js` (rootDir = `../../client` [test/client/jest.config.js:L6]) | [test/client/jest.config.js:L11] | `client/server/config/index.js` **[OBSERVED, in-suite]** |
| integration | `<rootDir>/client/server/config/index.js` (rootDir = `../..` [test/integration/jest.config.js:L6]) | [test/integration/jest.config.js:L3] | `client/server/config/index.js` |
| server | `calypso/server/config` (+ subpath `$1`) | [test/server/jest.config.js:L10-L11] | mapped alias under `client/server` |
| packages | _(no calypso-config mapping; only react-markdown)_ | [test/packages/jest.config.js:L6] | `packages/calypso-config/src/index.ts` **[OBSERVED — resolver applies]** |
| apps | _(no calypso-config mapping)_ | [test/apps/jest.config.js:L4] | `calypso:src` via resolver |

The two **[OBSERVED, in-suite]** rows are the `require.resolve` outputs already shown in Q3: client → `client/server/config/index.js`; packages → `packages/calypso-config/src/index.ts`. This proves Jest's in-test `require.resolve` honours **both** `moduleNameMapper` (client) **and** the custom resolver (packages).

**Answer to Q4 [OBSERVED].** The override lives in each config's `moduleNameMapper` (client [test/client/jest.config.js:L11], integration [test/integration/jest.config.js:L3], server [test/server/jest.config.js:L10-L11]) and, absent a mapping, in the shared custom resolver [packages/calypso-jest/src/module-resolver.js:L18]. The **same** specifier therefore resolves to one of three places depending on execution context: **(1)** a mapped file (`client/server/config/index.js`) under client/server/integration; **(2)** an in-repo `src` TypeScript file (`packages/calypso-config/src/index.ts`) under packages/apps; or **(3)** a built `dist/cjs` file when resolved by plain Node outside Jest. **Precedence: `moduleNameMapper` > custom `resolver`; and within the resolver, `calypso:src` > `main`.**

---

## Q5 — Initialization order & who provides the browser-like APIs

### The lifecycle

**[INFERRED from Jest docs, CONFIRMED by the probe below]** For each test file Jest runs, in order:

1. **Environment creation** — `jest-environment-node` or `jest-environment-jsdom` is instantiated. **`jest-environment-jsdom` is the provider of `window`/`document`/DOM**, created _here_, before any setup file runs.
2. **`setupFiles`** — run once per file, in the environment, **before** the test framework is installed.
3. **Test-framework install** — `expect`, `afterAll`, the `it`/`test` globals, etc. become available.
4. **`setupFilesAfterEnv`** — run after the framework, before the test body.
5. **Test file** executes.

### Client wiring (what runs, and in what order)

- The preset default `setupFilesAfterEnv` is `packages/calypso-jest/src/setup.js` [packages/calypso-jest/jest-preset.js:L10] (it mocks `CSS.supports` [packages/calypso-jest/src/setup.js:L3-L5]).
- The **client** config adds `setupFiles: [ 'jest-canvas-mock' ]` [test/client/jest.config.js:L20] and **replaces** `setupFilesAfterEnv` with `<rootDir>/../test/client/setup-test-framework.js` [test/client/jest.config.js:L21].
- That client setup file augments what jsdom lacks — `@testing-library/jest-dom` [test/client/setup-test-framework.js:L1], `nock.disableNetConnect()` [test/client/setup-test-framework.js:L9], the `beforeAll`/`afterAll` nock lifecycle [test/client/setup-test-framework.js:L11-L16,L18-L22], `TextEncoder`/`TextDecoder` [test/client/setup-test-framework.js:L25-L26], `CSS.supports` [test/client/setup-test-framework.js:L30-L32], `ResizeObserver` [test/client/setup-test-framework.js:L34], `fetch` [test/client/setup-test-framework.js:L36-L40], the `wpcom-proxy-request` mock [test/client/setup-test-framework.js:L44-L49], `crypto.randomUUID` [test/client/setup-test-framework.js:L52], `matchMedia` [test/client/setup-test-framework.js:L54-L63], `ReadableStream`/`TransformStream`/`Worker` [test/client/setup-test-framework.js:L66-L68], `structuredClone` [test/client/setup-test-framework.js:L71-L73], and `crypto.subtle` [test/client/setup-test-framework.js:L76-L79].

### Confirming the order by observation

A **[NON-CANONICAL]** instrumentation config wraps the _real_ client setup file with logging hooks: one in `setupFiles`, one at the start of `setupFilesAfterEnv` (EARLY), the real client setup, one after it (LATE), and one in the test. It forces `testEnvironment: 'jsdom'` and runs only its own probe.

**COMMAND**

```
mkdir -p client/__blitzy_probe__/order
cat > client/__blitzy_probe__/order/logger.js <<'LOGGER'
module.exports = function logPhase( phase ) {
	const pad = ( s, n ) => ( s + ' '.repeat( n ) ).slice( 0, n );
	console.log(
		'BLITZY_ORDER phase=' + pad( phase, 45 ) +
			' window=' + typeof window +
			' document=' + typeof document +
			' matchMedia=' + typeof matchMedia +
			' fetch=' + typeof fetch +
			' expect=' + typeof expect +
			' afterAll=' + typeof afterAll +
			' jest=' + typeof jest
	);
};
LOGGER
printf "require( './logger' )( 'setupFiles(before-framework)' );\n" > client/__blitzy_probe__/order/log-setupfiles.js
printf "require( './logger' )( 'setupFilesAfterEnv(EARLY)' );\n" > client/__blitzy_probe__/order/log-early.js
printf "require( './logger' )( 'setupFilesAfterEnv(LATE-after-client-setup)' );\n" > client/__blitzy_probe__/order/log-late.js
cat > client/__blitzy_probe__/order/order.test.js <<'ORDERTEST'
test( 'blitzy order probe', () => {
	require( './logger' )( 'test' );
	expect( true ).toBe( true );
} );
ORDERTEST
cat > client/__blitzy_probe__/order.config.js <<'ORDERCFG'
const path = require( 'path' );
const base = require( '@automattic/calypso-jest' );
const REPO = path.join( __dirname, '../..' );
module.exports = {
	...base,
	rootDir: __dirname,
	testEnvironment: 'jsdom',
	testEnvironmentOptions: { url: 'https://example.com' },
	testMatch: [ '<rootDir>/order/order.test.js' ],
	cacheDirectory: path.join( REPO, '.cache/jest' ),
	setupFiles: [ require.resolve( './order/log-setupfiles.js' ) ],
	setupFilesAfterEnv: [
		require.resolve( './order/log-early.js' ),
		path.join( REPO, 'test/client/setup-test-framework.js' ),
		require.resolve( './order/log-late.js' ),
	],
};
ORDERCFG
TZ=UTC CI=true node_modules/.bin/jest -c=client/__blitzy_probe__/order.config.js --runInBand
rm -rf client/__blitzy_probe__
```

**OBSERVED [OBSERVED]** (complete, unedited)

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/__blitzy_probe__/order/order.test.js
  ● Console

    console.log
      BLITZY_ORDER phase=setupFiles(before-framework)                  window=object document=object matchMedia=undefined fetch=undefined expect=undefined afterAll=undefined jest=object

      at logPhase (order/logger.js:5:11)

    console.log
      BLITZY_ORDER phase=setupFilesAfterEnv(EARLY)                     window=object document=object matchMedia=undefined fetch=undefined expect=function afterAll=function jest=object

      at log (order/logger.js:3:10)

    console.log
      BLITZY_ORDER phase=setupFilesAfterEnv(LATE-after-client-setup)   window=object document=object matchMedia=function fetch=function expect=function afterAll=function jest=object

      at log (order/logger.js:3:10)

    console.log
      BLITZY_ORDER phase=test                                          window=object document=object matchMedia=function fetch=function expect=function afterAll=function jest=object

      at log (order/logger.js:3:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.868 s, estimated 1 s
Ran all test suites.
```

### What the four lines prove

| Signal | `setupFiles` | `setupFilesAfterEnv` EARLY | `setupFilesAfterEnv` LATE | test | Conclusion |
| --- | --- | --- | --- | --- | --- |
| `window` / `document` | `object` | `object` | `object` | `object` | jsdom provisions them **at env creation**, before any setup file. |
| `expect` / `afterAll` | **`undefined`** | **`function`** | `function` | `function` | the framework installs **between** `setupFiles` and `setupFilesAfterEnv`. |
| `jest` (object) | `object` | `object` | `object` | `object` | `jest` (for `jest.mock`) is available already in `setupFiles`. |
| `matchMedia` / `fetch` | `undefined` | **`undefined`** | **`function`** | `function` | provided by `test/client/setup-test-framework.js`, which runs at the `setupFilesAfterEnv` phase — not by jsdom. |

**Answer to Q5.** What loads first is the **test environment** (`jest-environment-jsdom` for a jsdom file), created before any setup code; it is the source of `window`/`document` — observed as `object` even in the earliest `setupFiles` phase. The framework (`expect`, `afterAll`) installs _after_ `setupFiles` and _before_ `setupFilesAfterEnv` — observed as the `undefined -> function` transition. The **browser-like augmentations** the client tests rely on (`matchMedia`, `fetch`, `ResizeObserver`, `crypto.*`, …) are **not** from jsdom; they are added by `test/client/setup-test-framework.js` at the `setupFilesAfterEnv` phase — observed as the `undefined -> function` transition for `matchMedia`/`fetch` only at the LATE hook. The ordering itself is **[INFERRED]** from Jest's documentation and **[CONFIRMED]** by this probe; the probe **config** is **[NON-CANONICAL]** (instrumentation), but it invokes the **real** `test/client/setup-test-framework.js`.

---

## Q6 — Read-only proof

Every probe directory is created and deleted inside its own command block (see each COMMAND above), and the direct-resolver script runs inline via a `node - <<'NODE'` heredoc with no on-disk file. The following confirms the source tree is unchanged.

**COMMAND**

```
echo "--- source tree status (everything EXCEPT the answer doc) ---"
git status --porcelain -- . ':(exclude)blitzy/documentation'
echo "(empty above = no source/reference file modified)"
echo "--- leftover probe artifacts (excluding node_modules) ---"
find . -path ./node_modules -prune -o -name '*__blitzy_probe*' -print
echo "(empty above = none remain)"
echo "--- writable outputs are gitignored ---"
git check-ignore .cache/jest packages/calypso-config/dist node_modules
```

**OBSERVED [OBSERVED]**

```
--- source tree status (everything EXCEPT the answer doc) ---
(empty above = no source/reference file modified)
--- leftover probe artifacts (excluding node_modules) ---
(empty above = none remain)
--- writable outputs are gitignored ---
.cache/jest
packages/calypso-config/dist
node_modules
```

**Answer to Q6 [OBSERVED].** Scoping `git status --porcelain` to everything **except** the answer document prints **zero lines** — no source or reference file was modified. No `__blitzy_probe*` artifact remains anywhere outside `node_modules`. The reason running Jest never dirties the tree is that its writable outputs are gitignored: `/.cache/` [.gitignore:L15] and `packages/*/dist/` [.gitignore:L69] (both confirmed by `git check-ignore`), plus `node_modules/` [.gitignore:L17]. The only tracked addition from this whole task is this answer document, `blitzy/documentation/wp-calypso_be7e5cc64162.md` (there is no `blitzy`/`documentation` entry in `.gitignore`).

---

## Synthesis — why some tests pass in isolation but can fail in the full suite

The isolated-vs-full-suite discrepancy is not one bug; it is the interaction of several **observed** ingredients. The specific failure that materialises depends on **file/test ordering**, which differs between a single-file run and the whole suite (that causal step is labeled _inferred_; the ingredients are all _observed_).

1. **Per-file environment variability under one command [OBSERVED].** `test-client` runs **498** jsdom files (Q2) alongside many node files in the _same_ invocation; `test-packages` runs a **22 jsdom / 36 node** project mix in one run (Q1). A file that implicitly assumes `window`/`document`/`localStorage` passes when run alone _if_ it carries the `@jest-environment jsdom` docblock; a node-env sibling (or a file that forgot the docblock) sees those as `undefined` (Q2). Because Jest's default global cleanup and worker/file scheduling differ between "one file" and "the full suite," the same file can be scheduled and initialised differently in the two cases.

2. **Resolution divergence [OBSERVED].** The same specifier `@automattic/calypso-config` loads `packages/calypso-config/src/index.ts` under packages but `client/server/config/index.js` under client (Q3/Q4), and a built `dist/cjs` file under plain Node. A test that (directly or transitively) relies on the behavior of one target can break under a suite that maps the specifier to a different implementation.

3. **Shared-global / setup provisioning [OBSERVED wiring; INFERRED bleed mechanism].** The browser-ish APIs are **`jest.fn()` mocks** installed by `setupFilesAfterEnv` — `fetch` [test/client/setup-test-framework.js:L36-L40], `matchMedia` [test/client/setup-test-framework.js:L54-L63], `CSS.supports` [test/client/setup-test-framework.js:L30-L32] — and network is globally blocked by `nock.disableNetConnect()` [test/client/setup-test-framework.js:L9] with `beforeAll`/`afterAll` `nock.activate`/`restore`/`cleanAll` [test/client/setup-test-framework.js:L11-L16,L18-L22]. The wiring is observed (Q5). The _classic_ isolation flake — a mock's call state, a leftover `nock` interceptor, or a fake timer left armed by one file affecting the next — is the **inferred** mechanism: it only manifests when files share a worker and run in a particular order, which is exactly what changes between an isolated run and the full suite.

4. **Build-state artifact [OBSERVED; NON-CANONICAL / environment-specific].** With `packages/*/dist` present (built by `yarn install`), Jest emits `jest-haste-map: duplicate manual mock found: wpcom-proxy-request` and lists `src` vs `dist/cjs` vs `dist/esm` copies (seen verbatim in the Q1-integration, Q1-packages, and Q3 outputs). Because `src` and `dist` share Haste module names, which copy "wins" can depend on scan order. This occurs **only** when `dist` exists (not in a fresh unbuilt clone), so it is a build-state contributor, not a canonical guarantee.

**Conclusion.** A test that quietly depends on (a) a particular environment's globals, (b) a particular resolution target, or (c) clean shared mock/network state will pass when run by itself — where its assumptions happen to hold and nothing precedes it — yet can fail in the full suite, where a differently-environed neighbour, a suite-specific `moduleNameMapper`, an order-dependent Haste collision, or leaked mock/`nock`/timer state changes the conditions under which that same file executes. The concrete failing case is a function of ordering; the ingredients above are the observed reasons the ordering matters.

---

## Coverage pass

| Question / named item | Where addressed | Evidence |
| --- | --- | --- |
| **Q1** — runtime env of each of the six commands, and how they differ | Q1 | `--showConfig` (4 node), packages `22/36`, apps `3` jsdom, per-suite probes |
| `test-build-tools` / `test-client` / `test-integration` / `test-apps` / `test-packages` / `test-server` (all six by name) | Q1 command table | [package.json:L121,L122,L125,L127,L129,L131] + probe per suite |
| aggregate `test` excludes apps & integration | Q1 | [package.json:L120] |
| **Q2** — a capability in one context but not another | Q2 | client node-vs-jsdom probe |
| `window` / `document` / `self` / `localStorage` (jsdom-only) | Q2 table | `BLITZY_PROBE_JSON` |
| `setImmediate` (reverse case: node-only) | Q2 | `BLITZY_PROBE_JSON` |
| 498 docblock count, all `jsdom` | Q2 | grep count + histogram (x2) |
| **Q3** — which file an internal dep loads; differs by execution? | Q3 | `calypso-products` 17/243; in-suite `require.resolve` |
| `@automattic/calypso-products` -> `@automattic/calypso-config` -> `@automattic/create-calypso-config` | Q3 | [packages/calypso-products/package.json:L39]; resolve outputs |
| **Q4** — where imports are overridden; runtime resolution | Q4 | direct resolver script; per-suite `moduleNameMapper` matrix |
| custom resolver `calypso:src` > `main` | Q4 | [packages/calypso-jest/src/module-resolver.js:L18]; resolver output |
| all five `moduleNameMapper` suites (client/integration/server/packages/apps) | Q4 matrix | [test/client/jest.config.js:L11] / [test/integration/jest.config.js:L3] / [test/server/jest.config.js:L10-L11] / [test/packages/jest.config.js:L6] / — (apps: none) |
| **Q5** — init order; who provides browser APIs; when available | Q5 | 4 `BLITZY_ORDER` lines |
| each lifecycle phase (env create -> setupFiles -> framework -> setupFilesAfterEnv -> test) | Q5 table | probe |
| **Q6** — read-only proof | Q6 | scoped `git status --porcelain` empty |
| Synthesis — "passes in isolation, fails in full suite" | Synthesis | ties Q1–Q6 |

---

## Appendix — probe sources (created under throwaway paths, then deleted)

Every probe body below is embedded **inline** in the relevant COMMAND block above (so each block is runnable as written); the sources are consolidated here for convenience. Q6 confirms none of them remain on disk after each block cleans up.

**Globals probe** — used for the per-suite `env-node.js` files (and, prefixed with `/** @jest-environment jsdom */`, for `env-jsdom.js`):

```js
test( 'blitzy env probe', () => {
	const info = {
		hasWindow: typeof window,
		hasDocument: typeof document,
		hasNavigator: typeof navigator,
		userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : null,
		hasSelf: typeof self,
		hasProcess: typeof process,
		hasSetImmediate: typeof setImmediate,
		hasTextEncoder: typeof TextEncoder,
		hasFetch: typeof fetch,
		hasMatchMedia: typeof matchMedia,
		hasResizeObserver: typeof ResizeObserver,
		hasCSS: typeof CSS,
		hasReadableStream: typeof ReadableStream,
		hasStructuredClone: typeof structuredClone,
		hasLocalStorage: typeof localStorage,
	};
	console.log( 'BLITZY_PROBE_JSON=' + JSON.stringify( info ) );
	expect( true ).toBe( true );
} );
```

**Resolve probe** — used for the packages and client `resolve.js` files (Q3):

```js
test( 'blitzy resolve probe', () => {
	console.log( 'BLITZY_RESOLVE_CONFIG=' + require.resolve( '@automattic/calypso-config' ) );
	try {
		console.log( 'BLITZY_RESOLVE_CCC=' + require.resolve( '@automattic/create-calypso-config' ) );
	} catch ( e ) {
		console.log( 'BLITZY_RESOLVE_CCC_ERR=' + e.message.split( '\n' )[ 0 ] );
	}
	expect( true ).toBe( true );
} );
```

**Direct resolver script (Q4)** and **lifecycle instrumentation config (Q5)** are shown in full inside their respective COMMAND blocks above; both are throwaway (`node - <<'NODE'` writes no file; the Q5 config lives under `client/__blitzy_probe__/` and is deleted in the same block).

---

_Document generated from direct runtime observation on branch `wp-calypso_be7e5cc64162` (HEAD `be7e5cc641`), Jest `29.7.0` / jsdom `20.0.3` / Node `v22.23.1`. All probe directories are created and removed within their own command blocks. After this document is committed, `git status --porcelain` is empty; the only tracked artifact this task adds is this file._
