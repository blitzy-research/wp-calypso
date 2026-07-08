# Why tests pass in isolation but fail in the full suite: a runtime dissection of `wp-calypso`'s Jest execution contexts

This report answers, from **observed runtime output**, why some tests in the `Automattic/wp-calypso` monorepo pass when run in isolation but fail when run as part of the full suite. Every claim below is backed by a command that was actually executed in this checkout and its complete, unedited output. Each answer follows the pattern **direct answer → command(s) + verbatim output → `file:line` evidence → rationale**.

## Executive summary

The root cause is that the **same test code runs under different runtime environments and different module-resolution rules depending on which suite/command executes it.** The repository defines seven Jest execution contexts (client, packages, server, build-tools, integration, apps, e2e) that share one preset but diverge in three ways: (1) the `testEnvironment` is `node` by default but `jsdom` in some suites/packages/files, so browser globals like `window`/`document` exist in one context and not another; (2) each context loads a **different setup file**, so an injected global such as `matchMedia` is a function in one suite and `undefined` in another; and (3) a custom `enhanced-resolve` resolver plus per-suite `moduleNameMapper` overrides make the *same* import specifier resolve to *different files* depending on context. A test that depends on any of these context-specific facts therefore passes in the context it was written for and fails in another.

## Toolchain (canonical configuration used for every observation)

All commands were run from the repository root `/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e` in the default/canonical configuration, after `yarn install` had completed.

```bash
$ node --version
v22.23.1
$ node_modules/.bin/jest --version
29.7.0
$ yarn --version
4.0.2
$ node -e "console.log(require('jest-environment-jsdom/package.json').version)"
29.7.0
$ node -e "console.log(require('jsdom/package.json').version)"
20.0.3
$ git branch --show-current
blitzy-fc213574-ff44-4dec-b754-98f45e636026
$ git log --oneline -1 be7e5cc641622d153040491fd5625c6cb83e12eb
be7e5cc641 Reader: Show login prompts on all logged out reader streams
$ git diff --name-status be7e5cc641622d153040491fd5625c6cb83e12eb..HEAD
A	blitzy/documentation/wp-calypso_be7e5cc64162.md
```

- **Node v22.23.1** (exact value observed via `node --version` above) — satisfies `engines.node = ^v22.9.0` [package.json:L57]; `.nvmrc` pins `22.9.0` [.nvmrc:L1]. **Deviation noted honestly:** the provided setup script proposed installing Node 20.x (`setup_20.x`), which would violate `engines`; the canonical runtime for this repository is Node 22, and Node **v22.23.1** is what is actually installed and used for every observation in this report.
- **Yarn 4.0.2** via corepack — `packageManager: yarn@4.0.2` [package.json:L422].
- **Jest 29.7.0** at `node_modules/.bin/jest`; **jest-environment-jsdom 29.7.0** bundling **jsdom 20.0.3**.
- **Commit/branch note (baseline vs HEAD, distinguished explicitly, with stable anchors).** Two commits are relevant, and they are anchored on values that stay reproducible:
  - The **baseline (source-branch) commit** is `be7e5cc641622d153040491fd5625c6cb83e12eb` (`git log --oneline -1 be7e5cc641…` → `be7e5cc641 Reader: Show login prompts on all logged out reader streams`). This is the repository state the investigation actually ran against; it is a fixed, always-resolvable commit hash. The deliverable is named after the *source* branch `wp-calypso_be7e5cc64162` (hence `wp-calypso_be7e5cc64162.md`), derived from this baseline commit's short hash.
  - **HEAD** is the *destination*-branch commit that adds **this** document. The single, stable, reproducible invariant — shown above — is `git diff --name-status be7e5cc641…..HEAD` → `A blitzy/documentation/wp-calypso_be7e5cc64162.md`: relative to baseline, the *only* change is the addition of this one file (no existing repository file is modified — see Q8). HEAD's own commit hash is assigned when the document is committed and changes on any re-commit/amend, so it is deliberately **not** pinned here; verify the `diff` invariant rather than expecting a frozen HEAD hash. This is the correction to the earlier revision, which pinned a HEAD hash that goes stale the moment the document is re-committed.
  - The working tree is checked out on the *destination* branch `blitzy-fc213574-ff44-4dec-b754-98f45e636026`; `git branch --show-current` reports that destination branch name, exactly as observed. Every other command in this report resolves paths/behavior identically at baseline and HEAD because none of the investigated evidence-base files changed between them (the only change is the addition of this documentation file).

A note on probe output formatting: several observations use a small probe test that logs lines prefixed with `PROBE|`. Those lines are extracted from Jest's console output with `| grep -oE 'PROBE\|.*'` (Jest indents `console.log` output; the `grep` strips that cosmetic indentation). The `PROBE|` content is verbatim. Any config used purely to *isolate* what the environment provides versus what a setup file injects (a "bare" config with no setup files) is explicitly labeled a **non-canonical isolation probe**.

---

## Q1 — What test commands exist, and what runtime environment does each actually use?

**Direct answer.** The root `package.json` `scripts` block exposes six suite commands — `test-client`, `test-packages`, `test-server`, `test-build-tools`, `test-integration`, `test-apps` — plus the aggregate `test`, which runs only four of them (`test-client test-packages test-server test-build-tools`) via `run-s`. A seventh suite is the Playwright E2E suite (`test/e2e/jest.config.js`), invoked separately. Each command runs Jest with a suite-specific config. The resolved `testEnvironment` is **`node` by default** (from the shared base preset) and **`jsdom` only where explicitly opted in**: the entire `apps` suite, 22 of the 58 `packages` projects, and individual client/component test files via a `@jest-environment jsdom` docblock. The aggregate `test` command **excludes** integration, apps, and e2e.

**Command + verbatim output — the test commands (values):**

```bash
$ node -e "const p=require('./package.json'); for (const k of Object.keys(p.scripts)) if (k==='test'||k.startsWith('test-')) console.log(k+' = '+p.scripts[k]);"
test = run-s -s test-client test-packages test-server test-build-tools
test-build-tools = jest -c=test/build-tools/jest.config.js
test-client = TZ=UTC jest -c=test/client/jest.config.js
test-client:watch = yarn run test-client --watch
test-desktop:e2e = echo 'Deprecated, run `cd desktop && yarn run test:e2e` instead'
test-integration = jest -c=test/integration/jest.config.js
test-integration:watch = yarn run test-integration --watch
test-apps = jest -c=test/apps/jest.config.js
test-apps:watch = yarn run test-apps --watch
test-packages = jest -c=test/packages/jest.config.js
test-packages:watch = yarn run test-packages --watch
test-server = jest -c=test/server/jest.config.js
test-server:coverage = yarn run test-server --coverage
test-server:watch = yarn run test-server --watch
```

**Command + verbatim output — the exact line numbers of each `test`/`test-*` key** (this pasted `grep -nE` output is the direct source of every line-number claim below):

```bash
$ grep -nE '"test(-[a-z-]+)?":' package.json
52:		"test": [
120:		"test": "run-s -s test-client test-packages test-server test-build-tools",
121:		"test-build-tools": "jest -c=test/build-tools/jest.config.js",
122:		"test-client": "TZ=UTC jest -c=test/client/jest.config.js",
125:		"test-integration": "jest -c=test/integration/jest.config.js",
127:		"test-apps": "jest -c=test/apps/jest.config.js",
129:		"test-packages": "jest -c=test/packages/jest.config.js",
131:		"test-server": "jest -c=test/server/jest.config.js",
```

Reading the pasted `grep` output directly: the **test scripts** are `test`=**L120**, `test-build-tools`=**L121**, `test-client`=**L122**, `test-integration`=**L125**, `test-apps`=**L127**, `test-packages`=**L129**, `test-server`=**L131**. The regex `"test(-[a-z-]+)?":` intentionally matches only keys ending in `":` (so the `:watch`/`:coverage` variants at L123/L126/L128/L130/L132/L133 and the deprecated `test-desktop:e2e` are excluded). It also matches one **non-script** line — **L52 `"test": [`** — which is the `browserslist` `test` target (an array of target browsers), *not* an npm script; it is included here exactly as the command emits it and is called out so the output is complete and honest rather than filtered.

**Command + verbatim output — resolved `testEnvironment` for each single-config suite:**

```bash
$ for cfg in test/client/jest.config.js test/build-tools/jest.config.js test/server/jest.config.js test/integration/jest.config.js; do echo "--- $cfg ---"; node_modules/.bin/jest --showConfig -c=$cfg 2>/dev/null | grep '"testEnvironment"' | head -1; done
--- test/client/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",
--- test/build-tools/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",
--- test/server/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",
--- test/integration/jest.config.js ---
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",
```

All four resolve to `jest-environment-node`.

**Command + verbatim output — the projects fan-out suites (packages and apps) are a mix:**

```bash
$ node_modules/.bin/jest --showConfig -c=test/packages/jest.config.js 2>/dev/null | grep '"testEnvironment"' | sort | uniq -c
     22       "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-jsdom/build/index.js",
     36       "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",

$ node_modules/.bin/jest --showConfig -c=test/apps/jest.config.js 2>/dev/null | grep '"testEnvironment"' | sort | uniq -c
      3       "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-jsdom/build/index.js",
```

The `packages` suite has 58 projects: **22 jsdom + 36 node**. The `apps` suite has **3 jsdom** projects.

**Command + verbatim output — which packages override to jsdom, and confirmation that `@automattic/components` does not:**

```bash
$ grep -rl "testEnvironment.*jsdom" packages/*/jest.config.js
packages/block-renderer/jest.config.js
packages/calypso-products/jest.config.js
packages/calypso-sentry/jest.config.js
packages/calypso-url/jest.config.js
packages/command-palette/jest.config.js
packages/composite-checkout/jest.config.js
packages/dataviews/jest.config.js
packages/design-picker/jest.config.js
packages/design-preview/jest.config.js
packages/domain-picker/jest.config.js
packages/global-styles/jest.config.js
packages/help-center/jest.config.js
packages/launchpad/jest.config.js
packages/odie-client/jest.config.js
packages/onboarding/jest.config.js
packages/search/jest.config.js
packages/shopping-cart/jest.config.js
packages/site-admin/jest.config.js
packages/sites/jest.config.js
packages/subscriber/jest.config.js
packages/verbum-block-editor/jest.config.js
packages/wpcom-checkout/jest.config.js

--- count ---
22

$ node_modules/.bin/jest --showConfig -c=packages/components/jest.config.js 2>/dev/null | grep '"testEnvironment"' | head -1
      "testEnvironment": "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-environment-node/build/index.js",
```

The 22 packages that set `jsdom` in their own `jest.config.js` are: **block-renderer, calypso-products, calypso-sentry, calypso-url, command-palette, composite-checkout, dataviews, design-picker, design-preview, domain-picker, global-styles, help-center, launchpad, odie-client, onboarding, search, shopping-cart, site-admin, sites, subscriber, verbum-block-editor, wpcom-checkout**. `@automattic/components` does **not** override — it resolves to `jest-environment-node`; individual test files opt into jsdom via a docblock (demonstrated in Q3).

**Command + verbatim output — the E2E suite uses a custom environment class:**

```bash
$ grep -n 'class JestEnvironmentPlaywright\|extends NodeEnvironment\|export default' packages/calypso-e2e/src/jest-playwright-config/environment.ts
46:class JestEnvironmentPlaywright extends NodeEnvironment {
459:export default JestEnvironmentPlaywright;

--- head import of NodeEnvironment in environment.ts ---
11:import NodeEnvironment from 'jest-environment-node';
46:class JestEnvironmentPlaywright extends NodeEnvironment {

$ cat -n packages/calypso-e2e/src/jest-playwright-config/index.js
     1	const path = require( 'path' );
     2	
     3	/** @type {import('@jest/types').Config.InitialOptions} */
     4	const config = {
     5		globalSetup: path.join( __dirname, 'global-setup.ts' ),
     6		runner: 'groups',
     7		testEnvironment: path.join( __dirname, 'environment.ts' ),
     8		testRunner: 'jest-circus/runner',
     9		testTimeout: process.env.PWDEBUG === '1' ? 10 * 60 * 1000 : 2 * 60 * 1000,
    10		verbose: true,
    11	};
    12	
    13	module.exports = config;
```

The E2E environment is a custom `JestEnvironmentPlaywright` that **extends `NodeEnvironment`** (i.e., it is still a node environment at its core, with Playwright browser control layered on).

**Stability.** The `--showConfig` environment resolutions and the fan-out counts (22/36 for packages, 3 for apps) were identical across two runs.

**`file:line` evidence.**
- Base default environment: `packages/calypso-jest/jest-preset.js:L11` → `testEnvironment: 'node'` (also `L9` custom `resolver`, `L10` base `setupFilesAfterEnv` = `src/setup.js`, `L13`–`L16` `transform` with `babel-jest` `rootMode:'upward'` at `L14` and the asset transform at `L15`).
- Apps override: `test/apps/jest-preset.js:L7` → `testEnvironment: 'jsdom'` (also `L11` `setupFiles: ['jest-canvas-mock']`, `L13` reuses `../client/setup-test-framework.js`).
- Integration explicit: `test/integration/jest.config.js:L7` → `testEnvironment: 'node'`.
- Fan-out configs: `test/packages/jest.config.js:L4` → `projects: ['<rootDir>/packages/*/jest.config.js']`; `test/apps/jest.config.js:L4` → `projects: ['<rootDir>/apps/*/jest.config.js']`.
- Components runs in the packages suite: `packages/components/jest.config.js:L2` → `preset: '../../test/packages/jest-preset.js'`.
- E2E: `packages/calypso-e2e/src/jest-playwright-config/environment.ts:L11` (`import NodeEnvironment from 'jest-environment-node'`), `:L46` (`class JestEnvironmentPlaywright extends NodeEnvironment`), `:L459` (`export default`); `packages/calypso-e2e/src/jest-playwright-config/index.js:L6`–`L10`.
- Aggregate `test`: `package.json:L120` → `run-s -s test-client test-packages test-server test-build-tools`.

**Rationale.** All unit/component suites share the `@automattic/calypso-jest` preset whose default `testEnvironment` is `node`; suites, packages, and individual files that need a DOM opt into `jsdom`. Because the aggregate `test` runs client+packages+server+build-tools (not integration/apps/e2e), and because environments differ within the packages suite itself (22 jsdom vs 36 node), the *same* test code can execute under a node environment in one invocation and a jsdom environment in another. This per-context environment split is the first reason identical test code behaves differently between an "isolated" run and a "full-suite" run.

---

## Q2 — What is available globally in each context, and what exists in one but not another?

**Direct answer.** `window`, `document`, and `localStorage` exist **only under the `jsdom` environment**. Node-22 natives (`fetch`, `crypto`, `structuredClone`, `ReadableStream`, `TransformStream`, `TextEncoder`, `navigator`) are present in the **node** environment but **absent in a bare jsdom environment** unless a setup file injects them. Capabilities like `matchMedia`, `ResizeObserver`, `Worker`, and `CSS` are **never environment-native** — they exist only when a per-context setup file injects them. Concrete cross-context differences (something that exists in one context but not another):

- **`matchMedia`** is a `function` in **client** and **packages**, but `undefined` in **server**.
- **`crypto.randomUUID()`** returns a **real UUID** in **client** and **server**, but the literal string **`'fake-uuid'`** in **packages**.
- **`Worker`** is a `function` in **client**, but `undefined` in **packages** and **server**.
- **`CSS`** is an `object` in **client** and **packages**, but `undefined` in **server**.

**Probe methodology (fully reproducible).** A single probe test logs `typeof` for a fixed key list, then calls `crypto.randomUUID()`. Its body is identical in every context:

```js
test( 'blitzy globals probe', () => {
	const g = globalThis;
	const keys = [ 'window', 'document', 'navigator', 'matchMedia', 'fetch', 'ResizeObserver', 'crypto', 'structuredClone', 'ReadableStream', 'TransformStream', 'Worker', 'CSS', 'TextEncoder', 'localStorage' ];
	const rows = keys.map( ( k ) => 'PROBE|typeof ' + k + ' = ' + typeof g[ k ] );
	rows.push( 'PROBE|crypto.randomUUID() = ' + ( g.crypto && typeof g.crypto.randomUUID === 'function' ? g.crypto.randomUUID() : 'N/A' ) );
	rows.push( 'PROBE|typeof crypto.subtle = ' + ( g.crypto ? typeof g.crypto.subtle : 'N/A' ) );
	console.log( '\n' + rows.join( '\n' ) + '\n' );
	expect( true ).toBe( true );
} );
```

The exact setup below writes that body once and materializes it into (a) each suite's `testMatch` directory — the base preset's `testMatch` is `<rootDir>/**/test/*.[jt]s?(x)` [packages/calypso-jest/jest-preset.js:L12] — and (b) three **non-canonical isolation** configs (a bare environment with **no setup files**) used to attribute each global to either the environment or a setup file. These probe directories are outside version control and are removed afterward (the cleanup command is shown at the end of this section, and Q8 verifies the tree is clean). Run every command below from the repository root:

```bash
$ cat > /tmp/blitzy_globals_probe.js <<'EOF'
test( 'blitzy globals probe', () => {
	const g = globalThis;
	const keys = [ 'window', 'document', 'navigator', 'matchMedia', 'fetch', 'ResizeObserver', 'crypto', 'structuredClone', 'ReadableStream', 'TransformStream', 'Worker', 'CSS', 'TextEncoder', 'localStorage' ];
	const rows = keys.map( ( k ) => 'PROBE|typeof ' + k + ' = ' + typeof g[ k ] );
	rows.push( 'PROBE|crypto.randomUUID() = ' + ( g.crypto && typeof g.crypto.randomUUID === 'function' ? g.crypto.randomUUID() : 'N/A' ) );
	rows.push( 'PROBE|typeof crypto.subtle = ' + ( g.crypto ? typeof g.crypto.subtle : 'N/A' ) );
	console.log( '\n' + rows.join( '\n' ) + '\n' );
	expect( true ).toBe( true );
} );
EOF

# (a) One probe file per canonical suite context (base testMatch = <rootDir>/**/test/*.[jt]s?(x)):
$ mkdir -p client/blitzy_probe/test packages/components/blitzy_probe/test client/server/blitzy_probe/test
$ cp /tmp/blitzy_globals_probe.js client/blitzy_probe/test/probe_client_node.js
$ { printf '/**\n * @jest-environment jsdom\n */\n'; cat /tmp/blitzy_globals_probe.js; } > client/blitzy_probe/test/probe_client_jsdom.js
$ cp /tmp/blitzy_globals_probe.js packages/components/blitzy_probe/test/probe_pkg.js
$ cp /tmp/blitzy_globals_probe.js client/server/blitzy_probe/test/probe_server.js

# (b) Three non-canonical isolation configs (bare environment, NO setup files):
$ mkdir -p blitzy_iso_probe
$ cp /tmp/blitzy_globals_probe.js blitzy_iso_probe/probe.test.js
$ printf "require( '@testing-library/jest-dom' );\n" > blitzy_iso_probe/jestdom_setup.js
$ cat > blitzy_iso_probe/jest.node.js <<'EOF'
module.exports = {
	rootDir: __dirname,
	testEnvironment: 'node',
	testMatch: [ '<rootDir>/probe.test.js' ],
	transform: {},
};
EOF
$ cat > blitzy_iso_probe/jest.jsdom.js <<'EOF'
module.exports = {
	rootDir: __dirname,
	testEnvironment: 'jsdom',
	testMatch: [ '<rootDir>/probe.test.js' ],
	transform: {},
};
EOF
$ cat > blitzy_iso_probe/jest.node_jestdom.js <<'EOF'
module.exports = {
	rootDir: __dirname,
	testEnvironment: 'node',
	testMatch: [ '<rootDir>/probe.test.js' ],
	setupFilesAfterEnv: [ '<rootDir>/jestdom_setup.js' ],
	transform: {},
};
EOF
```

Each probe is invoked with `--silent=false` and the `PROBE|` lines are extracted with `grep -oE 'PROBE\|.*'` (which also strips Jest's cosmetic `console.log` indentation). The exact per-context command is shown inside each fenced block below.

### Isolation probes (non-canonical) — what the *environment alone* provides

**Bare `jest-environment-node`, NO setup files** (isolation probe — non-canonical):

```bash
$ node_modules/.bin/jest -c=blitzy_iso_probe/jest.node.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = undefined
PROBE|typeof document = undefined
PROBE|typeof navigator = object
PROBE|typeof matchMedia = undefined
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = undefined
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = undefined
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = undefined
PROBE|crypto.randomUUID() = 2b3ce5ee-95d3-4dbf-90b3-121b2821b476
PROBE|typeof crypto.subtle = object
```

**Bare `jest-environment-jsdom`, NO setup files** (isolation probe — non-canonical):

```bash
$ node_modules/.bin/jest -c=blitzy_iso_probe/jest.jsdom.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = object
PROBE|typeof document = object
PROBE|typeof navigator = object
PROBE|typeof matchMedia = undefined
PROBE|typeof fetch = undefined
PROBE|typeof ResizeObserver = undefined
PROBE|typeof crypto = object
PROBE|typeof structuredClone = undefined
PROBE|typeof ReadableStream = undefined
PROBE|typeof TransformStream = undefined
PROBE|typeof Worker = undefined
PROBE|typeof CSS = undefined
PROBE|typeof TextEncoder = undefined
PROBE|typeof localStorage = object
PROBE|crypto.randomUUID() = N/A
PROBE|typeof crypto.subtle = undefined
```

These two isolation probes prove attribution: `window`/`document`/`localStorage` come **only** from the jsdom environment (they flip `undefined`→`object` when switching node→jsdom), while `fetch`/`structuredClone`/`ReadableStream`/`TransformStream`/`TextEncoder` and a usable `crypto.randomUUID`/`crypto.subtle` are **Node-22 natives** present in the node env but **absent** in a bare jsdom env (jsdom's `crypto` object exists but has no `randomUUID`, hence `N/A`).

**Bare `jest-environment-node` + ONLY `require('@testing-library/jest-dom')`** (isolation probe — proves where `CSS` comes from in the packages context):

```bash
$ node_modules/.bin/jest -c=blitzy_iso_probe/jest.node_jestdom.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = undefined
PROBE|typeof document = undefined
PROBE|typeof navigator = object
PROBE|typeof matchMedia = undefined
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = undefined
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = object
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = undefined
PROBE|crypto.randomUUID() = 999b4838-ef89-4bab-bbc5-03485f4f61ef
PROBE|typeof crypto.subtle = object
```

`CSS` flips to `object` here while `matchMedia`/`window` stay `undefined` — proving that in the packages context **`CSS` is injected by the `@testing-library/jest-dom` import**, not by the base preset.

### Canonical suite probes — what each *real* context provides

**CLIENT suite, no docblock (node environment):**

```bash
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --silent=false client/blitzy_probe/test/probe_client_node.js 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = undefined
PROBE|typeof document = undefined
PROBE|typeof navigator = object
PROBE|typeof matchMedia = function
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = function
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = function
PROBE|typeof CSS = object
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = undefined
PROBE|crypto.randomUUID() = b509aa00-3320-4c6e-90b3-6f8aa85af20d
PROBE|typeof crypto.subtle = object
```

**CLIENT suite, WITH `/** @jest-environment jsdom */` docblock (jsdom environment)** — same suite config, the docblock probe file:

```bash
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --silent=false client/blitzy_probe/test/probe_client_jsdom.js 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = object
PROBE|typeof document = object
PROBE|typeof navigator = object
PROBE|typeof matchMedia = function
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = function
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = function
PROBE|typeof CSS = object
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = object
PROBE|crypto.randomUUID() = 1b9659f8-166b-4f59-9ef9-9e864124bf4d
PROBE|typeof crypto.subtle = object
```

Only `window`, `document`, and `localStorage` change (`undefined`→`object`) when the docblock flips the environment to jsdom; every other global is identical because the client `setupFilesAfterEnv` injects them regardless of environment.

**PACKAGES suite via `@automattic/components` (node environment):**

```bash
$ node_modules/.bin/jest -c=packages/components/jest.config.js --silent=false packages/components/blitzy_probe/test/probe_pkg.js 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = undefined
PROBE|typeof document = undefined
PROBE|typeof navigator = object
PROBE|typeof matchMedia = function
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = function
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = object
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = undefined
PROBE|crypto.randomUUID() = fake-uuid
PROBE|typeof crypto.subtle = object
```

**SERVER suite (node environment):**

```bash
$ node_modules/.bin/jest -c=test/server/jest.config.js --silent=false client/server/blitzy_probe/test/probe_server.js 2>&1 | grep -oE 'PROBE\|.*'
PROBE|typeof window = undefined
PROBE|typeof document = undefined
PROBE|typeof navigator = object
PROBE|typeof matchMedia = undefined
PROBE|typeof fetch = function
PROBE|typeof ResizeObserver = undefined
PROBE|typeof crypto = object
PROBE|typeof structuredClone = function
PROBE|typeof ReadableStream = function
PROBE|typeof TransformStream = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = undefined
PROBE|typeof TextEncoder = function
PROBE|typeof localStorage = undefined
PROBE|crypto.randomUUID() = 540112ac-3513-4496-be81-73fde49e3384
PROBE|typeof crypto.subtle = object
```

### Consolidated cross-context table

| global | bare node (isolation) | bare jsdom (isolation) | client (node) | client (jsdom docblock) | packages | server |
|---|---|---|---|---|---|---|
| `window` | undefined | object | undefined | **object** | undefined | undefined |
| `document` | undefined | object | undefined | **object** | undefined | undefined |
| `localStorage` | undefined | object | undefined | **object** | undefined | undefined |
| `navigator` | object | object | object | object | object | object |
| `matchMedia` | undefined | undefined | **function** | function | **function** | **undefined** |
| `fetch` | function | undefined | function | function | function | function |
| `ResizeObserver` | undefined | undefined | function | function | function | undefined |
| `crypto` | object | object | object | object | object | object |
| `structuredClone` | function | undefined | function | function | function | function |
| `ReadableStream` | function | undefined | function | function | function | function |
| `TransformStream` | function | undefined | function | function | function | function |
| `Worker` | undefined | undefined | **function** | function | **undefined** | **undefined** |
| `CSS` | undefined | undefined | object | object | **object** | **undefined** |
| `TextEncoder` | function | undefined | function | function | function | function |
| `crypto.randomUUID()` | real UUID | N/A | real UUID | real UUID | **`'fake-uuid'`** | real UUID |
| `crypto.subtle` | object | undefined | object | object | object | object |

### Stability (≥2 runs)

`typeof` values were identical across two runs in every context. `crypto.randomUUID()` is intentionally non-deterministic where it is a real UUID and deterministic where it is stubbed. Each context was run twice, extracting the differentiating keys:

```bash
$ for i in 1 2; do echo "--- PACKAGES run $i ---"; node_modules/.bin/jest -c=packages/components/jest.config.js --silent=false packages/components/blitzy_probe/test/probe_pkg.js 2>&1 | grep -oE "PROBE\|(typeof matchMedia|typeof Worker|typeof CSS|crypto.randomUUID).*"; done
--- PACKAGES run 1 ---
PROBE|typeof matchMedia = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = object
PROBE|crypto.randomUUID() = fake-uuid
--- PACKAGES run 2 ---
PROBE|typeof matchMedia = function
PROBE|typeof Worker = undefined
PROBE|typeof CSS = object
PROBE|crypto.randomUUID() = fake-uuid

$ for i in 1 2; do echo "--- SERVER run $i ---"; node_modules/.bin/jest -c=test/server/jest.config.js --silent=false client/server/blitzy_probe/test/probe_server.js 2>&1 | grep -oE "PROBE\|(typeof matchMedia|typeof CSS|crypto.randomUUID).*"; done
--- SERVER run 1 ---
PROBE|typeof matchMedia = undefined
PROBE|typeof CSS = undefined
PROBE|crypto.randomUUID() = ef2edcbf-1dd8-4f01-bc04-b2907ca298e0
--- SERVER run 2 ---
PROBE|typeof matchMedia = undefined
PROBE|typeof CSS = undefined
PROBE|crypto.randomUUID() = 50fe0a11-9f6b-4ed7-b5de-174d88a5ffee

$ for i in 1 2; do echo "--- CLIENT(node) run $i ---"; TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --silent=false client/blitzy_probe/test/probe_client_node.js 2>&1 | grep -oE "PROBE\|(typeof matchMedia|typeof Worker|crypto.randomUUID).*"; done
--- CLIENT(node) run 1 ---
PROBE|typeof matchMedia = function
PROBE|typeof Worker = function
PROBE|crypto.randomUUID() = 3650d7b1-2bca-4390-af47-b57f088396a3
--- CLIENT(node) run 2 ---
PROBE|typeof matchMedia = function
PROBE|typeof Worker = function
PROBE|crypto.randomUUID() = 9b6bc1b8-d2c9-4596-9ba4-bde9fc9d1533
```

Reported exactly as observed: the `typeof` values (`matchMedia`, `Worker`, `CSS`) are identical across both runs in every context. In **packages**, `crypto.randomUUID()` is the deterministic literal `'fake-uuid'` both runs; in **server** and **client (node)** it is a **fresh real UUID each run** (server: `ef2edcbf…` then `50fe0a11…`; client: `3650d7b1…` then `9b6bc1b8…`). This value is non-deterministic **by design** and is not normalized toward any fixed value. (The individual full-probe blocks above were captured on separate invocations, so their real UUIDs differ from these — that is the expected, honestly-reported behavior of a per-call random UUID.)

### Attribution (`file:line`) — exactly which mechanism provides each global

- `window` / `document` / `localStorage` ⇐ the **jsdom environment** only (see Q6). Proven by the bare-jsdom vs bare-node isolation probes and by the client docblock flipping them `undefined`→`object`.
- `fetch`, `crypto` (incl. `randomUUID`, `subtle`), `structuredClone`, `ReadableStream`, `TransformStream`, `TextEncoder`, `navigator` ⇐ **Node-22 natives** in the node env (present in bare-node, absent in bare-jsdom).
- Client-injected globals — `test/client/setup-test-framework.js`: `TextEncoder`/`TextDecoder` `L25`–`L26`, `CSS` `L30`–`L32`, `ResizeObserver` `L34`, `fetch` mock `L36`–`L40`, `crypto.randomUUID = () => nodeCrypto.randomUUID()` `L52` (returns a **real** UUID), `matchMedia` `L54`–`L63`, `ReadableStream`/`TransformStream`/`Worker` `L66`–`L68`, `structuredClone` `L71`–`L73`, `crypto.subtle` `L76`–`L79`; imports `@testing-library/jest-dom` `L1`; `nock.disableNetConnect()` `L9`.
- Packages-injected globals — `test/packages/setup.js`: `crypto.randomUUID = () => 'fake-uuid'` `L3`, `ResizeObserver` `L5`, `matchMedia` `L7`–`L16`, imports `@testing-library/jest-dom` `L1`. The packages setup does **not** define `Worker`, which is why `Worker=undefined` in packages.
- `CSS` in the packages context ⇐ the **`@testing-library/jest-dom` import** (`test/packages/setup.js:L1`), NOT the base preset — verified above by the bare-node + jest-dom-only isolation probe (`CSS=object` while `matchMedia`/`window` stayed `undefined`).
- Server context — `test/server/setup-test-framework.js` defines **none** of `matchMedia`/`ResizeObserver`/`fetch`/`CSS`/`Worker`; it only calls `nock.disableNetConnect()` `L4` and mocks `wpcom-proxy-request` `L21`. Additionally, `test/server/jest.config.js:L13` sets `setupFilesAfterEnv: [require.resolve('./setup-test-framework.js')]`, which **replaces** (does not merge with) the base preset's `setupFilesAfterEnv` (`packages/calypso-jest/src/setup.js`, which sets `global.CSS = { supports: jest.fn() }` at `L3`–`L5`). That replacement is why `CSS=undefined` in server. `fetch`/`crypto.randomUUID` in server come from **Node-22 natives**, not any setup file.
- Base-preset injected global — `packages/calypso-jest/src/setup.js:L3`–`L5`: `global.CSS = { supports: jest.fn() }` (applies wherever the base `setupFilesAfterEnv` is not overridden).

**Rationale.** Because the environment differs (node vs jsdom) **and** each context loads a different setup file, the *same* global name can be a function in one suite and `undefined` in another. A test relying on `matchMedia` passes under client/packages but throws under server; a test asserting a deterministic UUID passes under packages (`'fake-uuid'`) but not under client/server (a real UUID). This is a core mechanism behind "passes in isolation, fails in the suite."

**Cleanup (removes every Q2 probe; leaves the tree unchanged — see Q8):**

```bash
$ rm -rf client/blitzy_probe packages/components/blitzy_probe client/server/blitzy_probe blitzy_iso_probe /tmp/blitzy_globals_probe.js
$ git status --porcelain
```

`git status --porcelain` prints nothing after cleanup (the only working-tree change in this repository is this documentation file itself).

---

## Q6 — What provides the browser-like APIs, and when does it become available?

**Direct answer.** The browser-like environment is provided by **`jest-environment-jsdom@29.7.0`**, which bundles **`jsdom@20.0.3`**. It becomes available at **environment construction time — before `setupFiles`, before the test framework is installed, and before the test file runs** — and only in `jsdom` contexts. As of Jest 28 the built-in jsdom environment was removed from Jest's default install and split into this standalone `jest-environment-jsdom` package, so in Jest 29 (used here) it is already a separate package. The DOM globals (`window`, `document`, `localStorage`) originate here; other "browser-ish" capabilities (`matchMedia`, `fetch`, `ResizeObserver`, `Worker`, `CSS`) are **not** provided by jsdom and are instead injected later by the per-context setup files (see Q2 attribution).

**Command + verbatim output — version of the provider and its bundled jsdom:**

```bash
$ node -e "console.log(require('jest-environment-jsdom/package.json').version)"
29.7.0
$ node -e "console.log(require('jsdom/package.json').version)"
20.0.3
```

**Command + verbatim output — timing: `window`/`document` exist from the environment constructor (STAGE1), before any setup file.** This uses the same reproducible initialization-order probe defined in Q5/Q7 below (`blitzy_initorder_probe/`); the jsdom run shows `window`/`document` already present in the environment constructor:

```bash
$ node_modules/.bin/jest -c=blitzy_initorder_probe/jest.jsdom.config.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=object typeof this.global.document=object
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=object
PROBE|STAGE4|setupFilesAfterEnv | typeof jest=object typeof expect=function typeof beforeAll=function typeof test=function typeof window=object
PROBE|STAGE5|test-file module scope | typeof expect=function typeof window=object
PROBE|STAGE5b|beforeAll hook fired
PROBE|STAGE6|inside test() body
```

Contrast the node run, where `window` is `undefined` at every stage:

```bash
$ node_modules/.bin/jest -c=blitzy_initorder_probe/jest.node.config.js --silent=false 2>&1 | grep -oE 'PROBE\|STAGE[12]\|.*'
PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=undefined typeof this.global.matchMedia=undefined
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=undefined
```

**`file:line` evidence.**
- jsdom opt-in points: `test/apps/jest-preset.js:L7` (`testEnvironment: 'jsdom'`); the 22 package `jest.config.js` overrides listed in Q1; per-file `/** @jest-environment jsdom */` docblocks (e.g., `packages/components/src/external-link/test/index.js`, shown in Q3).
- Corroborating in-repo comment: `test/apps/jest-preset.js:L12` notes the setup file "includes a lot of globals that don't exist, like fetch, matchMedia, etc." — matching the Q2 attribution that those come from setup files, not jsdom.

**Rationale.** jsdom is the DOM provider and is constructed first (STAGE1), so any DOM access at module-import time works only if the environment is jsdom. Everything else that looks browser-like (`matchMedia`, `fetch`, …) is layered on afterward by setup files, which is exactly why a jsdom environment alone (the bare-jsdom isolation probe) still reports `matchMedia=undefined` and `fetch=undefined`.


---

## Q3 — When an internal dependency is imported, what file actually loads? Does it differ by execution method?

**Direct answer.** Yes — it differs by execution method. `@automattic/components` depends on the internal package `@automattic/i18n-utils` (`packages/components/package.json:L34` → `"@automattic/i18n-utils": "workspace:^"`). **Under Jest, the import resolves to the untranspiled TypeScript source `packages/i18n-utils/src/index.ts`**, because the repository's custom resolver prioritizes the `calypso:src` field. **Under plain Node / a production build, the same import targets the `main` field `dist/cjs/index.js`, which does not exist in this checkout → `MODULE_NOT_FOUND`.**

**Command + verbatim output — the package fields and the filesystem reality:**

```bash
$ node -e "const p=require('./packages/i18n-utils/package.json'); console.log('main:',p.main,'| module:',p.module,'| calypso:src:',p['calypso:src'],'| exports:',JSON.stringify(p.exports));"
main: dist/cjs/index.js | module: dist/esm/index.js | calypso:src: src/index.ts | exports: undefined

$ ls -la packages/i18n-utils/src/index.ts ; ls -la packages/i18n-utils/dist/cjs 2>/dev/null || echo "packages/i18n-utils/dist/cjs => ABSENT"
-rw-r--r-- 1 root root 372 Jul  8 04:22 packages/i18n-utils/src/index.ts
packages/i18n-utils/dist/cjs => ABSENT
```

`src/index.ts` exists (372 bytes); `dist/cjs` is **absent** (only `dist/esm`, `dist/types`, and `tsconfig.tsbuildinfo` were produced during install).

**Command + verbatim output — (1) the custom resolver invoked directly** resolves to source:

```bash
$ node -e "const resolve=require('./packages/calypso-jest/src/module-resolver.js'); const path=require('path'); const basedir=path.resolve('packages/components/src'); console.log('RESOLVER|', path.relative(process.cwd(), resolve('@automattic/i18n-utils',{basedir})));"
RESOLVER| packages/i18n-utils/src/index.ts
```

**Reproducible probe setup** — two throwaway probe files in `packages/components/blitzy_probe/test/` (matched by the packages suite's `testMatch`), removed at the end of this section:

```bash
$ mkdir -p packages/components/blitzy_probe/test
$ cat > packages/components/blitzy_probe/test/resolve-i18n.js <<'EOF'
test( 'resolve @automattic/i18n-utils under Jest', () => {
	console.log( 'PROBE|require.resolve(@automattic/i18n-utils) = ' + require.resolve( '@automattic/i18n-utils' ) );
	expect( true ).toBe( true );
} );
EOF
$ cat > packages/components/blitzy_probe/test/import-i18n-node.js <<'EOF'
import '@automattic/i18n-utils';

test( 'import @automattic/i18n-utils at top level in node env', () => {
	expect( true ).toBe( true );
} );
EOF
```

**Command + verbatim output — (2) `require.resolve` inside a real Jest test** (packages/components context, which uses the custom resolver) resolves to source and the test passes (complete, unedited output):

```bash
$ node_modules/.bin/jest -c=packages/components/jest.config.js --silent=false packages/components/blitzy_probe/test/resolve-i18n.js 2>&1
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS packages/components/blitzy_probe/test/resolve-i18n.js
  ● Console

    console.log
      PROBE|require.resolve(@automattic/i18n-utils) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/packages/i18n-utils/src/index.ts

      at Object.log (blitzy_probe/test/resolve-i18n.js:2:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.7 s, estimated 1 s
Ran all test suites matching /packages\/components\/blitzy_probe\/test\/resolve-i18n.js/i.
```

**Command + verbatim output — (3) plain Node `require.resolve`** (which uses `main`) errors because `dist/cjs/index.js` does not exist:

```bash
$ node -e "try{const p=require.resolve('@automattic/i18n-utils',{paths:[require('path').resolve('packages/components')]});console.log('NODE| resolved =>',p);}catch(e){console.log('NODE| ERROR',e.code+':',e.message.split(String.fromCharCode(10))[0]);}"
NODE| ERROR MODULE_NOT_FOUND: Cannot find module '/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/@automattic/i18n-utils/dist/cjs/index.js'. Please verify that the package.json has a valid "main" entry
```

**Command + verbatim output — (4) a genuine test run** proving source resolution works end-to-end. The test file opts into jsdom via a docblock. Complete, unedited output (`2>&1`, including the Browserslist stderr warning that Jest prints before the run):

```bash
$ head -3 packages/components/src/external-link/test/index.js
/**
 * @jest-environment jsdom
 */

$ node_modules/.bin/jest -c=packages/components/jest.config.js "external-link/test/index.js" 2>&1
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS packages/components/src/external-link/test/index.js

Test Suites: 1 passed, 1 total
Tests:       10 passed, 10 total
Snapshots:   0 total
Time:        3.136 s, estimated 5 s
Ran all test suites matching /external-link\/test\/index.js/i.
```

The three `Browserslist: … 17 months old` lines are emitted to stderr by `browserslist` (the `caniuse-lite` data shipped in `node_modules` is stale); they are informational and do not affect the result. The only run-to-run-variable line is `Time` (observed `Time: 3.136 s, estimated 5 s` on the first run and `Time: 3.104 s` on an immediate re-run — Jest drops the `estimated` suffix once it has a cached timing); `PASS`, `Test Suites: 1 passed, 1 total`, and `Tests: 10 passed, 10 total` are identical on every run.

**Command + verbatim output — (5) context-dependence within Jest itself.** Importing `@automattic/i18n-utils` at top level in a **node**-environment test (the `import-i18n-node.js` probe created above, which has no jsdom docblock) throws from a transitive dependency. Complete, unedited output:

```bash
$ node_modules/.bin/jest -c=packages/components/jest.config.js packages/components/blitzy_probe/test/import-i18n-node.js 2>&1
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
FAIL packages/components/blitzy_probe/test/import-i18n-node.js
  ● Test suite failed to run

    Trying to initialize the configuration outside of a browser context.

      16 |  */
      17 | if ( 'undefined' === typeof window ) {
    > 18 | 	throw new Error( 'Trying to initialize the configuration outside of a browser context.' );
         | 	      ^
      19 | }
      20 |
      21 | if ( ! window.configData ) {

      at Object.<anonymous> (../calypso-config/src/index.ts:18:8)
      at Object.require (../i18n-utils/src/utils.ts:1:1)
      at Object.require (../i18n-utils/src/index.ts:16:1)
      at Object.require (blitzy_probe/test/import-i18n-node.js:1:1)

Test Suites: 1 failed, 1 total
Tests:       0 total
Snapshots:   0 total
Time:        0.927 s
Ran all test suites matching /packages\/components\/blitzy_probe\/test\/import-i18n-node.js/i.
```

This is because `@automattic/calypso-config/src/index.ts` throws when `typeof window === 'undefined'` — a concrete "passes in jsdom / fails in node" case that explains why `external-link`'s test needs the jsdom docblock.

**Cleanup (removes the Q3 probes):**

```bash
$ rm -rf packages/components/blitzy_probe
```

**Stability.** Runs 1 and 2 were identical: the resolver and in-Jest `require.resolve` both returned `packages/i18n-utils/src/index.ts`; plain Node returned `MODULE_NOT_FOUND` both times.

**`file:line` evidence.**
- Custom resolver — `packages/calypso-jest/src/module-resolver.js`: `L16` `enhancedResolve.create.sync({`, `L17` `extensions: ['.json','.js','.jsx','.ts','.tsx']`, `L18` `mainFields: ['calypso:src','main']`, `L19` `conditionNames: ['calypso:src','node','require']`, `L22`–`L23` the exported `function(request, options){ return resolver(options.basedir, request)… }`.
- `packages/i18n-utils/package.json`: `main` `L7` = `dist/cjs/index.js`; `module` `L8` = `dist/esm/index.js`; `calypso:src` `L9` = `src/index.ts`.
- `packages/components/package.json`: `name` `L2` = `@automattic/components`; `main` `L8` = `dist/cjs/index.js`; `calypso:src` `L10` = `src/index.ts`; dependency `L34` = `"@automattic/i18n-utils": "workspace:^"`.
- `packages/components/jest.config.js:L2` → `preset: '../../test/packages/jest-preset.js'` (so components runs in the packages suite, env `node`).
- Transitive throw — `packages/calypso-config/src/index.ts:L17` (`if ( 'undefined' === typeof window ) {`) and `:L18` (`throw new Error( 'Trying to initialize the configuration outside of a browser context.' );`).

**Rationale.** The resolver's `mainFields`/`conditionNames` put `calypso:src` first, so all monorepo packages load untranspiled TypeScript source under Jest with no pre-build step. Production/Node resolution uses `main` (`dist/cjs/index.js`), which for many monorepo packages is never built. Hence the *same* import points at a real file under Jest but an absent file under Node — the second core reason behavior diverges by execution context. The bonus case (5) shows the divergence can even occur *within* Jest, between node-env and jsdom-env test files, when a transitive dependency requires `window`.

---

## Q4 — Where does the test infrastructure override import paths, and does the same import resolve differently by context?

**Direct answer.** Yes — the same import resolves to different files by context. There are two override layers. (a) A per-suite **`moduleNameMapper`** redirects `@automattic/calypso-config`: to `<rootDir>/server/config/index.js` in **client** (rootDir = `client`) and to `<rootDir>/client/server/config/index.js` in **integration**, and to `calypso/server/config` in **server**. (b) The repo-wide **custom resolver** redirects *all* internal-package imports to their `calypso:src` source. As a result, `@automattic/calypso-config` resolves to `packages/calypso-config/src/index.ts` in the **packages** context (no mapper → custom resolver → source), but to `client/server/config/index.js` in the **client/server/integration** contexts (mapper).

**Command + verbatim output — the mapper targets per context:**

```bash
$ for cfg in test/client/jest.config.js test/server/jest.config.js test/integration/jest.config.js; do echo "===== $cfg ====="; node_modules/.bin/jest --showConfig -c=$cfg 2>/dev/null | grep -A1 "@automattic/calypso-config" | grep -vE "^--$"; done
===== test/client/jest.config.js =====
          "^@automattic/calypso-config$",
          "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js"
===== test/server/jest.config.js =====
          "^@automattic/calypso-config$",
          "calypso/server/config"
          "^@automattic/calypso-config/(.*)$",
          "calypso/server/config/$1"
===== test/integration/jest.config.js =====
          "^@automattic/calypso-config$",
          "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js"
```

**Reproducible probe setup** — an identical `resolve-cfg.js` probe placed in each suite's discovery path (packages/client/server use the base `testMatch` `<rootDir>/**/test/*.[jt]s?(x)`; the integration suite matches `<rootDir>/client/**/integration/*.[jt]s`). All probes are removed at the end of this section:

```bash
$ mkdir -p packages/components/blitzy_probe/test client/blitzy_probe/test \
    client/server/blitzy_probe/test client/blitzy_probe_integration/integration
$ PROBE='test( "resolve @automattic/calypso-config", () => {
	console.log( "PROBE|require.resolve(@automattic/calypso-config) = " + require.resolve( "@automattic/calypso-config" ) );
	expect( true ).toBe( true );
} );'
$ echo "$PROBE" > packages/components/blitzy_probe/test/resolve-cfg.js        # packages (components)
$ echo "$PROBE" > client/blitzy_probe/test/resolve-cfg.js                     # client
$ echo "$PROBE" > client/server/blitzy_probe/test/resolve-cfg.js              # server
$ echo "$PROBE" > client/blitzy_probe_integration/integration/resolve-cfg.js  # integration
```

**Command + verbatim output — the same import resolved inside real Jest tests, in all four contexts** (`require.resolve('@automattic/calypso-config')`; filtered with `grep -oE 'PROBE\|.*'` to the single line each test logs — the grep is part of the command shown):

```bash
# PACKAGES (components) — no mapper -> custom resolver -> source:
$ node_modules/.bin/jest -c=packages/components/jest.config.js --silent=false "blitzy_probe/test/resolve-cfg" 2>&1 | grep -oE 'PROBE\|.*'
PROBE|require.resolve(@automattic/calypso-config) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/packages/calypso-config/src/index.ts

# CLIENT — mapper -> client/server/config/index.js:
$ TZ=UTC node_modules/.bin/jest -c=test/client/jest.config.js --silent=false "blitzy_probe/test/resolve-cfg" 2>&1 | grep -oE 'PROBE\|.*'
PROBE|require.resolve(@automattic/calypso-config) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js

# SERVER — mapper (calypso/server/config) -> client/server/config/index.js:
$ node_modules/.bin/jest -c=test/server/jest.config.js --silent=false "blitzy_probe/test/resolve-cfg" 2>&1 | grep -oE 'PROBE\|.*'
PROBE|require.resolve(@automattic/calypso-config) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js

# INTEGRATION — mapper -> client/server/config/index.js:
$ node_modules/.bin/jest -c=test/integration/jest.config.js --silent=false "resolve-cfg" 2>&1 | grep -oE 'PROBE\|.*'
PROBE|require.resolve(@automattic/calypso-config) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js
```

**Complete, unedited integration-context run** (the full Jest output behind the integration line above, including the `jest-haste-map` duplicate-mock notices and the Browserslist stderr warning that Jest prints before the run):

```text
$ node_modules/.bin/jest -c=test/integration/jest.config.js --silent=false "resolve-cfg" 2>&1
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
PASS client/blitzy_probe_integration/integration/resolve-cfg.js
  ● Console

    console.log
      PROBE|require.resolve(@automattic/calypso-config) = /tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/client/server/config/index.js

      at Object.log (client/blitzy_probe_integration/integration/resolve-cfg.js:2:10)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.5 s, estimated 1 s
Ran all test suites matching /resolve-cfg/i.
```

The identical specifier `@automattic/calypso-config` resolves to `packages/calypso-config/src/index.ts` in the **packages** context but `client/server/config/index.js` in the **client**, **server**, and **integration** contexts. (The `jest-haste-map: duplicate manual mock` notices come from `packages/plans-grid-next` shipping **three** copies of the same mock file — `packages/plans-grid-next/src/__mocks__/wpcom-proxy-request.js`, `packages/plans-grid-next/dist/cjs/__mocks__/wpcom-proxy-request.js`, and `packages/plans-grid-next/dist/esm/__mocks__/wpcom-proxy-request.js`; they are informational and unrelated to the resolution result.)

The **resolution target** (`…/client/server/config/index.js`), the `PASS` line, and the `Test Suites: 1 passed, 1 total` / `Tests: 1 passed, 1 total` / `Snapshots: 0 total` counts are identical on every run — that is the actual Q4 answer, and it is stable. The transcript above is one **warm-haste-map** run. Two things *do* vary run-to-run and must be reported honestly: (i) the `Time` line, and (ii) the **order and pairing of the two `duplicate manual mock` notices**. Because three files share the `wpcom-proxy-request` mock name, Jest emits a notice for each of the two *adjacent* pairs it encounters while walking its haste map, and that walk order is not stable across runs — so the same unchanged input prints either `{src, dist/cjs}` + `{dist/cjs, dist/esm}` or `{src, dist/esm}` + `{dist/esm, dist/cjs}`. The repo-wide scan that surfaces these copies comes from `test/integration/jest.config.js:L6` (`rootDir: '../..'`), which makes the integration haste map cover `packages/plans-grid-next`.

**Run-to-run variance (honest reproduction).** Running the *same unchanged* probe repeatedly and reducing each run to the ordered list of the four mock-path fragments (two per notice) shows the pairing is not stable. Across repeated 30-run batches (warm haste map, no cache clearing) the `{src, dist/cjs}` / `{dist/cjs, dist/esm}` pairing (call it *variant A*, matching the transcript above) is always the large majority, while the `{src, dist/esm}` / `{dist/esm, dist/cjs}` pairing (*variant B*) always recurs as a minority. The exact split itself fluctuates from batch to batch — four independent 30-run batches gave variant-A : variant-B counts of 26:4, 26:4, 29:1, and 24:6 — so the distribution is genuinely non-deterministic rather than a fixed ratio. One such batch:

```bash
$ for i in $(seq 1 30); do \
    node_modules/.bin/jest -c=test/integration/jest.config.js --silent=false "resolve-cfg" 2>&1 \
      | grep -E 'wpcom-proxy-request\.js' \
      | sed -E 's|.*/plans-grid-next/([a-z/]+)/__mocks__.*|\1|' | paste -sd',' -; \
  done | sort | uniq -c | sort -rn
     26 src,dist/cjs,dist/cjs,dist/esm
      4 src,dist/esm,dist/esm,dist/cjs
```

Clearing the haste map before each run (so it is rebuilt every time) makes the rebuild-order sensitivity even more visible; the split again fluctuates but keeps the same shape — three independent 20-run fresh-cache batches gave variant-A : variant-B counts of 17:3, 16:4, and 16:4 — and, unlike the warm case, every one of those fresh builds additionally emits a `Haste module naming collision` notice for `@automattic/fingerprintjs` (present in all 20 runs of each batch) that a warm cache suppresses. One such batch:

```bash
$ for i in $(seq 1 20); do \
    node_modules/.bin/jest --clearCache -c=test/integration/jest.config.js >/dev/null 2>&1; \
    node_modules/.bin/jest -c=test/integration/jest.config.js --silent=false "resolve-cfg" 2>&1 \
      | grep -E 'wpcom-proxy-request\.js' \
      | sed -E 's|.*/plans-grid-next/([a-z/]+)/__mocks__.*|\1|' | paste -sd',' -; \
  done | sort | uniq -c | sort -rn
     17 src,dist/cjs,dist/cjs,dist/esm
      3 src,dist/esm,dist/esm,dist/cjs
```

The alternate pairing (*variant B*) prints these two notices verbatim in place of the two shown in the transcript above:

```text
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/packages/plans-grid-next/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/packages/plans-grid-next/dist/esm/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/packages/plans-grid-next/dist/esm/__mocks__/wpcom-proxy-request.js
    * <rootDir>/packages/plans-grid-next/dist/cjs/__mocks__/wpcom-proxy-request.js
```

And the `@automattic/fingerprintjs` collision notice emitted only on a fresh (rebuilt) haste map, verbatim:

```text
jest-haste-map: Haste module naming collision: @automattic/fingerprintjs
  The following files share their name; please adjust your hasteImpl:
    * <rootDir>/packages/fingerprintjs/package.json
    * <rootDir>/packages/fingerprintjs/dist/esm/package.json
```

Across all of these runs the substantive result never changed: `require.resolve('@automattic/calypso-config')` always resolved to `…/client/server/config/index.js`, and the suite always reported `PASS` with `Tests: 1 passed, 1 total`. Only the incidental `Time` value and the haste-map notice order/pairing (and, on a cold cache, the extra `@automattic/fingerprintjs` collision line) differ between runs — none of which affects the resolution the question asks about. (`file:line` — the three shared mock files: `packages/plans-grid-next/src/__mocks__/wpcom-proxy-request.js`, `packages/plans-grid-next/dist/cjs/__mocks__/wpcom-proxy-request.js`, `packages/plans-grid-next/dist/esm/__mocks__/wpcom-proxy-request.js`; the collision files: `packages/fingerprintjs/package.json`, `packages/fingerprintjs/dist/esm/package.json`; the repo-wide scan: `test/integration/jest.config.js:L6`.)

**Cleanup (removes the Q4 probes):**

```bash
$ rm -rf packages/components/blitzy_probe client/blitzy_probe \
    client/server/blitzy_probe client/blitzy_probe_integration
```

**Command + verbatim output — why `calypso/server/config` (server mapper) equals `client/server/config` physically:**

```bash
$ node -e "console.log('client workspace name =', require('./client/package.json').name)"
client workspace name = calypso

$ ls -la client/server/config/index.js
-rw-r--r-- 1 root root 524 Jul  8 04:22 client/server/config/index.js

$ node -e "const p=require('./packages/calypso-config/package.json'); console.log('calypso-config main:',p.main,'| calypso:src:',p['calypso:src']);" ; ls -la packages/calypso-config/src/index.ts
calypso-config main: dist/cjs/index.js | calypso:src: src/index.ts
-rw-r--r-- 1 root root 3133 Jul  8 04:22 packages/calypso-config/src/index.ts
```

The `client` workspace is named `calypso`, so the bare specifier `calypso/server/config` (server mapper) resolves into the client workspace directory — making `calypso/server/config` and `client/server/config` the **same physical location**. Both target files exist (`client/server/config/index.js` = 524 bytes; `packages/calypso-config/src/index.ts` = 3133 bytes).

**Stability.** Runs 1 and 2 produced identical resolutions in all four contexts (packages → `packages/calypso-config/src/index.ts`; client, server, and integration → `client/server/config/index.js`).

**`file:line` evidence.**
- `test/client/jest.config.js:L11` → `'^@automattic/calypso-config$': '<rootDir>/server/config/index.js'` (rootDir = `../../client`), `L12` react-markdown mapper, `L20` `setupFiles: ['jest-canvas-mock']`, `L21` `setupFilesAfterEnv`.
- `test/server/jest.config.js:L10`–`L11` → `'^@automattic/calypso-config$': 'calypso/server/config'` plus `'^@automattic/calypso-config/(.*)$': 'calypso/server/config/$1'` (rootDir = `../../client/server`).
- `test/integration/jest.config.js:L3` → `'^@automattic/calypso-config$'` mapped to `<rootDir>/client/server/config/index.js` (rootDir = `../..`), `L7` env node, `L8` resolver.
- `client/package.json:L2` → `name: "calypso"`.
- `packages/calypso-config/package.json`: `name` `L2` = `@automattic/calypso-config`; `main` `L9` = `dist/cjs/index.js`; `module` `L10` = `dist/esm/index.js`; `calypso:src` `L11` = `src/index.ts`.
- Resolver `calypso:src` priority again: `packages/calypso-jest/src/module-resolver.js:L18`–`L19`.
- The bare `calypso/server/config` specifier resolves into the `calypso` (client) workspace, so it targets the same `client/server/config/index.js`; `client/server/config/index.js` = 524 bytes, `packages/calypso-config/src/index.ts` = 3133 bytes.

**Rationale.** In client/server/integration the mapper deliberately swaps the published config package for the in-repo server config module (client reaches it via an absolute `<rootDir>/server/config/index.js` path; server reaches the same physical file via the package-name specifier `calypso/server/config`). In the packages context there is no such mapper, so the custom resolver loads the actual `@automattic/calypso-config` source. A test importing `@automattic/calypso-config` therefore executes **different code** depending on which suite runs it — the third core mechanism for context-dependent pass/fail.


---

## Q5 & Q7 — When a test runs, what loads first? (And verifying the order by observing accessibility at distinct points.)

**Direct answer.** For each test file the initialization order is:

1. **The `testEnvironment` is constructed** — a jsdom environment creates `window`/`document` here (STAGE1).
2. **`setupFiles` run** (e.g., `jest-canvas-mock`) — the `jest` object exists, but `expect`/`beforeAll`/`test` do **not** yet (STAGE2).
3. **The test framework is installed.**
4. **`setupFilesAfterEnv` run** (e.g., `setup-test-framework.js`) — now `expect`/`beforeAll`/`test` **do** exist, so setup can call `jest.fn`, `beforeAll`, etc. (STAGE4).
5. **The test-file module scope executes** (STAGE5).
6. **`beforeAll` hooks fire** (STAGE5b).
7. **The test body runs** (STAGE6).

**Probe methodology.** A temporary, untracked probe directory created **inside the repo** (`blitzy_initorder_probe/`, deleted at the end of this section) defines two custom `testEnvironment`s that `extend` the real `jest-environment-node` / `jest-environment-jsdom` and log `typeof this.global.window/…` from their constructor (STAGE1); a `setupFiles` script logs at STAGE2, a `setupFilesAfterEnv` script logs at STAGE4, and the test file logs at module scope (STAGE5), in a `beforeAll` hook (STAGE5b), and inside the test body (STAGE6). Create it exactly as follows:

```bash
$ mkdir -p blitzy_initorder_probe/test

$ cat > blitzy_initorder_probe/env-node.js <<'EOF'
const Base = require( 'jest-environment-node' ).default;
module.exports = class extends Base {
	constructor( ...args ) {
		super( ...args );
		console.log(
			'PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=' +
				typeof this.global.window +
				' typeof this.global.matchMedia=' +
				typeof this.global.matchMedia
		);
	}
};
EOF

$ cat > blitzy_initorder_probe/env-jsdom.js <<'EOF'
const Base = require( 'jest-environment-jsdom' ).default;
module.exports = class extends Base {
	constructor( ...args ) {
		super( ...args );
		console.log(
			'PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=' +
				typeof this.global.window +
				' typeof this.global.document=' +
				typeof this.global.document
		);
	}
};
EOF

$ cat > blitzy_initorder_probe/setup-file.js <<'EOF'
console.log(
	'PROBE|STAGE2|setupFiles | typeof jest=' + typeof jest +
		' typeof expect=' + typeof expect +
		' typeof beforeAll=' + typeof beforeAll +
		' typeof test=' + typeof test +
		' typeof window=' + typeof window
);
EOF

$ cat > blitzy_initorder_probe/setup-after.js <<'EOF'
console.log(
	'PROBE|STAGE4|setupFilesAfterEnv | typeof jest=' + typeof jest +
		' typeof expect=' + typeof expect +
		' typeof beforeAll=' + typeof beforeAll +
		' typeof test=' + typeof test +
		' typeof window=' + typeof window
);
EOF

$ cat > blitzy_initorder_probe/test/order.test.js <<'EOF'
console.log( 'PROBE|STAGE5|test-file module scope | typeof expect=' + typeof expect + ' typeof window=' + typeof window );
beforeAll( () => {
	console.log( 'PROBE|STAGE5b|beforeAll hook fired' );
} );
test( 'init order', () => {
	console.log( 'PROBE|STAGE6|inside test() body' );
	expect( true ).toBe( true );
} );
EOF

$ cat > blitzy_initorder_probe/jest.node.config.js <<'EOF'
const path = require( 'path' );
module.exports = {
	rootDir: __dirname,
	testEnvironment: path.join( __dirname, 'env-node.js' ),
	setupFiles: [ path.join( __dirname, 'setup-file.js' ) ],
	setupFilesAfterEnv: [ path.join( __dirname, 'setup-after.js' ) ],
	testMatch: [ path.join( __dirname, 'test', 'order.test.js' ) ],
	transform: {},
};
EOF

$ cat > blitzy_initorder_probe/jest.jsdom.config.js <<'EOF'
const path = require( 'path' );
module.exports = {
	rootDir: __dirname,
	testEnvironment: path.join( __dirname, 'env-jsdom.js' ),
	setupFiles: [ path.join( __dirname, 'setup-file.js' ) ],
	setupFilesAfterEnv: [ path.join( __dirname, 'setup-after.js' ) ],
	testMatch: [ path.join( __dirname, 'test', 'order.test.js' ) ],
	transform: {},
};
EOF
```

(Placing the probe in-repo — rather than in `/tmp` as an earlier iteration did — lets `require('jest-environment-node')` resolve from the repo's own `node_modules` by the normal upward lookup, avoiding the `Cannot find module 'jest-environment-node'` failure a `/tmp`-located module hits; `transform: {}` disables Babel so the plain-CJS probe files run as-is.)

**Command + verbatim output — NODE environment:**

```bash
$ node_modules/.bin/jest -c=blitzy_initorder_probe/jest.node.config.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=undefined typeof this.global.matchMedia=undefined
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=undefined
PROBE|STAGE4|setupFilesAfterEnv | typeof jest=object typeof expect=function typeof beforeAll=function typeof test=function typeof window=undefined
PROBE|STAGE5|test-file module scope | typeof expect=function typeof window=undefined
PROBE|STAGE5b|beforeAll hook fired
PROBE|STAGE6|inside test() body
```

**Command + verbatim output — JSDOM environment:**

```bash
$ node_modules/.bin/jest -c=blitzy_initorder_probe/jest.jsdom.config.js --silent=false 2>&1 | grep -oE 'PROBE\|.*'
PROBE|STAGE1|testEnvironment constructor | typeof this.global.window=object typeof this.global.document=object
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=object
PROBE|STAGE4|setupFilesAfterEnv | typeof jest=object typeof expect=function typeof beforeAll=function typeof test=function typeof window=object
PROBE|STAGE5|test-file module scope | typeof expect=function typeof window=object
PROBE|STAGE5b|beforeAll hook fired
PROBE|STAGE6|inside test() body
```

The key transition is **STAGE2 → STAGE4**: `expect` is `undefined` in `setupFiles` but `function` in `setupFilesAfterEnv`. That is the empirical proof that **the framework is installed between `setupFiles` and `setupFilesAfterEnv`**. Note also that in the jsdom run `window` is already `object` at STAGE1 (environment construction), confirming the Q6 timing claim.

**Command + verbatim output — stability across 2 runs of the STAGE2→STAGE4 transition (node env):**

```bash
$ for i in 1 2; do echo "--- RUN $i ---"; node_modules/.bin/jest -c=blitzy_initorder_probe/jest.node.config.js --silent=false 2>&1 | grep -oE 'PROBE\|STAGE[24]\|.*'; done
--- RUN 1 ---
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=undefined
PROBE|STAGE4|setupFilesAfterEnv | typeof jest=object typeof expect=function typeof beforeAll=function typeof test=function typeof window=undefined
--- RUN 2 ---
PROBE|STAGE2|setupFiles | typeof jest=object typeof expect=undefined typeof beforeAll=undefined typeof test=undefined typeof window=undefined
PROBE|STAGE4|setupFilesAfterEnv | typeof jest=object typeof expect=function typeof beforeAll=function typeof test=function typeof window=undefined
```

Identical both runs.

**Command + verbatim output — tying the order to the real client suite** (which setup files run, and in what role):

```bash
$ node_modules/.bin/jest --showConfig -c=test/client/jest.config.js 2>/dev/null | grep -A1 "\"setupFiles\"\|\"setupFilesAfterEnv\""
      "setupFiles": [
        "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/node_modules/jest-canvas-mock/lib/index.js"
--
      "setupFilesAfterEnv": [
        "/tmp/blitzy/wp-calypso/blitzy-fc213574-ff44-4dec-b754-98f45e636026_651c8e/test/client/setup-test-framework.js"
```

So the real client order is: **environment → `jest-canvas-mock` (setupFiles) → framework install → `setup-test-framework.js` (setupFilesAfterEnv) → test file.**

**Cleanup (removes the init-order probe):**

```bash
$ rm -rf blitzy_initorder_probe
```

**`file:line` evidence.**
- `test/client/jest.config.js:L20` → `setupFiles: ['jest-canvas-mock']`; `:L21` → `setupFilesAfterEnv: ['<rootDir>/../test/client/setup-test-framework.js']`.
- Base `setupFilesAfterEnv`: `packages/calypso-jest/jest-preset.js:L10`.
- Apps mirror: `test/apps/jest-preset.js:L11` (`setupFiles: ['jest-canvas-mock']`), `L13` (`setupFilesAfterEnv` reusing the client framework).

**Corroboration (labeled).** The official Jest documentation corroborates this ordering: `setupFiles` scripts are executed before `setupFilesAfterEnv` and before the test code itself (and before the test framework is installed), whereas for `setupFilesAfterEnv` "having the test framework installed makes Jest globals, `jest` object and `expect` accessible in the modules" ([jestjs.io/docs/configuration](https://jestjs.io/docs/configuration)). This is corroboration only; the STAGE2→STAGE4 transitions above are the empirical proof.

**Rationale.** Anything a test relies on at import time (module scope, STAGE5) must already be present by that point. Globals injected by `setupFilesAfterEnv` (e.g., `matchMedia`, `fetch`) are in place before the test module loads, but `expect`/`beforeAll` are **not** available in `setupFiles` (STAGE2). This ordering, combined with *which* setup file a given context loads (Q2), determines exactly what a test sees when it runs — and therefore whether it passes in one context and fails in another.

---

## Q8 — Read-only integrity

**Direct answer.** The investigation modified **no** existing repository file. Every temporary probe used in this report lived in a clearly-named, untracked throwaway directory inside the repo (`client/blitzy_probe/`, `packages/components/blitzy_probe/`, `client/server/blitzy_probe/`, `client/blitzy_probe_integration/`, and `blitzy_initorder_probe/`), and each was deleted at the end of its section. The **only** permanent change to the repository, relative to the baseline commit, is the addition of this single documentation file — which is committed and tracked normally (it is **not** untracked and **not** git-ignored). Once the document is committed, the working tree is completely clean.

**Command + verbatim output — the working tree is clean and the sole change vs. baseline is this document** (run after the document has been committed; `git status --porcelain` and `git diff --stat` print nothing, so their empty output is shown by the immediately following prompt):

```bash
$ git status --porcelain
$ git diff --stat
$ git diff --name-status be7e5cc641622d153040491fd5625c6cb83e12eb..HEAD
A	blitzy/documentation/wp-calypso_be7e5cc64162.md
$ git ls-files blitzy/documentation/wp-calypso_be7e5cc64162.md
blitzy/documentation/wp-calypso_be7e5cc64162.md
$ git check-ignore blitzy/documentation/wp-calypso_be7e5cc64162.md ; echo "exit=$?"
exit=1
```

`git status --porcelain` prints nothing (clean working tree); `git diff --stat` prints nothing (no unstaged modifications); `git diff --name-status be7e5cc641…..HEAD` shows exactly one line, `A blitzy/documentation/wp-calypso_be7e5cc64162.md`, proving the sole difference from baseline is the *addition* of this file (no `M`/`D` for any existing file); `git ls-files` shows the file is tracked; and `git check-ignore` returns nothing with exit code `1`, confirming the deliverable is **not** ignored by `.gitignore`. (This corrects the earlier revision, which showed the document as an *untracked* `?? blitzy/` entry — that was stale, because the deliverable is committed and therefore tracked.)

**Command + verbatim output — every temporary probe directory has been removed** (no residue left by the investigation):

```bash
$ for d in client/blitzy_probe packages/components/blitzy_probe client/server/blitzy_probe client/blitzy_probe_integration blitzy_initorder_probe blitzy_iso_probe; do [ -e "$d" ] && echo "PRESENT: $d" || echo "absent: $d"; done
absent: client/blitzy_probe
absent: packages/components/blitzy_probe
absent: client/server/blitzy_probe
absent: client/blitzy_probe_integration
absent: blitzy_initorder_probe
absent: blitzy_iso_probe
```

**Rationale.** The read-only mandate requires that observation not mutate any tracked file. Every probe lived only in a clearly-named throwaway directory that was created, executed for output capture, and then removed within its own section (see the "Cleanup" blocks under Q2, Q3, Q4, and Q5/Q7). The `git diff --name-status be7e5cc641…..HEAD` output above is the single authoritative check: it lists only `A blitzy/documentation/wp-calypso_be7e5cc64162.md`, so no pre-existing repository file was added to, modified, or deleted — the investigation is non-invasive and its clean-tree result is reproducible from the commands shown.

---

## Reproducibility note

Every command above was executed from the repository root under the canonical toolchain (Node v22.23.1, Yarn 4.0.2, Jest 29.7.0, jest-environment-jsdom 29.7.0 bundling jsdom 20.0.3). All `typeof` values and resolution results were confirmed stable across at least two runs. The only intentionally non-deterministic value is `crypto.randomUUID()` in the client and server contexts, which returns a fresh real UUID on each call (contrasted with the deterministic literal `'fake-uuid'` injected in the packages context); this is reported exactly as observed and is not normalized.

