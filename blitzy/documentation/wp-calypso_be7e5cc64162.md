# Root-Cause: Unpredictable "Back" Button in the wp-calypso Legacy Signup Flow (`/start`)

> **Repository:** `Automattic/wp-calypso`
> **Answer-file name** is derived from the **source branch** `wp-calypso_be7e5cc64162`, which points at commit `be7e5cc641622d153040491fd5625c6cb83e12eb`. All investigated source files are byte-identical to that commit, so the code observed here **is** the source at `be7e5cc641`.
> **Execution** was performed on the assigned Blitzy platform branch `blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385`, whose working `HEAD` is `61555b16f0332ebd2454e157c11937f525b5c1a1` (the `be7e5cc641` base **plus** this one added document). The filename therefore encodes the source branch, while the run happened on the platform branch — the difference is intentional and expected. (See §11.5.)
> **Scope:** Read-only investigation. The only file added to the repository is this document. A temporary jest observation harness was created under the repository's test glob (so the repo's real transform/module resolution applied) and **deleted** afterward; §11.4 shows the final clean `git status`.

## 1. Summary (direct answer)

The back destination for a given legacy-signup step is decided by a single method — **`NavigationLink.getBackUrl()`** at `client/signup/navigation-link/index.jsx:78-115` — whose return value becomes the Back control's anchor `href` (assigned to `hrefUrl` at `client/signup/navigation-link/index.jsx:183-186` and rendered onto `<Button … href={ hrefUrl } … >` at `:192`). Because the legacy signup step render never wires a `goToPreviousStep` handler (`client/signup/main.jsx:798-814` passes only `goToNextStep` at `:805` and `goToStep` at `:806`; a repository grep finds **no** `goToPreviousStep` in `main.jsx`), the click handler's back branch (`client/signup/navigation-link/index.jsx:125-127`) is **unwired in this legacy production caller** (it is reachable code — it fires when the prop *is* supplied, and the co-located test supplies a `jest.fn()` — but the signup render never supplies it). The computed **`href`** is therefore what actually drives back navigation.

`getBackUrl()` evaluates its inputs in a **fixed precedence** (the order of statements in the method) and the first satisfied branch wins:

1. **Component prop `backUrl` (the override)** — `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85`. This early return short-circuits **before** any flow-position logic runs, so a truthy `backUrl` always wins. Note that one common *source* of this prop is a **query argument**: `?back_to=/…` is transformed into the `backUrl` prop by `StepWrapper`'s `connect()` (`client/signup/step-wrapper/index.jsx:273-283`). So a *specific, special* query argument (`back_to`) **can** decide the destination — by being promoted to the highest-precedence prop before `getBackUrl` runs.
2. **Flow position** — otherwise `getPreviousStep()` (`:47-76`) computes the previous step from `signupProgress` + the current `stepName`, and `getStepUrl()` (`client/signup/utils.js:45-69`) builds the URL.
3. **Ordinary query-string arguments** — every *other* query argument only *decorates* the final URL (appended via `addQueryArgs`, `client/signup/utils.js:68`); ordinary query args never change *which* step is targeted.

The apparent randomness is **not random**. For a **fixed, complete input/environment tuple** the destination is deterministic (§8.4 shows byte-identical output across repeated runs). The tuple is larger than four values, however: it includes `direction`, `backUrl`, `flowName`, `signupProgress` (and each progressed step's `lastKnownFlow`, `stepSectionName`, `wasSkipped`), the current `stepName`, `userLoggedIn`, `queryParams` (with a `window.location.search` fallback), the resolved locale (`getLocaleSlug()`), `window.location.pathname` (which selects the `/start` vs `/setup` framework prefix), and the **active feature-config flags + excluded steps** that shape what `flows.getFlow()` returns (§8.1). The user-perceived unpredictability is the interaction of these deterministic inputs across different accumulated real-world states — most sharply the two "surprise" outcomes: a `{ stepName: null }` previous-step result builds the flow-root URL (**snaps to the first step**), and a previously-progressed step's foreign `lastKnownFlow` redirects the URL into another flow (**slips into a different flow**).

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

| Component | Value | Source |
|-----------|-------|--------|
| Node.js | `v22.23.1` (satisfies repo `engines: ^v22.9.0`; `.nvmrc` pins `22.9.0`) | `node --version` (§11.1) |
| Package manager | `yarn 4.0.2` via corepack (`packageManager: yarn@4.0.2`) | `yarn --version` (§11.1) |
| Test runner | `jest@29.7.0` + `@testing-library/react@16.2.0` (existing devDeps) | `package.json` |
| Router | `@automattic/calypso-router@0.7.0` (a page.js fork) | `package.json` |

`node_modules` was already present (no install was required). The canonical client-test invocation is `TZ=UTC jest -c=test/client/jest.config.js <path>`, exposed as the `test-client` script; the bare `yarn jest <path>` form is **not** used (with no root jest config, jest's default `testMatch` does not match the repo's `test/index.jsx` convention). The client jest environment runs with `NODE_ENV=test`, so `@automattic/calypso-config` loads `config/test.json`, and `jsdom` sets `window.location` to `https://example.com` (pathname `/`), which is why `getStepUrl` resolves the `/start` framework prefix (§7.1).

### 3.2 Canonical observation — the real modules, run under the repo's jest transform

The **canonical** evidence in this document comes from exercising the **real repository modules** — no signup-utility mocks:

- A temporary jest harness (`client/signup/navigation-link/test/blitzy_adhoc_test_canonical.jsx`, full source in §11.3) placed under the repo's test glob (`<rootDir>/**/test/*.[jt]s?(x)` per `packages/calypso-jest/jest-preset.js`) so that the repository's **real** babel transform, module-alias resolution (`calypso/…`), and feature-config loading applied. It:
  1. reads the **real** active configuration — `isEnabled('signup/social-first')` from `@automattic/calypso-config`, and the **real** `flows.getFlow(...)` step lists from `calypso/signup/config/flows` (§8.1);
  2. renders the **unconnected** `NavigationLink` (the named export) with the **real** `calypso/signup/utils` (so `getBackUrl → getPreviousStep → getStepUrl / isFirstStepInFlow / getFilteredSteps` all execute for real) and reads the **real** rendered anchor `href` — producing real `/start/…` URLs (§8.2, scenarios A/B/D/E/F and visibility);
  3. renders the **connected** `StepWrapper` (the default export) over a **real** Redux store, dispatching the **real** `setRoute()` action so that `getCurrentQueryArguments` returns a real `back_to`, exercising `StepWrapper`'s `connect()` `back_to → backUrl` resolution end-to-end (§8.2, scenarios C/G and `shouldHideNavButtons`).

- The pre-existing co-located unit test `client/signup/navigation-link/test/index.jsx` is also run (§11.2). **It is useful but limited**: it `jest.mock`s `calypso/signup/utils` (`client/signup/navigation-link/test/index.jsx:7-12`) and renders the **unconnected** `NavigationLink`, so it proves the *call arguments* passed to `getStepUrl` and the literal `href` for the `backUrl`-override case — but it does **not** compute real `/start/…` URLs, resolve `StepWrapper` query state, or exercise routing. The concrete URLs in this document therefore come from the real-module harness above, not from that mocked test.

**One disclosed deviation from the full production path.** In part (2), rendering the *unconnected* `NavigationLink` supplies `userLoggedIn` and `signupProgress` as explicit props rather than through the Redux `connect()` wrapper (`client/signup/navigation-link/index.jsx:204-215`), which merely injects `isUserLoggedIn(state)` and `getSignupProgress(state)`. The decision logic and URL construction that produce the `href` are the **real** functions; only those two selector pass-throughs are supplied directly. Part (3) uses the real connected component and store, with no such deviation. Nothing about the decider (`getBackUrl`/`getPreviousStep`) or the URL builder (`getStepUrl`) is re-implemented or mocked. (The earlier revision of this document relied on a standalone `/tmp` re-implementation of the helpers; that approach has been **removed** in favor of executing the real modules.)

**Runtime click dispatch is not executed here.** The harness reads the computed `href`; it does not simulate a page.js click. The claim that clicking a same-origin, non-external Back link is intercepted and dispatched client-side is therefore **[inferred]** from the router source (§4.2), not executed in jsdom.

### 3.3 Reproducing the reported intermittency (same input, repeated)

For a "sometimes X, sometimes Y" report, the **same unchanged input** was run repeatedly and the distribution reported:

- The real-module harness was executed **twice** as separate OS processes; its clean observation capture was **byte-identical** across both runs (`sha256` match and empty `diff`, §11.1/§8.4).
- The pre-existing unit test was executed **twice**; both runs reported `16 passed, 16 total`, `EXIT_CODE=0` (§11.2).

The observed distribution is therefore **100% identical across runs** — i.e. the destination is deterministic for a fixed complete input tuple, and the perceived randomness comes from variation in that tuple across sessions, not from nondeterminism in the code (§8.4).

### 3.4 The active flow reality (why `/start/onboarding` is the wrong entry to model)

Two facts about the **active** configuration materially change any per-step analysis, and both were confirmed at runtime (§8.1):

- **`signup/social-first` is enabled**, so the first "token" step of the onboarding-shaped flows is **`user-social`**, not `user` (`client/signup/config/flows-pure.js:13-14` `getUserSocialStepOrFallback`; the flag is `true` in `config/test.json`, `config/production.json`, and every other environment config).
- For a **logged-in** user, `flows.getFlow()` **removes** the token-providing step via `removeUserStepFromFlow` (`client/signup/config/flows.js:216-225,262-280`, filtering steps where `stepConfig[stepName].providesToken` is `true`; `user`/`user-social` both set `providesToken: true` at `client/signup/config/steps-pure.js:112-136,138-162`). So the logged-in `onboarding` flow resolves to `['domains','plans']`, **not** `['user','domains','plans']`.

More decisively, the **`onboarding` flow itself no longer runs on `/start`**: the `/start` middleware chain (`client/signup/index.web.js:16-22`) runs `controller.redirectToFlow` (`:18`) **before** `controller.start` (`:20`), and `redirectToFlow` **redirects the `onboarding` flow to `/setup`** (`client/signup/controller.js:179-197`, guarded by `isOnboardingFlow(flowName)` and calling `getStepUrl(…, '/setup')` then `window.location.replace(url)`). Therefore this document models per-step legacy behavior on a **real, non-redirected legacy `/start` flow — `onboarding-pm`** (`client/signup/config/flows-pure.js:154-155`, steps `[ userSocialStep, 'domains', 'plans' ]`, no `forceLogin`, not matched by `isOnboardingFlow`) — and treats the `onboarding → /setup` redirect as an explicitly demonstrated fact rather than a legacy entry point (§8.5).

### 3.5 Read-only guarantee & cleanup

No existing source file was modified; no permanent tests were added; no dependencies were changed. The temporary harness (`client/signup/navigation-link/test/blitzy_adhoc_test_canonical.jsx`) and the `/tmp/blitzy_obs/` logs it wrote are removed during cleanup (§11.4 shows the exact `rm` command and the resulting `git status --porcelain`, which lists **only** this one new document).

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

Therefore `this.props.goToPreviousStep` is `undefined` inside `NavigationLink`, the back branch never fires in production, and the same-origin `href` produced by `getBackUrl()` is what actually drives back navigation. This is consistent with the pre-existing test asserting both halves independently (§11.2): `should call goToPreviousStep() only when the direction is back and clicked` (verifies the branch *when the prop is supplied*) and `should set a proper url as href prop when the direction is "back".` (verifies the `href` is computed from the back logic).

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

**Answer:** The precedence is fixed by the *order of statements* inside `getBackUrl()`. The **first** meaningful branch is an early return on the component prop `backUrl` (`client/signup/navigation-link/index.jsx:78-115`, quoted verbatim without elision — the annotations that follow are in prose, not in the source):

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

**The precedence rule (REQ-4):** the statement `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85` returns **before** `getPreviousStep()` is ever called (that call is at `:98`). A truthy `backUrl` prop therefore *unconditionally* wins over flow position and over ordinary query args — this single early return is the entire "precedence rule" that lets the override take control. Note it returns the prop **verbatim**: no `getStepUrl` construction, no `addQueryArgs` decoration, no locale — which is why the observed override href is exactly `/home` (§8.2, scenario C) rather than a decorated `/start/…` URL.

**When the three inputs disagree (REQ-2):**

- **Component prop `backUrl`** beats everything (early return at `:83-85`). Observed canonically in scenario **C**: with `?back_to=/home` resolved into `backUrl`, the rendered `href` is `/home` regardless of flow position (§8.2).
- **Flow position** (`getPreviousStep` at `:98`, `getStepUrl` at `:108-114`) decides the target step only when `backUrl` is falsy. Observed in scenario **A** (per-step) and **B/E** (§8.2).
- **Query-string arguments** split into two distinct roles:
  - The **special** argument **`back_to`** is *not* an ordinary decorator: `StepWrapper.connect()` promotes it to the `backUrl` prop (`client/signup/step-wrapper/index.jsx:273-283`), so it enters at **precedence 1**. This is the mechanism by which a query argument *can* choose the destination (§6). Observed in scenarios **C** (valid) and **G** (invalid, guarded out).
  - **All other** query arguments enter as `queryParams` (falling back to `window.location.search` at `:87-89`), are passed as the last argument to `getStepUrl` (`:113`), and are appended to the already-built path by `addQueryArgs` (`client/signup/utils.js:68`). They only *decorate* the URL and never change which step is targeted. Observed in scenario **F**: `?ref=logged-out-homepage` appears on the URL but the target step is unchanged (§8.2).

### 5.1 Precedence rule table

| Precedence | Input | Source | Effect on destination |
|-----------|-------|--------|-----------------------|
| 1 (highest) | Component prop `backUrl` | Hardcoded step config (e.g. mailbox `backUrl: 'mailbox-domain/'` at `client/signup/config/steps-pure.js:399`); step-provided from `signupDependencies.back_to` (§6.2); or the **special `back_to` query arg** resolved in `StepWrapper` `connect()` (`client/signup/step-wrapper/index.jsx:273-283`) | Returned verbatim (`:83-85`); step-by-step logic bypassed; also forces the Back button onto the first step via `allowBackFirstStep` (`client/signup/step-wrapper/index.jsx:65`) |
| 2 | Flow position | `signupProgress` + current `stepName` via `getPreviousStep()` (`:47-76`) → `getStepUrl()` | May be `{ stepName: null }` (→ flow root) or a specific progressed step; carries that step's `lastKnownFlow` (§7) |
| 3 (lowest) | Ordinary query-string arguments | `queryParams` prop, or `window.location.search` fallback (`:87-89`) | Only decorate the final built URL via `addQueryArgs`; never change the targeted step |

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

The control at `positionInFlow === 0` is hidden **unless** any of: a `stepSectionName` is present, or `allowBackFirstStep` is true. Because `StepWrapper.renderBack()` sets `allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }` (`client/signup/step-wrapper/index.jsx:65`), **the mere presence of a `backUrl` forces the Back button to render even on the first step**, and — via the precedence rule — its `href` is the override. That is exactly the "external back target treated like a quiet override even when the current step should not be eligible for it." There is one more upstream gate: `renderBack()` returns `null` entirely when `shouldHideNavButtons` is true (`client/signup/step-wrapper/index.jsx:51-53`). All four conditions (suppressed; forced by `allowBackFirstStep`; forced by `backUrl`; forced by `stepSectionName`; and hidden by `shouldHideNavButtons`) were exercised canonically — see §8.3 and §8.2 scenario set.

### 6.2 All `backUrl` origins (not only the AAP-named ones)

Beyond the `?back_to=` query path, several steps set `backUrl` (or its equivalents) directly, with **differing validation**. This inventory is drawn from a repository-wide search (§11.1):

| Origin | Location | Note |
|--------|----------|------|
| `?back_to=/…` query arg (slash-guarded) | `client/signup/step-wrapper/index.jsx:274-277` | Guard applies only to the query-derived value |
| Hardcoded component prop | `client/signup/config/steps-pure.js:399` | mailbox step `props: { backUrl: 'mailbox-domain/', … }` |
| Step-provided from `signupDependencies.back_to` (as `ownProps.backUrl`) | `client/signup/steps/difm-site-picker/index.tsx:43`; `client/signup/steps/new-or-existing-site/index.tsx:22,31`; `client/signup/steps/site-options/index.tsx:27` | Enters via `ownProps.backUrl`, **bypassing** the `startsWith('/')` guard (§6, `:277`) |
| Email step default | `client/signup/steps/emails/index.jsx:122` | `backUrl = 'domains/'` default, passed to `StepWrapper` at `:144` with `allowBackFirstStep={ !! backUrl }` at `:152` |
| Domains step (many targets) | `client/signup/steps/domains/index.jsx:1367-1440` | Computes `backUrl` from a large `if/else` chain: `previousStepBackUrl` (`:1391-1392`), `domainManagementRoot()` (`:1394`), `/plugins` (`:1400`), `/themes` (`:1403`), a **flow-definition-based** `getStepUrl( flowName, previousStepName )` guarded by `'plans-first' === flowName` (`:1405-1406`), a site-editor `wp-admin` URL (`:1411`), `getStepUrl( flowName, stepName, null, this.getLocale() )` (`:1414`), a `/setup/onboarding/playground` URL (`:1417`), `siteUrl` with `isExternalBackUrl = true` (`:1420-1422`), `/home/${siteSlug}` (`:1424`), `/settings/general/${siteSlug}` (`:1427`), and an `externalBackUrl` from `getExternalBackUrl` (`:1434-1436`) that also sets `isExternalBackUrl = true` (`:1439`) |
| Domains external-source overrides | `client/signup/steps/domains/utils.js:9-31` | `backUrlSourceOverrides` map + `getExternalBackUrl(source, sectionName)` (validated with `valid-url`) |
| WooCommerce install transfer | `client/signup/steps/woocommerce-install/transfer/index.tsx:75` | `backUrl={ \`/woocommerce-installation/${ domain }\` }` |

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
- **First-step guard** (`:50-52`): if `isFirstStepInFlow(...)`, return `{ stepName: null }`. (Note: at the first position the control is usually *not even rendered* — §6.1 — so this branch's URL is rarely user-visible.)
- **Build progressed steps** (`:56-60`): `getFilteredSteps(...)` restricted to the flow's steps and to the current login state, then `.filter( step => ! step.wasSkipped )` drops skipped steps.
- **Empty relevant progress** (`:61-63`): if none remain, return `{ stepName: null }` → flow root. Observed via the connected `StepWrapper` fallthrough (empty store progress) in §8.2 scenario G/"no back_to".
- **Locate current step** (`:66-68`): `findIndex` by `stepName`.
- **Edge branch A — current step absent** (`:70-72`): `if ( currentStepIndexInProgress === -1 ) return filteredProgressedSteps.pop();` → snap to the **last** progressed step. This is the correct description of **partial, non-empty** progress whose current step is absent (scenario **B**, observed `/start/onboarding-pm/domains`).
- **Edge branch B — normal / index 0** (`:75`): `return filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep;` → the immediately-previous progressed step, or (at index 0) `{ stepName: null }` → flow root.

> **Correction of a common misstatement.** A `{ stepName: null }` result arises specifically from the **first-step guard**, **empty relevant progress**, or the **index-0** case. It is **not** the general outcome of "partial progress": partial, non-empty progress whose current step is *absent* takes the `findIndex === -1` branch and returns `pop()` — i.e. the **last** progressed step — as scenario **B** demonstrates.

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

The progress-based method can carry a step's foreign `lastKnownFlow` (which rewrites the destination flow), while the definition-based helper only names a step within the *current* flow. The domains step uses the definition-based helper for one of its `backUrl` branches (`client/signup/steps/domains/index.jsx:1406`). The *name* of the previous step can match between the two while the *destination flow* silently differs — observed in scenario **E**, where the previous step's `lastKnownFlow` (`onboarding-with-email`) rewrites the URL into a different flow (§8.2).

**Named functions for REQ-5:** `NavigationLink.getPreviousStep` (bypassed), with helpers `isFirstStepInFlow`, `getFilteredSteps`, `getStepUrl`, and the sibling `getPreviousStepName`.

---

## 8. REQ-6 — Per-step observation (canonical)

All values in this section were **captured at runtime** by the real-module harness (§3.2, source in §11.3), run under the repository's real jest transform and module resolution. The exact commands and complete raw output are in §11; the clean capture (written by the harness and shown via `cat`) is quoted verbatim in §11.3.

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

This confirms §3.4 at runtime: the token step is **`user-social`** (not `user`), and for a **logged-in** user the token step is **removed** (`onboarding` and `onboarding-pm` both resolve to `["domains","plans"]`). The default flow is `onboarding` — the flow that redirects to `/setup` (§8.5) — which is why the per-step demonstration below uses the real, non-redirected legacy flow **`onboarding-pm`**.

### 8.2 Observed per-step and per-scenario destinations (canonical)

The table separates **whether the Back control renders** (from `NavigationLink.render`, §6.1) from **the computed `href`** (from `getBackUrl`). Scenario **A** is the primary step-by-step path; **B–G** are the secondary/edge conditions. Scenarios **A, B, D, E, F** were captured via the unconnected `NavigationLink` with **real utils**; **C, G** via the **connected `StepWrapper`** + real `setRoute` (the real `back_to` → `backUrl` path).

| Scenario | Condition | Step (position) | Back rendered? | Computed `href` |
|----------|-----------|-----------------|----------------|-----------------|
| A (logged-in, flow `[domains,plans]`) | step-by-step, full progress, no override | `domains` (0) | **No** — first-step suppressed | (computed root would be `/start/onboarding-pm`) |
| A (logged-in) | step-by-step, full progress, no override | `plans` (1) | Yes | `/start/onboarding-pm/domains` |
| A (logged-out, flow `[user-social,domains,plans]`) | step-by-step, full progress, no override | `user-social` (0) | **No** — first-step suppressed | — |
| A (logged-out) | step-by-step, full progress, no override | `domains` (1) | Yes | `/start/onboarding-pm/user-social/en` |
| A (logged-out) | step-by-step, full progress, no override | `plans` (2) | Yes | `/start/onboarding-pm/domains/en` |
| B (logged-in) | current step **absent** from progress (`findIndex === -1` → `pop()`) | `plans` (absent) | Yes | `/start/onboarding-pm/domains` (snaps to last progressed) |
| C (connected `StepWrapper`) | valid `?back_to=/home` (starts with `/`) | `domains` | Yes (forced by `backUrl`) | `/home` (override; `getPreviousStep` bypassed) |
| D (logged-in) | hardcoded component `backUrl` (mailbox real value) | mailbox step | Yes (forced by `backUrl`) | `mailbox-domain/` (returned verbatim) |
| E (logged-in) | previous step carries a different `lastKnownFlow` | `plans` (prev `domains`, `lastKnownFlow='onboarding-with-email'`) | Yes | `/start/onboarding-with-email/domains` (slips into another flow) |
| F (logged-in) | ordinary query arg carried into the built URL | `plans` | Yes | `/start/onboarding-pm/domains?ref=logged-out-homepage` |
| G (connected `StepWrapper`) | invalid `?back_to=home` (no leading `/`) → guard discards it | `domains` | Yes | `/start/onboarding-pm/en` (override ignored; **falls through** to flow-position logic) |

Key reads from this table:

- **REQ-6 / the "snap to first step":** at the first position the control is **suppressed** (rows `domains(0)` logged-in and `user-social(0)` logged-out show *Back rendered? No*). The `{ stepName: null }` → flow-root behavior is therefore mostly visible not at the first step itself, but when *earlier* progress is empty/absent or `back_to`/`lastKnownFlow` interacts (scenarios G and the empty-progress fallthrough). The old conflation of "computed URL" with "what renders" at position 0 is corrected here.
- **Precedence (REQ-2/REQ-4):** scenario **C** shows the `backUrl` override producing `/home` verbatim; scenario **F** shows an ordinary query arg only decorating `/start/onboarding-pm/domains`; scenario **G** shows the guard discarding an invalid `back_to` so flow-position logic runs.
- **The two "surprises":** scenario **E** is the cross-flow slip (`/start/onboarding-with-email/domains`); the flow-root/first-step behavior is the `{ stepName: null }` path of §7.

### 8.3 Visibility conditions (canonical)

Captured from `PART C` (unconnected `NavigationLink` at `positionInFlow === 0`) and `PART D` (`shouldHideNavButtons`):

```text
first-step, no override/section | domains(0) | rendered=false   (suppressed)
first-step + allowBackFirstStep | domains(0) | rendered=true    (visible)
first-step + backUrl            | domains(0) | rendered=true    href="/home"
first-step + stepSectionName    | domains(0) | rendered=true    (visible)
shouldHideNavButtons=true       | domains(1) | rendered=false   (back not rendered)
```

These directly exercise the four conditions of the `render()` suppression block (`client/signup/navigation-link/index.jsx:154-161`) plus the `shouldHideNavButtons` gate (`client/signup/step-wrapper/index.jsx:51-53`).

### 8.4 Repeated-run determinism (reproducing the "never truly random" claim)

The **same unchanged input** was run repeatedly:

- **Real-module harness, two separate processes:** the clean observation capture was **byte-identical** — verified by `sha256sum` (both `d0e14dc9c00c44ef9572defba5004e6f6f64d3831067cf9b4829c3bf5154553c`) and an empty `diff` (§11.1).
- **Pre-existing unit test, two runs:** both `16 passed, 16 total`, `EXIT_CODE=0` (§11.2).

**Conclusion:** for a fixed, complete input/environment tuple (§1, §3.3) the back destination is **deterministic** — the observed distribution across runs is 100% identical. The perceived unpredictability is the interaction of these deterministic inputs across different accumulated real-world states (whether a `back_to` was present and valid, whether the current step is in progress, what `lastKnownFlow` a prior step recorded, login state, locale, and the active flow config).

### 8.5 The `onboarding → /setup` redirect (explicitly demonstrated, not assumed)

`/start/onboarding` does **not** render as a legacy step; it is redirected to the modern `/setup` framework before the legacy `start` controller runs. The `/start` route registration wires the middleware in this order (`client/signup/index.web.js:16-22`, quoted verbatim); note that `controller.redirectToFlow` (`:18`) runs **before** `controller.start` (`:20`):

```js
controller.saveInitialContext,
controller.redirectWithoutLocaleIfLoggedIn,
controller.redirectToFlow,
controller.setSelectedSiteForSignup,
controller.start,
```

and `redirectToFlow` performs the redirect for the onboarding flow (`client/signup/controller.js:179-197`, quoted without elision):

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
    return;
}
```

`isOnboardingFlow` matches exactly the `onboarding` flow (`packages/onboarding/src/utils/flows.ts:102-104`). **[inferred at runtime]:** this document did not execute the full page.js controller pipeline (it needs a real `window.location.replace` navigation), so the redirect is grounded in the source above and in the runtime fact (§8.1) that `flows.getFlow('onboarding', …)` no longer yields the `['user','domains','plans']` shape assumed by a naive `/start/onboarding` model. The legacy per-step evidence therefore uses `onboarding-pm`, which is *not* matched by `isOnboardingFlow` and therefore renders on `/start`.

---

## 9. Mapping the user's exact words to mechanisms

- **"snaps straight to the first step"** → `getPreviousStep()` returns `{ stepName: null }` via the first-step guard (`isFirstStepInFlow`), **empty relevant progress**, or the **index-0** case (`client/signup/navigation-link/index.jsx:47-76`) — **not** merely "partial progress" (see the §7 correction; partial progress with an absent current step instead `pop()`s to the last step, scenario B). Then `getStepUrl( …, null, … )` builds the flow-root URL (`client/signup/utils.js:45-69`). For the *default* `onboarding` flow the root is `/start` (default-flow omission), but that flow redirects to `/setup` (§8.5); for a real legacy flow like `onboarding-pm` the root is `/start/onboarding-pm`. Observed via the connected-`StepWrapper` empty-progress fallthrough (scenario G / "no back_to", §8.2).

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

  (`saveSignupStep`: `lastKnownFlow` at `:117`, spread into the step at `:122`; `submitSignupStep`: `lastKnownFlow` at `:130`, spread at `:145`.) A step progressed under a *different* flow therefore redirects the Back URL into that other flow. Observed as scenario **E → `/start/onboarding-with-email/domains`** (§8.2).

- **"never feels truly random"** → **Confirmed deterministic** (§8.4): identical complete inputs produce byte-identical destinations across repeated in-process and cross-process runs. The apparent randomness is the interaction of the full deterministic input tuple (§1) across different accumulated states.

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

This Stepper logic is acknowledged for completeness only; it cannot validate the legacy `/start` behavior. A downstream reader should confirm which framework the user's specific flow uses before generalizing; note that the default `onboarding` flow is redirected from `/start` to `/setup` (§8.5), so a user reporting this on the "onboarding" flow may in fact be on the Stepper.

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
$ git rev-parse HEAD
61555b16f0332ebd2454e157c11937f525b5c1a1
$ git branch --show-current
blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385

# REQ-1 proof: goToPreviousStep is never passed in the signup render
$ grep -n 'goToPreviousStep' client/signup/main.jsx ; echo "EXIT_CODE=$?"
EXIT_CODE=1

# backUrl origin inventory (REQ-3)
$ grep -rn "backUrl" client/signup/steps/ | grep -iv test
# (results summarized in §6.2)

# Determinism of the real-module harness (two separate processes)
$ sha256sum /tmp/blitzy_obs/obs_run1.txt /tmp/blitzy_obs/obs_run2.txt
d0e14dc9c00c44ef9572defba5004e6f6f64d3831067cf9b4829c3bf5154553c  /tmp/blitzy_obs/obs_run1.txt
d0e14dc9c00c44ef9572defba5004e6f6f64d3831067cf9b4829c3bf5154553c  /tmp/blitzy_obs/obs_run2.txt
$ diff /tmp/blitzy_obs/obs_run1.txt /tmp/blitzy_obs/obs_run2.txt ; echo "DIFF_EXIT=$?"
DIFF_EXIT=0
```

`DIFF_EXIT=0` with no output means the two runs are byte-identical.

### 11.2 Pre-existing unit test — both runs (canonical command, complete summary)

Command (run twice, with output redirected and exit code captured):

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false \
    > /tmp/blitzy_obs/preexist_run1.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
```

Run #1 — complete captured log (verbatim `cat`; the `Browserslist` notice is the tool's own stderr; note jest prints **no** per-run time on the `PASS` line for the first, uncached run):

```text
$ cat /tmp/blitzy_obs/preexist_run1.log
Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/navigation-link/test/index.jsx

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.253 s, estimated 6 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
```

Run #2 — determinism (identical suite/test counts and exit code; only the per-run timing text differs — the cached second run prints `(5.068 s)` on the `PASS` line and omits `estimated`):

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false \
    > /tmp/blitzy_obs/preexist_run2.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ sed -n '4,10p' /tmp/blitzy_obs/preexist_run2.log
PASS client/signup/navigation-link/test/index.jsx (5.068 s)

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.348 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
```

Both runs: `EXIT_CODE=0`, `Tests: 16 passed, 16 total`. This test's built-in `getPreviousStep()` assertions map onto the branches in §7: `2nd → 1st`, `1st → nullish`, `3rd → 2nd`, `unknown → last in progress` (the `pop()` edge branch), `no flow steps → nullish`, and `skipped steps ignored`. Because it mocks `calypso/signup/utils` (`client/signup/navigation-link/test/index.jsx:7-12`), it proves *call arguments* and the literal override `href`, not the concrete `/start/…` URLs (§3.2).

### 11.3 Real-module harness — canonical capture

Command (run twice, as separate processes; the second run wrote `obs_run2.txt`, byte-identical per §11.1):

```text
$ BLITZY_OBS_OUT=/tmp/blitzy_obs/obs_run1.txt \
  CI=true yarn test-client client/signup/navigation-link/test/blitzy_adhoc_test_canonical.jsx --ci --watchAll=false \
  > /tmp/blitzy_obs/jest_run1.log 2>&1
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ grep -E 'PASS |Tests:' /tmp/blitzy_obs/jest_run1.log
PASS client/signup/navigation-link/test/blitzy_adhoc_test_canonical.jsx (6.056 s)
Tests:       1 passed, 1 total
```

Clean observation capture (the harness writes this file; shown verbatim):

```text
$ cat /tmp/blitzy_obs/obs_run1.txt
########## BLITZY-CANONICAL-BEGIN ##########

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
first-step + backUrl | domains(0) | true | "/home"
first-step + stepSectionName | domains(0) | true | (expect visible)

== PART D: StepWrapper connect() back_to -> backUrl (CANONICAL connected path) ==
SCENARIO | back_to | BACK RENDERED? | HREF
   route.query.current fed to connect() | {"back_to":"/home"}
C valid back_to (starts with /) | '/home' | true | "/home"
   route.query.current fed to connect() | {"back_to":"home"}
G invalid back_to (no leading /) | 'home' | true | "/start/onboarding-pm/en"  (fell through to flow-position logic)
   route.query.current fed to connect() | false
no back_to (empty progress fallthrough) | none | true | "/start/onboarding-pm/en"
   route.query.current fed to connect() | {"back_to":"/home"}
shouldHideNavButtons=true | '/home' | false | (expect back NOT rendered)

########## BLITZY-CANONICAL-END ##########
```

Harness source (temporary; created under the repo test glob so the repo's real transform/module resolution applied, then deleted — §11.4). It is reproduced here **complete and verbatim** — no logic is elided. It contains **no** run-specific absolute checkout path; the output path defaults to `/tmp/blitzy_obs/canonical_observations.txt` and is overridable via the `BLITZY_OBS_OUT` environment variable (which the runs above set):

```jsx
/** @jest-environment jsdom */
/*
 * BLITZY TEMPORARY OBSERVATION HARNESS (NOT committed; deleted after capture).
 * Purpose: capture CANONICAL runtime evidence for the Back-button root-cause doc by
 * exercising the REAL repository modules (no util mock):
 *   - real calypso/signup/utils (getBackUrl chain: getStepUrl/isFirstStepInFlow/getFilteredSteps)
 *   - real calypso/signup/config/flows (+ @automattic/calypso-config feature flags)
 *   - real render of the UNCONNECTED NavigationLink (named export) with real utils
 *   - real render of the CONNECTED StepWrapper (default export) over a real Redux store,
 *     with the real setRoute() action feeding getCurrentQueryArguments -> back_to resolution
 * It intentionally does NOT mock calypso/signup/utils.
 */
import fs from 'fs';
import { isEnabled } from '@automattic/calypso-config';
import { render } from '@testing-library/react';
import { createStore, applyMiddleware } from 'redux';
import { thunk } from 'redux-thunk';
import flows from 'calypso/signup/config/flows';
import { NavigationLink } from 'calypso/signup/navigation-link';
import StepWrapper from 'calypso/signup/step-wrapper';
import initialReducer from 'calypso/state/reducer';
// eslint-disable-next-line no-restricted-imports
import routeReducer from 'calypso/state/route/reducer';
import { setRoute } from 'calypso/state/route/actions';
import { renderWithProvider } from 'calypso/test-helpers/testing-library';

const translate = ( s ) => s;

// Build a progress object keyed by stepName (matches Redux signup.progress shape).
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

// Render the CONNECTED StepWrapper over a real store with a given back_to query arg.
function observeStepWrapperBack( { flowName, stepName, positionInFlow, backTo, shouldHideNavButtons } ) {
	// The `route` slice is normally registered on the global redux-store singleton via
	// `calypso/state/route/init`; a locally-created store must add it explicitly so that
	// the real setRoute() action populates route.query.current (read by getCurrentQueryArguments).
	const reducer = initialReducer.addReducer( [ 'route' ], routeReducer );
	const store = createStore( reducer, applyMiddleware( thunk ) );
	if ( backTo !== undefined ) {
		store.dispatch(
			setRoute( `/start/${ flowName }/${ stepName }`, { back_to: backTo } )
		);
	}
	// Observed real state fed to StepWrapper's connect() mapStateToProps.
	const qa = store.getState()?.route?.query?.current;
	line( '   route.query.current fed to connect()', JSON.stringify( qa ) );
	const { container, unmount } = renderWithProvider(
		<StepWrapper
			flowName={ flowName }
			stepName={ stepName }
			positionInFlow={ positionInFlow }
			hideFormattedHeader
			shouldHideNavButtons={ shouldHideNavButtons }
		/>,
		{ store }
	);
	const el = container.querySelector( '.navigation-link.back' );
	const result = { rendered: !! el, href: el ? el.getAttribute( 'href' ) : null };
	unmount();
	return result;
}

const OUT = [];
function line( ...parts ) {
	const s = parts.join( ' | ' );
	OUT.push( s );
	// eslint-disable-next-line no-console
	console.log( s );
}

describe( 'BLITZY canonical Back-button observation', () => {
	test( 'capture all conditions', () => {
		line( '\n########## BLITZY-CANONICAL-BEGIN ##########' );

		// ---- PART A: active configuration & real flow resolution ----
		line( '\n== PART A: active config + real flows.getFlow ==' );
		line( "isEnabled('signup/social-first')", String( isEnabled( 'signup/social-first' ) ) );
		line(
			"flows.getFlow('onboarding', loggedOut).steps",
			JSON.stringify( flows.getFlow( 'onboarding', false ).steps )
		);
		line(
			"flows.getFlow('onboarding', loggedIn).steps",
			JSON.stringify( flows.getFlow( 'onboarding', true ).steps )
		);
		line(
			"flows.getFlow('onboarding-pm', loggedOut).steps",
			JSON.stringify( flows.getFlow( 'onboarding-pm', false ).steps )
		);
		line(
			"flows.getFlow('onboarding-pm', loggedIn).steps",
			JSON.stringify( flows.getFlow( 'onboarding-pm', true ).steps )
		);
		line( 'flows.defaultFlowName', JSON.stringify( flows.defaultFlowName ) );

		// ---- PART B: UNCONNECTED NavigationLink, REAL utils (decider + URL builder) ----
		// Featured legacy flow: onboarding-pm (NOT redirected to /setup).
		const FLOW = 'onboarding-pm';

		// Full progress for logged-IN flow ['domains','plans'] (user-social removed).
		const progLoggedIn = {
			domains: mkStep( 'domains' ),
			plans: mkStep( 'plans' ),
		};
		// Full progress for logged-OUT flow ['user-social','domains','plans'].
		const progLoggedOut = {
			'user-social': mkStep( 'user-social' ),
			domains: mkStep( 'domains' ),
			plans: mkStep( 'plans' ),
		};

		line( '\n== PART B1: per-step, LOGGED-IN (onboarding-pm => [domains,plans]) ==' );
		line( 'SCENARIO', 'STEP(position)', 'BACK RENDERED?', 'HREF' );
		[ [ 'domains', 0 ], [ 'plans', 1 ] ].forEach( ( [ step, pos ] ) => {
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: progLoggedIn,
				stepName: step,
				positionInFlow: pos,
			} );
			line( 'A(logged-in)', `${ step }(${ pos })`, String( r.rendered ), JSON.stringify( r.href ) );
		} );

		line( '\n== PART B2: per-step, LOGGED-OUT (onboarding-pm => [user-social,domains,plans]) ==' );
		line( 'SCENARIO', 'STEP(position)', 'BACK RENDERED?', 'HREF' );
		[ [ 'user-social', 0 ], [ 'domains', 1 ], [ 'plans', 2 ] ].forEach( ( [ step, pos ] ) => {
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: false,
				signupProgress: progLoggedOut,
				stepName: step,
				positionInFlow: pos,
			} );
			line( 'A(logged-out)', `${ step }(${ pos })`, String( r.rendered ), JSON.stringify( r.href ) );
		} );

		line( '\n== PART B3: edge scenarios B/D/E/F (logged-in) ==' );
		// B: current step ABSENT from progress (findIndex === -1 -> pop() last progressed)
		{
			const partial = { domains: mkStep( 'domains' ) }; // plans absent
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: partial,
				stepName: 'plans',
				positionInFlow: 1,
			} );
			line( 'B current-step-absent -> pop()', 'plans(absent)', String( r.rendered ), JSON.stringify( r.href ) );
		}
		// D: hardcoded component backUrl (mailbox step real config value 'mailbox-domain/')
		{
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: progLoggedIn,
				stepName: 'plans',
				positionInFlow: 1,
				backUrl: 'mailbox-domain/',
			} );
			line( 'D hardcoded backUrl', 'mailbox-domain/', String( r.rendered ), JSON.stringify( r.href ) );
		}
		// E: previous step carries a DIFFERENT lastKnownFlow -> cross-flow slip
		{
			const cross = {
				domains: mkStep( 'domains', { lastKnownFlow: 'onboarding-with-email' } ),
				plans: mkStep( 'plans', { lastKnownFlow: FLOW } ),
			};
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: cross,
				stepName: 'plans',
				positionInFlow: 1,
			} );
			line( 'E cross-flow lastKnownFlow', 'plans(prev domains@onboarding-with-email)', String( r.rendered ), JSON.stringify( r.href ) );
		}
		// F: query args decorate the built URL (do not change target step)
		{
			const r = observeNavLink( {
				flowName: FLOW,
				userLoggedIn: true,
				signupProgress: progLoggedIn,
				stepName: 'plans',
				positionInFlow: 1,
				queryParams: { ref: 'logged-out-homepage' },
			} );
			line( 'F query-arg decoration', 'plans', String( r.rendered ), JSON.stringify( r.href ) );
		}

		// ---- PART C: visibility conditions (UNCONNECTED NavigationLink) ----
		line( '\n== PART C: visibility at first step (position 0) ==' );
		{
			const base = { flowName: FLOW, userLoggedIn: true, signupProgress: progLoggedIn, stepName: 'domains', positionInFlow: 0 };
			line( 'first-step, no override/section', 'domains(0)', String( observeNavLink( base ).rendered ), '(expect suppressed)' );
			line( 'first-step + allowBackFirstStep', 'domains(0)', String( observeNavLink( { ...base, allowBackFirstStep: true } ).rendered ), '(expect visible)' );
			line( 'first-step + backUrl', 'domains(0)', String( observeNavLink( { ...base, backUrl: '/home', allowBackFirstStep: true } ).rendered ), JSON.stringify( observeNavLink( { ...base, backUrl: '/home', allowBackFirstStep: true } ).href ) );
			line( 'first-step + stepSectionName', 'domains(0)', String( observeNavLink( { ...base, stepSectionName: 'some-section' } ).rendered ), '(expect visible)' );
		}

		// ---- PART D: CONNECTED StepWrapper back_to resolution (C, G) + shouldHideNavButtons ----
		line( '\n== PART D: StepWrapper connect() back_to -> backUrl (CANONICAL connected path) ==' );
		line( 'SCENARIO', 'back_to', 'BACK RENDERED?', 'HREF' );
		{
			const c = observeStepWrapperBack( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, backTo: '/home' } );
			line( 'C valid back_to (starts with /)', "'/home'", String( c.rendered ), JSON.stringify( c.href ) );
			const g = observeStepWrapperBack( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, backTo: 'home' } );
			line( 'G invalid back_to (no leading /)', "'home'", String( g.rendered ), JSON.stringify( g.href ) + '  (fell through to flow-position logic)' );
			const none = observeStepWrapperBack( { flowName: FLOW, stepName: 'domains', positionInFlow: 1 } );
			line( 'no back_to (empty progress fallthrough)', 'none', String( none.rendered ), JSON.stringify( none.href ) );
			const hidden = observeStepWrapperBack( { flowName: FLOW, stepName: 'domains', positionInFlow: 1, backTo: '/home', shouldHideNavButtons: true } );
			line( 'shouldHideNavButtons=true', "'/home'", String( hidden.rendered ), '(expect back NOT rendered)' );
		}

		line( '\n########## BLITZY-CANONICAL-END ##########\n' );
		const outPath = process.env.BLITZY_OBS_OUT || '/tmp/blitzy_obs/canonical_observations.txt';
		fs.writeFileSync( outPath, OUT.join( '\n' ) + '\n' );
		expect( true ).toBe( true );
	} );
} );
```

### 11.4 Read-only verification & cleanup (`git status --porcelain`)

This answer document already existed at HEAD (the review targeted it), so after the rewrite it shows as **modified** (` M`), not untracked. The temporary harness is untracked (`??`) and is deleted during cleanup, after which the working tree's only change is this document:

```text
$ rm -f client/signup/navigation-link/test/blitzy_adhoc_test_canonical.jsx
$ rm -rf /tmp/blitzy_obs
$ git status --porcelain --untracked-files=all
 M blitzy/documentation/wp-calypso_be7e5cc64162.md
$ git diff --stat be7e5cc641622d153040491fd5625c6cb83e12eb -- client/ packages/ config/
$ echo "DIFF_EXIT=$?"
DIFF_EXIT=0
```

After cleanup, `git status` lists **only** this one modified document; the `git diff --stat` against the source-branch base `be7e5cc641` for `client/`, `packages/`, and `config/` produces **no output** (all 14 AAP reference files and supporting config/manifests are byte-identical to `be7e5cc641` — the only commit between `be7e5cc641` and the execution `HEAD` added this documentation file and nothing else). No existing product-source file was modified, no permanent tests were added, and no dependencies were changed.

### 11.5 Source-branch filename vs execution branch/HEAD

The answer file is named `wp-calypso_be7e5cc64162.md` after the **source branch** `wp-calypso_be7e5cc64162` (commit `be7e5cc641622d153040491fd5625c6cb83e12eb`), per the SWE-AtlasQnA-Repo naming rule. The investigation actually executed on the **assigned Blitzy platform branch** `blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385`, whose `HEAD` (`61555b16f0…`) is the `be7e5cc641` base plus this single added document. Because every investigated source file is unchanged from `be7e5cc641` (§11.4), the runtime observations reflect the source at `be7e5cc641` exactly.

---

## 12. Coverage checklist

Final pass confirming every question and every named mechanism is addressed with observed evidence and `file:line` grounding:

| Item | Where addressed | Status |
|------|-----------------|--------|
| **REQ-1** decider | §4 — `NavigationLink.getBackUrl` (`navigation-link/index.jsx:78-115`), href wiring `:183-192`, unwired `handleClick` back branch `:125-127`, `main.jsx:805-806` omits `goToPreviousStep` (grep, §11.1) | ✓ |
| **REQ-2** which input wins | §5 — precedence `backUrl` > flow position > ordinary query; `back_to` promoted to `backUrl`; observed C/F/G | ✓ |
| **REQ-3** external override source | §6 — `StepWrapper` connect() `back_to` → `backUrl` (`step-wrapper/index.jsx:273-283`) + full `backUrl` origin inventory + `ownProps` guard bypass | ✓ |
| **REQ-4** precedence rule | §5 — early return `if ( this.props.backUrl ) return this.props.backUrl;` (`navigation-link/index.jsx:83-85`) | ✓ |
| **REQ-5** bypassed step-by-step path | §7 — `getPreviousStep` (`navigation-link/index.jsx:47-76`) + helpers + duality + partial-progress correction | ✓ |
| **REQ-6** per-step observation | §8 — canonical per-step + visibility + `onboarding → /setup` redirect + determinism | ✓ |
| Active config: `social-first`, logged-in step removal | §3.4, §8.1 (`flows.getFlow` observed) | ✓ |
| `onboarding → /setup` redirect | §8.5 (`controller.js:179-197`, `index.web.js:16-22`) | ✓ |
| Ordinary query vs special `back_to` | §1, §5 | ✓ |
| Router interception scoped (same-origin/non-external); slash check ≠ validation | §4.2 (`calypso-router/src/index.js:776,800,892-900`) | ✓ |
| Determinism over full state/env tuple | §1, §3.3, §8.4 | ✓ |
| Named: `getBackUrl`, `getPreviousStep`, `getPreviousStepName`, `getStepUrl`, `isFirstStepInFlow`, `getFilteredSteps`, `handleClick`, `StepWrapper.renderBack`+connect, `removeUserStepFromFlow`, `allowBackFirstStep`, `shouldHideNavButtons`, `saveSignupStep`/`submitSignupStep` | §4–§9 | ✓ |
| Conditions A–G + visibility + skipped/empty/partial/absent progress | §8.2, §8.3 | ✓ |
| User phrases (snap / slip / not random) | §9 | ✓ |
| page.js interception (real fork citation + upstream link) | §4.2 | ✓ |
| Secondary Stepper `/setup` (`canUserGoBack` `:54-58`) | §10 | ✓ |
| Read-only, temp harness removed, `git status` clean, filename vs branch | §3.5, §11.4, §11.5 | ✓ |

**All six questions and every named function, condition, and user phrase are addressed with exact `file:line` references and observed output; runtime click dispatch and the onboarding redirect navigation are the only conclusions labeled `[inferred]`, each grounded in cited source.**
