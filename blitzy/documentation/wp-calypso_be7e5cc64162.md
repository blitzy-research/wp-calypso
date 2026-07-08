# `createSelector` & `treeSelect` — Caching/Memoization Behavior (runtime-verified)

> A precise, **runtime-verified** answer to how the monorepo's two state-management selector
> utilities cache/memoize results, and a definitive diagnosis of why a component that
> "retrieves filtered data from a central store" can observe **stale results after the
> underlying data has changed**.
>
> Every factual claim below is grounded in a `path:line` citation into the (unmodified) source
> and/or in the **complete, unedited** output of temporary observation probes that were built
> on the real entry points, run **twice** for stability, and then deleted. This was a
> strictly **read-only** investigation: no tracked source, test, README, config, or manifest
> file was modified.

---

## TL;DR

- The repository ships **two** memoized-selector utilities, and the six questions below are
  answered for **both**:
  - **`@automattic/state-utils` `createSelector`** — the dominant central-store selector
    factory (94 importers), at `packages/state-utils/src/create-selector/index.ts` (113 lines).
    It delegates per-argument caching to **`lodash` `memoize`** and dependant-change detection
    to **`@wordpress/is-shallow-equal`**.
  - **`@automattic/tree-select` `treeSelect`** — a lower-level cached selector (10 importers),
    at `packages/tree-select/src/index.ts` (131 lines). It implements a hand-rolled
    `WeakMap`/`Map` dependency tree and has **no** external memoization library.
- **The single most important cause of "stale results after the data changed" is
  `createSelector`.** It returns a **stale** value when the dependants it shallow-compares stay
  **reference-equal** despite **mutated nested contents** (i.e. in-place mutation of state).
  `isShallowEqual` sees no change, so the whole `lodash` cache is *not* cleared and the previously
  memoized value is returned unchanged — the underlying selector is never re-run
  (`packages/state-utils/src/create-selector/index.ts:L103-L104`). This was reproduced at
  runtime: after an in-place add of a second matching post, `r1 === r2` is `true` and the
  underlying selector was called only **once**; switching to an immutable replacement fixes it.
- **How each utility detects change differs.** `createSelector` compares the *current* vs
  *previous* dependants with `isShallowEqual` and, on any mismatch, **flushes the entire
  `lodash` cache** (`packages/state-utils/src/create-selector/index.ts:L103-L104`). `treeSelect` instead keys a nested `WeakMap`/`Map`
  tree by the **reference identity** of each dependent and only ever evicts stale branches via
  **garbage collection** of the `WeakMap` (`packages/tree-select/src/index.ts:L84,L96-L99`).
- **Concrete call counts (scale N = 1000, stable across two runs):** for **both** utilities the
  underlying selector is invoked **1** time for 1000 identical (cache-hit) calls, **1000** times
  for 1000 distinct-key (cache-miss) calls, and **2** times when one dependant/dependent change
  occurs between two otherwise-identical calls.
- **Programmatic clearing exists for both:** `treeSelect` exposes `clearCache()`
  (`packages/tree-select/src/index.ts:L96-L99`); `createSelector` exposes the underlying `lodash` cache handle via
  `selector.memoizedSelector.cache.clear()` (`packages/state-utils/src/create-selector/index.ts:L110-L112`).

---

## 1. Environment & how to reproduce

### Toolchain (observed)

| Tool | Version | Note |
|------|---------|------|
| Node.js | `v22.23.1` | Satisfies the repo engines `^v22.9.0` (`package.json`); `.nvmrc` pins `22.9.0`. |
| Yarn | `4.0.2` | Pinned via `"packageManager": "yarn@4.0.2"`; enabled through Corepack. |
| Jest | `29.7.0` | Via `@automattic/calypso-jest`; runs tests from TypeScript **source** (`calypso:src`) through `babel-jest`. |

### Setup commands

```bash
# Node satisfies engines "^v22.9.0" (.nvmrc 22.9.0); enable the pinned Yarn:
corepack enable
corepack prepare yarn@4.0.2 --activate
yarn install
```

`yarn install` completes and resolves the workspace (the packages under investigation are
independent Yarn workspaces). No dependency was added, upgraded, or removed for this
investigation.

### Baseline — run the two existing suites

The two utilities already ship Jest suites that instrument the underlying selector with
`jest.fn()` spies and assert exact invocation counts. Running them establishes that the exact
code paths execute and can be observed.

**Exact command:**

```bash
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/index.js packages/tree-select/test/index.js
```

**Complete, unedited output:**

```text
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
PASS packages/state-utils/src/create-selector/test/index.js
PASS packages/tree-select/test/index.js

Test Suites: 2 passed, 2 total
Tests:       30 passed, 30 total
Snapshots:   0 total
Time:        0.873 s, estimated 1 s
Ran all test suites matching /packages\/state-utils\/src\/create-selector\/test\/index.js|packages\/tree-select\/test\/index.js/i in 2 projects.
```

**Two things to note so readers aren't confused:**

1. The three leading blocks — the two `jest-haste-map: duplicate manual mock found:
   wpcom-proxy-request` warnings and the `Browserslist … 17 months old` notice — are
   **pre-existing global-harness noise** emitted by the shared Jest preset (they originate from
   built `dist/` mocks elsewhere in the monorepo), **not** from these two suites. They are
   harmless and were **not** "fixed" (doing so would modify tracked files / the lockfile).
2. The trailing `… in 2 projects` is expected: the root harness
   `test/packages/jest.config.js:L4` globs **all** packages via
   `projects: [ '<rootDir>/packages/*/jest.config.js' ]`, so passing the two specific test-file
   paths as positional arguments simply scopes the run to those two files while Jest still
   reports the two matching projects. (`Time` naturally varies run-to-run; it is the only
   value that changes between runs.)

### Development-mode guards are active under Jest

Jest sets `process.env.NODE_ENV = "test"` (confirmed in both probe outputs below), which is
**not** `"production"`. Both utilities gate their development-only safety checks on
`process.env.NODE_ENV !== 'production'`, so under Jest those guards are **active**:

- `createSelector`'s complex-argument warning (`packages/state-utils/src/create-selector/index.ts:L41-L51`), and
- `treeSelect`'s object-argument `throw` and its function-argument validation
  (`packages/tree-select/src/index.ts:L57-L63,L75-L79`).

---

## 2. The worked example — "retrieves filtered data from a central store"

The user's phrase maps directly onto the `getSitePosts`-style filtered selector that is present
as a fixture in **both** test suites: a selector that filters posts by site id. This is the
exact shape used throughout the probes below.

**`createSelector` fixture** (`packages/state-utils/src/create-selector/test/index.js:L10,L13`):

```js
const selector = jest.fn( ( state, siteId ) => filter( state.posts, { site_ID: siteId } ) );
// …
getSitePosts = createSelector( selector, ( state ) => state.posts );
```

**`treeSelect` fixture** (`packages/tree-select/test/index.js:L14-L18`):

```js
selector = jest.fn( ( [ posts ], siteId ) =>
    Object.values( posts ).filter( ( p ) => p.siteId === siteId )
);
getDependents = jest.fn( ( state ) => [ state.posts ] );
getSitePosts = treeSelect( getDependents, selector );
```

The "central store" is the modularized Redux state tree described in
`docs/modularized-state.md` — state is divided into top-level portions that are registered and
persisted independently. Both selectors read a slice of that store (here `state.posts`) and
cache the derived result.

### How the evidence below was produced

Two temporary Jest specs (the "probes") were created next to the existing suites so the shared
preset's `testMatch` discovered them, importing the **real** entry points via relative paths
(`import createSelector from '../'` and `import treeSelect from '../src'`). Each probe mirrors
the suites' `jest.fn()` spy pattern and accumulates its observations into a single `console.log`
for clean capture. They were run with:

```bash
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/probe_cs_TEMP.js packages/tree-select/test/probe_ts_TEMP.js
```

The command was executed **twice**; the output was **identical across both runs except the
single Jest `Time:` line** (wall-clock timing only — the suite print order was stable,
`createSelector` then `treeSelect`). The full scripts and the **complete, unedited** output of
**both** runs (including all Jest wrapper output and suite totals) are reproduced in the
[Appendix](#appendix). Each answer below quotes the relevant slice. The probes were **deleted**
after capture — see [§5 Cleanup](#5-cleanup--repository-left-unchanged).

---

## Q1 — The precise comparison mechanism that determines cache hits and misses

### Direct answer

- **`createSelector`** decides hit vs. miss in **two stages**. First it recomputes the
  *dependants* and compares them against the previous dependants with **`isShallowEqual`**
  (member-wise strict `===`). If they are **not** shallow-equal, it **clears the entire
  `lodash` cache**; then it delegates to the `lodash`-memoized function whose cache **key is
  `args.join()`** by default. So a value is reused only when (a) the dependants are shallow-equal
  to last time **and** (b) an entry already exists for the joined-arguments key.
- **`treeSelect`** decides hit vs. miss by **walking a nested `WeakMap`/`Map` tree keyed by the
  reference identity of each dependent**, then querying the final leaf `Map` by the arguments
  key (`getCacheKey( ...args )`, default `args.join()`). A value is reused only when the exact
  same dependent object references are seen **and** the leaf already holds that key.

### Mechanism (named functions and exact lines)

**`createSelector`** — the returned wrapper function
(`packages/state-utils/src/create-selector/index.ts:L96-L112`):

```ts
let currentDependants = getDependantsFn( state, ...( args as TDepProps ) );   // L98
if ( ! Array.isArray( currentDependants ) ) {                                  // L99
    currentDependants = [ currentDependants ];                                 // L100
}

if ( lastDependants && ! isShallowEqual( currentDependants, lastDependants ) ) {  // L103
    memoizedSelector.cache.clear?.();                                              // L104  ← whole-cache flush
}

lastDependants = currentDependants;                                            // L107
return memoizedSelector( state, ...args );                                     // L109
```

- The memoized function is built once with `const memoizedSelector = memoize( selector,
  getCacheKey );` (`packages/state-utils/src/create-selector/index.ts:L90`). Per `lodash`'s contract, the second argument is the
  **resolver** that computes the cache key; the cache is exposed as the `.cache` property and
  implements the `Map` interface.
- The default cache key is `args.join()` (`packages/state-utils/src/create-selector/index.ts:L36-L52`, specifically the `return
  args.join();` at `packages/state-utils/src/create-selector/index.ts:L38`/`packages/state-utils/src/create-selector/index.ts:L50`), i.e. the state object is dropped and only the extra arguments
  form the key.
- **`isShallowEqual`** compares each member by strict `===`; objects and arrays therefore
  compare **by reference**, while primitives can be strictly equal across instances. That single
  fact drives both the hit/miss decision here and the stale-results diagnosis in
  [Q3](#q3--per-argument-entries-vs-invalidation-and-the-stale-results-diagnosis).

**`treeSelect`** — the cached selector body (`packages/tree-select/src/index.ts:L69-L94`):

```ts
const dependents = getDependents( state, ...args );                            // L73
// …dev-mode object-argument guard at L75-L79…
const leafCache: Map< string, Result > = dependents.reduce( insertDependentKey, cache );  // L84
const key = getCacheKey( ...args );                                            // L86
if ( leafCache.has( key ) ) {                                                  // L87
    return leafCache.get( key ) as Result;                                     // L88
}
const value = selector( dependents, ...( args as SArgs ) );                    // L91
leafCache.set( key, value );                                                   // L92
return value;                                                                  // L93
```

- `insertDependentKey` (`packages/tree-select/src/index.ts:L116-L131`) descends/creates one map level per dependent,
  keyed by the dependent's **object reference** (`map.get( weakMapKey )`). Intermediate levels
  are `WeakMap`s; the **last** level is a regular `Map` because its key is the string
  `args.join()` (`packages/tree-select/src/index.ts:L128`). The default key generator is `defaultGetCacheKey = ( ...args
  ) => args.join()` (`packages/tree-select/src/index.ts:L11-L12`).

### Observed evidence (exact command + relevant output slice)

```bash
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/probe_cs_TEMP.js packages/tree-select/test/probe_ts_TEMP.js
```

**`createSelector`:**

```text
----- Q1: hit vs miss -----
same state ref + same arg, 2 calls => selector calls = 1 (expect 1: cache HIT)
same state ref, NEW arg 38303081 => selector calls = 2 (expect 2: new args.join key)
NEW state.posts reference, arg 2916284 => selector calls = 3 (expect 3: isShallowEqual false => cache.clear then recompute)
```

**`treeSelect`:**

```text
----- Q1: hit vs miss -----
same posts ref + same arg, 2 calls => selector calls = 1 (expect 1: WeakMap leaf HIT)
same posts ref, NEW arg site2 => selector calls = 2 (expect 2: leaf Map key from args.join)
NEW posts reference, arg site1 => selector calls = 3 (expect 3: dependent identity changed => fresh WeakMap branch)
```

### Cause → effect

- **Call 1 → 2 (same dependants, new argument):** the joined-argument key changes
  (`2916284` → `38303081`, or `site1` → `site2`), so it is a **miss** and the selector runs — the
  count rises to `2`. For `createSelector` this is a *new `lodash` cache entry*; for `treeSelect`
  it is a *new key in the same leaf `Map`* (the dependent branch is unchanged).
- **Call 2 → 3 (dependant/dependent reference changed, argument reused):** a **new**
  `state.posts` object is passed. For `createSelector`, `isShallowEqual( [newPosts], [oldPosts]
  )` is `false` → `memoizedSelector.cache.clear?.()` wipes the cache → the reused argument now
  misses → count rises to `3`. For `treeSelect`, the new `posts` reference indexes a **fresh
  `WeakMap` branch** that has never held the key → miss → count rises to `3`. Both reach `3`
  through different structures but the same underlying principle: **change is detected by
  reference identity**, not by value.

---

## Q2 — How many times the underlying selector is actually called when dependent state changes between calls (concrete numbers)

### Direct answer

Measured at **scale N = 1000** and **confirmed stable across two identical runs**, the underlying
selector is invoked:

| Scenario | `createSelector` | `treeSelect` |
|----------|:----------------:|:------------:|
| 1000 **identical** calls (cache hit) | **1** | **1** |
| 1000 calls with **distinct argument keys** (cache miss each) | **1000** | **1000** |
| 2 calls, argument fixed, **one dependant/dependent reference change** between them | **2** | **2** |

In other words: the underlying selector runs **exactly once per distinct (dependants-generation,
argument-key) combination**. When dependent state changes reference between two calls with the
same argument, the selector runs **twice** — once before and once after the change.

### Mechanism

- The `1`/`1000` split is the memoization core: identical inputs resolve to the same cache key
  and short-circuit before the selector runs (`createSelector` via `lodash` `memoize`,
  `packages/state-utils/src/create-selector/index.ts:L90,L109`; `treeSelect` via `leafCache.has( key )`, `packages/tree-select/src/index.ts:L87-L88`).
- The `2` for a single dependent change is exactly the cache-bust path from
  [Q1](#q1--the-precise-comparison-mechanism-that-determines-cache-hits-and-misses):
  `isShallowEqual` false → `cache.clear` (`packages/state-utils/src/create-selector/index.ts:L103-L104`), or a fresh
  `WeakMap` branch (`packages/tree-select/src/index.ts:L84`).

These numbers align with the packages' own canonical assertions:

- `createSelector`: `expect( selector ).toHaveBeenCalledTimes( 1 )` for a repeated identical call
  (`packages/state-utils/src/create-selector/test/index.js:L62`) and
  `toHaveBeenCalledTimes( 2 )` when the argument differs / watched state changes
  (`packages/state-utils/src/create-selector/test/index.js:L111`, and again at `packages/state-utils/src/create-selector/test/index.js:L153`).
- `treeSelect`: `expect( selector.mock.calls ).toHaveLength( 1 )` for a repeated identical call
  (`packages/tree-select/test/index.js:L45`) and `toHaveLength( 2 )` for non-cached calls
  (`packages/tree-select/test/index.js:L139`).

### Observed evidence (exact command + relevant output slice)

Same probe command as Q1. **The `N = 1000` scale and the two-run stability directly satisfy the
"state the run scale and confirm stability" requirement for a frequency question.**

**`createSelector`:**

```text
----- Q2: call counts (SCALE stated) -----
HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
MISS path: N = 1000 distinct arg keys (same state) => selector calls = 1000 (expect 1000 )
RECOMPUTE-after-dependant-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
```

**`treeSelect`:**

```text
----- Q2: call counts (SCALE stated) -----
HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
MISS path: N = 1000 distinct arg keys (same dependent) => selector calls = 1000 (expect 1000 )
RECOMPUTE-after-dependent-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
```

### Cause → effect

- **HIT = 1 out of 1000:** every one of the 1000 calls presents the same dependants and the same
  argument key, so calls 2…1000 short-circuit on a cache hit and never reach the selector.
- **MISS = 1000 out of 1000:** each call uses a *distinct* argument (`i` from `0…999`, or `'k'+i`),
  so every call produces a new key and therefore a fresh selector invocation. For
  `createSelector` these 1000 entries all coexist in one `lodash` cache (the dependants never
  changed, so no flush happened); for `treeSelect` they coexist as 1000 keys in a single leaf
  `Map`.
- **RECOMPUTE = 2:** with the argument held constant, changing the dependent's reference once
  forces exactly one additional computation — the "dependent state changes between calls" case
  the question asks about.

---


## Q3 — Per-argument entries vs. invalidation (and the stale-results diagnosis)

### Direct answer

- **`createSelector`: both, in tension.** It **does** maintain **separate entries per unique
  argument key** (distinct `args.join()` keys coexist in the `lodash` cache), **but** *any* change
  to the shallow-compared dependants **flushes the entire cache at once** — not just the entry for
  the current argument. So calling with a *different argument* does **not** invalidate previous
  entries, whereas a *dependant change* invalidates **all** of them together.
- **`treeSelect`: per-dependent coexistence.** It keeps entries for **unique dependents
  simultaneously**; distinct dependent branches live side by side in the `WeakMap` tree and are
  **not** flushed when you query a different dependent. Old branches are reclaimed **only** by
  garbage collection of the `WeakMap` when their dependent objects become unreachable — there is
  no synchronous whole-cache flush.
- **The stale-results symptom comes from `createSelector`.** If state is **mutated in place** so
  that the dependant reference stays the same, `isShallowEqual` reports "no change", the cache is
  **not** cleared, and the previously cached value is returned — the change is invisible and the
  selector is **not** re-run.

### Mechanism

- `createSelector` per-argument entries: each distinct `args.join()` key is a separate `lodash`
  `memoize` entry (`packages/state-utils/src/create-selector/index.ts:L90`, key at `packages/state-utils/src/create-selector/index.ts:L36-L52`).
- `createSelector` whole-cache flush: `memoizedSelector.cache.clear?.()` on any shallow-inequality
  of dependants (`packages/state-utils/src/create-selector/index.ts:L103-L104`). This clears **every** entry, not the one
  for the current argument.
- `treeSelect` simultaneous dependents: the nested `WeakMap`/`Map` structure
  (`packages/tree-select/src/index.ts:L84`, built by `insertDependentKey` at `packages/tree-select/src/index.ts:L116-L131`) stores a
  distinct leaf per dependent-reference path; the package's own test names this behavior
  ("should maintain the cache for unique dependents simultaneously",
  `packages/tree-select/test/index.js:L161-L186`, asserting `toHaveLength( 2 )` at `packages/tree-select/test/index.js:L185`). The
  comment at `packages/tree-select/src/index.ts:L81-L83` states the intent: garbage-collect values based on
  outdated dependents.

### Observed evidence (exact command + relevant output slice)

**`createSelector` — per-argument entries coexist, then a single dependant change flushes all:**

```text
----- Q3: per-arg entries, whole-cache flush, STALE -----
after calling args 2916284 & 38303081 (same state):
  cache.has('2916284') = true
  cache.has('38303081') = true (=> SEPARATE per-arg entries coexist)
after ONE dependant change (new posts ref), a single arg call:
  cache.has('2916284') = true (recomputed key present)
  cache.has('38303081') = false (=> the OTHER arg entry was FLUSHED by whole-cache clear)
```

**`createSelector` — the stale value, reproduced on the same unchanged input reference:**

```text
STALE reproduction (in-place mutation, dependants stay reference-equal):
  r1 (before mutation) = [{"ID":1,"site_ID":100,"title":"first"}]
  r2 (after in-place add of a 2nd matching post) = [{"ID":1,"site_ID":100,"title":"first"}]
  r1 === r2 ? true   staleSelector calls = 1 (=> STALE: change invisible, selector NOT re-run)
  after IMMUTABLE replace: r3 = [{"ID":1,"site_ID":100,"title":"first"},{"ID":2,"site_ID":100,"title":"second (added after r1)"}]  staleSelector calls = 2 (=> fresh)
```

**`treeSelect` — unique dependents coexist (no whole-cache flush):**

```text
----- Q3: unique dependents coexist simultaneously -----
calls id1,id2,id1 => selector calls = 2 (expect 2: BOTH dependent branches retained simultaneously; id1 re-hit)
note: old dependent branches are evicted ONLY by GC of the WeakMap (not observable synchronously)
```

### Cause → effect (the stale-results diagnosis, in full)

1. The probe first calls `getSitePosts( state, 2916284 )` and `getSitePosts( state, 38303081 )`
   with the **same** `state.posts` reference. Both argument keys are cached side by side:
   `cache.has('2916284')` and `cache.has('38303081')` are **both `true`**. This proves entries are
   kept **per argument**, and that calling with a different argument does **not** evict the
   previous one.
2. The probe then calls once more with a **new `state.posts` object** and argument `2916284`.
   Because the dependant reference changed, `isShallowEqual` is `false`, so
   `memoizedSelector.cache.clear?.()` runs (`packages/state-utils/src/create-selector/index.ts:L104`). The just-recomputed
   `2916284` entry is present (`true`), but `38303081` is now **`false`** — proving the change
   flushed the **whole** cache, not just one key.
3. **The stale case:** using a fresh selector, `r1 = getStale( state, 100 )` returns one post.
   The probe then **mutates `state.posts` in place** (adds a second matching post) *without*
   changing the `state.posts` reference. On the next call, `getDependants` returns the **same**
   `posts` reference, so `isShallowEqual( [posts], [posts] )` is `true`, the cache is **not**
   cleared, and the cached value is returned: `r1 === r2` is **`true`**, the added post is
   **invisible**, and `staleSelector` was called only **once**. Switching to an **immutable**
   replacement (`{ ...posts }`) changes the reference, busts the cache, and yields the correct
   `r3` containing both posts (selector now called **twice**).

**This is the precise condition behind "stale results after the underlying data has changed":**
`createSelector` (and `treeSelect` alike) detect change by **reference identity of the
dependants**. Any in-place mutation that preserves those references is undetectable, so the
memoized value survives the data change. The fix is the standard Redux discipline of **immutable
updates** — which both READMEs assume (`packages/tree-select/README.md:L40`;
`packages/state-utils/src/create-selector/README.md:L40`).

> `treeSelect` has the same reference-identity sensitivity, but its practical failure mode
> differs: because each dependent reference indexes its own branch, an in-place mutation that
> keeps the reference will likewise return the stale branch value; a new reference simply routes
> to a fresh branch rather than flushing everything.

---

## Q4 — Is there a way to programmatically clear the entire cache for a selector?

### Direct answer

**Yes, for both — via different handles.**

- **`treeSelect`** returns a selector with a **`clearCache()`** method that reassigns a fresh
  `WeakMap`, discarding the entire dependency tree in one call.
- **`createSelector`** does not add a bespoke method, but it **exposes the underlying `lodash`
  cache** on the returned function's **`memoizedSelector`** property, so
  `selector.memoizedSelector.cache.clear()` wipes every entry.

### Mechanism

- `treeSelect`: `cachedSelector.clearCache = () => { cache = new WeakMap(); };`
  (`packages/tree-select/src/index.ts:L96-L99`). Because a `WeakMap` has no `clear()` method, the
  implementation recreates it — the comment at `packages/tree-select/src/index.ts:L97` says exactly this. The `CachedSelector`
  interface declares `clearCache: () => void` (`packages/tree-select/src/index.ts:L29-L32`). The package test exercises it
  ("should bust the cache when clearCache() method is called",
  `packages/tree-select/test/index.js:L197-L217`).
- `createSelector`: the wrapper is `Object.assign( function(…){…}, { memoizedSelector } )`
  (`packages/state-utils/src/create-selector/index.ts:L96-L112`), and `memoizedSelector` is the
  `lodash`-memoized function whose `Map`-interface cache exposes `clear()`. The package's own
  suite relies on this in `beforeEach`: `getSitePosts.memoizedSelector.cache.clear();`
  (`packages/state-utils/src/create-selector/test/index.js:L17`). The README documents the
  handle: "you can manage the internal Lodash `memoize.Cache` instance on the `memoizedSelector`
  property" (`packages/state-utils/src/create-selector/README.md:L67-L69`).

### Observed evidence (before / after)

**`createSelector`:**

```text
----- Q4: programmatic clearing (memoizedSelector.cache.clear) -----
BEFORE clear: cache.has('2916284') = true  selector calls = 1
AFTER  clear: cache.has('2916284') = false
after clear, same call again => selector calls = 2 (=> recomputed)
```

**`treeSelect`:**

```text
----- Q4: clearCache() -----
BEFORE clear: memoizedResult === firstResult ? true  selector calls = 1 (expect true, 1)
AFTER  clearCache(): afterClearResult === firstResult ? false  selector calls = 2 (expect false, 2 => fresh WeakMap, recompute)
```

### Cause → effect

- **`createSelector`:** before clearing, the `2916284` entry exists (`cache.has('2916284') =
  true`) and the selector had run once. Calling `memoizedSelector.cache.clear()` empties the
  cache (`cache.has('2916284') = false`), so the identical follow-up call **misses** and the
  selector runs again — count `1 → 2`.
- **`treeSelect`:** before clearing, a repeated call returns the *same object* (`memoizedResult
  === firstResult` is `true`) with one computation. `clearCache()` swaps in a brand-new
  `WeakMap`, so the next call routes to an empty tree, recomputes, and returns a **different**
  object reference (`afterClearResult === firstResult` is `false`) — count `1 → 2`.

Both confirm the clear is **total**: the entire cache for the selector is discarded, not a single
entry.

---


## Q5 — What happens when the dependency getter returns nullish values vs. primitive values (numbers, booleans)?

### Direct answer

The two utilities behave **very differently** here:

- **`treeSelect` is strict.** A dependent that is **`null` or `undefined`** is accepted and
  **collapses onto one shared sentinel key** (`NULLISH_KEY`), so `null` and `undefined` map to the
  **same** cache branch. A dependent that is a **non-null primitive** — a **number** (including
  `0`), a **boolean** (including `false`), or a string — **throws a `TypeError`**.
- **`createSelector` is tolerant.** Its `isShallowEqual` comparison path accepts **both** nullish
  and primitive dependants without throwing; each simply participates in the shallow equality
  check. One nuance: because the comparison is strict `===`, changing a dependant from `null` to
  `undefined` counts as a change and **busts** the cache.

### Mechanism

**`treeSelect`** — `insertDependentKey` (`packages/tree-select/src/index.ts:L116-L131`):

```ts
if ( key != null && Object( key ) !== key ) {                                  // L118
    throw new TypeError( 'key must be an object, `null`, or `undefined`' );     // L119
}
const weakMapKey = key || NULLISH_KEY;                                         // L121
```

- `Object( key ) !== key` is `true` for any **primitive** (boxing produces a different object), so
  a non-null primitive dependent hits the `throw` at `packages/tree-select/src/index.ts:L119`. The exact message is
  **`key must be an object, \`null\`, or \`undefined\``**.
- For `null`/`undefined`, the guard's `key != null` short-circuits (no throw), and `key ||
  NULLISH_KEY` substitutes the shared singleton `const NULLISH_KEY = {};`
  (`packages/tree-select/src/index.ts:L107`) — so both nullish values index the **same** `WeakMap` entry.
- The package's suite matches this exactly: "should memoize a nullish value returned by
  getDependents" with dependents `[ null, undefined ]`
  (`packages/tree-select/test/index.js:L219-L229`), and "throws on a non-nullish primitive value
  returned by getDependents" iterating `[ true, 1, 'a', false, '', 0 ]`
  (`packages/tree-select/test/index.js:L231-L241`).

**`createSelector`** — there is no such guard. Dependants flow straight into
`isShallowEqual( currentDependants, lastDependants )` (`packages/state-utils/src/create-selector/index.ts:L103`), whose
member-wise strict `===` handles `null`, `undefined`, numbers, and booleans alike (a non-array
dependant is first wrapped into `[ value ]` at `packages/state-utils/src/create-selector/index.ts:L99-L101`).

### Observed evidence

**`treeSelect` — nullish share a key; every non-null primitive throws (covers `null`,
`undefined`, number `1`, boolean `true`, plus `'a'`, `0`, `false`):**

```text
----- Q5: nullish vs primitive dependents -----
dependent null THEN undefined => selector calls = 1 (expect 1: null & undefined SHARE NULLISH_KEY)
  dependent = null => OK (memoized)
  dependent = undefined => OK (memoized)
  dependent = number 1 => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = boolean true => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = string a => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = number 0 => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = boolean false => THREW "TypeError: key must be an object, `null`, or `undefined`"
```

**`createSelector` — tolerates nullish and primitive dependants alike (each: no throw, memoized
to 1 call); `null` → `undefined` busts the cache:**

```text
----- Q5: nullish vs primitive dependants (createSelector) -----
  getDependants -> null: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> undefined: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> number 5: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> boolean true: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  dependant null -> undefined (arg fixed): selector calls = 2 (expect 2: null !== undefined under ===, cache busts)
```

### Cause → effect

- **`treeSelect`, nullish:** calling once with dependent `null` and once with dependent
  `undefined` yields **one** selector call, because `key || NULLISH_KEY` maps both to the same
  `NULLISH_KEY` branch — the second call is a hit. Reported as `selector calls = 1`.
- **`treeSelect`, primitive:** `number 1`, `boolean true`, `'a'`, `number 0`, and `boolean false`
  each throw the **`TypeError`** from `packages/tree-select/src/index.ts:L119` — note that `0` and `false` throw too (the guard keys
  off `Object(key) !== key`, not truthiness). This is a hard failure, not a silent fallback.
- **`createSelector`, both:** for `null`, `undefined`, `5`, and `true` as the *dependant*, two
  identical calls produce **one** selector call and **no throw** — the value is simply compared by
  `isShallowEqual`. The last line shows the strict-equality nuance: a dependant that transitions
  `null → undefined` is treated as a change (`null !== undefined`), so the cache busts and the
  selector runs a second time (`selector calls = 2`).

---

## Q6 — Can I customize how cache keys are generated when passing complex query objects as arguments?

### Direct answer

**Yes, for both** — each accepts a custom key generator that **overrides the default
`args.join()`**, and that is precisely how you make **complex query objects** usable as arguments:

- **`treeSelect`** takes it via the **`options.getCacheKey`** field (third parameter). With it,
  two *distinct* query objects that produce the same key return the **same** memoized result.
  **Without** it, passing an object argument **throws** in development.
- **`createSelector`** takes it as its **third parameter** (`getCacheKey`). With it, entries are
  keyed by whatever string you return (e.g. `CUSTOM2916284`). **Without** it, complex object
  arguments trigger a development **warning** (they still "work" but collapse to unreliable keys).

### Mechanism

- Default key generators both reduce to `args.join()`:
  `packages/tree-select/src/index.ts:L11-L12` (`defaultGetCacheKey`) and
  `packages/state-utils/src/create-selector/index.ts:L36-L52` (`DEFAULT_GET_CACHE_KEY`, returning `args.join()`).
- **`treeSelect` custom key:** `const { getCacheKey = defaultGetCacheKey } = options;`
  (`packages/tree-select/src/index.ts:L67`), used as `const key = getCacheKey( ...args );`
  (`packages/tree-select/src/index.ts:L86`). The development-mode object-argument guard only fires **when the default key
  is in use**: `if ( getCacheKey === defaultGetCacheKey && args.some( isObject ) ) throw new
  Error( 'Do not pass objects as arguments to a treeSelector' );`
  (`packages/tree-select/src/index.ts:L75-L79`). Supplying a custom `getCacheKey` bypasses that guard and
  enables object arguments. Test: "accepts a getCacheKey option that enables object arguments"
  with `getCacheKey: ( query ) => \`key:${ query.siteId }\``
  (`packages/tree-select/test/index.js:L243-L264`, asserting `firstResult` is `secondResult` at
  `packages/tree-select/test/index.js:L263`); the throw path is asserted at `packages/tree-select/test/index.js:L95-L101`.
- **`createSelector` custom key:** the third parameter `getCacheKey = DEFAULT_GET_CACHE_KEY`
  (`packages/state-utils/src/create-selector/index.ts:L88`) is passed straight to `memoize( selector, getCacheKey )`
  (`packages/state-utils/src/create-selector/index.ts:L90`). Under the **default** key in development, the complex-argument check warns:
  `hasInvalidArg = args.some( ( arg ) => arg && ! VALID_ARG_TYPES.includes( typeof arg ) )` then
  `warn( 'Do not pass complex objects as arguments for a memoized selector' )`
  (`packages/state-utils/src/create-selector/index.ts:L42-L47`; `VALID_ARG_TYPES = [ 'number', 'boolean', 'string' ]` at
  `packages/state-utils/src/create-selector/index.ts:L13`). Test: "should accept an optional custom cache key generating function" producing
  `cache.has('CUSTOM2916284')` (`packages/state-utils/src/create-selector/test/index.js:L257-L269`,
  assertion at `packages/state-utils/src/create-selector/test/index.js:L266`); the warning-count assertion (`toHaveBeenCalledTimes( 3 )`) is at
  `packages/state-utils/src/create-selector/test/index.js:L78`.

### Observed evidence

**`treeSelect` — default key rejects objects; custom `getCacheKey` enables them (two distinct
objects → one memoized result):**

```text
----- Q6: custom cache keys / object args -----
default key + object arg => "Error: Do not pass objects as arguments to a treeSelector"
custom getCacheKey (query)=>key:query.siteId : firstResult === secondResult ? true (expect true: distinct objects, same generated key)
firstResult = [{"id":"id1","text":"post 1","siteId":"site1"},{"id":"id2","text":"post 2","siteId":"site1"}]
```

**`createSelector` — default key warns 3× for the complex args in a mixed run; custom key stores a
custom entry:**

```text
----- Q6: custom cache keys / complex args -----
default key, 9 calls w/ mixed args => warn (@wordpress/warning) calls = 3 (expect 3: {}, [], and [] in (1,[]))
warn message[0] = "Do not pass complex objects as arguments for a memoized selector"
custom key (state, siteId) => 'CUSTOM'+siteId : cache.has('CUSTOM2916284') = true (expect true)
```

### Cause → effect

- **`treeSelect`, without a custom key:** the object argument `{}` makes `args.some( isObject )`
  true while `getCacheKey === defaultGetCacheKey`, so the guard throws
  **`Error: Do not pass objects as arguments to a treeSelector`** (`packages/tree-select/src/index.ts:L77`).
  This is a deliberate stop: `args.join()` would stringify every object to the useless
  `"[object Object]"`, silently colliding all queries.
- **`treeSelect`, with `getCacheKey: (query) => \`key:${query.siteId}\``:** two **distinct**
  objects `{ siteId: 'site1', foo: 'bar' }` and `{ siteId: 'site1', foo: 'baz' }` generate the
  **same** key `key:site1`, so the second call is a **hit** and returns the *same* object
  (`firstResult === secondResult` is `true`). This is exactly the "complex query object" use case
  the question raises.
- **`createSelector`, without a custom key:** across nine mixed-argument calls, the three
  complex/object arguments — `{}`, `[]`, and the `[]` inside `(1, [])` — each trip
  `hasInvalidArg`, producing **3** warnings; the first message is verbatim **`Do not pass complex
  objects as arguments for a memoized selector`**. (`createSelector` only *warns*; unlike
  `treeSelect` it does not throw — but the resulting `args.join()` keys are unreliable for
  objects.)
- **`createSelector`, with a custom key `(state, siteId) => \`CUSTOM${siteId}\``:** the entry is
  stored under exactly that string — `cache.has('CUSTOM2916284')` is **`true`** — giving you full
  control over how complex arguments are reduced to a stable key.

---


## 3. Known discrepancy (noted, **not** fixed)

While investigating `treeSelect`, one documentation inconsistency was observed and is recorded
here per the read-only scope of this task. **It was not corrected.**

- The **implementation** signature is `treeSelect( getDependents, selector, options = {} )` —
  **`getDependents` first** (`packages/tree-select/src/index.ts:L52-L56`).
- The **README code examples** call it the other way around, `treeSelect( selector, getDependents
  )` — **selector first** (`packages/tree-select/README.md:L19` and `packages/tree-select/README.md:L49`).
  Curiously, the README's *prose* argument list is in the correct order (getDependents then
  selector, `packages/tree-select/README.md:L10-L11`); only the two runnable snippets are reversed.
- The package's own **passing test** uses the **correct** order,
  `treeSelect( getDependents, selector )` (`packages/tree-select/test/index.js:L18`).

Because the two arguments are both functions, calling with them reversed does not fail fast at
the `isFunction` guard (`packages/tree-select/src/index.ts:L57-L63`) — it simply wires dependents and the
selector backwards, which is a plausible source of caller confusion and could itself masquerade
as "unexpected caching behavior." This is documented as an observation only.

---

## 4. Summary table

| Aspect | `createSelector` (`@automattic/state-utils`) | `treeSelect` (`@automattic/tree-select`) |
|--------|----------------------------------------------|------------------------------------------|
| Q1 — hit/miss comparison | `isShallowEqual` on dependants (strict `===`, by reference) + `lodash` cache keyed by `args.join()` (`packages/state-utils/src/create-selector/index.ts:L103,L90,L36-L52`) | Nested `WeakMap`/`Map` keyed by dependent **reference identity**, leaf keyed by `getCacheKey(...args)` (`packages/tree-select/src/index.ts:L84-L88`) |
| Q2 — call counts (N=1000) | HIT **1**, MISS **1000**, one-change **2** | HIT **1**, MISS **1000**, one-change **2** |
| Q3 — per-arg vs. invalidation | Per-arg entries coexist, but **any** dependant change flushes **all** (`packages/state-utils/src/create-selector/index.ts:L104`) | Unique dependents coexist; eviction only via **GC** of the `WeakMap` (`packages/tree-select/src/index.ts:L96-L99` for explicit clear) |
| Stale-results cause | In-place mutation keeps dependant reference equal → `isShallowEqual` true → no clear → stale value | Same reference-identity sensitivity per dependent branch |
| Q4 — programmatic clear | `selector.memoizedSelector.cache.clear()` (`packages/state-utils/src/create-selector/index.ts:L110-L112`) | `selector.clearCache()` (`packages/tree-select/src/index.ts:L96-L99`) |
| Q5 — nullish dependants | Tolerated (shallow-compared); `null`→`undefined` busts cache | `null`/`undefined` share `NULLISH_KEY` (`packages/tree-select/src/index.ts:L107,L121`) |
| Q5 — primitive dependants | Tolerated (number/boolean shallow-compared, no throw) | **Throws `TypeError`** (`packages/tree-select/src/index.ts:L118-L119`), incl. `0`/`false` |
| Q6 — custom cache key | 3rd param `getCacheKey` (`packages/state-utils/src/create-selector/index.ts:L88`); default warns on complex args (`packages/state-utils/src/create-selector/index.ts:L42-L47`) | `options.getCacheKey` (`packages/tree-select/src/index.ts:L67`); default **throws** on object args (`packages/tree-select/src/index.ts:L75-L79`) |

---

## 5. Cleanup — repository left unchanged

This was a strictly read-only investigation. The two temporary probe specs
(`packages/state-utils/src/create-selector/test/probe_cs_TEMP.js` and
`packages/tree-select/test/probe_ts_TEMP.js`) were created only to capture the output in
[§A.5](#a5-complete-unedited-probe-output) and were **deleted** immediately afterward; they are
**absent** from the repository. No tracked source, test, README, config, or manifest file was
modified — the only change this investigation contributes is the addition of **this one
document**.

Final repository state (this deliverable is committed on the branch):

- **The working tree is clean** — `git status --porcelain` prints nothing (installed
  `node_modules` and the `.cache/` Jest transform cache are git-ignored and are not tracked
  changes).
- **The baseline-to-HEAD diff contains exactly one added file** — `git diff --name-status
  be7e5cc641..HEAD` reports precisely:

  ```text
  A	blitzy/documentation/wp-calypso_be7e5cc64162.md
  ```

- **The temporary probe scripts are absent** — no `probe_cs_TEMP.js` / `probe_ts_TEMP.js` (nor
  any other `*_TEMP.*` observation artifact) remains anywhere in the tracked tree.

For transparency: while the investigation was in progress, before this document was committed,
`git status --porcelain` briefly showed the file as a single untracked entry. That was a
transient pre-commit state; the final, committed condition is the clean working tree described
above.

---

## Appendix

### A.1 Package versions (declared / resolved)

| Package | Version | Source |
|---------|---------|--------|
| `@automattic/state-utils` | `1.0.0-alpha.4` | `packages/state-utils/package.json:L3` |
| `@automattic/tree-select` | `2.0.0` | `packages/tree-select/package.json:L3` |
| `lodash` | declared `^4.17.21`, resolved `4.17.21` | `packages/state-utils/package.json:L35` |
| `@wordpress/is-shallow-equal` | declared `^5.21.0`, resolved `5.21.0` | `packages/state-utils/package.json:L33` |
| `@wordpress/warning` | declared `^3.21.0`, resolved `3.21.0` | `packages/state-utils/package.json:L34` |
| `tslib` | declared `^2.3.0`, resolved `2.6.3` | `packages/tree-select/package.json:L37` (sole runtime dep) |

> `@automattic/tree-select` declares **no** external memoization library — its only runtime
> dependency is `tslib` (`packages/tree-select/package.json:L36-L38`). `createSelector` delegates
> per-argument caching to `lodash` `memoize` and dependant-change detection to
> `@wordpress/is-shallow-equal`.

### A.2 Exact commands

```bash
# Setup
corepack enable
corepack prepare yarn@4.0.2 --activate
yarn install

# Baseline (existing suites): 2 suites / 30 tests
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/index.js packages/tree-select/test/index.js

# Probes (run TWICE for stability): 2 suites / 2 tests each run
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/probe_cs_TEMP.js packages/tree-select/test/probe_ts_TEMP.js
```


### A.3 Probe script 1 — `packages/state-utils/src/create-selector/test/probe_cs_TEMP.js`

```js
/* TEMPORARY OBSERVATION PROBE for createSelector — Q1..Q6. To be deleted after capture. */
import { filter } from 'lodash';
import warn from '@wordpress/warning';
import createSelector from '../';

jest.mock( '@wordpress/warning', () => jest.fn() );

const out = [];
const log = ( ...a ) => out.push( a.join( ' ' ) );

test( 'createSelector probe Q1..Q6', () => {
	log( '===== createSelector PROBE =====' );
	log( 'process.env.NODE_ENV =', JSON.stringify( process.env.NODE_ENV ) );

	const selector = jest.fn( ( state, siteId ) => filter( state.posts, { site_ID: siteId } ) );
	const getSitePosts = createSelector( selector, ( state ) => state.posts );

	const postA = { ID: 841, site_ID: 2916284, title: 'Hello World' };
	const postB = { ID: 413, site_ID: 38303081, title: 'Ribs & Chicken' };

	// -------- Q1: comparison mechanism (hit vs miss) --------
	log( '' );
	log( '----- Q1: hit vs miss -----' );
	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	const state1 = { posts: { a: postA } };
	getSitePosts( state1, 2916284 );
	getSitePosts( state1, 2916284 );
	log( 'same state ref + same arg, 2 calls => selector calls =', selector.mock.calls.length, '(expect 1: cache HIT)' );

	getSitePosts( state1, 38303081 );
	log( 'same state ref, NEW arg 38303081 => selector calls =', selector.mock.calls.length, '(expect 2: new args.join key)' );

	const state2 = { posts: { a: postA, b: postB } };
	getSitePosts( state2, 2916284 );
	log( 'NEW state.posts reference, arg 2916284 => selector calls =', selector.mock.calls.length, '(expect 3: isShallowEqual false => cache.clear then recompute)' );

	// -------- Q2: concrete call counts at STATED scale --------
	log( '' );
	log( '----- Q2: call counts (SCALE stated) -----' );
	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	const N = 1000;
	const s = { posts: { a: postA } };
	for ( let i = 0; i < N; i++ ) getSitePosts( s, 2916284 );
	log( 'HIT path: N =', N, 'identical calls => selector calls =', selector.mock.calls.length, '(expect 1)' );

	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	for ( let i = 0; i < N; i++ ) getSitePosts( s, i );
	log( 'MISS path: N =', N, 'distinct arg keys (same state) => selector calls =', selector.mock.calls.length, '(expect', N, ')' );

	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	const sA = { posts: { a: postA } };
	getSitePosts( sA, 2916284 );
	const sB = { posts: { a: postA, b: postB } };
	getSitePosts( sB, 2916284 );
	log( 'RECOMPUTE-after-dependant-change: 2 calls, arg fixed, posts ref changed => selector calls =', selector.mock.calls.length, '(expect 2)' );

	// -------- Q3: per-arg entries + whole-cache invalidation + STALE reproduction --------
	log( '' );
	log( '----- Q3: per-arg entries, whole-cache flush, STALE -----' );
	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	const st = { posts: { a: postA, b: postB } };
	getSitePosts( st, 2916284 );
	getSitePosts( st, 38303081 );
	log( 'after calling args 2916284 & 38303081 (same state):' );
	log( "  cache.has('2916284') =", getSitePosts.memoizedSelector.cache.has( '2916284' ) );
	log( "  cache.has('38303081') =", getSitePosts.memoizedSelector.cache.has( '38303081' ), '(=> SEPARATE per-arg entries coexist)' );
	const stNew = { posts: { a: postA } };
	getSitePosts( stNew, 2916284 );
	log( 'after ONE dependant change (new posts ref), a single arg call:' );
	log( "  cache.has('2916284') =", getSitePosts.memoizedSelector.cache.has( '2916284' ), '(recomputed key present)' );
	log( "  cache.has('38303081') =", getSitePosts.memoizedSelector.cache.has( '38303081' ), '(=> the OTHER arg entry was FLUSHED by whole-cache clear)' );

	log( '' );
	log( 'STALE reproduction (in-place mutation, dependants stay reference-equal):' );
	const staleSelector = jest.fn( ( state, siteId ) => filter( state.posts, { site_ID: siteId } ) );
	const getStale = createSelector( staleSelector, ( state ) => state.posts );
	const posts = { a: { ID: 1, site_ID: 100, title: 'first' } };
	const stState = { posts };
	const r1 = getStale( stState, 100 );
	log( '  r1 (before mutation) =', JSON.stringify( r1 ) );
	posts.b = { ID: 2, site_ID: 100, title: 'second (added after r1)' };
	const r2 = getStale( stState, 100 );
	log( '  r2 (after in-place add of a 2nd matching post) =', JSON.stringify( r2 ) );
	log( '  r1 === r2 ?', r1 === r2, '  staleSelector calls =', staleSelector.mock.calls.length, '(=> STALE: change invisible, selector NOT re-run)' );
	const stState2 = { posts: { ...posts } };
	const r3 = getStale( stState2, 100 );
	log( '  after IMMUTABLE replace: r3 =', JSON.stringify( r3 ), ' staleSelector calls =', staleSelector.mock.calls.length, '(=> fresh)' );

	// -------- Q4: programmatic clearing --------
	log( '' );
	log( '----- Q4: programmatic clearing (memoizedSelector.cache.clear) -----' );
	selector.mockClear();
	getSitePosts.memoizedSelector.cache.clear();
	const s4 = { posts: { a: postA } };
	getSitePosts( s4, 2916284 );
	log( "BEFORE clear: cache.has('2916284') =", getSitePosts.memoizedSelector.cache.has( '2916284' ), ' selector calls =', selector.mock.calls.length );
	getSitePosts.memoizedSelector.cache.clear();
	log( "AFTER  clear: cache.has('2916284') =", getSitePosts.memoizedSelector.cache.has( '2916284' ) );
	getSitePosts( s4, 2916284 );
	log( 'after clear, same call again => selector calls =', selector.mock.calls.length, '(=> recomputed)' );

	// -------- Q5: nullish vs primitive DEPENDANTS (createSelector tolerates both) --------
	log( '' );
	log( '----- Q5: nullish vs primitive dependants (createSelector) -----' );
	const mkDepProbe = ( depVal ) => {
		const sel = jest.fn( () => 'R' );
		const gs = createSelector( sel, () => depVal );
		let threw = null;
		try {
			gs( {}, 1 );
			gs( {}, 1 );
		} catch ( e ) {
			threw = e && e.message;
		}
		return { threw, calls: sel.mock.calls.length };
	};
	for ( const [ label, val ] of [ [ 'null', null ], [ 'undefined', undefined ], [ 'number 5', 5 ], [ 'boolean true', true ] ] ) {
		const r = mkDepProbe( val );
		log( '  getDependants -> ' + label + ': threw =', JSON.stringify( r.threw ), ' selector calls(2 identical) =', r.calls, '(expect no throw, 1 call)' );
	}
	const selNU = jest.fn( () => 'R' );
	let depNU = null;
	const gsNU = createSelector( selNU, () => depNU );
	gsNU( {}, 1 );
	depNU = undefined;
	gsNU( {}, 1 );
	log( '  dependant null -> undefined (arg fixed): selector calls =', selNU.mock.calls.length, '(expect 2: null !== undefined under ===, cache busts)' );

	// -------- Q6: custom cache keys --------
	log( '' );
	log( '----- Q6: custom cache keys / complex args -----' );
	warn.mockClear();
	const gsWarn = createSelector( selector, ( state ) => state.posts );
	const sw = { posts: {} };
	gsWarn( sw, 1 );
	gsWarn( sw, '' );
	gsWarn( sw, 'foo' );
	gsWarn( sw, true );
	gsWarn( sw, null );
	gsWarn( sw, undefined );
	gsWarn( sw, {} );
	gsWarn( sw, [] );
	gsWarn( sw, 1, [] );
	log( 'default key, 9 calls w/ mixed args => warn (@wordpress/warning) calls =', warn.mock.calls.length, '(expect 3: {}, [], and [] in (1,[]))' );
	log( 'warn message[0] =', JSON.stringify( warn.mock.calls[ 0 ] && warn.mock.calls[ 0 ][ 0 ] ) );
	const getCustom = createSelector(
		selector,
		( state ) => state.posts,
		( state, siteId ) => `CUSTOM${ siteId }`
	);
	getCustom( { posts: {} }, 2916284 );
	log( "custom key (state, siteId) => 'CUSTOM'+siteId : cache.has('CUSTOM2916284') =", getCustom.memoizedSelector.cache.has( 'CUSTOM2916284' ), '(expect true)' );

	log( '' );
	log( '===== END createSelector PROBE =====' );

	// eslint-disable-next-line no-console
	console.log( '\n' + out.join( '\n' ) );
} );
```

### A.4 Probe script 2 — `packages/tree-select/test/probe_ts_TEMP.js`

```js
/* TEMPORARY OBSERVATION PROBE for treeSelect — Q1..Q6. To be deleted after capture. */
import treeSelect from '../src';

const out = [];
const log = ( ...a ) => out.push( a.join( ' ' ) );

test( 'treeSelect probe Q1..Q6', () => {
	log( '===== treeSelect PROBE =====' );
	log( 'process.env.NODE_ENV =', JSON.stringify( process.env.NODE_ENV ) );

	const post1 = { id: 'id1', text: 'post 1', siteId: 'site1' };
	const post2 = { id: 'id2', text: 'post 2', siteId: 'site1' };
	const post3 = { id: 'id3', text: 'post 3', siteId: 'site2' };

	const mk = () => {
		const selector = jest.fn( ( [ posts ], siteId ) =>
			Object.values( posts ).filter( ( p ) => p.siteId === siteId )
		);
		const getDependents = jest.fn( ( state ) => [ state.posts ] );
		return { selector, getSitePosts: treeSelect( getDependents, selector ) };
	};

	// -------- Q1 --------
	log( '' );
	log( '----- Q1: hit vs miss -----' );
	{
		const { selector, getSitePosts } = mk();
		const posts = { [ post1.id ]: post1, [ post2.id ]: post2, [ post3.id ]: post3 };
		const state = { posts };
		getSitePosts( state, 'site1' );
		getSitePosts( state, 'site1' );
		log( 'same posts ref + same arg, 2 calls => selector calls =', selector.mock.calls.length, '(expect 1: WeakMap leaf HIT)' );
		getSitePosts( state, 'site2' );
		log( 'same posts ref, NEW arg site2 => selector calls =', selector.mock.calls.length, '(expect 2: leaf Map key from args.join)' );
		const state2 = { posts: { ...posts } };
		getSitePosts( state2, 'site1' );
		log( 'NEW posts reference, arg site1 => selector calls =', selector.mock.calls.length, '(expect 3: dependent identity changed => fresh WeakMap branch)' );
	}

	// -------- Q2 --------
	log( '' );
	log( '----- Q2: call counts (SCALE stated) -----' );
	{
		const { selector, getSitePosts } = mk();
		const state = { posts: { [ post1.id ]: post1, [ post2.id ]: post2, [ post3.id ]: post3 } };
		const N = 1000;
		for ( let i = 0; i < N; i++ ) getSitePosts( state, 'site1' );
		log( 'HIT path: N =', N, 'identical calls => selector calls =', selector.mock.calls.length, '(expect 1)' );
	}
	{
		const { selector, getSitePosts } = mk();
		const state = { posts: { [ post1.id ]: post1, [ post2.id ]: post2, [ post3.id ]: post3 } };
		const N = 1000;
		for ( let i = 0; i < N; i++ ) getSitePosts( state, 'k' + i );
		log( 'MISS path: N =', N, 'distinct arg keys (same dependent) => selector calls =', selector.mock.calls.length, '(expect', N, ')' );
	}
	{
		const { selector, getSitePosts } = mk();
		const p = { [ post1.id ]: post1 };
		getSitePosts( { posts: p }, 'site1' );
		getSitePosts( { posts: { [ post1.id ]: { ...post1, modified: true } } }, 'site1' );
		log( 'RECOMPUTE-after-dependent-change: 2 calls, arg fixed, posts ref changed => selector calls =', selector.mock.calls.length, '(expect 2)' );
	}

	// -------- Q3 --------
	log( '' );
	log( '----- Q3: unique dependents coexist simultaneously -----' );
	{
		const spy = jest.fn( ( [ post ] ) => ( { ...post, withData: true } ) );
		const getPostByIdWithData = treeSelect( ( state, postId ) => [ state.posts[ postId ] ], spy );
		const state = { posts: { [ post1.id ]: post1, [ post2.id ]: post2 } };
		getPostByIdWithData( state, post1.id );
		getPostByIdWithData( state, post2.id );
		getPostByIdWithData( state, post1.id );
		log( 'calls id1,id2,id1 => selector calls =', spy.mock.calls.length, '(expect 2: BOTH dependent branches retained simultaneously; id1 re-hit)' );
		log( 'note: old dependent branches are evicted ONLY by GC of the WeakMap (not observable synchronously)' );
	}

	// -------- Q4 --------
	log( '' );
	log( '----- Q4: clearCache() -----' );
	{
		const { selector, getSitePosts } = mk();
		const state = { posts: { [ post1.id ]: post1, [ post2.id ]: post2, [ post3.id ]: post3 } };
		const firstResult = getSitePosts( state, 'site1' );
		const memoizedResult = getSitePosts( state, 'site1' );
		log( 'BEFORE clear: memoizedResult === firstResult ?', memoizedResult === firstResult, ' selector calls =', selector.mock.calls.length, '(expect true, 1)' );
		getSitePosts.clearCache();
		const afterClearResult = getSitePosts( state, 'site1' );
		log( 'AFTER  clearCache(): afterClearResult === firstResult ?', afterClearResult === firstResult, ' selector calls =', selector.mock.calls.length, '(expect false, 2 => fresh WeakMap, recompute)' );
	}

	// -------- Q5 --------
	log( '' );
	log( '----- Q5: nullish vs primitive dependents -----' );
	{
		const sel = jest.fn( () => ( {} ) );
		let dep = null;
		const s = treeSelect( () => [ dep ], sel );
		s( {} );
		dep = undefined;
		s( {} );
		log( 'dependent null THEN undefined => selector calls =', sel.mock.calls.length, '(expect 1: null & undefined SHARE NULLISH_KEY)' );
	}
	for ( const [ label, val ] of [ [ 'null', null ], [ 'undefined', undefined ], [ 'number 1', 1 ], [ 'boolean true', true ], [ "string a", 'a' ], [ 'number 0', 0 ], [ 'boolean false', false ] ] ) {
		const s = treeSelect( () => [ val ], () => [] );
		let threw = null;
		try {
			s( {} );
		} catch ( e ) {
			threw = e && ( e.constructor.name + ': ' + e.message );
		}
		log( '  dependent = ' + label + ' => ' + ( threw ? 'THREW ' + JSON.stringify( threw ) : 'OK (memoized)' ) );
	}

	// -------- Q6 --------
	log( '' );
	log( '----- Q6: custom cache keys / object args -----' );
	{
		const { getSitePosts } = mk();
		let threw = null;
		try {
			getSitePosts( { posts: {} }, {} );
		} catch ( e ) {
			threw = e && ( e.constructor.name + ': ' + e.message );
		}
		log( 'default key + object arg => ' + JSON.stringify( threw ) );
	}
	{
		const state = { posts: { [ post1.id ]: post1, [ post2.id ]: post2, [ post3.id ]: post3 } };
		const memoizedSelector = treeSelect(
			( s ) => [ s.posts ],
			( [ posts ], query ) => Object.values( posts ).filter( ( p ) => p.siteId === query.siteId ),
			{ getCacheKey: ( query ) => `key:${ query.siteId }` }
		);
		const firstResult = memoizedSelector( state, { siteId: 'site1', foo: 'bar' } );
		const secondResult = memoizedSelector( state, { siteId: 'site1', foo: 'baz' } );
		log( 'custom getCacheKey (query)=>key:query.siteId : firstResult === secondResult ?', firstResult === secondResult, '(expect true: distinct objects, same generated key)' );
		log( 'firstResult =', JSON.stringify( firstResult ) );
	}

	log( '' );
	log( '===== END treeSelect PROBE =====' );

	// eslint-disable-next-line no-console
	console.log( '\n' + out.join( '\n' ) );
} );
```

### A.5 Complete, unedited probe output

Both temporary probes were recreated on the real package entry points and executed with the
exact command below, **twice**, capturing the complete merged `stdout`+`stderr` each time. Both
runs exited `0` and reported `Test Suites: 2 passed, 2 total` / `Tests: 2 passed, 2 total`. The
two runs were **byte-identical except a single line** — the Jest `Time:` value (wall-clock
timing) — as the `diff` in §A.5.3 shows; the suite print order was stable (`createSelector`
first, then `treeSelect`) in both. The **complete, unedited** output of each run is reproduced
verbatim below, including the pre-existing global-harness warnings (`jest-haste-map` duplicate
mock + `Browserslist`), the Jest `● Console` / `console.log` prefixes and `at Object.log (…)`
stack lines, the `PASS` lines, and the suite/test totals.

**Exact command (run twice):**

```bash
yarn jest -c test/packages/jest.config.js packages/state-utils/src/create-selector/test/probe_cs_TEMP.js packages/tree-select/test/probe_ts_TEMP.js
```

#### A.5.1 Run 1 — complete, unedited output

```text
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
PASS packages/state-utils/src/create-selector/test/probe_cs_TEMP.js
  ● Console

    console.log
      
      ===== createSelector PROBE =====
      process.env.NODE_ENV = "test"
      
      ----- Q1: hit vs miss -----
      same state ref + same arg, 2 calls => selector calls = 1 (expect 1: cache HIT)
      same state ref, NEW arg 38303081 => selector calls = 2 (expect 2: new args.join key)
      NEW state.posts reference, arg 2916284 => selector calls = 3 (expect 3: isShallowEqual false => cache.clear then recompute)
      
      ----- Q2: call counts (SCALE stated) -----
      HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
      MISS path: N = 1000 distinct arg keys (same state) => selector calls = 1000 (expect 1000 )
      RECOMPUTE-after-dependant-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
      
      ----- Q3: per-arg entries, whole-cache flush, STALE -----
      after calling args 2916284 & 38303081 (same state):
        cache.has('2916284') = true
        cache.has('38303081') = true (=> SEPARATE per-arg entries coexist)
      after ONE dependant change (new posts ref), a single arg call:
        cache.has('2916284') = true (recomputed key present)
        cache.has('38303081') = false (=> the OTHER arg entry was FLUSHED by whole-cache clear)
      
      STALE reproduction (in-place mutation, dependants stay reference-equal):
        r1 (before mutation) = [{"ID":1,"site_ID":100,"title":"first"}]
        r2 (after in-place add of a 2nd matching post) = [{"ID":1,"site_ID":100,"title":"first"}]
        r1 === r2 ? true   staleSelector calls = 1 (=> STALE: change invisible, selector NOT re-run)
        after IMMUTABLE replace: r3 = [{"ID":1,"site_ID":100,"title":"first"},{"ID":2,"site_ID":100,"title":"second (added after r1)"}]  staleSelector calls = 2 (=> fresh)
      
      ----- Q4: programmatic clearing (memoizedSelector.cache.clear) -----
      BEFORE clear: cache.has('2916284') = true  selector calls = 1
      AFTER  clear: cache.has('2916284') = false
      after clear, same call again => selector calls = 2 (=> recomputed)
      
      ----- Q5: nullish vs primitive dependants (createSelector) -----
        getDependants -> null: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> undefined: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> number 5: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> boolean true: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        dependant null -> undefined (arg fixed): selector calls = 2 (expect 2: null !== undefined under ===, cache busts)
      
      ----- Q6: custom cache keys / complex args -----
      default key, 9 calls w/ mixed args => warn (@wordpress/warning) calls = 3 (expect 3: {}, [], and [] in (1,[]))
      warn message[0] = "Do not pass complex objects as arguments for a memoized selector"
      custom key (state, siteId) => 'CUSTOM'+siteId : cache.has('CUSTOM2916284') = true (expect true)
      
      ===== END createSelector PROBE =====

      at Object.log (src/create-selector/test/probe_cs_TEMP.js:163:10)

PASS packages/tree-select/test/probe_ts_TEMP.js
  ● Console

    console.log
      
      ===== treeSelect PROBE =====
      process.env.NODE_ENV = "test"
      
      ----- Q1: hit vs miss -----
      same posts ref + same arg, 2 calls => selector calls = 1 (expect 1: WeakMap leaf HIT)
      same posts ref, NEW arg site2 => selector calls = 2 (expect 2: leaf Map key from args.join)
      NEW posts reference, arg site1 => selector calls = 3 (expect 3: dependent identity changed => fresh WeakMap branch)
      
      ----- Q2: call counts (SCALE stated) -----
      HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
      MISS path: N = 1000 distinct arg keys (same dependent) => selector calls = 1000 (expect 1000 )
      RECOMPUTE-after-dependent-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
      
      ----- Q3: unique dependents coexist simultaneously -----
      calls id1,id2,id1 => selector calls = 2 (expect 2: BOTH dependent branches retained simultaneously; id1 re-hit)
      note: old dependent branches are evicted ONLY by GC of the WeakMap (not observable synchronously)
      
      ----- Q4: clearCache() -----
      BEFORE clear: memoizedResult === firstResult ? true  selector calls = 1 (expect true, 1)
      AFTER  clearCache(): afterClearResult === firstResult ? false  selector calls = 2 (expect false, 2 => fresh WeakMap, recompute)
      
      ----- Q5: nullish vs primitive dependents -----
      dependent null THEN undefined => selector calls = 1 (expect 1: null & undefined SHARE NULLISH_KEY)
        dependent = null => OK (memoized)
        dependent = undefined => OK (memoized)
        dependent = number 1 => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = boolean true => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = string a => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = number 0 => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = boolean false => THREW "TypeError: key must be an object, `null`, or `undefined`"
      
      ----- Q6: custom cache keys / object args -----
      default key + object arg => "Error: Do not pass objects as arguments to a treeSelector"
      custom getCacheKey (query)=>key:query.siteId : firstResult === secondResult ? true (expect true: distinct objects, same generated key)
      firstResult = [{"id":"id1","text":"post 1","siteId":"site1"},{"id":"id2","text":"post 2","siteId":"site1"}]
      
      ===== END treeSelect PROBE =====

      at Object.log (test/probe_ts_TEMP.js:146:10)


Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.046 s
Ran all test suites matching /packages\/state-utils\/src\/create-selector\/test\/probe_cs_TEMP.js|packages\/tree-select\/test\/probe_ts_TEMP.js/i in 2 projects.
```

#### A.5.2 Run 2 — complete, unedited output

```text
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
PASS packages/state-utils/src/create-selector/test/probe_cs_TEMP.js
  ● Console

    console.log
      
      ===== createSelector PROBE =====
      process.env.NODE_ENV = "test"
      
      ----- Q1: hit vs miss -----
      same state ref + same arg, 2 calls => selector calls = 1 (expect 1: cache HIT)
      same state ref, NEW arg 38303081 => selector calls = 2 (expect 2: new args.join key)
      NEW state.posts reference, arg 2916284 => selector calls = 3 (expect 3: isShallowEqual false => cache.clear then recompute)
      
      ----- Q2: call counts (SCALE stated) -----
      HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
      MISS path: N = 1000 distinct arg keys (same state) => selector calls = 1000 (expect 1000 )
      RECOMPUTE-after-dependant-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
      
      ----- Q3: per-arg entries, whole-cache flush, STALE -----
      after calling args 2916284 & 38303081 (same state):
        cache.has('2916284') = true
        cache.has('38303081') = true (=> SEPARATE per-arg entries coexist)
      after ONE dependant change (new posts ref), a single arg call:
        cache.has('2916284') = true (recomputed key present)
        cache.has('38303081') = false (=> the OTHER arg entry was FLUSHED by whole-cache clear)
      
      STALE reproduction (in-place mutation, dependants stay reference-equal):
        r1 (before mutation) = [{"ID":1,"site_ID":100,"title":"first"}]
        r2 (after in-place add of a 2nd matching post) = [{"ID":1,"site_ID":100,"title":"first"}]
        r1 === r2 ? true   staleSelector calls = 1 (=> STALE: change invisible, selector NOT re-run)
        after IMMUTABLE replace: r3 = [{"ID":1,"site_ID":100,"title":"first"},{"ID":2,"site_ID":100,"title":"second (added after r1)"}]  staleSelector calls = 2 (=> fresh)
      
      ----- Q4: programmatic clearing (memoizedSelector.cache.clear) -----
      BEFORE clear: cache.has('2916284') = true  selector calls = 1
      AFTER  clear: cache.has('2916284') = false
      after clear, same call again => selector calls = 2 (=> recomputed)
      
      ----- Q5: nullish vs primitive dependants (createSelector) -----
        getDependants -> null: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> undefined: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> number 5: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        getDependants -> boolean true: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
        dependant null -> undefined (arg fixed): selector calls = 2 (expect 2: null !== undefined under ===, cache busts)
      
      ----- Q6: custom cache keys / complex args -----
      default key, 9 calls w/ mixed args => warn (@wordpress/warning) calls = 3 (expect 3: {}, [], and [] in (1,[]))
      warn message[0] = "Do not pass complex objects as arguments for a memoized selector"
      custom key (state, siteId) => 'CUSTOM'+siteId : cache.has('CUSTOM2916284') = true (expect true)
      
      ===== END createSelector PROBE =====

      at Object.log (src/create-selector/test/probe_cs_TEMP.js:163:10)

PASS packages/tree-select/test/probe_ts_TEMP.js
  ● Console

    console.log
      
      ===== treeSelect PROBE =====
      process.env.NODE_ENV = "test"
      
      ----- Q1: hit vs miss -----
      same posts ref + same arg, 2 calls => selector calls = 1 (expect 1: WeakMap leaf HIT)
      same posts ref, NEW arg site2 => selector calls = 2 (expect 2: leaf Map key from args.join)
      NEW posts reference, arg site1 => selector calls = 3 (expect 3: dependent identity changed => fresh WeakMap branch)
      
      ----- Q2: call counts (SCALE stated) -----
      HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
      MISS path: N = 1000 distinct arg keys (same dependent) => selector calls = 1000 (expect 1000 )
      RECOMPUTE-after-dependent-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)
      
      ----- Q3: unique dependents coexist simultaneously -----
      calls id1,id2,id1 => selector calls = 2 (expect 2: BOTH dependent branches retained simultaneously; id1 re-hit)
      note: old dependent branches are evicted ONLY by GC of the WeakMap (not observable synchronously)
      
      ----- Q4: clearCache() -----
      BEFORE clear: memoizedResult === firstResult ? true  selector calls = 1 (expect true, 1)
      AFTER  clearCache(): afterClearResult === firstResult ? false  selector calls = 2 (expect false, 2 => fresh WeakMap, recompute)
      
      ----- Q5: nullish vs primitive dependents -----
      dependent null THEN undefined => selector calls = 1 (expect 1: null & undefined SHARE NULLISH_KEY)
        dependent = null => OK (memoized)
        dependent = undefined => OK (memoized)
        dependent = number 1 => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = boolean true => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = string a => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = number 0 => THREW "TypeError: key must be an object, `null`, or `undefined`"
        dependent = boolean false => THREW "TypeError: key must be an object, `null`, or `undefined`"
      
      ----- Q6: custom cache keys / object args -----
      default key + object arg => "Error: Do not pass objects as arguments to a treeSelector"
      custom getCacheKey (query)=>key:query.siteId : firstResult === secondResult ? true (expect true: distinct objects, same generated key)
      firstResult = [{"id":"id1","text":"post 1","siteId":"site1"},{"id":"id2","text":"post 2","siteId":"site1"}]
      
      ===== END treeSelect PROBE =====

      at Object.log (test/probe_ts_TEMP.js:146:10)


Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        0.855 s, estimated 1 s
Ran all test suites matching /packages\/state-utils\/src\/create-selector\/test\/probe_cs_TEMP.js|packages\/tree-select\/test\/probe_ts_TEMP.js/i in 2 projects.
```

#### A.5.3 Exact difference between the two runs

The only line that differs across the two runs is the Jest `Time:` line (wall-clock timing is
non-deterministic; the `estimated 1 s` suffix appears once Jest has a cached prior duration).
Every other line — every substantive value, the `PASS` lines, the suite/test totals, and the
suite print order — is identical:

```diff
--- run1
+++ run2
@@ -113,5 +113,5 @@
 Test Suites: 2 passed, 2 total
 Tests:       2 passed, 2 total
 Snapshots:   0 total
-Time:        1.046 s
+Time:        0.855 s, estimated 1 s
 Ran all test suites matching /packages\/state-utils\/src\/create-selector\/test\/probe_cs_TEMP.js|packages\/tree-select\/test\/probe_ts_TEMP.js/i in 2 projects.
```

#### A.5.4 Normalized `console.log` payloads (per probe)

For readability, the blocks below isolate just the `console.log` content each probe emits — the
same text that appears inside the `● Console` sections of the complete runs above.

**`createSelector` probe output:**

```text
===== createSelector PROBE =====
process.env.NODE_ENV = "test"

----- Q1: hit vs miss -----
same state ref + same arg, 2 calls => selector calls = 1 (expect 1: cache HIT)
same state ref, NEW arg 38303081 => selector calls = 2 (expect 2: new args.join key)
NEW state.posts reference, arg 2916284 => selector calls = 3 (expect 3: isShallowEqual false => cache.clear then recompute)

----- Q2: call counts (SCALE stated) -----
HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
MISS path: N = 1000 distinct arg keys (same state) => selector calls = 1000 (expect 1000 )
RECOMPUTE-after-dependant-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)

----- Q3: per-arg entries, whole-cache flush, STALE -----
after calling args 2916284 & 38303081 (same state):
  cache.has('2916284') = true
  cache.has('38303081') = true (=> SEPARATE per-arg entries coexist)
after ONE dependant change (new posts ref), a single arg call:
  cache.has('2916284') = true (recomputed key present)
  cache.has('38303081') = false (=> the OTHER arg entry was FLUSHED by whole-cache clear)

STALE reproduction (in-place mutation, dependants stay reference-equal):
  r1 (before mutation) = [{"ID":1,"site_ID":100,"title":"first"}]
  r2 (after in-place add of a 2nd matching post) = [{"ID":1,"site_ID":100,"title":"first"}]
  r1 === r2 ? true   staleSelector calls = 1 (=> STALE: change invisible, selector NOT re-run)
  after IMMUTABLE replace: r3 = [{"ID":1,"site_ID":100,"title":"first"},{"ID":2,"site_ID":100,"title":"second (added after r1)"}]  staleSelector calls = 2 (=> fresh)

----- Q4: programmatic clearing (memoizedSelector.cache.clear) -----
BEFORE clear: cache.has('2916284') = true  selector calls = 1
AFTER  clear: cache.has('2916284') = false
after clear, same call again => selector calls = 2 (=> recomputed)

----- Q5: nullish vs primitive dependants (createSelector) -----
  getDependants -> null: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> undefined: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> number 5: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  getDependants -> boolean true: threw = null  selector calls(2 identical) = 1 (expect no throw, 1 call)
  dependant null -> undefined (arg fixed): selector calls = 2 (expect 2: null !== undefined under ===, cache busts)

----- Q6: custom cache keys / complex args -----
default key, 9 calls w/ mixed args => warn (@wordpress/warning) calls = 3 (expect 3: {}, [], and [] in (1,[]))
warn message[0] = "Do not pass complex objects as arguments for a memoized selector"
custom key (state, siteId) => 'CUSTOM'+siteId : cache.has('CUSTOM2916284') = true (expect true)

===== END createSelector PROBE =====
```

**`treeSelect` probe output:**

```text
===== treeSelect PROBE =====
process.env.NODE_ENV = "test"

----- Q1: hit vs miss -----
same posts ref + same arg, 2 calls => selector calls = 1 (expect 1: WeakMap leaf HIT)
same posts ref, NEW arg site2 => selector calls = 2 (expect 2: leaf Map key from args.join)
NEW posts reference, arg site1 => selector calls = 3 (expect 3: dependent identity changed => fresh WeakMap branch)

----- Q2: call counts (SCALE stated) -----
HIT path: N = 1000 identical calls => selector calls = 1 (expect 1)
MISS path: N = 1000 distinct arg keys (same dependent) => selector calls = 1000 (expect 1000 )
RECOMPUTE-after-dependent-change: 2 calls, arg fixed, posts ref changed => selector calls = 2 (expect 2)

----- Q3: unique dependents coexist simultaneously -----
calls id1,id2,id1 => selector calls = 2 (expect 2: BOTH dependent branches retained simultaneously; id1 re-hit)
note: old dependent branches are evicted ONLY by GC of the WeakMap (not observable synchronously)

----- Q4: clearCache() -----
BEFORE clear: memoizedResult === firstResult ? true  selector calls = 1 (expect true, 1)
AFTER  clearCache(): afterClearResult === firstResult ? false  selector calls = 2 (expect false, 2 => fresh WeakMap, recompute)

----- Q5: nullish vs primitive dependents -----
dependent null THEN undefined => selector calls = 1 (expect 1: null & undefined SHARE NULLISH_KEY)
  dependent = null => OK (memoized)
  dependent = undefined => OK (memoized)
  dependent = number 1 => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = boolean true => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = string a => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = number 0 => THREW "TypeError: key must be an object, `null`, or `undefined`"
  dependent = boolean false => THREW "TypeError: key must be an object, `null`, or `undefined`"

----- Q6: custom cache keys / object args -----
default key + object arg => "Error: Do not pass objects as arguments to a treeSelector"
custom getCacheKey (query)=>key:query.siteId : firstResult === secondResult ? true (expect true: distinct objects, same generated key)
firstResult = [{"id":"id1","text":"post 1","siteId":"site1"},{"id":"id2","text":"post 2","siteId":"site1"}]

===== END treeSelect PROBE =====
```

---

_End of document. The temporary probe scripts were deleted after capture and are **absent**; this
deliverable is committed on the branch, the working tree is **clean** (`git status --porcelain` is
empty), and the baseline-to-HEAD diff (`git diff --name-status be7e5cc641..HEAD`) contains exactly
one added file: `A blitzy/documentation/wp-calypso_be7e5cc64162.md`._
