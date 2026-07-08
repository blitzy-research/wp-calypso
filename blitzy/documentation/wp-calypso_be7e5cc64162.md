# ExPlat Client — Behavioral Investigation & Answers

**Package under investigation:** `@automattic/explat-client` **v0.1.0** [`packages/explat-client/package.json:L2-L3`]
**Repository:** wp-calypso monorepo
**HEAD commit:** `be7e5cc641622d153040491fd5625c6cb83e12eb` (branch `wp-calypso_be7e5cc64162`)
**Toolchain (repo-canonical):** Yarn **4.0.2** [`package.json:packageManager`], Node **≥ 22.9.0** [`package.json:engines.node`, `.nvmrc` = `22.9.0`]; observed `node --version` = **v22.23.1** (see §7 for the Node-version discrepancy note).

---

## How this document was produced (methodology)

This is a **read-only, evidence-driven ("run-first") investigation**. Every answer below was produced by **first building and running the real code paths and capturing the actual output**, and only then writing the prose. Each answer carries:

1. the **exact command** that produced the result,
2. the **actual, complete, unedited captured output** (raw returned objects, log lines, counts, timings), and
3. a **`path:line` citation** to the responsible source at HEAD, plus the **named responsible function** and a **cause→effect** rationale.

**The real public API was exercised via dependency injection.** All observations drive the actual public factory `createExPlatClient` [`packages/explat-client/src/index.ts:L8-L11`] with an injected `Config` (`fetchExperimentAssignment`, `getAnonId`, `logError`, `isDevelopmentMode`) [`packages/explat-client/src/types.ts:L28-L39`]. Injecting `fetchExperimentAssignment` is the client's intended extension point — this is how Calypso itself wires the client [`client/lib/explat/index.ts:L10-L15`] — so it is the canonical (non-synthetic) observation path. **The live wpcom network was never called.**

**Observation environment.** The package's Jest `testEnvironment` is `'node'` (inherited from `@automattic/calypso-jest` via the shared preset [`packages/explat-client/jest.config.js:L2`]), so there is no global `window`. Because `createExPlatClient` throws when `window` is undefined [`packages/explat-client/src/create-explat-client.ts:L72-L74`], every observation spec calls `setBrowserContext()` (sets `global.window = {}`) and `localStorage.clear()` in a `beforeEach`, mirroring the real suite's setup [`packages/explat-client/src/test/create-explat-client.ts:L47-L51`]. Timing/magnitude/distribution observations use **real timers** (no `jest.useFakeTimers()`); the deterministic suite that proves "tests passing" (§1) uses fake timers + a mocked `monotonicNow`.

**Read-only note.** No existing tracked repository file was modified, added, or deleted. The **only** retained new file is this document. The temporary observation specs used to capture the output below were deleted after capture (see §7). In particular, the **stale docstring** on the synchronous getter [`packages/explat-client/src/create-explat-client.ts:L31-L37`] is **reported** as a Q5 finding, **not fixed**.

**Scale & stability.** Magnitude/frequency/timing values were observed at scale (N=100 concurrent callers, N=60 independent timeouts, a real 61-second cache-expiry wait) and confirmed **stable across ≥2 runs**. The one genuinely run-to-run-variable behavior (the 5000 ms vs 10000 ms fetch timeout) was reproduced by running the **same unchanged input repeatedly** and reporting the observed **distribution** — the variability is part of the answer and was not engineered away.

---

## Section 1 — Precondition: "make sure the package tests are passing"

**Command**

```bash
CI=1 yarn workspace @automattic/explat-client test
```

`scripts.test` is `"yarn jest"` [`packages/explat-client/package.json:L25`]; the package's `jest.config.js` extends the shared preset `../../test/packages/jest-preset.js` [`packages/explat-client/jest.config.js:L2`], which builds on `@automattic/calypso-jest`. The `calypso:src` field (`src/index.ts`) [`packages/explat-client/package.json:L12`] makes Jest run directly against untranspiled TypeScript — **no build step is required**. `CI=1` disables watch mode.

**Actual, unedited output (tail):**

```text
PASS src/internal/test/requests.ts
PASS src/internal/test/validations.ts
PASS src/internal/test/experiment-assignment-store.ts
PASS src/internal/test/timing.ts
PASS src/test/index.ts
PASS src/test/create-ssr-safe-dummy-explat-client.ts
PASS src/internal/test/experiment-assignments.ts
PASS src/internal/test/local-storage.ts
PASS src/test/create-explat-client.ts
A worker process has failed to exit gracefully and has been force exited. This is likely caused by tests leaking due to improper teardown. Try running with --detectOpenHandles to find leaks. Active timers can also cause this, ensure that .unref() was called on them.

Test Suites: 9 passed, 9 total
Tests:       81 passed, 81 total
Snapshots:   23 passed, 23 total
Time:        2.428 s, estimated 3 s
Ran all test suites.
```

**Result: the suite is green — `9 passed` suites / `81 passed` tests / `23 passed` snapshots.** The nine suites correspond exactly to the nine colocated test files: six in `src/internal/test/` (`requests`, `validations`, `experiment-assignment-store`, `timing`, `experiment-assignments`, `local-storage`) and three in `src/test/` (`index`, `create-ssr-safe-dummy-explat-client`, `create-explat-client`). The only console noise is benign: a Browserslist "caniuse-lite is old" notice (omitted above) and Jest's "worker process has failed to exit gracefully" message, caused by active timers in the timing tests — neither affects the pass result.

---

## Section 2 — Q1: "I heard the client is designed to never throw exceptions, but I want to verify this."

**Short answer.** Confirmed for the async load path: `loadExperimentAssignment` **resolves** (never rejects/throws) for success, network-failure, invalid-name, and timeout inputs. There is exactly **one** genuine throw in the whole surface — and it is *not* from `loadExperimentAssignment`: constructing the **browser** client outside a browser context throws `"Running outside of a browser context."`.

**Command**

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-main --silent=false
```

**Actual, unedited output (Q1 lines):**

```text
Q1_SUCCESS {"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783486165601,"ttl":3600}
Q1_FAILURE_THREW false
Q1_FAILURE {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165605,"ttl":60,"isFallbackExperimentAssignment":true}
Q1_INVALID {"experimentName":"Invalid-Name!","variationName":null,"retrievedTimestamp":1783486165609,"ttl":60,"isFallbackExperimentAssignment":true}
Q1_THROW Running outside of a browser context.
```

(Run twice; identical apart from the `retrievedTimestamp` values. The timeout input also resolves without throwing — see §3, `Q2_TIMEOUT_RESULT`.)

**Responsible function & cause→effect.**

- **`loadExperimentAssignment`** wraps its entire body in a `try/catch` [`packages/explat-client/src/create-explat-client.ts:L113-L183`]. Any error thrown inside — including the explicit invalid-name throw [`L115-L117`], a rejecting/timed-out fetch, or a "could not fetch" error [`L146-L148`] — is caught at [`L151`], logged via `safeLogError` (source `loadExperimentAssignment-initialError`) [`L151-L156`], and then the method returns a **stale-or-fresh fallback** assignment instead of rethrowing [`L160-L182`]. Even the fallback branch is defended by a second `try/catch` [`L173-L182`]. This is why `Q1_FAILURE_THREW` is `false` and every input yields a resolved object.
- Invalid names are rejected by **`Validation.isName`** (regex `/^[a-z0-9_]*$/`) [`packages/explat-client/src/internal/validations.ts:L11-L13`]; `"Invalid-Name!"` fails it, so the internal throw is caught and converted to the fallback (`Q1_INVALID`), not surfaced.
- The fallback object itself comes from **`createFallbackExperimentAssignment`**, documented as a function that "must never throw" [`packages/explat-client/src/internal/experiment-assignments.ts:L25-L38`].

**The one genuine throw.** The browser factory guards its entry: `if ( typeof window === 'undefined' ) throw new Error( 'Running outside of a browser context.' )` [`packages/explat-client/src/create-explat-client.ts:L72-L74`]. `Q1_THROW` captures this exact message. Note this is a **construction-time** throw of the browser factory, not a throw from `loadExperimentAssignment`. In real SSR you would not hit it, because the **public** entry switches to an SSR-safe dummy client when `window` is undefined: `typeof window === 'undefined' ? createSsrSafeDummyExPlatClient : createBrowserExPlatClient` [`packages/explat-client/src/index.ts:L8-L9`], and the dummy's methods only log and return a fallback [`packages/explat-client/src/create-explat-client.ts:L258-L283`]. The throw is therefore specific to calling the **browser** factory directly with no `window`.

**Corroboration.** The package README states the load method is "Designed to never throw" [`packages/explat-client/README.md:L44`], matching the observed runtime behavior.

---

## Section 3 — Q2: "What actually happens when the server is unavailable or takes too long to respond? What does the response look like in that situation, and what variation would a user be assigned to?"

**Short answer.** In **both** the "server unavailable" (fetch rejects) and "server too slow" (fetch times out) cases, `loadExperimentAssignment` returns a **fallback `ExperimentAssignment`** with `variationName: null`, `ttl: 60`, and `isFallbackExperimentAssignment: true`. The error is **logged, not thrown**. Because `variationName === null`, the user is assigned the **default / control experience**.

### 3a — Server unavailable (fetch rejects)

**Command**

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-main --silent=false
```

**Actual, unedited output (Q2 reject lines):**

```text
Q2_REJECT_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165611,"ttl":60,"isFallbackExperimentAssignment":true}
Q2_REJECT_LOGS [{"message":"ECONNREFUSED wpcom","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]
```

### 3b — Server too slow (fetch never resolves → real timeout)

**Command** (real timers; run **3 times** to observe the run-to-run variability):

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-timeout --silent=false
```

**Actual, unedited output across 3 runs:**

```text
# RUN 1
Q2_TIMEOUT_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486211565,"ttl":60,"isFallbackExperimentAssignment":true}
Q2_TIMEOUT_LOG [{"message":"Promise has timed-out after 5000ms.","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]
Q2_TIMEOUT_ELAPSED_MS 5002
Q2_DIST_N 60
Q2_DIST_TALLY {"5000":28,"10000":32}

# RUN 2
Q2_TIMEOUT_LOG [{"message":"Promise has timed-out after 10000ms.","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]
Q2_TIMEOUT_ELAPSED_MS 10003
Q2_DIST_N 60
Q2_DIST_TALLY {"5000":28,"10000":32}

# RUN 3
Q2_TIMEOUT_LOG [{"message":"Promise has timed-out after 10000ms.","experimentName":"experiment_name_a","source":"loadExperimentAssignment-initialError"}]
Q2_TIMEOUT_ELAPSED_MS 10002
Q2_DIST_TALLY {"5000":39,"10000":21}
```

**The response object (both cases).** It is the fallback produced by **`createFallbackExperimentAssignment`** [`packages/explat-client/src/internal/experiment-assignments.ts:L25-L38`]:

- `variationName: null` [`L34`] — **the user gets the default/control experience.** Per the README, `variationName === null` means "return the default experience"; `variationName !== null` means the treatment experience (currently always `'treatment'`) [`packages/explat-client/README.md:L22-L23`].
- `ttl: Math.max( minimumTtl, ttl )` = `Math.max( 60, 60 )` = **60** [`L36`, with `minimumTtl = 60` at `packages/explat-client/src/internal/experiment-assignments.ts:L21`].
- `isFallbackExperimentAssignment: true` [`L37`].

**Logged, not thrown.** The rejecting/timing-out fetch surfaces as a caught error inside `loadExperimentAssignment`, logged with `source: 'loadExperimentAssignment-initialError'` [`packages/explat-client/src/create-explat-client.ts:L151-L156`] — exactly the `Q2_REJECT_LOGS` / `Q2_TIMEOUT_LOG` lines. The method then returns the fallback [`L160-L182`]; no exception escapes.

**The timeout mechanism and its run-to-run variability.** The fetch is raced against a timeout by **`Timing.timeoutPromise`** [`packages/explat-client/src/create-explat-client.ts:L142-L145`], which rejects with the message `Promise has timed-out after ${ timeoutMilliseconds }ms.` [`packages/explat-client/src/internal/timing.ts:L23-L36`, message at `L31`]. The timeout duration itself is chosen per load: it defaults to `EXPERIMENT_FETCH_TIMEOUT` = `10000` [`packages/explat-client/src/create-explat-client.ts:L16`] but is set to `5000` when `Math.random() > 0.5` [`packages/explat-client/src/create-explat-client.ts:L134-L138`]. This is a deliberate A/B experiment on the timeout value.

**Scale, stability, and reproduced variability.** I ran the **same unchanged input** repeatedly rather than constructing a variant that hides the coin-flip. The single-timeout elapsed time was **5002 ms (run 1), 10003 ms (run 2), 10002 ms (run 3)** — demonstrating that **both** the 5000 ms and 10000 ms branches genuinely occur run-to-run. Over **N = 60** independent timeouts per run, the 5000/10000 split was **{5000: 28, 10000: 32}**, **{5000: 28, 10000: 32}**, and **{5000: 39, 10000: 21}** — roughly 50/50, consistent with a `Math.random() > 0.5` gate. The message and elapsed time always agree (a "5000ms" message accompanies a ~5002 ms elapsed; a "10000ms" message accompanies a ~10002–10003 ms elapsed).

---

## Section 4 — Q3: "If multiple parts of my application request the same experiment assignment at the exact same time, how many network calls actually get made?"

**Short answer.** **Exactly one.** At a scale of **N = 100** simultaneous callers for a single experiment, exactly **one** fetch is made — in both the success and the failure case.

**Command**

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-main --silent=false
```

**Actual, unedited output (Q3 lines; identical across 2 runs):**

```text
Q3_SUCCESS N=100 fetchCount=1 allIdentical=true
Q3_FAILURE N=100 fetchCount=1 logCount=100 variation=null
```

**Responsible function & cause→effect.** The fetch-and-store step is wrapped in **`Timing.asyncOneAtATime`** by **`createWrappedExperimentAssignmentFetchAndStore`** [`packages/explat-client/src/create-explat-client.ts:L82-L90`], and a per-experiment record ensures the wrapper is created once per experiment name [`L91-L94`, `L127-L132`]. `asyncOneAtATime` keeps a single `lastPromise`: the first caller invokes the wrapped function and stores its promise; every subsequent caller that arrives before it settles receives the **same** promise; once it settles, a `.finally` resets `lastPromise` to `null` [`packages/explat-client/src/internal/timing.ts:L44-L54`]. So 100 concurrent `loadExperimentAssignment('experiment_name_a')` calls share one in-flight fetch → `fetchCount = 1`.

- **Success case:** all 100 callers resolve to the *same* stored assignment (`allIdentical = true`), and only one network call occurred.
- **Failure case:** still only **one** shared network call (`fetchCount = 1`). The shared fetch rejects once; each of the 100 callers then independently runs its own catch/fallback path — hence `logCount = 100` (each caller logs its own `loadExperimentAssignment-initialError`) and each receives a `variationName: null` fallback (`variation = null`). The single-network-call guarantee holds even when the request fails.

**Scale & stability.** Observed at **N = 100** concurrent callers and confirmed **stable across 2 runs** (identical output both times).

---

## Section 5 — Q4: "When I request the same experiment multiple times in quick succession, does each request go to the network? And what happens after enough time passes for the cache to expire?"

**Short answer.** **No — repeated same-experiment requests within the TTL are cache hits with zero extra network calls.** Once the cached assignment's TTL expires, the very next request triggers **exactly one** refetch.

**Command** (real timers; `AFTER TTL` performs a real 61-second wait):

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-cache --silent=false
```

**Actual, unedited output (identical across 2 runs apart from the timestamp):**

```text
Q4_WITHIN_TTL afterFirstLoad=1 after_100_more_reloads=1
Q4_STORED {"experimentName":"experiment_name_a","variationName":"treatment","retrievedTimestamp":1783486288739,"ttl":60}
Q4_AFTER_TTL waitMs=61000 countAfterFirst=1 countStillWithinTTL=1 countAfterExpiry=2
```

**Within the TTL → cache hit, zero extra network calls.** After the first load performs one fetch (`afterFirstLoad = 1`), **100 further loads** of the same experiment (50 sequential + 50 concurrent) make **no additional network calls** (`after_100_more_reloads = 1`). The gate is **`ExperimentAssignments.isAlive`**: `monotonicNow() < experimentAssignment.ttl * MILLISECONDS_PER_SECOND + experimentAssignment.retrievedTimestamp` [`packages/explat-client/src/internal/experiment-assignments.ts:L8-L14`]. In `loadExperimentAssignment`, a stored-and-alive assignment is returned immediately before any fetch is considered [`packages/explat-client/src/create-explat-client.ts:L119-L125`].

**After the TTL elapses → exactly one refetch.** With an injected server `ttl` of `1` second and a real **61-second** wait, the counters go `countAfterFirst = 1` → `countStillWithinTTL = 1` (a second load moments after the first is still a cache hit) → `countAfterExpiry = 2` after the wait. That is **one** refetch once the entry is no longer alive.

**The TTL floor (why `Q4_STORED` shows `ttl: 60`, not `1`).** Even though the injected server returned `ttl: 1`, the stored assignment's `ttl` is **60**. The client floors the server TTL: `const ttl = Math.max( ExperimentAssignments.minimumTtl, responseTtl )` [`packages/explat-client/src/internal/requests.ts:L93`], with `minimumTtl = 60` [`packages/explat-client/src/internal/experiment-assignments.ts:L21`]. `minimumTtl` exists to cap the request rate per experiment (a floor on time-between-requests). This is why my expiry test had to wait **> 60 s** (61 s) rather than ~1 s — the server's `ttl: 1` is raised to the 60 s floor. In production the server TTL is 3600 s [`packages/explat-client/README.md:L42`].

**Where the cache lives.** Assignments are stored in LocalStorage under the key `explat-experiment--{name}` — the prefix is `explat-experiment-` [`packages/explat-client/src/internal/experiment-assignment-store.ts:L9`] and the key function appends another `-` plus the name [`L14-L15`], yielding the double dash. In the Node test environment (no real `window.localStorage`), the client transparently uses an in-memory polyfill [`packages/explat-client/src/internal/local-storage.ts:L33-L38`], which is why `localStorage.clear()` in each spec's `beforeEach` isolates the observations. The store's `storeExperimentAssignment` throws in exactly one situation — a race guard that refuses to overwrite a newer assignment with an older-timestamped one [`packages/explat-client/src/internal/experiment-assignment-store.ts:L27-L34`]; it is not hit on the normal cache paths above.

**Scale & stability.** Within-TTL observed at **100 extra reloads**; expiry observed with a real **61 s** wait; both **stable across 2 runs**.

---

## Section 6 — Q5: "The client has both an async loading method and a synchronous getter. What happens if I call the synchronous getter before the async load has finished? I need to know if this will break my application or if the client handles it gracefully."

**Short answer.** **It will not break your application — the client degrades gracefully.** Calling the synchronous getter `dangerouslyGetExperimentAssignment` **before** (or **during**) an in-flight `loadExperimentAssignment` returns a non-throwing **null-variation fallback** (`variationName: null`, `isFallbackExperimentAssignment: true`). In **development mode** it additionally logs a warning; in **production mode** it is **silent**. Despite its name and a stale docstring, it does **not** throw.

**Command**

```bash
cd packages/explat-client && CI=1 yarn jest zz-obs-main --silent=false
```

**Actual, unedited output (Q5 lines; identical across 2 runs apart from timestamps):**

```text
Q5_DEV_BEFORE_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165662,"ttl":60,"isFallbackExperimentAssignment":true}
Q5_DEV_BEFORE_LOGS [{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]
Q5_PROD_BEFORE_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165663,"ttl":60,"isFallbackExperimentAssignment":true}
Q5_PROD_BEFORE_LOGS []
Q5_DEV_DURING_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165665,"ttl":60,"isFallbackExperimentAssignment":true}
Q5_DEV_DURING_LOGS [{"message":"Trying to dangerously get an ExperimentAssignment that hasn't loaded.","experimentName":"experiment_name_a","source":"dangerouslyGetExperimentAssignment-error"}]
Q5_PROD_DURING_RESULT {"experimentName":"experiment_name_a","variationName":null,"retrievedTimestamp":1783486165866,"ttl":60,"isFallbackExperimentAssignment":true}
Q5_PROD_DURING_LOGS []
Q5_MAYBE_BEFORE_RESULT null
Q5_MAYBE_BEFORE_LOGS []
```

**Responsible function & cause→effect.** `dangerouslyGetExperimentAssignment` [`packages/explat-client/src/create-explat-client.ts:L184-L224`] tries to retrieve a stored assignment; when none is present it throws internally `"Trying to dangerously get an ExperimentAssignment that hasn't loaded."` [`L191-L195`], but that throw is caught inside the method [`L214`]. The catch block:

1. logs the error **only** when `config.isDevelopmentMode` is true, with `source: 'dangerouslyGetExperimentAssignment-error'` [`L215-L221`]; and
2. returns `createFallbackExperimentAssignment( experimentName )` [`L222`].

That is exactly what the captures show:

- **Before load, development mode** (`Q5_DEV_BEFORE_*`): null-variation fallback **plus** the dev warning log.
- **Before load, production mode** (`Q5_PROD_BEFORE_*`): the **same** null-variation fallback, but **no log** (`[]`).
- **During an in-flight load** (`Q5_DEV_DURING_*`, `Q5_PROD_DURING_*`): identical behavior to "before load," because while the fetch is in flight the assignment has not yet been stored, so the getter takes the same not-loaded path (dev logs, prod silent).

**Contrast — `dangerouslyGetMaybeLoadedExperimentAssignment`.** The sibling getter returns a literal `null` (not a fallback object) for a valid-but-unloaded name, and does not log [`packages/explat-client/src/create-explat-client.ts:L225-L248`, returning `null` at `L234-L235`]. This is confirmed by `Q5_MAYBE_BEFORE_RESULT null` with empty logs. This is the variant intended for the React hook, which needs to distinguish "not loaded yet" (`null`) from a real assignment.

**Finding — stale docstring (reported, not fixed).** The in-code docstring for `dangerouslyGetExperimentAssignment` still claims it throws:

> "Get an already loaded Experiment Assignment, will throw if there is an error, e.g. if it hasn't been loaded."

located at [`packages/explat-client/src/create-explat-client.ts:L31-L37`] (the "will throw" claim is on `L32`). The runtime evidence above shows it **never throws** — it logs (in dev) and returns a fallback. Per the read-only constraint of this investigation, this discrepancy is **reported here and left unmodified**. It is corroborated by the corrected README — "It now logs and won't throw." [`packages/explat-client/README.md:L65`] — and the CHANGELOG entry "Change dangerouslyGetExperimentAssignment to log rather than throw" [`packages/explat-client/CHANGELOG.md:L21`]. So the *documentation contract* and *runtime behavior* agree that it will not throw; only the in-code docstring is stale.

**React consumption context.** In practice the synchronous getter is consumed through the React binding. `useExperiment` starts the async load in an effect — `exPlatClient.loadExperimentAssignment( experimentName ).then( () => { ... forceUpdate(); } )` [`packages/explat-client-react-helpers/src/index.tsx:L75`] — and, on render, reads the **null-returning** synchronous getter `exPlatClient.dangerouslyGetMaybeLoadedExperimentAssignment( experimentName )` [`L99-L100`], returning `[ ! maybeExperimentAssignment, maybeExperimentAssignment ]` i.e. `[ isLoading, assignment | null ]` [`L101`]. So the React layer uses the `null`-returning variant (loading state = "no assignment yet"), not the fallback-returning `dangerouslyGetExperimentAssignment`. The real Calypso wiring constructs the client via `createExPlatClient( { fetchExperimentAssignment, getAnonId, logError, isDevelopmentMode } )` [`client/lib/explat/index.ts:L10-L15`]; its live wpcom fetch [`client/lib/explat/internals/fetch-experiment-assignment.ts`] is precisely the dependency that the injected `fetchExperimentAssignment` replaced throughout this investigation (the network was never called).

---

## Section 7 — Coverage pass & notes

### Coverage checklist (every condition the questions imply)

| Condition / named item | Answered in | Observed evidence | Primary citation |
|---|---|---|---|
| Precondition: package tests passing | §1 | `9 passed / 81 passed / 23 passed` | `packages/explat-client/package.json:L25` |
| Success (happy path) | §2 | `Q1_SUCCESS … variationName:"treatment", ttl:3600` | `create-explat-client.ts:L119-L150` |
| Network failure (fetch rejects) | §2, §3a | `Q1_FAILURE_THREW false`; `Q2_REJECT_RESULT … null` | `create-explat-client.ts:L151-L182` |
| Timeout (fetch too slow) | §3b | `Q2_TIMEOUT_RESULT … null`; `Promise has timed-out after 5000/10000ms.` | `timing.ts:L23-L36`; `create-explat-client.ts:L134-L138` |
| Invalid experiment name | §2 | `Q1_INVALID … null` (no throw) | `validations.ts:L11-L13` |
| Response object shape on failure | §3 | `{variationName:null, ttl:60, isFallbackExperimentAssignment:true}` | `experiment-assignments.ts:L25-L38` |
| Variation assigned on failure | §3 | `variationName: null` ⇒ default/control experience | `README.md:L22-L23`; `experiment-assignments.ts:L34` |
| Single vs concurrent callers | §4 | `Q3_SUCCESS N=100 fetchCount=1`; `Q3_FAILURE N=100 fetchCount=1` | `timing.ts:L44-L54`; `create-explat-client.ts:L82-L90` |
| Cache hit within TTL | §5 | `Q4_WITHIN_TTL afterFirstLoad=1 after_100_more_reloads=1` | `experiment-assignments.ts:L8-L14` |
| Cache expiry (after TTL) | §5 | `Q4_AFTER_TTL … countAfterExpiry=2` (61 s wait) | `create-explat-client.ts:L119-L145` |
| TTL floor to 60 | §5 | `Q4_STORED … "ttl":60` (server returned 1) | `requests.ts:L93`; `experiment-assignments.ts:L21` |
| Sync-get before load | §6 | `Q5_DEV_BEFORE_*`, `Q5_PROD_BEFORE_*` | `create-explat-client.ts:L214-L223` |
| Sync-get during in-flight load | §6 | `Q5_DEV_DURING_*`, `Q5_PROD_DURING_*` | `create-explat-client.ts:L184-L224` |
| Development vs production logging | §6 | dev logs `dangerouslyGetExperimentAssignment-error`; prod `[]` | `create-explat-client.ts:L215-L221` |
| `dangerouslyGetMaybeLoadedExperimentAssignment` null contrast | §6 | `Q5_MAYBE_BEFORE_RESULT null` | `create-explat-client.ts:L234-L235` |
| Out-of-browser construction throw (the one genuine throw) | §2 | `Q1_THROW Running outside of a browser context.` | `create-explat-client.ts:L72-L74` |
| Stale docstring reported (not fixed) | §6 | docstring "will throw" vs runtime never-throws | `create-explat-client.ts:L31-L37` |
| Never-throws contract corroborated | §2, §6 | README "Designed to never throw"; "now logs and won't throw" | `README.md:L44`, `L65` |

### Node-version discrepancy (observed)

The provisioned container's observed runtime is:

```bash
$ node --version
v22.23.1
```

The user-provided environment setup instruction references installing **Node 20.x**, which **fails** the repository's `engines.node = "^v22.9.0"` constraint [`package.json:engines.node`] and the pinned `.nvmrc = 22.9.0`. The repo canonically requires Node ≥ 22.9.0; the observed **v22.23.1** satisfies `^v22.9.0`, so all commands in this document were run on the repo-canonical Node line. (If Node 20.x were used, Yarn would reject the workspace on the `engines` check.)

### Scope of the findings

All behavior documented here pertains to **`@automattic/explat-client` v0.1.0** [`packages/explat-client/package.json:L3`] as it exists at HEAD `be7e5cc641622d153040491fd5625c6cb83e12eb`. The observations drove the real public `createExPlatClient` API via dependency injection; no product source file was modified, and the temporary observation specs (`zz-obs-main.ts`, `zz-obs-timeout.ts`, `zz-obs-cache.ts`) created under `packages/explat-client/src/test/` were deleted after capture, leaving this document as the only new file in the repository.
