# Root-Cause: Unpredictable "Back" Button in the wp-calypso Legacy Signup Flow (`/start`)

> **Repository:** `Automattic/wp-calypso`
>
> **Answer-file name** is derived from the **source branch** `wp-calypso_be7e5cc64162`, which points at commit `be7e5cc641622d153040491fd5625c6cb83e12eb`. All investigated source files are byte-identical to that commit, so the code observed here **is** the source at `be7e5cc641`.
> **Execution** was performed on the assigned Blitzy platform branch `blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385`, which is the `be7e5cc641` base **plus** documentation-only commits that add this one document (no product-source file changes versus `be7e5cc641`; verified in §11.4). The filename therefore encodes the source branch, while the run happened on the platform branch — the difference is intentional and expected. (See §11.5.)
> **Scope:** Read-only investigation. The only file added to the repository is this document. All temporary observation scripts (a jest harness plus its custom jest config and module resolver) were created **entirely outside the checkout**, under `/tmp/blitzy_obs/`, and were removed afterward; the repository was never written to except for this document. §11.4 shows the final clean `git status`.

## 1. Summary (direct answer)

The back destination for a given legacy-signup step is decided by a single method — **`NavigationLink.getBackUrl()`** at `client/signup/navigation-link/index.jsx:78-115` — whose return value becomes the Back control's anchor `href` (assigned to `hrefUrl` at `client/signup/navigation-link/index.jsx:183-186` and rendered onto `<Button … href={ hrefUrl } … >` at `:192`). Because the legacy signup step render never wires a `goToPreviousStep` handler (`client/signup/main.jsx:798-814` passes only `goToNextStep` at `:805` and `goToStep` at `:806`; a repository grep finds **no** `goToPreviousStep` in `main.jsx`), the click handler's back branch (`client/signup/navigation-link/index.jsx:125-127`) is **unwired in this legacy production caller** (it is reachable code — it fires when the prop _is_ supplied, and the co-located test supplies a `jest.fn()` — but the signup render never supplies it). The computed **`href`** is therefore what actually drives back navigation.

`getBackUrl()` evaluates its inputs in a **fixed precedence** (the order of statements in the method) and the first satisfied branch wins:

1. **Component prop `backUrl` (the override)** — `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85`. This early return short-circuits **before** any flow-position logic runs, so a truthy `backUrl` always wins. Note that one common _source_ of this prop is a **query argument**: `?back_to=/…` is transformed into the `backUrl` prop by `StepWrapper`'s `connect()` (`client/signup/step-wrapper/index.jsx:273-283`). So a _specific, special_ query argument (`back_to`) **can** decide the destination — by being promoted to the highest-precedence prop before `getBackUrl` runs.
2. **Flow position** — otherwise `getPreviousStep()` (`:47-76`) computes the previous step from `signupProgress` + the current `stepName`, and `getStepUrl()` (`client/signup/utils.js:45-69`) builds the URL.
3. **Ordinary query-string arguments** — every _other_ query argument only _decorates_ the final URL (appended via `addQueryArgs`, `client/signup/utils.js:68`); ordinary query args never change _which_ step is targeted.

The apparent randomness is **not random**. For a **fixed, complete input/environment tuple** the destination is deterministic (§8.5 shows byte-identical output across repeated runs). The tuple is larger than four values, however: it includes `direction`, `backUrl`, `flowName`, `signupProgress` (and each progressed step's `lastKnownFlow`, `stepSectionName`, `wasSkipped`), the current `stepName`, `userLoggedIn`, `queryParams` (with a `window.location.search` fallback), the resolved locale (`getLocaleSlug()`), `window.location.pathname` (which selects the `/start` vs `/setup` framework prefix), and the **active feature-config flags + excluded steps** that shape what `flows.getFlow()` returns (§8.1). The user-perceived unpredictability is the interaction of these deterministic inputs across different accumulated real-world states — most sharply the two "surprise" outcomes: a `{ stepName: null }` previous-step result builds the flow-root URL (**snaps to the first step**), and a previously-progressed step's foreign `lastKnownFlow` redirects the URL into another flow (**slips into a different flow**).

---

## 2. The user's report (verbatim)

> "The Back button is supposed to move one step backward, but every so often it snaps straight to the first step or slips out into an entirely different flow, and it never feels truly random."

This document answers, by name and with observed evidence, the six questions this report raises:

- **REQ-1** — What actually decides the destination for a given step? (§4)
- **REQ-2** — Which inputs win when flow position, component props, and query-string args disagree? (§5)
- **REQ-3** — Where does the external back target (the "quiet override") come from? (§6)
- **REQ-4** — What precedence rule lets that override take control? (§5)
- **REQ-5** — What step-by-step code path is being bypassed? (§7)
- **REQ-6** — What is the computed/rendered destination for each step position? (§8)

---

## 3. Methodology & environment

Per the governing rule (**SWE-AtlasQnA-Repo**), this investigation **ran the code first** and the answers below are grounded in captured runtime output plus exact `file:line` references naming the function/method that performs the work. Statements reasoned from source rather than executed at runtime are labeled **[inferred]**.

### 3.1 Toolchain (observed)

| Component       | Value                                                                   | Source                   |
| --------------- | ----------------------------------------------------------------------- | ------------------------ |
| Node.js         | `v22.23.1` (satisfies repo `engines: ^v22.9.0`; `.nvmrc` pins `22.9.0`) | `node --version` (§11.1) |
| Package manager | `yarn 4.0.2` via corepack (`packageManager: yarn@4.0.2`)                | `yarn --version` (§11.1) |
| Test runner     | `jest@29.7.0` + `@testing-library/react@16.2.0` (existing devDeps)      | `package.json`           |
| Router          | `@automattic/calypso-router@0.7.0` (a page.js fork)                     | `package.json`           |

`node_modules` was already present (no install was required). The canonical client-test invocation is `TZ=UTC jest -c=test/client/jest.config.js <path>`, exposed as the `test-client` script; the bare `yarn jest <path>` form is **not** used (with no root jest config, jest's default `testMatch` does not match the repo's `test/index.jsx` convention). The client jest environment runs with `NODE_ENV=test`, so `@automattic/calypso-config` loads `config/test.json`, and `jsdom` sets `window.location` to `https://example.com` (pathname `/`), which is why `getStepUrl` resolves the `/start` framework prefix (§7.1).

### 3.2 Observation vehicles: the CANONICAL pre-existing test and a NON-CANONICAL outside-checkout harness

This document draws on **two** runtime vehicles, and it is careful to keep their proof boundaries distinct.

**(a) The CANONICAL vehicle — the pre-existing configured test.** The only evidence labeled **CANONICAL** in this document is the repository's own co-located test `client/signup/navigation-link/test/index.jsx`, executed through the repository's real, unmodified test configuration via `CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false` (§11.2). This is the canonical entry point because it is an existing repository test run under the repository's canonical client-jest configuration — nothing about it is authored for this investigation. **It is, however, limited in what it proves:** it `jest.mock`s `calypso/signup/utils` (`client/signup/navigation-link/test/index.jsx:7-12`) and renders the **unconnected** `NavigationLink`, so it proves the _call arguments_ passed to `getStepUrl` and the literal `href` for the `backUrl`-override case — but it does **not** compute real `/start/…` URLs, resolve `StepWrapper` query state, or exercise routing. Both runs report `16 passed, 16 total`, `EXIT_CODE=0` (§11.2).

**(b) The NON-CANONICAL vehicle — a custom outside-checkout harness that runs the real modules.** To observe the concrete `/start/…` URLs the canonical test does not compute, a custom jest harness was created **entirely outside the checkout**, under `/tmp/blitzy_obs/` (harness `backnav.obs.jsx`, a custom `jest.config.cjs`, and a custom module `resolver.cjs`; full sources in §11.3). The custom jest config **extends** the repository's own `test/client/jest.config.js` (real babel transform, real `calypso/…` module-alias resolution, real feature-config loading, `jsdom` environment) with `rootDir` pointed at the repo's `client/` and a `testMatch` for `/tmp/blitzy_obs/**/*.obs.jsx`; the custom resolver wraps the repo's calypso-jest resolver and, on a resolution miss, retries with `basedir=<repo>/client` so that `@babel/runtime`, `calypso/*`, and other `node_modules` packages resolve correctly for a harness file that physically lives in `/tmp`. **Even though this harness imports and executes the real, unmodified repository modules** — `@automattic/calypso-config` (`isEnabled`), `calypso/signup/config/flows` (`flows.getFlow`), the real `calypso/signup/utils` (`getBackUrl → getPreviousStep → getStepUrl / isFirstStepInFlow / getFilteredSteps`), the **unconnected** `NavigationLink` named export, and the **connected** `StepWrapper` default export over a real Redux store with a real `setRoute()` — it is a **custom, direct-import harness and is therefore NON-CANONICAL** under this checkpoint's explicit proof boundary (a value obtained through any custom harness, direct named-component import, or custom connected-component render is non-canonical; only the pre-existing configured test is canonical). Every result produced by this harness — including the connected-`StepWrapper` `back_to → backUrl` renders — is labeled **NON-CANONICAL** throughout §8 and §11.3. Nothing in the decider (`getBackUrl`/`getPreviousStep`) or the URL builder (`getStepUrl`) is re-implemented or mocked; the harness only supplies inputs and reads the rendered anchor `href`.

**A further disclosed deviation within the NON-CANONICAL harness.** When the harness renders the **unconnected** `NavigationLink` (§8.2 scenarios A/B/D/E/F, §8.3 visibility), it supplies `userLoggedIn` and `signupProgress` as explicit props rather than through the Redux `connect()` wrapper (`client/signup/navigation-link/index.jsx:204-215`), which merely injects `isUserLoggedIn(state)` and `getSignupProgress(state)`. Bypassing that wrapper is a second reason those rows are non-canonical; the decision logic and URL construction that produce the `href` are nonetheless the real functions. The connected-`StepWrapper` renders (§8.2 C/G, §8.4) do use the real connected component and store, but remain **NON-CANONICAL** because they are still driven by a custom harness rather than the pre-existing configured test.

**Runtime click dispatch is not executed here.** Neither vehicle simulates a page.js click; the harness only reads the computed `href`. The claim that clicking a same-origin, non-external Back link is intercepted and dispatched client-side is therefore **[inferred]** from the router source (§4.2), not executed in jsdom.

### 3.3 Reproducing the reported intermittency (same input, repeated)

For a "sometimes X, sometimes Y" report, the **same unchanged input** was run repeatedly and the distribution reported:

- The NON-CANONICAL outside-checkout harness was executed **twice** as separate OS processes; its clean observation capture was **byte-identical** across both runs (`sha256` match and empty `diff`, §11.1/§8.5).
- The CANONICAL pre-existing unit test was executed **twice**; both runs reported `16 passed, 16 total`, `EXIT_CODE=0` (§11.2).

The observed distribution is therefore **100% identical across runs** — i.e. the destination is deterministic for a fixed complete input tuple, and the perceived randomness comes from variation in that tuple across sessions, not from nondeterminism in the code (§8.5).

### 3.4 The active flow reality (why `/start/onboarding` is the wrong entry to model)

Two facts about the **active** configuration materially change any per-step analysis, and both were confirmed at runtime (§8.1):

- **`signup/social-first` is enabled**, so the first "token" step of the onboarding-shaped flows is **`user-social`**, not `user` (`client/signup/config/flows-pure.js:13-14` `getUserSocialStepOrFallback`; the flag is `true` in `config/test.json`, `config/production.json`, and every other environment config).
- For a **logged-in** user, `flows.getFlow()` **removes** the token-providing step via `removeUserStepFromFlow` (`client/signup/config/flows.js:216-225,262-280`, filtering steps where `stepConfig[stepName].providesToken` is `true`; `user`/`user-social` both set `providesToken: true` at `client/signup/config/steps-pure.js:112-136,138-162`). So the logged-in `onboarding` flow resolves to `['domains','plans']`, **not** `['user','domains','plans']`.

More decisively, the **`onboarding` flow itself no longer runs on `/start`**: the `/start` middleware chain (`client/signup/index.web.js:16-22`) runs `controller.redirectToFlow` (`:18`) **before** `controller.start` (`:20`), and `redirectToFlow` **redirects the `onboarding` flow to `/setup`** (`client/signup/controller.js:179-202`, guarded by `isOnboardingFlow(flowName)` and calling `getStepUrl(…, '/setup')` then `window.location.replace(url)`). Therefore this document models per-step legacy behavior on a **real, non-redirected legacy `/start` flow — `onboarding-pm`** (`client/signup/config/flows-pure.js:154-155`, steps `[ userSocialStep, 'domains', 'plans' ]`, no `forceLogin`, not matched by `isOnboardingFlow`) — and treats the `onboarding → /setup` redirect as an explicitly demonstrated fact rather than a legacy entry point (§8.6).

### 3.5 Read-only guarantee & cleanup

No existing source file was modified; no permanent tests were added; no dependencies were changed. All temporary observation scripts — the harness `backnav.obs.jsx`, its custom `jest.config.cjs`, its custom `resolver.cjs`, and the run logs — live **entirely under `/tmp/blitzy_obs/`, outside the checkout**, and are removed during cleanup (§11.4 shows the exact `rm -rf /tmp/blitzy_obs` command, the resulting clean `git status --porcelain`, and the `git diff` against the source-branch base `be7e5cc641` confirming the net change is **only** this one document). At no point was any file created inside the repository working tree other than this document.

---

## 4. REQ-1 — The decider

**Answer: `NavigationLink.getBackUrl()` at `client/signup/navigation-link/index.jsx:78-115` decides the destination**, and its return value is rendered as the Back control's anchor `href`.

The render wiring (`client/signup/navigation-link/index.jsx:183-200`, quoted without elision):

```js
const hrefUrl =
	this.props.direction === 'forward' && this.props.forwardUrl
		? this.props.forwardUrl
		: this.getBackUrl();
return (
	<Button
		primary={ primary }
		borderless={ borderless }
		className={ buttonClasses }
		href={ hrefUrl }
		onClick={ this.handleClick }
		rel={ this.props.rel }
	>
		{ backGridicon }
		{ text }
		{ forwardGridicon }
	</Button>
);
```

So for a back control (`direction === 'back'`), `href = this.getBackUrl()` (`:186`, `:192`).

### 4.1 Why the `href` — not a click handler — navigates

`handleClick` (`client/signup/navigation-link/index.jsx:117-132`) calls a JS navigation function on the **back** path only when `goToPreviousStep` is provided (quoted without elision):

```js
handleClick = () => {
	if ( this.props.direction === 'forward' ) {
		this.props.submitSignupStep(
			{ stepName: this.props.stepName },
			this.props.defaultDependencies
		);

		this.props.goToNextStep();
	} else if ( this.props.goToPreviousStep ) {
		this.props.goToPreviousStep();
	}

	if ( ! this.props.disabledTracksOnClick ) {
		this.recordClick();
	}
};
```

The `else if ( this.props.goToPreviousStep )` branch (`:125-127`) is **reachable code** — it fires whenever the prop is supplied, and the co-located test verifies exactly that (`should call goToPreviousStep() only when the direction is back and clicked`, `client/signup/navigation-link/test/index.jsx:145-152`, which supplies a `jest.fn()`). It is simply **unwired in this legacy production caller**: `goToPreviousStep` is never passed down the signup render chain.

- `StepWrapper.renderBack()` forwards whatever it received: `goToPreviousStep={ this.props.goToPreviousStep }` (`client/signup/step-wrapper/index.jsx:57`).
- The signup flow render in `client/signup/main.jsx:798-814` passes `goToNextStep={ this.goToNextStep }` (`:805`) and `goToStep={ this.goToStep }` (`:806`) to the step component — but **not** `goToPreviousStep`:

  ```text
  $ grep -n 'goToPreviousStep' client/signup/main.jsx
  $ echo "EXIT_CODE=$?"
  EXIT_CODE=1
  ```

  (Exit code `1` = `grep` found no matches, confirming `goToPreviousStep` never appears in `main.jsx`. Steps render `StepWrapper` internally and receive `signupDependencies` at `client/signup/main.jsx:809`, but never `goToPreviousStep`.)

Therefore `this.props.goToPreviousStep` is `undefined` inside `NavigationLink`, the back branch never fires in production, and the same-origin `href` produced by `getBackUrl()` is what actually drives back navigation. This is consistent with the pre-existing test asserting both halves independently (§11.2): `should call goToPreviousStep() only when the direction is back and clicked` (verifies the branch _when the prop is supplied_) and `should set a proper url as href prop when the direction is "back".` (verifies the `href` is computed from the back logic).

### 4.2 page.js anchor-click interception — scoped precisely

wp-calypso routes through `@automattic/calypso-router`, a fork of **page.js** (upstream: <https://github.com/visionmedia/page.js>). Its document-level click handler (`Page.prototype.clickHandler`, `packages/calypso-router/src/index.js:725`) intercepts a click and dispatches it client-side **only for eligible links**. It explicitly **bails out** (letting the browser perform a normal navigation) when any of the following hold:

- the anchor has `download` **or** `rel="external"` (`packages/calypso-router/src/index.js:776`);
- the `href` is a `mailto:` link (`:787`);
- the link is **cross-origin** — `if ( ! svg && ! this.sameOrigin( el.href ) ) return;` (`:800`), where `sameOrigin` compares protocol, hostname, and port (`Page.prototype.sameOrigin`, `:892-900`).

Consequences for the Back control:

- For a **same-origin, non-external** `href` (the common case — a `/start/…` path built by `getStepUrl`), the click is [inferred] intercepted and dispatched client-side. This document did not execute a page.js click in jsdom, so the interception itself is source-grounded, not runtime-observed.
- `StepWrapper.renderBack()` sets `rel={ this.props.isExternalBackUrl ? 'external' : '' }` (`client/signup/step-wrapper/index.jsx:63`). When a step marks the back target external (e.g. the domains step sets `isExternalBackUrl = true` for the `'site'` source at `client/signup/steps/domains/index.jsx:1422`, and for the external-source override at `:1439`), the anchor gets `rel="external"` and page.js **does not** intercept it — the browser performs a full navigation.
- The `back_to` guard in `StepWrapper` (`backTo?.startsWith( '/' )`, `client/signup/step-wrapper/index.jsx:275`) is a **prefix check only**. It is **not** full URL validation, **not** a same-origin guarantee (a protocol-relative value such as `//evil.example/x` also starts with `/` and would be treated by the browser as cross-origin), and **not** authorization or access control. It merely decides whether the query-derived value is used as `backUrl` (§6). A computed Back URL is not "authorized" by starting with `/`.

**Named functions for REQ-1:** `NavigationLink.getBackUrl` (computes the destination) and `NavigationLink.render` (attaches it as the anchor `href`); `NavigationLink.handleClick` (the back branch that is inert because `goToPreviousStep` is never passed by the signup render).

---

## 5. REQ-2 & REQ-4 — Precedence, and the rule that lets the override win

**Answer:** The precedence is fixed by the _order of statements_ inside `getBackUrl()`. The **first** meaningful branch is an early return on the component prop `backUrl` (`client/signup/navigation-link/index.jsx:78-115`, quoted verbatim without elision — the annotations that follow are in prose, not in the source):

```js
getBackUrl() {
	if ( this.props.direction !== 'back' ) {
		return;
	}

	if ( this.props.backUrl ) {
		return this.props.backUrl;
	}

	const fallbackQueryParams = window.location.search
		? Object.fromEntries( new URLSearchParams( window.location.search ).entries() )
		: undefined;

	const {
		flowName,
		signupProgress,
		stepName,
		userLoggedIn,
		queryParams = fallbackQueryParams,
	} = this.props;
	const previousStep = this.getPreviousStep( flowName, signupProgress, stepName );

	const stepSectionName = get(
		this.props.signupProgress,
		[ previousStep.stepName, 'stepSectionName' ],
		''
	);

	const locale = ! userLoggedIn ? getLocaleSlug() : '';

	return getStepUrl(
		previousStep.lastKnownFlow || this.props.flowName,
		previousStep.stepName,
		stepSectionName,
		locale,
		queryParams
	);
}
```

Reading the branches in order: the **highest-precedence override** is the `if ( this.props.backUrl ) { return this.props.backUrl; }` early return at `:83-85` (REQ-4); the **flow-position** computation is `this.getPreviousStep( flowName, signupProgress, stepName )` at `:98`; and the **ordinary query args** (`queryParams`, with the `window.location.search` fallback at `:87-89`) are the final argument to `getStepUrl` at `:113`, so they only decorate the built URL (`:108-114`).

**The precedence rule (REQ-4):** the statement `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85` returns **before** `getPreviousStep()` is ever called (that call is at `:98`). A truthy `backUrl` prop therefore _unconditionally_ wins over flow position and over ordinary query args — this single early return is the entire "precedence rule" that lets the override take control. Note it returns the prop **verbatim**: no `getStepUrl` construction, no `addQueryArgs` decoration, no locale — which is why the observed override href is exactly `/home` (§8.2, scenario C) rather than a decorated `/start/…` URL.

**When the three inputs disagree (REQ-2):**

- **Component prop `backUrl`** beats everything (early return at `:83-85`). Observed via the NON-CANONICAL harness in scenario **C** (connected `StepWrapper`): with `?back_to=/home` resolved into `backUrl`, the rendered `href` is `/home` regardless of flow position (§8.2).
- **Flow position** (`getPreviousStep` at `:98`, `getStepUrl` at `:108-114`) decides the target step only when `backUrl` is falsy. Observed in scenario **A** (per-step) and **B/E** (§8.2).
- **Query-string arguments** split into two distinct roles:
  - The **special** argument **`back_to`** is _not_ an ordinary decorator: `StepWrapper.connect()` promotes it to the `backUrl` prop (`client/signup/step-wrapper/index.jsx:273-283`), so it enters at **precedence 1**. This is the mechanism by which a query argument _can_ choose the destination (§6). Observed in scenarios **C** (valid) and **G** (invalid, guarded out).
  - **All other** query arguments enter as `queryParams` (falling back to `window.location.search` at `:87-89`), are passed as the last argument to `getStepUrl` (`:113`), and are appended to the already-built path by `addQueryArgs` (`client/signup/utils.js:68`). They only _decorate_ the URL and never change which step is targeted. Observed in scenario **F**: `?ref=logged-out-homepage` appears on the URL but the target step is unchanged (§8.2).

### 5.1 Precedence rule table

| Precedence  | Input                           | Source                                                                                                                                                                                                                                                                                               | Effect on destination                                                                                                                                                           |
| ----------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 (highest) | Component prop `backUrl`        | Hardcoded step config (e.g. mailbox `backUrl: 'mailbox-domain/'` at `client/signup/config/steps-pure.js:399`); step-provided from `signupDependencies.back_to` (§6.2); or the **special `back_to` query arg** resolved in `StepWrapper` `connect()` (`client/signup/step-wrapper/index.jsx:273-283`) | Returned verbatim (`:83-85`); step-by-step logic bypassed; also forces the Back button onto the first step via `allowBackFirstStep` (`client/signup/step-wrapper/index.jsx:65`) |
| 2           | Flow position                   | `signupProgress` + current `stepName` via `getPreviousStep()` (`:47-76`) → `getStepUrl()`                                                                                                                                                                                                            | May be `{ stepName: null }` (→ flow root) or a specific progressed step; carries that step's `lastKnownFlow` (§7)                                                               |
| 3 (lowest)  | Ordinary query-string arguments | `queryParams` prop, or `window.location.search` fallback (`:87-89`)                                                                                                                                                                                                                                  | Only decorate the final built URL via `addQueryArgs`; never change the targeted step                                                                                            |

---

## 6. REQ-3 — The external override source

**Answer:** The external back target most commonly enters as the `?back_to=/…` query argument, resolved into the `backUrl` prop inside **`StepWrapper`'s `connect()` `mapStateToProps`** at `client/signup/step-wrapper/index.jsx:273-283` (quoted without elision):

```js
export default connect( ( state, ownProps ) => {
	const backToParam = getCurrentQueryArguments( state )?.back_to?.toString();
	const backTo = backToParam?.startsWith( '/' ) ? backToParam : undefined;

	const backUrl = ownProps.backUrl ?? backTo;

	return {
		backUrl,
		userLoggedIn: isUserLoggedIn( state ),
	};
} )( localize( StepWrapper ) );
```

- `:274` — reads `back_to` from the current query arguments (`getCurrentQueryArguments(state)` = `state.route.query.current`, `client/state/selectors/get-current-query-arguments.js:10`).
- `:275` — **GUARD:** a `back_to` value that does **not** start with `/` is discarded (`backTo` becomes `undefined`). As noted in §4.2, this is a prefix check, not full validation. This is scenario **G** below.
- `:277` — `const backUrl = ownProps.backUrl ?? backTo;` — a caller-provided `ownProps.backUrl` takes priority; otherwise the guarded query-derived `backTo` is used.

**Important asymmetry:** the `startsWith('/')` guard applies **only** to the query-derived `backTo`. A `backUrl` arriving as `ownProps.backUrl` — including a **dependency-derived** `back_to` that a step read from `signupDependencies` and passed down (§6.2) — **bypasses the guard entirely** via the `??` on `:277`.

The resolved `backUrl` (and the visibility flag) are then passed straight into the child (`client/signup/step-wrapper/index.jsx:50-70`, quoted without elision):

```js
renderBack() {
	if ( this.props.shouldHideNavButtons ) {
		return null;
	}
	return (
		<NavigationLink
			direction="back"
			goToPreviousStep={ this.props.goToPreviousStep }
			flowName={ this.props.flowName }
			positionInFlow={ this.props.positionInFlow }
			stepName={ this.props.stepName }
			stepSectionName={ this.props.stepSectionName }
			backUrl={ this.props.backUrl }
			rel={ this.props.isExternalBackUrl ? 'external' : '' }
			labelText={ this.props.backLabelText }
			allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }
			backIcon="chevron-left"
			queryParams={ this.props.queryParams }
		/>
	);
}
```

### 6.1 Why the override defeats the first-step suppression (and full visibility conditions)

The Back control is normally suppressed on the first step by `NavigationLink.render()` (`client/signup/navigation-link/index.jsx:154-161`, quoted without elision):

```js
if (
	this.props.positionInFlow === 0 &&
	this.props.direction === 'back' &&
	! this.props.stepSectionName &&
	! this.props.allowBackFirstStep
) {
	return null;
}
```

The control at `positionInFlow === 0` is hidden **unless** any of: a `stepSectionName` is present, or `allowBackFirstStep` is true. Because `StepWrapper.renderBack()` sets `allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }` (`client/signup/step-wrapper/index.jsx:65`), **a _truthy_ `backUrl` forces the Back button to render even on the first step**, and — via the precedence rule — its `href` is the override. That is exactly the "external back target treated like a quiet override even when the current step should not be eligible for it."

**The operative word is _truthy_, not merely _present_.** The `!!` in `!! this.props.backUrl` coerces the value, so a **falsy** `backUrl` does **not** force first-step visibility. The empty string is the important case: `StepWrapper`'s `const backUrl = ownProps.backUrl ?? backTo` (`client/signup/step-wrapper/index.jsx:277`) uses the **nullish** coalescing `??`, so an `ownProps.backUrl` of `''` is **not** nullish and is retained as `backUrl = ''` (it suppresses the query-derived `back_to` from being used) — yet `!! '' === false`, so `allowBackFirstStep` is **not** forced, and separately `getBackUrl`'s early return `if ( this.props.backUrl )` (`client/signup/navigation-link/index.jsx:83-85`) does **not** fire for `''`, so there is **no override** and the flow-position logic runs. This was observed at runtime: the empty-string `backUrl=''` row at `positionInFlow === 0` renders **`false`** (Back suppressed), while a truthy `backUrl='/home'` at the same position renders **`true`** (§8.3), and the connected `ownProps.backUrl=''` cross-product likewise falls through to flow-position logic (§8.4).

There is one more upstream gate: `renderBack()` returns `null` entirely when `shouldHideNavButtons` is true (`client/signup/step-wrapper/index.jsx:51-53`). All of these conditions (suppressed; forced by `allowBackFirstStep`; forced by a truthy `backUrl`; suppressed by an empty-string `backUrl`; forced by `stepSectionName`; and hidden by `shouldHideNavButtons`) were exercised via the **NON-CANONICAL** harness — see §8.3 and the §8.2/§8.4 scenario sets.

### 6.2 All `backUrl` origins (not only the AAP-named ones)

Beyond the `?back_to=` query path, several steps set `backUrl` (or its equivalents) directly, with **differing validation**. This inventory is drawn from a repository-wide search (§11.1):

| Origin                                                                  | Location                                                                                                                                                         | Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `?back_to=/…` query arg (slash-guarded)                                 | `client/signup/step-wrapper/index.jsx:274-277`                                                                                                                   | Guard applies only to the query-derived value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Hardcoded component prop                                                | `client/signup/config/steps-pure.js:399`                                                                                                                         | mailbox step `props: { backUrl: 'mailbox-domain/', … }`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Step-provided from `signupDependencies.back_to` (as `ownProps.backUrl`) | `client/signup/steps/difm-site-picker/index.tsx:43`; `client/signup/steps/new-or-existing-site/index.tsx:22,31`; `client/signup/steps/site-options/index.tsx:27` | Enters via `ownProps.backUrl`, **bypassing** the `startsWith('/')` guard (§6, `:277`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Email step default                                                      | `client/signup/steps/emails/index.jsx:122`                                                                                                                       | `backUrl = 'domains/'` default, passed to `StepWrapper` at `:144` with `allowBackFirstStep={ !! backUrl }` at `:152`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Domains step (many targets)                                             | `client/signup/steps/domains/index.jsx:1367-1440`                                                                                                                | Computes `backUrl` from a large `if/else` chain: `previousStepBackUrl` (`:1391-1392`), `domainManagementRoot()` (`:1394`), `/plugins` (`:1400`), `/themes` (`:1403`), a **flow-definition-based** `getStepUrl( flowName, previousStepName )` guarded by `'plans-first' === flowName` (`:1405-1406`), a site-editor `wp-admin` URL (`:1411`), `getStepUrl( flowName, stepName, null, this.getLocale() )` (`:1414`), a `/setup/onboarding/playground` URL (`:1417`), `siteUrl` with `isExternalBackUrl = true` (`:1420-1422`), `/home/${siteSlug}` (`:1424`), `/settings/general/${siteSlug}` (`:1427`), and an `externalBackUrl` from `getExternalBackUrl` (`:1434-1436`) that also sets `isExternalBackUrl = true` (`:1439`) |
| Domains external-source overrides                                       | `client/signup/steps/domains/utils.js:9-31`                                                                                                                      | `backUrlSourceOverrides` map + `getExternalBackUrl(source, sectionName)` (validated with `valid-url`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| WooCommerce install transfer                                            | `client/signup/steps/woocommerce-install/transfer/index.tsx:75`                                                                                                  | `backUrl={ \`/woocommerce-installation/${ domain }\` }`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |

The domains step in particular has its own **flow-definition-based** previous-step override (`getStepUrl( flowName, previousStepName )` at `client/signup/steps/domains/index.jsx:1406`, where `previousStepName` comes from `getPreviousStepName` — the flow-definition helper of §7.2), independent of `NavigationLink.getPreviousStep`.

**Named function for REQ-3:** the `connect()` `mapStateToProps` closure in `client/signup/step-wrapper/index.jsx` (the `back_to` → `backUrl` resolver), feeding `NavigationLink` via `StepWrapper.renderBack`; plus the per-step producers above.

---

## 7. REQ-5 — The bypassed step-by-step path

**Answer:** The step-by-step (flow-position) computation that a truthy `backUrl` skips is **`NavigationLink.getPreviousStep( flowName, signupProgress, currentStepName )`** at `client/signup/navigation-link/index.jsx:47-76`. When `backUrl` is truthy, `getBackUrl()` returns at `:83-85` and this method is never called (`:98`).

Full logic (quoted without elision):

```js
getPreviousStep( flowName, signupProgress, currentStepName ) {
	const previousStep = { stepName: null };

	if ( isFirstStepInFlow( flowName, currentStepName, this.props.userLoggedIn ) ) {
		return previousStep;
	}

	//Progressed steps will be filtered and sorted in relation to the steps definition of the current flow
	//Skipped steps are also filtered out
	const filteredProgressedSteps = getFilteredSteps(
		flowName,
		signupProgress,
		this.props.userLoggedIn
	).filter( ( step ) => ! step.wasSkipped );
	if ( filteredProgressedSteps.length === 0 ) {
		return previousStep;
	}

	//Find previous step in current relevant filtered progress
	const currentStepIndexInProgress = filteredProgressedSteps.findIndex(
		( step ) => step.stepName === currentStepName
	);

	// Current step isn't finished, so isn't part of the progress array yet, go to the top of the progress array.
	if ( currentStepIndexInProgress === -1 ) {
		return filteredProgressedSteps.pop();
	}

	return filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep;
}
```

Branch-by-branch (each mapped to observed evidence in §8.2):

- **Default** `previousStep = { stepName: null }` (`:48`). A `null` step name later builds the flow-root URL.
- **First-step guard** (`:50-52`): if `isFirstStepInFlow(...)`, return `{ stepName: null }`. (Note: at the first position the control is usually _not even rendered_ — §6.1 — so this branch's URL is rarely user-visible.)
- **Build progressed steps** (`:56-60`): `getFilteredSteps(...)` restricted to the flow's steps and to the current login state, then `.filter( step => ! step.wasSkipped )` drops skipped steps.
- **Empty relevant progress** (`:61-63`): if none remain, return `{ stepName: null }` → flow root. Observed via the connected `StepWrapper` fallthrough (empty store progress) in §8.2 scenario G/"no back_to".
- **Locate current step** (`:66-68`): `findIndex` by `stepName`.
- **Edge branch A — current step absent** (`:70-72`): `if ( currentStepIndexInProgress === -1 ) return filteredProgressedSteps.pop();` → snap to the **last** progressed step. This is the correct description of **partial, non-empty** progress whose current step is absent (scenario **B**, observed `/start/onboarding-pm/domains`).
- **Edge branch B — normal / index 0** (`:75`): `return filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep;` → the immediately-previous progressed step, or (at index 0) `{ stepName: null }` → flow root.

> **Correction of a common misstatement.** A `{ stepName: null }` result arises specifically from the **first-step guard**, **empty relevant progress**, or the **index-0** case. It is **not** the general outcome of "partial progress": partial, non-empty progress whose current step is _absent_ takes the `findIndex === -1` branch and returns `pop()` — i.e. the **last** progressed step — as scenario **B** demonstrates.

### 7.1 Supporting helpers (each named and cited)

**`isFirstStepInFlow`** — `client/signup/utils.js:28-31`:

```js
export function isFirstStepInFlow( flowName, stepName, isUserLoggedIn ) {
	const { steps: stepsBelongingToFlow } = flows.getFlow( flowName, isUserLoggedIn );
	return stepsBelongingToFlow.indexOf( stepName ) === 0;
}
```

**`getFilteredSteps`** — `client/signup/utils.js:137-150` (quoted without elision):

```js
export function getFilteredSteps( flowName, progress, isUserLoggedIn ) {
	const flow = flows.getFlow( flowName, isUserLoggedIn );

	if ( ! flow ) {
		return [];
	}

	return sortBy(
		// filter steps...
		filter( progress, ( step ) => includes( flow.steps, step.stepName ) ),
		// then order according to the flow definition...
		( { stepName } ) => flow.steps.indexOf( stepName )
	);
}
```

Both call `flows.getFlow( flowName, isUserLoggedIn )`, so the **login state and active config flags** feed directly into the previous-step computation — the same `removeUserStepFromFlow` / `social-first` filtering described in §3.4. This is why the logged-in vs logged-out per-step results differ (§8.2).

**`getStepUrl`** — `client/signup/utils.js:45-69` (builds the final URL, quoted without elision):

```js
export function getStepUrl(
	flowName,
	stepName,
	stepSectionName,
	localeSlug,
	params = {},
	frameworkParam = null
) {
	const flow = flowName ? `/${ flowName }` : '';
	const step = stepName ? `/${ stepName }` : '';
	const section = stepSectionName ? `/${ stepSectionName }` : '';
	const locale = localeSlug ? `/${ localeSlug }` : '';
	const framework =
		frameworkParam ||
		( typeof window !== 'undefined' && window.location.pathname.startsWith( '/setup' )
			? '/setup'
			: '/start' );

	const url =
		flowName === defaultFlowName && framework === '/start'
			? // we don't include the default flow name in the route in /start
			  framework + step + section + locale
			: framework + flow + step + section + locale;
	return addQueryArgs( params, url );
}
```

Two behaviors of `getStepUrl` matter for the observed URLs:

- **Framework prefix** (`:57-61`): `/setup` if `window.location.pathname` starts with `/setup`, else `/start`. In the jsdom test environment the pathname is `/`, so the prefix is `/start` (§3.1).
- **Default-flow omission** (`:63-67`): the flow segment is omitted from the path **only** when `flowName === defaultFlowName` (`'onboarding'`, `client/signup/config/flows.js:243-250`) **and** the framework is `/start`. For the **non-default** `onboarding-pm` flow used here, the flow segment **is** included — which is why the observed URLs are `/start/onboarding-pm/…` (§8.2). For a logged-out user, `getStepUrl` additionally appends the locale segment (`/en`), because `locale = ! userLoggedIn ? getLocaleSlug() : ''` in `getBackUrl` (`client/signup/navigation-link/index.jsx:106`) — observed in the logged-out rows of §8.2.

### 7.2 The `getPreviousStep` / `getPreviousStepName` duality (a documented confusion source)

There are **two** different "previous step" computations that can disagree:

- **`NavigationLink.getPreviousStep()`** — progress-based (`client/signup/navigation-link/index.jsx:47-76`), used to build the Back `href`.
- **`getPreviousStepName()`** — flow-definition-based (`client/signup/utils.js:85-88`):

  ```js
  export function getPreviousStepName( flowName, currentStepName, isUserLoggedIn ) {
  	const flow = flows.getFlow( flowName, isUserLoggedIn );
  	return flow.steps[ flow.steps.indexOf( currentStepName ) - 1 ];
  }
  ```

The progress-based method can carry a step's foreign `lastKnownFlow` (which rewrites the destination flow), while the definition-based helper only names a step within the _current_ flow. The domains step uses the definition-based helper for one of its `backUrl` branches (`client/signup/steps/domains/index.jsx:1406`). The _name_ of the previous step can match between the two while the _destination flow_ silently differs — observed in scenario **E**, where the previous step's `lastKnownFlow` (`onboarding-with-email`) rewrites the URL into a different flow (§8.2).

**Named functions for REQ-5:** `NavigationLink.getPreviousStep` (bypassed), with helpers `isFirstStepInFlow`, `getFilteredSteps`, `getStepUrl`, and the sibling `getPreviousStepName`.

---

## 8. REQ-6 — Per-step observation (NON-CANONICAL harness)

All destination values in this section were **captured at runtime** by the **NON-CANONICAL** outside-checkout harness (§3.2(b), source in §11.3), whose custom jest config **extends** the repository's real client-jest configuration (real babel transform, real `calypso/…` module resolution, real feature-config, `jsdom`). Because the harness is a custom, direct-import vehicle rather than the pre-existing configured test, **every value in §8 is NON-CANONICAL** — the destinations are real (the real decider and URL builder execute unmodified), but the entry point is not the canonical one. The exact commands and complete raw output are in §11; the clean capture (written by the harness and shown via `cat`) is quoted byte-faithfully in §11.3.

### 8.1 Active configuration and real flow resolution (observed)

Captured from `PART A` (real `@automattic/calypso-config` + real `flows.getFlow`):

```text
isEnabled('signup/social-first') | true
flows.getFlow('onboarding', loggedOut).steps | ["user-social","domains","plans"]
flows.getFlow('onboarding', loggedIn).steps | ["domains","plans"]
flows.getFlow('onboarding-pm', loggedOut).steps | ["user-social","domains","plans"]
flows.getFlow('onboarding-pm', loggedIn).steps | ["domains","plans"]
flows.defaultFlowName | "onboarding"
```

This confirms §3.4 at runtime: the token step is **`user-social`** (not `user`), and for a **logged-in** user the token step is **removed** (`onboarding` and `onboarding-pm` both resolve to `["domains","plans"]`). The default flow is `onboarding` — the flow that redirects to `/setup` (§8.6) — which is why the per-step demonstration below uses the real, non-redirected legacy flow **`onboarding-pm`**.

### 8.2 Observed per-step and per-scenario destinations (NON-CANONICAL harness)

The table separates **whether the Back control renders** (from `NavigationLink.render`, §6.1) from **the computed `href`** (from `getBackUrl`). Scenario **A** is the primary step-by-step path; **B–G** are the secondary/edge conditions. Scenarios **A, B, D, E, F** were captured via the unconnected `NavigationLink` with **real utils** — an entry point that additionally bypasses the Redux `connect()` wrapper (the disclosed deviation of §3.2), though the decision/URL logic that runs is the real, unmodified code; **C, G** were captured through the **connected `StepWrapper`** + real `setRoute` (the real `back_to` → `backUrl` path). **All rows are NON-CANONICAL** because they come from the custom harness, not the pre-existing configured test (§3.2).

| Scenario                                           | Condition                                                            | Step (position)                                                   | Back rendered?                 | Computed `href`                                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------------- |
| A (logged-in, flow `[domains,plans]`)              | step-by-step, full progress, no override                             | `domains` (0)                                                     | **No** — first-step suppressed | (computed root would be `/start/onboarding-pm`)                                        |
| A (logged-in)                                      | step-by-step, full progress, no override                             | `plans` (1)                                                       | Yes                            | `/start/onboarding-pm/domains`                                                         |
| A (logged-out, flow `[user-social,domains,plans]`) | step-by-step, full progress, no override                             | `user-social` (0)                                                 | **No** — first-step suppressed | —                                                                                      |
| A (logged-out)                                     | step-by-step, full progress, no override                             | `domains` (1)                                                     | Yes                            | `/start/onboarding-pm/user-social/en`                                                  |
| A (logged-out)                                     | step-by-step, full progress, no override                             | `plans` (2)                                                       | Yes                            | `/start/onboarding-pm/domains/en`                                                      |
| B (logged-in)                                      | current step **absent** from progress (`findIndex === -1` → `pop()`) | `plans` (absent)                                                  | Yes                            | `/start/onboarding-pm/domains` (snaps to last progressed)                              |
| C (connected `StepWrapper`)                        | valid `?back_to=/home` (starts with `/`)                             | `domains`                                                         | Yes (forced by `backUrl`)      | `/home` (override; `getPreviousStep` bypassed)                                         |
| D (logged-in)                                      | hardcoded component `backUrl` (mailbox real value)                   | mailbox step                                                      | Yes (forced by `backUrl`)      | `mailbox-domain/` (returned verbatim)                                                  |
| E (logged-in)                                      | previous step carries a different `lastKnownFlow`                    | `plans` (prev `domains`, `lastKnownFlow='onboarding-with-email'`) | Yes                            | `/start/onboarding-with-email/domains` (slips into another flow)                       |
| F (logged-in)                                      | ordinary query arg carried into the built URL                        | `plans`                                                           | Yes                            | `/start/onboarding-pm/domains?ref=logged-out-homepage`                                 |
| G (connected `StepWrapper`)                        | invalid `?back_to=home` (no leading `/`) → guard discards it         | `domains`                                                         | Yes                            | `/start/onboarding-pm/en` (override ignored; **falls through** to flow-position logic) |

**Canonicality of these rows — all NON-CANONICAL.** Rows **A, B, D, E, F** were captured through the _unconnected_ `NavigationLink`, which additionally bypasses the Redux `connect()` wrapper (see §3.2). The real `getBackUrl → getPreviousStep → getStepUrl / isFirstStepInFlow / getFilteredSteps` decision and URL logic still executes unmodified, so the destinations are real, but the render entry point is not the production one. Rows **C** and **G** were captured through the **connected** `StepWrapper` over a real Redux store with a real `setRoute`, exercising the real `back_to → backUrl` resolution end-to-end. Both groups are **NON-CANONICAL**: they are produced by the custom outside-checkout harness (§3.2(b)), not by the pre-existing configured test `client/signup/navigation-link/test/index.jsx`, which is the only CANONICAL vehicle (§11.2).

Key reads from this table:

- **REQ-6 / the "snap to first step":** at the first position the control is **suppressed** (rows `domains(0)` logged-in and `user-social(0)` logged-out show _Back rendered? No_). The `{ stepName: null }` → flow-root behavior is therefore mostly visible not at the first step itself, but when _earlier_ progress is empty/absent or `back_to`/`lastKnownFlow` interacts (scenarios G and the empty-progress fallthrough). The old conflation of "computed URL" with "what renders" at position 0 is corrected here.
- **Precedence (REQ-2/REQ-4):** scenario **C** shows the `backUrl` override producing `/home` verbatim; scenario **F** shows an ordinary query arg only decorating `/start/onboarding-pm/domains`; scenario **G** shows the guard discarding an invalid `back_to` so flow-position logic runs.
- **The two "surprises":** scenario **E** is the cross-flow slip (`/start/onboarding-with-email/domains`); the flow-root/first-step behavior is the `{ stepName: null }` path of §7.

### 8.3 Visibility conditions (NON-CANONICAL harness)

Captured from `PART C` of the harness capture (unconnected `NavigationLink` at `positionInFlow === 0`); shown byte-faithfully from §11.3:

```text
== PART C: visibility at first step (position 0) ==
first-step, no override/section | domains(0) | false | (expect suppressed)
first-step + allowBackFirstStep | domains(0) | true | (expect visible)
first-step + truthy backUrl | domains(0) | true | "/home"
first-step + stepSectionName | domains(0) | true | (expect visible)
first-step + empty-string backUrl='' | domains(0) | false | (expect suppressed: !!"" is false)
```

These directly exercise the conditions of the `render()` suppression block (`client/signup/navigation-link/index.jsx:154-161`): the control is suppressed by default at position 0, and forced visible by `allowBackFirstStep`, by a **truthy** `backUrl`, or by a `stepSectionName`. Critically, the last row confirms the §6.1 correction — an **empty-string** `backUrl=''` is **falsy**, so it does **not** force visibility (`rendered=false`); "truthy," not "present," is the operative condition. The `shouldHideNavButtons` gate (`client/signup/step-wrapper/index.jsx:51-53`) is exercised separately through the connected `StepWrapper` in §8.4 (`PART D`).

### 8.4 Additional mandatory conditions — override cross-product, dependency chain, default-flow omission, query encoding (NON-CANONICAL harness)

The harness exercises every remaining condition the six questions imply. All output below is quoted **byte-faithfully** from the single capture in §11.3 (same run, same file); each block names the exact code path it exercises. Every row here is **NON-CANONICAL** (custom harness, §3.2(b)).

**PART D — `StepWrapper.connect()` `back_to → backUrl` resolution (connected render) + `shouldHideNavButtons`.** Real Redux store, real `setRoute(path, { back_to })`, read by `getCurrentQueryArguments` and resolved at `client/signup/step-wrapper/index.jsx:274-277`:

```text
== PART D: StepWrapper connect() back_to -> backUrl (NON-CANONICAL connected render) ==
C valid back_to (starts with /) | {"back_to":"/home"} | true | "/home"
G invalid back_to (no leading /) | {"back_to":"home"} | true | "/start/onboarding-pm/en"  (guard discards -> flow-position logic)
no back_to (empty progress fallthrough) | {} | true | "/start/onboarding-pm/en"
shouldHideNavButtons=true | {"back_to":"/home"} | false | (expect back NOT rendered)
```

- **C** — a valid `/`-prefixed `back_to` passes the `startsWith('/')` guard (`:275`) and becomes the `backUrl` override, so `getBackUrl` returns `/home` verbatim.
- **G** — `home` (no leading `/`) is discarded by the guard (`backTo → undefined`), so the flow-position logic runs, yielding the flow root `/start/onboarding-pm/en`.
- **no `back_to`** — an empty query with empty progress produces no override; `getPreviousStep` yields the flow root.
- **`shouldHideNavButtons=true`** — `renderBack()` returns `null` before any `NavigationLink` renders (`client/signup/step-wrapper/index.jsx:51-53`).

**PART E — protocol-relative `back_to` (security-relevant).** A `//evil.example/x` value **passes** the `startsWith('/')` prefix check (§4.2 emphasizes this is a prefix check, not full validation), so it is accepted as the override:

```text
== PART E: protocol-relative back_to //evil.example/x (connected) ==
protocol-relative back_to | {"back_to":"//evil.example/x"} | true | "//evil.example/x"  (passes startsWith("/") guard -> override)
```

Any leading `/`, including the protocol-relative `//host` form, satisfies the guard at `client/signup/step-wrapper/index.jsx:275`.

**PART F — `ownProps.backUrl` vs query `back_to` (the `??` cross-product).** With a query `back_to=/query-home` present, the connected resolver `const backUrl = ownProps.backUrl ?? backTo` (`client/signup/step-wrapper/index.jsx:277`) is exercised across truthy / null / empty-string `ownProps.backUrl`:

```text
== PART F: ownProps.backUrl vs query back_to (connected, query back_to=/query-home) ==
ownProps truthy vs query | '/dependency-home' | true | "/dependency-home"  (ownProps ?? backTo -> ownProps wins, bypasses guard)
ownProps null vs query | null | true | "/query-home"  (null is nullish -> query back_to used)
ownProps '' vs query (pos 1) | '' | true | "/start/onboarding-pm/en"  ('' not nullish -> backUrl=''; falsy -> no override, flow logic)
ownProps '' vs query (pos 0) | '' | false | (expect suppressed: allowBackFirstStep = !!"" is false)
```

- **truthy `ownProps.backUrl`** wins over the query value **and bypasses the `startsWith('/')` guard** (the guard applies only to the query-derived `backTo`; §6).
- **null `ownProps.backUrl`** is nullish, so `??` falls through to the guarded query `back_to` (`/query-home`).
- **empty-string `ownProps.backUrl`** is **not** nullish, so `??` retains `backUrl=''` (suppressing the query value) — but `''` is falsy, so `getBackUrl` does not early-return and the flow-position logic runs (pos 1 → `/start/onboarding-pm/en`); at pos 0 the Back control is **suppressed** because `allowBackFirstStep = !!'' = false`. This is the runtime evidence for the §6.1 correction.

**PART G — real step-provided dependency override chain.** The actual `NewOrExistingSiteStep` (`client/signup/steps/new-or-existing-site/index.tsx:22,31`) reads `const { back_to: backUrl } = signupDependencies` and passes it to `StepWrapper` as `ownProps.backUrl`, bypassing the query guard:

```text
== PART G: real NewOrExistingSiteStep signupDependencies.back_to -> backUrl ==
real step dependency back_to | '/dependency-home' | true | "/dependency-home"  (step reads signupDependencies.back_to -> ownProps.backUrl, bypasses guard)
```

This is the third `backUrl` origin of §6.2 (`signupDependencies.back_to`) observed end-to-end through the real step component.

**PART H — default-flow (`onboarding`) URL omission on `/start`.** `getStepUrl` omits the flow segment when `flowName === defaultFlowName && framework === '/start'` (`client/signup/utils.js:63-67`):

```text
== PART H: default-flow (onboarding) omission on /start ==
getStepUrl('onboarding','domains','','') | "/start/domains"  (default flow segment omitted)
getStepUrl('onboarding',null,'','') | "/start"  (flow root)
getStepUrl('onboarding-pm','domains','','') | "/start/onboarding-pm/domains"  (non-default: flow segment kept)
rendered NavigationLink default flow | onboarding plans(1) | true | "/start/domains"  (default-flow omission in href)
```

For the default `onboarding` flow the flow name is omitted (`/start/domains`; root `/start`); a non-default flow keeps its segment (`/start/onboarding-pm/domains`). The last row shows the omission flowing through a real rendered `NavigationLink` `href`. (`onboarding` itself is redirected to `/setup` before it renders on `/start`, §8.6; this block isolates the URL-builder behavior of `getStepUrl`.)

**PART I — Unicode / metacharacter query encoding.** `getStepUrl` appends query args via `addQueryArgs` (`client/signup/utils.js:68`), which percent-encodes reserved and non-ASCII characters:

```text
== PART I: unicode/metacharacter query encoding (addQueryArgs) ==
unicode/metachar query | plans | true | "/start/onboarding-pm/domains?ref=a%26b%3Dc&q=%3Cscript%3Ealert%281%29%3C%2Fscript%3E&u=caf%C3%A9%E2%98%95"
```

`&`/`=` inside a value become `%26`/`%3D`, `<script>alert(1)</script>` is fully percent-encoded (inert as a URL query), and `café☕` is UTF-8 percent-encoded. The query only _decorates_ the URL — it never changes the target step.

**PART J — `window.location.search` fallback.** When no `queryParams` prop is supplied, `getBackUrl` falls back to `window.location.search` (`client/signup/navigation-link/index.jsx:87-96`, `queryParams = fallbackQueryParams`):

```text
== PART J: window.location.search fallback (no queryParams prop) ==
search fallback | window.location.search='?fallback=from-search' | true | "/start/onboarding-pm/domains?fallback=from-search"
```

With `queryParams` omitted, the `?fallback=from-search` from `window.location.search` is carried into the built URL.

### 8.5 Repeated-run determinism (reproducing the "never truly random" claim)

The **same unchanged input** was run repeatedly:

- **NON-CANONICAL outside-checkout harness, two separate OS processes:** the clean observation capture was **byte-identical** — verified by `sha256sum` (both `37156c442a3a83c2e7c844fb64d7f63d3fde34a5da29239e7dc22750dc863f2a`) and an empty `diff` (§11.1).
- **CANONICAL pre-existing test, two runs:** both `16 passed, 16 total`, `EXIT_CODE=0` (§11.2).

**Conclusion:** for a fixed, complete input/environment tuple (§1, §3.3) the back destination is **deterministic** — the observed distribution across runs is 100% identical. The perceived unpredictability is the interaction of these deterministic inputs across different accumulated real-world states (whether a `back_to` was present and valid, whether the current step is in progress, what `lastKnownFlow` a prior step recorded, login state, locale, and the active flow config).

### 8.6 The `onboarding → /setup` redirect (explicitly demonstrated, not assumed)

`/start/onboarding` does **not** render as a legacy step; it is redirected to the modern `/setup` framework before the legacy `start` controller runs. The `/start` route registration wires the middleware in this order (`client/signup/index.web.js:16-22`, quoted verbatim); note that `controller.redirectToFlow` (`:18`) runs **before** `controller.start` (`:20`):

```js
controller.saveInitialContext,
controller.redirectWithoutLocaleIfLoggedIn,
controller.redirectToFlow,
controller.setSelectedSiteForSignup,
controller.start,
```

and `redirectToFlow` performs the redirect for the onboarding flow (`client/signup/controller.js:179-202`, quoted verbatim without elision):

```js
if ( isOnboardingFlow( flowName ) ) {
	setReferrerPolicy();
	let url =
		getStepUrl(
			flowName,
			getStepName( context.params ),
			getStepSectionName( context.params ),
			localeFromParams ?? localeFromStore,
			null,
			'/setup'
		) +
		( context.querystring ? '?' + context.querystring : '' ) +
		( context.hashstring ? '#' + context.hashstring : '' );

	if ( document.referrer ) {
		url = addQueryArgs( { start_ref: document.referrer }, url );
	}

	window.location.replace( url );
	// skip the rest to avoid the `page.redirect` call below.
	// Don't call next() here, we don't need the subsequent middlewares to run.
	// next();
	return;
}
```

`isOnboardingFlow` matches exactly the `onboarding` flow (`packages/onboarding/src/utils/flows.ts:102-104`). **[inferred at runtime]:** this document did not execute the full page.js controller pipeline (it needs a real `window.location.replace` navigation), so the redirect is grounded in the source above and in the runtime fact (§8.1) that `flows.getFlow('onboarding', …)` no longer yields the `['user','domains','plans']` shape assumed by a naive `/start/onboarding` model. The legacy per-step evidence therefore uses `onboarding-pm`, which is _not_ matched by `isOnboardingFlow` and therefore renders on `/start`.

---

## 9. Mapping the user's exact words to mechanisms

- **"snaps straight to the first step"** → `getPreviousStep()` returns `{ stepName: null }` via the first-step guard (`isFirstStepInFlow`), **empty relevant progress**, or the **index-0** case (`client/signup/navigation-link/index.jsx:47-76`) — **not** merely "partial progress" (see the §7 correction; partial progress with an absent current step instead `pop()`s to the last step, scenario B). Then `getStepUrl( …, null, … )` builds the flow-root URL (`client/signup/utils.js:45-69`). For the _default_ `onboarding` flow the root is `/start` (default-flow omission), but that flow redirects to `/setup` (§8.6); for a real legacy flow like `onboarding-pm` the root is `/start/onboarding-pm`. Observed via the connected-`StepWrapper` empty-progress fallthrough (scenario "no back_to", §8.4 PART D).

- **"slips out into an entirely different flow"** → the return statement uses the previous step's `lastKnownFlow`: `return getStepUrl( previousStep.lastKnownFlow || this.props.flowName, previousStep.stepName, … )` (`client/signup/navigation-link/index.jsx:108-114`). That `lastKnownFlow` is stamped onto every progressed step in `client/state/signup/progress/actions.js` (quoted without elision):

  ```js
  export function saveSignupStep( step ) {
  	return ( dispatch, getState ) => {
  		const lastKnownFlow = getCurrentFlowName( getState() );
  		const lastUpdated = Date.now();
  		dispatch( {
  			type: SIGNUP_PROGRESS_SAVE_STEP,
  			step: { ...step, lastKnownFlow, lastUpdated },
  		} );
  	};
  }
  ```

  (`saveSignupStep`: `lastKnownFlow` at `:117`, spread into the step at `:122`; `submitSignupStep`: `lastKnownFlow` at `:130`, spread at `:145`.) A step progressed under a _different_ flow therefore redirects the Back URL into that other flow. Observed as scenario **E → `/start/onboarding-with-email/domains`** (§8.2).

- **"never feels truly random"** → **Confirmed deterministic** (§8.5): identical complete inputs produce byte-identical destinations across repeated in-process and cross-process runs. The apparent randomness is the interaction of the full deterministic input tuple (§1) across different accumulated states.

---

## 10. Secondary framework note — the modern Stepper (`/setup`)

The repository has **two** onboarding frameworks. The **primary** subject of this analysis is the legacy signup framework at `/start` (class components, Redux `connect`, page.js `href` navigation via `NavigationLink` inside `StepWrapper`) — the three named inputs (flow position, component props, query-string arguments) map precisely onto its `NavigationLink.getBackUrl` precedence. The **modern Stepper** at `/setup` is a parallel system with its **own** cross-flow hazard, in `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts`:

- The comment at `:47-53` explicitly warns that `previousStep` is persisted and can be a step from another flow:

  ```js
  /**
   * If the previous step is defined in the store, and the current step is not the first step, we can go back.
   * We need to make sure we're not at the first step because `previousStep` is persisted and can be a step from another flow or another run of the current flow.
   * …
   */
  ```

- `canUserGoBack` is the guard expression at `:54-58`:

  ```js
  const canUserGoBack =
  	stepData?.previousStep &&
  	currentStepRoute !== stepSlugs[ 0 ] &&
  	history.length > 1 &&
  	stepData.previousStep !== currentStepRoute;
  ```

- The default `goBack` handler calls `history.back()` (`:128`, within the `...( canUserGoBack && { … } )` handler at `:123-130`), and a flow-defined `goBack` overrides it at `:134-141` — the comment at `:131-133` states that the flow "is the ultimate authority on navigation."

This Stepper logic is acknowledged for completeness only; it cannot validate the legacy `/start` behavior. A downstream reader should confirm which framework the user's specific flow uses before generalizing; note that the default `onboarding` flow is redirected from `/start` to `/setup` (§8.6), so a user reporting this on the "onboarding" flow may in fact be on the Stepper.

---

## 11. Evidence appendix

Every displayed raw capture below is preceded by the exact command that produced it; explanatory prose is kept outside the fenced blocks.

### 11.1 Commands run (environment, searches, determinism, cleanup)

```text
# Environment
$ node --version
v22.23.1
$ yarn --version
4.0.2
# Immutable source-branch base under investigation (stable anchor; the platform HEAD advances as documentation-only commits land)
$ git rev-parse be7e5cc641
be7e5cc641622d153040491fd5625c6cb83e12eb
$ git branch --show-current
blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385

# REQ-1 proof: goToPreviousStep is never passed in the signup render
$ grep -n 'goToPreviousStep' client/signup/main.jsx ; echo "EXIT_CODE=$?"
EXIT_CODE=1

# backUrl origin inventory (REQ-3)
$ grep -rn "backUrl" client/signup/steps/ | grep -iv test
# (results summarized in §6.2)

# Determinism of the NON-CANONICAL outside-checkout harness (two separate OS processes)
$ sha256sum /tmp/blitzy_obs/obs_run1.txt /tmp/blitzy_obs/obs_run2.txt
37156c442a3a83c2e7c844fb64d7f63d3fde34a5da29239e7dc22750dc863f2a  /tmp/blitzy_obs/obs_run1.txt
37156c442a3a83c2e7c844fb64d7f63d3fde34a5da29239e7dc22750dc863f2a  /tmp/blitzy_obs/obs_run2.txt
$ diff /tmp/blitzy_obs/obs_run1.txt /tmp/blitzy_obs/obs_run2.txt ; echo "DIFF_EXIT=$?"
DIFF_EXIT=0
```

`DIFF_EXIT=0` with no output means the two runs are byte-identical.

### 11.2 CANONICAL pre-existing test — both runs (complete captured logs)

This is the **only CANONICAL** vehicle (§3.2(a)): the repository's own configured test, run twice with output redirected and exit code captured. The complete `cat` of each log is shown; the only element that differs between runs is jest's non-deterministic wall-clock timing text (the suite/test counts and exit code are identical).

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false \
    > /tmp/blitzy_obs/preexist_run1.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
```

Run #1 — complete captured log (`cat`; the `Browserslist` notice is the tool's own stderr):

```text
$ cat /tmp/blitzy_obs/preexist_run1.log
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/navigation-link/test/index.jsx (5.092 s)

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.376 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
```

Run #2 — complete captured log (`cat`; identical suite/test counts and exit code; only the wall-clock timing text differs — this run printed `(5.071 s)` on the `PASS` line and `estimated 6 s` on the `Time` line):

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false \
    > /tmp/blitzy_obs/preexist_run2.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ cat /tmp/blitzy_obs/preexist_run2.log
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/navigation-link/test/index.jsx (5.071 s)

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.348 s, estimated 6 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
```

Both runs: `EXIT_CODE=0`, `Tests: 16 passed, 16 total`. This test's built-in `getPreviousStep()` assertions map onto the branches in §7: `2nd → 1st`, `1st → nullish`, `3rd → 2nd`, `unknown → last in progress` (the `pop()` edge branch), `no flow steps → nullish`, and `skipped steps ignored`. Because it mocks `calypso/signup/utils` (`client/signup/navigation-link/test/index.jsx:7-12`), it proves _call arguments_ and the literal override `href`, not the concrete `/start/…` URLs — which is why the concrete URLs come from the NON-CANONICAL harness (§3.2).

### 11.3 NON-CANONICAL outside-checkout harness — sources and byte-faithful capture

The harness and its jest configuration live **entirely outside the checkout**, under `/tmp/blitzy_obs/`. Three files are reproduced below **complete and verbatim** (shown in `text` fences so their bytes — including the tab indentation — are preserved exactly, and so they are not reflowed by any later Markdown formatter): the module resolver `resolver.cjs`, the jest config `jest.config.cjs`, and the harness `backnav.obs.jsx`. No file was created or modified inside the repository working tree.

Command (run twice, as separate OS processes; the second run wrote `obs_run2.txt`, byte-identical per §11.1). It invokes the repository's **own local** jest (`node_modules/.bin/jest`, jest@29.7.0) with the out-of-checkout config; the per-run `PASS` timing is jest's non-deterministic wall clock (the suite/test counts and exit code are stable):

```text
$ BLITZY_OBS_OUT=/tmp/blitzy_obs/obs_run1.txt \
  CI=true TZ=UTC ./node_modules/.bin/jest \
  -c /tmp/blitzy_obs/jest.config.cjs /tmp/blitzy_obs/backnav.obs.jsx --ci --watchAll=false \
  > /tmp/blitzy_obs/jest_run1.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ grep -E 'PASS |Tests:' /tmp/blitzy_obs/jest_run1.log
PASS ../../../blitzy_obs/backnav.obs.jsx (5.971 s)
Tests:       1 passed, 1 total
```

`jest` prints the harness path as `../../../blitzy_obs/backnav.obs.jsx` because the config's `rootDir` is the repo's `client/` directory; the file physically lives at `/tmp/blitzy_obs/backnav.obs.jsx`.

Clean observation capture — the harness writes this file, shown **byte-faithfully** via `cat`. The file begins with the `##########` BEGIN marker on its first line (**no leading blank line**) and ends with exactly **one** trailing newline after the END marker; its `sha256` is `37156c442a3a83c2e7c844fb64d7f63d3fde34a5da29239e7dc22750dc863f2a` (identical across both runs, §11.1):

```text
$ cat /tmp/blitzy_obs/obs_run1.txt
########## BLITZY-OBSERVATION-BEGIN (NON-CANONICAL) ##########

== PART A: active config + real flows.getFlow ==
isEnabled('signup/social-first') | true
flows.getFlow('onboarding', loggedOut).steps | ["user-social","domains","plans"]
flows.getFlow('onboarding', loggedIn).steps | ["domains","plans"]
flows.getFlow('onboarding-pm', loggedOut).steps | ["user-social","domains","plans"]
flows.getFlow('onboarding-pm', loggedIn).steps | ["domains","plans"]
flows.defaultFlowName | "onboarding"

== PART B1: per-step, LOGGED-IN (onboarding-pm => [domains,plans]) ==
SCENARIO | STEP(position) | BACK RENDERED? | HREF
A(logged-in) | domains(0) | false | null
A(logged-in) | plans(1) | true | "/start/onboarding-pm/domains"

== PART B2: per-step, LOGGED-OUT (onboarding-pm => [user-social,domains,plans]) ==
SCENARIO | STEP(position) | BACK RENDERED? | HREF
A(logged-out) | user-social(0) | false | null
A(logged-out) | domains(1) | true | "/start/onboarding-pm/user-social/en"
A(logged-out) | plans(2) | true | "/start/onboarding-pm/domains/en"

== PART B3: edge scenarios B/D/E/F (logged-in) ==
B current-step-absent -> pop() | plans(absent) | true | "/start/onboarding-pm/domains"
D hardcoded backUrl | mailbox-domain/ | true | "mailbox-domain/"
E cross-flow lastKnownFlow | plans(prev domains@onboarding-with-email) | true | "/start/onboarding-with-email/domains"
F query-arg decoration | plans | true | "/start/onboarding-pm/domains?ref=logged-out-homepage"

== PART C: visibility at first step (position 0) ==
first-step, no override/section | domains(0) | false | (expect suppressed)
first-step + allowBackFirstStep | domains(0) | true | (expect visible)
first-step + truthy backUrl | domains(0) | true | "/home"
first-step + stepSectionName | domains(0) | true | (expect visible)
first-step + empty-string backUrl='' | domains(0) | false | (expect suppressed: !!"" is false)

== PART D: StepWrapper connect() back_to -> backUrl (NON-CANONICAL connected render) ==
SCENARIO | route.query.current | BACK RENDERED? | HREF
C valid back_to (starts with /) | {"back_to":"/home"} | true | "/home"
G invalid back_to (no leading /) | {"back_to":"home"} | true | "/start/onboarding-pm/en"  (guard discards -> flow-position logic)
no back_to (empty progress fallthrough) | {} | true | "/start/onboarding-pm/en"
shouldHideNavButtons=true | {"back_to":"/home"} | false | (expect back NOT rendered)

== PART E: protocol-relative back_to //evil.example/x (connected) ==
protocol-relative back_to | {"back_to":"//evil.example/x"} | true | "//evil.example/x"  (passes startsWith("/") guard -> override)

== PART F: ownProps.backUrl vs query back_to (connected, query back_to=/query-home) ==
SCENARIO | ownProps.backUrl | BACK RENDERED? | HREF
ownProps truthy vs query | '/dependency-home' | true | "/dependency-home"  (ownProps ?? backTo -> ownProps wins, bypasses guard)
ownProps null vs query | null | true | "/query-home"  (null is nullish -> query back_to used)
ownProps '' vs query (pos 1) | '' | true | "/start/onboarding-pm/en"  ('' not nullish -> backUrl=''; falsy -> no override, flow logic)
ownProps '' vs query (pos 0) | '' | false | (expect suppressed: allowBackFirstStep = !!"" is false)

== PART G: real NewOrExistingSiteStep signupDependencies.back_to -> backUrl ==
real step dependency back_to | '/dependency-home' | true | "/dependency-home"  (step reads signupDependencies.back_to -> ownProps.backUrl, bypasses guard)

== PART H: default-flow (onboarding) omission on /start ==
getStepUrl('onboarding','domains','','') | "/start/domains"  (default flow segment omitted)
getStepUrl('onboarding',null,'','') | "/start"  (flow root)
getStepUrl('onboarding-pm','domains','','') | "/start/onboarding-pm/domains"  (non-default: flow segment kept)
rendered NavigationLink default flow | onboarding plans(1) | true | "/start/domains"  (default-flow omission in href)

== PART I: unicode/metacharacter query encoding (addQueryArgs) ==
unicode/metachar query | plans | true | "/start/onboarding-pm/domains?ref=a%26b%3Dc&q=%3Cscript%3Ealert%281%29%3C%2Fscript%3E&u=caf%C3%A9%E2%98%95"

== PART J: window.location.search fallback (no queryParams prop) ==
search fallback | window.location.search='?fallback=from-search' | true | "/start/onboarding-pm/domains?fallback=from-search"

########## BLITZY-OBSERVATION-END (NON-CANONICAL) ##########
```

**Module resolver** `resolver.cjs` — reuses the repo's calypso-jest resolver and, on a resolution miss for a `/tmp` requester, retries with `basedir=<repo>/client` (verbatim):

```text
const path = require( 'path' );
const REPO = '/tmp/blitzy/wp-calypso/blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385_645bc6';
const REPO_CLIENT = path.join( REPO, 'client' );
// Reuse the repo's own calypso-jest resolver (enhanced-resolve honoring calypso:src/main).
const calypsoResolver = require( path.join( REPO, 'node_modules/@automattic/calypso-jest/src/module-resolver.js' ) );

module.exports = function ( request, options ) {
	try {
		return calypsoResolver( request, options );
	} catch ( e ) {
		// The harness lives outside the checkout (/tmp), so bare specifiers
		// (calypso/*, @babel/runtime, react, redux, @testing-library/*, ...) cannot
		// resolve from its /tmp basedir. Retry as if the requiring file lived in the
		// repo client/ dir, so resolution walks the repo node_modules + the
		// node_modules/calypso -> ../client symlink.
		return calypsoResolver( request, { ...options, basedir: REPO_CLIENT } );
	}
};
```

**Jest config** `jest.config.cjs` — **extends** the repository's `test/client/jest.config.js` (real babel transform, real feature-config, `jsdom`), pointing `rootDir` at the repo `client/` and matching `/tmp/blitzy_obs/**/*.obs.jsx` (verbatim):

```text
const path = require( 'path' );
const REPO = '/tmp/blitzy/wp-calypso/blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385_645bc6';
const base = require( path.join( REPO, 'test/client/jest.config.js' ) );
const assetTransform = require.resolve(
	path.join( REPO, 'node_modules/@automattic/calypso-jest/src/asset-transform.js' )
);

module.exports = {
	...base,
	rootDir: path.join( REPO, 'client' ),
	roots: [ path.join( REPO, 'client' ), '/tmp/blitzy_obs' ],
	testMatch: [ '/tmp/blitzy_obs/**/*.obs.jsx' ],
	cacheDirectory: '/tmp/blitzy_obs/.jestcache',
	resolver: '/tmp/blitzy_obs/resolver.cjs',
	transform: {
		'\\.[jt]sx?$': [
			'babel-jest',
			{ configFile: path.join( REPO, 'babel.config.js' ), root: REPO, rootMode: 'root' },
		],
		'\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': assetTransform,
	},
};
```

**Harness** `backnav.obs.jsx` — reproduced **complete and verbatim, no logic elided**. Two `jest.mock` calls isolate orthogonal side-effects only (an unrelated network `fetch` in `triggerGuidesForStep`, and the `DIFMLanding` content body), leaving the real `back_to → backUrl → StepWrapper → NavigationLink` chain and the real `getBackUrl`/`getStepUrl` logic intact:

```text
/** @jest-environment jsdom */
/*
 * BLITZY TEMPORARY OBSERVATION HARNESS — NON-CANONICAL.
 * Lives OUTSIDE the repository checkout (/tmp/blitzy_obs), run via an out-of-checkout
 * jest config (/tmp/blitzy_obs/jest.config.cjs) that reuses the repo's real babel
 * transform + a resolver wrapper so the REAL repository modules load:
 *   - real calypso/signup/utils (getBackUrl chain: getStepUrl/isFirstStepInFlow/getFilteredSteps)
 *   - real calypso/signup/config/flows (+ @automattic/calypso-config feature flags)
 *   - real UNCONNECTED NavigationLink (named export) with real utils
 *   - real CONNECTED StepWrapper (default export) over a real Redux store + real setRoute()
 *   - real step NewOrExistingSiteStep (dependency-derived back_to -> backUrl chain)
 * This is a custom harness (NOT the pre-existing configured test), so ALL results here
 * are NON-CANONICAL. It does NOT mock calypso/signup/utils.
 * Deleted after capture; the repository is left unchanged.
 */
// Isolate an UNRELATED network side-effect: the signup steps call triggerGuidesForStep()
// in a useEffect, which issues a real fetch to public-api.wordpress.com (blocked by nock in
// the test env). It is orthogonal to back-navigation; mocking only this keeps the REAL
// back_to -> backUrl -> StepWrapper -> NavigationLink chain intact.
// Isolate the step's CONTENT body (DIFMLanding) — it fetches site/plan state that is
// orthogonal to the Back control. Stubbing only the content keeps the REAL
// NewOrExistingSiteStep dependency read + real StepWrapper/NavigationLink chain intact;
// the Back href comes from StepWrapper.renderBack (backUrl), never from stepContent.
jest.mock( 'calypso/my-sites/marketing/do-it-for-me/difm-landing', () => () => null );

jest.mock( 'calypso/lib/guides/trigger-guides-for-step', () => ( {
	triggerGuidesForStep: () => {},
} ) );

import { isEnabled } from '@automattic/calypso-config';
import { render } from '@testing-library/react';
import fs from 'fs';
import { createStore, applyMiddleware } from 'redux';
import { thunk } from 'redux-thunk';
import flows from 'calypso/signup/config/flows';
import { getStepUrl } from 'calypso/signup/utils';
import { NavigationLink } from 'calypso/signup/navigation-link';
import StepWrapper from 'calypso/signup/step-wrapper';
import NewOrExistingSiteStep from 'calypso/signup/steps/new-or-existing-site';
import initialReducer from 'calypso/state/reducer';
// eslint-disable-next-line no-restricted-imports
import routeReducer from 'calypso/state/route/reducer';
import { setRoute } from 'calypso/state/route/actions';
import { renderWithProvider } from 'calypso/test-helpers/testing-library';

const translate = ( s ) => s;

const OUT = [];
function line( ...parts ) {
	const s = parts.join( ' | ' );
	OUT.push( s );
	// eslint-disable-next-line no-console
	console.log( s );
}
function blank() {
	OUT.push( '' );
	// eslint-disable-next-line no-console
	console.log( '' );
}

// progress object keyed by stepName (matches Redux signup.progress shape).
const mkStep = ( stepName, extra = {} ) => ( {
	stepName,
	stepSectionName: '',
	wasSkipped: false,
	...extra,
} );

// Render the UNCONNECTED NavigationLink with REAL utils; return {rendered, href}.
function observeNavLink( props ) {
	const { container, unmount } = render(
		<NavigationLink direction="back" translate={ translate } { ...props } />
	);
	const el = container.querySelector( '.navigation-link.back' );
	const result = { rendered: !! el, href: el ? el.getAttribute( 'href' ) : null };
	unmount();
	return result;
}

// Build a real store with the `route` slice registered (setRoute populates route.query.current).
function makeStore( path, query ) {
	const reducer = initialReducer.addReducer( [ 'route' ], routeReducer );
	const store = createStore( reducer, applyMiddleware( thunk ) );
	if ( query !== undefined ) {
		store.dispatch( setRoute( path, query ) );
	}
	return store;
}

// Render the CONNECTED StepWrapper (real connect()) over a real store; return {rendered, href, qa}.
function observeStepWrapper( { flowName, stepName, positionInFlow, path, query, ownBackUrl, shouldHideNavButtons } ) {
	const store = makeStore( path, query );
	const qa = store.getState()?.route?.query?.current;
	const extra = {};
	if ( ownBackUrl !== undefined ) {
		extra.backUrl = ownBackUrl;
	}
	const { container, unmount } = renderWithProvider(
		<StepWrapper
			flowName={ flowName }
			stepName={ stepName }
			positionInFlow={ positionInFlow }
			hideFormattedHeader
			shouldHideNavButtons={ shouldHideNavButtons }
			{ ...extra }
		/>,
		{ store }
	);
	const el = container.querySelector( '.navigation-link.back' );
	const result = { rendered: !! el, href: el ? el.getAttribute( 'href' ) : null, qa };
	unmount();
	return result;
}

// Render the REAL NewOrExistingSiteStep (dependency-derived back_to -> backUrl chain).
function observeRealStep( { flowName, stepName, positionInFlow, backTo } ) {
	const store = makeStore( `/start/${ flowName }/${ stepName }`, {} );
	const { container, unmount } = renderWithProvider(
		<NewOrExistingSiteStep
			flowName={ flowName }
			stepName={ stepName }
			positionInFlow={ positionInFlow }
			existingSiteCount={ 0 }
			goToNextStep={ () => {} }
			goToStep={ () => {} }
			submitSignupStep={ () => {} }
			signupDependencies={ { back_to: backTo } }
			translate={ translate }
		/>,
		{ store }
	);
	const el = container.querySelector( '.navigation-link.back' );
	const result = { rendered: !! el, href: el ? el.getAttribute( 'href' ) : null };
	unmount();
	return result;
}

describe( 'BLITZY NON-CANONICAL Back-button observation', () => {
	test( 'capture all conditions', () => {
		line( '########## BLITZY-OBSERVATION-BEGIN (NON-CANONICAL) ##########' );

		// ---- PART A: active configuration & real flow resolution ----
		blank();
		line( '== PART A: active config + real flows.getFlow ==' );
		line( "isEnabled('signup/social-first')", String( isEnabled( 'signup/social-first' ) ) );
		line( "flows.getFlow('onboarding', loggedOut).steps", JSON.stringify( flows.getFlow( 'onboarding', false ).steps ) );
		line( "flows.getFlow('onboarding', loggedIn).steps", JSON.stringify( flows.getFlow( 'onboarding', true ).steps ) );
		line( "flows.getFlow('onboarding-pm', loggedOut).steps", JSON.stringify( flows.getFlow( 'onboarding-pm', false ).steps ) );
		line( "flows.getFlow('onboarding-pm', loggedIn).steps", JSON.stringify( flows.getFlow( 'onboarding-pm', true ).steps ) );
		line( 'flows.defaultFlowName', JSON.stringify( flows.defaultFlowName ) );

		const FLOW = 'onboarding-pm';
		const progLoggedIn = { domains: mkStep( 'domains' ), plans: mkStep( 'plans' ) };
		const progLoggedOut = {
			'user-social': mkStep( 'user-social' ),
			domains: mkStep( 'domains' ),
			plans: mkStep( 'plans' ),
		};

		// ---- PART B1: per-step, LOGGED-IN ----
		blank();
		line( '== PART B1: per-step, LOGGED-IN (onboarding-pm => [domains,plans]) ==' );
		line( 'SCENARIO', 'STEP(position)', 'BACK RENDERED?', 'HREF' );
		[ [ 'domains', 0 ], [ 'plans', 1 ] ].forEach( ( [ step, pos ] ) => {
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: step, positionInFlow: pos } );
			line( 'A(logged-in)', `${ step }(${ pos })`, String( r.rendered ), JSON.stringify( r.href ) );
		} );

		// ---- PART B2: per-step, LOGGED-OUT ----
		blank();
		line( '== PART B2: per-step, LOGGED-OUT (onboarding-pm => [user-social,domains,plans]) ==' );
		line( 'SCENARIO', 'STEP(position)', 'BACK RENDERED?', 'HREF' );
		[ [ 'user-social', 0 ], [ 'domains', 1 ], [ 'plans', 2 ] ].forEach( ( [ step, pos ] ) => {
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: false, signupProgress: progLoggedOut, stepName: step, positionInFlow: pos } );
			line( 'A(logged-out)', `${ step }(${ pos })`, String( r.rendered ), JSON.stringify( r.href ) );
		} );

		// ---- PART B3: edge scenarios B/D/E/F (logged-in) ----
		blank();
		line( '== PART B3: edge scenarios B/D/E/F (logged-in) ==' );
		{
			const partial = { domains: mkStep( 'domains' ) }; // plans absent
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: partial, stepName: 'plans', positionInFlow: 1 } );
			line( 'B current-step-absent -> pop()', 'plans(absent)', String( r.rendered ), JSON.stringify( r.href ) );
		}
		{
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: 'plans', positionInFlow: 1, backUrl: 'mailbox-domain/' } );
			line( 'D hardcoded backUrl', 'mailbox-domain/', String( r.rendered ), JSON.stringify( r.href ) );
		}
		{
			const cross = {
				domains: mkStep( 'domains', { lastKnownFlow: 'onboarding-with-email' } ),
				plans: mkStep( 'plans', { lastKnownFlow: FLOW } ),
			};
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: cross, stepName: 'plans', positionInFlow: 1 } );
			line( 'E cross-flow lastKnownFlow', 'plans(prev domains@onboarding-with-email)', String( r.rendered ), JSON.stringify( r.href ) );
		}
		{
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: 'plans', positionInFlow: 1, queryParams: { ref: 'logged-out-homepage' } } );
			line( 'F query-arg decoration', 'plans', String( r.rendered ), JSON.stringify( r.href ) );
		}

		// ---- PART C: visibility conditions at first step (UNCONNECTED NavigationLink) ----
		blank();
		line( '== PART C: visibility at first step (position 0) ==' );
		{
			const base = { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: 'domains', positionInFlow: 0 };
			line( 'first-step, no override/section', 'domains(0)', String( observeNavLink( base ).rendered ), '(expect suppressed)' );
			line( 'first-step + allowBackFirstStep', 'domains(0)', String( observeNavLink( { ...base, allowBackFirstStep: true } ).rendered ), '(expect visible)' );
			const wb = observeNavLink( { ...base, backUrl: '/home', allowBackFirstStep: true } );
			line( 'first-step + truthy backUrl', 'domains(0)', String( wb.rendered ), JSON.stringify( wb.href ) );
			line( 'first-step + stepSectionName', 'domains(0)', String( observeNavLink( { ...base, stepSectionName: 'some-section' } ).rendered ), '(expect visible)' );
			// Empty-string backUrl: allowBackFirstStep=!!'' is false => Back stays HIDDEN at first step.
			const eb0 = observeNavLink( { ...base, backUrl: '', allowBackFirstStep: !! '' } );
			line( "first-step + empty-string backUrl=''", 'domains(0)', String( eb0.rendered ), '(expect suppressed: !!\"\" is false)' );
		}

		// ---- PART D: CONNECTED StepWrapper back_to resolution (C, G) + shouldHideNavButtons ----
		blank();
		line( '== PART D: StepWrapper connect() back_to -> backUrl (NON-CANONICAL connected render) ==' );
		line( 'SCENARIO', 'route.query.current', 'BACK RENDERED?', 'HREF' );
		{
			const c = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '/home' } } );
			line( 'C valid back_to (starts with /)', JSON.stringify( c.qa ), String( c.rendered ), JSON.stringify( c.href ) );
			const g = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: 'home' } } );
			line( 'G invalid back_to (no leading /)', JSON.stringify( g.qa ), String( g.rendered ), JSON.stringify( g.href ) + '  (guard discards -> flow-position logic)' );
			const none = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: {} } );
			line( 'no back_to (empty progress fallthrough)', JSON.stringify( none.qa ), String( none.rendered ), JSON.stringify( none.href ) );
			const hidden = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '/home' }, shouldHideNavButtons: true } );
			line( 'shouldHideNavButtons=true', JSON.stringify( hidden.qa ), String( hidden.rendered ), '(expect back NOT rendered)' );
		}

		// ---- PART E: protocol-relative back_to (prefix guard is not a same-origin guarantee) ----
		blank();
		line( '== PART E: protocol-relative back_to //evil.example/x (connected) ==' );
		{
			const pr = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '//evil.example/x' } } );
			line( 'protocol-relative back_to', JSON.stringify( pr.qa ), String( pr.rendered ), JSON.stringify( pr.href ) + '  (passes startsWith("/") guard -> override)' );
		}

		// ---- PART F: ownProps.backUrl vs query back_to conflict (?? nullish coalescing on :277) ----
		blank();
		line( '== PART F: ownProps.backUrl vs query back_to (connected, query back_to=/query-home) ==' );
		line( 'SCENARIO', 'ownProps.backUrl', 'BACK RENDERED?', 'HREF' );
		{
			const truthy = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '/query-home' }, ownBackUrl: '/dependency-home' } );
			line( 'ownProps truthy vs query', "'/dependency-home'", String( truthy.rendered ), JSON.stringify( truthy.href ) + '  (ownProps ?? backTo -> ownProps wins, bypasses guard)' );
			const nul = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '/query-home' }, ownBackUrl: null } );
			line( 'ownProps null vs query', 'null', String( nul.rendered ), JSON.stringify( nul.href ) + '  (null is nullish -> query back_to used)' );
			const empty1 = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, path: `/start/${ FLOW }/domains`, query: { back_to: '/query-home' }, ownBackUrl: '' } );
			line( "ownProps '' vs query (pos 1)", "''", String( empty1.rendered ), JSON.stringify( empty1.href ) + '  (\'\' not nullish -> backUrl=\'\'; falsy -> no override, flow logic)' );
			const empty0 = observeStepWrapper( { flowName: FLOW, stepName: 'domains', positionInFlow: 0, path: `/start/${ FLOW }/domains`, query: { back_to: '/query-home' }, ownBackUrl: '' } );
			line( "ownProps '' vs query (pos 0)", "''", String( empty0.rendered ), '(expect suppressed: allowBackFirstStep = !!\"\" is false)' );
		}

		// ---- PART G: real step-provided dependency override chain (NewOrExistingSiteStep) ----
		blank();
		line( '== PART G: real NewOrExistingSiteStep signupDependencies.back_to -> backUrl ==' );
		{
			const rs = observeRealStep( { flowName: FLOW, stepName: 'new-or-existing-site', positionInFlow: 1, backTo: '/dependency-home' } );
			line( 'real step dependency back_to', "'/dependency-home'", String( rs.rendered ), JSON.stringify( rs.href ) + '  (step reads signupDependencies.back_to -> ownProps.backUrl, bypasses guard)' );
		}

		// ---- PART H: default-flow omission (flowName === defaultFlowName on /start) ----
		blank();
		line( '== PART H: default-flow (onboarding) omission on /start ==' );
		line( "getStepUrl('onboarding','domains','','')", JSON.stringify( getStepUrl( 'onboarding', 'domains', '', '' ) ) + '  (default flow segment omitted)' );
		line( "getStepUrl('onboarding',null,'','')", JSON.stringify( getStepUrl( 'onboarding', null, '', '' ) ) + '  (flow root)' );
		line( "getStepUrl('onboarding-pm','domains','','')", JSON.stringify( getStepUrl( 'onboarding-pm', 'domains', '', '' ) ) + '  (non-default: flow segment kept)' );
		{
			// Rendered NavigationLink on the default 'onboarding' flow (logged-in [domains,plans]).
			const r = observeNavLink( { flowName: 'onboarding', userLoggedIn: true, signupProgress: { domains: mkStep( 'domains' ), plans: mkStep( 'plans' ) }, stepName: 'plans', positionInFlow: 1 } );
			line( 'rendered NavigationLink default flow', 'onboarding plans(1)', String( r.rendered ), JSON.stringify( r.href ) + '  (default-flow omission in href)' );
		}

		// ---- PART I: richer unicode/metacharacter query encoding ----
		blank();
		line( '== PART I: unicode/metacharacter query encoding (addQueryArgs) ==' );
		{
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: progLoggedIn,
				stepName: 'plans',
				positionInFlow: 1,
				queryParams: { ref: 'a&b=c', q: '<script>alert(1)</script>', u: 'caf\u00e9\u2615' },
			} );
			line( 'unicode/metachar query', 'plans', String( r.rendered ), JSON.stringify( r.href ) );
		}

		// ---- PART J: window.location.search query fallback (queryParams prop undefined) ----
		blank();
		line( '== PART J: window.location.search fallback (no queryParams prop) ==' );
		{
			const origHref = window.location.href;
			window.history.replaceState( null, '', '/?fallback=from-search' );
			const r = observeNavLink( { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: 'plans', positionInFlow: 1 } );
			line( 'search fallback', "window.location.search='?fallback=from-search'", String( r.rendered ), JSON.stringify( r.href ) );
			// restore jsdom location so no cross-contamination
			window.history.replaceState( null, '', origHref );
		}

		blank();
		line( '########## BLITZY-OBSERVATION-END (NON-CANONICAL) ##########' );

		const outPath = process.env.BLITZY_OBS_OUT || '/tmp/blitzy_obs/observations.txt';
		fs.writeFileSync( outPath, OUT.join( '\n' ) + '\n' );
		expect( true ).toBe( true );
	} );
} );
```

### 11.4 Read-only verification & cleanup (`git status` + diff vs the immutable base)

All temporary observation scripts live **outside the checkout** under `/tmp/blitzy_obs/` and are removed during cleanup with a single `rm -rf` of that directory; this answer document is committed. No temporary file was ever created inside the repository working tree, so cleanup touches nothing under version control. The read-only guarantee is anchored on the **immutable source-branch base** `be7e5cc641`: after cleanup and commit the working tree is clean, and the only difference between `be7e5cc641` and the platform branch is the addition of this one document.

```text
$ rm -rf /tmp/blitzy_obs
$ git status --porcelain --untracked-files=all
$ echo "STATUS_EXIT=$?"
STATUS_EXIT=0
$ git diff --stat be7e5cc641622d153040491fd5625c6cb83e12eb -- client/ packages/ config/
$ echo "DIFF_EXIT=$?"
DIFF_EXIT=0
$ git diff --name-only be7e5cc641622d153040491fd5625c6cb83e12eb
blitzy/documentation/wp-calypso_be7e5cc64162.md
```

After cleanup and commit, `git status --porcelain` produces **no output** (a clean working tree); the `git diff --stat` against the source-branch base `be7e5cc641` for `client/`, `packages/`, and `config/` also produces **no output** (all 14 AAP reference files and supporting config/manifests are byte-identical to `be7e5cc641`). The `git diff --name-only` against that base lists exactly **one path** — this documentation file — so, however many documentation-only commits the platform branch accumulates, the net change versus `be7e5cc641` is only this document. No existing product-source file was modified, no permanent tests were added, and no dependencies were changed.

### 11.5 Source-branch filename vs execution branch/HEAD

The answer file is named `wp-calypso_be7e5cc64162.md` after the **source branch** `wp-calypso_be7e5cc64162` (commit `be7e5cc641622d153040491fd5625c6cb83e12eb`), per the SWE-AtlasQnA-Repo naming rule. The investigation actually executed on the **assigned Blitzy platform branch** `blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385`, which is the `be7e5cc641` base plus documentation-only commits that add this single document. Because every investigated source file is unchanged from `be7e5cc641` (§11.4), the runtime observations reflect the source at `be7e5cc641` exactly.

---

## 12. Coverage checklist

Final pass confirming every question and every named mechanism is addressed with `file:line` grounding. The **Evidence** column states honestly _how_ each item was established, using this legend:

- **CANONICAL run** — observed by executing the pre-existing configured test `client/signup/navigation-link/test/index.jsx` (§11.2). This is the only canonical runtime vehicle.
- **NON-CANONICAL run** — observed by executing the custom outside-checkout harness (§11.3); the destinations are real (real modules), but the entry point is not the canonical one.
- **Source** — grounded in cited source code that was read (and, where a `getStepUrl`/config value, executed), not established as a distinct rendered-runtime signal.
- **[inferred]** — reasoned from cited source; **not** executed at runtime.

| Item                                                                                                                                                                                                                                                                                          | Where addressed                                                                                                                                                                                        | Evidence                                                                                  | Status                                |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- | ------------------------------------- |
| **REQ-1** decider — `getBackUrl` computes the `href`; `goToPreviousStep` unwired                                                                                                                                                                                                              | §4 — `NavigationLink.getBackUrl` (`navigation-link/index.jsx:78-115`), href wiring `:183-192`, unwired `handleClick` back branch `:125-127`, `main.jsx:805-806` omits `goToPreviousStep` (grep, §11.1) | Source + CANONICAL run (call-args) + NON-CANONICAL run (rendered href)                    | ✅ Addressed                          |
| **REQ-2** which input wins                                                                                                                                                                                                                                                                    | §5 — precedence `backUrl` > flow position > ordinary query; `back_to` promoted to `backUrl`; observed C, F, G and the ownProps×query cross-product                                                     | Source + NON-CANONICAL run (§8.2 C/F/G, §8.4 PART F)                                      | ✅ Addressed                          |
| **REQ-3** external override source                                                                                                                                                                                                                                                            | §6 — `StepWrapper` connect() `back_to` → `backUrl` (`step-wrapper/index.jsx:273-283`) + full `backUrl` origin inventory + `ownProps` guard bypass                                                      | Source + NON-CANONICAL run (§8.4 PART D/E/F/G)                                            | ✅ Addressed                          |
| **REQ-4** precedence rule (truthy early return)                                                                                                                                                                                                                                               | §5 — `if ( this.props.backUrl ) return this.props.backUrl;` (`navigation-link/index.jsx:83-85`)                                                                                                        | Source + NON-CANONICAL run (C override vs F empty-string fallthrough)                     | ✅ Addressed                          |
| **REQ-5** bypassed step-by-step path                                                                                                                                                                                                                                                          | §7 — `getPreviousStep` (`navigation-link/index.jsx:47-76`) + helpers + duality + partial-progress correction                                                                                           | Source + CANONICAL run (`getPreviousStep` assertions) + NON-CANONICAL run (concrete URLs) | ✅ Addressed                          |
| **REQ-6** per-step observation                                                                                                                                                                                                                                                                | §8.2 per-step + §8.3 visibility + §8.4 extra conditions + §8.5 determinism + §8.6 redirect                                                                                                             | NON-CANONICAL run (every §8 destination value)                                            | ✅ Addressed (NON-CANONICAL)          |
| CANONICAL pre-existing test passes 16/16 (twice)                                                                                                                                                                                                                                              | §11.2                                                                                                                                                                                                  | CANONICAL run                                                                             | ✅ Observed                           |
| Determinism (byte-identical ×2, `sha256 37156c44…`)                                                                                                                                                                                                                                           | §3.3, §8.5, §11.1                                                                                                                                                                                      | NON-CANONICAL run (harness capture) + CANONICAL run (test counts)                         | ✅ Observed                           |
| First-step suppression; a **truthy** `backUrl` forces visibility; an empty-string `''` does **not** (falsy)                                                                                                                                                                                   | §6.1, §8.3, §8.4 PART F                                                                                                                                                                                | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Protocol-relative `back_to` passes the slash guard                                                                                                                                                                                                                                            | §8.4 PART E                                                                                                                                                                                            | NON-CANONICAL run                                                                         | ✅ Observed                           |
| `ownProps.backUrl` vs query `back_to` (`??` truthy/null/empty-string)                                                                                                                                                                                                                         | §8.4 PART F                                                                                                                                                                                            | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Real step-provided dependency `back_to` chain                                                                                                                                                                                                                                                 | §8.4 PART G (`steps/new-or-existing-site/index.tsx:22,31`)                                                                                                                                             | NON-CANONICAL run (real step)                                                             | ✅ Observed                           |
| Default-flow URL omission (`/start/domains`, root `/start`)                                                                                                                                                                                                                                   | §8.4 PART H (`utils.js:63-67`)                                                                                                                                                                         | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Unicode / metacharacter query encoding                                                                                                                                                                                                                                                        | §8.4 PART I (`utils.js:68` `addQueryArgs`)                                                                                                                                                             | NON-CANONICAL run                                                                         | ✅ Observed                           |
| `window.location.search` query fallback                                                                                                                                                                                                                                                       | §8.4 PART J (`navigation-link/index.jsx:87-96`)                                                                                                                                                        | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Cross-flow `lastKnownFlow` slip                                                                                                                                                                                                                                                               | §8.2 E (`state/signup/progress/actions.js:117,130`)                                                                                                                                                    | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Active config: `social-first`, logged-in step removal                                                                                                                                                                                                                                         | §3.4, §8.1 (`flows.getFlow` executed)                                                                                                                                                                  | NON-CANONICAL run + Source                                                                | ✅ Observed                           |
| Ordinary query vs special `back_to`                                                                                                                                                                                                                                                           | §1, §5, §8.2 F                                                                                                                                                                                         | Source + NON-CANONICAL run                                                                | ✅ Addressed                          |
| Conditions A–G + visibility + skipped/empty/partial/absent progress                                                                                                                                                                                                                           | §8.2, §8.3, §8.4                                                                                                                                                                                       | NON-CANONICAL run                                                                         | ✅ Observed                           |
| Named functions (`getBackUrl`, `getPreviousStep`, `getPreviousStepName`, `getStepUrl`, `isFirstStepInFlow`, `getFilteredSteps`, `handleClick`, `StepWrapper.renderBack`+connect, `removeUserStepFromFlow`, `allowBackFirstStep`, `shouldHideNavButtons`, `saveSignupStep`/`submitSignupStep`) | §4–§9                                                                                                                                                                                                  | Source (with runtime where noted)                                                         | ✅ Addressed                          |
| User phrases (snap / slip / not random)                                                                                                                                                                                                                                                       | §9                                                                                                                                                                                                     | Source + NON-CANONICAL run                                                                | ✅ Addressed                          |
| Router interception scoped (same-origin/non-external); slash check ≠ validation; page.js real-fork + upstream citation                                                                                                                                                                        | §4.2 (`calypso-router/src/index.js:776,800,892-900`)                                                                                                                                                   | Source + web research; **click dispatch not executed**                                    | ⚠️ Source + `[inferred]` (click)      |
| `onboarding → /setup` redirect                                                                                                                                                                                                                                                                | §8.6 (`controller.js:179-202`, `index.web.js:16-22`)                                                                                                                                                   | Source + config runtime (§8.1); **redirect navigation not executed**                      | ⚠️ Source + `[inferred]` (navigation) |
| Secondary Stepper `/setup` (`canUserGoBack` `:54-58`)                                                                                                                                                                                                                                         | §10                                                                                                                                                                                                    | Source                                                                                    | ✅ Addressed                          |
| Read-only, temp scripts **outside** checkout & removed, `git status` clean, filename vs branch                                                                                                                                                                                                | §3.5, §11.4, §11.5                                                                                                                                                                                     | Observed (git)                                                                            | ✅ Observed                           |

**Honest coverage statement.** All six questions and every named function, condition, and user phrase are addressed with exact `file:line` references. Every runtime value in §8 is **observed** — the CANONICAL pre-existing test (§11.2) supplies the canonical call-argument/override-href signal, and the **NON-CANONICAL** outside-checkout harness (§11.3) supplies the concrete rendered destinations for every enumerated condition (including the previously-missing protocol-relative, `ownProps`×query, real-step-dependency, default-flow-omission, Unicode-query, and search-fallback cases). Exactly two conclusions are **`[inferred]`**, each grounded in cited source and explicitly labeled as such: the page.js **click dispatch** (§4.2) and the `onboarding → /setup` **redirect navigation** (§8.6); neither was executed through its full runtime pipeline in jsdom.
