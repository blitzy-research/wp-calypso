# `@automattic/explat-client` — Behavioral Investigation (Q0–Q4)

> **Branch:** `wp-calypso_be7e5cc64162` · **HEAD:** `be7e5cc641` · **Package:** `@automattic/explat-client@0.1.0` ([`packages/explat-client/package.json:2-3`](../../packages/explat-client/package.json))
>
> **Toolchain used (canonical, repo-authoritative):** Node.js `v22.23.1`, Yarn `4.0.2`.

This document answers five behavioral questions about the standalone experiment-assignment client at `packages/explat-client/`. **Every behavioral claim is grounded in runtime output that was captured by actually running the client's code** (the network boundary is mocked at the client's own dependency-injection seam — the production server was never contacted; this is stated plainly wherever relevant). Read-only source analysis is used only to name the responsible function/value, and such statements are labeled **INFERRED**.

## How to read this document (OBSERVED vs INFERRED)

- **OBSERVED** — the statement is demonstrated by captured runtime output that is pasted verbatim below it, together with the exact command that produced it.
- **INFERRED** — the statement is derived from reading the source (with a `file:line` citation) and was **not** directly exercised at runtime. Examples in this doc: the production TTL of 3600 s (only the README states it; the production server was not contacted), the industry-standard name "single-flight / request-coalescing", and the SSR-dummy code path (deliberately **not** exercised and labeled **non-canonical**).
- Every behavioral sentence carries a `file:line` citation into `packages/explat-client/**` and an OBSERVED/INFERRED tag.

---

## Methodology & canonical harness

### The canonical entry point requires a browser context

The real client only runs when `window` is defined. `createExPlatClient` throws `'Running outside of a browser context.'` when `typeof window === 'undefined'` ([`create-explat-client.ts:72-74`](../../packages/explat-client/src/create-explat-client.ts)). The public export gate resolves to the SSR **dummy** (`createSsrSafeDummyExPlatClient`, [`create-explat-client.ts:258-283`](../../packages/explat-client/src/create-explat-client.ts)) in plain Node and to the real browser client only when `window` exists ([`index.ts:8-9`](../../packages/explat-client/src/index.ts)). **INFERRED** (the SSR-dummy branch was not exercised and any value from it would be **non-canonical**).

Therefore every observation spec sets `global.window = {}` via `setBrowserContext()` ([`test-common.ts:22-26`](../../packages/explat-client/src/internal/test-common.ts)) so the real browser client is exercised. **OBSERVED** — every spec below constructs the client through `createExPlatClient` after `setBrowserContext()` and all four tests pass, proving the real (non-dummy) client was built.

### The dependency-injection seam is the canonical observation point

The network boundary is `config.fetchExperimentAssignment({ anonId, experimentName })` ([`requests.ts:80-91`](../../packages/explat-client/src/internal/requests.ts), the call itself at `:87-90`). The client is explicitly designed around dependency injection: the network call, anon-id retrieval, error logging, and dev-mode flag are all supplied through the `Config` object ([`types.ts:28-39`](../../packages/explat-client/src/types.ts)). Substituting a Jest mock here is the client's **designed, canonical** extension point — **not** a bypass. Counting the mock's calls is how Q2 and Q3 are measured. **INFERRED** (design intent, from the `Config` interface); the mock is used as **OBSERVED** evidence below.

### No pre-build is required for the tests

`packages/explat-client/jest.config.js:2` sets `preset: '../../test/packages/jest-preset.js'`, which wraps `@automattic/calypso-jest` ([`packages/calypso-jest/jest-preset.js`](../../packages/calypso-jest/jest-preset.js)). That preset uses a custom resolver that maps the `calypso:src` field to **untranspiled TypeScript source**, sets `testEnvironment: 'node'` (`packages/calypso-jest/jest-preset.js:11`), and `testMatch: ['<rootDir>/**/test/*.[jt]s?(x)', '!**/.eslintrc.*']` (`packages/calypso-jest/jest-preset.js:12`). So Jest runs directly against the `.ts` files, and any spec placed under `packages/explat-client/src/test/` is auto-discovered. **INFERRED** (from the preset config); corroborated **OBSERVED** by Q0 running the `.ts` suites directly with no build step.

### Runtime version reconciliation (canonical configuration)

The environment setup instructions mentioned Node 20.x, but the repository is authoritative and requires Node 22.x — `engines.node = "^v22.9.0"` and `.nvmrc = 22.9.0`, with `packageManager = "yarn@4.0.2"`. The **repo-authoritative Node 22.x** was used (observed `v22.23.1`) so the client runs exactly as the project intends. This document reports that canonical configuration. **OBSERVED:**

```bash
node --version && yarn --version && cat .nvmrc
```
```
v22.23.1
4.0.2
22.9.0
```

Workspace install (non-interactive, from the repository root):

```bash
CI=true yarn install
```
```
➤ YN0000: · Yarn 4.0.2
➤ YN0000: ┌ Resolution step
➤ YN0000: └ Completed in 0s 496ms
➤ YN0000: ┌ Fetch step
➤ YN0000: └ Completed in 4s 536ms
➤ YN0000: ┌ Link step
➤ YN0000: └ Completed in 1s 69ms
➤ YN0000: · Done in 12s 212ms
```
The install exited `0`. (It completes quickly here because the workspace dependencies were already present in the environment cache; a cold install takes longer and may also fetch Playwright browsers — that is normal and does not affect the client.)

### Harness details (mirrors the existing suite)

The observation specs mirror the existing suite `packages/explat-client/src/test/create-explat-client.ts`:

- Import `'@automattic/calypso-polyfills'` **first** (fixes the `regeneratorRuntime is not defined` error) — as the existing suite does at `src/test/create-explat-client.ts:2`.
- `createMockedConfig` returns `{ logError: jest.fn(), fetchExperimentAssignment: jest.fn(), getAnonId: jest.fn(), isDevelopmentMode: false, ...override }` — mirroring `src/test/create-explat-client.ts:21-27`.
- `delayedValue(value, ms)` ([`test-common.ts:16-17`](../../packages/explat-client/src/internal/test-common.ts)) simulates network latency; `ONE_DELAY = 1` ([`test-common.ts:20`](../../packages/explat-client/src/internal/test-common.ts)).
- A module-level `jest.spyOn(Timing, 'monotonicNow')` controls time; `beforeEach` runs `jest.resetAllMocks(); setBrowserContext(); localStorage.clear();`.

**CRITICAL harness insight (saves the reader hours):** `jest.resetAllMocks()` **neuters** the `jest.spyOn(Timing, 'monotonicNow')` spy so it returns `undefined`, which corrupts `retrievedTimestamp` and produces an *invalid* assignment (triggering a spurious secondary `loadExperimentAssignment-fallbackError` "Invalid ExperimentAssignment" log). The fix used to obtain canonical output is to reinstate a strictly-increasing clock in `beforeEach` **after** `resetAllMocks`, mirroring the real `monotonicNow` ([`timing.ts:10-16`](../../packages/explat-client/src/internal/timing.ts)):

```ts
let clock = Date.now();
spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
```

Individual tests may override this (e.g. Q3 pins a constant value to keep a cached assignment alive, then jumps past the TTL).

### Storage backing

Under a bare `window = {}`, `window.localStorage` is undefined, so the client uses its in-memory polyfill ([`local-storage.ts:33-40`](../../packages/explat-client/src/internal/local-storage.ts)). The cache key is `explat-experiment--<name>` (**double dash**): `localStorageExperimentAssignmentKeyPrefix = 'explat-experiment-'` ([`experiment-assignment-store.ts:9`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)) plus `-` plus the name ([`experiment-assignment-store.ts:14-15`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)). **INFERRED** (from the store source); the cache-hit counts in Q3 are **OBSERVED**.

### The two temporary observation specs (ephemeral; deleted afterward)

The full text of the two specs is included below so every claim is reproducible. They were placed under `packages/explat-client/src/test/` so the preset `testMatch` discovers them, run to capture output, and then **deleted** (see the Cleanup section). Jest prints `console.log` to **stderr**, so output was captured with `2>&1`.

<details>
<summary><code>src/test/blitzy_tmp_observation.ts</code> (Q1a, Q2, Q3, Q4)</summary>

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — created to capture runtime behavior for Q1a, Q2, Q3, Q4.
// This file is ephemeral and MUST be deleted after capturing output; it is never committed.
// This is required to fix the "regeneratorRuntime is not defined" error
import '@automattic/calypso-polyfills';

import { createExPlatClient } from '../create-explat-client';
import localStorage from '../internal/local-storage';
import { delayedValue, ONE_DELAY, setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

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
	// resetAllMocks() neuters the monotonicNow spy (would return undefined and corrupt
	// retrievedTimestamp). Reinstate a strictly-increasing clock that mirrors the real
	// monotonicNow (timing.ts:10-16).
	let clock = Date.now();
	spiedMonotonicNow.mockImplementation( () => ( clock += 1 ) );
} );

describe( 'Q1A server-unavailable (fetch rejects)', () => {
	it( 'never throws; returns null-variation fallback', async () => {
		const mockedConfig = createMockedConfig();
		( mockedConfig.fetchExperimentAssignment as MockedFunction ).mockImplementation(
			() =>
				new Promise( ( _res, rej ) =>
					rej( new Error( 'ECONNREFUSED simulated-server-unavailable' ) )
				)
		);
		const client = createExPlatClient( mockedConfig );
		let threw = false;
		let result = null;
		try {
			result = await client.loadExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			threw = true;
		}
		console.log( 'Q1A_THREW=' + threw );
		console.log( 'Q1A_RESULT=' + JSON.stringify( result ) );
		console.log(
			'Q1A_FETCH_CALLS=' +
				( mockedConfig.fetchExperimentAssignment as MockedFunction ).mock.calls.length
		);
		console.log(
			'Q1A_LOGERROR=' +
				JSON.stringify( ( mockedConfig.logError as MockedFunction ).mock.calls )
		);
	} );
} );

describe( 'Q2 concurrency / request deduplication (single-flight)', () => {
	it( 'N=8 simultaneous loads of the same name => exactly 1 fetch', async () => {
		const N = 8;
		const mockedConfig = createMockedConfig();
		( mockedConfig.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
			delayedValue(
				{ ttl: 60, variations: { experiment_name_a: 'treatment' } },
				ONE_DELAY
			)
		);
		const client = createExPlatClient( mockedConfig );
		const results = await Promise.all(
			Array.from( { length: N } ).map( () =>
				client.loadExperimentAssignment( 'experiment_name_a' )
			)
		);
		const allEqual = results.every(
			( r ) => JSON.stringify( r ) === JSON.stringify( results[ 0 ] )
		);
		console.log( 'Q2_N=' + N );
		console.log(
			'Q2_FETCH_CALLS=' +
				( mockedConfig.fetchExperimentAssignment as MockedFunction ).mock.calls.length
		);
		console.log( 'Q2_ALL_EQUAL=' + allEqual );
		console.log( 'Q2_RESULT0=' + JSON.stringify( results[ 0 ] ) );
	} );
} );

describe( 'Q3 caching behavior / TTL', () => {
	it( '3 quick loads => 1 fetch; load past TTL => 2nd fetch', async () => {
		const mockedConfig = createMockedConfig();
		( mockedConfig.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
			delayedValue(
				{ ttl: 60, variations: { experiment_name_a: 'treatment' } },
				ONE_DELAY
			)
		);
		const client = createExPlatClient( mockedConfig );

		// Hold the clock constant so the cached assignment stays alive.
		const firstDate = Date.now();
		spiedMonotonicNow.mockImplementation( () => firstDate );

		await client.loadExperimentAssignment( 'experiment_name_a' );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		console.log(
			'Q3_FETCH_AFTER_3_QUICK=' +
				( mockedConfig.fetchExperimentAssignment as MockedFunction ).mock.calls.length
		);

		// Advance the clock strictly past the 60s TTL (ttl*1000 + retrievedTimestamp).
		spiedMonotonicNow.mockImplementation( () => firstDate + 60 * 1000 + 1 );
		await client.loadExperimentAssignment( 'experiment_name_a' );
		console.log(
			'Q3_FETCH_AFTER_TTL=' +
				( mockedConfig.fetchExperimentAssignment as MockedFunction ).mock.calls.length
		);
	} );
} );

describe( 'Q4 async load vs synchronous getters', () => {
	it( 'sync getters before load do not break the app; both states observed', async () => {
		const mockedConfig = createMockedConfig( { isDevelopmentMode: true } );
		( mockedConfig.fetchExperimentAssignment as MockedFunction ).mockImplementation( () =>
			delayedValue(
				{ ttl: 60, variations: { experiment_name_a: 'treatment' } },
				ONE_DELAY
			)
		);
		const client = createExPlatClient( mockedConfig );

		// --- NOT-YET-LOADED state ---
		let dgetThrew = false;
		let dgetResult = null;
		try {
			dgetResult = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		} catch ( e ) {
			dgetThrew = true;
		}
		console.log( 'Q4_DGET_NOTLOADED_THREW=' + dgetThrew );
		console.log( 'Q4_DGET_NOTLOADED_RESULT=' + JSON.stringify( dgetResult ) );
		console.log(
			'Q4_DGET_NOTLOADED_LOGERROR=' +
				JSON.stringify( ( mockedConfig.logError as MockedFunction ).mock.calls )
		);

		const dmaybeNotLoaded =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		console.log( 'Q4_DMAYBE_NOTLOADED_RESULT=' + JSON.stringify( dmaybeNotLoaded ) );

		// --- LOADED state ---
		await client.loadExperimentAssignment( 'experiment_name_a' );
		const dgetLoaded = client.dangerouslyGetExperimentAssignment( 'experiment_name_a' );
		const dmaybeLoaded =
			client.dangerouslyGetMaybeLoadedExperimentAssignment( 'experiment_name_a' );
		console.log( 'Q4_DGET_LOADED=' + JSON.stringify( dgetLoaded ) );
		console.log( 'Q4_DMAYBE_LOADED=' + JSON.stringify( dmaybeLoaded ) );
	} );
} );
```
</details>

<details>
<summary><code>src/test/blitzy_tmp_timeout.ts</code> (Q1b timeout distribution)</summary>

```ts
// TEMPORARY OBSERVATION SPEC (blitzy_tmp_*) — Q1b: slow server (fetch never resolves => timeout).
// Ephemeral; MUST be deleted after capturing output; never committed.
// This is required to fix the "regeneratorRuntime is not defined" error
import '@automattic/calypso-polyfills';

import { createExPlatClient } from '../create-explat-client';
import localStorage from '../internal/local-storage';
import { setBrowserContext } from '../internal/test-common';
import * as Timing from '../internal/timing';
import type { Config } from '../types';

type MockedFunction = ReturnType< typeof jest.fn >;

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

describe( 'Q1B slow server (fetch never resolves => timeout)', () => {
	it( 'never throws; returns null-variation fallback; records the randomized timeout', async () => {
		jest.useFakeTimers();
		const mockedConfig = createMockedConfig();
		// A promise that never resolves nor rejects — simulates a hung/slow server.
		( mockedConfig.fetchExperimentAssignment as MockedFunction ).mockImplementation(
			() => new Promise( () => undefined )
		);
		const client = createExPlatClient( mockedConfig );

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

		// Advance fake timers far enough to trip either the 5000ms or 10000ms timeout,
		// flushing intervening microtasks (getAnonId resolution, Promise.race settle).
		await jest.advanceTimersByTimeAsync( 10 * 1000 );
		await promise;

		const logErrorCalls = ( mockedConfig.logError as MockedFunction ).mock.calls;
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

		jest.useRealTimers();
	} );
} );
```
</details>

---

## Q0 — Test baseline (run the suite)

**Question:** Does the client package's test suite pass cleanly, establishing a trustworthy baseline before any behavioral claim?

The package's `test` script is `yarn jest` ([`package.json:25`](../../packages/explat-client/package.json)). The canonical command and its **complete, unedited** output:

```bash
cd packages/explat-client && CI=true yarn jest --ci
```
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
PASS src/test/index.ts
PASS src/internal/test/experiment-assignment-store.ts
PASS src/internal/test/validations.ts
PASS src/internal/test/local-storage.ts
PASS src/internal/test/timing.ts
PASS src/internal/test/experiment-assignments.ts
PASS src/test/create-ssr-safe-dummy-explat-client.ts
PASS src/test/create-explat-client.ts
A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --detectOpenHandles to find leaks. Active timers can also cause this, ensure that .unref() was called on them.

Test Suites: 9 passed, 9 total
Tests:       81 passed, 81 total
Snapshots:   23 passed, 23 total
Time:        2.465 s, estimated 3 s
Ran all test suites.
```

**OBSERVED:** the baseline is trustworthy — **9 suites / 81 tests / 23 snapshots all pass** (exit code `0`). This matches the expected totals exactly; only the wall-clock `Time` legitimately varies between runs.

The 9 test files (= 9 suites), confirmed via `find src -path '*/test/*' -name '*.ts'`:

```
src/internal/test/experiment-assignment-store.ts
src/internal/test/experiment-assignments.ts
src/internal/test/local-storage.ts
src/internal/test/requests.ts
src/internal/test/timing.ts
src/internal/test/validations.ts
src/test/create-explat-client.ts
src/test/create-ssr-safe-dummy-explat-client.ts
src/test/index.ts
```

**Benign, non-failing messages** (labeled so a reader isn't alarmed):

- `A worker process has failed to exit gracefully and has been force exited...` — **OBSERVED**. This is caused by a **leaked real timer** from `Timing.timeoutPromise`'s `setTimeout` ([`timing.ts:23-36`](../../packages/explat-client/src/internal/timing.ts)), which is not cleared when the fetch wins the `Promise.race`. It does **not** fail any test (the suite still reports `9 passed` / exit `0`).
- `Browserslist: browsers data (caniuse-lite) is 17 months old` — **OBSERVED**, informational only (offline environment), unrelated to the client.

---

## Q1 — "Designed to never throw" (server unavailable + timeout)

**Question:** The client is documented as "Designed to never throw" ([`README.md:44`](../../packages/explat-client/README.md)). What happens when the assignment server is unavailable (the network call rejects) or takes too long (the fetch times out): what does the returned object look like, and which variation is the user assigned?

Documented contract framing: "Designed to never throw" ([`README.md:44`](../../packages/explat-client/README.md)); "Change dangerouslyGetExperimentAssignment to log rather than throw" ([`CHANGELOG.md:21`](../../packages/explat-client/CHANGELOG.md)); "Shortened timeout" ([`CHANGELOG.md:17`](../../packages/explat-client/CHANGELOG.md)).

### Q1a — Server unavailable (fetch rejects)

The fetch stub rejects with `ECONNREFUSED simulated-server-unavailable`. Command and **complete, unedited** output (`console.log` goes to stderr, captured with `2>&1`):

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_observation.ts 2>&1
```
```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_observation.ts
  ● Console

    console.log
      Q1A_THREW=false

      at Object.log (src/test/blitzy_tmp_observation.ts:53:11)

    console.log
      Q1A_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783961825553,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_observation.ts:55:11)

    console.log
      Q1A_FETCH_CALLS=1

      at Object.log (src/test/blitzy_tmp_observation.ts:57:11)

    console.log
      Q1A_LOGERROR=[[{"message":"ECONNREFUSED simulated-server-unavailable","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]]

      at Object.log (src/test/blitzy_tmp_observation.ts:62:11)
```

*(The remainder of this run's output — the Q2/Q3/Q4 markers and the `Test Suites: 1 passed` summary — is reproduced in the Q2, Q3, and Q4 sections below, since all four questions share this one spec file.)*

**OBSERVED findings:**

- `Q1A_THREW=false` — the client did **not** throw. This is the outer `try/catch` of `loadExperimentAssignment` converting the rejection into a fallback ([`create-explat-client.ts:113-183`](../../packages/explat-client/src/create-explat-client.ts)).
- `Q1A_FETCH_CALLS=1` — the injected `config.fetchExperimentAssignment` was invoked exactly once ([`requests.ts:87-90`](../../packages/explat-client/src/internal/requests.ts)).
- `Q1A_LOGERROR=[[{"message":"ECONNREFUSED simulated-server-unavailable","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]]` — the failure was logged **once** with source `loadExperimentAssignment-initialError`, from the initial `catch` ([`create-explat-client.ts:151-157`](../../packages/explat-client/src/create-explat-client.ts)).
- `Q1A_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783961825553,"ttl":60,"isFallbackExperimentAssignment":true}` — the returned object is the **fallback**: `variationName: null`, `ttl: 60`, `isFallbackExperimentAssignment: true`. This is produced by the fallback branch ([`create-explat-client.ts:160-172`](../../packages/explat-client/src/create-explat-client.ts)) via `createFallbackExperimentAssignment` ([`experiment-assignments.ts:29-38`](../../packages/explat-client/src/internal/experiment-assignments.ts)), whose `ttl` is `Math.max(minimumTtl, ttl)` with `minimumTtl = 60` ([`experiment-assignments.ts:21`](../../packages/explat-client/src/internal/experiment-assignments.ts)).

**Which variation does the user get?** `variationName === null`, which the contract defines as the **default (control)** experience ([`README.md:20-24`](../../packages/explat-client/README.md), specifically line 22: "`variationName === null`: This means you should return the default experience."). **OBSERVED** (the null value) + **INFERRED** (its "default/control" meaning, from the README).

### Q1b — Slow server (fetch never resolves → timeout)

The fetch stub is `() => new Promise(() => undefined)` (never settles). Fake timers are advanced past the timeout so no real waiting occurs. Command and **complete, unedited** output of a single run:

```bash
cd packages/explat-client && CI=true yarn jest --ci src/test/blitzy_tmp_timeout.ts 2>&1
```
```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS src/test/blitzy_tmp_timeout.ts
  ● Console

    console.log
      Q1B_THREW=false

      at Object.log (src/test/blitzy_tmp_timeout.ts:68:11)

    console.log
      Q1B_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783961971436,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_timeout.ts:70:11)

    console.log
      Q1B_ERROR_MESSAGE="Promise has timed-out after 5000ms."

      at Object.log (src/test/blitzy_tmp_timeout.ts:72:11)

    console.log
      Q1B_ERROR_SOURCE="loadExperimentAssignment-initialError"

      at Object.log (src/test/blitzy_tmp_timeout.ts:74:11)

    console.log
      Q1B_TIMEOUT_MS=5000

      at Object.log (src/test/blitzy_tmp_timeout.ts:76:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.922 s
Ran all test suites matching /src\/test\/blitzy_tmp_timeout.ts/i.
```

**OBSERVED findings:**

- `Q1B_THREW=false` — no throw on timeout either.
- `Q1B_ERROR_MESSAGE="Promise has timed-out after 5000ms."` — the timeout is enforced by `Timing.timeoutPromise`, which **rejects** (it does not resolve) with `` `Promise has timed-out after ${timeoutMilliseconds}ms.` `` ([`timing.ts:23-36`](../../packages/explat-client/src/internal/timing.ts), string at line 31). That rejection is what the outer `try/catch` converts into the fallback, and it is the source of the benign "worker failed to exit gracefully" warning (the leaked `setTimeout`).
- `Q1B_ERROR_SOURCE="loadExperimentAssignment-initialError"` — same log source as Q1a ([`create-explat-client.ts:151-157`](../../packages/explat-client/src/create-explat-client.ts)).
- `Q1B_RESULT={...,"variationName":null,...,"ttl":60,"isFallbackExperimentAssignment":true}` — the **same null-variation fallback** as Q1a.

#### Randomized timeout — distribution over 20 identical runs

`EXPERIMENT_FETCH_TIMEOUT = 10000` ([`create-explat-client.ts:16`](../../packages/explat-client/src/create-explat-client.ts)) is **halved to `5000`** when `Math.random() > 0.5` ([`create-explat-client.ts:134-138`](../../packages/explat-client/src/create-explat-client.ts)). Per the run-to-run-inconsistency rule, the **same unchanged spec was run 20 times** (scale = 20) **without** mocking `Math.random`, and the observed distribution is reported (not stabilized).

```bash
cd packages/explat-client
for i in $(seq 1 20); do
  CI=true yarn jest --ci src/test/blitzy_tmp_timeout.ts 2>&1 \
    | grep -oE 'Q1B_TIMEOUT_MS=[0-9]+' | cut -d= -f2
done
```

Raw sequence captured (20 runs, in order):

```
10000 5000 10000 5000 10000 10000 5000 10000 10000 10000 10000 10000 10000 5000 5000 10000 5000 10000 5000 10000
```

| Timeout chosen | Count (of 20) | Percentage | Source |
|----------------|---------------|------------|--------|
| `5000 ms`      | 7             | 35%        | `Math.random() > 0.5` branch ([`create-explat-client.ts:136-137`](../../packages/explat-client/src/create-explat-client.ts)) |
| `10000 ms`     | 13            | 65%        | `EXPERIMENT_FETCH_TIMEOUT` default ([`create-explat-client.ts:16`](../../packages/explat-client/src/create-explat-client.ts)) |
| other / UNKNOWN | 0            | 0%         | — |

**OBSERVED:** both `5000 ms` and `10000 ms` occur; the split is genuinely random and varies between runs (this particular sample landed 7/13, consistent with an underlying ~50/50 coin flip within normal variance for 20 trials). The invariant behavior held across **all 20 runs** — verified by aggregating the per-run markers:

```bash
grep -h 'Q1B_THREW=' q1b_run_*.log | sort | uniq -c
grep -h 'Q1B_RESULT=' q1b_run_*.log | grep -oE '"variationName":[^,]*' | sort | uniq -c
grep -h 'Q1B_RESULT=' q1b_run_*.log | grep -oE '"ttl":[0-9]+,"isFallbackExperimentAssignment":true' | sort | uniq -c
grep -h 'Q1B_ERROR_SOURCE=' q1b_run_*.log | sort | uniq -c
```
```
     20 Q1B_THREW=false
     20 "variationName":null
     20 "ttl":60,"isFallbackExperimentAssignment":true
     20 Q1B_ERROR_SOURCE="loadExperimentAssignment-initialError"
```

Only the chosen timeout (`5000`/`10000` ms) varies run-to-run; the returned object, the null variation, the `ttl: 60`, and the log source are constant across all 20 runs.

### Q1 conclusion

**OBSERVED:** the client never throws in either failure mode (server-unavailable *or* timeout). In both cases it returns `{ experimentName, variationName: null, retrievedTimestamp: <number>, ttl: 60, isFallbackExperimentAssignment: true }` — the **default/control** experience — after logging the failure **once** with source `loadExperimentAssignment-initialError`. This confirms the "Designed to never throw" contract ([`README.md:44`](../../packages/explat-client/README.md)).

---


## Q2 — Concurrency / request deduplication (single-flight)

**Question:** How many network calls are actually made when multiple parts of the application request the **same** experiment assignment simultaneously? (The client promises it will "only make one request at a time" per experiment — [`create-explat-client.ts:22-23`](../../packages/explat-client/src/create-explat-client.ts).)

**N = 8** simultaneous `loadExperimentAssignment('experiment_name_a')` calls are fired under `Promise.all`; the fetch stub returns `delayedValue({ ttl: 60, variations: { experiment_name_a: 'treatment' } }, ONE_DELAY)`. The relevant markers from the shared spec run (`src/test/blitzy_tmp_observation.ts`, same command as Q1a):

```
    console.log
      Q2_N=8

      at Object.log (src/test/blitzy_tmp_observation.ts:89:11)

    console.log
      Q2_FETCH_CALLS=1

      at Object.log (src/test/blitzy_tmp_observation.ts:91:11)

    console.log
      Q2_ALL_EQUAL=true

      at Object.log (src/test/blitzy_tmp_observation.ts:96:11)

    console.log
      Q2_RESULT0={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825609,"ttl":60}

      at Object.log (src/test/blitzy_tmp_observation.ts:98:11)
```

**OBSERVED findings:**

- `Q2_FETCH_CALLS=1` — despite **8** concurrent callers, the injected `config.fetchExperimentAssignment` was invoked **exactly once** ([`requests.ts:87-90`](../../packages/explat-client/src/internal/requests.ts)).
- `Q2_ALL_EQUAL=true` — all 8 callers received the identical result object.
- `Q2_RESULT0={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825609,"ttl":60}` — the shared result is the real (non-fallback) `treatment` assignment.

**Stability across runs.** The whole spec was run **3 times** total (`obs_run1/2/3.log`); the deduplication count was constant:

```bash
for i in 1 2 3; do grep -oE 'Q2_FETCH_CALLS=[0-9]+|Q2_ALL_EQUAL=(true|false)' /tmp/blitzy_scratch/obs_run$i.log; done
```
```
Q2_FETCH_CALLS=1
Q2_ALL_EQUAL=true
Q2_FETCH_CALLS=1
Q2_ALL_EQUAL=true
Q2_FETCH_CALLS=1
Q2_ALL_EQUAL=true
```

**Responsible mechanism (by name).** Deduplication is produced by two cooperating pieces:

1. The single-flight primitive `Timing.asyncOneAtATime` ([`timing.ts:44-54`](../../packages/explat-client/src/internal/timing.ts)), which memoizes `lastPromise` and returns the same in-flight promise to every caller until it settles, resetting to `null` in `.finally` ([`timing.ts:48-50`](../../packages/explat-client/src/internal/timing.ts)).
2. The per-experiment-name registry `experimentNameToWrappedExperimentAssignmentFetchAndStore` ([`create-explat-client.ts:82-94`](../../packages/explat-client/src/create-explat-client.ts)), which stores exactly one wrapped fetch-and-store function per experiment name and reuses it ([`create-explat-client.ts:127-132`](../../packages/explat-client/src/create-explat-client.ts)).

**Pattern name (INFERRED terminology, grounded in the observed behavior).** This is the industry-standard **single-flight / request-coalescing** pattern (a.k.a. request deduplication) — a defense against the "thundering herd" / cache-stampede problem, in which many identical concurrent requests are merged into one backend call while the remaining callers await the shared result. A recognized trade-off is the shared **"error blast radius"**: when the single in-flight call fails, all waiting callers receive the same failure — exactly the fallback-to-every-caller behavior Q1 demonstrates on failure. **INFERRED** (the terminology and trade-off framing are read-only; the 1-call count and shared-result equality are **OBSERVED** above).

---

## Q3 — Caching behavior / TTL

**Question:** Does each repeated request for the same experiment reach the network, and what happens once the cached assignment's TTL expires?

Fetch counts are observed **before** and **after** TTL expiry within one test. The relevant markers from the shared spec run:

```
    console.log
      Q3_FETCH_AFTER_3_QUICK=1

      at Object.log (src/test/blitzy_tmp_observation.ts:121:11)

    console.log
      Q3_FETCH_AFTER_TTL=2

      at Object.log (src/test/blitzy_tmp_observation.ts:130:11)
```

**OBSERVED findings:**

- `Q3_FETCH_AFTER_3_QUICK=1` — three quick repeated loads of the same name (with `monotonicNow` pinned constant) triggered **only one** network call; the 2nd and 3rd loads were **cache hits**.
- `Q3_FETCH_AFTER_TTL=2` — after advancing `monotonicNow` to `firstDate + 60*1000 + 1` (just past the 60 s TTL) and loading again, a **second** network call was made (a refetch).

**Responsible code (by name).**

- The cache-hit gate returns the stored assignment **without** a network call when `isAlive` is true ([`create-explat-client.ts:119-125`](../../packages/explat-client/src/create-explat-client.ts)).
- `isAlive` computes `monotonicNow() < ttl * MILLISECONDS_PER_SECOND + retrievedTimestamp` ([`experiment-assignments.ts:8-14`](../../packages/explat-client/src/internal/experiment-assignments.ts)). With the clock pinned to `firstDate`, `firstDate < 60000 + firstDate` is true → cache hit; after jumping to `firstDate + 60001`, `firstDate + 60001 < firstDate + 60000` is false → refetch. **OBSERVED** via the counts above; the arithmetic is **INFERRED** from the source.
- The 60-second floor is `minimumTtl = 60` ([`experiment-assignments.ts:21`](../../packages/explat-client/src/internal/experiment-assignments.ts)); the server-returned `ttl` is floored via `Math.max(minimumTtl, responseTtl)` ([`requests.ts:93`](../../packages/explat-client/src/internal/requests.ts)).

**Production TTL (INFERRED, not exercised).** The README documents a production TTL of **3600 seconds** ([`README.md:42`](../../packages/explat-client/README.md): "Respects the server returned TTL (3600 seconds in production at the time of writing)."). This was **not** exercised — the production server was never contacted; the observed `ttl: 60` comes from the mocked response floored by `minimumTtl`. **INFERRED.**

**Cache backing (INFERRED).** State is persisted to `localStorage` under the key `explat-experiment--<name>` (double dash) — prefix `explat-experiment-` ([`experiment-assignment-store.ts:9`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)) plus `-` plus the name ([`experiment-assignment-store.ts:14-15`](../../packages/explat-client/src/internal/experiment-assignment-store.ts)). Under the bare `window = {}` observation context, `window.localStorage` is undefined, so the in-memory polyfill is used ([`local-storage.ts:33-40`](../../packages/explat-client/src/internal/local-storage.ts)). **INFERRED** (from the store/storage source); the cache-hit/refetch **counts** are **OBSERVED**.

---

## Q4 — Asynchronous load vs synchronous getter

**Question:** The client exposes an asynchronous loading method (`loadExperimentAssignment`) and a synchronous getter (`dangerouslyGetExperimentAssignment`). What happens when the synchronous getter is invoked **before** the asynchronous load has finished — does it break the consuming application, or is it handled gracefully?

Both the **not-yet-loaded** and **loaded** states are observed, for **both** synchronous getters, in development mode (`isDevelopmentMode: true`). The relevant markers from the shared spec run:

```
    console.log
      Q4_DGET_NOTLOADED_THREW=false

      at Object.log (src/test/blitzy_tmp_observation.ts:157:11)

    console.log
      Q4_DGET_NOTLOADED_RESULT={"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783961825616,"ttl":60,"isFallbackExperimentAssignment":true}

      at Object.log (src/test/blitzy_tmp_observation.ts:159:11)

    console.log
      Q4_DGET_NOTLOADED_LOGERROR=[[{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]]

      at Object.log (src/test/blitzy_tmp_observation.ts:161:11)

    console.log
      Q4_DMAYBE_NOTLOADED_RESULT=null

      at Object.log (src/test/blitzy_tmp_observation.ts:169:11)

    console.log
      Q4_DGET_LOADED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825617,"ttl":60}

      at Object.log (src/test/blitzy_tmp_observation.ts:177:11)

    console.log
      Q4_DMAYBE_LOADED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825617,"ttl":60}

      at Object.log (src/test/blitzy_tmp_observation.ts:179:11)
```

And the run's tail summary (the full spec passes all 4 tests):

```
Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        4.559 s
Ran all test suites matching /src\/test\/blitzy_tmp_observation.ts/i.
Jest did not exit one second after the test run has completed.
```

**OBSERVED findings — `dangerouslyGetExperimentAssignment` (the "dangerous" synchronous getter):**

- `Q4_DGET_NOTLOADED_THREW=false` — calling it **before** any load does **not** throw and does **not** break the application.
- `Q4_DGET_NOTLOADED_RESULT={...,"variationName":null,...,"ttl":60,"isFallbackExperimentAssignment":true}` — it returns the **null-variation (default/control) fallback**.
- `Q4_DGET_NOTLOADED_LOGERROR=[[{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]]` — in development mode it logs once with source `dangerouslyGetExperimentAssignment-error`.
- `Q4_DGET_LOADED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825617,"ttl":60}` — **after** `await loadExperimentAssignment(...)`, it returns the loaded `treatment` assignment.

**OBSERVED findings — `dangerouslyGetMaybeLoadedExperimentAssignment` (the sibling getter):**

- `Q4_DMAYBE_NOTLOADED_RESULT=null` — before load it returns **`null`** (its intended "not loaded yet" signal), **not** a fallback.
- `Q4_DMAYBE_LOADED={"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783961825617,"ttl":60}` — after load it returns the loaded `treatment` assignment.

**Responsible code (by name).** `dangerouslyGetExperimentAssignment` ([full method `create-explat-client.ts:184-224`](../../packages/explat-client/src/create-explat-client.ts)) **throws internally** when nothing is stored ([`create-explat-client.ts:192-194`](../../packages/explat-client/src/create-explat-client.ts)), **catches its own throw** ([`create-explat-client.ts:214`](../../packages/explat-client/src/create-explat-client.ts)), logs **only in development mode** with source `dangerouslyGetExperimentAssignment-error` ([`create-explat-client.ts:215-221`](../../packages/explat-client/src/create-explat-client.ts)), and returns `createFallbackExperimentAssignment` ([`create-explat-client.ts:222`](../../packages/explat-client/src/create-explat-client.ts)). Its sibling `dangerouslyGetMaybeLoadedExperimentAssignment` instead returns `null` when nothing is stored ([`create-explat-client.ts:225-249`](../../packages/explat-client/src/create-explat-client.ts), specifically `:234-236`).

**Contract cross-reference.** The README checklist requires `loadExperimentAssignment` to be called (significantly) before `dangerouslyGetExperimentAssignment` ([`README.md:68-73`](../../packages/explat-client/README.md)), and notes "It now logs and won't throw" ([`README.md:65`](../../packages/explat-client/README.md)). The observed not-loaded behavior matches this contract. **OBSERVED** + **INFERRED** (contract text).

### Q4 conclusion

**OBSERVED:** calling the synchronous getter before the async load has resolved does **not** break the consuming application. `dangerouslyGetExperimentAssignment` returns a null-variation (default/control) fallback (logging only in development mode), while `dangerouslyGetMaybeLoadedExperimentAssignment` returns `null` — its intended "not loaded yet" signal (used by the `useExperiment` hook, per [`CHANGELOG.md:5`](../../packages/explat-client/CHANGELOG.md)).

---


## Response-shape reference (Q1 "what does the returned object look like")

The returned object is an `ExperimentAssignment` ([`types.ts:3-24`](../../packages/explat-client/src/types.ts)):

| Field | Type | Notes |
|-------|------|-------|
| `experimentName` | `string` | the requested experiment name |
| `variationName` | `string \| null` | `null` ⇒ default/control ([`README.md:20-24`](../../packages/explat-client/README.md)); otherwise the treatment (currently `'treatment'`) |
| `retrievedTimestamp` | `number` | when the assignment was retrieved (from `monotonicNow()`) |
| `ttl` | `number` | time-to-live in seconds; floored to `minimumTtl = 60` ([`experiment-assignments.ts:21`](../../packages/explat-client/src/internal/experiment-assignments.ts)) |
| `isFallbackExperimentAssignment?` | `boolean` | `true` marks a fallback (server not reachable) |

The dependency-injection `Config` interface ([`types.ts:28-39`](../../packages/explat-client/src/types.ts)) supplies `fetchExperimentAssignment`, `getAnonId`, `logError`, and `isDevelopmentMode`.

**Observed fallback object shape** (Q1a / Q1b / Q4-not-loaded): `{ experimentName, variationName: null, retrievedTimestamp: <number>, ttl: 60, isFallbackExperimentAssignment: true }`. **Observed successful (loaded) shape** (Q2 / Q4-loaded): `{ experimentName, variationName: "treatment", retrievedTimestamp: <number>, ttl: 60 }` (no `isFallbackExperimentAssignment` key). Both are **OBSERVED** verbatim in the sections above.

---

## Cleanup & repository-hygiene confirmation

Both temporary observation specs were deleted after their output was captured:

```bash
rm -v packages/explat-client/src/test/blitzy_tmp_observation.ts \
      packages/explat-client/src/test/blitzy_tmp_timeout.ts
```
```
removed 'src/test/blitzy_tmp_observation.ts'
removed 'src/test/blitzy_tmp_timeout.ts'
```

No tracked file was modified during the investigation:

```bash
git diff --name-only        # tracked modifications
git diff --stat HEAD        # tracked changes vs HEAD
```
```
(empty — no tracked files changed)
```

The only entry in the tracked tree is this one new documentation file:

```bash
git status --porcelain --untracked-files=all
```
```
?? blitzy/documentation/wp-calypso_be7e5cc64162.md
```

Residual runtime artifacts exist **only** in gitignored locations (verified with `git check-ignore`, which echoes a path only when it is ignored):

```bash
git check-ignore node_modules packages/explat-client/node_modules .cache/jest packages/explat-client/dist
```
```
node_modules
packages/explat-client/node_modules
.cache/jest
packages/explat-client/dist
```

**OBSERVED:** the tracked tree is clean apart from the single deliverable `blitzy/documentation/wp-calypso_be7e5cc64162.md`; all `blitzy_tmp_*` files (including their Jest transform-cache copies under `.cache/jest/`) were removed.

---

## Coverage pass

### Questions

| Item | Addressed | Label | Evidence |
|------|-----------|-------|----------|
| **Q0** — test baseline | ✅ | OBSERVED | 9 suites / 81 tests / 23 snapshots pass |
| **Q1a** — server unavailable (reject) | ✅ | OBSERVED | `Q1A_THREW=false`, null fallback, 1 fetch, `loadExperimentAssignment-initialError` |
| **Q1b** — timeout (never resolves) | ✅ | OBSERVED | `Q1B_THREW=false`, null fallback; `"Promise has timed-out after {5000\|10000}ms."` |
| **Q1b** — timeout **distribution** (≥20 runs) | ✅ | OBSERVED | 20 runs: `5000 ms ×7`, `10000 ms ×13`; raw sequence + invariants table |
| **Q2** — concurrency / dedup (N=8) | ✅ | OBSERVED | `Q2_FETCH_CALLS=1`, `Q2_ALL_EQUAL=true`; stable over 3 runs |
| **Q3** — cache hit | ✅ | OBSERVED | `Q3_FETCH_AFTER_3_QUICK=1` |
| **Q3** — post-TTL refetch | ✅ | OBSERVED | `Q3_FETCH_AFTER_TTL=2` |
| **Q4** — `dangerouslyGetExperimentAssignment` not-loaded | ✅ | OBSERVED | `THREW=false`, null fallback, dev-mode log |
| **Q4** — `dangerouslyGetExperimentAssignment` loaded | ✅ | OBSERVED | returns `treatment` |
| **Q4** — `dangerouslyGetMaybeLoadedExperimentAssignment` not-loaded | ✅ | OBSERVED | returns `null` |
| **Q4** — `dangerouslyGetMaybeLoadedExperimentAssignment` loaded | ✅ | OBSERVED | returns `treatment` |

### Named mechanisms / functions / values

| Item | Label | Where addressed |
|------|-------|-----------------|
| `createExPlatClient` (index gate) | OBSERVED (browser client built) / INFERRED (gate) | Methodology; [`index.ts:8-9`](../../packages/explat-client/src/index.ts) |
| `createSsrSafeDummyExPlatClient` | INFERRED, **non-canonical** (not exercised) | Methodology; [`create-explat-client.ts:258-283`](../../packages/explat-client/src/create-explat-client.ts) |
| `loadExperimentAssignment` | OBSERVED | Q1, Q2, Q3; [`create-explat-client.ts:113-183`](../../packages/explat-client/src/create-explat-client.ts) |
| `dangerouslyGetExperimentAssignment` | OBSERVED | Q4; [`create-explat-client.ts:184-224`](../../packages/explat-client/src/create-explat-client.ts) |
| `dangerouslyGetMaybeLoadedExperimentAssignment` | OBSERVED | Q4; [`create-explat-client.ts:225-249`](../../packages/explat-client/src/create-explat-client.ts) |
| `Timing.asyncOneAtATime` | OBSERVED (effect) / INFERRED (mechanism) | Q2; [`timing.ts:44-54`](../../packages/explat-client/src/internal/timing.ts) |
| `Timing.timeoutPromise` | OBSERVED (reject message) | Q1b; [`timing.ts:23-36`](../../packages/explat-client/src/internal/timing.ts) |
| `Timing.monotonicNow` | OBSERVED (used to drive Q3) | Methodology, Q3; [`timing.ts:10-16`](../../packages/explat-client/src/internal/timing.ts) |
| `experimentNameToWrappedExperimentAssignmentFetchAndStore` | INFERRED (per-name registry) | Q2; [`create-explat-client.ts:82-94`](../../packages/explat-client/src/create-explat-client.ts) |
| `isAlive` | OBSERVED (effect) / INFERRED (formula) | Q3; [`experiment-assignments.ts:8-14`](../../packages/explat-client/src/internal/experiment-assignments.ts) |
| `minimumTtl` (= 60) | OBSERVED (`ttl:60`) | Q1, Q3; [`experiment-assignments.ts:21`](../../packages/explat-client/src/internal/experiment-assignments.ts) |
| `createFallbackExperimentAssignment` | OBSERVED | Q1, Q4; [`experiment-assignments.ts:29-38`](../../packages/explat-client/src/internal/experiment-assignments.ts) |
| `config.fetchExperimentAssignment` (DI seam) | OBSERVED (call counts) | Q1, Q2, Q3; [`requests.ts:87-90`](../../packages/explat-client/src/internal/requests.ts) |
| `setBrowserContext` | OBSERVED (client built) | Methodology; [`test-common.ts:22-26`](../../packages/explat-client/src/internal/test-common.ts) |
| localStorage key `explat-experiment--<name>` | INFERRED | Q3; [`experiment-assignment-store.ts:9,14-15`](../../packages/explat-client/src/internal/experiment-assignment-store.ts) |

### Conditions / flags

| Item | Label | Where addressed |
|------|-------|-----------------|
| `EXPERIMENT_FETCH_TIMEOUT` (= 10000) | OBSERVED (`10000ms` in dist) | Q1b; [`create-explat-client.ts:16`](../../packages/explat-client/src/create-explat-client.ts) |
| `Math.random() > 0.5` timeout A/B | OBSERVED (5000/10000 split) | Q1b; [`create-explat-client.ts:134-138`](../../packages/explat-client/src/create-explat-client.ts) |
| `isDevelopmentMode` | OBSERVED (Q4 dev-mode log) | Q4; [`create-explat-client.ts:215-221`](../../packages/explat-client/src/create-explat-client.ts) |
| `variationName === null` (default/control) | OBSERVED | Q1, Q4; [`README.md:20-24`](../../packages/explat-client/README.md) |
| `isFallbackExperimentAssignment` | OBSERVED | Q1, Q4; [`experiment-assignments.ts:37`](../../packages/explat-client/src/internal/experiment-assignments.ts) |
| browser-context guard | INFERRED (not tripped; client built in browser context) | Methodology; [`create-explat-client.ts:72-74`](../../packages/explat-client/src/create-explat-client.ts) |

### Rule compliance

- **Investigated by running first, then wrote.** ✅ Every behavioral claim above is backed by captured runtime output with the exact command.
- **Run-to-run inconsistency honored.** ✅ The randomized fetch timeout (Q1b) was run 20 times without mocking `Math.random`; the observed `5000/10000` distribution is reported (not stabilized).
- **Canonical entry point exercised.** ✅ The real browser client was built via `setBrowserContext()`; the SSR-dummy path was **not** exercised and is labeled **non-canonical**.
- **Network not contacted.** ✅ The network was mocked at the client's own dependency-injection seam `config.fetchExperimentAssignment`; the production server was never contacted.
- **INFERRED statements labeled.** ✅ The production 3600 s TTL ([`README.md:42`](../../packages/explat-client/README.md)), the "single-flight / request-coalescing" terminology, the per-name registry mechanism, and the SSR-dummy path are the only non-observed items and are each labeled INFERRED / non-canonical.
- **Read-only scope respected.** ✅ No tracked repository file was modified; the only committed artifact is this document; all temporary specs were removed (git-clean confirmation above).

