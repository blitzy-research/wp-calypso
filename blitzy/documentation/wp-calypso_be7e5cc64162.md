# Caching / Memoization Behavior of Calypso's Memoized Selectors — Investigation & Answer

> **Question being answered (verbatim):** _"I have a component that retrieves filtered data from a central store, and I'm seeing stale results being returned in certain conditions even after the underlying data has changed."_ Plus six specific follow-up questions (Q1–Q6, below).

This is a **run-first, read-only investigation**. Every behavioral claim below was produced by executing the **real** utility source through its canonical public API and capturing the complete, unedited output; no source file was modified. Each claim carries the exact command, the complete unedited output, and a full-path `file:line` reference. **Observed** facts (from my runs) are separated from **Inferred** facts (reasoned from the source).

All code fences are labelled: `sh` for commands, `text` for captured program/test output, and `ts`/`js` for source excerpts.

---

## 0. Investigation metadata & methodology

| Item                         | Value                                                                                                       |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Repository commit            | `be7e5cc641622d153040491fd5625c6cb83e12eb`                                                                  |
| Node.js runtime              | `v22.23.1` (satisfies `engines.node: "^v22.9.0"` and `.nvmrc: 22.9.0`)                                      |
| Package manager              | `yarn@4.0.2` (Corepack)                                                                                     |
| Primary utility              | `@automattic/tree-select` v2.0.0 `[packages/tree-select/package.json:L2-L3]`                                |
| Secondary utility (contrast) | `@automattic/state-utils` → `createSelector` `[packages/state-utils/src/create-selector/index.ts:L74-L113]` |
| Harness location             | `/tmp/obs` (outside the repository; removed after capture)                                                  |
| Deliverable                  | this file only — repository left byte-for-byte unchanged otherwise                                          |

**Runtime-version resolution (per AAP §0.8.2).** The environment setup notes mention installing Node 20.x, but the repository's `engines.node` requires `^v22.9.0` and `.nvmrc` pins `22.9.0`. Node 20 would violate the engine constraint, so the **canonical supported runtime, Node 22.9.0+ (installed `v22.23.1`)**, was used for all observations. This lets the utility run exactly as it does in CI (`cimg/node:22.9.0`). Exact captured versions are in the **Evidence Appendix (§L.5)**.

**Command conventions.** Every command below was run with the working directory at the repository root and the environment variable `REPO` exported to that same absolute path, i.e. `REPO="$PWD" = /tmp/blitzy/wp-calypso/blitzy-5e68b844-58da-44ac-a571-c83312f22855_d114d9`. Observation scripts import the real source with a dynamic import so they are portable and re-runnable:

```js
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;
```

**Why a standalone harness is canonical.** `treeSelect` is a pure, self-contained module whose only runtime dependency (`tslib`) is not exercised by the memoization logic `[packages/tree-select/package.json:L36-L38]`. Node 22 strips TypeScript types natively, so the **real** `packages/tree-select/src/index.ts` was imported directly with no build step, via:

```sh
REPO="$PWD" node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/<script>.mjs
```

A benign `MODULE_TYPELESS_PACKAGE_JSON` warning is emitted by Node when a `.ts` file is imported directly; the complete, unedited warning is reproduced in **§L.5**. It is a runtime artifact of type-stripping, **not** a code issue, and the repo `package.json` was **not** edited to silence it. By default `process.env.NODE_ENV` is `undefined`, so the utility's **development-mode guards are active** unless a scenario explicitly sets `NODE_ENV=production`.

**Two-run stability.** Every count-bearing scenario is driven by a self-contained script that executes the scenario **twice in-process** and prints a labelled `===== RUN 1 =====` / `===== RUN 2 =====` block; the `createSelector` contrast is additionally invoked as two separate processes (§L.6). Only values **identical across both runs** are reported. The at-scale scenario (Q2 B) issues **1000** identical calls. All commands and complete unedited outputs appear both inline (per question) and in the **Evidence Appendix (§L)**.

**Workspace integrity (test cache redirected out of the repository).** The committed Jest preset fixes its cache directory _inside_ the repository — `cacheDirectory: path.join( __dirname, '../../.cache/jest' )` `[test/packages/jest-preset.js:L10]`. Running Jest therefore writes generated, git-ignored files under `.cache/jest` (matched by `.gitignore:L15` `/.cache/`), which an ordinary `git status` cannot reveal. To keep the repository byte-clean, **every Jest command below overrides the cache with `--cacheDirectory=/tmp/obs/jest-cache`** (outside the repository). The ignored-artifact inventory proving `.cache/` is absent is in **§L.7**.

---

## 1. Executive summary (utility identification + root cause first)

**The utility.** The user's "central store" is Calypso's Redux global application state. For this legacy `client/state` selector path, Redux is the store described in the data-architecture doc — the "Third Era: Redux Global State Tree" and the "Current Recommendations" that all `client/state` selectors follow `[docs/our-approach-to-data.md:L43-L50,L85-L96]`. (That same document opens with an out-of-date notice recommending `@tanstack/react-query` for _new_ data needs `[docs/our-approach-to-data.md:L3]`; the utility under investigation is part of the pre-existing `client/state` Redux selector layer, not the newer React Query path.) The helper that memoizes "filtered data" derived from that store is **`treeSelect`** from `@automattic/tree-select` `[packages/tree-select/src/index.ts:L42-L102]`. Ten modules under `client/state/**` consume it — six in the comments selector tree, plus the invites, stats, and reader selector trees `[client/state/comments/selectors/get-hidden-comments-for-post.js]`, `[client/state/invites/selectors.js]`, `[client/state/stats/lists/selectors.js]`, `[client/state/reader/posts/selectors.js]`, `[client/state/reader/streams/selectors/get-reader-stream-transformed-items.ts]`; the representative consumer `getSiteStatsNormalizedData` is a textbook example `[client/state/stats/lists/selectors.js:L139-L159]`.

**The root cause of "stale results even after the underlying data has changed" (details in §H).** `treeSelect` decides cache hits by the **referential identity (`===`)** of the values your `getDependents` returns — _not_ by deep equality `[packages/tree-select/src/index.ts:L84-L89]`, `[packages/tree-select/README.md:L40]`. If your component **mutates data in place** (e.g., `state.posts.id1.status = 'draft'`) without producing a **new reference** for the depended-upon slice, the dependent reference is unchanged, so the selector reports a **cache hit and returns the previously cached (stale) value**. I reproduced this exactly: the underlying selector was **not** re-invoked and the stale array was returned (`before === after`). The fix pattern (immutable update → new reference) busts the cache and returns fresh data.

**The six questions, answered in one line each (full evidence follows):**

| #   | Question                                        | Answer (observed)                                                                                                                                                                                                                                                                                                       |
| --- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Q1  | Precise comparison mechanism                    | Two layers: a tree of `WeakMap`s compares each **dependent object by reference identity**; the terminal leaf `Map` compares the **generated string key** (default `args.join()`) by value (`Map` SameValueZero). **Not** deep equality. `[packages/tree-select/src/index.ts:L11-L12,L84-L89,L116-L131]`                 |
| Q2  | Underlying selector invocation counts           | same-state ×2 → **1**; same-state ×1000 → **1**; changed dependent → **2**; two distinct args → **2** `[packages/tree-select/test/index.js:L33-L46,L126-L159]`                                                                                                                                                          |
| Q3  | Separate entries per argument, or invalidation? | **Multi-entry per distinct _generated_ key**: distinct generated keys create coexisting leaf entries; a new key does **not** evict a prior one. But the default `args.join()` can **collide** (e.g. `('a,b')` vs `('a','b')`), returning the first cached result. `[packages/tree-select/src/index.ts:L86-L93,L11-L12]` |
| Q4  | Programmatic full clear?                        | Yes — the returned selector exposes `clearCache()`, which recreates the root `WeakMap` `[packages/tree-select/src/index.ts:L96-L99]`                                                                                                                                                                                    |
| Q5  | Nullish vs primitive dependents                 | `null` and `undefined` are memoized via a shared `NULLISH_KEY` sentinel; non-nullish primitives (number incl. `0`, boolean incl. `false`, string incl. `''`) **throw** `TypeError` `[packages/tree-select/src/index.ts:L107,L116-L120]`                                                                                 |
| Q6  | Customize cache keys for complex objects?       | Yes — pass `options.getCacheKey`. Without it, an object argument **throws** in dev; with it, objects dedupe by the returned key `[packages/tree-select/src/index.ts:L24-L27,L67,L75-L78,L86]`                                                                                                                           |

---

## Section A — Utility identification & overview

**Observed / Inferred.** `treeSelect` is a factory that returns a cached selector `[packages/tree-select/src/index.ts:L42-L102]`. Its documented purpose is to cache state-derived results "for use with the Redux global application state … whenever either the computation over state or React's rerenders are expensive" `[packages/tree-select/README.md:L3]`. This maps directly to the user's "component that retrieves filtered data from a central store": for this legacy `client/state` path the "central store" is Redux `[docs/our-approach-to-data.md:L43-L50,L85-L96]`, and the memoized selector sits between the store and the component.

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

Other consumers follow the same order, e.g. `getHiddenCommentsForPost` `[client/state/comments/selectors/get-hidden-comments-for-post.js:L8-L18]`, `getPostCommentsTree` `[client/state/comments/selectors/get-post-comments-tree.js:L16-L61]`, and `getInviteForSite` `[client/state/invites/selectors.js:L80-L96]`.

Where behavior differs, this document contrasts `treeSelect` with `createSelector` from `@automattic/state-utils` `[packages/state-utils/src/create-selector/index.ts:L74-L113]` (full contrast in §I).

---

## Section B — Q1: the precise comparison mechanism (cache hit vs miss)

**Mechanism (Inferred from source, then Observed at runtime).** `treeSelect` compares in **two distinct layers**, and a cache **HIT requires both**:

1. **Dependent objects — reference identity (`WeakMap`/`Map` object-key lookup).** On each call the dependents array is folded into a tree: `const leafCache = dependents.reduce( insertDependentKey, cache );` `[packages/tree-select/src/index.ts:L84]`. Each dependent value becomes a **key** in a map node `[packages/tree-select/src/index.ts:L116-L131]`. Interior nodes are `WeakMap`s and the root cache is a `WeakMap` `[packages/tree-select/src/index.ts:L65,L128]`. Because map lookups on **object** keys use **reference identity**, a dependent that is a _different object but deeply equal_ walks to a **different** node — a miss. The README states this: change is detected when a piece of state is "no longer referentially equal to its previous state (as opposed to a deep equality check)" `[packages/tree-select/README.md:L40]`.
2. **Arguments — generated string key compared by value (terminal `Map`).** The **last** node is a plain `Map` (not a `WeakMap`), because its key is the **string** produced by `getCacheKey(...args)` (default `args.join()`) `[packages/tree-select/src/index.ts:L11-L12,L128]`. The leaf lookup is `const key = getCacheKey( ...args ); if ( leafCache.has( key ) ) return leafCache.get( key );` `[packages/tree-select/src/index.ts:L86-L89]`. A `Map` compares keys with **SameValueZero**, which for the generated **strings** is **value equality** — _not_ object reference identity. So two different calls whose `getCacheKey(...args)` produce the _same string_ hit the same leaf entry (see the collision discussion in §D). Otherwise the selector recomputes and stores `leafCache.set( key, value )` `[packages/tree-select/src/index.ts:L91-L93]`.

In short: **all dependent objects must be reference-identical (WeakMap identity path) _and_ the generated string key must be value-equal in the terminal `Map`.** These are different comparison semantics on different layers; the old shorthand "everything is compared by `===`" is imprecise for the terminal string key.

**Cache structure diagram:**

```text
        Root WeakMap
             |  key = dependents[0]  (OBJECT, by reference identity)
             v
          WeakMap
             |  key = dependents[1]  (OBJECT, by reference identity)
             v
        Leaf Map (plain Map)
             |  key = getCacheKey(...args)  (STRING, by value / SameValueZero)
             v
        Cached result
```

**Observed (runtime).** Two calls with the **same** `state.posts` reference and the same argument produce **one** computation and return the **identical** object; a **deep-equal but different** `state.posts` reference forces a **second** computation and a **different** result object (though equal by value). This proves the dependent layer is identity-based, not deep-equality.

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q1_comparison.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
same-ref: calls = 1 | r1===r2 (identity) = true
deep-equal-diff-ref: calls = 2 | r1===r3 = false
JSON r1 == JSON r3 (values equal) = true
===== RUN 2 =====
same-ref: calls = 1 | r1===r2 (identity) = true
deep-equal-diff-ref: calls = 2 | r1===r3 = false
JSON r1 == JSON r3 (values equal) = true
```

- `same-ref … calls = 1` → same dependent reference + same arg = cache HIT (selector ran once) `[packages/tree-select/src/index.ts:L86-L89]`.
- `deep-equal-diff-ref … calls = 2` and `r1===r3 = false` → a new (deep-equal) dependent reference is a MISS `[packages/tree-select/src/index.ts:L84]`.
- `JSON … values equal = true` → the two results are value-equal, confirming the miss is purely reference-driven on the dependent layer, not value-driven.

**Contrast (Observed, §I).** `createSelector` compares differently: it snapshots the dependants and clears its cache when a **shallow** (`@wordpress/is-shallow-equal`) comparison against the previous snapshot fails `[packages/state-utils/src/create-selector/index.ts:L103-L104]`, using lodash `memoize` for the per-argument cache `[packages/state-utils/src/create-selector/index.ts:L90]`.

**Rationale.** Reference-equality on the dependent layer (not deep equality) is precisely why in-place mutation yields stale results — see §H.

---

## Section C — Q2: how many times the underlying selector runs (with concrete numbers)

**Run scale & stability (Observed).** A call counter wraps the underlying `selector`. The script runs the whole set **twice in-process** (`RUN 1`/`RUN 2`); the counts were **identical across both runs**. The at-scale scenario issues **1000** identical calls.

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q2_counts.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
A same-state x2 -> calls = 1
B same-state x1000 -> calls = 1
C changed-dependent (spread) -> calls = 2
D distinct-args -> calls = 2
===== RUN 2 =====
A same-state x2 -> calls = 1
B same-state x1000 -> calls = 1
C changed-dependent (spread) -> calls = 2
D distinct-args -> calls = 2
```

| Scenario | Sequence                                             | Underlying selector calls | Why                                                                                                    |
| -------- | ---------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------ |
| A        | same `state`, same arg, ×2                           | **1**                     | second call is a cache HIT `[packages/tree-select/src/index.ts:L87-L89]`                               |
| B        | same `state`, same arg, ×1000                        | **1**                     | memoization holds at scale; only the first call computes `[packages/tree-select/src/index.ts:L87-L93]` |
| C        | changed dependent via immutable spread `{ ...post }` | **2**                     | new reference for the dependent = MISS `[packages/tree-select/src/index.ts:L84]`                       |
| D        | two distinct arguments                               | **2**                     | distinct leaf keys `args.join()` = two computations `[packages/tree-select/src/index.ts:L86]`          |

**Cross-reference to the committed test suite (canonical "test runs").** These numbers match the repository's own assertions:

- "should cache the result of a selector function" asserts `selector.mock.calls` has length **1** after two identical calls `[packages/tree-select/test/index.js:L33-L46]`.
- "should call selector when making non-cached calls" asserts length **2** for two distinct arguments `[packages/tree-select/test/index.js:L126-L140]`.
- "should bust the cache when watched state changes" asserts length **2** after an immutable spread copy `[packages/tree-select/test/index.js:L142-L159]`.

Running the committed suite (complete unedited output in **§L.2**) confirms **17 passed, 17 total**.

> **Reconciliation (Observed vs the plan's stated figure).** The investigation plan described the suite as "15 tests"; the **observed** committed suite at this commit contains **17 tests** (all passing) `[packages/tree-select/test/index.js:L21-L264]`. Per the run-first mandate, the observed value (17) is reported. The count-bearing assertions cited above are unaffected.

---

## Section D — Q3: does the cache keep separate entries per argument, or invalidate?

**Answer (Observed): multi-entry, keyed by the _generated_ cache key — not by the raw arguments.** Distinct **generated keys** create **separate, coexisting** leaf entries; calling the selector with a new generated key does **not** invalidate a prior entry, as long as the dependents for that prior entry remain referentially stable. The leaf is a plain `Map` that stores one entry per distinct `getCacheKey(...args)` and only ever grows via `leafCache.set(...)` `[packages/tree-select/src/index.ts:L86-L93]`; there is no eviction of sibling keys.

**Critical caveat — the default key can collide (Observed).** The default key function is `args.join()` `[packages/tree-select/src/index.ts:L11-L12]`. Coexistence is therefore guaranteed **only for arguments that produce _distinct generated strings_**. When two semantically different calls stringify to the **same** join, they share one entry and the first cached result is returned for both. This is the mechanism a component must understand to avoid _incorrect_ cache hits.

**Observed (runtime).** Three demonstrations run twice in-process: (1) coexistence with `id1, id2, id1`; (2) the join collision `('a,b')` vs `('a','b')`; (3) numeric/string `1` vs `'1'`; and the nullish/empty collision `null` vs `undefined` vs `''` (all join to `""`).

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q3_coexistence.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
coexistence: calls = 2 (3 calls issued: id1,id2,id1)
first id1 result survived: a === c = true
id2 entry distinct: a === b = false
COLLISION join: "a,b" vs "a","b" -> calls = 1 | same result = true
COLLISION numeric/string: 1 vs "1" -> calls = 1 | same result = true
COLLISION nullish/empty: null vs undefined vs "" -> calls = 1 | all same result = true
===== RUN 2 =====
coexistence: calls = 2 (3 calls issued: id1,id2,id1)
first id1 result survived: a === c = true
id2 entry distinct: a === b = false
COLLISION join: "a,b" vs "a","b" -> calls = 1 | same result = true
COLLISION numeric/string: 1 vs "1" -> calls = 1 | same result = true
COLLISION nullish/empty: null vs undefined vs "" -> calls = 1 | all same result = true
```

- `coexistence: calls = 2` for **3** issued calls, `a === c = true`, `a === b = false` → the intervening `id2` call did **not** evict the `id1` entry; the first `id1` result object is returned again on the third call (distinct keys coexist).
- `COLLISION join: "a,b" vs "a","b" -> calls = 1 | same result = true` → a single argument `'a,b'` and two arguments `'a','b'` both produce `args.join() === "a,b"`, so the second call is served the **first** cached result although the argument shapes differ.
- `COLLISION numeric/string: 1 vs "1" -> calls = 1 | same result = true` → `1` and `'1'` both join to `"1"`.
- `COLLISION nullish/empty: null vs undefined vs "" -> calls = 1 | all same result = true` → `null`, `undefined`, and `''` all join to `""`.

**Consequence / guidance.** "Separate entry per argument" is true only when the **generated key uniquely encodes every argument dimension that affects the result**. With the default `args.join()`, arguments whose string joins coincide will share a cache entry and can return a stale/incorrect result. To keep distinct calls distinct, supply an `options.getCacheKey` that unambiguously serializes each dimension (see §G) — exactly what `getSiteStatsNormalizedData` does by delimiting `siteId`, `statType`, and a serialized `query` `[client/state/stats/lists/selectors.js:L156-L157]`.

**Cross-reference.** The coexistence result mirrors the committed test "should maintain the cache for unique dependents simultaneously", which asserts exactly **2** computations for the `id1,id2,id1` sequence `[packages/tree-select/test/index.js:L161-L186]`. The default-key-collision risk is the same reason the object-argument guard exists (§G) `[packages/tree-select/src/index.ts:L75-L79]` and is documented for the sibling utility in `[packages/state-utils/src/create-selector/README.md:L48]`.

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

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q4_clearcache.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
before clearCache: calls = 1 | memoizedResult === firstResult = true
after clearCache:  calls = 2 | afterClearResult === firstResult = false
value still equal after clear: JSON equal = true
===== RUN 2 =====
before clearCache: calls = 1 | memoizedResult === firstResult = true
after clearCache:  calls = 2 | afterClearResult === firstResult = false
value still equal after clear: JSON equal = true
```

- `before … calls = 1 … === firstResult = true` → the second call was a cache HIT.
- `after … calls = 2 … === firstResult = false` → `clearCache()` forced recomputation; the new result is a different object reference `[packages/tree-select/src/index.ts:L96-L99]`.
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

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q5_nullish_primitive.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
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
===== RUN 2 =====
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
- **number** → `1` throws; `0` (a falsy number) **also** throws — note `0` is _not_ treated as nullish here, because the throw check at `[packages/tree-select/src/index.ts:L118]` runs **before** the `|| NULLISH_KEY` fallback at `[packages/tree-select/src/index.ts:L121]`.
- **boolean** → `true` throws; `false` (a falsy boolean) **also** throws, for the same ordering reason.
- string (incl. empty `''`) → throws as well, shown for completeness.

**Cross-reference.** The committed tests assert this precisely: "should memoize a nullish value returned by getDependents" returns identical results for `[ null, undefined ]` `[packages/tree-select/test/index.js:L219-L229]`; "throws on a non-nullish primitive value returned by getDependents" iterates exactly `[ true, 1, 'a', false, '', 0 ]` and expects each to throw `[packages/tree-select/test/index.js:L231-L241]`.

**CRITICAL NUANCE — do not conflate dependents with arguments (Observed).** Q5 is about values `getDependents` **returns**. Primitives passed as **arguments** to the selector are an entirely different code path: they are joined into the cache key via `getCacheKey(...args)` and do **not** throw. The output above shows arguments `[ 1, '', 'foo', true, null, undefined ]` **all** succeeding (`threw = false`), matching the committed test "should not throw an error in development when given primitives" `[packages/tree-select/test/index.js:L115-L124]`. Keep the two cases separate: **primitive dependents throw; primitive arguments are fine.**

**Environment note (forward-reference to §K).** The primitive-**dependent** `TypeError` at `[packages/tree-select/src/index.ts:L118-L120]` is **unconditional** — it is _not_ wrapped in a `NODE_ENV` guard and therefore throws in **both** development and production.

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

The rationale: the default key is `args.join()`, and objects stringify to `"[object Object]"`, which would collide across distinct objects and produce incorrect cache hits (the general collision hazard from §D).

**With `options.getCacheKey`, object arguments are enabled and deduplication is controlled by the returned key.**

**Observed (runtime).**

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/q6_cachekey.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
object-arg WITHOUT getCacheKey: threw = true | Error: Do not pass objects as arguments to a treeSelector
with getCacheKey, two same-siteId objects: calls = 1 | firstResult === secondResult = true
result value = ["id1","id2"]
then distinct siteId: calls = 2 | thirdResult === firstResult = false
===== RUN 2 =====
object-arg WITHOUT getCacheKey: threw = true | Error: Do not pass objects as arguments to a treeSelector
with getCacheKey, two same-siteId objects: calls = 1 | firstResult === secondResult = true
result value = ["id1","id2"]
then distinct siteId: calls = 2 | thirdResult === firstResult = false
```

- Without a custom key, `getSitePosts(state, { siteId: 'site1' })` throws `Error: Do not pass objects as arguments to a treeSelector` `[packages/tree-select/src/index.ts:L75-L78]`.
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

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/rootcause.mjs
```

Complete unedited output (both runs identical):

```text
===== RUN 1 =====
IN-PLACE: initial result = ["id1"] | calls = 1
IN-PLACE: after mutation = ["id1"] | calls = 1 | correct would be []
IN-PLACE: same cached array returned: before === after = true
IN-PLACE: STALE (still shows id1) = true
IMMUTABLE: initial result = ["id1"] | calls = 1
IMMUTABLE: after update = [] | calls = 2
IMMUTABLE: FRESH (now []) = true
===== RUN 2 =====
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

**Definitive statement.** The stale-results symptom is caused by **mutating data in place without creating a new reference** for the slice returned by `getDependents`. Because `treeSelect` uses referential-identity comparison on the dependent layer (not deep equality), such a mutation is invisible to the cache and the stale value is served. Redux's own guidance — never mutate state in place, always produce new references — is what keeps `treeSelect` correct; the README notes this dependency on referential inequality directly `[packages/tree-select/README.md:L40]`. _(Per the read-only constraint and the task scope, this document explains and reproduces the behavior; it does not modify the utility or any consumer.)_

---

## Section I — `treeSelect` vs `createSelector` contrast

| Dimension                | `treeSelect` `[packages/tree-select/src/index.ts:L1-L131]`                                                                                                                                                                                                                                                                                                                                   | `createSelector` `[packages/state-utils/src/create-selector/index.ts:L1-L113]`                                                                                             |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Change detection         | Reference identity of each dependent object (`WeakMap` tree), plus value equality of the generated string key in the leaf `Map` `[packages/tree-select/src/index.ts:L84-L89,L116-L131]`                                                                                                                                                                                                      | **Shallow** equality of the dependants snapshot via `@wordpress/is-shallow-equal` `[packages/state-utils/src/create-selector/index.ts:L103-L104]`                          |
| Cache structure          | Tree of `WeakMap`s + leaf `Map` per-argument `[packages/tree-select/src/index.ts:L84,L128]`                                                                                                                                                                                                                                                                                                  | lodash `memoize` cache keyed by `args.join()` `[packages/state-utils/src/create-selector/index.ts:L90]`, README `[packages/state-utils/src/create-selector/README.md:L48]` |
| Multi-argument behavior  | **Multi-entry** per distinct generated key: entries coexist; a changed dependent **selects a different branch** rather than deleting the old one `[packages/tree-select/src/index.ts:L86-L93]`                                                                                                                                                                                               | **Wholesale clear**: any dependants-snapshot change clears the entire memoize cache `[packages/state-utils/src/create-selector/index.ts:L103-L104]`                        |
| Programmatic clear       | `selector.clearCache()` recreates root `WeakMap` `[packages/tree-select/src/index.ts:L96-L99]`                                                                                                                                                                                                                                                                                               | `selector.memoizedSelector.cache.clear()` `[packages/state-utils/src/create-selector/index.ts:L90,L111]`                                                                   |
| Complex object arguments | **Throws** in dev without `getCacheKey` `[packages/tree-select/src/index.ts:L75-L78]`                                                                                                                                                                                                                                                                                                        | **Warns** (does not throw) in dev `[packages/state-utils/src/create-selector/index.ts:L41-L52]`                                                                            |
| Invalidation / GC        | No active per-branch eviction: an old branch is **retained** and its cached object is still returned if you reuse the original dependent; branches keyed by dependents that become unreachable are **GC-eligible** because `WeakMap` keys are weak `[packages/tree-select/README.md:L4]`. Only `clearCache()` actively replaces the whole root `[packages/tree-select/src/index.ts:L96-L99]` | lodash `Map`-based cache; cleared wholesale on dependants-snapshot change `[packages/state-utils/src/create-selector/index.ts:L103-L104]`                                  |

**Correction on "invalidation" (Observed).** `treeSelect` does **not** invalidate a branch when a dependent changes. A changed dependent reference simply routes the tree walk down a **different** `WeakMap` path, creating new nodes; the original branch is **left intact**. The Q3 observation proves retention directly: after calling with `id1`, then `id2`, then `id1` again, the third call returned the **original** `id1` result object (`a === c = true`) — the `id1` branch was retained, not evicted (§D). Old branches whose dependent objects are no longer referenced anywhere become eligible for garbage collection precisely because they are held via `WeakMap` keys `[packages/tree-select/README.md:L4]`; the only _active_ whole-cache reset is `clearCache()` `[packages/tree-select/src/index.ts:L96-L99]`.

**Observed — wholesale-clear contrast.** Using the real `createSelector` logic (provenance labelled below), I drove two distinct arguments under a **stable** dependants snapshot, then changed the dependants reference. The scenario is run **twice in-process** (`RUN 1`/`RUN 2`) and the whole command is **invoked twice as separate processes** (both invocations shown in **§L.6**). First invocation:

```sh
CS_BUNDLE=/tmp/obs/cs/create-selector.bundle.mjs node --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/cs/createselector_contrast.mjs
```

Complete unedited output (invocation 1; invocation 2 in §L.6 is identical):

```text
===== RUN 1 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== RUN 2 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== WARN (complex object argument) =====
complex-arg: threw = false | warnCount = 1 | message = "Do not pass complex objects as arguments for a memoized selector"
```

- Under a stable dependants snapshot, `argA` and `argB` **coexist** (calls `1 → 2 → 2`), similar to `treeSelect` (§D).
- When the dependants reference changes (shallow-unequal), **both** argument entries are wiped and recomputed (calls `3`, then `4`) — the **wholesale** invalidation `[packages/state-utils/src/create-selector/index.ts:L103-L104]`. This is the key behavioral difference from `treeSelect`'s per-branch **retention** (§D).

**PROVENANCE LABEL (mandatory — NON-CANONICAL bundled path).** `createSelector` imports `import { memoize } from 'lodash'`, `import isShallowEqual from '@wordpress/is-shallow-equal'`, and `import warn from '@wordpress/warning'` `[packages/state-utils/src/create-selector/index.ts:L1-L3]`. Node's raw ESM loader **cannot** consume the CJS **named** export `memoize` from `lodash` (it fails with `SyntaxError: Named export 'memoize' not found`), which the repo's build normally handles. So — unlike the fully-canonical `treeSelect` runs — the `createSelector` values come from a **bundled** path: the **real in-repo source** was bundled read-only with `esbuild`, which resolves the CJS named import exactly like the repo bundler. The complete, reproducible provenance (source hash, byte-identical copy, exact `esbuild` command and flags, bundle hash, dependency resolution from the installed tree **and** the lockfile, and both process invocations) is in **§L.6**. Key facts:

- Real source hash: `sha256(packages/state-utils/src/create-selector/index.ts) = 2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02` (unchanged before/after bundling; `git status` stayed clean).
- Bundle produced by `esbuild@0.25.1`; bundle `sha256 = e228332500b29580272e454804801512da5858ec00d0b1c4b1f2e1ed6d317305`.
- Dependencies resolved against the repository's **installed** tree, whose **exact locked** versions (from `yarn.lock`, distinct from the caret ranges in the manifest) are:
  - `lodash` → manifest range `^4.17.21` `[packages/state-utils/package.json:L35]`; **locked** `4.17.21` `[yarn.lock:L24074-L24078]`.
  - `@wordpress/is-shallow-equal` → manifest range `^5.21.0` `[packages/state-utils/package.json:L33]`; **locked** `5.21.0` `[yarn.lock:L10683-L10690]`.
  - `@wordpress/warning` → manifest range `^3.21.0` `[packages/state-utils/package.json:L34]`; **locked** `3.21.0` `[yarn.lock:L11095-L11100]`.

**Warn behavior (version-sensitive — labelled).** The complex-argument path **warns, it does not throw** `[packages/state-utils/src/create-selector/index.ts:L41-L52]`. Observed: `threw = false`, message `Do not pass complex objects as arguments for a memoized selector`, and the selector still returns a value. The installed `@wordpress/warning@3.21.0` gates its `console.warn` on `globalThis.SCRIPT_DEBUG === true` (**not** `NODE_ENV`) and dedupes by message (read from the installed `node_modules/@wordpress/warning/build/index.js`). Hence a standalone run shows `warnCount = 1` **only** when `SCRIPT_DEBUG` is set (as it was here) and only once per distinct message (process-global dedupe). **The canonical warn-count assertion is the committed test** — it mocks `warn` (bypassing the gate/dedupe) and asserts it is called **3** times for the three complex-argument inputs `[packages/state-utils/src/create-selector/test/index.js:L65-L78]`. Reported as: warn **observed firing** (dev flag on, count 1 per message) **and** committed-test-canonical (3); the `SCRIPT_DEBUG`/version gate is **Inferred** from the installed package source.

**Repository-grounded framing (no external dependency added).** The sibling utility's own README frames `createSelector` as achieving a goal "similar" to `reselect`, calling argument support "a key differentiator from `reselect`" `[packages/state-utils/src/create-selector/README.md:L46]`. That repository-grounded reference is the only ecosystem comparison retained here; no unsourced external memoizer claims are made, and no dependency is added.

---

## Section J — Documentation discrepancy (flagged, NOT fixed)

**Observed.** The `packages/tree-select/README.md` usage examples show the arguments in the **reversed** order — `treeSelect( selector, getDependents )`:

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

```sh
REPO=$PWD node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/devprod.mjs
```

```text
===== RUN 1 =====
NODE_ENV = undefined
treeSelect(undefined, undefined) at creation: threw = true | treeSelect: invalid arguments passed, selector and getDependents must both be functions
object arg {} to selector: threw = true | Do not pass objects as arguments to a treeSelector
primitive dependent (number 1): threw = true | TypeError: key must be an object, `null`, or `undefined`
===== RUN 2 =====
NODE_ENV = undefined
treeSelect(undefined, undefined) at creation: threw = true | treeSelect: invalid arguments passed, selector and getDependents must both be functions
object arg {} to selector: threw = true | Do not pass objects as arguments to a treeSelector
primitive dependent (number 1): threw = true | TypeError: key must be an object, `null`, or `undefined`
```

**Observed — production mode (`NODE_ENV=production`):**

```sh
REPO=$PWD NODE_ENV=production node --experimental-strip-types --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/devprod.mjs
```

```text
===== RUN 1 =====
NODE_ENV = "production"
treeSelect(undefined, undefined) at creation: threw = false
object arg {} to selector: threw = false
primitive dependent (number 1): threw = true | TypeError: key must be an object, `null`, or `undefined`
===== RUN 2 =====
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

| Error text                                                                                | Guard location                                  | Throws in dev? | Throws in prod?         |
| ----------------------------------------------------------------------------------------- | ----------------------------------------------- | -------------- | ----------------------- |
| `treeSelect: invalid arguments passed, selector and getDependents must both be functions` | `[packages/tree-select/src/index.ts:L57-L63]`   | Yes            | **No**                  |
| `Do not pass objects as arguments to a treeSelector`                                      | `[packages/tree-select/src/index.ts:L75-L78]`   | Yes            | **No**                  |
| `` TypeError: key must be an object, `null`, or `undefined` ``                            | `[packages/tree-select/src/index.ts:L118-L120]` | Yes            | **Yes (unconditional)** |

---

## Section L — Evidence appendix

### L.1 Q1–Q6 + root cause → command → observed result → source citation

| Item                            | Command (harness in `/tmp/obs`)                | Observed key result                                                                    | Source citation                                                                                                  |
| ------------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Q1 comparison                   | `node … q1_comparison.mjs`                     | same-ref calls=1 (hit); deep-equal diff-ref calls=2 (miss), values equal               | `[packages/tree-select/src/index.ts:L84-L89,L116-L131]`, `[packages/tree-select/README.md:L40]`                  |
| Q2 counts                       | `node … q2_counts.mjs`                         | 1 / 1 (×1000) / 2 / 2                                                                  | `[packages/tree-select/test/index.js:L33-L46,L126-L159]`                                                         |
| Q3 coexistence + collisions     | `node … q3_coexistence.mjs`                    | 3 calls → 2 computations; first id1 survives; join/numeric/nullish collisions → 1 comp | `[packages/tree-select/src/index.ts:L86-L93,L11-L12]`, `[packages/tree-select/test/index.js:L161-L186]`          |
| Q4 clearCache                   | `node … q4_clearcache.mjs`                     | before: same ref; after clearCache: new ref, equal value                               | `[packages/tree-select/src/index.ts:L96-L99]`, `[packages/tree-select/test/index.js:L197-L217]`                  |
| Q5 nullish/primitive            | `node … q5_nullish_primitive.mjs`              | null+undefined memoized; [true,1,'a',false,'',0] throw; primitive args OK              | `[packages/tree-select/src/index.ts:L107,L116-L120]`, `[packages/tree-select/test/index.js:L115-L124,L219-L241]` |
| Q6 getCacheKey                  | `node … q6_cachekey.mjs`                       | object-arg no-key throws; with key, same-siteId dedupes (calls=1); distinct → 2        | `[packages/tree-select/src/index.ts:L24-L27,L67,L75-L78,L86]`, `[packages/tree-select/test/index.js:L243-L264]`  |
| Root cause                      | `node … rootcause.mjs`                         | in-place mutation → calls stays 1 (stale, before===after); immutable → calls 2 (fresh) | `[packages/tree-select/src/index.ts:L84-L89]`, `[packages/tree-select/test/index.js:L142-L159]`                  |
| Dev vs prod                     | `node … devprod.mjs` (± `NODE_ENV=production`) | arg guards dev-only; primitive-dependent throw unconditional                           | `[packages/tree-select/src/index.ts:L57-L63,L75-L79,L118-L120]`                                                  |
| createSelector contrast         | `node … cs/createselector_contrast.mjs`        | wholesale clear 1,2,2 → 3,4; warn (not throw), warnCount=1                             | `[packages/state-utils/src/create-selector/index.ts:L41-L52,L90,L103-L104]`                                      |
| Committed tree-select suite     | see §L.2                                       | 17 passed, 17 total                                                                    | `[packages/tree-select/test/index.js:L21-L264]`                                                                  |
| Committed create-selector suite | see §L.3                                       | 13 passed, 13 total                                                                    | `[packages/state-utils/src/create-selector/test/index.js:L7-L291]`                                               |

### L.2 Committed tree-select suite — complete unedited output (canonical "test runs" for Q2)

Cache redirected out of the repository via `--cacheDirectory=/tmp/obs/jest-cache` (§L.7). Complete combined stdout+stderr, unedited (including the pre-existing haste-map and Browserslist notices):

```sh
CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci --cacheDirectory=/tmp/obs/jest-cache "packages/tree-select"
```

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
PASS packages/tree-select/test/index.js
  index
    #treeSelect
      ✓ should create a function which returns the expected value when called (3 ms)
      ✓ should cache the result of a selector function
      ✓ should cache the result of a selector function that has multiple dependents (1 ms)
      ✓ should throw an error if getDependents is missing (9 ms)
      ✓ should throw an error if selector is missing (1 ms)
      ✓ should not throw an error in production for missing args
      ✓ should throw an error in development when given object arguments (2 ms)
      ✓ should not throw an error in production even when given object arguments (1 ms)
      ✓ should not throw an error in development when given primitives (1 ms)
      ✓ should call selector when making non-cached calls
      ✓ should bust the cache when watched state changes
      ✓ should maintain the cache for unique dependents simultaneously
      ✓ should call dependant state getter with dependents and arguments
      ✓ should bust the cache when clearCache() method is called
      ✓ should memoize a nullish value returned by getDependents
      ✓ throws on a non-nullish primitive value returned by getDependents (2 ms)
      ✓ accepts a getCacheKey option that enables object arguments (1 ms)

Test Suites: 1 passed, 1 total
Tests:       17 passed, 17 total
Snapshots:   0 total
Time:        0.871 s
Ran all test suites matching /packages\/tree-select/i.
```

> **Reproducibility caveat (incidental, non-asserted fields).** Every value asserted from the block above is stable across runs: `Test Suites: 1 passed, 1 total`, `Tests: 17 passed, 17 total`, all 17 test names, the `PASS packages/tree-select/test/index.js` line, and the wording of the `jest-haste-map` and `Browserslist` notices. A literal byte-for-byte re-run will differ only in incidental fields that are **not** part of any reported claim: the `Time:` value (observed `0.834 s` / `0.871 s`), the per-test `(N ms)` timings (which change value and appear or disappear between runs), the ordering — and occasional presence — of the two `jest-haste-map` duplicate-mock notices (a fresh run here showed the `dist/esm` pair before the `dist/cjs` pair; the block above shows the reverse), and the `Browserslist … N months old` age (time-sensitive; it increments as the pinned `caniuse-lite` ages). This mirrors the PID caveat in §L.5.

> **Note on the stderr notices (N3 provenance).** The `jest-haste-map: duplicate manual mock found: wpcom-proxy-request` notice arises because a **tracked** source mock `[packages/plans-grid-next/src/__mocks__/wpcom-proxy-request.js:L1-L13]` coexists with **ignored, generated** build copies under `dist/cjs` and `dist/esm` (matched by `.gitignore:L69` `/packages/*/dist/`, produced by the workspace `postinstall` build). The `dist/` copies are **not** committed files — only the `src/` mock is tracked. The `Browserslist: … caniuse-lite is 17 months old` notice is likewise pre-existing repo state. Neither affects the tree-select assertions.

### L.3 Committed create-selector suite — complete unedited output

```sh
CI=true node_modules/.bin/jest -c test/packages/jest.config.js --watchAll=false --ci --cacheDirectory=/tmp/obs/jest-cache "packages/state-utils/src/create-selector"
```

```text
jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/src/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/esm/__mocks__/wpcom-proxy-request.js

jest-haste-map: duplicate manual mock found: wpcom-proxy-request
  The following files share their name; please delete one of them:
    * <rootDir>/dist/esm/__mocks__/wpcom-proxy-request.js
    * <rootDir>/dist/cjs/__mocks__/wpcom-proxy-request.js

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS packages/state-utils/src/create-selector/test/index.js
  index
    ✓ should expose its memoized function (1 ms)
    ✓ should create a function which returns the expected value when called (2 ms)
    ✓ should cache the result of a selector function (1 ms)
    ✓ should warn against complex arguments in development mode
    ✓ should return the expected value of differing arguments (1 ms)
    ✓ should bust the cache when watched state changes (1 ms)
    ✓ should accept an array of dependent state values
    ✓ should accept an array of dependent selectors (1 ms)
    ✓ should default to watching entire state, returning cached result if no changes
    ✓ should default to watching entire state, busting if state has changed
    ✓ should accept an optional custom cache key generating function
    ✓ should call dependant state getter with arguments (1 ms)
    ✓ should handle an array of selectors instead of a dependant state getter

Test Suites: 1 passed, 1 total
Tests:       13 passed, 13 total
Snapshots:   0 total
Time:        0.884 s
Ran all test suites matching /packages\/state-utils\/src\/create-selector/i.
```

> **Reproducibility caveat (incidental, non-asserted fields).** Every value asserted from the block above is stable across runs: `Test Suites: 1 passed, 1 total`, `Tests: 13 passed, 13 total`, all 13 test names, the `PASS packages/state-utils/src/create-selector/test/index.js` line, and the wording of the `jest-haste-map` and `Browserslist` notices. A literal byte-for-byte re-run will differ only in incidental fields that are **not** part of any reported claim: the `Time:` value (observed `0.838 s` / `0.843 s`), the per-test `(N ms)` timings (which change value and appear or disappear between runs), the ordering — and occasional presence — of the two `jest-haste-map` duplicate-mock notices, and the `Browserslist … N months old` age (time-sensitive). This mirrors the PID caveat in §L.5.

### L.4 Observed vs Inferred

**Observed (captured directly from my runs):**

- Cache hits are reference-based on the dependent layer: same-ref hit / deep-equal-diff-ref miss (§B).
- Invocation counts 1 / 1(×1000) / 2 / 2 (§C), identical across both in-process runs.
- Multi-entry coexistence: 3 calls → 2 computations, first entry survives; default-key collisions (comma-join, numeric/string, nullish/empty) each collapse to 1 computation (§D).
- `clearCache()` forces a new result reference (§E).
- `null`/`undefined` dependents memoized; `[true,1,'a',false,'',0]` dependents throw the exact `TypeError`; primitive **arguments** do not throw (§F).
- Object argument without `getCacheKey` throws the exact `Error`; with `getCacheKey`, same-`siteId` objects dedupe (§G).
- In-place mutation → stale cached array (calls stays 1); immutable update → fresh (calls 2) (§H).
- `createSelector` wholesale-clear sequence 1,2,2 → 3,4; complex-arg warns (does not throw) (§I).
- Dev-only arg guards vs unconditional primitive-dependent throw (§K).
- Both committed suites pass (17 and 13 tests) (§L.2, §L.3).

**Inferred (reasoned from source, not directly executed as a standalone assertion):**

- Interior tree nodes are `WeakMap`s and only the leaf is a plain `Map` `[packages/tree-select/src/index.ts:L128]` (inferred from the code; the runtime behavior is consistent with it but the node types were not separately introspected).
- The terminal `Map` compares its generated string key by SameValueZero/value equality (standard `Map` semantics); the runtime collisions in §D are the Observed consequence.
- `0`/`false`/`''` dependents throw because the throw check `[packages/tree-select/src/index.ts:L118]` precedes the `|| NULLISH_KEY` fallback `[packages/tree-select/src/index.ts:L121]` (source-order reasoning; the throw itself is Observed).
- `@wordpress/warning@3.21.0` gates on `globalThis.SCRIPT_DEBUG` and dedupes by message (read from the installed package source; the resulting `warnCount=1` is Observed).
- The `WeakMap` GC-friendliness claim `[packages/tree-select/README.md:L4]` is a documented design property, not something exercised at runtime here.

### L.5 Runtime versions, source hashes, and the complete Node type-stripping warning

```sh
node --version
corepack yarn --version
node -e 'for (const p of ["lodash","@wordpress/is-shallow-equal","@wordpress/warning"]) console.log(p, require(p + "/package.json").version)'
sha256sum packages/tree-select/src/index.ts packages/state-utils/src/create-selector/index.ts
```

```text
v22.23.1
4.0.2
lodash 4.17.21
@wordpress/is-shallow-equal 5.21.0
@wordpress/warning 3.21.0
0fd597d86049de3cfc9f75d2a624e8d48dd282ff8eabe539c381890cb3b37514  packages/tree-select/src/index.ts
2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02  packages/state-utils/src/create-selector/index.ts
```

**Complete, unedited Node warning** emitted when the real `.ts` source is imported directly (captured WITHOUT the `--disable-warning` flag; the PID varies per run). This is a runtime type-stripping artifact, not a code defect; `packages/tree-select/package.json` was **not** modified to silence it:

```sh
REPO=$PWD node --experimental-strip-types /tmp/obs/q1_comparison.mjs
```

```text
===== RUN 1 =====
same-ref: calls = 1 | r1===r2 (identity) = true
deep-equal-diff-ref: calls = 2 | r1===r3 = false
JSON r1 == JSON r3 (values equal) = true
===== RUN 2 =====
same-ref: calls = 1 | r1===r2 (identity) = true
deep-equal-diff-ref: calls = 2 | r1===r3 = false
JSON r1 == JSON r3 (values equal) = true
(node:51867) [MODULE_TYPELESS_PACKAGE_JSON] Warning: Module type of file:///tmp/blitzy/wp-calypso/blitzy-5e68b844-58da-44ac-a571-c83312f22855_d114d9/packages/tree-select/src/index.ts is not specified and it doesn't parse as CommonJS.
Reparsing as ES module because module syntax was detected. This incurs a performance overhead.
To eliminate this warning, add "type": "module" to /tmp/blitzy/wp-calypso/blitzy-5e68b844-58da-44ac-a571-c83312f22855_d114d9/packages/tree-select/package.json.
(Use `node --trace-warnings ...` to show where the warning was created)
```

### L.6 createSelector provenance & both process invocations (NON-CANONICAL bundled path)

Complete, reproducible provenance for the `createSelector` contrast (§I). The bundle input is the **real in-repo source**; `esbuild` reads it (and the repo's installed dependencies) read-only and writes output to `/tmp/obs/cs`. The source hash is unchanged before/after and `git status` stays clean:

```sh
# 1) byte-identical copy + hashes of source and copy
cp packages/state-utils/src/create-selector/index.ts /tmp/obs/cs/create-selector.source.ts
sha256sum packages/state-utils/src/create-selector/index.ts /tmp/obs/cs/create-selector.source.ts
# 2) esbuild bundle of the REAL in-repo source (read-only), output to /tmp
node_modules/.bin/esbuild --version
node_modules/.bin/esbuild packages/state-utils/src/create-selector/index.ts \
  --bundle --platform=node --format=esm \
  --outfile=/tmp/obs/cs/create-selector.bundle.mjs
# 3) bundle hash
sha256sum /tmp/obs/cs/create-selector.bundle.mjs
# 4) installed dependency versions (resolved tree)
node -e "for (const p of ['lodash','@wordpress/is-shallow-equal','@wordpress/warning']) console.log(p+' '+require(p+'/package.json').version)"
# 5) locked versions (yarn.lock)
sed -n '24074,24076p;10683,10686p;11095,11097p' yarn.lock
# 6) confirm source unchanged after bundling
git status --porcelain packages/state-utils/src/create-selector/index.ts; echo "(porcelain end)"
```

```text
2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02  packages/state-utils/src/create-selector/index.ts
2c351c697acb433e47c01799710c0c48018cd158c7c34ac6cd154597d5ae3c02  /tmp/obs/cs/create-selector.source.ts
0.25.1

  ../../../obs/cs/create-selector.bundle.mjs  226.5kb

⚡ Done in 22ms
e228332500b29580272e454804801512da5858ec00d0b1c4b1f2e1ed6d317305  /tmp/obs/cs/create-selector.bundle.mjs
lodash 4.17.21
@wordpress/is-shallow-equal 5.21.0
@wordpress/warning 3.21.0
"@wordpress/is-shallow-equal@npm:^5.21.0":
  version: 5.21.0
  resolution: "@wordpress/is-shallow-equal@npm:5.21.0"
  dependencies:
"@wordpress/warning@npm:3.21.0":
  version: 3.21.0
  resolution: "@wordpress/warning@npm:3.21.0"
"lodash@npm:^4.17.14, lodash@npm:^4.17.15, lodash@npm:^4.17.19, lodash@npm:^4.17.20, lodash@npm:^4.17.21, lodash@npm:^4.17.4, lodash@npm:~4.17.21":
  version: 4.17.21
  resolution: "lodash@npm:4.17.21"
(porcelain end)
```

> **Reproducibility caveat (incidental, non-asserted field).** Every value asserted from the provenance block above is stable across runs: the two source `sha256` hashes, the `esbuild` version (`0.25.1`), the bundle size (`226.5kb`), the bundle `sha256` (`e228332500b29580…`, verified byte-identical across repeated rebuilds), the resolved and locked dependency versions, and the empty `git status --porcelain`. A literal byte-for-byte re-run will differ only in esbuild's incidental `⚡ Done in Nms` timing (observed `21ms` / `22ms`), which is **not** part of any reported claim. This mirrors the PID caveat in §L.5.

**Both process invocations of the contrast (same command, fresh process each) — complete unedited output, identical:**

Invocation 1:

```sh
CS_BUNDLE=/tmp/obs/cs/create-selector.bundle.mjs node --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/cs/createselector_contrast.mjs
```

```text
===== RUN 1 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== RUN 2 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== WARN (complex object argument) =====
complex-arg: threw = false | warnCount = 1 | message = "Do not pass complex objects as arguments for a memoized selector"
```

Invocation 2:

```sh
CS_BUNDLE=/tmp/obs/cs/create-selector.bundle.mjs node --disable-warning=MODULE_TYPELESS_PACKAGE_JSON /tmp/obs/cs/createselector_contrast.mjs
```

```text
===== RUN 1 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== RUN 2 =====
argA (state1):          calls = 1
argB (state1):          calls = 2
argA again (state1):    calls = 2 (coexist -> stays 2)
argA (state2, new ref): calls = 3 (wholesale clear -> recompute)
argB (state2):          calls = 4 (B entry was wiped too)
===== WARN (complex object argument) =====
complex-arg: threw = false | warnCount = 1 | message = "Do not pass complex objects as arguments for a memoized selector"
```

### L.7 Workspace integrity

This investigation adds exactly one file (this document) and modifies no source file. Two workspace-hygiene measures were taken so that running the code left the repository byte-clean:

1. **Jest cache redirected out of the repo.** Every committed-suite run used `--cacheDirectory=/tmp/obs/jest-cache`, so Jest never recreated the git-ignored `/.cache/` directory (`.gitignore:L15`) inside the working tree. Verified absent after the runs:

```sh
git status --porcelain --ignored -- .cache
test -e .cache && echo PRESENT || echo ABSENT
```

```text
(empty => .cache absent)
ABSENT
```

2. **Harness lives outside the repo.** All observation scripts, the `createSelector` copy, the esbuild bundle, and the Jest cache reside under `/tmp/obs` (outside the working tree); nothing was written into the repository during evidence capture.

> **N3 note — ignored/generated build artifacts.** The duplicate-mock stderr notice in §L.2 references `dist/cjs` and `dist/esm` copies of `wpcom-proxy-request.js`. Those `dist/` copies are **generated build output** matched by `.gitignore:L69` (`/packages/*/dist/`) and are **not** tracked; only the `src/__mocks__/` mock is a committed file. They are therefore repo state produced by the workspace `postinstall` build, not files introduced by this investigation.

**Finalization evidence (filled in during the commit phase).** The following two commands — removal of the out-of-repo harness and the final repository status — are recorded with their real, unedited output in the commit phase of this work:

**Harness removal.**

```sh
rm -rf /tmp/obs
ls -d /tmp/obs 2>&1 || echo '/tmp/obs removed (absent)'
```

```text
ls: cannot access '/tmp/obs': No such file or directory
/tmp/obs removed (absent)
```

**Final repository status (working tree at finalization, immediately before committing this document).** The single tracked change is this document; `.cache` absence is shown in item 1 above.

```sh
git status --porcelain
```

```text
 M blitzy/documentation/wp-calypso_be7e5cc64162.md
```

### L.8 Observation harness scripts (verbatim, M7)

Every count and behavior above was produced by these self-contained scripts. Each resolves the real source via the `REPO` environment variable, imports the canonical entry point, and runs each scenario **twice** (`RUN 1` / `RUN 2`) in a single process. They are reproduced here byte-for-byte from `/tmp/obs`.

#### `/tmp/obs/q1_comparison.mjs`

```js
// Q1: comparison mechanism — referential identity vs deep equality.
// Self-contained; runs the scenario TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	let calls = 0;
	const selector = ( [ posts ], siteId ) => {
		calls++;
		return Object.values( posts ).filter( ( p ) => p.siteId === siteId );
	};
	const getSitePosts = treeSelect( ( state ) => [ state.posts ], selector );

	const posts = { id1: { id: 'id1', siteId: 'site1' } };
	const state1 = { posts };
	const r1 = getSitePosts( state1, 'site1' );
	const r2 = getSitePosts( state1, 'site1' ); // same reference + same arg
	const sameRefCalls = calls;

	// deep-equal but DIFFERENT reference for state.posts
	const state2 = { posts: { id1: { id: 'id1', siteId: 'site1' } } };
	const r3 = getSitePosts( state2, 'site1' );

	return {
		sameRefCalls,
		identity_r1_eq_r2: r1 === r2,
		totalCalls: calls,
		diffRef_r1_eq_r3: r1 === r3,
		valuesEqual: JSON.stringify( r1 ) === JSON.stringify( r3 ),
	};
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log(
		'same-ref: calls = ' + o.sameRefCalls + ' | r1===r2 (identity) = ' + o.identity_r1_eq_r2
	);
	console.log(
		'deep-equal-diff-ref: calls = ' + o.totalCalls + ' | r1===r3 = ' + o.diffRef_r1_eq_r3
	);
	console.log( 'JSON r1 == JSON r3 (values equal) = ' + o.valuesEqual );
}
```

#### `/tmp/obs/q2_counts.mjs`

```js
// Q2: invocation counts — same-state x2, same-state x1000, changed dependent, distinct args.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function makeSel() {
	let calls = 0;
	const selector = ( [ posts ], siteId ) => {
		calls++;
		return Object.values( posts ).filter( ( p ) => p.siteId === siteId );
	};
	const sel = treeSelect( ( state ) => [ state.posts ], selector );
	return { sel, calls: () => calls };
}

function scenario() {
	// A: same state, same arg, x2
	const a = makeSel();
	const stateA = { posts: { id1: { id: 'id1', siteId: 'site1' } } };
	a.sel( stateA, 'site1' );
	a.sel( stateA, 'site1' );
	const A = a.calls();

	// B: same state, same arg, x1000
	const b = makeSel();
	const stateB = { posts: { id1: { id: 'id1', siteId: 'site1' } } };
	for ( let i = 0; i < 1000; i++ ) b.sel( stateB, 'site1' );
	const B = b.calls();

	// C: changed dependent via immutable spread
	const c = makeSel();
	const post1 = { id: 'id1', siteId: 'site1' };
	c.sel( { posts: { id1: post1 } }, 'site1' );
	c.sel( { posts: { id1: { ...post1, modified: true } } }, 'site1' );
	const C = c.calls();

	// D: two distinct arguments (same state)
	const d = makeSel();
	const stateD = {
		posts: { id1: { id: 'id1', siteId: 'site1' }, id3: { id: 'id3', siteId: 'site2' } },
	};
	d.sel( stateD, 'site1' );
	d.sel( stateD, 'site2' );
	const D = d.calls();

	return { A, B, C, D };
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log( 'A same-state x2 -> calls = ' + o.A );
	console.log( 'B same-state x1000 -> calls = ' + o.B );
	console.log( 'C changed-dependent (spread) -> calls = ' + o.C );
	console.log( 'D distinct-args -> calls = ' + o.D );
}
```

#### `/tmp/obs/q3_coexistence.mjs`

```js
// Q3: multi-entry coexistence per distinct GENERATED key + default args.join() collisions.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	// (1) Coexistence: id1, id2, id1 -> 2 computations, first id1 survives.
	let calls1 = 0;
	const getPostByIdWithData = treeSelect(
		( state, postId ) => [ state.posts[ postId ] ],
		( [ post ] ) => {
			calls1++;
			return { ...post, withData: true };
		}
	);
	const state = { posts: { id1: { id: 'id1' }, id2: { id: 'id2' } } };
	const a = getPostByIdWithData( state, 'id1' );
	const b = getPostByIdWithData( state, 'id2' );
	const c = getPostByIdWithData( state, 'id1' );

	// (2) Default-key collision: single arg "a,b" vs two args "a","b" -> same generated key "a,b".
	let calls2 = 0;
	const collideSel = treeSelect(
		( s ) => [ s.node ],
		( [ n ], ...args ) => {
			calls2++;
			return { args: args.slice() };
		}
	);
	const cstate = { node: { v: 1 } };
	const x = collideSel( cstate, 'a,b' ); // key: "a,b"
	const y = collideSel( cstate, 'a', 'b' ); // key: "a,b" (COLLISION)

	// (3) Numeric/string collision: 1 vs '1' -> both "1".
	let calls3 = 0;
	const numSel = treeSelect(
		( s ) => [ s.node ],
		( [ n ], v ) => {
			calls3++;
			return { v };
		}
	);
	const nstate = { node: { v: 1 } };
	const p = numSel( nstate, 1 ); // key "1"
	const q = numSel( nstate, '1' ); // key "1" (COLLISION)

	// (4) Nullish/empty collision: null, undefined, '' arguments all join() to "".
	let calls4 = 0;
	const nullSel = treeSelect(
		( s ) => [ s.node ],
		( [ n ], v ) => {
			calls4++;
			return { v: String( v ) };
		}
	);
	const nnstate = { node: { v: 1 } };
	const rNull = nullSel( nnstate, null ); // key ""
	const rUndef = nullSel( nnstate, undefined ); // key "" (COLLISION)
	const rEmpty = nullSel( nnstate, '' ); // key "" (COLLISION)

	return {
		coexistCalls: calls1,
		firstSurvives: a === c,
		distinctEntries: a === b,
		collideCalls: calls2,
		collideSame: x === y,
		numCalls: calls3,
		numSame: p === q,
		nullCalls: calls4,
		nullSame: rNull === rUndef && rUndef === rEmpty,
	};
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log( 'coexistence: calls = ' + o.coexistCalls + ' (3 calls issued: id1,id2,id1)' );
	console.log( 'first id1 result survived: a === c = ' + o.firstSurvives );
	console.log( 'id2 entry distinct: a === b = ' + o.distinctEntries );
	console.log(
		'COLLISION join: "a,b" vs "a","b" -> calls = ' +
			o.collideCalls +
			' | same result = ' +
			o.collideSame
	);
	console.log(
		'COLLISION numeric/string: 1 vs "1" -> calls = ' + o.numCalls + ' | same result = ' + o.numSame
	);
	console.log(
		'COLLISION nullish/empty: null vs undefined vs "" -> calls = ' +
			o.nullCalls +
			' | all same result = ' +
			o.nullSame
	);
}
```

#### `/tmp/obs/q4_clearcache.mjs`

```js
// Q4: programmatic full clear via selector.clearCache().
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	let calls = 0;
	const getSitePosts = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ], siteId ) => {
			calls++;
			return Object.values( posts ).filter( ( p ) => p.siteId === siteId );
		}
	);
	const state = { posts: { id1: { id: 'id1', siteId: 'site1' } } };

	const firstResult = getSitePosts( state, 'site1' );
	const memoizedResult = getSitePosts( state, 'site1' );
	const before = { calls, same: memoizedResult === firstResult };

	getSitePosts.clearCache();
	const afterClearResult = getSitePosts( state, 'site1' );
	const after = {
		calls,
		same: afterClearResult === firstResult,
		valueEqual: JSON.stringify( afterClearResult ) === JSON.stringify( firstResult ),
	};

	return { before, after };
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log(
		'before clearCache: calls = ' +
			o.before.calls +
			' | memoizedResult === firstResult = ' +
			o.before.same
	);
	console.log(
		'after clearCache:  calls = ' +
			o.after.calls +
			' | afterClearResult === firstResult = ' +
			o.after.same
	);
	console.log( 'value still equal after clear: JSON equal = ' + o.after.valueEqual );
}
```

#### `/tmp/obs/q5_nullish_primitive.mjs`

```js
// Q5: nullish vs primitive DEPENDENTS (getDependents return values) + primitive ARGUMENTS.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	// Nullish dependents [null, undefined] -> memoized.
	const nullishSel = treeSelect(
		() => [ null, undefined ],
		() => []
	);
	const s = {};
	const r1 = nullishSel( s );
	const r2 = nullishSel( s );
	const nullishMemoized = r1 === r2;

	// Non-nullish primitive DEPENDENTS -> throw TypeError. Include number(0), boolean(false), string('').
	const primitiveDeps = [ true, 1, 'a', false, '', 0 ];
	const depResults = primitiveDeps.map( ( primitive ) => {
		const sel = treeSelect(
			() => [ primitive ],
			() => []
		);
		try {
			sel( {} );
			return { dependent: primitive, threw: false, message: null };
		} catch ( e ) {
			return { dependent: primitive, threw: true, message: e.constructor.name + ': ' + e.message };
		}
	} );

	// Primitive ARGUMENTS (distinct path) -> do NOT throw.
	const argSel = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ] ) => Object.values( posts )
	);
	const primitiveArgs = [ 1, '', 'foo', true, null, undefined ];
	const argResults = primitiveArgs.map( ( a ) => {
		try {
			argSel( { posts: [] }, a );
			return { arg: a, threw: false };
		} catch ( e ) {
			return { arg: a, threw: true };
		}
	} );

	return { nullishMemoized, depResults, argResults };
}

const show = ( v ) => ( typeof v === 'string' ? JSON.stringify( v ) : String( v ) );
for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log(
		'nullish dependents [null, undefined]: firstResult === secondResult = ' + o.nullishMemoized
	);
	console.log( '--- non-nullish primitive dependents (each thrown error captured) ---' );
	for ( const d of o.depResults )
		console.log(
			'dependent = ' + show( d.dependent ) + ' | threw = ' + d.threw + ' | ' + d.message
		);
	console.log( '--- primitive ARGUMENTS to selector (distinct from dependents) ---' );
	for ( const a of o.argResults ) console.log( 'arg = ' + show( a.arg ) + ' | threw = ' + a.threw );
}
```

#### `/tmp/obs/q6_cachekey.mjs`

```js
// Q6: custom getCacheKey for object arguments; object-arg guard without it.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	// Without getCacheKey: object argument throws (dev).
	const noKeySel = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ], query ) => Object.values( posts ).filter( ( p ) => p.siteId === query.siteId )
	);
	let withoutThrew = false,
		withoutMsg = null;
	try {
		noKeySel( { posts: {} }, { siteId: 'site1' } );
	} catch ( e ) {
		withoutThrew = true;
		withoutMsg = e.constructor.name + ': ' + e.message;
	}

	// With getCacheKey: object arguments enabled; dedupe by returned key.
	let calls = 0;
	const keyedSel = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ], query ) => {
			calls++;
			return Object.values( posts )
				.filter( ( p ) => p.siteId === query.siteId )
				.map( ( p ) => p.id );
		},
		{ getCacheKey: ( query ) => `key:${ query.siteId }` }
	);
	const state = {
		posts: { id1: { id: 'id1', siteId: 'site1' }, id2: { id: 'id2', siteId: 'site1' } },
	};
	const firstResult = keyedSel( state, { siteId: 'site1', foo: 'bar' } );
	const secondResult = keyedSel( state, { siteId: 'site1', foo: 'baz' } ); // same key -> dedupe
	const callsAfterDedupe = calls;
	const thirdResult = keyedSel( state, { siteId: 'site2' } ); // distinct key
	const callsAfterDistinct = calls;

	return {
		withoutThrew,
		withoutMsg,
		callsAfterDedupe,
		dedupe: firstResult === secondResult,
		value: JSON.stringify( firstResult ),
		callsAfterDistinct,
		distinct: thirdResult === firstResult,
	};
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log( 'object-arg WITHOUT getCacheKey: threw = ' + o.withoutThrew + ' | ' + o.withoutMsg );
	console.log(
		'with getCacheKey, two same-siteId objects: calls = ' +
			o.callsAfterDedupe +
			' | firstResult === secondResult = ' +
			o.dedupe
	);
	console.log( 'result value = ' + o.value );
	console.log(
		'then distinct siteId: calls = ' +
			o.callsAfterDistinct +
			' | thirdResult === firstResult = ' +
			o.distinct
	);
}
```

#### `/tmp/obs/rootcause.mjs`

```js
// ROOT CAUSE: in-place mutation (stable dependent ref) -> stale; immutable update -> fresh.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	// Derived value: ids of posts whose status === 'published'.
	let calls = 0;
	const getPublishedIds = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ] ) => {
			calls++;
			return Object.values( posts )
				.filter( ( p ) => p.status === 'published' )
				.map( ( p ) => p.id );
		}
	);

	// IN-PLACE mutation: state.posts reference stays stable.
	const postsObj = { id1: { id: 'id1', status: 'published' } };
	const state = { posts: postsObj };
	const initial = getPublishedIds( state, 'x' );
	const initialCalls = calls;
	postsObj.id1.status = 'draft'; // mutate in place, no new reference
	const afterMutation = getPublishedIds( state, 'x' );
	const inPlace = {
		initial: JSON.stringify( initial ),
		initialCalls,
		after: JSON.stringify( afterMutation ),
		afterCalls: calls,
		sameArray: initial === afterMutation,
		stale: JSON.stringify( afterMutation ) === JSON.stringify( [ 'id1' ] ),
	};

	// IMMUTABLE update: new state.posts + new post object.
	calls = 0;
	const getPublishedIds2 = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ] ) => {
			calls++;
			return Object.values( posts )
				.filter( ( p ) => p.status === 'published' )
				.map( ( p ) => p.id );
		}
	);
	const post = { id: 'id1', status: 'published' };
	const s1 = { posts: { id1: post } };
	const immInitial = getPublishedIds2( s1, 'x' );
	const immInitialCalls = calls;
	const s2 = { posts: { id1: { ...post, status: 'draft' } } };
	const immAfter = getPublishedIds2( s2, 'x' );
	const immutable = {
		initial: JSON.stringify( immInitial ),
		initialCalls: immInitialCalls,
		after: JSON.stringify( immAfter ),
		afterCalls: calls,
	};

	return { inPlace, immutable };
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log(
		'IN-PLACE: initial result = ' + o.inPlace.initial + ' | calls = ' + o.inPlace.initialCalls
	);
	console.log(
		'IN-PLACE: after mutation = ' +
			o.inPlace.after +
			' | calls = ' +
			o.inPlace.afterCalls +
			' | correct would be []'
	);
	console.log( 'IN-PLACE: same cached array returned: before === after = ' + o.inPlace.sameArray );
	console.log( 'IN-PLACE: STALE (still shows id1) = ' + o.inPlace.stale );
	console.log(
		'IMMUTABLE: initial result = ' + o.immutable.initial + ' | calls = ' + o.immutable.initialCalls
	);
	console.log(
		'IMMUTABLE: after update = ' + o.immutable.after + ' | calls = ' + o.immutable.afterCalls
	);
	console.log( 'IMMUTABLE: FRESH (now []) = ' + ( o.immutable.after === '[]' ) );
}
```

#### `/tmp/obs/devprod.mjs`

```js
// Dev vs production guards. NODE_ENV read from environment.
// Self-contained; runs the whole set TWICE and labels each run.
const treeSelect = ( await import( process.env.REPO + '/packages/tree-select/src/index.ts' ) )
	.default;

function scenario() {
	// 1) invalid-args guard (dev-only): treeSelect(undefined, undefined) at creation.
	let creationThrew = false,
		creationMsg = null;
	try {
		treeSelect( undefined, undefined );
	} catch ( e ) {
		creationThrew = true;
		creationMsg = e.message;
	}

	// 2) object-argument guard (dev-only): object arg to selector.
	const sel = treeSelect(
		( state ) => [ state.posts ],
		( [ posts ] ) => Object.values( posts )
	);
	let objThrew = false,
		objMsg = null;
	try {
		sel( { posts: {} }, {} );
	} catch ( e ) {
		objThrew = true;
		objMsg = e.message;
	}

	// 3) primitive-dependent guard (UNCONDITIONAL): number 1 dependent.
	const primSel = treeSelect(
		() => [ 1 ],
		() => []
	);
	let primThrew = false,
		primMsg = null;
	try {
		primSel( {} );
	} catch ( e ) {
		primThrew = true;
		primMsg = e.constructor.name + ': ' + e.message;
	}

	return { creationThrew, creationMsg, objThrew, objMsg, primThrew, primMsg };
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = scenario();
	console.log( '===== ' + run + ' =====' );
	console.log( 'NODE_ENV = ' + JSON.stringify( process.env.NODE_ENV ) );
	console.log(
		'treeSelect(undefined, undefined) at creation: threw = ' +
			o.creationThrew +
			( o.creationMsg ? ' | ' + o.creationMsg : '' )
	);
	console.log(
		'object arg {} to selector: threw = ' + o.objThrew + ( o.objMsg ? ' | ' + o.objMsg : '' )
	);
	console.log(
		'primitive dependent (number 1): threw = ' +
			o.primThrew +
			( o.primMsg ? ' | ' + o.primMsg : '' )
	);
}
```

#### `/tmp/obs/cs/createselector_contrast.mjs`

```js
// createSelector contrast: coexistence under stable snapshot + WHOLESALE clear on snapshot change,
// plus complex-arg WARN (not throw). Loads the esbuild-bundled REAL source (non-canonical path).
// Runs the coexistence/clear contrast TWICE in-process; the whole command is also invoked twice.
globalThis.SCRIPT_DEBUG = true; // enable @wordpress/warning dev warnings (gates on globalThis.SCRIPT_DEBUG === true)
const createSelector = ( await import( process.env.CS_BUNDLE ) ).default;

function contrast() {
	let calls = 0;
	// createSelector signature: (selector, getDependants, getCacheKey?). selector is FIRST.
	const sel = createSelector(
		( state, siteId ) => {
			calls++;
			return Object.values( state.posts ).filter( ( p ) => p.siteId === siteId );
		},
		( state ) => [ state.posts ]
	);
	const posts = { id1: { id: 'id1', siteId: 'a' }, id2: { id: 'id2', siteId: 'b' } };
	const state1 = { posts };
	sel( state1, 'a' );
	const c1 = calls; // argA
	sel( state1, 'b' );
	const c2 = calls; // argB (distinct arg, SAME snapshot) -> coexist
	sel( state1, 'a' );
	const c3 = calls; // argA again -> cached
	const state2 = { posts: { ...posts } }; // NEW reference -> snapshot shallow-unequal
	sel( state2, 'a' );
	const c4 = calls; // wholesale clear -> recompute
	sel( state2, 'b' );
	const c5 = calls; // B entry was wiped too -> recompute
	return { c1, c2, c3, c4, c5 };
}

for ( const run of [ 'RUN 1', 'RUN 2' ] ) {
	const o = contrast();
	console.log( '===== ' + run + ' =====' );
	console.log( 'argA (state1):          calls = ' + o.c1 );
	console.log( 'argB (state1):          calls = ' + o.c2 );
	console.log( 'argA again (state1):    calls = ' + o.c3 + ' (coexist -> stays ' + o.c3 + ')' );
	console.log( 'argA (state2, new ref): calls = ' + o.c4 + ' (wholesale clear -> recompute)' );
	console.log( 'argB (state2):          calls = ' + o.c5 + ' (B entry was wiped too)' );
}

// Complex-arg WARN (observed once; @wordpress/warning dedupes by message process-globally).
let warnCount = 0,
	warnMsg = null,
	threw = false;
const origWarn = console.warn;
console.warn = ( ...a ) => {
	warnCount++;
	warnMsg = a.join( ' ' );
};
try {
	const warnSel = createSelector(
		( state, q ) => Object.values( state.posts ),
		( state ) => [ state.posts ]
	);
	warnSel( { posts: {} }, { complex: true } );
	warnSel( { posts: {} }, { complex: true } ); // same message -> deduped
} catch ( e ) {
	threw = true;
}
console.warn = origWarn;
console.log( '===== WARN (complex object argument) =====' );
console.log(
	'complex-arg: threw = ' +
		threw +
		' | warnCount = ' +
		warnCount +
		' | message = ' +
		JSON.stringify( warnMsg )
);
```

### L.9 Coverage checklist — every named item answered

| Prompt item (verbatim intent)                                                                                 | Answered in | Named sub-items covered                                                                                                      |
| ------------------------------------------------------------------------------------------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Stale results "even after the underlying data has changed"                                                    | §1, §H      | in-place mutation vs immutable update; reference-identity root cause                                                         |
| Q1 — "the precise comparison mechanism that determines cache hits and misses"                                 | §B          | reference identity (WeakMap layer) **and** generated-string value equality (terminal Map layer)                              |
| Q2 — "how many times the underlying selector function actually gets called … concrete numbers from test runs" | §C          | same-state (1), same-state ×1000 (1), changed dependent (2), multi-argument (2); cross-referenced to committed suite         |
| Q3 — "separate entries for each unique filter argument … or … invalidates the previous cached result"         | §D          | multi-entry per distinct **generated key**; `args.join()` comma collision; numeric/string collision; nullish/empty collision |
| Q4 — "any way to programmatically clear the entire cache for a selector"                                      | §E          | `clearCache()` (tree-select); `memoizedSelector.cache` (createSelector, §I)                                                  |
| Q5 — "nullish values versus primitive values like numbers or booleans"                                        | §F          | `null`; `undefined`; number (incl. `0`); boolean (incl. `false`); string (incl. `''`); primitive **arguments** contrasted    |
| Q6 — "customize how cache keys are generated … complex query objects as arguments"                            | §G          | default `args.join()`; object-arg guard `Error`; `options.getCacheKey`; dedupe by returned key                               |
| Comparison to `createSelector` / ecosystem norms                                                              | §I          | shallow-equality; wholesale clear; complex-arg warn-not-throw; reselect framing grounded in sibling `README.md:L46`          |
| README argument-order discrepancy                                                                             | §J          | flagged (not corrected) per read-only constraint                                                                             |
| Dev vs production behavior                                                                                    | §K          | dev-only arg guards; unconditional primitive-dependent throw                                                                 |

All six questions and every enumerated example (`null`, `undefined`, number, boolean, string, complex query objects) are addressed explicitly and by name above.

---

_End of investigation report. All behavioral claims are grounded in the unedited command output reproduced in Section L and in `file:line` citations to the source at commit `be7e5cc641622d153040491fd5625c6cb83e12eb`. No source file was modified; this document is the sole addition._
