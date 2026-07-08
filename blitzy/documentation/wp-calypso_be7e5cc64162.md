# Why the Signup **Back** Button Sometimes Navigates Somewhere Unexpected

**Investigation target:** Calypso's classic `/start` signup / onboarding flow (`client/signup/**`).
**Deliverable type:** Read‑only, evidence‑backed Q&A investigation. No product code, tests, configuration, or dependencies were changed. The only durable artifact is this document.
**Methodology:** The answer was produced by **running the real code first** (the real modules under the repository's Jest harness), capturing the **complete, unedited** runtime output, and only then writing the explanation from what was observed.

---

## 1. One‑paragraph summary of the finding

In classic `/start` signup, the Back control is an anchor whose `href` is **exactly** the return value of `NavigationLink.getBackUrl()` [`client/signup/navigation-link/index.jsx`:L183-L186, L192]; the click handler does **not** drive signup navigation because `goToPreviousStep` is only invoked when that prop is supplied [`client/signup/navigation-link/index.jsx`:L125-L126]. Inside `getBackUrl()` [`client/signup/navigation-link/index.jsx`:L78-L115] there is a strict precedence ladder: a `direction !== 'back'` guard [L79-L81], then an **unconditional** `if ( this.props.backUrl ) { return this.props.backUrl; }` [L83-L85], and only then the computed step‑by‑step branch [L87-L114]. That unconditional `backUrl` return — which sits **above** every first‑step / position / eligibility check and before `getPreviousStep()` is ever called — is the single root cause of the two reported symptoms. When an external back target (`props.backUrl`, sourced from a `back_to` query argument or a step‑declared value) is present, Back "slips out into an entirely different flow"; when it is absent, the computed branch can resolve the previous step to `{ stepName: null }` (first step, or empty/unmatched progress), which `getStepUrl()` collapses to the flow **root**, so Back "snaps straight to the first step." Runtime observation shows the behavior is **fully deterministic** given a fixed input tuple — two independent runs produced byte‑identical output — so the perceived "randomness" is variation in the inputs (most notably whether a `backUrl` override is present and the shape of `signupProgress`), not nondeterminism.

---

## 2. The question being answered (verbatim)

> "The Back button is supposed to move one step backward, but every so often it snaps straight to the first step or slips out into an entirely different flow, and it never feels truly random."

> "I keep getting the sense that an external back target is being treated like a quiet override even when the current step should not be eligible for it."

This decomposes into six explicit sub‑questions (answered in §5) plus one implicit question (answered in §7):

1. What decides the destination for a given step?
2. Which inputs win when flow position, component props, and query‑string args disagree?
3. Where does the "external back target" override come from?
4. What precedence rule lets that override take control even when the current step "should not be eligible" for it?
5. What code path handles the expected step‑by‑step navigation that is being bypassed?
6. Observe the computed destination for each step position (runtime evidence table).
7. *(Implicit)* Is the behavior deterministic — i.e., why does it "never feel truly random"?

---

## 3. Methodology — run the code first

### 3.1 Environment (canonical; the configuration a normal contributor uses)

| Component | Value | Evidence |
|---|---|---|
| Node.js | `v22.23.1` (satisfies the repo's canonical `^v22.9.0`) | `.nvmrc` = `22.9.0`; `package.json` `engines.node` |
| Yarn | `4.0.2` | pinned via `packageManager` in `package.json`; provisioned with `corepack enable` |
| Test runner | Jest `29.7.0` via the `test-client` config | `test/client/jest.config.js`, which extends `@automattic/calypso-jest` (preset `testMatch: '<rootDir>/**/test/*.[jt]s?(x)'`, `rootDir: client`) |
| Module resolution | `calypso/*` → `client/*` | Yarn‑workspace symlink `node_modules/calypso -> ../client` (because `client/package.json`'s `name` is `calypso`), created by `yarn install` |

The real modules were exercised through the `calypso/*` alias — **no re‑implementation and no stubs**. This matters: the repository's own reference test `client/signup/navigation-link/test/index.jsx` **mocks** `calypso/signup/utils` (L7‑L12), which would defeat the investigation. The observation script below deliberately uses the **real** `getStepUrl` / `getPreviousStepName` / `NavigationLink` instead.

### 3.2 The temporary observation script

A temporary Jest script was placed at `client/signup/test/zz_observe_backnav.js` (that `test/` directory already exists, and placing the file there satisfies the `testMatch` preset). It opts into `jsdom` on its first line, imports the **real** modules (`export default Flows` requires `.default` under CommonJS `require()`; the named `NavigationLink` is the **unconnected** class, `export class NavigationLink` [`client/signup/navigation-link/index.jsx`:L17]), exercises the primary path **and every edge branch** (first step, empty progress, override present, query‑arg fallback, current‑step‑not‑in‑progress → `pop()`), demonstrates `getStepUrl` framework/flow shaping, and runs the whole battery **twice** to prove determinism.

> This script is **temporary**. It was deleted after the run so the repository is left byte‑for‑byte unchanged (verified with `git status --porcelain`). It is reproduced here only so the evidence is fully repeatable.

```js
/** @jest-environment jsdom */
/* eslint-disable */
const flows = require( 'calypso/signup/config/flows' ).default;
const { NavigationLink } = require( 'calypso/signup/navigation-link' );
const { getStepUrl, getPreviousStepName } = require( 'calypso/signup/utils' );

const LOGGED_OUT = false; // logged-out keeps the first "user-social" step and appends locale "en"
const FLOW = 'onboarding';

function setSearch( search ) {
	// control window.location.search for the query-arg fallback branch
	window.history.replaceState( null, '', '/start' + ( search || '' ) );
}
function mkProgress( names ) {
	const p = {};
	names.forEach( ( n ) => { p[ n ] = { stepName: n, stepSectionName: '', wasSkipped: false }; } );
	return p;
}
function getBackUrl( props ) {
	return new NavigationLink( props ).getBackUrl(); // real unconnected class
}

function battery() {
	const results = {};
	const steps = flows.getFlow( FLOW, LOGGED_OUT ).steps;
	results.onboardingSteps = steps;

	setSearch( '' );
	results.perStep = steps.map( ( stepName, i ) => {
		const prev = getPreviousStepName( FLOW, stepName, LOGGED_OUT );
		return { position: i, stepName,
			getPreviousStepName: prev === undefined ? '(undefined)' : prev,
			getStepUrl: getStepUrl( FLOW, prev ) };
	} );

	setSearch( '' );
	results.shaping = {
		nonDefaultFlow: getStepUrl( 'do-it-for-me', 'new-or-existing-site' ),
		nullStep: getStepUrl( FLOW, null ),
	};

	const base = { direction: 'back', flowName: FLOW, userLoggedIn: LOGGED_OUT };
	setSearch( '' );
	results.override_present = getBackUrl( { ...base, backUrl: '/home/example.wordpress.com',
		stepName: 'plans', positionInFlow: 2, signupProgress: mkProgress( [ 'user-social', 'domains' ] ) } );
	setSearch( '' );
	results.normal_one_step_back = getBackUrl( { ...base, stepName: 'domains', positionInFlow: 1,
		signupProgress: mkProgress( [ 'user-social', 'domains' ] ) } );
	setSearch( '' );
	results.current_not_in_progress_pop = getBackUrl( { ...base, stepName: 'plans', positionInFlow: 2,
		signupProgress: mkProgress( [ 'user-social', 'domains' ] ) } );
	setSearch( '' );
	results.first_step = getBackUrl( { ...base, stepName: 'user-social', positionInFlow: 0,
		allowBackFirstStep: true, signupProgress: mkProgress( [ 'user-social', 'domains' ] ) } );
	setSearch( '' );
	results.empty_progress_non_first = getBackUrl( { ...base, stepName: 'plans', positionInFlow: 2,
		signupProgress: {} } );
	setSearch( '?back_to=%2Fexternal&ref=abc' );
	results.query_arg_fallback = getBackUrl( { ...base, stepName: 'domains', positionInFlow: 1,
		signupProgress: mkProgress( [ 'user-social', 'domains' ] ) } );
	setSearch( '' );
	return results;
}

describe( 'zz_observe_backnav', () => {
	it( 'observes signup Back-button destinations via the real modules', () => {
		const run1 = battery();
		const run2 = battery();
		const deterministic = JSON.stringify( run1 ) === JSON.stringify( run2 );
		console.log( '===BACKNAV_OBSERVATION_START===' );
		console.log( JSON.stringify( { run1, run2, deterministic }, null, 2 ) );
		console.log( '===BACKNAV_OBSERVATION_END===' );
		expect( deterministic ).toBe( true );
	} );
} );
```

**Scenario‑fidelity notes (all confirmed at runtime):**

- `flows.getFlow('onboarding', false)` returns `steps = ['user-social','domains','plans']`. With `isUserLoggedIn = true`, `getFlow` calls `removeUserStepFromFlow` [`client/signup/config/flows.js`:L216, invoked at L275], dropping the user step — so **logged‑out** (`false`) is required to keep `user-social` as step 0.
- `user-social` (not `user`) is step 0 because the feature flag `signup/social-first` is enabled in the canonical config; `flows-pure.js` picks `isEnabled( 'signup/social-first' ) ? 'user-social' : 'user'` [`client/signup/config/flows-pure.js`:L13-L14], assigns it to `userSocialStep` [L30], and the `onboarding` flow uses `steps: [ userSocialStep, 'domains', 'plans' ]`.
- Logged‑out also causes the computed branch to append locale `en` (see Answer 5 / §5.5), which is why the `getBackUrl()` results carry an `/en` suffix while the pure `getStepUrl` per‑step table (Answer 6) does not.

### 3.3 The exact command

```
CI=true TZ=UTC npx jest -c=test/client/jest.config.js zz_observe_backnav --ci --runInBand
```

This command was run **twice** (two independent invocations) and produced byte‑identical observation blocks (see §7).

---

## 4. The complete, unedited captured output

Below is the **complete, unedited** output of one invocation of the exact command above. A single invocation already contains **both** the internal `run1` and `run2` data blocks in full (the script runs the battery twice), so the full `run2` block is present and not abbreviated. The incidental `Browserslist … 17 months old` banner is included verbatim. (Environment note: in this container the run did **not** emit a `punycode` `DeprecationWarning`; only what was actually printed is shown.)

Command:

```
CI=true TZ=UTC npx jest -c=test/client/jest.config.js zz_observe_backnav --ci --runInBand
```

Output (verbatim):

```
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/test/zz_observe_backnav.js (5.414 s)
  ● Console

    console.log
      ===BACKNAV_OBSERVATION_START===

      at Object.log (signup/test/zz_observe_backnav.js:70:11)

    console.log
      {
        "run1": {
          "onboardingSteps": [
            "user-social",
            "domains",
            "plans"
          ],
          "perStep": [
            {
              "position": 0,
              "stepName": "user-social",
              "getPreviousStepName": "(undefined)",
              "getStepUrl": "/start"
            },
            {
              "position": 1,
              "stepName": "domains",
              "getPreviousStepName": "user-social",
              "getStepUrl": "/start/user-social"
            },
            {
              "position": 2,
              "stepName": "plans",
              "getPreviousStepName": "domains",
              "getStepUrl": "/start/domains"
            }
          ],
          "shaping": {
            "nonDefaultFlow": "/start/do-it-for-me/new-or-existing-site",
            "nullStep": "/start"
          },
          "override_present": "/home/example.wordpress.com",
          "normal_one_step_back": "/start/user-social/en",
          "current_not_in_progress_pop": "/start/domains/en",
          "first_step": "/start/en",
          "empty_progress_non_first": "/start/en",
          "query_arg_fallback": "/start/user-social/en?back_to=%2Fexternal&ref=abc"
        },
        "run2": {
          "onboardingSteps": [
            "user-social",
            "domains",
            "plans"
          ],
          "perStep": [
            {
              "position": 0,
              "stepName": "user-social",
              "getPreviousStepName": "(undefined)",
              "getStepUrl": "/start"
            },
            {
              "position": 1,
              "stepName": "domains",
              "getPreviousStepName": "user-social",
              "getStepUrl": "/start/user-social"
            },
            {
              "position": 2,
              "stepName": "plans",
              "getPreviousStepName": "domains",
              "getStepUrl": "/start/domains"
            }
          ],
          "shaping": {
            "nonDefaultFlow": "/start/do-it-for-me/new-or-existing-site",
            "nullStep": "/start"
          },
          "override_present": "/home/example.wordpress.com",
          "normal_one_step_back": "/start/user-social/en",
          "current_not_in_progress_pop": "/start/domains/en",
          "first_step": "/start/en",
          "empty_progress_non_first": "/start/en",
          "query_arg_fallback": "/start/user-social/en?back_to=%2Fexternal&ref=abc"
        },
        "deterministic": true
      }

      at Object.log (signup/test/zz_observe_backnav.js:71:11)

    console.log
      ===BACKNAV_OBSERVATION_END===

      at Object.log (signup/test/zz_observe_backnav.js:72:11)


Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        5.716 s
Ran all test suites matching /zz_observe_backnav/i.
```

The second independent invocation of the same command produced an observation block that was **byte‑identical** to the one above (see §7 for the determinism proof and hashes). The only differences between the two full logs were the incidental Jest timing lines (`PASS … (5.414 s)` vs `(5.429 s)`, and the trailing `Time:` line).

---

## 5. The six answers

Each answer pairs (a) the exact implementing code with `file:line` references, (b) the value actually captured by running that code (from §4), and (c) a cause→effect explanation.

### 5.1 Answer (1) — What decides the destination for a given step?

**The single deciding function is `NavigationLink.getBackUrl()`** [`client/signup/navigation-link/index.jsx`:L78-L115]. The signup Back control is a `<Button>` rendered as an anchor whose `href` is the return value of that method:

- In `render()`, `hrefUrl` is computed as `this.props.direction === 'forward' && this.props.forwardUrl ? this.props.forwardUrl : this.getBackUrl()` [`client/signup/navigation-link/index.jsx`:L183-L186] and passed straight through as `href={ hrefUrl }` [L192]. For a Back button (`direction === 'back'`) `hrefUrl` is therefore exactly `getBackUrl()`.
- The click handler does **not** drive signup back‑navigation. `handleClick` only invokes `goToPreviousStep` **when that prop is provided**: `else if ( this.props.goToPreviousStep ) { this.props.goToPreviousStep(); }` [`client/signup/navigation-link/index.jsx`:L125-L126]. In the classic `/start` `StepWrapper` path no such handler performs the navigation — the anchor `href` does.

**Cause→effect:** because Back is a plain anchor, the browser navigates to whatever string `getBackUrl()` returns. There is no competing code path; the destination is 100% the return value of `getBackUrl()`.

**Captured evidence:** every scenario in §4 is a direct call of `NavigationLink.getBackUrl()` (via `new NavigationLink( props ).getBackUrl()`), and each produced exactly one destination string (e.g. `override_present` → `"/home/example.wordpress.com"`), confirming this method alone decides the target.

### 5.2 Answer (2) & (4) — Precedence order, and why the override wins even when "ineligible"

`getBackUrl()` [`client/signup/navigation-link/index.jsx`:L78-L115] is a strict top‑to‑bottom precedence ladder:

1. **Direction guard** — `if ( this.props.direction !== 'back' ) { return; }` returns `undefined` for a non‑Back link [L79-L81].
2. **Unconditional external override** — `if ( this.props.backUrl ) { return this.props.backUrl; }` [L83-L85]. This returns **before** any first‑step / position / eligibility check and **before `getPreviousStep()` is ever called** [`getPreviousStep` is only invoked later at L98]. This is precisely the precedence rule that lets an external back target "take control even when the current step should not be eligible for it."
3. **Computed step‑by‑step branch** — everything at L87-L114 is reached **only** when `props.backUrl` is falsy.

**Precedence winner:** `props.backUrl` (the external override) **>** the computed flow/step logic. Query‑string arguments are **not** a competing top‑level input for the *decision*: in the computed branch they are merely parsed from `window.location.search` [L87-L89] and re‑appended to the resulting URL via `getStepUrl`'s `params` [L108-L114]. So the three "inputs" the user intuits — flow position, component props, query args — are **not** peers: `props.backUrl` short‑circuits everything, the computed branch uses flow position, and query args only decorate the computed result.

**Why an "ineligible" step still shows and overrides:** `StepWrapper` computes `allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }` [`client/signup/step-wrapper/index.jsx`:L65]. So the mere presence of a `backUrl` sets `allowBackFirstStep = true`, which suppresses `render()`'s early `return null` that would otherwise hide Back on step 0 (that early return requires `positionInFlow === 0 && direction === 'back' && ! stepSectionName && ! allowBackFirstStep` [`client/signup/navigation-link/index.jsx`:L154-L161]). Net effect: a `backUrl` both **reveals** Back on an otherwise‑ineligible first step *and* is **returned unconditionally** as the destination.

**Cause→effect:** the override is structurally superior to the flow logic because it is checked first and returns immediately; eligibility is never consulted.

**Captured evidence:** `override_present` — a **mid‑flow** step (`plans`, `positionInFlow: 2`) with `backUrl='/home/example.wordpress.com'` — returned `"/home/example.wordpress.com"` (§4). No flow/step math ran; the raw override was echoed back.

### 5.3 Answer (3) — Where the "external back target" override comes from

`props.backUrl` (and the `back_to` query argument that feeds it) originates from several routes, all of which ultimately populate the `backUrl` prop that the ladder returns unconditionally:

- **`back_to` query argument via `StepWrapper` (the primary route).** `mapStateToProps` reads `const backToParam = getCurrentQueryArguments( state )?.back_to?.toString();` [`client/signup/step-wrapper/index.jsx`:L274], accepts it **only if it starts with `/`** — `const backTo = backToParam?.startsWith( '/' ) ? backToParam : undefined;` [L275] — and then sets `const backUrl = ownProps.backUrl ?? backTo;` [L277]. That `backUrl` is passed to `NavigationLink` via `backUrl={ this.props.backUrl }` [L62].
- **Dependency‑store channel (controller), gated to one flow.** `back_to` is dispatched into the signup dependency store **only for the `woocommerce-install` flow**: `if ( 'woocommerce-install' === flowName ) { if ( context?.query?.back_to ) { context.store.dispatch( updateDependencies( { back_to: context.query.back_to } ) ); } }` [`client/signup/controller.js`:L226-L229].
- **`back_to` declared as a query‑provided dependency** for specific flows: `providesDependenciesInQuery` / `optionalDependenciesInQuery` include `'back_to'` at [`client/signup/config/flows-pure.js`:L410-L411], again at [L446-L447], and [L478-L479]; at the step level, `dependencies: [ 'siteSlug', 'back_to' ]` and `optionalDependencies: [ 'back_to' ]` [`client/signup/config/steps-pure.js`:L875-L876].
- **Step‑declared / hardcoded `backUrl` values.** A step config hardcodes `backUrl: 'mailbox-domain/'` [`client/signup/config/steps-pure.js`:L399]; a flow builder hardcodes `back_to: `/start/setup-site/store-features?siteSlug=${ siteSlug }`` [`client/signup/config/flows.js`:L174]; and the domains step computes its own `backUrl` across many branches [`client/signup/steps/domains/index.jsx`:L1367-L1430], using external‑source overrides `backUrlExternalSourceStepsOverrides = [ 'use-your-domain' ]` [`client/signup/steps/domains/utils.js`:L6] and `backUrlSourceOverrides = { 'business-name-generator': '/business-name-generator', 'domains': '/domains' }` [L9-L12].
- **Steps that read `back_to` and forward it.** Individual step components read the `back_to` dependency and pass it as `backUrl`, also setting `allowBackFirstStep={ !! backUrl }` — e.g. `const { back_to: backUrl } = signupDependencies;` in `difm-site-picker` [`client/signup/steps/difm-site-picker/index.tsx`:L43, backUrl/allowBackFirstStep at L124-L125], `new-or-existing-site` [`client/signup/steps/new-or-existing-site/index.tsx`:L31, L77-L78], and `site-options` [`client/signup/steps/site-options/index.tsx`:L27, L129-L130].

**Cause→effect:** any of these routes can set `props.backUrl` to a value that points **outside** the current flow (e.g. `/home/...`, `/business-name-generator`, a WooCommerce store URL). Because the ladder returns it unconditionally (§5.2), that external value becomes the Back destination.

**Captured evidence:** the `query_arg_fallback` scenario set `window.location.search = '?back_to=%2Fexternal&ref=abc'` with **no** `backUrl` prop; the result was `"/start/user-social/en?back_to=%2Fexternal&ref=abc"` (§4) — i.e. with no prop the `back_to` query is *not* consulted as an override by `getBackUrl` itself (that promotion happens in `StepWrapper.mapStateToProps` [L274-L277]); it is only re‑appended as a query arg. This demonstrates the distinction between the **prop** (which overrides) and the **raw query string** (which merely decorates) — see §5.5.

### 5.4 Answer (5) — The bypassed step‑by‑step path

When `props.backUrl` is absent, `getBackUrl()` falls through to the canonical "one step backward" logic that the override otherwise bypasses:

- **`getPreviousStep( flowName, signupProgress, currentStepName )`** [`client/signup/navigation-link/index.jsx`:L47-L76]:
  - initializes `const previousStep = { stepName: null };` [L48];
  - if `isFirstStepInFlow( flowName, currentStepName, this.props.userLoggedIn )` → returns `{ stepName: null }` [L50-L52];
  - builds `filteredProgressedSteps = getFilteredSteps( … ).filter( ( step ) => ! step.wasSkipped )` [L56-L60]; if that is empty → returns `{ stepName: null }` [L61-L63];
  - finds the current step's index via `findIndex` [L66-L68]; if not found (`=== -1`) → returns `filteredProgressedSteps.pop()` — **the last progressed step** [L71-L73]; otherwise returns `filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep` [L75].
- **`getFilteredSteps( flowName, progress, isUserLoggedIn )`** [`client/signup/utils.js`:L137-L150] filters `progress` to entries whose `stepName` belongs to the flow and sorts them by the flow's declared order (`sortBy( … flow.steps.indexOf( stepName ) )`).
- **`isFirstStepInFlow`** [`client/signup/utils.js`:L28-L31] = `flows.getFlow( … ).steps.indexOf( stepName ) === 0`.
- **`getPreviousStepName( flowName, currentStepName, isUserLoggedIn )`** [`client/signup/utils.js`:L85-L88] is the pure sibling helper = `flow.steps[ flow.steps.indexOf( currentStepName ) - 1 ]`; for the first step the index is `-1`, so it returns `undefined`.
- The resolved previous step then feeds **`getStepUrl( … )`** [`client/signup/utils.js`:L45-L69] (called at [`client/signup/navigation-link/index.jsx`:L108-L114]), which builds the URL: the framework prefix is `/setup` only if `window.location.pathname.startsWith( '/setup' )`, else `/start` [`client/signup/utils.js`:L57-L61]; the **default** flow name (`onboarding`, from `getDefaultFlowName()` returning `'onboarding'` [`client/signup/config/flows.js`:L243-L245]) is **omitted** from `/start` routes [L63-L67]; query args are appended via `addQueryArgs( params, url )` [L68]. A `null` step name collapses to the flow **root**.

**Cause→effect:** this is the "move exactly one step backward" behavior the user expects. It is only reached when there is no `backUrl` override; the override in §5.2 is what "bypasses" it.

**Captured evidence:** `normal_one_step_back` (current `domains` present in progress, no `backUrl`) → `"/start/user-social/en"` — exactly one step back via `filteredProgressedSteps[ index - 1 ]`. `current_not_in_progress_pop` (current `plans` not yet in progress) → `"/start/domains/en"` — the `pop()` branch [L71-L73] returned the last progressed step (`domains`).

### 5.5 Answer (6) — Per‑step computed destination (runtime table)

Using the default `onboarding` flow (`steps = ["user-social","domains","plans"]`), computed via `getPreviousStepName` + `getStepUrl` (no locale argument, so no `/en` suffix here). These are the **captured** values from §4:

| Step position | Step name | `getPreviousStepName` | Computed `getStepUrl` |
|---|---|---|---|
| 0 (first) | `user-social` | `(undefined)` | `/start` (flow root — "snaps to first step") |
| 1 | `domains` | `user-social` | `/start/user-social` |
| 2 | `plans` | `domains` | `/start/domains` |

**Cause→effect:** at position 0, `getPreviousStepName` computes `flow.steps[ -1 ]` = `undefined` [`client/signup/utils.js`:L85-L88]; `getStepUrl('onboarding', undefined)` yields `/start` because `stepName` is falsy (`const step = stepName ? … : '';` [L54]) and the default flow name is omitted [L63-L67]. At positions 1 and 2 the previous step is a real step name, so the URL is `/start/<previous-step>`.

---

## 6. The two reported symptoms mapped to concrete branches

### 6.1 "snaps straight to the first step"

This is the `getPreviousStep()` → `{ stepName: null }` path, which `getStepUrl(flow, null, …)` collapses to the flow **root**. It happens in two distinct sub‑cases:

- **First step** — `isFirstStepInFlow()` is true, so `getPreviousStep` returns `{ stepName: null }` immediately [`client/signup/navigation-link/index.jsx`:L50-L52].
- **Empty / unmatched progress on a non‑first step** — `filteredProgressedSteps.length === 0`, so `getPreviousStep` returns `{ stepName: null }` [`client/signup/navigation-link/index.jsx`:L61-L63].

Then `getStepUrl` produces the flow root because `previousStep.stepName` is `null` (falsy) [`client/signup/utils.js`:L54, L63-L67].

**Captured evidence:** `first_step` → `"/start/en"`; `empty_progress_non_first` → `"/start/en"` (§4). The pure step‑0 case in the per‑step table (§5.5) → `/start`. (The `/en` suffix is only present on the `getBackUrl` results because the caller is logged‑out — see §6.4.)

### 6.2 "slips out into an entirely different flow"

This is the **unconditional** `return this.props.backUrl` [`client/signup/navigation-link/index.jsx`:L83-L85], which bypasses all flow/step logic and returns whatever external target was supplied (§5.3).

**Captured evidence:** `override_present` → `"/home/example.wordpress.com"` (§4) — a destination entirely outside the `onboarding` `/start` flow.

### 6.3 "a quiet override even when the current step should not be eligible"

The override returns **before** any eligibility check (§5.2), and `allowBackFirstStep = !! backUrl` [`client/signup/step-wrapper/index.jsx`:L65] reveals the Back button even on step 0 where it would otherwise be hidden [`client/signup/navigation-link/index.jsx`:L154-L161]. So the external target is both *visible* and *authoritative* on a step the user considers ineligible.

### 6.4 `NavigationLink.getBackUrl()` precedence / edge‑case table (captured values)

All values below are taken verbatim from the §4 output. The caller is logged‑out, so the computed branch appends locale `en`.

| Scenario | Conditions | Observed `getBackUrl()` |
|---|---|---|
| Override present | `backUrl='/home/example.wordpress.com'`, mid‑flow `plans` | `/home/example.wordpress.com` |
| Normal one‑step‑back | no `backUrl`, current `domains` present in progress | `/start/user-social/en` |
| Current step not yet in progress | no `backUrl`, current `plans` not in progress | `/start/domains/en` (`pop()` = last progressed step) |
| First step | no `backUrl`, current `user-social`, `isFirstStepInFlow` true | `/start/en` (flow root) |
| Empty progress, non‑first | no `backUrl`, current `plans`, empty progress | `/start/en` (flow root) |
| Query‑arg fallback | `?back_to=%2Fexternal&ref=abc` in `window.location.search`, no `backUrl` prop | `/start/user-social/en?back_to=%2Fexternal&ref=abc` |

### 6.5 `getStepUrl` framework/flow shaping (captured values)

| Call | Observed result | Why |
|---|---|---|
| `getStepUrl('do-it-for-me', 'new-or-existing-site')` | `/start/do-it-for-me/new-or-existing-site` | non‑default flow name **appears** in the path [`client/signup/utils.js`:L63-L67] |
| `getStepUrl('onboarding', null)` | `/start` | default flow name omitted **and** null step collapses to the flow root [`client/signup/utils.js`:L54, L63-L67] |

### 6.6 Query‑argument handling detail

In the computed branch, `fallbackQueryParams` is parsed from `window.location.search` [`client/signup/navigation-link/index.jsx`:L87-L89] and threaded through to `getStepUrl` as `params` [L108-L114], so query args such as `back_to` and `ref` are **re‑appended** to the destination (they persist across a back navigation) but are **not** the deciding factor for *which step* is chosen. This is exactly why `query_arg_fallback` produced `"/start/user-social/en?back_to=%2Fexternal&ref=abc"` — the step (`user-social`) was chosen by the normal one‑step‑back logic, and the query string was merely carried along. The logged‑out `en` suffix comes from `const locale = ! userLoggedIn ? getLocaleSlug() : '';` [L106] (`getLocaleSlug()` defaults to `en`).

---

## 7. Determinism — answering "it never feels truly random"

**The behavior is fully deterministic.** Two forms of evidence establish this:

1. **In‑process:** the script runs the entire battery **twice** (`run1` and `run2`) and compares them with `JSON.stringify( run1 ) === JSON.stringify( run2 )`; the captured output shows `"deterministic": true` and the two blocks are visibly identical (§4).
2. **Across independent invocations:** the exact command was run twice. The observation block (the payload between the `===BACKNAV_OBSERVATION_START===` / `===BACKNAV_OBSERVATION_END===` markers) was **byte‑identical** across both runs.

Verification commands and their output:

```
$ sed -n '/===BACKNAV_OBSERVATION_START===/,/===BACKNAV_OBSERVATION_END===/p' run1.log | sha256sum
ea2eed42d5b7a8516765a8a060c48d7b5149f2a13bc8357a1cd7fec202c6ba8b  -
$ sed -n '/===BACKNAV_OBSERVATION_START===/,/===BACKNAV_OBSERVATION_END===/p' run2.log | sha256sum
ea2eed42d5b7a8516765a8a060c48d7b5149f2a13bc8357a1cd7fec202c6ba8b  -
$ diff <block1> <block2>   # (no output — byte-identical)
```

The only differences between the two **full** logs were incidental Jest timing lines (`PASS … (5.414 s)` vs `(5.429 s)` and the trailing `Time:` line), which do not affect any observed destination.

**Conclusion:** the destination is a pure function of the input tuple `( flowName, position/stepName, props.backUrl, back_to query, signupProgress )`. The perceived "randomness" is variation in those inputs — most notably (a) whether a `backUrl` override is present (which flips between the unconditional‑override result and the computed result), and (b) the exact shape of `signupProgress`, which flips the computed branch between the normal `[ index - 1 ]` result, the `pop()` (last‑progressed) result, and the `null` → flow‑root result. Nothing in the code path consults a clock, a random source, or unstable global state to decide the step.

---

## 8. Secondary framework note (acknowledged, not the target)

Calypso maintains **two** signup frameworks: the classic **"Start"** framework (`client/signup/**`, served under `/start`) — the target of this investigation — and the newer **"Stepper"** framework (`client/landing/stepper/**`, served under `/setup`), which implements its own per‑flow `goBack()` and its own `back_to` handling. The shared `getStepUrl()` selects the `/setup` prefix only when `window.location.pathname.startsWith( '/setup' )` [`client/signup/utils.js`:L57-L61]; otherwise it uses `/start`. The three‑input precedence the user describes (flow position vs. `backUrl` prop vs. `back_to` query) is specifically the classic `NavigationLink` / `StepWrapper` path; Stepper is mentioned only for completeness and was neither modified nor exhaustively analyzed.

---

## 9. Coverage pass

**Six sub‑questions** — all answered with `file:line` + captured value + cause→effect:

- [x] **(1)** Destination is decided by `getBackUrl()` (anchor `href`; click handler is a no‑op for signup) — §5.1.
- [x] **(2)** Precedence: `props.backUrl` override **>** computed flow/step logic; query args only decorate — §5.2, §6.6.
- [x] **(3)** Override sources: `back_to` query via `StepWrapper` `mapStateToProps`, controller dependency channel (woocommerce‑install), flow/step `back_to` declarations, hardcoded/computed `backUrl` — §5.3.
- [x] **(4)** The override returns unconditionally **above** every eligibility/position/first‑step check, and `allowBackFirstStep = !!backUrl` reveals it on step 0 — §5.2, §6.3.
- [x] **(5)** Bypassed path: `getPreviousStep` → `getFilteredSteps` / `isFirstStepInFlow` / `getPreviousStepName` → `getStepUrl` — §5.4.
- [x] **(6)** Per‑step computed destination table — §5.5.

**Implicit question** — [x] Determinism established with two‑run byte‑identical evidence — §7.

**Named mechanisms** — each addressed with evidence:

- [x] `getBackUrl()` [`navigation-link/index.jsx`:L78-L115] — §5.1, §5.2.
- [x] `getPreviousStep()` [`navigation-link/index.jsx`:L47-L76] — §5.4.
- [x] `getPreviousStepName()` [`utils.js`:L85-L88] — §5.4, §5.5.
- [x] `getStepUrl()` [`utils.js`:L45-L69] — §5.4, §5.5, §6.5.
- [x] `getFilteredSteps()` [`utils.js`:L137-L150] — §5.4.
- [x] `isFirstStepInFlow()` [`utils.js`:L28-L31] — §5.4, §6.1.
- [x] `StepWrapper` `back_to`→`backUrl` mapping and `allowBackFirstStep = !!backUrl` [`step-wrapper/index.jsx`:L274-L277, L65, L62] — §5.3, §6.3.
- [x] controller `back_to` dispatch (woocommerce‑install) [`controller.js`:L226-L229] — §5.3.

**Edge branches** — each triggered and observed:

- [x] First step (`isFirstStepInFlow` true) → `/start/en` — §6.1.
- [x] Empty progress, non‑first → `/start/en` — §6.1.
- [x] Override present → `/home/example.wordpress.com` — §6.2.
- [x] Query‑arg fallback → `/start/user-social/en?back_to=%2Fexternal&ref=abc` — §6.6.
- [x] Current step not in progress → `pop()` → `/start/domains/en` — §5.4.

**Scope / read‑only** — [x] No product code, tests, config, or dependencies were changed. The temporary observation script was deleted after capturing output; the only durable artifact is this document.

---

## 10. Bottom line

The Back destination is entirely the return of `getBackUrl()`. Its **first substantive line** — `if ( this.props.backUrl ) { return this.props.backUrl; }` [`client/signup/navigation-link/index.jsx`:L83-L85] — is an **unconditional** external override that sits above every eligibility, position, and first‑step check. That single rule explains "slips out into an entirely different flow" (an external `backUrl` is returned verbatim) and, in its absence, the `{ stepName: null }` → flow‑root path explains "snaps straight to the first step." The system is deterministic; the apparent randomness is input variation (presence of a `backUrl` override and the shape of `signupProgress`). *This is an explanation only — no code was changed.*
