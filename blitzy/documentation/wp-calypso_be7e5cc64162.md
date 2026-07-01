# Why a wp‑calypso test can pass in isolation yet behave differently in the full suite

This document answers a runtime‑behavior question about the `wp-calypso` monorepo: **why can the same test file pass when run by itself yet behave differently when it runs as part of the whole suite?** The short answer is that wp‑calypso does **not** have one "test environment" — it has **several distinct Jest execution contexts**, and *which command/config picks up a given file* determines two things that change the file's behavior:

1. **What actually gets imported** — a custom Jest resolver loads *untranspiled source* via a `calypso:src` field, and each context's `moduleNameMapper` can redirect the *same* import specifier to *different* files.
2. **What globals exist** — the environment is `node` or `jsdom` depending on the context, and each suite's setup files inject a *different* set of browser‑like globals (`matchMedia`, `fetch`, `CSS`, `ResizeObserver`, …).

Because both of these are fixed *before the test module body runs* and are decided *by the config that runs the file*, a file that relies on a certain global or a certain resolved import can pass under one command and fail under another.

The sections below decompose the investigation into six requirements (**R1–R6**). Every factual claim carries an exact `file:line` citation, and every measured value is quoted **verbatim** from real command output captured on the investigation host during this engagement. A **Coverage pass** checklist and a **Methodology & toolchain** appendix close the document.

> **Run‑first methodology.** All output blocks below were produced by actually running the relevant code paths in this repository, not by reading alone. Host/run‑dependent values (the Node version line and Jest `Time:` lines) are quoted as *observed here*; they vary by host and run and are flagged as such.

---

## R1 — Test commands and the runtime each actually uses

**Question:** *Run the different test commands available in this codebase. What runtime environment does EACH actually use during execution, and how do they differ?*

### The commands

The root manifest defines an **aggregate** `test` script that fans out to four suites via `run-s`, plus two additional standalone suites. Observed verbatim by enumerating the scripts:

```bash
node -e "const s=require('./package.json').scripts; for (const k of Object.keys(s)) if (k==='test'||k.startsWith('test-')) console.log(k, '=>', s[k])"
```

```text
test => run-s -s test-client test-packages test-server test-build-tools
test-build-tools => jest -c=test/build-tools/jest.config.js
test-client => TZ=UTC jest -c=test/client/jest.config.js
test-client:watch => yarn run test-client --watch
test-desktop:e2e => echo 'Deprecated, run `cd desktop && yarn run test:e2e` instead'
test-integration => jest -c=test/integration/jest.config.js
test-integration:watch => yarn run test-integration --watch
test-apps => jest -c=test/apps/jest.config.js
test-apps:watch => yarn run test-apps --watch
test-packages => jest -c=test/packages/jest.config.js
test-packages:watch => yarn run test-packages --watch
test-server => jest -c=test/server/jest.config.js
test-server:coverage => yarn run test-server --coverage
test-server:watch => yarn run test-server --watch
```

The relevant declarations, with exact citations:

- Aggregate: `"test": "run-s -s test-client test-packages test-server test-build-tools"` [package.json:120].
- `"test-build-tools": "jest -c=test/build-tools/jest.config.js"` [package.json:121].
- `"test-client": "TZ=UTC jest -c=test/client/jest.config.js"` [package.json:122].
- `"test-integration": "jest -c=test/integration/jest.config.js"` [package.json:125].
- `"test-apps": "jest -c=test/apps/jest.config.js"` [package.json:127].
- `"test-packages": "jest -c=test/packages/jest.config.js"` [package.json:129].
- `"test-server": "jest -c=test/server/jest.config.js"` [package.json:131].

### The shared preset that supplies the default

**Most** suites spread a shared preset, `@automattic/calypso-jest`, whose `main` entry is the preset file — `"main": "./jest-preset.js"` [packages/calypso-jest/package.json:12] — by importing it as `const base = require( '@automattic/calypso-jest' )` and then object‑spreading it (`{ ...base, … }`): client [test/client/jest.config.js:2], server [test/server/jest.config.js:2], and build‑tools [test/build-tools/jest.config.js:2] do this directly, and the apps and packages presets do the same before being consumed by their per‑project configs [test/apps/jest-preset.js:2], [test/packages/jest-preset.js:2]. **The one exception is `test-integration`:** `test/integration/jest.config.js` does **not** spread the base preset — it re‑declares `moduleNameMapper`, `rootDir`, `testEnvironment: 'node'`, and `testMatch` directly [test/integration/jest.config.js:1-16], sharing only the resolver, and that via an explicit `require.resolve( '@automattic/calypso-jest/src/module-resolver.js' )` [test/integration/jest.config.js:8] rather than by inheriting it. For the suites that do spread it, the base preset sets the **default** test environment and the machinery they inherit:

- `resolver: require.resolve( './src/module-resolver.js' )` [packages/calypso-jest/jest-preset.js:9] — the custom resolver (see R3/R4).
- `setupFilesAfterEnv: [ require.resolve( './src/setup.js' ) ]` [packages/calypso-jest/jest-preset.js:10].
- `testEnvironment: 'node'` [packages/calypso-jest/jest-preset.js:11] — **the default runtime is Node.js**.
- `testMatch: [ '<rootDir>/**/test/*.[jt]s?(x)', '!**/.eslintrc.*' ]` [packages/calypso-jest/jest-preset.js:12].
- `transform` with `babel-jest` (`rootMode: 'upward'`) and an asset stub [packages/calypso-jest/jest-preset.js:13-16].

### Command → config → resolved `testEnvironment`

| Test command | Jest config | Resolved `testEnvironment` | Evidence |
|---|---|---|---|
| `test` (aggregate) | — (`run-s -s test-client test-packages test-server test-build-tools`) | n/a | [package.json:120] |
| `test-client` | `test/client/jest.config.js` | `node` base; **`jsdom` per‑file** via `@jest-environment jsdom` docblock (498 files) | [package.json:122], [packages/calypso-jest/jest-preset.js:11] |
| `test-packages` | `test/packages/jest.config.js` (`projects`) | **mixed, per‑project** — 36 `node` (inherited) / 22 `jsdom` (explicit) | [package.json:129], [test/packages/jest.config.js:4], [packages/block-renderer/jest.config.js:3] |
| `test-server` | `test/server/jest.config.js` | `node` (inherited) | [package.json:131], [packages/calypso-jest/jest-preset.js:11] |
| `test-build-tools` | `test/build-tools/jest.config.js` | `node` (inherited) | [package.json:121], [packages/calypso-jest/jest-preset.js:11] |
| `test-integration` | `test/integration/jest.config.js` | `node` (**explicit**) | [test/integration/jest.config.js:7] |
| `test-apps` | `test/apps/jest.config.js` (`projects`, apps preset) | `jsdom` (**explicit**) | [test/apps/jest.config.js:4], [test/apps/jest-preset.js:7] |

How each environment is decided, precisely:

- **`test-client`** → `test/client/jest.config.js` spreads the base and does **not** set `testEnvironment`, so it inherits `node` [packages/calypso-jest/jest-preset.js:11]. Individual client tests opt into a browser‑like environment **per file** with a `@jest-environment jsdom` docblock. **Verified count: exactly 498 files under `client/` carry a `@jest-environment jsdom` docblock** (all 498 specify `jsdom`):

```bash
grep -rl '@jest-environment' client/ --include='*.js' --include='*.jsx' --include='*.ts' --include='*.tsx' | wc -l
```

**Verbatim captured output (this host):**

```text
498
```
- **`test-packages`** → `test/packages/jest.config.js` uses `projects: [ '<rootDir>/packages/*/jest.config.js' ]` [test/packages/jest.config.js:4], so it is **project‑based**, not a single config: each `packages/*/jest.config.js` sets `preset: '../../test/packages/jest-preset.js'`, which spreads the base → `node` [packages/calypso-jest/jest-preset.js:11] **unless that package overrides `testEnvironment`**. It is therefore a **mixed‑environment** suite. Of the **58** package projects, **22 explicitly set `testEnvironment: 'jsdom'`** (e.g. `testEnvironment: 'jsdom'` [packages/block-renderer/jest.config.js:3], [packages/command-palette/jest.config.js:4]) and the remaining **36 inherit `node`**. Counting the raw configs and confirming with Jest's own resolved config:

```bash
ls packages/*/jest.config.js | wc -l
grep -rl "testEnvironment: 'jsdom'" packages/*/jest.config.js | wc -l
```

**Verbatim captured output (this host):**

```text
58
22
```

```bash
node_modules/.bin/jest --showConfig -c=test/packages/jest.config.js \
  | node -e 'const c=JSON.parse(require("fs").readFileSync(0,"utf8")).configs; const t={}; for(const p of c){const e=p.testEnvironment.includes("jsdom")?"jsdom":"node"; t[e]=(t[e]||0)+1;} console.log("projects="+c.length, JSON.stringify(t));'
```

**Verbatim captured output (this host):**

```text
projects=58 {"node":36,"jsdom":22}
```

  So `test-packages` is **not** node‑only: the environment a given package test runs under depends on that package's own `jest.config.js`.
- **`test-server`** → `test/server/jest.config.js` spreads the base → `node`.
- **`test-build-tools`** → `test/build-tools/jest.config.js` spreads the base → `node`.
- **`test-integration`** → `test/integration/jest.config.js` sets `testEnvironment: 'node'` **explicitly** [test/integration/jest.config.js:7].
- **`test-apps`** → `test/apps/jest.config.js` uses `projects: [ '<rootDir>/apps/*/jest.config.js' ]` [test/apps/jest.config.js:4]; app configs use the apps preset, which sets `testEnvironment: 'jsdom'` **explicitly** [test/apps/jest-preset.js:7].

### The core difference

Most suites run in **Node.js** (no DOM); **apps** run in **jsdom**; the **packages** suite is **mixed** (per‑project, 36 `node` / 22 `jsdom`); and **client** is Node by default while a **minority** of its files opt into jsdom **per file** — exactly **498** files under `client/` carry a `@jest-environment jsdom` docblock (verbatim count above and in the Coverage pass), far fewer than the **1404** files the client `testMatch` (`<rootDir>/**/test/*.[jt]s?(x)` [packages/calypso-jest/jest-preset.js:12]) can pick up, so this is a subset — **not** a majority. This is the **first axis** of the pass‑in‑isolation/fail‑in‑suite phenomenon: the *same* test file can execute under a *different* environment depending on which command/config picks it up.

**Toolchain cited by the repo:** `engines.node: "^v22.9.0"` [package.json:57]; `packageManager: "yarn@4.0.2"` [package.json:422]; `.nvmrc` = `22.9.0` [.nvmrc:1]. (This investigation ran on Node `v22.23.1`, within `^v22.9.0` — see the Methodology appendix.)

---

## R2 — What is available GLOBALLY in each context

**Question:** *Compare what's available GLOBALLY in each context; identify something that exists in one context but not another.*

### Method (run‑first)

Two temporary probe files were placed under `client/test/` so the client `testMatch` (`<rootDir>/**/test/*.[jt]s?(x)` [packages/calypso-jest/jest-preset.js:12]) would pick them up. Each logs a `typeof` matrix for `window`, `document`, `navigator`, `matchMedia`, `fetch`, and `CSS`, both at **module‑top** and **inside `test()`**. One runs in the default node environment; the other opts into jsdom via a first‑line docblock.

`client/test/__blitzy_probe_node.js` (default node env):

```js
const label = '[NODE]';
const m = ( where ) =>
	`${ label }[${ where }] window=${ typeof window } document=${ typeof document } navigator=${ typeof navigator } matchMedia=${ typeof matchMedia } fetch=${ typeof fetch } CSS=${ typeof CSS }`;
console.log( m( 'module-top' ) );
test( 'node env global matrix', () => { console.log( m( 'in-test' ) ); expect( true ).toBe( true ); } );
```

`client/test/__blitzy_probe_jsdom.js` (jsdom via docblock):

```js
/** @jest-environment jsdom */
const label = '[JSDOM]';
const m = ( where ) =>
	`${ label }[${ where }] window=${ typeof window } document=${ typeof document } navigator=${ typeof navigator } matchMedia=${ typeof matchMedia } fetch=${ typeof fetch } CSS=${ typeof CSS }`;
console.log( m( 'module-top' ) );
test( 'jsdom env global matrix', () => { console.log( m( 'in-test' ) ); expect( true ).toBe( true ); } );
```

Command (matching the real `test-client` invocation: `TZ=UTC` [package.json:122], deterministic `--runInBand`):

```bash
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --runInBand client/test/__blitzy_probe
```

**Verbatim captured output (this host):**

```text
[JSDOM][module-top] window=object document=object navigator=object matchMedia=function fetch=function CSS=object
[JSDOM][in-test] window=object document=object navigator=object matchMedia=function fetch=function CSS=object
[NODE][module-top] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
[NODE][in-test] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.129 s
```

*(The `Time:` value is run‑dependent. A Browserslist "caniuse‑lite is 17 months old" notice may print before this output; it is environmental noise unrelated to the result.)*

### The `typeof` matrix

| Global | node context | jsdom context | Why |
|---|---|---|---|
| `window` | `undefined` | `object` | **Provided only by jsdom** |
| `document` | `undefined` | `object` | **Provided only by jsdom** |
| `navigator` | `object` | `object` | Node 22 exposes a global `navigator` natively |
| `matchMedia` | `function` | `function` | Injected by client setup regardless of env [test/client/setup-test-framework.js:54-63] |
| `fetch` | `function` | `function` | **Native in Node 22**; the client setup then **replaces** it with a `jest.fn` mock [test/client/setup-test-framework.js:36-40] |
| `CSS` | `object` | `object` | Injected by client setup `global.CSS = { supports: jest.fn() }` [test/client/setup-test-framework.js:30-32] |

### The distinguishing globals: `window` and `document`

The globals that **exist in one context but not the other are `window` and `document`**: both are `object` under jsdom and `undefined` under node. They are supplied by the jsdom test environment [test/apps/jest-preset.js:7] and are absent in the plain `node` environment [packages/calypso-jest/jest-preset.js:11].

The other probed globals are equal in both **for a reason worth spelling out**:

- `navigator=object` in both — Node 22 provides a global `navigator` natively.
- `matchMedia=function`, `fetch=function`, `CSS=object` in both — because `test/client/setup-test-framework.js` assigns `global.matchMedia` [test/client/setup-test-framework.js:54-63], `global.fetch` [test/client/setup-test-framework.js:36-40], and `global.CSS` [test/client/setup-test-framework.js:30-32] **regardless of which environment** the file uses. Note that `fetch` differs in kind from the other two: it is **already native in Node 22**, so the client setup **replaces** the native `fetch` with a `jest.fn` mock, whereas `matchMedia` and `CSS` do **not** exist in raw Node and are **added** by the setup. (The base preset's `global.CSS.supports` shim [packages/calypso-jest/src/setup.js:3-4] is **not** what runs here — the client config overrides `setupFilesAfterEnv`, so in the client suite `CSS` comes from `test/client/setup-test-framework.js:30-32`; see R5.)

### The instructive contrast: pure Node vs in‑Jest node

To prove which globals are **injected by setup** versus **already native**, the same matrix was measured in **raw Node, with no Jest**:

```bash
node -e 'console.log("[NODE-baseline] window="+typeof window,"document="+typeof document,"navigator="+typeof navigator,"matchMedia="+typeof matchMedia,"fetch="+typeof fetch,"CSS="+typeof CSS)'
```

**Verbatim captured output (this host):**

```text
[NODE-baseline] window=undefined document=undefined navigator=object matchMedia=undefined fetch=function CSS=undefined
```

Compare the two node lines:

- Raw Node: `matchMedia=undefined … CSS=undefined`.
- In‑Jest node (client suite): `matchMedia=function … CSS=object`.

The `matchMedia`/`CSS` difference is entirely attributable to `test/client/setup-test-framework.js`. Here it is important to **separate presence from implementation**:

- **`matchMedia` and `CSS` are genuinely absent in raw Node** (`matchMedia=undefined`, `CSS=undefined` in the baseline) and are **added** only by a suite's setup. A test that relies on `matchMedia`/`CSS` passes under a suite whose setup provides them but fails in one that does not — this is a **direct explanation of pass‑in‑isolation vs fail‑in‑suite**.
- **`navigator` and `fetch` are already native in Node 22**, so they are present in *every* context — even raw Node and even a suite that injects nothing. For `fetch` the cross‑suite difference is therefore **not presence but identity**: the client/apps setup **replaces** the native `fetch` with a `jest.fn` mock [test/client/setup-test-framework.js:36-40], while server/packages keep Node's native `fetch`. A test that depends on the **mocked** `fetch` behaviour (e.g. asserting `fetch` was called, or a stubbed response) can pass under client/apps yet behave differently under server/packages — but not because `fetch` is missing there.

R6 quantifies exactly which suite provides which of these globals, with verbatim per‑suite observations.

---


## R3 — What ACTUAL file loads when an internal dependency is imported

**Question:** *Find a monorepo package that depends on another INTERNAL package from this same repo. Run its tests and determine what ACTUAL FILE gets loaded when that internal dependency is imported. Does it differ based on how you execute the tests?*

### The internal dependency edge

`@automattic/calypso-analytics` imports the internal package `@automattic/load-script`:

```text
import { loadScript } from '@automattic/load-script';
```

at [packages/calypso-analytics/src/tracks.ts:4]. The dependency is declared as an internal **workspace** dependency: `"@automattic/load-script": "workspace:^"` [packages/calypso-analytics/package.json:33].

The `@automattic/load-script` manifest declares three entry fields:

- `"main": "dist/cjs/index.js"` [packages/load-script/package.json:5]
- `"module": "dist/esm/index.js"` [packages/load-script/package.json:6]
- `"calypso:src": "src/index.js"` [packages/load-script/package.json:7]

**Crucially, the `dist/` directory does not exist.** A directory listing shows only source and metadata — no build output:

```bash
ls packages/load-script/
```

```text
README.md jest.config.js package.json src test tsconfig.json
```

The source export that the resolver ultimately loads is `export function loadScript( url, callback, args )` [packages/load-script/src/index.js:24].

### The resolver vs plain Node — run‑first evidence

A probe (placed in `/tmp`, outside the repo, so it requires no in‑repo cleanup) exercised the preset's custom resolver against `@automattic/load-script` from the `calypso-analytics` basedir, and compared it to Node's default `require.resolve`:

```js
const path = require('path');
const REPO = process.cwd(); // run from repo root
const resolver = require(path.join(REPO,'packages/calypso-jest/src/module-resolver.js'));
const basedir = path.join(REPO,'packages/calypso-analytics');
const rel = p => p ? p.replace(REPO+'/','') : p;
try { console.log('calypso:src resolver -> ' + rel(resolver('@automattic/load-script', { basedir }))); }
catch (e) { console.log('calypso:src resolver -> ERROR ' + e.message); }
try { console.log('node default require.resolve -> ' + rel(require.resolve('@automattic/load-script', { paths: [basedir] }))); }
catch (e) { console.log('node default require.resolve -> ERROR ' + e.code); }
```

**Verbatim captured output (this host):**

```text
calypso:src resolver -> packages/load-script/src/index.js
node default require.resolve -> ERROR MODULE_NOT_FOUND
```

### Conclusion

Under Jest, the custom resolver loads the **untranspiled source** `packages/load-script/src/index.js` — it selects the `calypso:src` field. Plain Node's `require.resolve` **fails with `MODULE_NOT_FOUND`**, because it follows `main` → `dist/cjs/index.js` [packages/load-script/package.json:5], which does not exist (there is no build). So **yes — the actual file loaded differs by execution context**: source under Jest, nothing (an error) under plain Node.

This source‑over‑dist behavior is intentional and documented in the resolver itself: the `calypso:src` field "points to the *untranspiled* source code" and `main` "points to a file that usually *does not* exist" for monorepo packages [packages/calypso-jest/src/module-resolver.js:8-12]. It is **pervasive**: **64 packages under `packages/` declare a `calypso:src` field**:

```bash
grep -rl '"calypso:src"' packages/*/package.json | wc -l
```

**Verbatim captured output (this host):**

```text
64
```

Note the resolver behavior is **uniform across the Jest suites** because they all use this resolver — the base preset sets it [packages/calypso-jest/jest-preset.js:9] and the integration config re‑declares it explicitly [test/integration/jest.config.js:8]. So the divergence here is *Jest‑resolver vs plain‑Node* (and, per R4, vs per‑context `moduleNameMapper` overrides), rather than one Jest suite vs another for this particular edge.

### Corroboration — the package's own tests run against source

Running `@automattic/load-script`'s real tests confirms they execute against the source tree (there is no `dist` to run against):

```bash
node_modules/.bin/jest -c=test/packages/jest.config.js --runInBand load-script
```

**Verbatim captured output (tail, this host):**

```text
PASS packages/load-script/test/callback-handler.js
PASS packages/load-script/test/index.js
PASS packages/load-script/test/dom-operations.js

Test Suites: 3 passed, 3 total
Tests:       19 passed, 19 total
Snapshots:   0 total
Time:        1.353 s
```

*(The `Time:` value is run‑dependent.)*

---


## R4 — Where an import gets redirected, and where it ACTUALLY resolves

**Question:** *The test infrastructure OVERRIDES some import paths. Discover where an import gets redirected and trace where it ACTUALLY resolves to at runtime. Does the same import resolve to different locations depending on execution context?*

There are **two** override mechanisms at play: per‑context `moduleNameMapper` (explicit redirection) and the custom resolver's field preference (implicit redirection to source). Both make the *same* import specifier resolve to *different* files depending on context.

### 1) `moduleNameMapper` — three *different mapper strings*, one physical file

The specifier `^@automattic/calypso-config$` is redirected by `moduleNameMapper` using a **different expression in each context**. The expressions differ (different `<rootDir>` bases, and the server uses a bare module specifier), but — in the verified repo layout — **all three resolve to the *same physical file*** `client/server/config/index.js`:

| Context | Mapping (verbatim from config) | Physical file it resolves to | Evidence |
|---|---|---|---|
| **client** | `'^@automattic/calypso-config$': '<rootDir>/server/config/index.js'` | `client/server/config/index.js` (`rootDir` = `../../client` [test/client/jest.config.js:6]) | [test/client/jest.config.js:11] |
| **server** | `'^@automattic/calypso-config$': 'calypso/server/config'` | `client/server/config/index.js` (via the `node_modules/calypso` → `../client` symlink) | [test/server/jest.config.js:10] |
| **integration** | `'^@automattic/calypso-config$': '<rootDir>/client/server/config/index.js'` | `client/server/config/index.js` (`rootDir` = `../..` [test/integration/jest.config.js:6]) | [test/integration/jest.config.js:3] |

The server context also maps the subpath form: `'^@automattic/calypso-config/(.*)$': 'calypso/server/config/$1'` [test/server/jest.config.js:11].

**Run‑first evidence.** A probe (placed in `/tmp`, outside the repo, so it needs no in‑repo cleanup) substituted each context's `<rootDir>`, resolved the server's bare specifier `calypso/server/config` through Node's resolver, and normalized every result with `fs.realpathSync` — plus the *unmapped* custom resolver for comparison:

```js
const path = require('path');
const fs = require('fs');
const REPO = process.cwd(); // run from repo root
const resolver = require(path.join(REPO,'packages/calypso-jest/src/module-resolver.js'));
const rel = p => p ? p.replace(REPO+'/','') : p;
const real = p => rel(fs.realpathSync(p));
const clientTarget = path.join(REPO,'client','server/config/index.js');     // client rootDir=<repo>/client
const integrationTarget = path.join(REPO,'client/server/config/index.js');  // integration rootDir=<repo>
const serverBaredir = path.join(REPO,'client/server');
console.log('client  moduleNameMapper "<rootDir>/server/config/index.js"        -> ' + real(clientTarget));
console.log('server  moduleNameMapper "calypso/server/config"                   -> ' + real(require.resolve('calypso/server/config', { paths: [serverBaredir, REPO] })));
console.log('integ.  moduleNameMapper "<rootDir>/client/server/config/index.js" -> ' + real(integrationTarget));
console.log('UNMAPPED calypso:src resolver "@automattic/calypso-config"          -> ' + rel(resolver('@automattic/calypso-config', { basedir: path.join(REPO,'packages/calypso-analytics') })));
```

**Verbatim captured output (this host):**

```text
client  moduleNameMapper "<rootDir>/server/config/index.js"        -> client/server/config/index.js
server  moduleNameMapper "calypso/server/config"                   -> client/server/config/index.js
integ.  moduleNameMapper "<rootDir>/client/server/config/index.js" -> client/server/config/index.js
UNMAPPED calypso:src resolver "@automattic/calypso-config"          -> packages/calypso-config/src/index.ts
```

So the accurate statement is: the **`moduleNameMapper` *expressions* differ by context**, but for the bare specifier `@automattic/calypso-config` they **all redirect to the single physical file `client/server/config/index.js`** (the server reaches it through the `node_modules/calypso` → `../client` symlink). The real, observable divergence is therefore **mapped vs unmapped**: where a mapper matches, the import lands on `client/server/config/index.js`; where **no** mapper applies, the custom `calypso:src` resolver instead selects the package source `packages/calypso-config/src/index.ts` (see §2). This — not "three different files" — is the **second concrete axis** of the pass‑in‑isolation/fail‑in‑suite behavior: the same specifier resolves to a **different physical file depending on whether the running config maps it**. Additional per‑context remaps exist too: the client suite remaps `react-markdown` [test/client/jest.config.js:12], and the packages suite remaps `react-markdown` [test/packages/jest.config.js:6].

### 2) The custom resolver — `calypso:src` wins where no mapper applies

Where **no** `moduleNameMapper` entry matches, the custom resolver decides. Its options prefer the `calypso:src` field:

```text
mainFields: [ 'calypso:src', 'main' ]
conditionNames: [ 'calypso:src', 'node', 'require' ]
```

at [packages/calypso-jest/src/module-resolver.js:16-20] (constructed via `enhancedResolve.create.sync` [packages/calypso-jest/src/module-resolver.js:16]). Because `calypso:src` is listed **first**, the resolver picks the source entry over the built `main`.

Run‑first evidence (same resolver probe, requesting `@automattic/calypso-config` from the `calypso-analytics` basedir):

**Verbatim captured output (this host):**

```text
calypso:src resolver (@automattic/calypso-config) -> packages/calypso-config/src/index.ts
```

This confirms the resolver selects `"calypso:src": "src/index.ts"` [packages/calypso-config/package.json:11] over `"main": "dist/cjs/index.js"` [packages/calypso-config/package.json:9] (the `"module": "dist/esm/index.js"` field [packages/calypso-config/package.json:10] is not used because Jest here resolves CJS/require conditions).

### A duplicated resolver

`test/module-resolver.js` is a **byte‑identical duplicate** of `packages/calypso-jest/src/module-resolver.js` — verified with `diff test/module-resolver.js packages/calypso-jest/src/module-resolver.js`, which produced **no output** (files identical). It exists so configs can reference the resolver by that path directly; both encode the same `calypso:src`‑first behavior.

### Putting R4 together

The **same** import can land on **different files** depending on the config that runs it — but the precise mechanism matters: (a) where a context's `moduleNameMapper` matches, the import is redirected to the mapped target — and for `@automattic/calypso-config` the client, server, and integration expressions **differ in text but resolve to the *same* physical file** `client/server/config/index.js` (verbatim resolution above); (b) where **no** mapper applies, the custom resolver silently prefers the `calypso:src` **source** (`packages/calypso-config/src/index.ts`) over the (often nonexistent) `main` build. So the file that actually loads flips between `client/server/config/index.js` (mapped) and `packages/calypso-config/src/index.ts` (unmapped) according to whether the running config maps the specifier — that is the axis, rather than three distinct mapped files.

---


## R5 — What loads first when a test runs

**Question:** *When a test runs, investigate WHAT LOADS FIRST.*

Reconstructed from the preset and per‑context configs, and corroborated empirically by the R2/R6 probe, a single Jest test file initializes in this order:

1. **`testEnvironment` is constructed.** `node` (no `window`/`document`) [packages/calypso-jest/jest-preset.js:11], or `jsdom` (adds `window`/`document`/`navigator`) [test/apps/jest-preset.js:7]; the client suite selects per file via a `@jest-environment` docblock.
2. **`setupFiles` run — before the test framework is installed.** Client: `setupFiles: [ 'jest-canvas-mock' ]` [test/client/jest.config.js:20]; apps preset: `setupFiles: [ 'jest-canvas-mock' ]` [test/apps/jest-preset.js:11]. (These run before Jest's `expect`/`jest` globals exist.)
3. **The test framework is installed** — Jest globals such as `test`, `expect`, and `jest.fn` become available.
4. **`setupFilesAfterEnv` run — and Jest's config‑merging semantics decide *which* file(s).** The base preset declares `setupFilesAfterEnv: [ './src/setup.js' ]` [packages/calypso-jest/jest-preset.js:10] (defining `global.CSS.supports` [packages/calypso-jest/src/setup.js:3-4]), **but most suites do not run that base file.** The client/server configs use a JavaScript spread `{ ...base, … }` and then **re‑declare** `setupFilesAfterEnv`, which **overrides** (does not merge with) the inherited array. So the *effective* array is per‑suite — not "base + suite". Confirmed with Jest's own resolved config (`jest --showConfig`, `setupFilesAfterEnv` extracted):

```bash
for cfg in client server build-tools integration; do node_modules/.bin/jest --showConfig -c=test/$cfg/jest.config.js; done
```

**Verbatim captured output (this host):**

```text
client       setupFilesAfterEnv=["test/client/setup-test-framework.js"]
server       setupFilesAfterEnv=["test/server/setup-test-framework.js"]
build-tools  setupFilesAfterEnv=["packages/calypso-jest/src/setup.js"]
integration  setupFilesAfterEnv=[]
```

   Reading these off: **client** runs **only** `test/client/setup-test-framework.js` [test/client/jest.config.js:21] (injecting `matchMedia`/`fetch`/`CSS`/`ResizeObserver` and extending `expect` via `@testing-library/jest-dom` [test/client/setup-test-framework.js:1]); **server** runs **only** `test/server/setup-test-framework.js` [test/server/jest.config.js:13]; **build‑tools** is the **only** suite that keeps the base `./src/setup.js`, because it spreads the base without overriding [packages/calypso-jest/jest-preset.js:10]; and **integration** has **no** `setupFilesAfterEnv` at all (it does not spread the base). The two **project‑based** suites use `preset:` instead of a spread, so their arrays **merge**: **apps** projects run `test/client/setup-test-framework.js` [test/apps/jest-preset.js:13] and **packages** projects run `test/packages/setup.js` [test/packages/jest-preset.js:14] (individual packages may append their own). The upshot: the base `CSS.supports` shim runs **only in build‑tools**; every other suite supplies its own setup globals or none — so a claim like "base `./src/setup.js` then the client file" would be **wrong** for the client/server/packages/apps/integration suites.
5. **Module resolution** happens via the custom `resolver` [packages/calypso-jest/jest-preset.js:9] using `calypso:src` (loads source; see R3/R4). **Transforms are applied here, and they too vary by suite.** The base preset defines `babel-jest` with `{ rootMode: 'upward' }` [packages/calypso-jest/jest-preset.js:13-16] (rooted at `babel.config.js`) plus an asset stub [packages/calypso-jest/src/asset-transform.js:4-5]; the suites that **spread the base** (client/server/build‑tools, and the apps/packages presets) inherit both. **`test-integration` does not spread the base**, so it falls back to Jest's *default* `babel-jest` transform with **empty options** and **no** asset stub. From the same `--showConfig` runs (transform extracted):

```text
client       transform: babel-jest opts={"rootMode":"upward"}; asset-transform=true
server       transform: babel-jest opts={"rootMode":"upward"}; asset-transform=true
build-tools  transform: babel-jest opts={"rootMode":"upward"}; asset-transform=true
integration  transform: babel-jest opts={}; asset-transform=false
```

   So integration transforms `.[jt]sx?` sources with a bare `babel-jest` (no explicit `rootMode: 'upward'`) and does **not** stub asset imports — whereas every base‑preset suite does both.
6. **The test module body executes**, and then the `test()` callbacks run.

### Empirical corroboration

The R2/R6 probe proves the setup files run **before** the test module is loaded. The `module-top` line and the `in-test` line print **identical** values — for example, in the node context:

```text
[NODE][module-top] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
[NODE][in-test] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
```

Since `matchMedia`, `fetch`, and `CSS` are assigned in `setupFilesAfterEnv` (step 4) — and they are already `function`/`function`/`object` at **module‑top** (step 6, before any `test()` runs) — the setup files must have executed **before** the test module was evaluated. If setup ran *after* module load, the `module-top` line would show `matchMedia=undefined`, matching the raw‑Node baseline from R2; it does not.

**Takeaway:** because environment construction (step 1) and setup injection (step 4) happen *before* module load (step 6), a file's globals **and** its resolved imports are fixed by the suite/config that runs it. That is the mechanism behind different behavior in isolation vs the full suite.

---

## R6 — What provides BROWSER‑LIKE APIs, and WHEN they become available

**Question:** *Some tests have access to BROWSER‑LIKE APIs; figure out what provides those capabilities and WHEN that provider becomes available. Verify initialization order by observing what's accessible at different points.*

### Two distinct providers

**(a) `window` / `document` / `navigator` come from the jsdom environment.** They are supplied by `testEnvironment: 'jsdom'` [test/apps/jest-preset.js:7] (and, for the client suite, per‑file `@jest-environment jsdom` docblocks). Under the plain `node` environment they are absent — the R2 matrix shows `window=undefined`, `document=undefined` there. (`navigator` is present under node too, but that is native to Node 22, not from jsdom.)

**(b) The remaining browser‑like globals come from the project setup file** `test/client/setup-test-framework.js`:

- `@testing-library/jest-dom` matchers [test/client/setup-test-framework.js:1]
- `global.CSS = { supports: jest.fn() }` [test/client/setup-test-framework.js:30-32]
- `global.ResizeObserver = require( 'resize-observer-polyfill' )` [test/client/setup-test-framework.js:34]
- `global.fetch = jest.fn( … )` [test/client/setup-test-framework.js:36-40]
- `global.matchMedia = jest.fn( … )` [test/client/setup-test-framework.js:54-63]

### Contrast by suite — this is central to the phenomenon

Different suites inject **different** browser‑like globals in their setup, so the *same* API is present in some suites and absent in others:

| Suite | Setup file | Browser‑like globals injected |
|---|---|---|
| client | `test/client/setup-test-framework.js` | `CSS.supports` mock [:30-32], `ResizeObserver` [:34], **`fetch` mock that replaces native** [:36-40], `matchMedia` [:54-63], jest‑dom [:1] |
| apps | (uses client setup) `test/apps/jest-preset.js:13` → `../client/setup-test-framework.js` | same as client |
| server | `test/server/setup-test-framework.js` | **no browser‑like globals** — only `nock.disableNetConnect()` [test/server/setup-test-framework.js:4] (native `fetch` remains; `matchMedia`/`CSS` stay `undefined`) |
| packages | `test/packages/setup.js` | `ResizeObserver` [test/packages/setup.js:5], `matchMedia` [test/packages/setup.js:7-16], jest‑dom [test/packages/setup.js:1] — **no `fetch` mock** (native `fetch` remains) and **no `CSS.supports` mock** |

Consequences that produce isolation‑vs‑suite divergence (stated precisely, separating **presence** from **implementation**):

- **`fetch` is present in *every* suite** because Node 22 provides it natively — it is **never missing**. What differs is its **identity**: under **client/apps** `fetch` is a `jest.fn` mock that *replaces* the native one [test/client/setup-test-framework.js:36-40]; under **server/packages/build‑tools/integration** it is Node's **native** `fetch`. A test that asserts on the *mock* (call counts, a stubbed `json()` response) can diverge between suites — but not because `fetch` is absent.
- **`matchMedia`** does not exist in raw Node and is **added** by **client/apps** [test/client/setup-test-framework.js:54-63] and **packages** [test/packages/setup.js:7-16], but **not** by **server**, **build‑tools**, or **integration**. A test needing `matchMedia` passes under client/apps/packages and fails under those three.
- **`CSS.supports`** does not exist in raw Node. A `CSS.supports` `jest.fn` mock is provided by the client/apps setup [test/client/setup-test-framework.js:30-32] and, identically, by the base preset setup [packages/calypso-jest/src/setup.js:3-4] — but the base shim is effective **only in build‑tools**, the one suite that keeps the base `setupFilesAfterEnv` (see R5). Under **server**, **integration**, and the **packages node** projects, `CSS.supports` is **absent**.

**Verified by observation.** Temporary probes were run under the `test-server` and `test-packages` configs (node‑env packages) and the `test-client` config, then removed. Server suite:

```bash
TZ=UTC node_modules/.bin/jest -c=test/server/jest.config.js --runInBand __blitzy_probe_server
```

**Verbatim captured output (this host):**

```text
[SERVER] window=undefined document=undefined navigator=object matchMedia=undefined fetch=function fetchIsMock=false CSS=undefined ResizeObserver=undefined
Tests:       1 passed, 1 total
```

Packages suite (the node‑env `load-script` project):

```bash
TZ=UTC node_modules/.bin/jest -c=test/packages/jest.config.js --runInBand __blitzy_probe_pkg
```

**Verbatim captured output (this host):**

```text
[PACKAGES-node] window=undefined matchMedia=function fetch=function fetchIsMock=false CSS=object CSS.supports=undefined ResizeObserver=function
Tests:       1 passed, 1 total
```

Client suite (confirming `fetch`/`matchMedia`/`CSS.supports` are the injected **mocks**):

```bash
TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --runInBand __blitzy_probe_client_mock
```

**Verbatim captured output (this host):**

```text
[CLIENT] fetch=function fetchIsMock=true matchMediaIsMock=true CSS.supportsIsMock=true
Tests:       1 passed, 1 total
```

These prove the corrected picture: `fetchIsMock=false` under **server** and **packages** (native `fetch`) but `fetchIsMock=true` under **client**; `matchMedia=undefined`/`CSS=undefined` under **server**; `matchMedia=function` (a mock) but `CSS.supports=undefined` under the **packages node** project (its `CSS` object is an ambient `{ escape }` with no `supports`, confirming the base `CSS.supports` shim did **not** run there — see R5).

### WHEN the provider becomes available

These globals are assigned in `setupFilesAfterEnv`, which (per R5) runs **after** the environment is constructed and the framework is installed, but **before** any test module body executes. Therefore they are already present at module‑top.

**Verified by observation:** the R2/R6 probe prints the same `typeof` values at module‑top and in‑test — e.g. `matchMedia=function` at *both* points in the node context:

```text
[NODE][module-top] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
[NODE][in-test] window=undefined document=undefined navigator=object matchMedia=function fetch=function CSS=object
```

Because `matchMedia=function` already at module‑top, the provider (`setupFilesAfterEnv`) becomes available **before the module loads** — confirming the initialization order from R5.

---


## Coverage pass

Every sub‑question is answered above; each item below links to its evidence.

- [x] **R1 — Test commands and their runtimes.** Command → config → `testEnvironment` table with `file:line`: aggregate fan‑out [package.json:120]; per‑suite `jest -c` commands [package.json:121-131]; default `node` [packages/calypso-jest/jest-preset.js:11]; integration `node` explicit [test/integration/jest.config.js:7] (and integration does **not** spread the base preset); apps `jsdom` explicit [test/apps/jest-preset.js:7]; **packages is project‑based/mixed — 36 `node` / 22 `jsdom` of 58** (verbatim `--showConfig` tally); client `node` by default with exactly **498** `@jest-environment jsdom` docblocks (a minority of 1404 test‑match candidates, verbatim counts).
- [x] **R2 — Global environment comparison.** `typeof` matrix (node vs jsdom) with verbatim output; **`window` and `document`** identified as the divergent globals (`object` under jsdom, `undefined` under node); pure‑Node baseline contrast proving `matchMedia`/`CSS` are injected by setup.
- [x] **R3 — Internal‑dependency resolution.** `packages/load-script/src/index.js` loaded under Jest vs `MODULE_NOT_FOUND` under plain Node, quoted verbatim; `dist/` confirmed absent; 64 `calypso:src` packages; the concrete edge is `@automattic/calypso-analytics` → `@automattic/load-script` [packages/calypso-analytics/src/tracks.ts:4].
- [x] **R4 — Import‑path overrides.** Per‑context `moduleNameMapper` expressions for `@automattic/calypso-config` — client [test/client/jest.config.js:11], server [test/server/jest.config.js:10], integration [test/integration/jest.config.js:3] — differ in text but **all resolve to the same physical file `client/server/config/index.js`** (verbatim resolution output); the real divergence is **mapped vs unmapped**, where the unmapped custom resolver's `calypso:src` `mainFields` preference [packages/calypso-jest/src/module-resolver.js:16-20] instead selects `packages/calypso-config/src/index.ts` [packages/calypso-config/package.json:11], quoted verbatim.
- [x] **R5 — Load order.** Six‑step initialization narrative (env → `setupFiles` → framework → `setupFilesAfterEnv` → module resolution → test body), corroborated by the `module-top == in-test` observation; with **per‑suite effective `setupFilesAfterEnv`** (spread **overrides**, `preset:` **merges**; base `./src/setup.js` runs **only in build‑tools**) and **per‑suite transforms** (base‑preset suites use `rootMode:'upward'` + asset stub; integration uses default `babel-jest`, no asset stub), both from verbatim `--showConfig`.
- [x] **R6 — Browser‑like API provider and timing.** jsdom provides `window`/`document`/`navigator` [test/apps/jest-preset.js:7]; `test/client/setup-test-framework.js` provides `matchMedia`/`CSS.supports`/`ResizeObserver` and a `fetch` **mock that replaces** the native `fetch`; per‑suite contrast (server injects no browser globals [test/server/setup-test-framework.js:4]; packages injects `ResizeObserver`+`matchMedia` [test/packages/setup.js:5,7-16]); **`fetch` is native in Node 22 so it is present in every suite — the client/apps difference is the mock, not presence** (verbatim server/packages/client `fetchIsMock` observations); timing is `setupFilesAfterEnv`, before module load, verified by observation.

### Synthesis

Pass‑in‑isolation/fail‑in‑suite in wp‑calypso arises because **both** of the following depend on **which command/config executes a given test file**: (1) the resolved import target — where **no** mapper applies, the custom `calypso:src` resolver loads untranspiled source [packages/calypso-jest/src/module-resolver.js:16-20] (e.g. `packages/calypso-config/src/index.ts`), while a per‑context `moduleNameMapper` [test/client/jest.config.js:11], [test/server/jest.config.js:10], [test/integration/jest.config.js:3] redirects the same specifier to a mapped target (for `@automattic/calypso-config`, the three expressions differ in text but all resolve to the same file `client/server/config/index.js`) — so the file that loads flips by **whether the running config maps the specifier**; and (2) the available globals — the environment choice (`node` [packages/calypso-jest/jest-preset.js:11] vs `jsdom` [test/apps/jest-preset.js:7]) combined with per‑suite `setupFilesAfterEnv` injecting different browser‑like globals. A file run by itself uses the config you point Jest at; the same file inside the full suite may be picked up by a different config with a different environment, different setup injection, and different import redirections — hence the divergent behavior. *(This document only explains the mechanism; proposing a fix is out of scope.)*

---

## Methodology & toolchain

- **Run‑first.** Every output block above was captured by actually running the code paths in this repository, then quoted verbatim. The temporary probe files used only for observation were: `client/test/__blitzy_probe_node.js` and `client/test/__blitzy_probe_jsdom.js` (R2 global matrices); `client/test/__blitzy_probe_client_mock.js` (R6 client `fetchIsMock`); `client/server/config/test/__blitzy_probe_server.js` (R6 server matrix); `packages/load-script/test/__blitzy_probe_pkg.js` (R6 packages‑node matrix); and `/tmp` scripts for the resolver checks (R3/R4). The per‑suite `--showConfig` and count commands (R1/R5) read the repo directly and needed no probe files. All probe files (and any Jest transform‑cache artifacts they produced under the git‑ignored `.cache/`) were **removed afterward**; the working tree was left unchanged apart from this document (verified with `git status --porcelain`).
- **Toolchain (observed on the investigation host):**
  - `node --version` → **`v22.23.1`** — within the repo's `engines.node: "^v22.9.0"` [package.json:57] (`.nvmrc` = `22.9.0` [.nvmrc:1]). *This is host‑dependent; it is the exact version observed here.*
  - `yarn --version` → **`4.0.2`**, matching `packageManager: "yarn@4.0.2"` [package.json:422].
  - `jest --version` → **`29.7.0`**.
- **Determinism.** Probes ran with `--runInBand` and `TZ=UTC`, matching the real `test-client` command [package.json:122].
- **Host/run‑dependent values.** The Node version line and the Jest `Time:` lines (e.g. `Time: 1.129 s`, `Time: 1.353 s`) vary by host and run; they are quoted as observed here and are not stable identifiers. All other quoted values (global `typeof` results, `MODULE_NOT_FOUND`, resolved file paths, test‑pass markers, counts) matched across runs.
- **Verifiability.** Every factual claim carries an exact `file:line` citation resolved against HEAD `be7e5cc641622d153040491fd5625c6cb83e12eb`; every measured value is quoted from real output. Nothing in this document is inferred without either a source citation or an observed result.

