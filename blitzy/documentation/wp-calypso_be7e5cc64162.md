# wp-calypso Testing Infrastructure — An Onboarding Walkthrough (Run-First, Evidence-Backed)

This document explains how the `Automattic/wp-calypso` **testing infrastructure** works, aimed at a developer who is onboarding. It answers seven specific questions (Q1–Q7). It was produced **run-first**: before writing each answer I actually built/ran the relevant code path in the repository's canonical runtime, captured the **real, complete, unedited** output, and only then wrote the prose. Every command shown was really executed; every output block is the literal bytes the tool emitted.

**How to read the tags.** Each substantive claim is tagged:

- **[observed]** — backed by output captured from a command I actually ran (shown in the accompanying fenced block).
- **[inferred]** — read from source code (with a `file:line` citation) but not directly printed at runtime; where reasonable I still confirmed it by running.

Citations use the form `path:line` (or `path:start-end`). Line numbers refer to the repository's committed source at branch `wp-calypso_be7e5cc64162`. This document is the only file added by this task; the source repository is otherwise left byte-for-byte unchanged (see the Closing note for the read-only proof).

---

## Preamble — Canonical Runtime & Prerequisites

**Direct answer:** The canonical runtime is **Node `^v22.9.0`** and **`yarn@4.0.2`**, and `yarn install` (populating `node_modules`) is a hard prerequisite before anything — Jest suites or the config observation one-liners — can run.

### Runtime facts

- Node is pinned to `^v22.9.0`. The `engines` block begins at `package.json:56` and declares the Node constraint at `package.json:57` (`"node": "^v22.9.0"`) and the yarn constraint at `package.json:58` (`"yarn": "^4.0.0"`). The `.nvmrc` file pins `22.9.0` at `.nvmrc:1`. **[inferred]** (from source)
- The package manager is pinned to `yarn@4.0.2` at `package.json:422` (`"packageManager": "yarn@4.0.2"`), resolved via `.yarnrc.yml:5` (`yarnPath: .yarn/releases/yarn-4.0.2.cjs`). The release file is present on disk (`2733890` bytes ≈ 2.73 MB). `.yarnrc.yml` also sets `nodeLinker: node-modules` (`.yarnrc.yml:3`) and `enableGlobalCache: true` (`.yarnrc.yml:4`). **[observed]** (release-file size measured; contents read)
- `yarn install` is a **hard prerequisite**: nothing runs until `node_modules` exists. The canonical install command is `node .yarn/releases/yarn-4.0.2.cjs install`. In this environment `node_modules` and `nock` were already present (install had been run during setup), which is why every command below executed successfully. **[observed]**

### Observed Node version (reported, not assumed)

**Command:**

```
node --version
```

**Output (run twice, back-to-back, for stability):**

```
v22.23.1
v22.23.1
```

**[observed]** The installed Node is **`v22.23.1`**, stable across two consecutive runs, and it **satisfies** the `^v22.9.0` pin at `package.json:57`. (Note: an earlier scoping pass recorded `v22.22.2`; the exact patch differs but both satisfy the pin. Reporting exactly what I observed here: `v22.23.1`.) **[observed]**

### Two conflicts worth documenting (documented, NOT fixed — this is a read-only task)

1. **Runtime conflict.** The container setup instructions specify Node 20.x plus `corepack prepare yarn@stable`. The repository manifests, however, require Node `^v22.9.0` (`package.json:57`) and pin `yarn@4.0.2` (`package.json:422`). The `start` script's first guard is `npx check-node-version --package` (`package.json:110`), which would **reject** Node 20. **Resolution:** the repository manifests are authoritative, so the investigation used Node 22.x with the pinned yarn 4.0.2. **[inferred from manifests + observed Node version]**
2. **Config-selection docs conflict.** `config/README.md:3` documents environment selection by `NODE_ENV` only (default `"development"`). The **actual** resolver is `process.env.CALYPSO_ENV || process.env.NODE_ENV || 'development'` at `client/server/config/index.js:6`, i.e. `CALYPSO_ENV` takes precedence over `NODE_ENV`. **Resolution:** the running resolver is authoritative; the README understates the precedence. This is confirmed by the Q1 and Q6 runs below. **[observed via Q1/Q6 runs + file:line]**

### Test topology (7 suites)

The monorepo runs Jest across a **7-suite topology** — **Client**, **Server**, **Packages**, **Applications**, **Build Tools**, **Integration**, and **E2E (Playwright)** — with all suites except E2E sharing the `@automattic/calypso-jest` preset. **[inferred]** (from tech-spec §6.6 and the per-suite `jest.config.js` files read for Q2/Q4; each non-E2E suite `require( '@automattic/calypso-jest' )` and spreads/uses it — e.g. `test/client/jest.config.js:2,5`, `test/server/jest.config.js:2,5`.)

**Runtime used throughout this document:** Node `v22.23.1`, yarn `4.0.2` (invoked as `node .yarn/releases/yarn-4.0.2.cjs …`), default/canonical configuration, all commands run from the repository root.

---

## Q1 — Dev server bring-up

**Sub-question:** Start the development server to confirm it works — and what environment does it resolve to?

**Direct answer:** In its default/canonical configuration (no `CALYPSO_ENV` or `NODE_ENV` set, as a normal developer would start it), the server resolves its environment to **`development`**, loading `config/development.json`. The config module also exposes a working `isEnabled` function. **[observed]**

### Canonical entry point

- `package.json:110` — `"start": "npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build"`. The **first** thing `yarn start` does is enforce the Node-version pin via `check-node-version --package`. **[inferred]**
- `package.json:113` — `"start-build": "BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short"`. **[inferred]**
- Environment precedence lives at `client/server/config/index.js:6` — `env: process.env.CALYPSO_ENV || process.env.NODE_ENV || 'development'` (the options object spans `client/server/config/index.js:5-9`). The module exports the config API via `createConfig( serverData )` at `client/server/config/index.js:11` and attaches `clientData` at `client/server/config/index.js:12`. **[inferred]**

### Real entry point vs. feasible stand-in

A full browser boot of `yarn start` performs a heavy webpack build and then serves `build/server.js`. Rather than depend on a full browser boot, I exercised the **real resolver module** (`client/server/config/index.js`) directly with **no** `NODE_ENV`/`CALYPSO_ENV` set — which is exactly the environment a normal developer's shell has when they run `yarn start`. This is the canonical default path, not a bypass: the same module and the same line (`:6`) decide the environment for the dev server. The claim "the full browser server boots and serves pages" is **[inferred]** here (I did not complete a full webpack boot); the **environment resolution** result is **[observed]** below.

**Command:**

```
env -u NODE_ENV -u CALYPSO_ENV node -e "const c=require('./client/server/config/index.js'); console.log('env_id =', c('env_id')); console.log('typeof isEnabled =', typeof c.isEnabled);"
```

**Output:**

```
env_id = development
typeof isEnabled = function
```

### Rationale and citations

- **[observed]** With neither `CALYPSO_ENV` nor `NODE_ENV` set, `client/server/config/index.js:6` falls through to the literal `'development'`. The parser then loads `config/development.json`, whose `env_id` is `development` (`config/development.json:3`), so `config('env_id')` returns `development`.
- **[observed]** `typeof c.isEnabled === 'function'` confirms the exported config API is the fully-formed `ConfigApi` (the factory attaches `isEnabled` at `packages/create-calypso-config/src/index.ts:134`).
- **[inferred]** `yarn start` enforces the Node pin before anything else because `check-node-version --package` is the first command in the `start` script (`package.json:110`); this is why the runtime conflict in the Preamble matters at dev-server start.

---

## Q2 — Test vs. dev environment at boot

**Sub-question:** What does the test environment look like when it boots up compared to normal development?

**Direct answer:** Tests boot under **Jest** with **`NODE_ENV=test`** and a **base `testEnvironment: 'node'`**; jsdom is **opt-in per file** (via a `/** @jest-environment jsdom */` docblock), not a global default. Normal development boots with `NODE_ENV`/`CALYPSO_ENV` unset, so the environment resolves to **`development`**. The two boots therefore diverge at the very first step — the resolved environment — and, in the client suite, in a stack of Jest-specific overrides (module aliases, a custom resolver, asset stubs, jsdom URL, and a bespoke setup file). **[observed]** for the env divergence (Q1 = `development` vs. Q6 = `test`); **[inferred]** for the docblock convention (read from the preset/config; tech-spec §6.6.3.1).

### The base preset — `packages/calypso-jest/jest-preset.js`

Every non-E2E suite builds on this preset:

- `resolver: require.resolve( './src/module-resolver.js' )` — `packages/calypso-jest/jest-preset.js:9`
- `setupFilesAfterEnv: [ require.resolve( './src/setup.js' ) ]` — `packages/calypso-jest/jest-preset.js:10`
- `testEnvironment: 'node'` — `packages/calypso-jest/jest-preset.js:11`
- `testMatch: [ '<rootDir>/**/test/*.[jt]s?(x)', '!**/.eslintrc.*' ]` — `packages/calypso-jest/jest-preset.js:12`
- `transform` block — `packages/calypso-jest/jest-preset.js:13-16`: `'\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ]` (`:14`) and an asset transform for `gif|jpg|jpeg|png|svg|scss|sass|css` (`:15`).

**[inferred]** (read from the preset file). The base is a **node** environment; jsdom only appears when an individual test file opts in.

### The client suite — `test/client/jest.config.js`

- Spreads the base: `...base` — `test/client/jest.config.js:5`
- `rootDir: '../../client'` — `test/client/jest.config.js:6`
- `moduleNameMapper` — `test/client/jest.config.js:10-13`, most importantly `'^@automattic/calypso-config$': '<rootDir>/server/config/index.js'` (`test/client/jest.config.js:11`) — the pivot analysed in Q6.
- `testEnvironmentOptions: { url: 'https://example.com' }` — `test/client/jest.config.js:17-19` (URL at `:18`).
- `setupFiles: [ 'jest-canvas-mock' ]` — `test/client/jest.config.js:20`.
- **Overrides** `setupFilesAfterEnv` to `'<rootDir>/../test/client/setup-test-framework.js'` — `test/client/jest.config.js:21`.
- Test-only `globals: { google: {}, __i18n_text_domain__: 'default' }` — `test/client/jest.config.js:22-25`.

**[inferred]** (read from the file); the env-var side (`NODE_ENV=test`) is **[observed]** in Q3's harness output below.

### The server suite — `test/server/jest.config.js`

- Spreads the base: `...base` — `test/server/jest.config.js:5`
- `moduleNameMapper` remaps `'^@automattic/calypso-config$': 'calypso/server/config'` — `test/server/jest.config.js:10` (block `:9-12`).
- **Overrides** `setupFilesAfterEnv` to its own `'./setup-test-framework.js'` — `test/server/jest.config.js:13`. It inherits `testEnvironment: 'node'` from the base. **[inferred]**

### The integration suite — `test/integration/jest.config.js`

- Remaps `'^@automattic/calypso-config$': '<rootDir>/client/server/config/index.js'` — `test/integration/jest.config.js:3`.
- `testEnvironment: 'node'` — `test/integration/jest.config.js:7`.
- `resolver: require.resolve( '@automattic/calypso-jest/src/module-resolver.js' )` — `test/integration/jest.config.js:8`.
- It does **not** spread `...base` and has **no `setupFilesAfterEnv`** line at all — which is why the integration suite does not disable the network (see Q4). **[observed from file]**

### Key nuance (important)

Because the client and server suites **replace** `setupFilesAfterEnv`, the base `setup.js` (`packages/calypso-jest/src/setup.js`) does **not** run for them — it applies to the _other_ suites (Packages / Applications / Build Tools). The client suite supplies its own bootstrap in `test/client/setup-test-framework.js` (enumerated in Q3). **[observed from files]**

### Demonstrating the env divergence

The environment divergence is directly demonstrated by contrasting the Q1 run against the Q6/Q3 runs:

- Dev (no env set): `env_id = development` (Q1). **[observed]**
- Test (Jest / `NODE_ENV=test`): `env_id = test` (Q6) and `process.env.NODE_ENV = test` inside the harness (Q3). **[observed]**

**Rationale:** The dev server and the test harness are two different bootstraps of the _same_ config module; they differ first in the resolved `env` (`client/server/config/index.js:6`) and then, for the client suite, in the Jest-only overrides listed above.

---

## Q3 — Test-only globals / environment variables / polyfills

**Sub-question:** Run some tests and show which globals, environment variables, and polyfills only exist during test execution.

**Direct answer:** The client test harness injects a specific, enumerable set of globals/polyfills at bootstrap (in `test/client/setup-test-framework.js`), plus two Jest-config globals and one env var (`NODE_ENV=test`). None of these are provided by this bootstrap in normal dev/browser execution — in the browser they come from the real platform (or, in plain Node, are simply absent). Below is the **complete** enumeration (no elision), followed by a harness run that prints the live `typeof` of each. **[observed]** for the printed values; **[inferred]** for items enumerated only from `file:line`.

### The one test-only environment variable

- `process.env.NODE_ENV = 'test'` — set by Jest itself for the duration of a test run. Confirmed **[observed]** in the harness output below. This is what drives `config/test.json` selection (Q6) and the `config()` missing-key branch (Q7).

### Test-only globals via Jest config — `test/client/jest.config.js`

- `globals: { google: {}, __i18n_text_domain__: 'default' }` — `test/client/jest.config.js:22-25`. So `global.google` is an (empty) object and `global.__i18n_text_domain__` is the string `'default'` during tests. **[observed]** below.
- `setupFiles: [ 'jest-canvas-mock' ]` — `test/client/jest.config.js:20`. Stubs the Canvas API (`getContext`, etc.) before the framework loads. **[inferred]**

### Every item injected by `test/client/setup-test-framework.js` (complete, no elision)

- `import '@testing-library/jest-dom';` — `test/client/setup-test-framework.js:1` (adds custom DOM matchers such as `toBeInTheDocument`).
- `nock.disableNetConnect();` — `test/client/setup-test-framework.js:9` (blocks all real network; see Q4). Imported at `:6`.
- `beforeAll(() => { if ( ! nock.isActive() ) { nock.activate(); } })` — `test/client/setup-test-framework.js:11-16` (reactivates nock at test start).
- `afterAll(() => { nock.restore(); nock.cleanAll(); })` — `test/client/setup-test-framework.js:18-22` (cleanup; avoids memory leaks — see Q4).
- `global.TextEncoder = TextEncoder;` and `global.TextDecoder = TextDecoder;` — `test/client/setup-test-framework.js:25-26` (from `util`, imported at `:5`; needed by `ReactDOMServer`).
- `global.CSS = { supports: jest.fn() };` — `test/client/setup-test-framework.js:30-32` (jsdom/CSSDOM has no `CSS.supports`).
- `global.ResizeObserver = require( 'resize-observer-polyfill' );` — `test/client/setup-test-framework.js:34`.
- `global.fetch = jest.fn( () => Promise.resolve( { json: () => Promise.resolve() } ) );` — `test/client/setup-test-framework.js:36-40` (a stub that resolves to empty JSON).
- `jest.mock( 'wpcom-proxy-request', () => ( { __esModule: true, canAccessWpcomApis: jest.fn(), reloadProxy: jest.fn(), requestAllBlogsAccess: jest.fn() } ) );` — `test/client/setup-test-framework.js:44-49` (module mock, because it touches the `document` global).
- `global.crypto.randomUUID = () => nodeCrypto.randomUUID();` — `test/client/setup-test-framework.js:52` (Node crypto imported at `:3`).
- `global.matchMedia = jest.fn( ( query ) => ( { matches: false, media: query, onchange: null, addListener: jest.fn(), removeListener: jest.fn(), addEventListener: jest.fn(), removeEventListener: jest.fn(), dispatchEvent: jest.fn() } ) );` — `test/client/setup-test-framework.js:54-63`.
- `global.ReadableStream = ReadableStream;` — `test/client/setup-test-framework.js:66` (imported from `node:stream/web` at `:4`; used by `@wp-playground/client`).
- `global.TransformStream = TransformStream;` — `test/client/setup-test-framework.js:67` (also from `node:stream/web`, `:4`).
- `global.Worker = require( 'worker_threads' ).Worker;` — `test/client/setup-test-framework.js:68`.
- `if ( typeof global.structuredClone !== 'function' ) { global.structuredClone = ( obj ) => JSON.parse( JSON.stringify( obj ) ); }` — `test/client/setup-test-framework.js:71-73` (JSON-fallback, guarded).
- `if ( ! global.crypto.subtle ) { global.crypto.subtle = nodeCrypto.subtle; }` — `test/client/setup-test-framework.js:76-79` (guarded Node WebCrypto).

### The base stub (for the non-client suites) — `packages/calypso-jest/src/setup.js`

- `global.CSS = { supports: jest.fn() };` — `packages/calypso-jest/src/setup.js:3-5`. This applies to suites that do **not** override `setupFilesAfterEnv` (Packages / Applications / Build Tools). The client suite overrides it and re-declares `global.CSS` itself. **[inferred]**

### Module-resolution & asset conventions that exist only under Jest

- Custom resolver `packages/calypso-jest/src/module-resolver.js:16-20` — `enhancedResolve.create.sync({ … })` with `mainFields: [ 'calypso:src', 'main' ]` (`:18`) and `conditionNames: [ 'calypso:src', 'node', 'require' ]` (`:19`). This resolves monorepo packages to their **untranspiled `calypso:src`** source (so tests run without pre-built `dist`). **[inferred]**
- Asset transform `packages/calypso-jest/src/asset-transform.js:4-6` — `process()` returns `{ code: 'module.exports = ' + JSON.stringify( path.basename( filename ) ) + ';' }`, i.e. every imported image/style becomes the string of its basename. **[inferred]**

### Harness run that prints the live values

To prove these are actually present at runtime (not just in source), I created a throwaway spec that matches the client `testMatch` pattern, ran it through the client Jest config, captured the output, and deleted the spec immediately (via an `EXIT` trap) so the repository was left unchanged (confirmed with `git status --porcelain` — empty).

**Command:**

```
node .yarn/releases/yarn-4.0.2.cjs jest -c=test/client/jest.config.js client/state/country-states/test/blitzy_adhoc_test_globals.js
```

The temporary spec logged `typeof`/values for each injected item. **Output (complete, unedited):**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/country-states/test/blitzy_adhoc_test_globals.js
  ● Console

    console.log
      process.env.NODE_ENV               = test

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:2:10)

    console.log
      global.__i18n_text_domain__        = default

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:3:10)

    console.log
      typeof global.google               = object

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:4:10)

    console.log
      typeof global.CSS.supports         = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:5:10)

    console.log
      typeof global.ResizeObserver       = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:6:10)

    console.log
      typeof global.fetch                = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:7:10)

    console.log
      typeof global.matchMedia           = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:8:10)

    console.log
      typeof global.TextEncoder          = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:9:10)

    console.log
      typeof global.TextDecoder          = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:10:10)

    console.log
      typeof global.ReadableStream       = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:11:10)

    console.log
      typeof global.TransformStream      = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:12:10)

    console.log
      typeof global.Worker               = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:13:10)

    console.log
      typeof global.structuredClone      = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:14:10)

    console.log
      typeof global.crypto.randomUUID    = function

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:15:10)

    console.log
      typeof global.crypto.subtle        = object

      at Object.log (state/country-states/test/blitzy_adhoc_test_globals.js:16:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.771 s
Ran all test suites matching /client\/state\/country-states\/test\/blitzy_adhoc_test_globals.js/i.
```

### Rationale and coverage

- **[observed]** `process.env.NODE_ENV = test` — the sole test-only env var, confirmed live.
- **[observed]** `global.__i18n_text_domain__ = default` and `typeof global.google = object` — the two Jest-config globals (`test/client/jest.config.js:22-25`).
- **[observed]** `typeof global.CSS.supports = function` (`:30-32`), `ResizeObserver = function` (`:34`), `fetch = function` (`:36-40`), `matchMedia = function` (`:54-63`), `TextEncoder`/`TextDecoder = function` (`:25-26`), `ReadableStream`/`TransformStream`/`Worker = function` (`:66`/`:67`/`:68`), `structuredClone = function` (`:71-73`), `crypto.randomUUID = function` (`:52`), `crypto.subtle = object` (`:76-79`).
- **[inferred contrast]** In normal dev/browser these are provided by the real browser platform (e.g. `fetch`, `matchMedia`, `ResizeObserver`, `CSS.supports`, `crypto.subtle`) or, in plain Node without this bootstrap, are absent or unstubbed. The harness exists precisely to provide/stub them so browser-oriented code can run under Jest.

---

## Q4 — Network in tests (primary path + edge path)

**Sub-question:** When code tries to make a network request during tests, what actually happens?

**Direct answer:** In the client suite, all real network is **blocked at bootstrap** by `nock.disableNetConnect()` (`test/client/setup-test-framework.js:9`), and `global.fetch` is replaced by a stub that resolves to empty JSON (`test/client/setup-test-framework.js:36-40`). If code issues a request that no `nock` interceptor matches, the request **throws** a `nock` `NetConnectNotAllowedError` (code `ENETUNREACH`). The integration suite, by contrast, allows real network because it never loads the client bootstrap. **[observed]**

### Edge path — trigger an un-mocked request and capture the exact error

I reproduced the client bootstrap's network policy (`nock.disableNetConnect()`) and then issued an **un-mocked** HTTP request to the WordPress.com API host, capturing the raw error the request emitted.

**Command:**

```
node -e "const nock=require('nock'); nock.disableNetConnect(); const http=require('http'); const req=http.get('http://public-api.wordpress.com:80/rest/v1.1/me', r=>{console.log('UNEXPECTED', r.statusCode);process.exit(0);}); req.on('error', e=>{console.log('error.name    =', e.name); console.log('error.code    =', e.code); console.log('error.message =', e.message); process.exit(0);}); setTimeout(()=>{console.log('NO ERROR (timeout)');process.exit(1);},5000);"
```

**Output (byte-for-byte):**

```
error.name    = NetConnectNotAllowedError
error.code    = ENETUNREACH
error.message = Nock: Disallowed net connect for "public-api.wordpress.com:80/rest/v1.1/me"
```

**[observed]** The error name is `NetConnectNotAllowedError`, the code is `ENETUNREACH`, and the message is exactly `Nock: Disallowed net connect for "public-api.wordpress.com:80/rest/v1.1/me"`. This is what happens to any request that lacks a matching interceptor once `disableNetConnect()` is in effect — the request never leaves the process.

### Primary path (the common case)

In everyday tests, code that would hit the network is either (a) intercepted by a `nock` mock (see Q5), so it resolves against the mocked body, or (b) routed through the stubbed `global.fetch` (`test/client/setup-test-framework.js:36-40`), which resolves to `{ json: () => Promise.resolve() }` (empty JSON). Either way, no real socket is opened. **[observed for the block/stub via file:line + the edge run above]**

### Contrast — the integration suite allows network

The integration suite allows real network specifically because `test/integration/jest.config.js` has **no `setupFilesAfterEnv`** entry and does **not** spread the base preset — so it never loads `test/client/setup-test-framework.js`, and therefore `nock.disableNetConnect()` is never applied there. Its environment is plain `testEnvironment: 'node'` (`test/integration/jest.config.js:7`). **[observed from file]**

### Cleanup semantics

At the end of a client run, `afterAll` calls `nock.restore()` then `nock.cleanAll()` (`test/client/setup-test-framework.js:18-22`). `nock` overrides Node's `http.request`/`http.ClientRequest`; calling `nock.restore()` after suites returns those to normal and avoids Jest module-cache memory growth across runs. **[inferred from nock docs + observed file:line]**

**Rationale:** Network isolation is a deliberate default of the client harness — it guarantees deterministic tests and forces every outbound call to be explicitly mocked. The un-mocked case is intentionally loud (a thrown `NetConnectNotAllowedError`) so an unmocked call cannot silently pass.

---

## Q5 — Mocked API trace (dispatch sequence: before → during → after)

**Sub-question:** Show a test that mocks an API call and trace how the mocked response flows through the action creator back to the test assertion.

**Direct answer:** The `country-states` state module's test (`client/state/country-states/test/actions.js`) mocks the WordPress.com REST endpoint with `nock`, then invokes the `requestCountryStates` thunk (`client/state/country-states/actions.js:21-46`). The thunk dispatches `COUNTRY_STATES_REQUEST` **before** the call, awaits `wpcom.req.get(...)` (whose HTTP is intercepted by `nock`), and **after** the mocked reply dispatches `COUNTRY_STATES_RECEIVE` + `COUNTRY_STATES_REQUEST_SUCCESS` (happy path) or `COUNTRY_STATES_REQUEST_FAILURE` (the `/ca` 500 edge). The test asserts against a `jest.fn()` dispatch spy. The suite passes **5/5**, stable across two runs. **[observed]**

### Run it (twice, for stability)

**Command:**

```
node .yarn/releases/yarn-4.0.2.cjs jest -c=test/client/jest.config.js client/state/country-states/test/actions.js
```

**Output — RUN 1 (complete, unedited, including the benign Browserslist warning):**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/country-states/test/actions.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   0 total
Time:        0.945 s, estimated 1 s
Ran all test suites matching /client\/state\/country-states\/test\/actions.js/i.
```

**Output — RUN 2 (complete, unedited):**

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/state/country-states/test/actions.js

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   0 total
Time:        0.957 s, estimated 1 s
Ran all test suites matching /client\/state\/country-states\/test\/actions.js/i.
```

**[observed]** Both runs report `Tests: 5 passed, 5 total` and exit `0` — **stable 5/5** across two runs. The wall-clock `Time` differs run-to-run (`0.945 s` vs `0.957 s`), which is expected; the pass count is the stable anchor. The suite has **5 tests**: 1 in `#receiveCountryStates()` (`client/state/country-states/test/actions.js:17-36`) and 4 in `#requestCountryStates()` (`:38-94`). (The Browserslist "17 months old" line is a benign upstream caniuse-lite freshness warning and is included here unedited.)

### End-to-end trace (with before / during / after state)

1. **Interception is set up in `beforeAll`** via the `useNock` helper — `client/state/country-states/test/actions.js:39-52`. Two interceptors are registered on `nock( 'https://public-api.wordpress.com:443' ).persist()`:
   - Happy path: `.get( '/rest/v1.1/domains/supported-states/us' ).reply( 200, [ { code: 'AK', name: 'Alaska' }, { code: 'AS', name: 'American Samoa' } ] )` — `:40-46`.
   - **Edge case:** `.get( '/rest/v1.1/domains/supported-states/ca' ).reply( 500, { error: 'server_error', message: 'A server error occurred' } )` — `:47-51`.
2. **The thunk** `requestCountryStates( countryCode )` — `client/state/country-states/actions.js:21-46`.
3. **BEFORE the request** the thunk dispatches the request action: `dispatch( { type: COUNTRY_STATES_REQUEST, countryCode } )` — `client/state/country-states/actions.js:25-28`. (Asserted by the first test, `client/state/country-states/test/actions.js:54-61`.)
4. **DURING** it calls `wpcom.req.get()` with the template-literal path `/domains/supported-states/${ countryCode }` — `client/state/country-states/actions.js:30-31`. The transport is the `wpcom` client built in `client/lib/wp/browser.js:14-35`; the canonical (default) branch is `wpcom = new WPCOM( wpcomProxyRequest )` (`client/lib/wp/browser.js:21`), and the module `export default wpcom` at `client/lib/wp/browser.js:56` (type surface `client/lib/wp/index.d.ts:1-9`). WPCOM prepends `/rest/v1.1` to the path, so the outgoing request path is `/rest/v1.1/domains/supported-states/us`, which matches the `nock` interceptor; `nock` returns the mocked `200` body. **[observed test passes + inferred path-prefixing]**
5. **AFTER (success)** the `.then` handler dispatches the receive action then the success action — `client/state/country-states/actions.js:32-38`: first `dispatch( receiveCountryStates( countryStates, countryCode ) )` (`:33`; the action object `{ type: COUNTRY_STATES_RECEIVE, countryCode, countryStates }` comes from `client/state/country-states/actions.js:11-19`), then `dispatch( { type: COUNTRY_STATES_REQUEST_SUCCESS, countryCode } )` (`:34-37`). (Asserted by `client/state/country-states/test/actions.js:63-74` and `:76-83`.)
6. **AFTER (failure / `ca` edge)** the `.catch` handler dispatches the failure action — `client/state/country-states/actions.js:39-45`: `dispatch( { type: COUNTRY_STATES_REQUEST_FAILURE, countryCode, error } )`. (Asserted by `client/state/country-states/test/actions.js:85-93`, which checks `error: expect.objectContaining( { message: 'A server error occurred' } )` at `:90`.)
7. **Assertions** run against the `jest.fn()` spy created in `beforeEach` (`client/state/country-states/test/actions.js:13-15`); each test invokes the thunk as `requestCountryStates( 'us' )( spy )` (or `'ca'`) and asserts the spy was called with the expected action object.

### The `useNock` helper

The example uses the (deprecated) helper `client/test-helpers/use-nock/index.js`: it is marked `@deprecated` at `client/test-helpers/use-nock/index.js:10` (the JSDoc says to use `nock` directly), runs the `setupCallback` inside `beforeAll` at `:14`, and cleans up in `afterAll` via `nock.cleanAll()` at `:16-19`. It is still used by this example test. **[observed]**

### State transitions, explicitly

- **Happy path (`us`):** `COUNTRY_STATES_REQUEST` (before) → [mocked HTTP 200] → `COUNTRY_STATES_RECEIVE` then `COUNTRY_STATES_REQUEST_SUCCESS` (after).
- **Edge path (`ca`):** `COUNTRY_STATES_REQUEST` (before) → [mocked HTTP 500] → `COUNTRY_STATES_REQUEST_FAILURE` (after), carrying an `error` whose `message` is `A server error occurred`.

**Rationale:** The mocked body never travels over a real socket; `nock` satisfies `wpcom`'s underlying HTTP request in-process, so the thunk's promise resolves (or rejects) deterministically and the dispatched action sequence is fully predictable — which is exactly what the assertions pin down.

---

## Q6 — Config / feature-flag resolution divergence (tests vs. development)

**Sub-question:** How does configuration like feature flags get resolved differently in tests vs. development?

**Direct answer:** In client tests, a Jest `moduleNameMapper` alias remaps `@automattic/calypso-config` to the **server** config module, so config is read from environment JSON files (`config/test.json`) through the server resolver — **bypassing** the browser module's `window.configData` path. Because Jest forces `NODE_ENV=test`, the loaded environment is `test`, whereas the dev server loads `development`. The same feature flags therefore resolve to **different values** in the two environments. **[observed]**

### The pivot — the `moduleNameMapper` alias

In client tests the alias `'^@automattic/calypso-config$': '<rootDir>/server/config/index.js'` (`test/client/jest.config.js:11`) means every `import config from '@automattic/calypso-config'` in application code actually loads `client/server/config/index.js` during tests. That server module reads environment JSON via `client/server/config/parser.js`, and it **bypasses** the real browser module `packages/calypso-config/src/index.ts`, whose read path is `window.configData` (`packages/calypso-config/src/index.ts:21-47`) and which even throws when there is no `window` (`packages/calypso-config/src/index.ts:17-19`). That throw is one concrete reason tests must alias the module away — a `node` test environment has no `window`. **[observed]**

### Parser behavior — `client/server/config/parser.js`

- The parser is `module.exports = function ( configPath, defaultOpts ) { … }` at `client/server/config/parser.js:23`; it reads `opts.env`, `opts.enabledFeatures`, and `opts.disabledFeatures` from the passed options. (The default `env` is `'development'` when unspecified — `:24-29`.)
- It merges three files in order — `_shared.json` → `{env}.json` → `{env}.local.json` (`client/server/config/parser.js:31-35`) — using `assignWith` so the `features` object is **merged** while other keys are simple-assigned (`:42-47`).
- It applies `ENABLE_FEATURES`/`DISABLE_FEATURES` overrides onto `data.features` (`client/server/config/parser.js:49-58`).
- It applies `PROTOCOL`/`HOST`/`PORT` env overrides onto `protocol`/`hostname`/`port` (`client/server/config/parser.js:60-63`).
- It returns `{ serverData, clientData }` (`client/server/config/parser.js:87`), so the resolved config is a `{ serverData, clientData }` pair — inspect `serverData.env_id` and `serverData.features[...]`, not a flat object.
- Note: `config/secrets.json` is absent in this checkout, so the parser falls back to `config/empty-secrets.json` (`client/server/config/parser.js:36-38`). **[inferred from file:line; confirmed indirectly by the successful Q6 runs]**

### Run the parser under both environments

**Commands:**

```
NODE_ENV=test node -e "const p=require('./client/server/config/parser.js'); const {serverData}=p(require('path').resolve('config'),{env:process.env.NODE_ENV}); console.log(JSON.stringify({env_id:serverData.env_id,'checkout/checkout-version':serverData.features['checkout/checkout-version'],'google-my-business':serverData.features['google-my-business'],'individual-subscriber-stats':serverData.features['individual-subscriber-stats']}));"
```

```
NODE_ENV=development node -e "const p=require('./client/server/config/parser.js'); const {serverData}=p(require('path').resolve('config'),{env:process.env.NODE_ENV}); console.log(JSON.stringify({env_id:serverData.env_id,'checkout/checkout-version':serverData.features['checkout/checkout-version'],'google-my-business':serverData.features['google-my-business'],'individual-subscriber-stats':serverData.features['individual-subscriber-stats']}));"
```

**Output — `NODE_ENV=test`:**

```
{"env_id":"test","checkout/checkout-version":false,"google-my-business":false,"individual-subscriber-stats":false}
```

**Output — `NODE_ENV=development`:**

```
{"env_id":"development","checkout/checkout-version":true,"google-my-business":true,"individual-subscriber-stats":true}
```

### Anchor diff and citations

- **[observed]** `env_id` is `test` under `NODE_ENV=test` and `development` under `NODE_ENV=development`. Source: `config/test.json:3` (`"env_id": "test"`) vs. `config/development.json:3` (`"env_id": "development"`). Note that **both** files set `"env": "development"` (`config/test.json:2`, `config/development.json:2`) — so the **distinguishing** key is `env_id`, not `env`.
- **[observed]** Three example flags flip between the environments:
  - `checkout/checkout-version` — `false` at `config/test.json:33` vs. `true` at `config/development.json:44`.
  - `google-my-business` — `false` at `config/test.json:47` vs. `true` at `config/development.json:67`.
  - `individual-subscriber-stats` — `false` at `config/test.json:52` vs. `true` at `config/development.json:80`.
- **[inferred]** The base `config/_shared.json` (merged first into every environment) sets `"env_id": "shared"` (`config/_shared.json:3`) and `"features": {}` (`config/_shared.json:10`); the per-env file then supplies the real flags.
- **[inferred]** `config/client.json` is the allow-list of config keys exposed to the browser (it lists `env`, `env_id`, and `features`, among others — `config/client.json:5`, `:6`, `:8`).
- **[observed via file:line]** README-vs-resolver discrepancy: `config/README.md:3` says selection is by `NODE_ENV` only, but the resolver at `client/server/config/index.js:6` is `CALYPSO_ENV || NODE_ENV || 'development'`.

**Rationale:** Feature flags are plain JSON per environment, merged `_shared → {env} → {env}.local`. Because tests run with `env = test` and dev with `env = development`, the merge produces different `features` maps, and any flag whose value differs between `config/test.json` and `config/development.json` resolves differently. The alias (`test/client/jest.config.js:11`) is what makes tests read this server-side JSON path at all instead of the browser's `window.configData`.

---

## Q7 — How tests control what config returns (with proof)

**Sub-question:** How do tests control what values config returns, and show proof that a test actually uses a different value than the dev server would resolve.

**Direct answer:** Tests control config primarily by the environment Jest runs in: Jest forces `NODE_ENV=test`, and the aliased server resolver (`client/server/config/index.js`, via the Q6 `moduleNameMapper`) loads `config/test.json`. A test therefore gets the `test` environment's values; it can further override individual flags with `enable()`/`disable()` or with the `ACTIVE_FEATURE_FLAGS`/`ENABLE_FEATURES`/`DISABLE_FEATURES` env variables. The proof that a test resolves a _different_ value than the dev server is the same-flag diff from Q6 (e.g. `google-my-business` = `false` in test vs. `true` in dev), plus the missing-key behavior that differs by environment. **[observed]**

### `config()` / `isEnabled()` semantics — `packages/create-calypso-config/src/index.ts`

- `config( key )` returns `data[ key ]` when the key is present — `packages/create-calypso-config/src/index.ts:31-33`.
- **Missing-key edge:** it throws a `ReferenceError` **only** when `process.env.NODE_ENV === 'development'` — `packages/create-calypso-config/src/index.ts:35-40`; otherwise it returns `undefined` (logging a browser-only `console.error` guarded by `typeof window !== 'undefined'` — `:44-59`; `return undefined` at `:61`).
- `isEnabled( feature )` first honors the `ACTIVE_FEATURE_FLAGS` env override (`packages/create-calypso-config/src/index.ts:72-83`), then falls back to `data.features[ feature ]` (`:85`).
- Tests may also mutate flags at runtime through `enable( feature )` (`packages/create-calypso-config/src/index.ts:107-111`) and `disable( feature )` (`:118-122`), both attached to the API by the default factory (`:132-140`).

### PROOF #1 — same flag, two environments (from the Q6 runs)

`google-my-business` resolves **`false`** under `NODE_ENV=test` (`config/test.json:47`) but **`true`** under `NODE_ENV=development` (`config/development.json:67`). Side by side (captured outputs from Q6):

```
NODE_ENV=test        → {"env_id":"test","checkout/checkout-version":false,"google-my-business":false,"individual-subscriber-stats":false}
NODE_ENV=development → {"env_id":"development","checkout/checkout-version":true,"google-my-business":true,"individual-subscriber-stats":true}
```

**[observed]** A test running under `NODE_ENV=test` sees `google-my-business = false`, whereas the dev server (resolving `development`) sees `google-my-business = true`. Same flag, different resolved value — that is the concrete proof.

### PROOF #2 — missing-key behavior differs by environment (edge path)

Requesting a key that does not exist behaves differently depending on `NODE_ENV`, exercising the branch at `packages/create-calypso-config/src/index.ts:35-40`.

**Commands:**

```
NODE_ENV=development node -e "const c=require('./client/server/config/index.js'); try{ c('this_key_does_not_exist'); console.log('NO THROW'); }catch(e){ console.log(e.constructor.name+': '+String(e.message).split(String.fromCharCode(10))[0]); }"
```

```
NODE_ENV=test node -e "const c=require('./client/server/config/index.js'); console.log('returns:', String(c('this_key_does_not_exist')));"
```

**Output:**

```
NODE_ENV=development → ReferenceError: Could not find config value for key 'this_key_does_not_exist'
NODE_ENV=test        → returns: undefined
```

**[observed]** Under `development` a missing key throws `ReferenceError: Could not find config value for key 'this_key_does_not_exist'` (crash-early behavior, `:35-40`); under `test` the same call returns `undefined` (`:61`). This is a second, orthogonal way the test environment's config behavior diverges from the dev server's.

### The bypassed browser reader (for completeness)

The path the **dev server/browser** actually uses is `packages/calypso-config/src/index.ts`: it reads `window.configData` (`:21-47`) and calls `createConfig( configData )` (`:111`), re-exporting `isEnabled`/`enabledFeatures`/`enable`/`disable` (`:113-116`). Tests **bypass** this entirely via the Q6 alias, which is why they can control config through `NODE_ENV` + environment JSON instead of a live `window`. **[observed via file:line]**

**Rationale:** "Control" in tests is mostly declarative — pick the environment (`test`) and let the JSON decide — with imperative escape hatches (`enable`/`disable`, `ACTIVE_FEATURE_FLAGS`, `ENABLE_FEATURES`/`DISABLE_FEATURES`) when a test needs a specific flag regardless of environment. The two proofs show a value-level divergence (`google-my-business`) and a behavior-level divergence (missing-key throw vs. `undefined`).

---

## Closing — Coverage & Citations

### Coverage pass — every question and every named item

| Question                               | Addressed? | Key evidence                                                                                                                                                                                         |
| -------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Q1** Dev server bring-up             | ✅         | `env_id = development`, `typeof isEnabled = function`; env resolver `client/server/config/index.js:6`; `start` guard `package.json:110`                                                              |
| **Q2** Test vs. dev env at boot        | ✅         | base `testEnvironment: 'node'` (`packages/calypso-jest/jest-preset.js:11`); client overrides `setupFilesAfterEnv` (`test/client/jest.config.js:21`); env divergence dev=`development` vs test=`test` |
| **Q3** Test-only globals/env/polyfills | ✅         | full enumeration of `test/client/setup-test-framework.js` items + config globals (`test/client/jest.config.js:22-25`) + `NODE_ENV=test`; live `typeof` values captured from the harness              |
| **Q4** Network in tests                | ✅         | `nock.disableNetConnect()` (`:9`) + `fetch` stub (`:36-40`); un-mocked edge → `NetConnectNotAllowedError`/`ENETUNREACH`; integration contrast (no `setupFilesAfterEnv`); cleanup `:18-22`            |
| **Q5** Mocked API trace                | ✅         | `country-states` suite PASS 5/5 (x2); dispatch sequence REQUEST → RECEIVE+SUCCESS (us) / REQUEST → FAILURE (ca 500 edge); `useNock` helper                                                           |
| **Q6** Config/flag divergence          | ✅         | alias pivot (`test/client/jest.config.js:11`); parser `{serverData,clientData}`; `env_id` test/development + 3 flag diffs captured                                                                   |
| **Q7** Tests control config + proof    | ✅         | `config()`/`isEnabled()`/`enable()`/`disable()` semantics; PROOF #1 `google-my-business` false(test)/true(dev); PROOF #2 missing-key `ReferenceError`(dev)/`undefined`(test)                         |

Named-item checklist explicitly covered: **dev server** (Q1); **test-vs-dev boot** (Q2); the full enumerated list of **test-only globals/env/polyfills** — `NODE_ENV`, `google`, `__i18n_text_domain__`, `CSS.supports`, `ResizeObserver`, `fetch`, `matchMedia`, `TextEncoder`/`TextDecoder`, `ReadableStream`/`TransformStream`/`Worker`, `structuredClone`, `crypto.randomUUID`, `crypto.subtle`, `jest-canvas-mock`, `@testing-library/jest-dom`, the `wpcom-proxy-request` mock, the `calypso:src` resolver, and the asset transform (Q3); **un-mocked network + integration contrast** (Q4); the **mocked-API dispatch sequence including the `/ca` 500 failure edge** (Q5); the **config divergence pivot** (Q6); **config control + the missing-key `ReferenceError` edge + the flag proof** (Q7). Both documented conflicts (runtime; config-selection docs) are in the Preamble.

### Observed deviations from the scoping ground-truth (reported honestly)

- **Node version:** observed `v22.23.1` (an earlier scoping pass noted `v22.22.2`). Both satisfy `^v22.9.0`; I reported the actual observed value.
- **Q5 timing:** `Time` varies run-to-run (`0.945 s` / `0.957 s`); the stable anchor is the pass count `5 passed, 5 total`, which matched on both runs.
- **Parser signature:** the source is `module.exports = function ( configPath, defaultOpts )` (`client/server/config/parser.js:23`), reading `opts.env`/`opts.enabledFeatures`/`opts.disabledFeatures` — reported as-is rather than as a destructured paraphrase.
- **`config/client.json`:** the file is 33 lines listing 31 client-exposed keys; described accurately here.

All other anchor strings matched the scoping ground-truth verbatim (the `NetConnectNotAllowedError`/`ENETUNREACH`/`Nock: Disallowed net connect for "public-api.wordpress.com:80/rest/v1.1/me"` message; `env_id` `test`/`development`; the three flag values; `5 passed, 5 total`; `ReferenceError`/`undefined`).

### Files cited in this document

Manifests & runtime: `package.json`, `.nvmrc`, `.yarnrc.yml`.
Jest harness: `packages/calypso-jest/jest-preset.js`, `packages/calypso-jest/src/setup.js`, `packages/calypso-jest/src/module-resolver.js`, `packages/calypso-jest/src/asset-transform.js`, `test/client/jest.config.js`, `test/client/setup-test-framework.js`, `test/server/jest.config.js`, `test/integration/jest.config.js`.
Configuration: `client/server/config/index.js`, `client/server/config/parser.js`, `packages/create-calypso-config/src/index.ts`, `packages/calypso-config/src/index.ts`, `config/_shared.json`, `config/development.json`, `config/test.json`, `config/client.json`, `config/README.md`.
Data layer / transport: `client/state/country-states/actions.js`, `client/state/country-states/test/actions.js`, `client/lib/wp/browser.js`, `client/lib/wp/index.d.ts`, `client/test-helpers/use-nock/index.js`.

### Read-only outcome

Only this one file — `blitzy/documentation/wp-calypso_be7e5cc64162.md` — was added. The single temporary observation spec used in Q3 was created outside version control's tracked set and deleted immediately (its Jest transform-cache artifacts were removed too), and all other observations used `node -e` one-liners that write nothing. `git status --porcelain` confirms the tracked source tree is unchanged apart from this document; `node_modules/` and `.cache/` are gitignored. The repository is left byte-for-byte unchanged except for this answer document.
