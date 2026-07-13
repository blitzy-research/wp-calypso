# Caching / Memoization Behavior of Calypso's Memoized Selectors — Investigation & Answer

> **Question being answered (verbatim):** _"I have a component that retrieves filtered data from a central store, and I'm seeing stale results being returned in certain conditions even after the underlying data has changed."_ Plus six specific follow-up questions (Q1–Q6, below).

This is a **run-first, read-only investigation**. Every behavioral claim below was produced by executing the **real** utility source through its canonical public API and capturing the unedited output; no source file was modified. Each claim carries the exact command, the complete unedited output, and a `file:line` reference. **Observed** facts (from my runs) are separated from **Inferred** facts (reasoned from the source).

---

## 0. Investigation metadata & methodology

| Item                         | Value                                                                                              |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| Repository commit            | `be7e5cc641622d153040491fd5625c6cb83e12eb`                                                         |
| Node.js runtime              | `v22.23.1` (satisfies `engines.node: "^v22.9.0"` and `.nvmrc: 22.9.0`)                             |
| Package manager              | `yarn@4.0.2` (Corepack)                                                                            |
| Primary utility              | `@automattic/tree-select` v2.0.0 `[packages/tree-select/package.json:L2-L3]`                       |
| Secondary utility (contrast) | `@automattic/state-utils` → `createSelector` `[packages/state-utils/src/create-selector/index.ts]` |
| Harness location             | `/tmp/obs` (outside the repository; removed after capture)                                         |
| Deliverable                  | this file only — repository left byte-for-byte unchanged otherwise                                 |

**Runtime-version resolution (per AAP §0.8.2).** The environment setup notes mention installing Node 20.x, but the repository's `engines.node` requires `^v22.9.0` and `.nvmrc` pins `22.9.0`. Node 20 would violate the engine constraint, so the **canonical supported runtime, Node 22.9.0+ (installed `v22.23.1`)**, was used for all observations. This lets the utility run exactly as it does in CI (`cimg/node:22.9.0`).

**Why a standalone harness is canonical.** `treeSelect` is a pure, self-contained module whose only runtime dependency (`tslib`) is not exercised by the memoization logic `[packages/tree-select/package.json:L36-L38]`. Node 22 strips TypeScript types natively, so the **real** `src/index.ts` was imported directly with no build step:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/<script>.mjs
```

Each script imports the default export from the real source:

```js
import treeSelect from '<REPO>/packages/tree-select/src/index.ts';
```

where `<REPO>` = `/tmp/blitzy/wp-calypso/blitzy-5e68b844-58da-44ac-a571-c83312f22855_d114d9`. A benign `MODULE_TYPELESS_PACKAGE_JSON` warning is emitted by Node (see §L); it is a runtime artifact of type-stripping, **not** a code issue, and the repo `package.json` was **not** edited to silence it. By default `process.env.NODE_ENV` is `undefined`, so the utility's **development-mode guards are active** unless a scenario explicitly sets `NODE_ENV=production`.

Every count-bearing scenario was run **at least twice** and only values **identical across both runs** are reported. The exact commands and complete unedited outputs are in the **Evidence Appendix (§L)**.

---

## 1. Executive summary (utility identification + root cause first)

**The utility.** Calypso's "central store" is its Redux store `[docs/our-approach-to-data.md:L3]`. The helper that memoizes "filtered data" derived from that store is **`treeSelect`** from `@automattic/tree-select` `[packages/tree-select/src/index.ts:L42-L102]`. Ten modules under `client/state/**` consume it (comments, invites, reader, and stats selector trees); the representative consumer `getSiteStatsNormalizedData` is a textbook example `[client/state/stats/lists/selectors.js:L139-L159]`.

**The root cause of "stale results even after the underlying data has changed" (details in §H).** `treeSelect` decides cache hits by the **referential identity (`===`)** of the values your `getDependents` returns — _not_ by deep equality `[packages/tree-select/src/index.ts:L84-L89]`, `[packages/tree-select/README.md:L40]`. If your component **mutates data in place** (e.g., `state.posts.id1.status = 'draft'`) without producing a **new reference** for the depended-upon slice, the dependent reference is unchanged, so the selector reports a **cache hit and returns the previously cached (stale) value**. I reproduced this exactly: the underlying selector was **not** re-invoked and the stale array was returned (`before === after`). The fix pattern (immutable update → new reference) busts the cache and returns fresh data.

**The six questions, answered in one line each (full evidence follows):**

| #   | Question                                        | Answer (observed)                                                                                                                                                                                                       |
| --- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Q1  | Precise comparison mechanism                    | Reference identity (`===`) via a **tree of `WeakMap`s** on each dependent, terminating in a leaf `Map` keyed by `getCacheKey(...args)` (default `args.join()`). **Not** deep equality. `[src/index.ts:L11-L12,L84-L89]` |
| Q2  | Underlying selector invocation counts           | same-state ×2 → **1**; same-state ×1000 → **1**; changed dependent → **2**; two distinct args → **2** `[test/index.js:L33-L46,L126-L159]`                                                                               |
| Q3  | Separate entries per argument, or invalidation? | **Multi-entry**: distinct args create coexisting leaf entries; a new arg does **not** evict a prior arg's result. `id1,id2,id1` → 2 computations, first survives `[src/index.ts:L86-L93]`, `[test/index.js:L161-L186]`  |
| Q4  | Programmatic full clear?                        | Yes — the returned selector exposes `clearCache()`, which recreates the root `WeakMap` `[src/index.ts:L96-L99]`                                                                                                         |
| Q5  | Nullish vs primitive dependents                 | `null` and `undefined` are memoized via a shared `NULLISH_KEY` sentinel; non-nullish primitives (number incl. `0`, boolean incl. `false`, string incl. `''`) **throw** `TypeError` `[src/index.ts:L107,L116-L120]`      |
| Q6  | Customize cache keys for complex objects?       | Yes — pass `options.getCacheKey`. Without it, an object argument **throws** in dev; with it, objects dedupe by the returned key `[src/index.ts:L24-L27,L67,L75-L78,L86]`                                                |

---

## Section A — Utility identification & overview

**Observed / Inferred.** `treeSelect` is a factory that returns a cached selector `[packages/tree-select/src/index.ts:L42-L102]`. Its documented purpose is to cache state-derived results "for use with the Redux global application state … whenever either the computation over state or React's rerenders are expensive" `[packages/tree-select/README.md:L3]`. This maps directly to the user's "component that retrieves filtered data from a central store": the "central store" is Redux `[docs/our-approach-to-data.md:L3]`, and the memoized selector sits between the store and the component.

**Real signature (Observed from source).** The actual parameter order is **`getDependents` first, `selector` second**, then an options bag:

```ts
// packages/tree-select/src/index.ts:L42-L56
export default function treeSelect< ... >(
	getDependents: ( state: State, ...args: Args ) => Deps,
	selector: ( deps: Deps, ...args: SArgs ) => Result,
	options: Options< Args > = {}
): CachedSelector< State, Args, Result >
```

The returned value is a `CachedSelector` — a callable that also carries a `clearCache()` method `[packages/tree-select/src/index.ts:L29-L32]`.

**Real consumer usage (Observed from source).** `getSiteStatsNormalizedData` uses the **correct** `getDependents`-first order and supplies a custom `getCacheKey` to serialize a complex `query` object:

```js
// client/state/stats/lists/selectors.js:L139-L159
export const getSiteStatsNormalizedData = treeSelect(
	( state, siteId, statType, query ) => [
		getSiteStatsForQuery( state, siteId, statType, query ),
		getSite( state, siteId ),
	],
	( [ siteStats, site ], siteId, statType, query ) => {
		/* …normalize… */
	},
	{
		getCacheKey: ( siteId, statType, query ) =>
			[ siteId, statType, getSerializedStatsQuery( query ) ].join(),
	}
);
```

Other consumers follow the same order, e.g. `getHiddenCommentsForPost` `[client/state/comments/selectors/get-hidden-comments-for-post.js:L8-L18]` and `getPostCommentsTree` `[client/state/comments/selectors/get-post-comments-tree.js:L16-L61]`.

Where behavior differs, this document contrasts `treeSelect` with `createSelector` from `@automattic/state-utils` `[packages/state-utils/src/create-selector/index.ts]` (full contrast in §I).

---

## Section B — Q1: the precise comparison mechanism (cache hit vs miss)

**Mechanism (Inferred from source, then Observed at runtime).** `treeSelect` maintains a **tree of `WeakMap`s** keyed on the **referential identity** of each element returned by `getDependents(state, ...args)`, terminating in a **leaf `Map`** keyed by the string `getCacheKey(...args)` (default `args.join()`):

- The default cache-key function is `args.join()` `[packages/tree-select/src/index.ts:L11-L12]`.
- The root cache is a `WeakMap` `[packages/tree-select/src/index.ts:L65]`.
- On each call, the dependents array is folded into the tree: `const leafCache = dependents.reduce( insertDependentKey, cache );` `[packages/tree-select/src/index.ts:L84]`. `insertDependentKey` descends/creates one map node per dependent, using **each dependent value as the map key** `[packages/tree-select/src/index.ts:L116-L131]`.
- Interior nodes are `WeakMap`s; the **last** node (the leaf) is a plain `Map`, because its key is the string `args.join()` rather than an object `[packages/tree-select/src/index.ts:L128]`.
- A **cache HIT** requires every dependent reference to be unchanged **and** a matching argument key: `const key = getCacheKey( ...args ); if ( leafCache.has( key ) ) return leafCache.get( key );` `[packages/tree-select/src/index.ts:L86-L89]`. Otherwise the selector recomputes and stores `leafCache.set( key, value )` `[packages/tree-select/src/index.ts:L91-L93]`.

Because `WeakMap`/`Map` lookups compare keys by **reference identity (`===`)**, a dependent that is a _different object but deeply equal_ is treated as a **miss**. The README states this explicitly: change is detected by a piece of state being "no longer referentially equal to its previous state (as opposed to a deep equality check)" `[packages/tree-select/README.md:L40]`.

**Cache structure diagram:**

```
        Root WeakMap
             |  key = dependents[0]  (by ===)
             v
          WeakMap
             |  key = dependents[1]  (by ===)
             v
        Leaf Map (plain Map)
             |  key = getCacheKey(...args)  (default: args.join())
             v
        Cached result
```

**Observed (runtime).** Two calls with the **same** `state.posts` reference and the same argument produce **one** computation and return the **identical** object; a **deep-equal but different** `state.posts` reference forces a **second** computation and a **different** result object (though equal by value). This proves identity, not deep equality.

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q1_comparison.mjs
```

Unedited output (identical across both runs):

```
same-ref: calls = 1 | r1===r2 (identity) = true
deep-equal-diff-ref: calls = 2 | r1===r3 = false
JSON r1 == JSON r3 (values equal) = true
```

- `same-ref … calls = 1` → same dependent reference + same arg = cache HIT (selector ran once). `[L87-L89]`
- `deep-equal-diff-ref … calls = 2` and `r1===r3 = false` → a new (deep-equal) reference is a MISS. `[L84]`
- `JSON … values equal = true` → the two results are value-equal, confirming the miss is purely reference-driven, not value-driven.

**Contrast (Observed, §I).** `createSelector` compares differently: it snapshots the dependants and clears its cache when a **shallow** (`@wordpress/is-shallow-equal`) comparison against the previous snapshot fails `[packages/state-utils/src/create-selector/index.ts:L103-L104]`, using lodash `memoize` for the per-argument cache `[packages/state-utils/src/create-selector/index.ts:L90]`.

**Rationale.** Reference-equality (not deep equality) is precisely why in-place mutation yields stale results — see §H.

---

## Section C — Q2: how many times the underlying selector runs (with concrete numbers)

**Run scale & stability (Observed).** A call counter wraps the underlying `selector`. Each scenario was executed **twice**; the counts below were **identical across both runs**. The at-scale scenario issues **1000** identical calls.

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q2_counts.mjs
```

Unedited output (identical across both runs):

```
A same-state x2 -> calls = 1
B same-state x1000 -> calls = 1
C changed-dependent (spread) -> calls = 2
D distinct-args -> calls = 2
```

| Scenario | Sequence                                             | Underlying selector calls | Why                                                                               |
| -------- | ---------------------------------------------------- | ------------------------- | --------------------------------------------------------------------------------- |
| A        | same `state`, same arg, ×2                           | **1**                     | second call is a cache HIT `[src/index.ts:L87-L89]`                               |
| B        | same `state`, same arg, ×1000                        | **1**                     | memoization holds at scale; only the first call computes `[src/index.ts:L87-L93]` |
| C        | changed dependent via immutable spread `{ ...post }` | **2**                     | new reference for the dependent = MISS `[src/index.ts:L84]`                       |
| D        | two distinct arguments                               | **2**                     | distinct leaf keys `args.join()` = two computations `[src/index.ts:L86]`          |

**Cross-reference to the committed test suite (canonical "test runs").** These numbers match the repository's own assertions:

- "should cache the result of a selector function" asserts `selector.mock.calls` has length **1** after two identical calls `[packages/tree-select/test/index.js:L33-L46]`.
- "should call selector when making non-cached calls" asserts length **2** for two distinct arguments `[packages/tree-select/test/index.js:L126-L140]`.
- "should bust the cache when watched state changes" asserts length **2** after an immutable spread copy `[packages/tree-select/test/index.js:L142-L159]`.

Running the committed suite (see §L for full output) confirms these pass:

```
CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci "packages/tree-select"
```

```
Tests:       17 passed, 17 total
```

> **Reconciliation (Observed vs the plan's stated figure).** The investigation plan described the suite as "15 tests"; the **observed** committed suite at this commit contains **17 tests** (all passing) `[packages/tree-select/test/index.js:L21-L264]`. Per the run-first mandate, the observed value (17) is reported. The count-bearing assertions cited above are unaffected.

---

## Section D — Q3: does the cache keep separate entries per argument, or invalidate?

**Answer (Observed): multi-entry.** Distinct arguments create **separate, coexisting** leaf entries. Calling the selector with a new argument does **not** invalidate a prior argument's cached result, as long as the dependents for that prior argument remain referentially stable. This follows from the leaf being a plain `Map` that stores one entry per distinct `getCacheKey(...args)` and only ever grows via `leafCache.set(...)` `[packages/tree-select/src/index.ts:L86-L93]`; there is no eviction of other keys.

**Observed (runtime).** The committed "unique dependents simultaneously" scenario: call with `id1`, then `id2`, then `id1` again. Three calls are issued, but only **two computations** occur — the third call (`id1`) is served from the surviving first entry.

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q3_coexistence.mjs
```

Unedited output (identical across both runs):

```
calls = 2 (3 calls issued)
first id1 result survived: a === c = true
id2 entry distinct: a === b = false
```

- `calls = 2` for **3** issued calls → the intervening `id2` call did **not** evict the `id1` entry.
- `a === c = true` → the **first** `id1` result object is returned again on the third call (it survived).
- `a === b = false` → `id1` and `id2` hold distinct coexisting entries.

**Cross-reference.** This mirrors the committed test "should maintain the cache for unique dependents simultaneously", which asserts exactly **2** computations for the `id1,id2,id1` sequence `[packages/tree-select/test/index.js:L161-L186]`.

**Comparative framing (context only — no dependency added).** Standard `reselect`'s classic default memoizer keeps a cache of size **1**, so switching arguments invalidates the single entry (a subsequent switch-back recomputes). `treeSelect` is **multi-entry** and is conceptually aligned with `reselect` v5's `weakMapMemoize` and the `re-reselect` per-key cache family. This directly contrasts with `createSelector`, which performs a **wholesale** cache clear when its dependants snapshot changes (see §I).

---

## Section E — Q4: is there a way to programmatically clear the entire cache?

**Answer (Observed): yes — `selector.clearCache()`.** The returned cached selector carries a `clearCache` method that **recreates the root `WeakMap`**, since `WeakMap` has no native `.clear()`:

```ts
// packages/tree-select/src/index.ts:L96-L99
cachedSelector.clearCache = () => {
	// WeakMap doesn't have `clear` method, so we need to recreate it
	cache = new WeakMap();
};
```

Because the entire tree hangs off that single root `WeakMap` `[packages/tree-select/src/index.ts:L65,L84]`, replacing it invalidates **all** entries at once.

**Observed (runtime).** Before clearing, a repeated call returns the **identical** object; after `clearCache()`, the next call computes a **new** result object (equal by value, different reference).

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q4_clearcache.mjs
```

Unedited output (identical across both runs):

```
before clearCache: calls = 1 | memoizedResult === firstResult = true
after clearCache:  calls = 2 | afterClearResult === firstResult = false
value still equal after clear: JSON equal = true
```

- `before … calls = 1 … === firstResult = true` → the second call was a cache HIT.
- `after … calls = 2 … === firstResult = false` → `clearCache()` forced recomputation; the new result is a different object reference `[src/index.ts:L96-L99]`.
- `value still equal … JSON equal = true` → the recomputed value is equal by content, confirming only the cache (not the logic) changed.

**Cross-reference.** The committed test "should bust the cache when clearCache() method is called" asserts the post-clear result is `not.toBe` the pre-clear result `[packages/tree-select/test/index.js:L197-L217]`.

**Contrast (§I).** `createSelector` does not expose a `clearCache()`. Instead it exposes its underlying lodash-memoized function at `selector.memoizedSelector`, whose lodash cache can be cleared via `selector.memoizedSelector.cache.clear()` `[packages/state-utils/src/create-selector/index.ts:L90,L111]` (the committed state-utils test uses exactly this in its `beforeEach` `[packages/state-utils/src/create-selector/test/index.js:L17]`).

---

## Section F — Q5: nullish vs primitive values returned by the dependency getter

This question concerns what `getDependents` **returns** (the "dependents"). Each dependent becomes a key in the `WeakMap` tree via `insertDependentKey` `[packages/tree-select/src/index.ts:L116-L131]`. That function branches on the key's type:

```ts
// packages/tree-select/src/index.ts:L118-L121
if ( key != null && Object( key ) !== key ) {
	throw new TypeError( 'key must be an object, `null`, or `undefined`' );
}
const weakMapKey = key || NULLISH_KEY;
```

- **`null` and `undefined` (nullish) — memoized.** `key != null` is `false` for both, so the throw is skipped; the falsy key is replaced by a shared sentinel object `const NULLISH_KEY = {};` `[packages/tree-select/src/index.ts:L107,L121]`. Every nullish dependent thus maps to the **same** `WeakMap` slot, so repeated calls memoize normally.
- **Non-nullish primitives (number, boolean, string) — throw.** For a value like `1`, `true`, `'a'`, `0`, `false`, or `''`, `key != null` is `true` and `Object(key) !== key` is `true` (a primitive is not its own boxed object), so it **throws** a `TypeError` whose message is `` key must be an object, `null`, or `undefined` `` `[packages/tree-select/src/index.ts:L118-L120]`.

**Observed (runtime), addressing `null`, `undefined`, number, and boolean each by name:**

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q5_nullish_primitive.mjs
```

Unedited output (identical across both runs):

```
nullish dependents [null, undefined]: firstResult === secondResult = true
--- non-nullish primitive dependents (each thrown error captured) ---
dependent = true | threw = true | TypeError: key must be an object, `null`, or `undefined`
dependent = 1 | threw = true | TypeError: key must be an object, `null`, or `undefined`
dependent = "a" | threw = true | TypeError: key must be an object, `null`, or `undefined`
dependent = false | threw = true | TypeError: key must be an object, `null`, or `undefined`
dependent = "" | threw = true | TypeError: key must be an object, `null`, or `undefined`
dependent = 0 | threw = true | TypeError: key must be an object, `null`, or `undefined`
--- primitive ARGUMENTS to selector (distinct from dependents) ---
arg = 1 | threw = false
arg = "" | threw = false
arg = "foo" | threw = false
arg = true | threw = false
arg = null | threw = false
arg = undefined | threw = false
```

By name:

- **`null`** → memoized (shares `NULLISH_KEY`); `firstResult === secondResult = true`.
- **`undefined`** → memoized (shares `NULLISH_KEY`); returned in the same `[ null, undefined ]` dependents pair.
- **number** → `1` throws; `0` (a falsy number) **also** throws — note `0` is _not_ treated as nullish here, because the throw check at `[L118]` runs **before** the `|| NULLISH_KEY` fallback at `[L121]`.
- **boolean** → `true` throws; `false` (a falsy boolean) **also** throws, for the same ordering reason.
- string (incl. empty `''`) → throws as well, shown for completeness.

**Cross-reference.** The committed tests assert this precisely: "should memoize a nullish value returned by getDependents" returns identical results for `[ null, undefined ]` `[packages/tree-select/test/index.js:L219-L229]`; "throws on a non-nullish primitive value returned by getDependents" iterates exactly `[ true, 1, 'a', false, '', 0 ]` and expects each to throw `[packages/tree-select/test/index.js:L231-L241]`.

**CRITICAL NUANCE — do not conflate dependents with arguments (Observed).** Q5 is about values `getDependents` **returns**. Primitives passed as **arguments** to the selector are an entirely different code path: they are joined into the cache key via `getCacheKey(...args)` and do **not** throw. The output above shows arguments `[ 1, '', 'foo', true, null, undefined ]` **all** succeeding (`threw = false`), matching the committed test "should not throw an error in development when given primitives" `[packages/tree-select/test/index.js:L115-L124]`. Keep the two cases separate: **primitive dependents throw; primitive arguments are fine.**

**Environment note (forward-reference to §K).** The primitive-**dependent** `TypeError` at `[L118-L120]` is **unconditional** — it is _not_ wrapped in a `NODE_ENV` guard and therefore throws in **both** development and production. This differs from the two argument-guards, which are dev-only.

---

## Section G — Q6: customizing cache-key generation for complex query objects

**Answer (Observed): yes — supply `options.getCacheKey`.** The options bag accepts an optional `getCacheKey` `[packages/tree-select/src/index.ts:L24-L27]`, destructured with the default `args.join()` `[packages/tree-select/src/index.ts:L67]` and applied to the leaf key `[packages/tree-select/src/index.ts:L86]`.

**Without a custom key, object arguments throw in development.** A dev-only guard rejects object arguments when the default key function is in use:

```ts
// packages/tree-select/src/index.ts:L75-L79
if ( process.env.NODE_ENV !== 'production' ) {
	if ( getCacheKey === defaultGetCacheKey && args.some( isObject ) ) {
		throw new Error( 'Do not pass objects as arguments to a treeSelector' );
	}
}
```

The rationale: the default key is `args.join()`, and objects stringify to `"[object Object]"`, which would collide across distinct objects and produce incorrect cache hits.

**With `options.getCacheKey`, object arguments are enabled and deduplication is controlled by the returned key.**

**Observed (runtime).**

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q6_cachekey.mjs
```

Unedited output (identical across both runs):

```
object-arg WITHOUT getCacheKey: threw = true | Error: Do not pass objects as arguments to a treeSelector
with getCacheKey: calls = 1 | firstResult === secondResult = true
result value = ["id1","id2"]
distinct siteId key: calls = 2 | thirdResult === firstResult = false
```

- Without a custom key, `getSitePosts(state, { siteId: 'site1' })` throws `Error: Do not pass objects as arguments to a treeSelector` `[src/index.ts:L75-L78]`.
- With the option ``{ getCacheKey: ( query ) => `key:${ query.siteId }` }``, two **non-identical** objects that share `siteId` (`{ siteId: 'site1', foo: 'bar' }` and `{ siteId: 'site1', foo: 'baz' }`) produce the **same** key → `calls = 1`, `firstResult === secondResult = true` (deduped to one cached result).
- A distinct `siteId` produces a distinct key → `calls = 2`, `thirdResult === firstResult = false` (separate entry).

**Cross-reference.** The committed test "accepts a getCacheKey option that enables object arguments" builds exactly this selector with ``{ getCacheKey: ( query ) => `key:${ query.siteId }` }`` and asserts the second call returns the memoized result `[packages/tree-select/test/index.js:L243-L264]`.

**Real consumer pattern (Observed from source).** This is precisely how `getSiteStatsNormalizedData` serializes a complex `query` object into a stable string key `[client/state/stats/lists/selectors.js:L155-L158]`:

```js
{
	getCacheKey: ( siteId, statType, query ) =>
		[ siteId, statType, getSerializedStatsQuery( query ) ].join(),
}
```

---

## Section H — Stale-results ROOT CAUSE (the reported symptom) — centerpiece

**The cause (Observed + Inferred).** `treeSelect` decides cache hits by the **referential identity** of the values `getDependents` returns `[packages/tree-select/src/index.ts:L84-L89]`. When your component **mutates state in place** — changing a nested field without producing a **new reference** for the depended-upon slice — the dependent reference is unchanged, the tree walk lands on the **same** leaf entry, and `leafCache.has(key)` is `true`. The selector therefore **returns the previously cached (stale) value and does not recompute**. This is the exact condition in the symptom: _"stale results being returned … even after the underlying data has changed."_

**Reproduction (Observed).** To make staleness unambiguous, the selector returns a **derived** value (a filtered list of post ids whose `status === 'published'`), not a reference to the mutated objects. `getDependents` returns a **stable** `state.posts` reference while a nested field is mutated in place (`postsObj.id1.status = 'draft'`). The contrasting case performs an immutable update (new `state.posts` and new post object).

Command:

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/rootcause.mjs
```

Unedited output (identical across both runs):

```
IN-PLACE: initial result = ["id1"] | calls = 1
IN-PLACE: after mutation = ["id1"] | calls = 1 | correct would be []
IN-PLACE: same cached array returned: before === after = true
IN-PLACE: STALE (still shows id1) = true
IMMUTABLE: initial result = ["id1"] | calls = 1
IMMUTABLE: after update = [] | calls = 2
IMMUTABLE: FRESH (now []) = true
```

**Reading the output:**

- **In-place mutation (the bug).** After `postsObj.id1.status = 'draft'`, the correct derived result is `[]` (no published posts). Instead the selector returned the **stale** `["id1"]`, the underlying selector `calls` **stayed at 1** (never re-ran), and the **same array object** was returned (`before === after = true`). The reference of `state.posts` never changed, so `treeSelect` reported a hit `[packages/tree-select/src/index.ts:L84-L89]`.
- **Immutable update (the fix pattern).** Creating a **new** `state.posts` containing a **new** post object changed the dependent reference, busting the cache: `calls` became **2** and the **fresh** `[]` was returned.

**Cross-reference.** The immutable-update branch corresponds to the committed test "should bust the cache when watched state changes", which uses an immutable spread `{ ...post1, modified: true }` and asserts the selector runs a second time `[packages/tree-select/test/index.js:L142-L159]`.

**Definitive statement.** The stale-results symptom is caused by **mutating data in place without creating a new reference** for the slice returned by `getDependents`. Because `treeSelect` uses referential-identity comparison (not deep equality), such a mutation is invisible to the cache and the stale value is served. Redux's own guidance — never mutate state in place, always produce new references — is what keeps `treeSelect` correct; the README notes this dependency on referential inequality directly `[packages/tree-select/README.md:L40]`. _(Per the read-only constraint and the task scope, this document explains and reproduces the behavior; it does not modify the utility or any consumer.)_

---

## Section I — `treeSelect` vs `createSelector` contrast

| Dimension                | `treeSelect` `[packages/tree-select/src/index.ts]`                                 | `createSelector` `[packages/state-utils/src/create-selector/index.ts]`                            |
| ------------------------ | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Change detection         | Referential identity (`===`) per dependent, via `WeakMap` tree `[L65,L84-L89]`     | **Shallow** equality of the dependants snapshot via `@wordpress/is-shallow-equal` `[L103-L104]`   |
| Cache structure          | Tree of `WeakMap`s + leaf `Map` per-argument `[L84,L128]`                          | lodash `memoize` cache keyed by `args.join()` `[L90]`, README `[create-selector/README.md:L48]`   |
| Multi-argument behavior  | **Multi-entry**: per-argument results coexist; per-branch invalidation `[L86-L93]` | **Wholesale clear**: any dependants-snapshot change clears the entire memoize cache `[L103-L104]` |
| Programmatic clear       | `selector.clearCache()` recreates root `WeakMap` `[L96-L99]`                       | `selector.memoizedSelector.cache.clear()` `[L90,L111]`                                            |
| Complex object arguments | **Throws** in dev without `getCacheKey` `[L75-L78]`                                | **Warns** (does not throw) in dev `[L41-L52]`                                                     |
| GC characteristics       | `WeakMap`-based → entries for dead dependents are collectable `[README.md:L4]`     | lodash `Map`-based cache; cleared wholesale on change                                             |

**Observed — wholesale-clear contrast.** Using the real `createSelector` logic (provenance labeled below), I drove two distinct arguments under a **stable** dependants snapshot, then changed the dependants reference:

Command:

```
node --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/createselector_contrast.mjs
```

Unedited output (identical across both runs):

```
argA (state1):        calls = 1
argB (state1):        calls = 2
argA again (state1):  calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):        calls = 4 (B entry was wiped too)
complex-arg: threw = false | warnCount = 1 | message = "Do not pass complex objects as arguments for a memoized selector"
```

- Under a stable dependants snapshot, `argA` and `argB` **coexist** (calls `1 → 2 → 2`), similar to `treeSelect` (§D).
- When the dependants reference changes (shallow-unequal), **both** argument entries are wiped and recomputed (calls `3`, then `4`) — the **wholesale** invalidation `[packages/state-utils/src/create-selector/index.ts:L103-L104]`. This is the key behavioral difference from `treeSelect`'s per-branch retention (§D).

**PROVENANCE LABEL (mandatory — non-canonical dependency loading).** `createSelector` imports `import { memoize } from 'lodash'`, `@wordpress/is-shallow-equal`, and `@wordpress/warning` `[packages/state-utils/src/create-selector/index.ts:L1-L3]`. Node's raw ESM loader cannot read the CJS **named** export `memoize` from `lodash` (the repo's Babel/webpack build normally handles that interop). To run the **real** logic, the verbatim source file was copied out of the repository (sha256 `2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02`, **byte-identical** to the repo file) and bundled with `esbuild` (which resolves CJS named imports like the repo bundler). Crucially, the bundle resolved against the **repository's own already-installed, pinned** dependency versions:

- `lodash@4.17.21` (manifest range `^4.17.21`) `[packages/state-utils/package.json]`
- `@wordpress/is-shallow-equal@5.21.0` (range `^5.21.0`)
- `@wordpress/warning@3.21.0` (range `^3.21.0`)

These are the **exact pinned versions**, so the `createSelector` values here are sourced from the real logic + the repository's canonical dependency set, cross-referenced to the committed test suite. _(The `esbuild` step only read the repo source and wrote output to `/tmp/obs`; `git status` remained clean, and the source sha256 was unchanged.)_

**Warn behavior (version-sensitive — labeled).** The complex-argument path **warns, it does not throw** `[packages/state-utils/src/create-selector/index.ts:L41-L52]`. Observed: `threw = false`, message `Do not pass complex objects as arguments for a memoized selector`, and the selector still returns a value. The installed `@wordpress/warning@3.21.0` gates its `console.warn` on `globalThis.SCRIPT_DEBUG === true` (**not** `NODE_ENV`) and dedupes by message (verified in `node_modules/@wordpress/warning/build/index.js`). Hence a standalone run shows `warnCount = 1` **only** when `SCRIPT_DEBUG` is set (as it was here). **The canonical warn-count assertion is the committed test** — it mocks `warn` and asserts it is called **3** times for the complex-argument inputs `[packages/state-utils/src/create-selector/test/index.js:L65-L78]`. Reported as: warn **observed firing** (dev flag on) **and** committed-test-canonical (3); the `SCRIPT_DEBUG`/version gate is **Inferred** from the installed package source.

**Ecosystem framing (context only, no dependency added).** `treeSelect`'s `WeakMap` tree + per-argument leaf `Map` places it in the same family as `reselect` v5's `weakMapMemoize` and `re-reselect`'s per-key caches (multi-entry, reference-based), whereas classic `reselect` (cache size 1) and `createSelector`'s wholesale-clear behave as single-snapshot memoizers.

---

## Section J — Documentation discrepancy (flagged, NOT fixed)

**Observed.** The `README.md` usage examples show the arguments in the **reversed** order — `treeSelect( selector, getDependents )`:

- `const getSitePosts = treeSelect( selector, getDependents );` `[packages/tree-select/README.md:L19]`
- `const cachedSelector = treeSelect( selector, getDependents );` `[packages/tree-select/README.md:L49]`

This is **backwards** relative to the actual implementation signature, which is `treeSelect( getDependents, selector, options )` `[packages/tree-select/src/index.ts:L42-L56]`. The **code is correct**, as confirmed by every real consumer — e.g. `getSiteStatsNormalizedData` passes `getDependents` first `[client/state/stats/lists/selectors.js:L139-L159]`, as do `getHiddenCommentsForPost` `[client/state/comments/selectors/get-hidden-comments-for-post.js:L8-L18]` and `getPostCommentsTree` `[client/state/comments/selectors/get-post-comments-tree.js:L16-L61]`. The README **examples** are the erroneous part.

Per the read-only constraint of this investigation, this is noted as an **observation only**; the README is **not** edited.

---

## Section K — Environment note: development-mode vs production guards

`treeSelect` contains two **argument** guards wrapped in `process.env.NODE_ENV !== 'production'`, plus one **unconditional** dependent guard:

1. **Invalid-arguments guard (dev-only).** At creation, throws if `getDependents`/`selector` are not both functions `[packages/tree-select/src/index.ts:L57-L63]`.
2. **Object-argument guard (dev-only).** On call, throws for object arguments when the default key function is used `[packages/tree-select/src/index.ts:L75-L79]`.
3. **Primitive-dependent guard (UNCONDITIONAL).** Inside `insertDependentKey`, throws for a non-nullish primitive dependent — **not** wrapped in any `NODE_ENV` check `[packages/tree-select/src/index.ts:L118-L120]`.

**Observed — dev mode (`NODE_ENV` unset):**

```
node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/devprod.mjs
```

```
NODE_ENV = undefined
treeSelect(undefined, undefined) at creation: threw = true | treeSelect: invalid arguments passed, selector and getDependents must both be functions
object arg {} to selector: threw = true | Do not pass objects as arguments to a treeSelector
primitive dependent (number 1): threw = true | TypeError: key must be an object, `null`, or `undefined`
```

**Observed — production mode (`NODE_ENV=production`):**

```
NODE_ENV=production node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/devprod.mjs
```

```
NODE_ENV = "production"
treeSelect(undefined, undefined) at creation: threw = false
object arg {} to selector: threw = false
primitive dependent (number 1): threw = true | TypeError: key must be an object, `null`, or `undefined`
```

**Reading the outputs:**

- In **production**, the two argument guards are **skipped**: `treeSelect(undefined, undefined)` does **not** throw at creation, and an object argument does **not** throw `[packages/tree-select/src/index.ts:L57-L63,L75-L79]`.
- The primitive-**dependent** `TypeError` **still throws in production** — it is unconditional `[packages/tree-select/src/index.ts:L118-L120]`.

**Cross-reference.** Committed tests assert both sides: "should not throw an error in production for missing args" and "should not throw an error in production even when given object arguments" `[packages/tree-select/test/index.js:L86-L93,L103-L113]`; the unconditional dependent throw is asserted (without env-gating) in "throws on a non-nullish primitive value returned by getDependents" `[packages/tree-select/test/index.js:L231-L241]`.

**Which environment each observed error belongs to:**

| Error text                                                                                | Guard location             | Throws in dev? | Throws in prod?         |
| ----------------------------------------------------------------------------------------- | -------------------------- | -------------- | ----------------------- |
| `treeSelect: invalid arguments passed, selector and getDependents must both be functions` | `[src/index.ts:L57-L63]`   | Yes            | **No**                  |
| `Do not pass objects as arguments to a treeSelector`                                      | `[src/index.ts:L75-L78]`   | Yes            | **No**                  |
| `` TypeError: key must be an object, `null`, or `undefined` ``                            | `[src/index.ts:L118-L120]` | Yes            | **Yes (unconditional)** |

---

## Section L — Evidence appendix

### L.1 Q1–Q6 + root cause → command → observed output → source citation

| Item                            | Command (harness in `/tmp/obs`)                                                                                                   | Observed key result                                                                    | Source citation                                                        |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Q1 comparison                   | `node --experimental-strip-types … q1_comparison.mjs`                                                                             | same-ref calls=1 (hit); deep-equal diff-ref calls=2 (miss), values equal               | `[src/index.ts:L84-L89]`, `[README.md:L40]`                            |
| Q2 counts                       | `… q2_counts.mjs`                                                                                                                 | 1 / 1 (×1000) / 2 / 2                                                                  | `[test/index.js:L33-L46,L126-L159]`                                    |
| Q3 coexistence                  | `… q3_coexistence.mjs`                                                                                                            | 3 calls → 2 computations; first id1 survives                                           | `[src/index.ts:L86-L93]`, `[test/index.js:L161-L186]`                  |
| Q4 clearCache                   | `… q4_clearcache.mjs`                                                                                                             | before: same ref; after clearCache: new ref, equal value                               | `[src/index.ts:L96-L99]`, `[test/index.js:L197-L217]`                  |
| Q5 nullish/primitive            | `… q5_nullish_primitive.mjs`                                                                                                      | null+undefined memoized; [true,1,'a',false,'',0] throw; primitive args OK              | `[src/index.ts:L107,L116-L120]`, `[test/index.js:L115-L124,L219-L241]` |
| Q6 getCacheKey                  | `… q6_cachekey.mjs`                                                                                                               | object-arg no-key throws; with key, same-siteId dedupes (calls=1)                      | `[src/index.ts:L24-L27,L67,L75-L78,L86]`, `[test/index.js:L243-L264]`  |
| Root cause                      | `… rootcause.mjs`                                                                                                                 | in-place mutation → calls stays 1 (stale, before===after); immutable → calls 2 (fresh) | `[src/index.ts:L84-L89]`, `[test/index.js:L142-L159]`                  |
| Dev vs prod                     | `… devprod.mjs` (± `NODE_ENV=production`)                                                                                         | arg guards dev-only; primitive-dependent throw unconditional                           | `[src/index.ts:L57-L63,L75-L79,L118-L120]`                             |
| createSelector contrast         | `node … createselector_contrast.mjs`                                                                                              | wholesale clear 1,2,2 → 3,4; warn (not throw), warnCount=1                             | `[create-selector/index.ts:L41-L52,L90,L103-L104]`                     |
| Committed tree-select suite     | `CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci "packages/tree-select"`                     | 17 passed, 17 total                                                                    | `[test/index.js:L21-L264]`                                             |
| Committed create-selector suite | `CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci "packages/state-utils/src/create-selector"` | 13 passed, 13 total                                                                    | `[create-selector/test/index.js]`                                      |

### L.2 Committed tree-select suite — full unedited output (canonical "test runs" for Q2)

```
CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci "packages/tree-select"
```

```
PASS packages/tree-select/test/index.js
  index
    #treeSelect
      ✓ should create a function which returns the expected value when called (3 ms)
      ✓ should cache the result of a selector function (1 ms)
      ✓ should cache the result of a selector function that has multiple dependents (1 ms)
      ✓ should throw an error if getDependents is missing (10 ms)
      ✓ should throw an error if selector is missing (1 ms)
      ✓ should not throw an error in production for missing args (1 ms)
      ✓ should throw an error in development when given object arguments (2 ms)
      ✓ should not throw an error in production even when given object arguments (1 ms)
      ✓ should not throw an error in development when given primitives (5 ms)
      ✓ should call selector when making non-cached calls (1 ms)
      ✓ should bust the cache when watched state changes
      ✓ should maintain the cache for unique dependents simultaneously
      ✓ should call dependant state getter with dependents and arguments
      ✓ should bust the cache when clearCache() method is called (1 ms)
      ✓ should memoize a nullish value returned by getDependents
      ✓ throws on a non-nullish primitive value returned by getDependents (2 ms)
      ✓ accepts a getCacheKey option that enables object arguments (1 ms)

Test Suites: 1 passed, 1 total
Tests:       17 passed, 17 total
Snapshots:   0 total
Time:        0.683 s, estimated 3 s
Ran all test suites matching /packages\/tree-select/i.
```

> Non-fatal stderr noise (pre-existing repo state, unrelated to `treeSelect`): a `jest-haste-map: duplicate manual mock found: wpcom-proxy-request` notice (from committed `dist/` build artifacts alongside `src/`) and a `Browserslist: … caniuse-lite is 17 months old` notice. Neither affects the tree-select assertions.

### L.3 Observed vs Inferred

**Observed (captured directly from my runs):**

- Cache hits are reference-based: same-ref hit / deep-equal-diff-ref miss (§B).
- Invocation counts 1 / 1(×1000) / 2 / 2 (§C), stable across two runs.
- Multi-entry coexistence: 3 calls → 2 computations, first entry survives (§D).
- `clearCache()` forces a new result reference (§E).
- `null`/`undefined` dependents memoized; `[true,1,'a',false,'',0]` dependents throw the exact `TypeError`; primitive **arguments** do not throw (§F).
- Object argument without `getCacheKey` throws the exact `Error`; with `getCacheKey`, same-`siteId` objects dedupe (§G).
- In-place mutation → stale cached array (calls stays 1); immutable update → fresh (calls 2) (§H).
- `createSelector` wholesale-clear sequence 1,2,2 → 3,4; complex-arg warns (does not throw) (§I).
- Dev-only arg guards vs unconditional primitive-dependent throw (§K).
- Both committed suites pass (17 and 13 tests).

**Inferred (reasoned from source, not directly executed as a standalone assertion):**

- Interior tree nodes are `WeakMap`s and only the leaf is a plain `Map` `[src/index.ts:L128]` (inferred from the code; the runtime behavior is consistent with it but the node types were not separately introspected).
- `0`/`false`/`''` dependents throw because the throw check `[L118]` precedes the `|| NULLISH_KEY` fallback `[L121]` (source-order reasoning; the throw itself is Observed).
- `@wordpress/warning@3.21.0` gates on `globalThis.SCRIPT_DEBUG` and dedupes by message (read from the installed package source; the resulting `warnCount=1` is Observed).
- The `WeakMap` GC-friendliness claim `[README.md:L4]` is a documented design property, not something exercised at runtime here.

### L.4 Two-run stability & run scale

Every count-bearing scenario (Q2, Q3, Q4, Q6, root cause, createSelector contrast) was executed **twice**; the outputs quoted above were **identical across both runs**. The at-scale same-state scenario used a run scale of **1000** identical calls and still produced a single computation (§C).

### L.5 Benign runtime artifact

Importing a `.ts` file directly under Node's type-stripping emits this warning (captured verbatim, suppressed in scenario runs via `--disable-warning=MODULE_TYPELESS_PACKAGE_JSON`):

```
(node:22122) [MODULE_TYPELESS_PACKAGE_JSON] Warning: Module type of file:///…/packages/tree-select/src/index.ts is not specified and it doesn't parse as CommonJS.
Reparsing as ES module because module syntax was detected. This incurs a performance overhead.
To eliminate this warning, add "type": "module" to /…/packages/tree-select/package.json.
```

This is a **runtime import artifact, not a code defect**. Per the read-only constraint, `packages/tree-select/package.json` was **not** modified to silence it.

### L.6 Harness & repository integrity

- All observation scripts lived in `/tmp/obs` (outside the repository) and were removed after capture.
- The `createSelector` bundle was produced by `esbuild` reading the repo source read-only and writing to `/tmp/obs`.
- Source integrity verified unchanged: `sha256(packages/tree-select/src/index.ts) = 0fd597d86049de3cfc9f75d2a624e8d48dd282ff8eabe539c381890cb3b37514`; `sha256(packages/state-utils/src/create-selector/index.ts) = 2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02`.
- `git status --porcelain` was clean throughout the investigation except for this new document.

### L.7 Coverage checklist

- **Q1** ✔ comparison mechanism — referential `WeakMap` tree + leaf `Map` (§B).
- **Q2** ✔ counts 1 / 1(×1000) / 2 / 2, two-run stable (§C).
- **Q3** ✔ per-argument coexistence, `id1,id2,id1` → 2 computations (§D).
- **Q4** ✔ `clearCache()` recreates root `WeakMap` (§E).
- **Q5** ✔ `null`, `undefined` memoized; number (incl. `0`), boolean (incl. `false`), string (incl. `''`) dependents throw; arg-vs-dependent nuance (§F).
- **Q6** ✔ object-arg guard throw + `getCacheKey` dedupe + real consumer serialization (§G).
- **Root cause** ✔ in-place mutation → stale; immutable update → fresh (§H).
- **Contrast** ✔ `treeSelect` vs `createSelector`, with provenance + warn nuance labeled (§I).
- **README discrepancy** ✔ flagged, not fixed (§J).
- **Dev vs production** ✔ arg guards dev-only; dependent `TypeError` unconditional (§K).
