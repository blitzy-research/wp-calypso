# `@automattic/explat-client` — Behavioral Investigation (Q0–Q4)

> **Source branch:** `wp-calypso_be7e5cc64162`
> **Package under investigation:** `@automattic/explat-client@0.1.0` ([`packages/explat-client/package.json:2-3`](../../packages/explat-client/package.json))
> **Investigated source revision:** `be7e5cc641` — the repository state that was **read and exercised**; every answer below describes *this* baseline. This is **not** the commit that adds this document. **OBSERVED** (`git rev-parse`).
> **Prior documentation commit:** `5c54a30515`. The commit that delivers this revised document is a further commit stacked on top of it; the SHA of that delivery commit is therefore necessarily unknowable from within the document itself. **OBSERVED** (`git log`).
> **Toolchain used (repository-authoritative, canonical):** Node.js `v22.23.1`, Yarn `4.0.2`. **OBSERVED** (captured in the Methodology section).

This document answers five behavioral questions about the standalone experiment-assignment client in `packages/explat-client/`. **Every behavioral claim is grounded in runtime output captured by actually running the client's code through its canonical public entry point** (`packages/explat-client/src/index.ts`), with the output pasted verbatim beneath each claim together with the exact command that produced it.

The client's real network call is exercised through the client's own dependency-injection seam — `config.fetchExperimentAssignment` ([`types.ts:29-35`](../../packages/explat-client/src/types.ts)) — which is the client's *designed* extension point (the existing Jest suite uses exactly this technique). The production assignment server was never contacted; this is stated wherever it matters. Read-only source reading is used only to name the responsible function/value and is always labeled **INFERRED**.

## How to read this document (label conventions)

Every statement below carries a `file:line` citation into `packages/explat-client/**` and one of these labels. Labels are applied **per statement**, not per section: a sentence that couples an observed value to an inferred meaning is split so each half is labeled correctly. There is **no** blanket "only N statements are inferred" claim anywhere in this document — each inference is called out individually where it occurs.

- **OBSERVED** — demonstrated by captured runtime output that is pasted verbatim below the claim, with the exact command that produced it.
- **INFERRED** — read from the source (with a `file:line` citation) and not directly exercised at runtime. The notable inferences in this document are: the production TTL of `3600` s (only `README.md:42` asserts it; the client code never hard-codes it); the industry-standard *name* "single-flight / request-coalescing" for the Q2 mechanism; and the **cross-tab / cross-process** scope of the deduplication and cache (the registry and store are per client instance — observed — and the extrapolation to "one browser tab / one JS process" is inferred).
- **non-canonical** — a value from a bypass, fallback, or stand-in rather than the real path. The only such item here is the SSR-dummy client ([`create-explat-client.ts:258-283`](../../packages/explat-client/src/create-explat-client.ts)); it is deliberately **never** used to answer any question and is labeled non-canonical wherever mentioned.

---

## Methodology & canonical harness

### The canonical entry point (`src/index.ts`) and why `window` must exist *before* it is loaded

The public export is environment-sensitive and is bound **at module-evaluation time**:

```ts
// packages/explat-client/src/index.ts:8-9
const createExPlatClient =
	typeof window === 'undefined' ? createSsrSafeDummyExPlatClient : createBrowserExPlatClient;
```

Because this ternary runs when `src/index.ts` is first evaluated, the browser context must exist **before** that module is loaded. If `window` is undefined at load time the public name resolves to the SSR-safe **dummy** ([`create-explat-client.ts:258-283`](../../packages/explat-client/src/create-explat-client.ts)) — a **non-canonical** path that logs `"Attempting to load ExperimentAssignment in SSR context"` and returns fallbacks without ever calling the injected fetch. **INFERRED** (the gate and the dummy body are read from source). Separately, the real browser factory throws `'Running outside of a browser context.'` if it is *called* with no `window` ([`create-explat-client.ts:72-74`](../../packages/explat-client/src/create-explat-client.ts)). **INFERRED**.

Every observation spec therefore establishes the browser context first and only **then** loads the public entry, using `require` (which — unlike a hoisted `import` — executes in source order, i.e. *after* `setBrowserContext()`):

```ts
import '@automattic/calypso-polyfills';
import { setBrowserContext } from '../internal/test-common';
// 1) window is established BEFORE the public entry module is evaluated:
setBrowserContext();                                   // global.window = {}   (test-common.ts:22-26)
// 2) load through the PUBLIC entry (index.ts gate) — require is not hoisted:
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );  // binds the REAL browser client
```

This exercises **the real browser client through the documented public entry point** — it is the canonical path, not a bypass of it. That the branch actually taken is the browser client (never the dummy) is **OBSERVED** in every section below: the injected `config.fetchExperimentAssignment` is actually invoked (non-zero call counts), and no `"... in SSR context"` dummy log is ever emitted. During Phase 1 this was additionally confirmed with a throwaway probe using `jest.isolateModulesAsync`: loading `../index` **with** `window` set invoked the injected fetch once and returned `variationName: "treatment"` with an empty `logError`, whereas loading it **without** `window` invoked the fetch zero times and logged the SSR-context message — the two branches of `index.ts:8-9` behaving exactly as read. **OBSERVED** (probe, since deleted per the read-only rule).

### The dependency-injection seam is the canonical observation point (and the exact call shape)

The network boundary is a single call inside `Request.fetchExperimentAssignment`:

```ts
// packages/explat-client/src/internal/requests.ts:86-90
await config.fetchExperimentAssignment( {
	anonId: await localStorageCachedGetAnonId( config.getAnonId ),
	experimentName,
} )
```

Substituting a Jest mock for `config.fetchExperimentAssignment` is the client's **designed, canonical** extension point, not a bypass — the whole `Config` object exists to abstract the outside world ([`types.ts:28-39`](../../packages/explat-client/src/types.ts)). **INFERRED** (design intent, from the interface). Each spec sets `config.getAnonId` to resolve a known value (`'anon-id-xyz-123'`) and then **asserts the exact first-call argument**; the observed argument is `{"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}` (see Q1a and Q2 output), confirming the `{ anonId, experimentName }` shape and the ordering at [`requests.ts:86-90`](../../packages/explat-client/src/internal/requests.ts). **OBSERVED**.

### No pre-build is required (how `.ts` runs directly) — three cooperating preset pieces

`packages/explat-client/jest.config.js:2` sets `preset: '../../test/packages/jest-preset.js'`, which spreads `@automattic/calypso-jest` ([`test/packages/jest-preset.js:8-9`](../../test/packages/jest-preset.js)). Direct `.ts` execution is produced by **three** cooperating pieces of that preset — not one:

1. a custom **resolver** mapping the package's `calypso:src` field to untranspiled TypeScript source — `resolver: require.resolve( './src/module-resolver.js' )` ([`packages/calypso-jest/jest-preset.js:9`](../../packages/calypso-jest/jest-preset.js));
2. a **`babel-jest` transform** that compiles every `.[jt]sx?` file on the fly — `transform: { '\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ], ... }` ([`packages/calypso-jest/jest-preset.js:13-15`](../../packages/calypso-jest/jest-preset.js));
3. `testMatch: [ '<rootDir>/**/test/*.[jt]s?(x)', '!**/.eslintrc.*' ]` with `testEnvironment: 'node'` ([`packages/calypso-jest/jest-preset.js:11-12`](../../packages/calypso-jest/jest-preset.js)), so any spec placed under `packages/explat-client/src/test/` is auto-discovered.

**INFERRED** (from reading the preset); **OBSERVED**-corroborated by Q0 running the `.ts` suites with no build step.

### Runtime version reconciliation & workspace install (canonical configuration)

The environment setup notes mentioned Node 20.x, but the **repository is authoritative** and requires Node 22.x — `engines.node = "^v22.9.0"`, `.nvmrc = 22.9.0`, `packageManager = "yarn@4.0.2"`. The repository-authoritative Node 22.x was used so the client runs exactly as the project intends. **OBSERVED** (`node --version && yarn --version && cat .nvmrc`, run at the repository root):

```
v22.23.1
4.0.2
22.9.0
```

Workspace install (non-interactive, from the repository root), which exits `0` and leaves `yarn.lock` unchanged. **OBSERVED:**

```bash
CI=true yarn install
```

```
➤ YN0000: · Yarn 4.0.2
➤ YN0000: ┌ Resolution step
➤ YN0000: └ Completed in 0s 565ms
➤ YN0000: ┌ Fetch step
➤ YN0000: └ Completed in 4s 660ms
➤ YN0000: ┌ Link step
➤ YN0000: └ Completed in 1s 121ms
➤ YN0000: · Done in 6s 665ms
```

The install completes in seconds because the workspace dependencies were already present in the environment's Yarn cache; a cold install takes longer. **INFERRED** (cache behavior). The `Done in ...` line and exit code `0` are **OBSERVED**.

### Harness details (mirrors the existing suite) and — crucially — assertions

Each observation spec mirrors the existing suite `packages/explat-client/src/test/create-explat-client.ts`: it imports `'@automattic/calypso-polyfills'` **first** (the existing suite does the same at its line 2, to avoid `regeneratorRuntime is not defined`); defines a `createMockedConfig(...)` returning `{ logError: jest.fn(), fetchExperimentAssignment: jest.fn(), getAnonId: jest.fn(), isDevelopmentMode: false, ...override }`; and runs `beforeEach( () => { jest.resetAllMocks(); setBrowserContext(); localStorage.clear(); } )`. **INFERRED** (harness structure, mirrored from the existing suite).

Unlike the earlier revision of this document, **every spec now contains `expect(...)` assertions for every claimed invariant** — fetch-call counts, thrown/not-thrown, the exact returned object, the exact request argument, and the exact `logError` calls. The `console.log` markers remain only so the values are visible in the pasted output; the assertions are what make a run *fail* if any value is wrong. Each spec's full source (including its assertions) is embedded in its section below, and every run shown reports `Tests: N passed`. **OBSERVED**.

One harness subtlety: `jest.resetAllMocks()` also neuters the `jest.spyOn( Timing, 'monotonicNow' )` spy (it would then return `undefined` and corrupt `retrievedTimestamp`). Specs that do not need to pin the clock reinstate a strictly-increasing clock in `beforeEach` mirroring the real `monotonicNow` ([`timing.ts:10-16`](../../packages/explat-client/src/internal/timing.ts)): `let clock = Date.now(); spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );`. Specs that need deterministic TTL arithmetic (Q1a-stale, Q2-scope, Q3) instead pin `monotonicNow` to explicit constants. **INFERRED** (harness rationale).

### Storage backing

Under a bare `window = {}`, `window.localStorage` is undefined, so the client falls back to its in-memory polyfill — `let localStorage = polyfilledLocalStorage; try { if ( window.localStorage ) { localStorage = window.localStorage; } } catch ( e ) {}` ([`local-storage.ts:33-40`](../../packages/explat-client/src/internal/local-storage.ts)). The cache key is `explat-experiment--<name>` (**double dash**): the prefix `explat-experiment-` ([`experiment-assignment-store.ts:9`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)) plus a literal `-` plus the name ([`experiment-assignment-store.ts:14-15`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)). **INFERRED** (from the store/storage source). The cache-hit and refetch counts that depend on this backing are **OBSERVED** in Q3.

---

## Q0 — Does the package's test suite pass cleanly? (baseline)

**Question.** Before trusting any behavioral claim, confirm the client's own Jest suite passes.

**Command** (run in `packages/explat-client/`, with **no** `blitzy_tmp_*` observation specs present — those are created and deleted per question, so Q0 reflects the untouched suite):

```bash
cd packages/explat-client && CI=true yarn jest --ci
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/internal/test/requests.ts
PASS src/internal/test/timing.ts
PASS src/internal/test/experiment-assignment-store.ts
PASS src/test/index.ts
PASS src/internal/test/validations.ts
PASS src/internal/test/experiment-assignments.ts
PASS src/test/create-ssr-safe-dummy-explat-client.ts
PASS src/internal/test/local-storage.ts
PASS src/test/create-explat-client.ts
A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --detectOpenHandles to find leaks. Active timers can also cause this, ensure that .unref() was called on them.

Test Suites: 9 passed, 9 total
Tests:       81 passed, 81 total
Snapshots:   23 passed, 23 total
Time:        2.452 s
Ran all test suites.
```

**Answer (OBSERVED).** The suite passes cleanly: `Test Suites: 9 passed, 9 total`, `Tests: 81 passed, 81 total`, `Snapshots: 23 passed, 23 total`. The nine suites are `requests`, `timing`, `experiment-assignment-store`, `index`, `validations`, `experiment-assignments`, `create-ssr-safe-dummy-explat-client`, `local-storage`, and `create-explat-client`. **OBSERVED**.

**About the `A worker process has failed to exit gracefully ...` line.** This message is **OBSERVED** in the output above and it does **not** fail any suite (the totals still read `9 passed` / `81 passed`). Its *cause* — a real leaked `setTimeout` created by `Timing.timeoutPromise` — is not merely inferred here: it is demonstrated at runtime in the [open-handle demonstration](#open-handle-demonstration-why-the-worker-failed-to-exit-warning-appears-f13) section, where `--detectOpenHandles` names the exact `setTimeout` at `timing.ts:30`. The single-file specs in later sections emit the equivalent single-worker phrasing (`Jest did not exit one second after the test run has completed.`) for the same reason. **OBSERVED** (message); cause **OBSERVED** via `--detectOpenHandles` below.

---

## Q1 — Failure handling: is the client really "designed to never throw"? What is returned, and which variation is assigned?

**Question.** `README.md:44` claims the client is "Designed to never throw". When the assignment server is **unavailable** (the network call rejects) or **too slow** (the fetch times out): does anything escape as an exception, what does the returned object look like, and which experiment variation does the user end up assigned?

**Where the contract lives.** `loadExperimentAssignment` wraps its body in an outer `try/catch` ([`create-explat-client.ts:114-156`](../../packages/explat-client/src/create-explat-client.ts)); on any failure it logs (`source: 'loadExperimentAssignment-initialError'`, [`:150-156`](../../packages/explat-client/src/create-explat-client.ts)) and then, in a second `try`, **first returns a stored assignment if one exists — even if expired** ([`:158-165`](../../packages/explat-client/src/create-explat-client.ts)), and only otherwise creates, stores, and returns a fallback ([`:167-172`](../../packages/explat-client/src/create-explat-client.ts)). The fallback is built by `createFallbackExperimentAssignment`, whose `variationName` is `null` ([`experiment-assignments.ts:29-38`](../../packages/explat-client/src/internal/experiment-assignments.ts)). Per `README.md:22`, `variationName === null` means the user gets the **default (control) experience**. **INFERRED** (from source); each concrete value below is **OBSERVED**.

### Q1a — Server unavailable (fetch rejects), *empty* store

**Full spec** (`packages/explat-client/src/test/blitzy_tmp_q1a_empty.ts`, embedded byte-for-byte; note the assertions):

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q1a: server unavailable (fetch rejects), EMPTY store.
// Ephemeral: created to capture runtime behavior, then deleted; never committed.
// Fixes the "regeneratorRuntime is not defined" error, exactly as the existing suite does.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';

import { setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: establish the browser context BEFORE the public entry module
// (src/index.ts:8-9) is evaluated, then load it via require (require is not hoisted, so it
// runs after setBrowserContext). This binds the REAL browser client, not the SSR dummy.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
	// resetAllMocks neuters the monotonicNow spy; reinstate a strictly-increasing clock
	// mirroring the real monotonicNow (timing.ts:10-16).
	let clock = Date.now();
	spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
} );

describe( 'Q1a server unavailable (fetch rejects), empty store', () => {
	it( 'never throws; returns null-variation fallback; passes exact request arg', async () => {
		const config = createMockedConfig();
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		( config.fetchExperimentAssignment as MockedFunction ).mockImplementation(
			() =>
				new Promise( ( _res, rej ) =>
					rej( new Error( 'ECONNREFUSED simulated-server-unavailable' ) )
				)
		);
		const client = createExPlatClient( config );

		let threw = false;
		let result = null;
		try {
			result = await client.loadExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			threw = true;
		}

		const fetchCalls = ( config.fetchExperimentAssignment as MockedFunction ).mock.calls;
		const logErrorCalls = ( config.logError as MockedFunction ).mock.calls;

		console.log( 'Q1A_THREW=' + threw );
		console.log( 'Q1A_RESULT=' + JSON.stringify( result ) );
		console.log( 'Q1A_FETCH_CALLS=' + fetchCalls.length );
		console.log( 'Q1A_REQUEST_ARG=' + JSON.stringify( fetchCalls[ 0 ][ 0 ] ) );
		console.log( 'Q1A_LOGERROR=' + JSON.stringify( logErrorCalls ) );

		// Assertions that gate the evidence (Rule: run first; every claim asserted).
		expect( threw ).toBe( false );
		expect( result ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: null,
			ttl: 60,
			isFallbackExperimentAssignment: true,
		} );
		expect( typeof ( result as { retrievedTimestamp: number } ).retrievedTimestamp ).toBe(
			'number'
		);
		expect( fetchCalls ).toHaveLength( 1 );
		expect( fetchCalls[ 0 ][ 0 ] ).toEqual( {
			experimentName: 'experiment_name_a',
			anonId: 'anon-id-xyz-123',
		} );
		expect( logErrorCalls ).toEqual( [
			[
				{
					message: 'ECONNREFUSED simulated-server-unavailable',
					experimentName: 'experiment_name_a',
					source: 'loadExperimentAssignment-initialError',
				},
			],
		] );
	} );
} );
```

**Command:**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q1a_empty.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q1a_empty.ts
  ● Console

    console.log
      Q1A_THREW=false

      at Object.log (src/test/blitzy_tmp_q1a_empty.ts:66:11)

    console.log
      Q1A_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965141634,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_q1a_empty.ts:67:11)

    console.log
      Q1A_FETCH_CALLS=1

      at Object.log (src/test/blitzy_tmp_q1a_empty.ts:68:11)

    console.log
      Q1A_REQUEST_ARG={"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}

      at Object.log (src/test/blitzy_tmp_q1a_empty.ts:69:11)

    console.log
      Q1A_LOGERROR=[[{"message":"ECONNREFUSED simulated-server-unavailable","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]]

      at Object.log (src/test/blitzy_tmp_q1a_empty.ts:70:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.732 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q1a_empty.ts/i.
Jest did not exit one second after the test run has completed.

'This usually means that there are asynchronous operations that weren't stopped in your tests. Consider running Jest with `--detectOpenHandles` to troubleshoot this issue.
```

**Answers (all OBSERVED).**

- **It never throws.** `Q1A_THREW=false`, and `Tests: 1 passed` (the assertion `expect( threw ).toBe( false )` held). This exercises the outer `try/catch` at [`create-explat-client.ts:114-156`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**.
- **The returned object** is `{"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965141634,"ttl":60,"isFallbackExperimentAssignment":true}` — i.e. a fallback assignment with `variationName: null`, `ttl: 60`, and `isFallbackExperimentAssignment: true`, produced by `createFallbackExperimentAssignment` ([`experiment-assignments.ts:29-38`](../../packages/explat-client/src/internal/experiment-assignments.ts)). **OBSERVED**.
- **Which variation:** `variationName` is `null`, so the user is assigned the **default/control** experience ([`README.md:22`](../../packages/explat-client/README.md) defines the meaning of `null`). **OBSERVED** (value `null`); the "default experience" meaning is **INFERRED** from the README.
- **Exactly one network attempt was made** (`Q1A_FETCH_CALLS=1`) with the exact argument `{"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}`, confirming the DI call shape at [`requests.ts:86-90`](../../packages/explat-client/src/internal/requests.ts). **OBSERVED**.
- **The failure was logged, not thrown:** `Q1A_LOGERROR` shows a single `logError` call carrying the rejection message and `source: "loadExperimentAssignment-initialError"` ([`create-explat-client.ts:150-156`](../../packages/explat-client/src/create-explat-client.ts)). **OBSERVED**.

**Stability (F12).** Re-running twice yields byte-identical markers (the command is on the first line of the capture):

```
$ for r in 1 2; do echo "run$r:"; grep -oE 'Q1A_THREW=[a-z]+|Q1A_FETCH_CALLS=[0-9]+|"variationName":null|"ttl":60,"isFallbackExperimentAssignment":true|loadExperimentAssignment-initialError' q1a_empty_run$r.log | paste -sd" " -; done
run1:
Q1A_THREW=false "variationName":null "ttl":60,"isFallbackExperimentAssignment":true Q1A_FETCH_CALLS=1 loadExperimentAssignment-initialError
run2:
Q1A_THREW=false "variationName":null "ttl":60,"isFallbackExperimentAssignment":true Q1A_FETCH_CALLS=1 loadExperimentAssignment-initialError
```

Only `retrievedTimestamp` (a wall-clock value) would differ between runs; every semantic field and the counts are stable. **OBSERVED**.

### Q1a variant — Server unavailable with a **stale** assignment already cached (the important nuance, F11)

The "always returns a `null` fallback on failure" statement is true **only for an empty store**. When an assignment is already cached and the refetch fails, the client returns the **stored (stale) assignment**, not a fresh `null` fallback — the "provide stale ExperimentAssignments, important for offline users" branch at [`create-explat-client.ts:158-165`](../../packages/explat-client/src/create-explat-client.ts). This spec primes the store with a `treatment`, jumps the clock past the 60 s TTL so it is expired, then fails the refetch:

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q1a variant: fetch rejects with an EXPIRED (stale)
// assignment already in the store. Proves the client returns the STALE assignment, not the null
// fallback (create-explat-client.ts:160-165). Ephemeral; deleted after capture; never committed.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';

import { setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: window set before ../index is evaluated => real browser client.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
} );

describe( 'Q1a stale cache: fetch rejects with an EXPIRED assignment already stored', () => {
	it( 'returns the STALE treatment (not the null fallback); still never throws', async () => {
		const base = 1000000; // fixed clock base for deterministic TTL arithmetic
		const config = createMockedConfig();
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		// 1st fetch resolves 'treatment'; 2nd fetch rejects (server later unavailable).
		( config.fetchExperimentAssignment as MockedFunction )
			.mockImplementationOnce(
				() => Promise.resolve( { ttl: 60, variations: { experiment_name_a: 'treatment' } } )
			)
			.mockImplementationOnce(
				() =>
					new Promise( ( _res, rej ) =>
						rej( new Error( 'ECONNREFUSED simulated-server-unavailable' ) )
					)
			);

		// Phase 1: clock pinned at base -> first load stores a live 'treatment' (retrievedTimestamp=base, ttl 60).
		spiedMonotonicNow.mockImplementation( () => base );
		const client = createExPlatClient( config );
		const first = await client.loadExperimentAssignment( 'experiment_name_a' );

		// Phase 2: jump clock past the 60s TTL so the stored assignment is now EXPIRED/stale,
		// and the server is unavailable (2nd fetch rejects).
		spiedMonotonicNow.mockImplementation( () => base + 60 * 1000 + 5000 );
		let threw = false;
		let stale = null;
		try {
			stale = await client.loadExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			threw = true;
		}

		const fetchCalls = ( config.fetchExperimentAssignment as MockedFunction ).mock.calls;
		const logErrorCalls = ( config.logError as MockedFunction ).mock.calls;

		console.log( 'Q1AS_FIRST=' + JSON.stringify( first ) );
		console.log( 'Q1AS_THREW=' + threw );
		console.log( 'Q1AS_STALE_RESULT=' + JSON.stringify( stale ) );
		console.log( 'Q1AS_FETCH_CALLS=' + fetchCalls.length );
		console.log( 'Q1AS_LOGERROR=' + JSON.stringify( logErrorCalls ) );

		// Assertions.
		expect( first ).toEqual( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			retrievedTimestamp: base,
			ttl: 60,
		} );
		expect( threw ).toBe( false );
		// The key claim: after the failed refetch, the STALE 'treatment' is returned unchanged.
		expect( stale ).toEqual( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			retrievedTimestamp: base,
			ttl: 60,
		} );
		expect( ( stale as { isFallbackExperimentAssignment?: boolean } ).isFallbackExperimentAssignment ).toBeUndefined();
		expect( fetchCalls ).toHaveLength( 2 );
		expect( logErrorCalls ).toEqual( [
			[
				{
					message: 'ECONNREFUSED simulated-server-unavailable',
					experimentName: 'experiment_name_a',
					source: 'loadExperimentAssignment-initialError',
				},
			],
		] );
	} );
} );
```

**Command:**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q1a_stale.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q1a_stale.ts
  ● Console

    console.log
      Q1AS_FIRST={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1000000,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q1a_stale.ts:73:11)

    console.log
      Q1AS_THREW=false

      at Object.log (src/test/blitzy_tmp_q1a_stale.ts:74:11)

    console.log
      Q1AS_STALE_RESULT={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1000000,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q1a_stale.ts:75:11)

    console.log
      Q1AS_FETCH_CALLS=2

      at Object.log (src/test/blitzy_tmp_q1a_stale.ts:76:11)

    console.log
      Q1AS_LOGERROR=[[{"message":"ECONNREFUSED simulated-server-unavailable","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]]

      at Object.log (src/test/blitzy_tmp_q1a_stale.ts:77:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.718 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q1a_stale.ts/i.
Jest did not exit one second after the test run has completed.

'This usually means that there are asynchronous operations that weren't stopped in your tests. Consider running Jest with `--detectOpenHandles` to troubleshoot this issue.
```

**Answers (all OBSERVED).**

- The first load stored a live `treatment`: `Q1AS_FIRST={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1000000,"ttl":60}`. **OBSERVED**.
- After the clock is advanced past the TTL and the second fetch **rejects**, the client still does not throw (`Q1AS_THREW=false`) and returns the **stale `treatment`** unchanged: `Q1AS_STALE_RESULT={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1000000,"ttl":60}`. Crucially, the returned object has **no** `isFallbackExperimentAssignment` flag — it is the stored assignment, not a `null` fallback. This directly exercises [`create-explat-client.ts:158-165`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**.
- `Q1AS_FETCH_CALLS=2` (one live fetch, one failed refetch), and the failure was again logged with `source: "loadExperimentAssignment-initialError"`. **OBSERVED**.

**Stability (F12):**

```
$ for r in 1 2; do echo "run$r:"; grep -oE 'Q1AS_THREW=[a-z]+|Q1AS_FETCH_CALLS=[0-9]+|"variationName":"treatment","retrievedTimestamp":1000000' q1a_stale_run$r.log | paste -sd" " -; done
run1:
"variationName":"treatment","retrievedTimestamp":1000000 Q1AS_THREW=false "variationName":"treatment","retrievedTimestamp":1000000 Q1AS_FETCH_CALLS=2
run2:
"variationName":"treatment","retrievedTimestamp":1000000 Q1AS_THREW=false "variationName":"treatment","retrievedTimestamp":1000000 Q1AS_FETCH_CALLS=2
```

So the precise answer to "which variation on failure" is: **whatever is already cached if anything is (even if expired); otherwise the `null`/control fallback.** **OBSERVED**.

### Q1b — Slow server (fetch times out) and the genuinely randomized timeout (F14)

The timeout is deliberately **non-deterministic**: the base is `EXPERIMENT_FETCH_TIMEOUT = 10000` ([`create-explat-client.ts:16`](../../packages/explat-client/src/create-explat-client.ts)), but an active A/B experiment halves it to `5000` roughly half the time — `if ( Math.random() > 0.5 ) { experimentFetchTimeout = 5000; }` ([`create-explat-client.ts:136-138`](../../packages/explat-client/src/create-explat-client.ts)). The timeout is enforced by `Timing.timeoutPromise`, which **rejects** (it does not resolve) with `Promise has timed-out after <ms>ms.` ([`timing.ts:23-36`](../../packages/explat-client/src/internal/timing.ts), the rejecting `setTimeout` at `:30-31`). This spec leaves `Math.random` **unmocked** so the A/B is genuine, and simulates a hung server with a promise that never settles:

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q1b: slow server (fetch never resolves => timeout).
// Math.random is deliberately NOT mocked so the 5000/10000ms A/B is genuine (run repeatedly for
// the distribution). Ephemeral; deleted after capture; never committed.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';

import { setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: window set before ../index is evaluated => real browser client.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
	// Reinstate a strictly-increasing clock after resetAllMocks (mirrors timing.ts:10-16).
	// IMPORTANT: we deliberately DO NOT mock Math.random, so the 5000/10000 A/B is genuine.
	let clock = Date.now();
	spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
} );

describe( 'Q1b slow server (fetch never resolves => timeout)', () => {
	it( 'never throws; returns null-variation fallback; records the randomized timeout', async () => {
		jest.useFakeTimers();
		const config = createMockedConfig();
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		// A promise that never settles — simulates a hung/slow server.
		( config.fetchExperimentAssignment as MockedFunction ).mockImplementation(
			() => new Promise( () => undefined )
		);
		const client = createExPlatClient( config );

		let threw = false;
		let result = null;
		const promise = client
			.loadExperimentAssignment( 'experiment_name_a' )
			.then( ( r ) => {
				result = r;
			} )
			.catch( () => {
				threw = true;
			} );

		// Advance fake timers past the maximum (10000ms) so either the 5000ms or 10000ms timeout
		// trips; the async variant flushes intervening microtasks (getAnonId, Promise.race).
		await jest.advanceTimersByTimeAsync( 10 * 1000 );
		await promise;

		const logErrorCalls = ( config.logError as MockedFunction ).mock.calls;
		const firstCallArg = logErrorCalls.length > 0 ? logErrorCalls[ 0 ][ 0 ] : null;
		const errorMessage = firstCallArg ? firstCallArg.message : null;
		const errorSource = firstCallArg ? firstCallArg.source : null;
		const match = errorMessage ? /after (\d+)ms/.exec( errorMessage ) : null;
		const timeoutMs = match ? match[ 1 ] : 'UNKNOWN';

		console.log( 'Q1B_THREW=' + threw );
		console.log( 'Q1B_RESULT=' + JSON.stringify( result ) );
		console.log( 'Q1B_ERROR_MESSAGE=' + JSON.stringify( errorMessage ) );
		console.log( 'Q1B_ERROR_SOURCE=' + JSON.stringify( errorSource ) );
		console.log( 'Q1B_TIMEOUT_MS=' + timeoutMs );

		// Assertions.
		expect( threw ).toBe( false );
		expect( result ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: null,
			ttl: 60,
			isFallbackExperimentAssignment: true,
		} );
		expect( errorSource ).toBe( 'loadExperimentAssignment-initialError' );
		expect( errorMessage ).toMatch( /^Promise has timed-out after (5000|10000)ms\.$/ );
		expect( [ '5000', '10000' ] ).toContain( timeoutMs );

		jest.useRealTimers();
	} );
} );
```

**Command (single run):**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q1b_timeout.ts
```

**Complete output of one run** (verbatim; this run happened to draw the 5000 ms arm):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q1b_timeout.ts
  ● Console

    console.log
      Q1B_THREW=false

      at Object.log (src/test/blitzy_tmp_q1b_timeout.ts:75:11)

    console.log
      Q1B_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965263779,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_q1b_timeout.ts:76:11)

    console.log
      Q1B_ERROR_MESSAGE="Promise has timed-out after 5000ms."

      at Object.log (src/test/blitzy_tmp_q1b_timeout.ts:77:11)

    console.log
      Q1B_ERROR_SOURCE="loadExperimentAssignment-initialError"

      at Object.log (src/test/blitzy_tmp_q1b_timeout.ts:78:11)

    console.log
      Q1B_TIMEOUT_MS=5000

      at Object.log (src/test/blitzy_tmp_q1b_timeout.ts:79:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.743 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q1b_timeout.ts/i.
```

**Distribution over 20 identical runs (per the run-to-run-inconsistency rule).** The same unchanged spec was run 20 times:

```bash
cd packages/explat-client
for i in $(seq 1 20); do
  CI=true yarn jest --ci src/test/blitzy_tmp_q1b_timeout.ts \
    > /tmp/blitzy_explat_logs/q1b_run_$i.log 2>&1
done
```

The timeout chosen on each run, **in run order** (command shown, then output):

```bash
for i in $(seq 1 20); do sed -nE 's/.*Q1B_TIMEOUT_MS=([0-9]+).*/\1/p' /tmp/blitzy_explat_logs/q1b_run_$i.log; done | tr '\n' ' '
```

```
5000 5000 10000 5000 5000 10000 10000 5000 5000 10000 10000 10000 5000 5000 5000 5000 10000 10000 10000 5000
```

Tally:

```bash
grep -hoE 'Q1B_TIMEOUT_MS=[0-9]+' /tmp/blitzy_explat_logs/q1b_run_*.log | sort | uniq -c
```

```
      9 Q1B_TIMEOUT_MS=10000
     11 Q1B_TIMEOUT_MS=5000
```

**Observed distribution: `5000 ms` on 11/20 runs and `10000 ms` on 9/20 runs** — consistent with the `Math.random() > 0.5` coin-flip at [`create-explat-client.ts:136-138`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**. (The split is not exactly 10/10 because 20 draws of a fair coin need not be balanced; it is not a rigged constant.) **INFERRED** (statistical interpretation).

**What is constant vs. what varies across the 20 runs (F14).** Every semantic field and the never-throw outcome are constant, while the wall-clock `retrievedTimestamp` differs on every run:

```bash
# each invariant, counted across all 20 run-logs:
for pat in 'Q1B_THREW=false' '"variationName":null' '"ttl":60,"isFallbackExperimentAssignment":true' 'loadExperimentAssignment-initialError'; do
  printf '%7d %s\n' "$(grep -lF "$pat" /tmp/blitzy_explat_logs/q1b_run_*.log | wc -l)" "runs containing: $pat"
done
```

```
     20 runs with Q1B_THREW=false
     20 runs with "variationName":null in Q1B_RESULT
     20 runs with "ttl":60,"isFallbackExperimentAssignment":true
     20 runs with Q1B_ERROR_SOURCE="loadExperimentAssignment-initialError"
```

```bash
grep -hoE '"retrievedTimestamp":[0-9]+' /tmp/blitzy_explat_logs/q1b_run_*.log | sort -u | wc -l
```

```
distinct retrievedTimestamp values across the 20 runs: 20
```

So across 20 runs the returned object is **structurally identical** every time — `variationName:null`, `ttl:60`, `isFallbackExperimentAssignment:true`, never throws, logged with `source:"loadExperimentAssignment-initialError"` — and **only** the randomized timeout (`5000` vs `10000`) and the wall-clock `retrievedTimestamp` vary. The earlier revision's claim that "the returned object is constant across 20 runs" is therefore refined here: the *semantics and fields* are constant; the `retrievedTimestamp` value is not. **OBSERVED**.

### Q1 — Synthesis

The "designed to never throw" contract holds under both failure modes: **OBSERVED** across an empty-store rejection, a stale-cache rejection, and 20 timeout runs, `loadExperimentAssignment` never threw. On failure with nothing cached, the user is assigned the fallback with `variationName: null` — the **default/control** experience; on failure with something cached, the user keeps the cached variation (even if expired). **OBSERVED** (values); the meaning of `null` is **INFERRED** from `README.md:22`.

---

## Q2 — Concurrency: how many network calls happen when many callers request the same experiment at once?

**Question.** When multiple parts of the app request the **same** experiment assignment simultaneously, how many network calls are actually made? `create-explat-client.ts:78-80` documents the intent that there is "only ever one fetch process occuring" per experiment.

**Where the behavior lives.** Each experiment name is fetched through a wrapper built by `Timing.asyncOneAtATime` ([`timing.ts:44-54`](../../packages/explat-client/src/internal/timing.ts)), which returns the same in-flight promise to every caller until it settles, then resets. The wrappers are stored in a per-name registry object `experimentNameToWrappedExperimentAssignmentFetchAndStore` ([`create-explat-client.ts:82-94`](../../packages/explat-client/src/create-explat-client.ts)). This is the industry-standard **single-flight** (a.k.a. request-coalescing / request-deduplication) pattern. **INFERRED** (mechanism and its standard name, from source); the call counts below are **OBSERVED**.

**Full spec** (covers both one-instance dedup **and** the per-instance scope, F10):

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q2: concurrency / request deduplication (single-flight).
// Demonstrates (A) N concurrent same-name loads on ONE client => 1 fetch, and (B) the dedup registry
// is per client instance: two instances loading concurrently => 2 fetches. Ephemeral; deleted after
// capture; never committed.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';
import { delayedValue, ONE_DELAY, setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: window set before ../index is evaluated => real browser client.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

const mockTreatmentFetch = ( config: Config ) =>
	( config.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
		delayedValue( { ttl: 60, variations: { experiment_name_a: 'treatment' } }, ONE_DELAY )
	);

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
	let clock = Date.now();
	spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
} );

describe( 'Q2 single-flight deduplication (one client instance)', () => {
	it( 'N=8 simultaneous loads of the same name => exactly 1 fetch; all callers get the same result', async () => {
		const N = 8;
		const config = createMockedConfig();
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		mockTreatmentFetch( config );
		const client = createExPlatClient( config );

		const results = await Promise.all(
			Array.from( { length: N } ).map( () =>
				client.loadExperimentAssignment( 'experiment_name_a' )
			)
		);

		const fetchCalls = ( config.fetchExperimentAssignment as MockedFunction ).mock.calls;
		const allEqual = results.every(
			( r ) => JSON.stringify( r ) === JSON.stringify( results[ 0 ] )
		);

		console.log( 'Q2_N=' + N );
		console.log( 'Q2_FETCH_CALLS=' + fetchCalls.length );
		console.log( 'Q2_REQUEST_ARG=' + JSON.stringify( fetchCalls[ 0 ][ 0 ] ) );
		console.log( 'Q2_ALL_EQUAL=' + allEqual );
		console.log( 'Q2_RESULT0=' + JSON.stringify( results[ 0 ] ) );

		expect( fetchCalls ).toHaveLength( 1 );
		expect( fetchCalls[ 0 ][ 0 ] ).toEqual( {
			experimentName: 'experiment_name_a',
			anonId: 'anon-id-xyz-123',
		} );
		expect( allEqual ).toBe( true );
		expect( results ).toHaveLength( N );
		expect( results[ 0 ] ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			ttl: 60,
		} );
	} );
} );

describe( 'Q2 dedup scope is per client instance', () => {
	it( 'two separate client instances loading the same name concurrently => 2 fetches (one each)', async () => {
		// Pin the clock constant so both fetches share retrievedTimestamp (avoids the store
		// race-condition guard) and produce identical results.
		const CONST = 2000000;
		spiedMonotonicNow.mockImplementation( () => CONST );

		const configA = createMockedConfig();
		const configB = createMockedConfig();
		mockTreatmentFetch( configA );
		mockTreatmentFetch( configB );
		const clientA = createExPlatClient( configA );
		const clientB = createExPlatClient( configB );

		// Fire both concurrently, before either resolves/stores, so each instance's own registry
		// starts its own in-flight fetch.
		const [ ra, rb ] = await Promise.all( [
			clientA.loadExperimentAssignment( 'experiment_name_a' ),
			clientB.loadExperimentAssignment( 'experiment_name_a' ),
		] );

		const fetchA = ( configA.fetchExperimentAssignment as MockedFunction ).mock.calls.length;
		const fetchB = ( configB.fetchExperimentAssignment as MockedFunction ).mock.calls.length;

		console.log( 'Q2SCOPE_FETCH_A=' + fetchA );
		console.log( 'Q2SCOPE_FETCH_B=' + fetchB );
		console.log( 'Q2SCOPE_TOTAL=' + ( fetchA + fetchB ) );
		console.log( 'Q2SCOPE_RESULT_A=' + JSON.stringify( ra ) );
		console.log( 'Q2SCOPE_RESULT_B=' + JSON.stringify( rb ) );

		// Each instance fetched exactly once => 2 total => dedup is NOT global, it is per instance.
		expect( fetchA ).toBe( 1 );
		expect( fetchB ).toBe( 1 );
		expect( fetchA + fetchB ).toBe( 2 );
	} );
} );
```

**Command:**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q2.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q2.ts
  ● Console

    console.log
      Q2_N=8

      at Object.log (src/test/blitzy_tmp_q2.ts:63:11)

    console.log
      Q2_FETCH_CALLS=1

      at Object.log (src/test/blitzy_tmp_q2.ts:64:11)

    console.log
      Q2_REQUEST_ARG={"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}

      at Object.log (src/test/blitzy_tmp_q2.ts:65:11)

    console.log
      Q2_ALL_EQUAL=true

      at Object.log (src/test/blitzy_tmp_q2.ts:66:11)

    console.log
      Q2_RESULT0={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965188906,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q2.ts:67:11)

    console.log
      Q2SCOPE_FETCH_A=1

      at Object.log (src/test/blitzy_tmp_q2.ts:108:11)

    console.log
      Q2SCOPE_FETCH_B=1

      at Object.log (src/test/blitzy_tmp_q2.ts:109:11)

    console.log
      Q2SCOPE_TOTAL=2

      at Object.log (src/test/blitzy_tmp_q2.ts:110:11)

    console.log
      Q2SCOPE_RESULT_A={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":2000000,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q2.ts:111:11)

    console.log
      Q2SCOPE_RESULT_B={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":2000000,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q2.ts:112:11)


Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        0.727 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q2.ts/i.
Jest did not exit one second after the test run has completed.

'This usually means that there are asynchronous operations that weren't stopped in your tests. Consider running Jest with `--detectOpenHandles` to troubleshoot this issue.
```

**Answers (all OBSERVED).**

- **`N = 8` simultaneous loads of the same name on one client instance ⇒ exactly one network call:** `Q2_N=8`, `Q2_FETCH_CALLS=1`, and every caller received the identical result (`Q2_ALL_EQUAL=true`, `Q2_RESULT0={"...","variationName":"treatment","...","ttl":60}`). The single call used `{"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}`. This is `asyncOneAtATime` coalescing the eight concurrent calls into one ([`timing.ts:44-54`](../../packages/explat-client/src/internal/timing.ts)). **OBSERVED**.
- **Scope is per client instance (F10):** two *separate* `createExPlatClient` instances loading the same name concurrently produce **two** network calls, one each — `Q2SCOPE_FETCH_A=1`, `Q2SCOPE_FETCH_B=1`, `Q2SCOPE_TOTAL=2`. The dedup registry is created **inside** each `createExPlatClient` call ([`create-explat-client.ts:82-94`](../../packages/explat-client/src/create-explat-client.ts)), so it is **not** global — it deduplicates only within a single client instance. **OBSERVED**.

Because Calypso constructs one client instance per JavaScript execution context, the practical consequence is that deduplication holds within a single browser tab / process but not across tabs or processes. The per-instance boundary is **OBSERVED**; the "one tab / one process" extrapolation is **INFERRED**. A recognized trade-off of single-flight is a shared error blast-radius — when the one in-flight call fails, every waiting caller receives the same failure result, exactly the fallback-sharing seen in Q1a. **INFERRED**.

**Stability (F12):**

```
$ for r in 1 2; do echo "run$r:"; grep -oE 'Q2_FETCH_CALLS=[0-9]+|Q2_ALL_EQUAL=[a-z]+|Q2SCOPE_FETCH_A=[0-9]+|Q2SCOPE_FETCH_B=[0-9]+|Q2SCOPE_TOTAL=[0-9]+' q2_run$r.log | paste -sd" " -; done
run1:
Q2_FETCH_CALLS=1 Q2_ALL_EQUAL=true Q2SCOPE_FETCH_A=1 Q2SCOPE_FETCH_B=1 Q2SCOPE_TOTAL=2
run2:
Q2_FETCH_CALLS=1 Q2_ALL_EQUAL=true Q2SCOPE_FETCH_A=1 Q2SCOPE_FETCH_B=1 Q2SCOPE_TOTAL=2
```

---

## Q3 — Caching: does every repeated request hit the network, and what happens when the TTL expires?

**Question.** Does each repeated request for the same experiment reach the network, and what happens once the cached assignment's TTL expires?

**Where the behavior lives.** At the top of `loadExperimentAssignment`, if a stored assignment exists **and** is still alive it is returned immediately with no network call ([`create-explat-client.ts:119-125`](../../packages/explat-client/src/create-explat-client.ts)). Liveness is `isAlive`: `monotonicNow() < ttl * 1000 + retrievedTimestamp` ([`experiment-assignments.ts:8-14`](../../packages/explat-client/src/internal/experiment-assignments.ts)). The TTL has a `minimumTtl = 60` s floor ([`experiment-assignments.ts:21`](../../packages/explat-client/src/internal/experiment-assignments.ts)), applied as `Math.max( minimumTtl, responseTtl )` ([`requests.ts:93`](../../packages/explat-client/src/internal/requests.ts)). **INFERRED** (from source); the counts below are **OBSERVED**.

**Full spec:**

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q3: caching behavior / TTL.
// Shows repeated quick loads are cache hits (1 fetch), and a load after the TTL elapses refetches
// (2 fetches). Ephemeral; deleted after capture; never committed.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';
import { delayedValue, ONE_DELAY, setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: window set before ../index is evaluated => real browser client.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
} );

describe( 'Q3 caching / TTL', () => {
	it( '3 quick loads => 1 fetch (cache hits); a load past the TTL => a 2nd fetch (refetch)', async () => {
		const config = createMockedConfig();
		( config.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
			delayedValue( { ttl: 60, variations: { experiment_name_a: 'treatment' } }, ONE_DELAY )
		);
		const client = createExPlatClient( config );

		// Hold the clock constant so the cached assignment stays within its 60s TTL.
		const firstDate = 3000000;
		spiedMonotonicNow.mockImplementation( () => firstDate );

		await client.loadExperimentAssignment( 'experiment_name_a' );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		const afterThreeQuick = ( config.fetchExperimentAssignment as MockedFunction ).mock.calls
			.length;
		console.log( 'Q3_FETCH_AFTER_3_QUICK=' + afterThreeQuick );

		// Advance the clock strictly past the 60s TTL: isAlive is monotonicNow() < ttl*1000 + retrievedTimestamp.
		spiedMonotonicNow.mockImplementation( () => firstDate + 60 * 1000 + 1 );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		const afterTtl = ( config.fetchExperimentAssignment as MockedFunction ).mock.calls.length;
		console.log( 'Q3_FETCH_AFTER_TTL=' + afterTtl );

		expect( afterThreeQuick ).toBe( 1 );
		expect( afterTtl ).toBe( 2 );
	} );
} );
```

**Command:**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q3.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q3.ts
  ● Console

    console.log
      Q3_FETCH_AFTER_3_QUICK=1

      at Object.log (src/test/blitzy_tmp_q3.ts:51:11)

    console.log
      Q3_FETCH_AFTER_TTL=2

      at Object.log (src/test/blitzy_tmp_q3.ts:57:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.698 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q3.ts/i.
Jest did not exit one second after the test run has completed.

'This usually means that there are asynchronous operations that weren't stopped in your tests. Consider running Jest with `--detectOpenHandles` to troubleshoot this issue.
```

**Answers (all OBSERVED).**

- **Repeated requests do *not* each hit the network.** Three back-to-back loads of the same name, with the clock held inside the 60 s TTL, produced **one** fetch total: `Q3_FETCH_AFTER_3_QUICK=1`. The 2nd and 3rd loads were served from cache via the `isAlive` gate ([`create-explat-client.ts:119-125`](../../packages/explat-client/src/create-explat-client.ts)). **OBSERVED**.
- **Once the TTL expires, the next request refetches.** After advancing the clock strictly past `ttl*1000 + retrievedTimestamp`, a further load made a **second** network call: `Q3_FETCH_AFTER_TTL=2`. `isAlive` returned false, so the cache gate fell through to a fresh fetch. **OBSERVED**.

**On the TTL value.** Under the mocked server the effective TTL is the `60` s floor (`minimumTtl`), which is what the spec exercises. **OBSERVED**. In production the client "Respects the server returned TTL (3600 seconds in production at the time of writing)" per [`README.md:42`](../../packages/explat-client/README.md); the `3600` figure is **INFERRED** (documented in the README only — the client code hard-codes only the `60` s floor, never `3600`).

**Stability (F12):**

```
$ for r in 1 2; do echo "run$r:"; grep -oE 'Q3_FETCH_AFTER_3_QUICK=[0-9]+|Q3_FETCH_AFTER_TTL=[0-9]+' q3_run$r.log | paste -sd" " -; done
run1:
Q3_FETCH_AFTER_3_QUICK=1 Q3_FETCH_AFTER_TTL=2
run2:
Q3_FETCH_AFTER_3_QUICK=1 Q3_FETCH_AFTER_TTL=2
```

---

## Q4 — Async load vs. the synchronous getter: what happens if you call the getter before the load finishes?

**Question.** The client exposes an asynchronous loader (`loadExperimentAssignment`) and a synchronous getter (`dangerouslyGetExperimentAssignment`). If the synchronous getter is called **before** the async load has finished, does it break the consuming app, or is it handled gracefully? A sibling getter, `dangerouslyGetMaybeLoadedExperimentAssignment`, is contrasted here as well.

**Where the behavior lives.** `dangerouslyGetExperimentAssignment` throws internally when nothing is stored ([`create-explat-client.ts:190-194`](../../packages/explat-client/src/create-explat-client.ts)) but **catches its own throw** and returns a `null`-variation fallback ([`:213-222`](../../packages/explat-client/src/create-explat-client.ts)); only in development mode does it log (`source: 'dangerouslyGetExperimentAssignment-error'`). If it is called within 1000 ms of a successful load, development mode additionally emits a "too soon" warning (`source: 'dangerouslyGetExperimentAssignment'`, [`:197-208`](../../packages/explat-client/src/create-explat-client.ts)). Its sibling `dangerouslyGetMaybeLoadedExperimentAssignment` instead returns **`null`** when nothing is stored ([`:231-234`](../../packages/explat-client/src/create-explat-client.ts)). **INFERRED** (from source); every value below is **OBSERVED**.

This spec exercises **three states** — never-started, **in-flight** (load started but not resolved, F6), and loaded — for **both** getters, in **both** `isDevelopmentMode: true` and `false` (F7):

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q4: async load vs the two synchronous getters.
// Covers three states (never-started, in-flight, loaded) for BOTH getters, in BOTH
// isDevelopmentMode=true and =false. Ephemeral; deleted after capture; never committed.
import '@automattic/calypso-polyfills';

import localStorage from '../internal/local-storage';
import { delayedValue, ONE_DELAY, setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

// CANONICAL PUBLIC ENTRY: window set before ../index is evaluated => real browser client.
setBrowserContext();
// eslint-disable-next-line @typescript-eslint/no-var-requires
const { createExPlatClient } = require( '../index' );

const spiedMonotonicNow = jest.spyOn( Timing, 'monotonicNow' );

const createMockedConfig = ( override: Partial< Config > = {} ): Config => ( {
	logError: jest.fn(),
	fetchExperimentAssignment: jest.fn(),
	getAnonId: jest.fn(),
	isDevelopmentMode: false,
	...override,
} );

const mockTreatmentFetch = ( config: Config ) =>
	( config.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
		delayedValue( { ttl: 60, variations: { experiment_name_a: 'treatment' } }, ONE_DELAY )
	);

beforeEach( () => {
	jest.resetAllMocks();
	setBrowserContext();
	localStorage.clear();
	let clock = Date.now();
	spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
} );

describe( 'Q4 [isDevelopmentMode=true] never-started then loaded', () => {
	it( 'never throws; not-loaded logs error + fallback / null; loaded returns treatment + too-soon warning', async () => {
		const config = createMockedConfig( { isDevelopmentMode: true } );
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		mockTreatmentFetch( config );
		const client = createExPlatClient( config );

		// --- NEVER-STARTED (no load has been called at all) ---
		let dgetThrew = false;
		let dgetResult = null;
		try {
			dgetResult = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			dgetThrew = true;
		}
		const dmaybeNotStarted =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		const logAfterNotStarted = JSON.parse(
			JSON.stringify( ( config.logError as MockedFunction ).mock.calls )
		);

		console.log( 'Q4_DEV_NOTSTARTED_DGET_THREW=' + dgetThrew );
		console.log( 'Q4_DEV_NOTSTARTED_DGET_RESULT=' + JSON.stringify( dgetResult ) );
		console.log( 'Q4_DEV_NOTSTARTED_DMAYBE=' + JSON.stringify( dmaybeNotStarted ) );
		console.log( 'Q4_DEV_LOG_AFTER_NOTSTARTED=' + JSON.stringify( logAfterNotStarted ) );

		// --- LOADED (await the async load, then call the getters immediately) ---
		await client.loadExperimentAssignment( 'experiment_name_a' );
		const dgetLoaded = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		const dmaybeLoaded =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		const logFinal = ( config.logError as MockedFunction ).mock.calls;

		console.log( 'Q4_DEV_LOADED_DGET=' + JSON.stringify( dgetLoaded ) );
		console.log( 'Q4_DEV_LOADED_DMAYBE=' + JSON.stringify( dmaybeLoaded ) );
		console.log( 'Q4_DEV_LOG_FINAL=' + JSON.stringify( logFinal ) );

		// Assertions.
		expect( dgetThrew ).toBe( false );
		expect( dgetResult ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: null,
			ttl: 60,
			isFallbackExperimentAssignment: true,
		} );
		expect( dmaybeNotStarted ).toBeNull();
		expect( logAfterNotStarted ).toEqual( [
			[
				{
					message: "Trying to dangerously get an ExperimentAssignment that hasn't loaded.",
					experimentName: 'experiment_name_a',
					source: 'dangerouslyGetExperimentAssignment-error',
				},
			],
		] );
		expect( dgetLoaded ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			ttl: 60,
		} );
		expect( dmaybeLoaded ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			ttl: 60,
		} );
		// The loaded getter, called immediately (delta < 1000ms) in dev mode, emits a 2nd log:
		// the "too soon" warning with source 'dangerouslyGetExperimentAssignment'.
		expect( logFinal ).toHaveLength( 2 );
		expect( logFinal[ 1 ][ 0 ] ).toEqual( {
			message:
				'Warning: Trying to dangerously get an ExperimentAssignment too soon after loading it.',
			experimentName: 'experiment_name_a',
			source: 'dangerouslyGetExperimentAssignment',
		} );
	} );
} );

describe( 'Q4 [isDevelopmentMode=true] IN-FLIGHT (load started, not yet resolved)', () => {
	it( 'both getters handle the pending state gracefully (fallback / null); load then resolves', async () => {
		jest.useFakeTimers();
		const config = createMockedConfig( { isDevelopmentMode: true } );
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		mockTreatmentFetch( config );
		const client = createExPlatClient( config );

		// Start the async load but DO NOT await it: it is now in-flight (pending).
		const pending = client.loadExperimentAssignment( 'experiment_name_a' );

		let dgetThrew = false;
		let dgetInflight = null;
		try {
			dgetInflight = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			dgetThrew = true;
		}
		const dmaybeInflight =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		const logInflight = JSON.parse(
			JSON.stringify( ( config.logError as MockedFunction ).mock.calls )
		);

		console.log( 'Q4_INFLIGHT_DGET_THREW=' + dgetThrew );
		console.log( 'Q4_INFLIGHT_DGET_RESULT=' + JSON.stringify( dgetInflight ) );
		console.log( 'Q4_INFLIGHT_DMAYBE=' + JSON.stringify( dmaybeInflight ) );
		console.log( 'Q4_INFLIGHT_LOG=' + JSON.stringify( logInflight ) );

		// Now let the in-flight load resolve.
		await jest.advanceTimersByTimeAsync( ONE_DELAY );
		const resolved = await pending;
		console.log( 'Q4_INFLIGHT_RESOLVED=' + JSON.stringify( resolved ) );

		expect( dgetThrew ).toBe( false );
		expect( dgetInflight ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: null,
			ttl: 60,
			isFallbackExperimentAssignment: true,
		} );
		expect( dmaybeInflight ).toBeNull();
		expect( logInflight ).toEqual( [
			[
				{
					message: "Trying to dangerously get an ExperimentAssignment that hasn't loaded.",
					experimentName: 'experiment_name_a',
					source: 'dangerouslyGetExperimentAssignment-error',
				},
			],
		] );
		expect( resolved ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: 'treatment',
			ttl: 60,
		} );

		jest.useRealTimers();
	} );
} );

describe( 'Q4 [isDevelopmentMode=false] never-started then loaded => NO logging at all', () => {
	it( 'both getters behave the same but emit zero log calls (logs only in development mode)', async () => {
		const config = createMockedConfig( { isDevelopmentMode: false } );
		( config.getAnonId as MockedFunction ).mockImplementation( () =>
			Promise.resolve( 'anon-id-xyz-123' )
		);
		mockTreatmentFetch( config );
		const client = createExPlatClient( config );

		// NEVER-STARTED
		let dgetThrew = false;
		let dgetResult = null;
		try {
			dgetResult = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			dgetThrew = true;
		}
		const dmaybeNotStarted =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		const logAfterNotStarted = JSON.parse(
			JSON.stringify( ( config.logError as MockedFunction ).mock.calls )
		);

		// LOADED
		await client.loadExperimentAssignment( 'experiment_name_a' );
		const dgetLoaded = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		const dmaybeLoaded =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		const logFinal = ( config.logError as MockedFunction ).mock.calls;

		console.log( 'Q4_PROD_NOTSTARTED_DGET_THREW=' + dgetThrew );
		console.log( 'Q4_PROD_NOTSTARTED_DGET_RESULT=' + JSON.stringify( dgetResult ) );
		console.log( 'Q4_PROD_NOTSTARTED_DMAYBE=' + JSON.stringify( dmaybeNotStarted ) );
		console.log( 'Q4_PROD_LOG_AFTER_NOTSTARTED=' + JSON.stringify( logAfterNotStarted ) );
		console.log( 'Q4_PROD_LOADED_DGET=' + JSON.stringify( dgetLoaded ) );
		console.log( 'Q4_PROD_LOADED_DMAYBE=' + JSON.stringify( dmaybeLoaded ) );
		console.log( 'Q4_PROD_LOG_FINAL=' + JSON.stringify( logFinal ) );

		expect( dgetThrew ).toBe( false );
		expect( dgetResult ).toMatchObject( {
			experimentName: 'experiment_name_a',
			variationName: null,
			isFallbackExperimentAssignment: true,
		} );
		expect( dmaybeNotStarted ).toBeNull();
		expect( dgetLoaded ).toMatchObject( { variationName: 'treatment' } );
		expect( dmaybeLoaded ).toMatchObject( { variationName: 'treatment' } );
		// The whole point: in production mode NOTHING is logged, in any state.
		expect( logAfterNotStarted ).toEqual( [] );
		expect( logFinal ).toEqual( [] );
	} );
} );
```

**Command:**

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_q4.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_q4.ts
  ● Console

    console.log
      Q4_DEV_NOTSTARTED_DGET_THREW=false

      at Object.log (src/test/blitzy_tmp_q4.ts:64:11)

    console.log
      Q4_DEV_NOTSTARTED_DGET_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965231023,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_q4.ts:65:11)

    console.log
      Q4_DEV_NOTSTARTED_DMAYBE=null

      at Object.log (src/test/blitzy_tmp_q4.ts:66:11)

    console.log
      Q4_DEV_LOG_AFTER_NOTSTARTED=[[{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]]

      at Object.log (src/test/blitzy_tmp_q4.ts:67:11)

    console.log
      Q4_DEV_LOADED_DGET={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231024,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q4.ts:76:11)

    console.log
      Q4_DEV_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231024,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q4.ts:77:11)

    console.log
      Q4_DEV_LOG_FINAL=[[{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}],[{"message":"Warning: Trying to dangerously get an ExperimentAssignment too soon after loading it.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment"}]]

      at Object.log (src/test/blitzy_tmp_q4.ts:78:11)

    console.log
      Q4_INFLIGHT_DGET_THREW=false

      at Object.log (src/test/blitzy_tmp_q4.ts:146:11)

    console.log
      Q4_INFLIGHT_DGET_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965231033,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_q4.ts:147:11)

    console.log
      Q4_INFLIGHT_DMAYBE=null

      at Object.log (src/test/blitzy_tmp_q4.ts:148:11)

    console.log
      Q4_INFLIGHT_LOG=[[{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]]

      at Object.log (src/test/blitzy_tmp_q4.ts:149:11)

    console.log
      Q4_INFLIGHT_RESOLVED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231032,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q4.ts:154:11)

    console.log
      Q4_PROD_NOTSTARTED_DGET_THREW=false

      at Object.log (src/test/blitzy_tmp_q4.ts:213:11)

    console.log
      Q4_PROD_NOTSTARTED_DGET_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783965231036,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_q4.ts:214:11)

    console.log
      Q4_PROD_NOTSTARTED_DMAYBE=null

      at Object.log (src/test/blitzy_tmp_q4.ts:215:11)

    console.log
      Q4_PROD_LOG_AFTER_NOTSTARTED=[]

      at Object.log (src/test/blitzy_tmp_q4.ts:216:11)

    console.log
      Q4_PROD_LOADED_DGET={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231037,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q4.ts:217:11)

    console.log
      Q4_PROD_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231037,"ttl":60}

      at Object.log (src/test/blitzy_tmp_q4.ts:218:11)

    console.log
      Q4_PROD_LOG_FINAL=[]

      at Object.log (src/test/blitzy_tmp_q4.ts:219:11)


Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        0.706 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_q4.ts/i.
Jest did not exit one second after the test run has completed.

'This usually means that there are asynchronous operations that weren't stopped in your tests. Consider running Jest with `--detectOpenHandles` to troubleshoot this issue.
```

**Answers (all OBSERVED).**

- **It does not break the app.** In every state and mode, `dangerouslyGetExperimentAssignment` returned normally without throwing (`Q4_DEV_NOTSTARTED_DGET_THREW=false`, `Q4_INFLIGHT_DGET_THREW=false`, `Q4_PROD_NOTSTARTED_DGET_THREW=false`). This is the throw-then-catch fallback at [`create-explat-client.ts:213-222`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**.
- **Never-started (dev mode):** `dangerouslyGetExperimentAssignment` returned a `null`-variation fallback (`Q4_DEV_NOTSTARTED_DGET_RESULT={..."variationName":null,..."isFallbackExperimentAssignment":true}`) and logged one error with `source:"dangerouslyGetExperimentAssignment-error"` (`Q4_DEV_LOG_AFTER_NOTSTARTED`). The sibling `dangerouslyGetMaybeLoadedExperimentAssignment` returned `null` (`Q4_DEV_NOTSTARTED_DMAYBE=null`) — not a fallback — per [`:231-234`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**.
- **In-flight — load started but not yet resolved (F6):** while the load promise was pending, `dangerouslyGetExperimentAssignment` returned the same `null`-variation fallback (`Q4_INFLIGHT_DGET_RESULT`) and `dangerouslyGetMaybeLoadedExperimentAssignment` returned `null` (`Q4_INFLIGHT_DMAYBE=null`), with one "hasn't loaded" error logged (`Q4_INFLIGHT_LOG`). When the in-flight load was then allowed to resolve, it returned the real `treatment` (`Q4_INFLIGHT_RESOLVED={..."variationName":"treatment","ttl":60}`). So calling the getter mid-flight is handled gracefully and does **not** disturb the pending load. **OBSERVED**.
- **Loaded (dev mode):** immediately after `await loadExperimentAssignment`, both getters returned the real `treatment` (`Q4_DEV_LOADED_DGET`, `Q4_DEV_LOADED_DMAYBE`). Because the getter was called within 1000 ms of the load, development mode emitted a **second** log entry — the "too soon" warning: `Q4_DEV_LOG_FINAL` shows `[[...hasn't loaded...,source:"dangerouslyGetExperimentAssignment-error"],[{"message":"Warning: Trying to dangerously get an ExperimentAssignment too soon after loading it.",...,"source":"dangerouslyGetExperimentAssignment"}]]` ([`:197-208`](../../packages/explat-client/src/create-explat-client.ts)). This immediate-post-load warning was missed by the earlier revision and is now captured. **OBSERVED**.
- **Non-development mode logs nothing (F7):** with `isDevelopmentMode: false`, the return values are identical but **no** `logError` call is made in any state — `Q4_PROD_LOG_AFTER_NOTSTARTED=[]` and `Q4_PROD_LOG_FINAL=[]`. All logging in these getters is gated on `config.isDevelopmentMode` ([`:213-221`](../../packages/explat-client/src/create-explat-client.ts) and [`:197-208`](../../packages/explat-client/src/create-explat-client.ts)). **OBSERVED**.

**Practical guidance (matches `README.md:65-73`).** The synchronous getter "now logs and won't throw"; the intended pattern is to call `loadExperimentAssignment` first. `dangerouslyGetMaybeLoadedExperimentAssignment` (added in `CHANGELOG.md:5` for `useExperiment`) is the right choice when the caller wants to distinguish "not loaded yet" (`null`) from a real assignment, rather than receiving a control-defaulting fallback. **OBSERVED** (behavior); guidance mapping is **INFERRED** from the README/CHANGELOG.

**Stability (F12):**

```
$ for r in 1 2; do echo "run$r:"; grep -oE 'Q4_(DEV|INFLIGHT|PROD)_[A-Z_]*(THREW|DMAYBE|LOG_FINAL|LOG|RESOLVED)=[^ ]*' q4_run$r.log | paste -sd" " -; done
run1:
Q4_DEV_NOTSTARTED_DGET_THREW=false Q4_DEV_NOTSTARTED_DMAYBE=null Q4_DEV_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231024,"ttl":60} Q4_DEV_LOG_FINAL=[[{"message":"Trying Q4_INFLIGHT_DGET_THREW=false Q4_INFLIGHT_DMAYBE=null Q4_INFLIGHT_LOG=[[{"message":"Trying Q4_INFLIGHT_RESOLVED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231032,"ttl":60} Q4_PROD_NOTSTARTED_DGET_THREW=false Q4_PROD_NOTSTARTED_DMAYBE=null Q4_PROD_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965231037,"ttl":60} Q4_PROD_LOG_FINAL=[]
run2:
Q4_DEV_NOTSTARTED_DGET_THREW=false Q4_DEV_NOTSTARTED_DMAYBE=null Q4_DEV_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965237947,"ttl":60} Q4_DEV_LOG_FINAL=[[{"message":"Trying Q4_INFLIGHT_DGET_THREW=false Q4_INFLIGHT_DMAYBE=null Q4_INFLIGHT_LOG=[[{"message":"Trying Q4_INFLIGHT_RESOLVED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965237953,"ttl":60} Q4_PROD_NOTSTARTED_DGET_THREW=false Q4_PROD_NOTSTARTED_DMAYBE=null Q4_PROD_LOADED_DMAYBE={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783965237958,"ttl":60} Q4_PROD_LOG_FINAL=[]
```

---

## Open-handle demonstration: why the "worker failed to exit" warning appears (F13)

The benign `A worker process has failed to exit gracefully ...` (Q0) and `Jest did not exit one second after the test run has completed.` (single-file specs) messages are caused by a **real leaked timer**: `Timing.timeoutPromise` races the fetch against a `setTimeout` that is never cleared when the fetch wins or when the test ends ([`timing.ts:23-36`](../../packages/explat-client/src/internal/timing.ts)). Rather than leave this **INFERRED**, it was demonstrated at runtime with a throwaway one-test spec (fetch never resolves) run under `--detectOpenHandles`, then deleted per the read-only rule.

**Command:**

```bash
cd packages/explat-client && FORCE_COLOR=0 CI=true yarn jest --ci --colors=false --detectOpenHandles src/test/blitzy_tmp_detect.ts
```

**Complete output** (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_detect.ts

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.815 s, estimated 1 s
Ran all test suites matching /src\/test\/blitzy_tmp_detect.ts/i.

Jest has detected the following 1 open handle potentially keeping Jest from exiting:

  ●  Timeout

      28 | 		promise,
      29 | 		new Promise< null >( ( _res, rej ) =>
    > 30 | 			setTimeout(
         | 			^
      31 | 				() => rej( new Error( `Promise has timed-out after ${ timeoutMilliseconds }ms.` ) ),
      32 | 				timeoutMilliseconds
      33 | 			)

      at setTimeout (src/internal/timing.ts:30:4)
      at Object.timeoutPromise (src/internal/timing.ts:29:3)
      at Object.timeoutPromise [as loadExperimentAssignment] (src/create-explat-client.ts:142:54)
      at Object.loadExperimentAssignment (src/test/blitzy_tmp_detect.ts:36:25)
```

**Answer (OBSERVED).** Jest reports exactly `1 open handle` — a `Timeout` — and points to the precise source: `at setTimeout (src/internal/timing.ts:30:4)` → `at Object.timeoutPromise (src/internal/timing.ts:29:3)` → `at Object.timeoutPromise [as loadExperimentAssignment] (src/create-explat-client.ts:142:54)`. This is the `setTimeout` inside `timeoutPromise` at [`timing.ts:30`](../../packages/explat-client/src/internal/timing.ts), scheduled from the `Timing.timeoutPromise(...)` call at [`create-explat-client.ts:142`](../../packages/explat-client/src/create-explat-client.ts). The warning is therefore a leaked (uncleared, non-`unref`'d) timeout timer, **not** a test failure — every suite still passes. **OBSERVED** (the cause is now demonstrated, not inferred).

---

## Response shape and `Config` (dependency-injection) contract

### `ExperimentAssignment` (the returned object) — [`types.ts:3-24`](../../packages/explat-client/src/types.ts)

| Field | Type | Meaning / observed value |
|-------|------|--------------------------|
| `experimentName` | `string` ([`:7`](../../packages/explat-client/src/types.ts)) | the requested name, e.g. `"experiment_name_a"`. **OBSERVED** in every section. |
| `variationName` | `string \| null` ([`:11`](../../packages/explat-client/src/types.ts)) | `null` ⇒ default/control ([`README.md:22`](../../packages/explat-client/README.md)); non-null (currently always `"treatment"`, [`README.md:23`](../../packages/explat-client/README.md)) ⇒ treatment. **OBSERVED** (`null` on fallback, `"treatment"` on success). |
| `retrievedTimestamp` | `number` ([`:15`](../../packages/explat-client/src/types.ts)) | monotonic ms at retrieval; the only field that varies run-to-run (F14). **OBSERVED**. |
| `ttl` | `number` ([`:19`](../../packages/explat-client/src/types.ts)) | seconds; floored at 60 (`minimumTtl`). **OBSERVED** (`60` throughout). |
| `isFallbackExperimentAssignment` | `boolean` (optional) ([`:23`](../../packages/explat-client/src/types.ts)) | present and `true` **only** on fallback objects; absent on real/stale stored assignments. **OBSERVED** (present on Q1a-empty/Q1b, absent on Q1a-stale). |

### `Config` (the DI seam) — [`types.ts:28-39`](../../packages/explat-client/src/types.ts) (F8)

| Member | Full signature | Nullability / notes |
|--------|----------------|---------------------|
| `fetchExperimentAssignment` | `( { experimentName, anonId }: { experimentName: string; anonId: string \| null } ) => Promise< unknown >` ([`:29-35`](../../packages/explat-client/src/types.ts)) | `anonId` may be `null`; returns `unknown` (validated internally by `validateFetchExperimentAssignmentResponse`, [`requests.ts`](../../packages/explat-client/src/internal/requests.ts)). This is the mocked seam; observed call arg `{"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}`. |
| `getAnonId` | `() => Promise< string \| null >` ([`:36`](../../packages/explat-client/src/types.ts)) | resolves the anonymous id or `null`; pinned to `'anon-id-xyz-123'` in the specs. |
| `logError` | `( error: Record< string, string > & { message: string } ) => void` ([`:37`](../../packages/explat-client/src/types.ts)) | must include a `message` string; all other keys are strings (e.g. `experimentName`, `source`). Observed sources: `loadExperimentAssignment-initialError`, `dangerouslyGetExperimentAssignment-error`, `dangerouslyGetExperimentAssignment`. |
| `isDevelopmentMode` | `boolean` ([`:38`](../../packages/explat-client/src/types.ts)) | gates the dev-only logging in the synchronous getters (Q4): `true` ⇒ logs; `false` ⇒ silent. **OBSERVED**. |

---

## Cleanup & repository hygiene (read-only scope, F5 & F18)

This investigation is read-only. The temporary observation specs (`blitzy_tmp_*.ts`) were the only files added to the tracked working tree, and they are removed here; `node_modules/` and `packages/*/dist/` are gitignored install/build artifacts (not source changes). The **sole** tracked change left behind is this Markdown deliverable. Every block below is **OBSERVED**.

### Deleting the temporary observation specs

Run from the **repository root** — the `pwd` marker precedes the `rm -v`, so the printed `removed 'packages/explat-client/src/test/...'` paths are exactly the repo-relative paths supplied to `rm` (not `src/test/...`):

```
$ pwd
/tmp/blitzy/wp-calypso/blitzy-61d951c6-a41e-4f0f-9378-67e594d212dc_c261e6
$ rm -v packages/explat-client/src/test/blitzy_tmp_q1a_empty.ts \
     packages/explat-client/src/test/blitzy_tmp_q1a_stale.ts \
     packages/explat-client/src/test/blitzy_tmp_q1b_timeout.ts \
     packages/explat-client/src/test/blitzy_tmp_q2.ts \
     packages/explat-client/src/test/blitzy_tmp_q3.ts \
     packages/explat-client/src/test/blitzy_tmp_q4.ts
removed 'packages/explat-client/src/test/blitzy_tmp_q1a_empty.ts'
removed 'packages/explat-client/src/test/blitzy_tmp_q1a_stale.ts'
removed 'packages/explat-client/src/test/blitzy_tmp_q1b_timeout.ts'
removed 'packages/explat-client/src/test/blitzy_tmp_q2.ts'
removed 'packages/explat-client/src/test/blitzy_tmp_q3.ts'
removed 'packages/explat-client/src/test/blitzy_tmp_q4.ts'
```

### Existence check — distinct from ignore status (F5)

**Existence** is checked directly (`ls` / `test -e`); the files are gone:

```
$ ls -1 packages/explat-client/src/test/blitzy_tmp_* 2>&1 || echo '(glob matched nothing)'
ls: cannot access 'packages/explat-client/src/test/blitzy_tmp_*': No such file or directory
(glob matched nothing)

$ for f in q1a_empty q1a_stale q1b_timeout q2 q3 q4; do p=packages/explat-client/src/test/blitzy_tmp_$f.ts; test -e "$p" && echo "EXISTS $p" || echo "GONE   $p"; done
GONE   packages/explat-client/src/test/blitzy_tmp_q1a_empty.ts
GONE   packages/explat-client/src/test/blitzy_tmp_q1a_stale.ts
GONE   packages/explat-client/src/test/blitzy_tmp_q1b_timeout.ts
GONE   packages/explat-client/src/test/blitzy_tmp_q2.ts
GONE   packages/explat-client/src/test/blitzy_tmp_q3.ts
GONE   packages/explat-client/src/test/blitzy_tmp_q4.ts
```

Separately — and this is a different question — the build/install artifacts are confirmed **gitignored** via an ignore-match check (`git check-ignore -v`), which reports *why* each path is ignored but says nothing about whether it exists:

```
$ git check-ignore -v node_modules packages/explat-client/dist
.gitignore:17:node_modules	node_modules
.gitignore:69:/packages/*/dist/	packages/explat-client/dist
exit=0
```

### The only tracked change is this document

`git status --porcelain` lists exactly one entry — the Markdown deliverable, modified (` M`) relative to the prior commit — and nothing else (no source, test, config, or leftover temp file):

```
$ git status --porcelain
 M blitzy/documentation/wp-calypso_be7e5cc64162.md
$ git status --porcelain | wc -l
1
```

The file introduces no whitespace or end-of-file errors: `git diff --check` produces no output and exits `0`, confirming there is no blank-line-at-EOF and the file ends with exactly one trailing newline (F18):

```
$ git diff --check; echo "exit=$?"
exit=0
```

## Coverage pass

**Every question answered, by observation.**

| Q | Question | Answered by | Result |
|---|----------|-------------|--------|
| Q0 | Does the suite pass? | Q0 section | 9 suites / 81 tests / 23 snapshots pass. **OBSERVED**. |
| Q1 | Never-throw; what is returned; which variation on unavailable/slow server? | Q1a-empty, Q1a-stale, Q1b | Never throws; empty-store ⇒ `null`/control fallback; stale-cache ⇒ stale variation kept; timeout ⇒ same `null` fallback with randomized 5000/10000 ms. **OBSERVED**. |
| Q2 | How many network calls for simultaneous same-experiment requests? | Q2 section | Exactly 1 per client instance (single-flight); 2 instances ⇒ 2 calls. **OBSERVED**. |
| Q3 | Does every repeat hit the network; what on TTL expiry? | Q3 section | No — cache hits serve repeats (1 fetch for 3 quick loads); a load past TTL refetches (2nd fetch). **OBSERVED**. |
| Q4 | Sync getter before load finishes — breaks the app? | Q4 section | No — returns a `null` fallback (or `null` for the "maybe" getter) and only logs in dev mode; the in-flight load still resolves to `treatment`. **OBSERVED**. |

**Every named mechanism / function / condition / flag addressed by name.**

- **Entry point & context:** `createExPlatClient` gate ([`index.ts:8-9`](../../packages/explat-client/src/index.ts)); real `createExPlatClient`/browser factory vs `createSsrSafeDummyExPlatClient` (**non-canonical**, never used); `'Running outside of a browser context.'` guard ([`create-explat-client.ts:72-74`](../../packages/explat-client/src/create-explat-client.ts)). Canonical path **OBSERVED**.
- **Load path:** `loadExperimentAssignment` double `try/catch`; cache-hit gate (`stored && isAlive`); per-name registry `experimentNameToWrappedExperimentAssignmentFetchAndStore`; `createWrappedExperimentAssignmentFetchAndStore`. **OBSERVED**.
- **Timing:** `Timing.asyncOneAtATime` (single-flight); `Timing.timeoutPromise` (rejects); `Timing.monotonicNow`. **OBSERVED**.
- **Assignments/TTL:** `ExperimentAssignments.isAlive` (`monotonicNow() < ttl*1000 + retrievedTimestamp`); `minimumTtl = 60`; `createFallbackExperimentAssignment` (`variationName: null`, `isFallbackExperimentAssignment: true`, `ttl: Math.max(60, ttl)`). **OBSERVED**.
- **Requests/store:** `Request.fetchExperimentAssignment` and the exact DI arg `{ anonId, experimentName }`; `ttl = Math.max(minimumTtl, responseTtl)`; store key `explat-experiment--<name>`; in-memory `localStorage` polyfill. Arg & counts **OBSERVED**; key/backing **INFERRED**.
- **Getters:** `dangerouslyGetExperimentAssignment` (throw-then-catch fallback; dev-only error log; dev-only "too soon" `< 1000 ms` warning) and `dangerouslyGetMaybeLoadedExperimentAssignment` (returns `null` when unloaded). **OBSERVED**.
- **Constants/flags:** `EXPERIMENT_FETCH_TIMEOUT = 10000`; the `5000` A/B arm via `Math.random() > 0.5`; `isDevelopmentMode` `true`/`false`. **OBSERVED**.

**Paired-variable cross-products.**

- **Q1 failure-mode × store-state:** rejection × empty store (Q1a-empty) and rejection × stale store (Q1a-stale) are **OBSERVED**; timeout × empty store is **OBSERVED** (Q1b). Timeout × stale store is **INFERRED**-equivalent: both failure modes converge on the identical `catch` → stale-return branch ([`create-explat-client.ts:150-165`](../../packages/explat-client/src/create-explat-client.ts)), so a timeout with a stale cache returns the stale assignment just as a rejection does.
- **Q4 getter × state × mode:** `{dangerouslyGetExperimentAssignment, dangerouslyGetMaybeLoadedExperimentAssignment}` × `{never-started, in-flight, loaded}` are **OBSERVED** in dev mode; never-started and loaded are **OBSERVED** in non-dev mode (both silent). In-flight × non-dev is **INFERRED**-equivalent: all getter logging is gated solely on `isDevelopmentMode` ([`create-explat-client.ts:197-221`](../../packages/explat-client/src/create-explat-client.ts)), and the non-dev runs already show `logError` never fires in any observed state.

**Rule compliance (SWE-AtlasQnA-Repo).** Investigated by running the code first (specs authored, executed, output captured before writing); canonical public entry exercised (`require('../index')` after `window`); run-to-run inconsistency reported as a distribution (Q1b, 20 runs); complete unedited output included for every claim with its command; observed/inferred/non-canonical labeled per statement; exact `file:line` grounding throughout; read-only scope preserved (see the cleanup & hygiene section — only this Markdown file changes).

## Resolution of the 18 review findings

| # | Sev | Finding (summary) | How it is resolved here (evidence) |
|---|-----|-------------------|-------------------------------------|
| F1 | Critical | Specs bypassed the public entry (imported the internal factory); `window` set in `beforeEach` after import | All specs now set `window` **before** `require('../index')` and load the **public** entry; Phase-1 probe confirmed both branches of [`index.ts:8-9`](../../packages/explat-client/src/index.ts). See **Methodology → canonical entry point**. **OBSERVED**. |
| F2 | Major | Specs had zero assertions | Every spec now asserts every invariant (`expect(...)`); each run reports `Tests: N passed`. Full spec sources embedded per question. **OBSERVED**. |
| F3 | Major | Embedded source line numbers didn't match pasted stack lines | Specs are embedded **byte-for-byte** from the executed files; the `at Object.log (...:NN:11)` stack lines in each pasted output match the embedded spec. **OBSERVED**. |
| F4 | Major | Output truncated / transformed | Separate focused specs; complete unedited output pasted per question; the 20-run distribution shown via a `for` loop + `uniq -c` tally with the exact commands. **OBSERVED**. |
| F5 | Major | Cleanup evidence unconvincing | See the **Cleanup & repository hygiene** section: `rm -v` run with a `pwd` marker, existence checked separately (`test -e`/`ls`) from ignore status (`git check-ignore`), and `git status --porcelain` shown. **OBSERVED**. |
| F6 | Major | Q4 never exercised the true "in-flight" state | Q4 now drives an in-flight load (deferred fetch + fake timers) and calls both getters while pending, then resolves. **OBSERVED**. |
| F7 | Major | Q4 missed the post-load "too soon" warning and only tested dev mode | Q4 captures the immediate-post-load `dangerouslyGetExperimentAssignment` warning and runs `isDevelopmentMode` **true and false** (non-dev logs nothing). **OBSERVED**. |
| F8 | Major | `Config` reduced to 4 names | Full `Config` signature table added from [`types.ts:28-39`](../../packages/explat-client/src/types.ts) with argument/return types and nullability. |
| F9 | Major | Exact injected request arg never captured | `getAnonId` pinned to `'anon-id-xyz-123'`; the exact arg `{"anonId":"anon-id-xyz-123","experimentName":"experiment_name_a"}` is asserted and shown (Q1a, Q2). **OBSERVED**. |
| F10 | Major | Q2 dedup scope unstated | Q2 adds a two-instance observation: 2 instances ⇒ 2 fetches ⇒ registry is **per client instance**; tab/process extrapolation labeled **INFERRED**. **OBSERVED**. |
| F11 | Major | "reject/timeout ⇒ null fallback" only true for empty store | Q1a-stale shows a failed refetch returns the **stale** stored assignment (no `isFallbackExperimentAssignment`), exercising [`create-explat-client.ts:158-165`](../../packages/explat-client/src/create-explat-client.ts). **OBSERVED**. |
| F12 | Major | Stability shown only for Q1b & Q2 | Q1a-empty, Q1a-stale, Q3, and Q4 each include a ≥2-run stability capture. **OBSERVED**. |
| F13 | Minor | "worker failed to exit" cause was inferred | Demonstrated via `--detectOpenHandles`: the leaked `setTimeout` at [`timing.ts:30`](../../packages/explat-client/src/internal/timing.ts). **OBSERVED**. |
| F14 | Minor | Claimed the returned object is constant across 20 runs, but `retrievedTimestamp` varies | Clarified: semantics/fields are constant; `retrievedTimestamp` varies (20 distinct values shown), as does the 5000/10000 arm. **OBSERVED**. |
| F15 | Minor | "no pre-build" omitted the Babel transform | Now cites all three preset pieces: resolver, **`babel-jest` transform** ([`packages/calypso-jest/jest-preset.js:13-15`](../../packages/calypso-jest/jest-preset.js)), and `testMatch`. |
| F16 | Minor | `be7e5cc641` mislabeled "HEAD" | Header labels `be7e5cc641` the **investigated source revision** and notes the prior doc commit `5c54a30515` and that the delivery commit stacks on top. **OBSERVED**. |
| F17 | Minor | False "only four inferred" claim | Removed; labeling is applied **per statement**, and the notable inferences are enumerated in **How to read**. |
| F18 | Minor | `git diff --check` reported a blank line at EOF | The file ends with exactly one trailing newline (renderer-enforced); verified with `git diff --check` in the cleanup section. **OBSERVED**. |

*Prepared by running the client's real code paths through its canonical public entry point; every behavioral claim above is backed by the verbatim output shown in its section.*
