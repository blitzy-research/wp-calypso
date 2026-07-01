# wp-calypso Testing Infrastructure — Onboarding Walkthrough

> An evidence-based walkthrough of Calypso's test harness for a developer onboarding onto the
> repository. Every answer below is grounded in output **actually observed** by booting the dev
> server and running Jest — not derived from reading source alone.

---

## Methodology & Environment

This document was produced by **running the code first**, then writing each answer beside the
output it produced. The investigation was strictly **read-only**: no existing repository file was
modified, and every temporary observation script used to capture evidence was deleted afterward, so
`git status` shows only this new document.

Evidence was captured by:

1. **`yarn install`** — non-mutating; it only materializes the git-ignored `node_modules` (absent
   from the delivered checkout) and leaves `yarn.lock` and every manifest unchanged.
2. **Building and booting the dev server** — to confirm it serves (R1).
3. **Running Jest suites plus one temporary observation spec** — to capture the test environment,
   globals, env vars, polyfills, the network-lockdown behavior, the mocked-API trace, and the
   config-resolution proof (R2–R7). All temp scripts were removed when done.

### Environment literals

The runtime is pinned by the repository:

- Node is pinned to `22.9.0` — the entire `.nvmrc` file is the single line `22.9.0`.
- `package.json` declares the supported engines:

  ```jsonc
  // package.json
  "node": "^v22.9.0",   // [package.json:L57]
  "yarn": "^4.0.0",     // [package.json:L58]
  ```

- **Observed** Node during evidence capture was `v22.22.2`, which satisfies `^v22.9.0`. (Reported
  as observed — not an assumed value.)
- Yarn `4.0.2` is used via corepack:

  ```jsonc
  // package.json
  "packageManager": "yarn@4.0.2"   // [package.json:L422]
  ```

  ```yaml
  # .yarnrc.yml
  nodeLinker: node-modules                        # [.yarnrc.yml:L3]
  yarnPath: .yarn/releases/yarn-4.0.2.cjs         # [.yarnrc.yml:L5]
  ```

- The root `node_modules` was **absent** in the delivered checkout, so `yarn install` was run first
  before either the dev server or Jest could execute.

---

## R1 — "Start the development server to confirm it works"

**Claim: the dev server is started by the `start` script, which gates the Node version, prints a
welcome banner, builds, then serves.**

```jsonc
// package.json
"build": "...",                                                                 // [package.json:L64]
"start": "npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build",  // [package.json:L110]
"start-build": "BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short"                            // [package.json:L113]
```

The chain is: version gate → welcome banner → webpack build → serve the built server through
`bunyan` log formatting.

**Claim: the Node-version gate passes on the observed runtime.**

```console
$ node_modules/.bin/check-node-version --package
$ echo "exit=$?"
exit=0
```

Exit code `0` — Node `v22.22.2` satisfies `engines.node = "^v22.9.0"` [package.json:L57].

**Claim: `bin/welcome.js` prints the Calypso ASCII banner.**

```console
$ node bin/welcome.js
```

A cyan `calypso` ASCII-art banner is printed (rendered via `chalk.cyan(...)` in `bin/welcome.js`).

**Claim: `build/server.js` is a generated artifact — absent from source, produced by the webpack
build — and is git-ignored (so running it is not a repository change).**

```console
$ BROWSERSLIST_ENV=evergreen yarn run build-server
$ ls -l build/server.js
-rw-r--r-- ... 7.9M ... build/server.js
$ git check-ignore build/server.js
build/server.js
```

`build/server.js` exists at ~7.9 MB after the build, and `git check-ignore` confirms it is ignored.

**Claim: the server boots and reports its URL.** The verbatim boot log (piped through `bunyan -o short`):

```console
$ BROWSERSLIST_ENV=evergreen node build/server.js | node_modules/.bin/bunyan -o short
Failed to load ./.env.
20:36:23.825Z  INFO calypso: wp-calypso booted in 996ms - http://calypso.localhost:3000
Compiling assets... Wait until you see Ready! and then try http://calypso.localhost:3000/ again.
```

Benign warnings were also observed and are reported as-is: a `punycode` `DEP0040` deprecation
warning, and repeated Browserslist "caniuse-lite is 17 months old" warnings. Neither affects
serving.

**Claim: the server serves HTTP 200 at the default dev URL `http://calypso.localhost:3000`** (the
default dev URL per `config/README.md`).

```console
$ curl -sS -D - http://calypso.localhost:3000/
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 630
Connection: keep-alive
```

A direct probe against the loopback address confirms the same:

```console
HTTP_STATUS=200 SIZE=630 TIME=0.035142s
```

**Claim: the 630-byte body is Calypso's dev "waiting" page** (served while webpack compiles the
client on the fly).

```console
$ curl -s http://calypso.localhost:3000/ | grep -o '<h1>[^<]*</h1>'
<h1>Welcome to Calypso!</h1>
```

The body instructs the visitor to wait until webpack prints `READY!` before the page auto-refreshes
into the real application.

**Reasoning.** `yarn start` runs `check-node-version --package` (which fails fast on an unsupported
Node), prints the banner via `bin/welcome.js`, runs the webpack build to emit `build/server.js`,
then serves it through Express behind `bunyan` log formatting [package.json:L110, L113]. On the
first load it returns a lightweight 630-byte "waiting" page until the client assets finish
compiling. **Conclusion: the dev server works — it boots in ~1 s (`booted in 996ms`) and returns
HTTP 200.**

---

## R2 — "What does the test environment look like when it boots up compared to normal development?"

**Claim: there is NO top-level Jest *configuration* block in `package.json`; Jest is configured
per-suite.** (Precisely: `package.json` *does* contain a `"jest": "^29.7.0"` **devDependency** — a
version string, not a Jest config key.)

```jsonc
// package.json
"jest": "^29.7.0",   // [package.json:L290]  <-- devDependency version, NOT a config key
```

The suites are driven by explicit `-c=` config paths:

```jsonc
// package.json
"test": "run-s -s test-client test-packages test-server test-build-tools",   // [package.json:L120]
"test-client": "TZ=UTC jest -c=test/client/jest.config.js"                    // [package.json:L122]
```

**Claim: the harness is split into seven Jest "projects," each with its own config.**

```console
$ ls -1 test/apps/jest.config.js test/build-tools/jest.config.js test/client/jest.config.js \
        test/e2e/jest.config.js test/integration/jest.config.js test/packages/jest.config.js \
        test/server/jest.config.js
test/apps/jest.config.js
test/build-tools/jest.config.js
test/client/jest.config.js
test/e2e/jest.config.js
test/integration/jest.config.js
test/packages/jest.config.js
test/server/jest.config.js
```

**Claim: the base test environment is `node`, not a browser.**

```js
// packages/calypso-jest/jest-preset.js
testEnvironment: 'node',                              // [packages/calypso-jest/jest-preset.js:L11]
testMatch: [ '<rootDir>/**/test/*.[jt]s?(x)', ... ],  // [packages/calypso-jest/jest-preset.js:L12]
```

```js
// test/client/jest.config.js
rootDir: '../../client',                              // [test/client/jest.config.js:L6]
testEnvironmentOptions: { url: 'https://example.com' } // [test/client/jest.config.js:L18]
```

A temporary observation spec run under the client Jest config confirms there is no DOM by default:

```console
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --runTestsByPath <temp-spec> --verbose
typeof window = undefined
```

**Claim: DOM suites opt into `jsdom` per file via a docblock, and this is a widespread
convention.**

```console
$ grep -rl '@jest-environment jsdom' client --include='*.js' --include='*.jsx' \
        --include='*.ts' --include='*.tsx' | wc -l
498
```

498 client files add a `@jest-environment jsdom` docblock to opt into a browser-like environment.

**Claim: the runtime `NODE_ENV` differs between test and dev.**

```console
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --runTestsByPath <temp-spec> --verbose
process.env.NODE_ENV = test
process.env.CALYPSO_ENV = undefined
process.env.TZ = UTC
```

Under Jest, `NODE_ENV=test` and `TZ=UTC`. By contrast, the dev server runs with
`NODE_ENV=development` (the default when unset — see R6 and `config/README.md:L3`).

**Reasoning.** Tests execute under Jest with a shared `@automattic/calypso-jest` preset whose base
environment is `node` [packages/calypso-jest/jest-preset.js:L11], per-suite configs
(`-c=test/<suite>/jest.config.js`), and `NODE_ENV=test`/`TZ=UTC` set by the runner
[package.json:L122] — whereas the dev server runs a webpack build under `NODE_ENV=development`.
Browser-like DOM is **opt-in per test file** (via `@jest-environment jsdom`) rather than global,
which is why `typeof window` is `undefined` in the base environment.

---

## R3 — "Show me which globals, environment variables, and polyfills only exist during test execution"

The temporary observation spec (run under the client Jest config) printed the presence/type of each
injected item. The raw observed console dump:

```console
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --runTestsByPath <temp-spec> --verbose
NODE_ENV=test | CALYPSO_ENV=undefined | TZ=UTC | typeof window=undefined
google typeof=object value={}
__i18n_text_domain__ typeof=string value="default"
fetch typeof=function isMock=true
TextEncoder=function TextDecoder=function
CSS=object CSS.supports=function isMock=true
ResizeObserver=function
matchMedia=function isMock=true
structuredClone=function
crypto=object randomUUID=function subtle=object
ReadableStream=function TransformStream=function Worker=function
```

Each line is attributed to its source below.

### GLOBALS injected by the Jest config `globals` block

```js
// test/client/jest.config.js
globals: {                                   // [test/client/jest.config.js:L22]
    google: {},                              // [test/client/jest.config.js:L23]
    __i18n_text_domain__: 'default',         // [test/client/jest.config.js:L24]
},
```

- `google: {}` — Observed: `google typeof=object value={}`.
- `__i18n_text_domain__: 'default'` — Observed: `__i18n_text_domain__ typeof=string value="default"`.

### ENVIRONMENT VARIABLES

- `NODE_ENV=test` — set by Jest. Observed: `NODE_ENV=test`.
- `TZ=UTC` — set by the `test-client` script [package.json:L122]. Observed: `TZ=UTC`.

### POLYFILLS / MOCKS installed by `test/client/setup-test-framework.js`

The bootstrap begins by importing custom DOM matchers:

```js
// test/client/setup-test-framework.js
import '@testing-library/jest-dom';   // [test/client/setup-test-framework.js:L1]
```

Then it polyfills/mocks the browser surface that `node` does not provide:

- `TextEncoder` / `TextDecoder` [test/client/setup-test-framework.js:L25-L26] — Observed:
  `TextEncoder=function TextDecoder=function`.
- `global.CSS = { supports: jest.fn() }` [test/client/setup-test-framework.js:L30-L32] (also
  installed at the preset level in `packages/calypso-jest/src/setup.js:L3-L5`) — Observed:
  `CSS=object CSS.supports=function isMock=true`.
- `ResizeObserver = require('resize-observer-polyfill')` [test/client/setup-test-framework.js:L34] —
  Observed: `ResizeObserver=function`.
- `global.fetch = jest.fn(...)` [test/client/setup-test-framework.js:L36-L40] — Observed:
  `fetch typeof=function isMock=true`.
- `crypto.randomUUID` [test/client/setup-test-framework.js:L52] — Observed:
  `crypto=object randomUUID=function`.
- `matchMedia = jest.fn(...)` [test/client/setup-test-framework.js:L54-L63] — Observed:
  `matchMedia=function isMock=true`.
- `ReadableStream` / `TransformStream` / `Worker`
  [test/client/setup-test-framework.js:L66-L68] — Observed:
  `ReadableStream=function TransformStream=function Worker=function`.
- `structuredClone` fallback [test/client/setup-test-framework.js:L71-L73] — Observed:
  `structuredClone=function`.
- `crypto.subtle` fallback [test/client/setup-test-framework.js:L76-L79] — Observed:
  `crypto=object ... subtle=object`.

(The setup file also stubs the wpcom transport with
`jest.mock('wpcom-proxy-request')` [test/client/setup-test-framework.js:L44-L49].)

**Reasoning.** Because the base environment is `node` (no browser APIs) and Calypso's client code
assumes them, `setup-test-framework.js` polyfills/mocks the browser surface (`fetch`,
`CSS.supports`, `ResizeObserver`, `matchMedia`, streams, `structuredClone`, `crypto`) and the Jest
config injects app-specific globals (`google`, `__i18n_text_domain__`). None of these exist for the
dev server, which runs real browser code inside a real browser; they exist **only during test
execution**.

---

## R4 — "When code tries to make network requests during tests, what actually happens?"

**Claim: all outbound network is blocked by nock's lockdown.**

```js
// test/client/setup-test-framework.js
nock.disableNetConnect();   // [test/client/setup-test-framework.js:L9]
```

nock is (re)activated in `beforeAll` [test/client/setup-test-framework.js:L11-L16] and cleaned in
`afterAll` [test/client/setup-test-framework.js:L18-L22] around every suite.

**Claim: `global.fetch` is a Jest mock (not a real fetch).**

```js
// test/client/setup-test-framework.js
global.fetch = jest.fn( ... );   // [test/client/setup-test-framework.js:L36-L40]
```

```console
R4 fetch isMock=true
```

**Claim: an unmocked request throws a `NetConnectNotAllowedError`.** Firing an unmocked
`https.get('https://public-api.wordpress.com/rest/v1.1/me')` inside the client Jest environment
produced:

```json
{"name":"NetConnectNotAllowedError","code":"ENETUNREACH","message":"Nock: Disallowed net connect for \"public-api.wordpress.com:443/rest/v1.1/me\""}
```

**External corroboration.** nock's official documentation (github.com/nock/nock,
npmjs.com/package/nock) confirms that after `disableNetConnect()` a non-matching request produces a
`NetConnectNotAllowedError` with the message form `Nock: Disallowed net connect for "<host>:<port>"`
and error code `ENETUNREACH` — matching the captured output above. (The captured line is the primary
evidence; the docs corroborate the exact phrasing for the pinned `nock ^13.5.6`.)

**Repository corroboration.** `docs/testing/testing-overview.md` states that the network connection
is disabled for both the client-side and server-side test configurations (integration tests may use
a network connection).

**Reasoning.** The setup file calls `nock.disableNetConnect()` [test/client/setup-test-framework.js:L9]
so any real socket connection is refused and surfaced as a `NetConnectNotAllowedError` with code
`ENETUNREACH`, and `fetch` is stubbed with `jest.fn()` [test/client/setup-test-framework.js:L36-L40].
Tests must therefore mock every endpoint they touch (see R5) — nothing hits the real network. The
version in use is `nock ^13.5.6` [package.json:L299].

---

## R5 — "Show me a test that mocks an API call and trace how the mocked response flows through the action creator back to the test assertion"

The example is `client/state/user-suggestions/test/actions.js`.

**Claim: the suite passes when run.**

```console
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js \
      --runTestsByPath client/state/user-suggestions/test/actions.js --verbose
PASS client/state/user-suggestions/test/actions.js
  actions
    #receiveUserSuggestions()
      ✓ should return an action object (2 ms)
    #requestUserSuggestions
      ✓ should dispatch properly when receiving a valid response (13 ms)
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
```

**The flow, step by step (each tied to `file:line`):**

1. The HTTP endpoint is stubbed with nock:

   ```js
   // client/state/user-suggestions/test/actions.js
   nock( 'https://public-api.wordpress.com:443' )                       // [.../actions.js:L28]
       .get( '/rest/v1.1/users/suggest?site_id=' + siteId )             // [.../actions.js:L29]
       .reply( 200, deepFreeze( sampleSuccessResponse ) );              // [.../actions.js:L30]
   ```

2. The thunk action creator is dispatched:

   ```js
   const request = requestUserSuggestions( siteId )( dispatchSpy );     // [.../actions.js:L35]
   ```

3. nock intercepts the request (no real network — see R4) and returns the fixture.

4. The resulting Redux actions are asserted:

   ```js
   // synchronous REQUEST action
   USER_SUGGESTIONS_REQUEST { type, siteId }                            // [.../actions.js:L37-L40]
   // on resolution: SUCCESS carrying the fixture as `data`
   USER_SUGGESTIONS_REQUEST_SUCCESS { data: sampleSuccessResponse }     // [.../actions.js:L44-L48]
   // and RECEIVE carrying the suggestions array
   USER_SUGGESTIONS_RECEIVE { suggestions: sampleSuccessResponse.suggestions }  // [.../actions.js:L50-L54]
   ```

**Fixture.** `client/state/user-suggestions/test/sample-response.json` is exactly the `data` that
flows back into the success/receive actions:

```json
{
    "suggestions": [
        { "user_login": "wordpress1" },
        { "user_login": "wordpress2" }
    ]
}
```

**Reasoning.** The mocked response set by `nock(...).reply(200, fixture)` [.../actions.js:L28-L30] is
returned to the thunk's HTTP call; the thunk dispatches a `REQUEST` action synchronously
[.../actions.js:L37-L40], then — on the intercepted 200 — a `REQUEST_SUCCESS` carrying
`data = sampleSuccessResponse` [.../actions.js:L44-L48] and a `RECEIVE` carrying
`sampleSuccessResponse.suggestions` [.../actions.js:L50-L54]. The test asserts on the `dispatchSpy`
calls, so the fixture flows **fixture → thunk → dispatched actions → assertions** with no real
network. This nock-then-dispatch-then-assert shape is a repo-wide convention for action-creator
tests under `client/state`.

---

## R6 — "How does configuration like feature flags get resolved differently in tests versus development?"

**Claim: the active environment is resolved from env vars with a default.**

```js
// client/server/config/index.js
env: process.env.CALYPSO_ENV || process.env.NODE_ENV || 'development',   // [client/server/config/index.js:L6]
enabledFeatures: process.env.ENABLE_FEATURES,                            // [client/server/config/index.js:L7]
disabledFeatures: process.env.DISABLE_FEATURES,                          // [client/server/config/index.js:L8]
```

This is corroborated by `config/README.md:L3`, which states the server chooses the config file from
`NODE_ENV`, defaulting to `"development"`.

**Claim: under Jest, `NODE_ENV=test` so `config/test.json` is loaded; for the dev server
(`NODE_ENV=development`), `config/development.json` is loaded.** The resolved environment observed in
each context (see R7 for the commands):

```console
test context        -> resolved_env=test
development context -> resolved_env=development
```

**Claim: the client test suite rewires the browser config module to the server config module.**

```js
// test/client/jest.config.js
moduleNameMapper: {
    '^@automattic/calypso-config$': '<rootDir>/server/config/index.js',   // [test/client/jest.config.js:L11]
}
```

Why this matters: the real browser module `packages/calypso-config/src/index.ts` refuses to
initialize without a DOM —

```ts
// packages/calypso-config/src/index.ts
if ( 'undefined' === typeof window ) {                                    // [packages/calypso-config/src/index.ts:L17]
    throw new Error( 'Trying to initialize the configuration outside of a browser context.' );  // [.../index.ts:L18]
}
// ...
export const isEnabled = configApi.isEnabled;                            // [.../index.ts:L113]
```

Remapping `@automattic/calypso-config` to the Node/server config module lets browser code read the
JSON-file-backed flags during tests instead of `window.configData`. (The underlying `isEnabled`
implementation is shared via `packages/create-calypso-config/src/index.ts`, which both the browser
and server config build upon.)

**Reasoning.** Config selection is environment-driven
(`CALYPSO_ENV || NODE_ENV || 'development'` [client/server/config/index.js:L6]): the dev server reads
`config/development.json`, whereas Jest (with `NODE_ENV=test`) reads `config/test.json`. And because
the browser config module throws without a DOM [packages/calypso-config/src/index.ts:L17-L18], the
client suite maps `@automattic/calypso-config` to the server resolver
[test/client/jest.config.js:L11] so flags come from the Node-loaded JSON.

---

## R7 — "How do tests control what values config returns, and can you show me proof that a test actually uses a different value than the dev server would resolve?"

Tests control config through two mechanisms.

### Mechanism (a): authoritative `config/<env>.json` overrides

Tests get `config/test.json` values because `NODE_ENV=test` selects that file (see R6). This is the
default, harness-wide mechanism — no per-test code is needed.

### Mechanism (b): per-test mocking of the config module

A test may `jest.mock('@automattic/calypso-config')` and force return values. Example —
`client/jetpack-cloud/sections/agency-dashboard/sites-overview/hooks/test/use-default-site-columns.js`:

```js
// @jest-environment jsdom                                    // [.../use-default-site-columns.js:L2]
jest.mock( '@automattic/calypso-config' );                    // [.../use-default-site-columns.js:L8]
// ...
isEnabled.mockReturnValue( true );                            // [.../use-default-site-columns.js:L24]
// ...
isEnabled.mockReturnValue( false );                           // [.../use-default-site-columns.js:L35]
```

This is a common convention:

```console
$ grep -rl "jest.mock( '@automattic/calypso-config'" client | wc -l
46
```

46 client files mock the config module to control returned values.

### PROOF of divergence (the user's explicit ask)

Use `checkout/checkout-version`. The source literals show it diverges between dev and test:

```jsonc
// config/development.json
"checkout/checkout-version": true,    // [config/development.json:L44]
```

```jsonc
// config/test.json
"checkout/checkout-version": false,   // [config/test.json:L33]
```

```jsonc
// config/production.json (context/control — dev is the outlier)
"checkout/checkout-version": false,   // [config/production.json:L35]
```

Evaluating the **same** flag in each context produced two different results:

- **Test-side (Jest):** the temporary observation spec imported `{ isEnabled }` from
  `@automattic/calypso-config` (remapped to the server config per R6) and logged the result under
  `test/client/jest.config.js`.
- **Dev-side (standalone Node, NOT Jest):** a small script set `process.env.NODE_ENV='development'`,
  `require`d `client/server/config/index.js`, and logged
  `config.isEnabled('checkout/checkout-version')` — faithful to what the dev server would resolve.

The two observed lines, together, are the proof:

```console
NODE_ENV=development : resolved_env=development, isEnabled(checkout/checkout-version)=true
NODE_ENV=test        : resolved_env=test,        isEnabled(checkout/checkout-version)=false
```

### Accuracy note — `catch-js-errors` is NOT a divergence

`catch-js-errors` is `false` in test [config/test.json:L32] **and** was observed `false` under
development as well (the key is absent from `config/development.json`, so it resolves `false` there
too). Therefore `catch-js-errors` does **not** differ between environments. The divergent flag is
**`checkout/checkout-version` only**.

### Observed feature-flag counts (measured, not assumed)

```console
$ grep -cE '":[[:space:]]*(true|false),?[[:space:]]*$' config/development.json
182
$ grep -cE '":[[:space:]]*(true|false),?[[:space:]]*$' config/test.json
105
$ grep -cE '":[[:space:]]*(true|false),?[[:space:]]*$' config/production.json
160
```

`config/development.json` has **182** boolean flag lines, `config/test.json` has **105**, and
`config/production.json` has **160**.

**Reasoning.** Tests control config either by the environment-selected JSON (`config/test.json`,
because `NODE_ENV=test`) or by mocking the config module per test
(`isEnabled.mockReturnValue(...)` [.../use-default-site-columns.js:L24, L35]). The proof shows the
identical call `isEnabled('checkout/checkout-version')` returns `false` under the test config
[config/test.json:L33] but `true` under `development` [config/development.json:L44] — i.e., a test
genuinely resolves a **different value** than the dev server would.

---

## Coverage checklist

Every named item in the request is answered by name:

- [x] **Dev server boots & serves** — R1 (`wp-calypso booted in 996ms`, `HTTP/1.1 200 OK`)
- [x] **Test env vs development** — R2 (`node` base env [jest-preset.js:L11], `NODE_ENV=test` vs
  `development`, seven suites)
- [x] **Globals** — R3 (`google` [jest.config.js:L23], `__i18n_text_domain__` [jest.config.js:L24])
- [x] **Environment variables** — R3 (`NODE_ENV=test`, `TZ=UTC` [package.json:L122])
- [x] **Polyfills** — R3 (`fetch`, `CSS.supports`, `ResizeObserver`, `matchMedia`,
  `structuredClone`, `crypto`, streams, `TextEncoder`/`TextDecoder` —
  `setup-test-framework.js:L25-L79`)
- [x] **Network requests during tests** — R4 (`nock.disableNetConnect()`
  [setup-test-framework.js:L9] → `NetConnectNotAllowedError` / `ENETUNREACH`; `fetch` is a
  `jest.fn()` [setup-test-framework.js:L36-L40])
- [x] **Action creator trace** — R5 (`nock(...).reply` → `requestUserSuggestions` thunk →
  `REQUEST` / `REQUEST_SUCCESS` / `RECEIVE` assertions [actions.js:L28-L54])
- [x] **Feature flags (test vs dev resolution)** — R6 (`CALYPSO_ENV || NODE_ENV || 'development'`
  [config/index.js:L6]; `moduleNameMapper` remap [jest.config.js:L11])
- [x] **Proof of a different resolved value** — R7 (`checkout/checkout-version`: dev `true`
  [development.json:L44] vs test `false` [test.json:L33])
- [x] **Read-only** — no repository file was modified; the temporary observation scripts were
  deleted; `git status` is clean apart from this document.

---

## Read-only note

This investigation modified **no** existing repository file. The only artifact added is this
document, `blitzy/documentation/wp-calypso_be7e5cc64162.md`. Temporary observation scripts used to
capture the evidence above were created outside the tracked tree (or removed immediately after
capture) and have been deleted, so `git status` shows only this new file. Dependencies were not
changed — `yarn install` only materialized the git-ignored `node_modules`, leaving `yarn.lock` and
every manifest untouched.
