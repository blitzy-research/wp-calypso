# Root-Cause: Unpredictable "Back" Button in the wp-calypso Legacy Signup Flow (`/start`)

> **Repository:** `Automattic/wp-calypso` · **Branch:** `wp-calypso_be7e5cc64162` · **HEAD commit:** `be7e5cc641622d153040491fd5625c6cb83e12eb`
> **Scope:** Read-only investigation. The only file added to the repository is this document.

## 1. Summary (direct answer)

The back destination for a given signup step is decided by a single function — **`NavigationLink.getBackUrl()`** at `client/signup/navigation-link/index.jsx:78-115` — whose return value becomes the Back control's anchor `href` (`client/signup/navigation-link/index.jsx:183-186`, rendered onto `<Button href={ hrefUrl } … >` at `:192`). Because the legacy signup step render never wires a `goToPreviousStep` handler (`client/signup/main.jsx:800-815` passes only `goToNextStep` and `goToStep`; a repository-wide grep finds **no** `goToPreviousStep` in `main.jsx`), the click handler's back branch (`client/signup/navigation-link/index.jsx:125-127`) is dead code, and the computed `href` — intercepted client-side by the page.js router — is what actually navigates.

`getBackUrl()` evaluates its inputs in a **fixed precedence** and the first satisfied branch wins:

1. **Component prop `backUrl` (external override)** — `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85`. This early return short-circuits **before** any flow-position logic runs, so the override always wins.
2. **Flow position** — otherwise `getPreviousStep()` (`:47-76`) computes the previous step from `signupProgress` + the current `stepName`, and `getStepUrl()` (`client/signup/utils.js:45-69`) builds the URL.
3. **Query-string arguments** — only *decorate* the final URL (appended via `addQueryArgs`, `client/signup/utils.js:68`); they never change which step is targeted.

The apparent randomness is **not random**: the destination is a deterministic pure function of `backUrl`, `signupProgress`, the current `stepName`, and the query args. Repeated runs on the same input produce byte-identical output (shown below). The two "surprise" outcomes the user describes both live inside `getPreviousStep()`/`getStepUrl()`: a `{ stepName: null }` result builds the flow-root URL (**snaps to the first step**), and a previously-progressed step's `lastKnownFlow` redirects the URL into another flow (**slips into a different flow**).

---

## 2. The user's report (verbatim)

> "The Back button is supposed to move one step backward, but every so often it snaps straight to the first step or slips out into an entirely different flow, and it never feels truly random."

This document answers, by name and with evidence, the six questions this report raises:

- **REQ-1** — What actually decides the destination for a given step?
- **REQ-2** — Which inputs win when flow position, component props, and query-string args disagree?
- **REQ-3** — Where does the external back target (the "quiet override") come from?
- **REQ-4** — What precedence rule lets that override take control?
- **REQ-5** — What step-by-step code path is being bypassed?
- **REQ-6** — What is the computed destination for each step position (confirm the pattern)?

---

## 3. Methodology & environment

Per the governing rule (**SWE-AtlasQnA-Repo**), this investigation **ran the code first** and the answers below are grounded in captured runtime output plus `file:line` references naming the exact function/method. Statements that are reasoned rather than directly observed are labeled **[inferred]**.

### 3.1 Toolchain

| Component | Value | Source |
|-----------|-------|--------|
| Node.js | `v22.23.1` (satisfies repo `engines: ^v22.9.0`; `.nvmrc` pins `22.9.0`) | `node --version` |
| Package manager | `yarn 4.0.2` via corepack (`packageManager: yarn@4.0.2`) | `yarn --version` |
| Test runner | `jest@29.7.0` + `@testing-library/react@16.2.0` (existing devDeps) | `package.json` |
| Router | `@automattic/calypso-router@0.7.0` (a page.js fork) | `package.json` |

`node_modules` was already present (no install was required). Had it been missing, the canonical bootstrap is `NODE_OPTIONS=--max-old-space-size=8192 yarn install --immutable --inline-builds`.

### 3.2 Canonical vs NON-CANONICAL observation

- **Canonical capture** = the existing jest test `client/signup/navigation-link/test/index.jsx`, which renders `<NavigationLink>` through the real React + `@testing-library/react` path and reads the rendered anchor's `href`. It is run with:

  ```
  CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false --verbose
  ```

  (`test-client` expands to `TZ=UTC jest -c=test/client/jest.config.js`. The bare `yarn jest <path>` form is **not** used: with no root jest config, jest's default `testMatch` does not match the repo's `test/index.jsx` convention and reports "No tests found".) This test **mocks** the `calypso/signup/utils` helpers (`client/signup/navigation-link/test/index.jsx:7-12`) and asserts the *call arguments* passed to `getStepUrl` and the literal `href` for the `backUrl`-override case. It therefore proves the **decider wiring** (href = `getBackUrl()`, override precedence, first-step → `null`) but does not itself compute real `/start/...` URLs.

- **NON-CANONICAL capture** = a standalone Node script (`/tmp/blitzy_obs/repro_back_nav.js`, created **outside** the repository checkout and deleted afterward). It re-implements the four helpers *verbatim* and drives the real `onboarding` step list `['user','domains','plans']` to capture the actual per-step URLs for every scenario A–G. It is labeled **NON-CANONICAL** because it does not render the component through React/Redux/page.js. Its `addQueryArgs` re-implementation was independently verified to match the repository's real `client/lib/url/add-query-args.ts` for all URL shapes used here.

### 3.3 Reproducing the reported intermittency

For a "sometimes X, sometimes Y" report, the same unchanged input was run repeatedly. Both the canonical jest test and the NON-CANONICAL reproduction were executed **twice**; the reproduction additionally runs its full scenario matrix twice in-process. All outputs were **identical** across runs (see §8 and §11), confirming the destination is deterministic for fixed inputs.

### 3.4 Read-only guarantee & cleanup

No existing source file was modified; no tests were added; no dependencies were changed. All temporary scripts live under `/tmp` (outside the checkout) and are removed before completion. The final `git status --porcelain` (§11.4) shows **only** this one new document.

---

## 4. REQ-1 — The decider

**Answer: `NavigationLink.getBackUrl()` at `client/signup/navigation-link/index.jsx:78-115` is the decider.** Its return value is assigned to `hrefUrl` and rendered as the Back control's anchor `href`.

The render wiring (`client/signup/navigation-link/index.jsx:181-193`):

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
```

So for a back control (`direction === 'back'`), `href = this.getBackUrl()`.

### 4.1 Why the `href` — not a click handler — navigates

`handleClick` (`client/signup/navigation-link/index.jsx:117-132`) only calls a JS navigation function on the **back** path when `goToPreviousStep` is provided:

```js
handleClick = () => {
    if ( this.props.direction === 'forward' ) {
        this.props.submitSignupStep( { stepName: this.props.stepName }, this.props.defaultDependencies );
        this.props.goToNextStep();
    } else if ( this.props.goToPreviousStep ) {
        this.props.goToPreviousStep();
    }
    if ( ! this.props.disabledTracksOnClick ) {
        this.recordClick();
    }
};
```

But `goToPreviousStep` is **never wired** into legacy signup steps:

- `StepWrapper.renderBack()` forwards whatever it received: `goToPreviousStep={ this.props.goToPreviousStep }` (`client/signup/step-wrapper/index.jsx:57`).
- The signup flow render in `client/signup/main.jsx:800-815` passes only `goToNextStep={ this.goToNextStep }` (`:805`) and `goToStep={ this.goToStep }` (`:806`) to the step component — it does **not** pass `goToPreviousStep`. A repository grep confirms the prop appears **nowhere** in `main.jsx`:

  ```
  $ grep -n 'goToPreviousStep' client/signup/main.jsx
  NO MATCHES (confirmed goToPreviousStep is never passed)
  ```

Therefore `this.props.goToPreviousStep` is `undefined` inside `NavigationLink`, the `else if ( this.props.goToPreviousStep )` branch (`:125-127`) never fires, and the same-origin `href` produced by `getBackUrl()` is what actually drives back navigation.

This is confirmed by the **canonical** test, which asserts both halves independently:

- `✓ should call goToPreviousStep() only when the direction is back and clicked` — verifies the branch works *when the prop is supplied* (the test supplies a `jest.fn()`).
- `✓ should set a proper url as href prop when the direction is "back".` — verifies the `href` is computed from the back logic.

In production the prop is not supplied, so only the `href` matters. **[inferred]** that the click is dispatched client-side rather than causing a full page load — grounded by the page.js confirmation next.

### 4.2 page.js anchor-click interception (why the `href` is the effective trigger)

wp-calypso routes through `@automattic/calypso-router`, a fork of **page.js**. page.js installs a document-level click handler that intercepts same-origin anchor clicks (e.g. `<a href="/user/profile">`) and dispatches them through the client-side router instead of performing a full page navigation; this interception can be disabled with `page.start({ click: false })`. Because `getBackUrl()` returns a same-origin path (a `/start/...` URL built by `getStepUrl`, or an override like `/home`), clicking Back is intercepted by the router and dispatched client-side to that path. Source: page.js documentation, `github.com/visionmedia/page.js`.

**Named functions for REQ-1:** `NavigationLink.getBackUrl` (computes the destination) and `NavigationLink.render` (attaches it as the anchor `href`); `NavigationLink.handleClick` (the back branch that is inert because `goToPreviousStep` is never passed).


---

## 5. REQ-2 & REQ-4 — Precedence, and the rule that lets the override win

**Answer:** The precedence is fixed by the *order of statements* inside `getBackUrl()`. The **first** meaningful branch is an early return on the component prop `backUrl`:

```js
getBackUrl() {
    if ( this.props.direction !== 'back' ) {
        return;
    }
    if ( this.props.backUrl ) {
        return this.props.backUrl;          // <-- REQ-4: highest-precedence early return
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
    const previousStep = this.getPreviousStep( flowName, signupProgress, stepName );   // flow position
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
        queryParams                          // query args only decorate the URL
    );
}
```

(`client/signup/navigation-link/index.jsx:78-115`.)

**The precedence rule (REQ-4):** the statement `if ( this.props.backUrl ) { return this.props.backUrl; }` at `client/signup/navigation-link/index.jsx:83-85` returns **before** `getPreviousStep()` is ever called (that call is at `:98`). A truthy `backUrl` prop therefore *unconditionally* wins over flow position and over query args — this single early return is the entire "precedence rule" that lets the external override take control.

**When the three inputs disagree (REQ-2):**

- **Component prop `backUrl`** beats everything (early return at `:83-85`).
- **Flow position** (`getPreviousStep` at `:98`, `getStepUrl` at `:108-114`) decides the target step only when `backUrl` is falsy.
- **Query-string arguments** never change *which* step is targeted. They enter as `queryParams` (falling back to `window.location.search` at `:87-89`), are passed as the last argument to `getStepUrl` (`:113`), and are appended to the already-built path by `addQueryArgs` (`client/signup/utils.js:68`). They only *decorate* the URL.

Observed confirmation (NON-CANONICAL reproduction, §8): with a `backUrl` override present, every step returns the override (`/home`) regardless of flow position (scenario C); query args (`?ref=logged-out-homepage`) appear on the URL but do not change the target step `/start/domains` (scenario F).

### Precedence Rule Table

| Precedence | Input | Source | Effect |
|-----------|-------|--------|--------|
| 1 (highest) | Component prop `backUrl` | Hardcoded step config (e.g., mailbox `backUrl: 'mailbox-domain/'` at `client/signup/config/steps-pure.js:399`); step-provided from `signupDependencies.back_to`; or `?back_to=/...` resolved in `StepWrapper` connect() | Returned immediately; step-by-step logic bypassed; also forces Back onto first step via `allowBackFirstStep` |
| 2 | Flow position | `signupProgress` + current `stepName` via `getPreviousStep()` | May be `null` (→ flow root = first step) or the last-progressed step (via `pop()`) |
| 3 (lowest) | Query string arguments | `window.location.search` → `queryParams` | Only decorate the final built URL; never change the targeted step |


---

## 6. REQ-3 — The external override source

**Answer:** The external back target most commonly enters as the `?back_to=/...` query argument, resolved into the `backUrl` prop inside **`StepWrapper`'s `connect()` `mapStateToProps`** at `client/signup/step-wrapper/index.jsx:273-283`:

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

- `:274` — reads `back_to` from the query string.
- `:275` — **GUARD:** a `back_to` value that does **not** start with `/` is discarded (`backTo` becomes `undefined`). This is scenario **G** below.
- `:277` — a step-provided/hardcoded `backUrl` (`ownProps.backUrl`) takes priority; otherwise the guarded `back_to` is used.

The resolved `backUrl` is then passed straight into the child (`client/signup/step-wrapper/index.jsx:50-69`):

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

`backUrl` is passed at `:62`. Note also that `backUrl` is **not** declared in `NavigationLink`'s own propTypes — it arrives entirely from `StepWrapper`, which is why the override can feel "quiet".

### 6.1 Why the override is eligible even when it "should not be" — `allowBackFirstStep`

At `client/signup/step-wrapper/index.jsx:65`, `allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }`. The mere **presence** of a `backUrl` forces `allowBackFirstStep` true, which defeats the normal first-step suppression in `NavigationLink.render()` (`client/signup/navigation-link/index.jsx:154-161`):

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

Normally the Back button is hidden on the first step (`positionInFlow === 0`). But when a `backUrl` exists, `allowBackFirstStep` is true, the `return null` is skipped, the Back button renders even on the first step, and — via the precedence rule — its `href` is the override. This is exactly the "external back target treated like a quiet override even when the current step should not be eligible for it."

### 6.2 All `backUrl` origins (where an override can come from)

| Origin | Location | Snippet |
|--------|----------|---------|
| `?back_to=/...` query arg (guarded) | `client/signup/step-wrapper/index.jsx:274-277` | `const backUrl = ownProps.backUrl ?? backTo;` |
| Hardcoded component prop | `client/signup/config/steps-pure.js:399` | mailbox step `props: { backUrl: 'mailbox-domain/', … }` |
| Step-provided from `signupDependencies.back_to` | `client/signup/steps/difm-site-picker/index.tsx:43` | `const { back_to: backUrl } = signupDependencies;` |
| Step-provided from `signupDependencies.back_to` | `client/signup/steps/new-or-existing-site/index.tsx:22,31` | type `back_to?: string;` then `const { back_to: backUrl } = signupDependencies;` |
| Step-provided from `signupDependencies.back_to` | `client/signup/steps/site-options/index.tsx:27` | `const { siteTitle, tagline, siteId, back_to: backUrl } = signupDependencies;` |

**Named function for REQ-3:** the `connect()` `mapStateToProps` closure in `client/signup/step-wrapper/index.jsx` (the `back_to` → `backUrl` resolver), feeding `NavigationLink` via `StepWrapper.renderBack`.


---

## 7. REQ-5 — The bypassed step-by-step path

**Answer:** The step-by-step (flow-position) computation that the `backUrl` override skips is **`NavigationLink.getPreviousStep( flowName, signupProgress, currentStepName )`** at `client/signup/navigation-link/index.jsx:47-76`. When `backUrl` is truthy, `getBackUrl()` returns at `:83-85` and this function is never called.

Full logic (quoted, not elided):

```js
getPreviousStep( flowName, signupProgress, currentStepName ) {
    const previousStep = { stepName: null };
    if ( isFirstStepInFlow( flowName, currentStepName, this.props.userLoggedIn ) ) {
        return previousStep;
    }
    const filteredProgressedSteps = getFilteredSteps(
        flowName,
        signupProgress,
        this.props.userLoggedIn
    ).filter( ( step ) => ! step.wasSkipped );
    if ( filteredProgressedSteps.length === 0 ) {
        return previousStep;
    }
    const currentStepIndexInProgress = filteredProgressedSteps.findIndex(
        ( step ) => step.stepName === currentStepName
    );
    if ( currentStepIndexInProgress === -1 ) {
        return filteredProgressedSteps.pop();
    }
    return filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep;
}
```

Branch-by-branch:

- **Default** `previousStep = { stepName: null }` (`:48`). A `null` step name later builds the flow-root URL → the **first** step.
- **First-step guard** (`:50-52`): if `isFirstStepInFlow(...)`, return `{ stepName: null }`.
- **Build progressed steps** (`:56-60`): `getFilteredSteps(...)` filtered to drop `wasSkipped` steps.
- **Empty progress** (`:61-63`): if none remain, return `{ stepName: null }` → flow root.
- **Locate current step** (`:66-68`): `findIndex` by `stepName`.
- **Edge branch A — current step absent** (`:70-72`): `if ( currentStepIndexInProgress === -1 ) return filteredProgressedSteps.pop();` → snap to the **last** progressed step.
- **Edge branch B — normal / index 0** (`:75`): `return filteredProgressedSteps[ currentStepIndexInProgress - 1 ] || previousStep;` → the previous progressed step, or (at index 0) `{ stepName: null }` → flow root.

### 7.1 Supporting helpers (each named and cited)

**`isFirstStepInFlow`** — `client/signup/utils.js:28-31`:

```js
export function isFirstStepInFlow( flowName, stepName, isUserLoggedIn ) {
    const { steps: stepsBelongingToFlow } = flows.getFlow( flowName, isUserLoggedIn );
    return stepsBelongingToFlow.indexOf( stepName ) === 0;
}
```

**`getFilteredSteps`** — `client/signup/utils.js:137-150`:

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

**`getStepUrl`** — `client/signup/utils.js:45-69` (builds the final URL):

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

- **Framework prefix** (`:57-61`): `/setup` if `window.location.pathname` starts with `/setup`, else `/start`.
- **Default-flow omission** (`:63-67`): when `flowName === defaultFlowName` (`'onboarding'`) **and** framework is `/start`, the flow name is **omitted** from the path. This is why `getStepUrl('onboarding', null, …)` → `/start` (not `/start/onboarding`) and `getStepUrl('onboarding', 'user', …)` → `/start/user` (not `/start/onboarding/user`).

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

Observed (§8 reproduction): with full progress on `onboarding`, current `plans`, both agree on `domains`. But with cross-flow progress, current `domains`, both name `user` — yet `getPreviousStep` also carries that step's foreign `lastKnownFlow` (`onboarding-with-email`), which is what actually redirects the built URL into a different flow. The *name* can match while the *destination flow* silently differs.

**Named functions for REQ-5:** `NavigationLink.getPreviousStep` (bypassed), with helpers `isFirstStepInFlow`, `getFilteredSteps`, `getStepUrl`, and the sibling `getPreviousStepName`.


---

## 8. REQ-6 — Per-step observation (confirming the pattern)

The flow used is the real default flow **`onboarding`**, resolved by `getDefaultFlowName()` at `client/signup/config/flows.js:243-245` (returns `'onboarding'`) and assigned as `defaultFlowName` at `:250`. Its step list is defined at `client/signup/config/flows-pure.js:131-134`:

```js
{
    name: ONBOARDING_FLOW,
    steps: [ userSocialStep, 'domains', 'plans' ],
    destination: getSignupDestination,
```

where `userSocialStep` (`:30`) comes from `getUserSocialStepOrFallback` (`:13-14`):

```js
const getUserSocialStepOrFallback = () =>
    isEnabled( 'signup/social-first' ) ? 'user-social' : 'user';
```

With the `signup/social-first` flag disabled (the default), the observed step list is **`['user','domains','plans']`**, with `userLoggedIn = true` on the `/start` base.

### 8.1 Observed per-step destination table (captured at runtime)

The values below were **captured** by the NON-CANONICAL reproduction (§8.3); they match the expected behavior exactly. The per-step rows A are the primary step-by-step path; B–G are the secondary/edge conditions.

| Scenario | Condition | Step (position) | Observed back destination | Source |
|----------|-----------|-----------------|---------------------------|--------|
| A | Step-by-step, full progress, no override | `user` (0) | `/start` (flow root → renders FIRST step) | NON-CANONICAL |
| A | Step-by-step, full progress, no override | `domains` (1) | `/start/user` | NON-CANONICAL |
| A | Step-by-step, full progress, no override | `plans` (2) | `/start/domains` | NON-CANONICAL |
| B | Current step NOT in progress (`findIndex === -1` → `pop()`) | `plans` (absent) | `/start/domains` (snaps to last progressed) | NON-CANONICAL |
| C | External `?back_to=/home` (backUrl set) | every step | `/home` (getPreviousStep bypassed) | NON-CANONICAL |
| D | Hardcoded component `backUrl` on mailbox step | `mailbox` | `mailbox-domain/` | NON-CANONICAL |
| E | Previous step carries different `lastKnownFlow` | `domains` (prev `user`, lastKnownFlow `onboarding-with-email`) | `/start/onboarding-with-email/user` (slips into another flow) | NON-CANONICAL |
| F | Query args carried into built URL | `plans` | `/start/domains?ref=logged-out-homepage` | NON-CANONICAL |
| G | `back_to` FAILS `startsWith('/')` guard (e.g. `back_to=home`) | any step | override IGNORED → `/start/domains` (falls through to flow-position logic) | NON-CANONICAL |

### 8.2 Canonical capture — jest test (real render path)

Command and complete output (run #1). This proves the decider wiring through the real React render: the anchor `href` is computed by `getBackUrl()`, the `backUrl` override maps directly to the `href`, and the first-step case computes with `stepName = null`.

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false --verbose

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/navigation-link/test/index.jsx (5.202 s)
  NavigationLink
    ✓ should render Button element (43 ms)
    ✓ should render no icons when the direction prop is undefined (3 ms)
    ✓ should render right-arrow icon when the direction prop is "forward". (12 ms)
    ✓ should render left-arrow icon when the direction prop is "back". (9 ms)
    ✓ should set href prop to undefined when the direction is not "back". (7 ms)
    ✓ should set a proper url as href prop when the direction is "back". (14 ms)
    ✓ should call goToNextStep() only when the direction is forward and clicked (24 ms)
    ✓ should not call goToNextStep() when the direction is back (15 ms)
    ✓ should call goToPreviousStep() only when the direction is back and clicked (14 ms)
    ✓ should not call goToPreviousStep() when the direction is forward (12 ms)
    ✓ getPreviousStep() When in 2nd step should return 1st step (1 ms)
    ✓ getPreviousStep() When in 1st step should return nullish step
    ✓ getPreviousStep() When in 3rd step should return 2nd step (1 ms)
    ✓ getPreviousStep() When current steps is unknown, step should return last step in progress which belong to the current flow
    ✓ getPreviousStep() When current progress does not contain any step of the current flow return nullish step (1 ms)
    ✓ getPreviousStep() When there are skipped steps they should be ignored

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.795 s, estimated 8 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
EXIT_CODE=0
```

The test's built-in `getPreviousstep()` unit assertions map directly onto the branches in §7: `2nd → 1st`, `1st → nullish`, `3rd → 2nd`, `unknown → last in progress` (the `pop()` edge branch A), `no flow steps → nullish`, and `skipped steps ignored`. **Determinism:** a second identical run (§11.2) produced the same 16 passing tests and `EXIT_CODE=0` (only per-test millisecond timings differed).

**Canonical limitation.** Because the test mocks `calypso/signup/utils` (`client/signup/navigation-link/test/index.jsx:7-12`), `getStepUrl` returns a mock value; the test asserts the *arguments* passed to it, e.g. for the logged-out back case it asserts `getStepUrl` was called with `('test:flow', 'test:step1', 'test:section1', 'en', undefined)`, and for the first-step case with `('test:flow', null, '', 'en', undefined)`; for the override case it asserts the rendered `href` equals the literal `backUrl`. The concrete `/start/...` URLs therefore come from the NON-CANONICAL capture next.

### 8.3 NON-CANONICAL capture — standalone reproduction (real URLs)

A standalone script (`/tmp/blitzy_obs/repro_back_nav.js`, outside the checkout) re-implements `getBackUrl`/`getPreviousStep`/`getStepUrl`/`isFirstStepInFlow`/`getFilteredSteps`/`getPreviousStepName` verbatim (using the repository's own `lodash`) and models `addQueryArgs` (independently verified to match `client/lib/url/add-query-args.ts`). It runs the whole scenario matrix **twice in-process**. Complete output (shell invocation #1):

```text
$ node /tmp/blitzy_obs/repro_back_nav.js

================ RUN 1 ================
SCENARIO | CONDITION | STEP | OBSERVED BACK DESTINATION
A | step-by-step, full progress, no override | user | "/start"
A | step-by-step, full progress, no override | domains | "/start/user"
A | step-by-step, full progress, no override | plans | "/start/domains"
B | current step absent (findIndex===-1 -> pop()) | plans (absent) | "/start/domains"
C | external ?back_to=/home (override) | user | "/home"
C | external ?back_to=/home (override) | domains | "/home"
C | external ?back_to=/home (override) | plans | "/home"
D | hardcoded backUrl 'mailbox-domain/' | mailbox | "mailbox-domain/"
E | prev step lastKnownFlow='onboarding-with-email' | domains (prev user) | "/start/onboarding-with-email/user"
F | query args {ref: logged-out-homepage} | plans | "/start/domains?ref=logged-out-homepage"
G | back_to='home' fails startsWith('/') guard => ignored | plans | "/start/domains   [resolved backUrl=undefined]"

-- getPreviousStep vs getPreviousStepName (duality) --
full progress, current 'plans': getPreviousStep="domains"  getPreviousStepName="domains"
cross-flow progress, current 'domains': getPreviousStep.stepName="user" lastKnownFlow="onboarding-with-email"  getPreviousStepName="user"

================ RUN 2 ================
SCENARIO | CONDITION | STEP | OBSERVED BACK DESTINATION
A | step-by-step, full progress, no override | user | "/start"
A | step-by-step, full progress, no override | domains | "/start/user"
A | step-by-step, full progress, no override | plans | "/start/domains"
B | current step absent (findIndex===-1 -> pop()) | plans (absent) | "/start/domains"
C | external ?back_to=/home (override) | user | "/home"
C | external ?back_to=/home (override) | domains | "/home"
C | external ?back_to=/home (override) | plans | "/home"
D | hardcoded backUrl 'mailbox-domain/' | mailbox | "mailbox-domain/"
E | prev step lastKnownFlow='onboarding-with-email' | domains (prev user) | "/start/onboarding-with-email/user"
F | query args {ref: logged-out-homepage} | plans | "/start/domains?ref=logged-out-homepage"
G | back_to='home' fails startsWith('/') guard => ignored | plans | "/start/domains   [resolved backUrl=undefined]"

-- getPreviousStep vs getPreviousStepName (duality) --
full progress, current 'plans': getPreviousStep="domains"  getPreviousStepName="domains"
cross-flow progress, current 'domains': getPreviousStep.stepName="user" lastKnownFlow="onboarding-with-email"  getPreviousStepName="user"

DETERMINISM (RUN 1 === RUN 2): IDENTICAL
```

### 8.4 Repeated-run determinism (reproducing the "never truly random" claim)

The same unchanged input was run repeatedly:

- **In-process:** `RUN 1 === RUN 2: IDENTICAL` (printed above).
- **Across separate process invocations:** shell invocation #1 vs #2 were byte-identical:

  ```text
  $ diff repro_shell1.log repro_shell2.log
  DIFF_RESULT=IDENTICAL (no differences)
  ```

- **Canonical jest:** two runs both reported `16 passed, 16 total`, `EXIT_CODE=0`.

**Conclusion:** the back destination is a **deterministic** pure function of `backUrl`, `signupProgress`, current `stepName`, and query args. It is *not* random — the perceived unpredictability is the interaction of these deterministic inputs across different accumulated real-world states (e.g. whether a `back_to` was present, whether the current step is in progress, and what `lastKnownFlow` a prior step recorded).


---

## 9. Mapping the user's exact words to mechanisms

- **"snaps straight to the first step"** → `getPreviousStep()` returns `{ stepName: null }` — via the first-step guard (`isFirstStepInFlow`), empty/partial progress, or index 0 (`client/signup/navigation-link/index.jsx:47-76`). Then `getStepUrl( …, null, … )` builds the flow-root URL (`client/signup/utils.js:45-67`), which renders the FIRST step. Observed as scenario **A/`user` → `/start`** and, when progress is absent/empty, more broadly. The default-flow omission (`client/signup/utils.js:63-67`) makes the root `/start` rather than `/start/onboarding`.

- **"slips out into an entirely different flow"** → the return statement uses the previous step's `lastKnownFlow`: `return getStepUrl( previousStep.lastKnownFlow || this.props.flowName, previousStep.stepName, … )` (`client/signup/navigation-link/index.jsx:108-114`). That `lastKnownFlow` is stamped onto every progressed step in `client/state/signup/progress/actions.js`:

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

  (`saveSignupStep`: `lastKnownFlow` at `:117`, spread into the step at `:122`; `submitSignupStep`: `lastKnownFlow` at `:130`, spread at `:145`.) A step progressed under a *different* flow therefore redirects the Back URL into that other flow. Observed as scenario **E → `/start/onboarding-with-email/user`**.

- **"never feels truly random"** → **Confirmed deterministic** (§8.4): identical inputs produce byte-identical destinations across repeated in-process and cross-process runs. The destination is a pure function of `backUrl`, `signupProgress`, current `stepName`, and query args; the apparent randomness is the interaction of these deterministic inputs across different accumulated states.

---

## 10. Secondary framework note — the modern Stepper (`/setup`)

The repository has **two** onboarding frameworks. The **primary** subject of this analysis is the legacy signup framework at `/start` (class components, Redux `connect`, page.js `href` navigation via `NavigationLink` inside `StepWrapper`) — the three named inputs (flow position, component props, query-string arguments) map precisely onto its `NavigationLink.getBackUrl` precedence.

The **modern Stepper** at `/setup` is a parallel system with its **own** cross-flow hazard, in `client/landing/stepper/declarative-flow/internals/hooks/use-step-navigation-with-tracking/index.ts`:

- Comment at `:47-52` (esp. `:49`) explicitly warns that `previousStep` is persisted and can be a step from another flow:

  ```js
  /**
   * If the previous step is defined in the store, and the current step is not the first step, we can go back.
   * We need to make sure we're not at the first step because `previousStep` is persisted and can be a step from another flow or another run of the current flow.
   * …
   */
  ```

- `canUserGoBack` guards at `:53-57`:

  ```js
  const canUserGoBack =
      stepData?.previousStep &&
      currentStepRoute !== stepSlugs[ 0 ] &&
      history.length > 1 &&
      stepData.previousStep !== currentStepRoute;
  ```

- The default `goBack` handler calls `history.back()` at `:128` (within the handler at `:124-130`), and a flow-defined `goBack` overrides it at `:134-141` ("Flow is the ultimate authority on navigation").

This Stepper logic is acknowledged for completeness only. A downstream reader should confirm which framework the user's specific flow uses before generalizing; for the three named inputs, the legacy `/start` analysis above is the answer.


---

## 11. Evidence appendix

### 11.1 Commands run

```text
# Environment
node --version                     # v22.23.1
yarn --version                     # 4.0.2
git rev-parse HEAD                 # be7e5cc641622d153040491fd5625c6cb83e12eb
git branch --show-current          # blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385

# Canonical capture (real render path) — run twice
CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false --verbose

# NON-CANONICAL capture (standalone reproduction, outside the checkout) — run twice
node /tmp/blitzy_obs/repro_back_nav.js

# Confirm no repo files changed except the new document
git status --porcelain
```

### 11.2 Canonical jest run #2 (determinism)

```text
$ CI=true yarn test-client client/signup/navigation-link/test/index.jsx --ci --watchAll=false --verbose

Browserslist: browsers data (caniuse-lite) is 17 months old. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
PASS client/signup/navigation-link/test/index.jsx (5.186 s)
  NavigationLink
    ✓ should render Button element (43 ms)
    ✓ should render no icons when the direction prop is undefined (3 ms)
    ✓ should render right-arrow icon when the direction prop is "forward". (12 ms)
    ✓ should render left-arrow icon when the direction prop is "back". (8 ms)
    ✓ should set href prop to undefined when the direction is not "back". (6 ms)
    ✓ should set a proper url as href prop when the direction is "back". (13 ms)
    ✓ should call goToNextStep() only when the direction is forward and clicked (25 ms)
    ✓ should not call goToNextStep() when the direction is back (16 ms)
    ✓ should call goToPreviousStep() only when the direction is back and clicked (15 ms)
    ✓ should not call goToPreviousStep() when the direction is forward (13 ms)
    ✓ getPreviousStep() When in 2nd step should return 1st step
    ✓ getPreviousStep() When in 1st step should return nullish step (1 ms)
    ✓ getPreviousStep() When in 3rd step should return 2nd step
    ✓ getPreviousStep() When current steps is unknown, step should return last step in progress which belong to the current flow
    ✓ getPreviousStep() When current progress does not contain any step of the current flow return nullish step
    ✓ getPreviousStep() When there are skipped steps they should be ignored

Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Snapshots:   0 total
Time:        5.479 s, estimated 6 s
Ran all test suites matching /client\/signup\/navigation-link\/test\/index.jsx/i.
EXIT_CODE=0
```

Identical outcomes to run #1 (§8.2); only per-test millisecond timings differ.

### 11.3 NON-CANONICAL reproduction script (full contents)

> **NON-CANONICAL.** This script re-implements the helper logic verbatim to capture concrete URLs; it does not render `<NavigationLink>` through React/Redux/page.js. It was created at `/tmp/blitzy_obs/repro_back_nav.js` (outside the repository checkout) and removed after use (§11.4). SHA-256 at authoring time: `66de4aaef01032504b29d5f5f938ce69607d9e9696b6db3bf7b1300489ba768f`.

```js
/*
 * NON-CANONICAL standalone reproduction of the wp-calypso legacy signup
 * Back-button destination logic. Lives OUTSIDE the repo checkout (/tmp) and is
 * deleted after use so the repository is left unchanged.
 *
 * It faithfully re-implements (copied verbatim from source at HEAD be7e5cc6):
 *   - NavigationLink.getBackUrl()      client/signup/navigation-link/index.jsx:L78-L115
 *   - NavigationLink.getPreviousStep() client/signup/navigation-link/index.jsx:L47-L76
 *   - getStepUrl()                     client/signup/utils.js:L45-L69
 *   - isFirstStepInFlow()              client/signup/utils.js:L28-L31
 *   - getFilteredSteps()               client/signup/utils.js:L137-L150
 *   - getPreviousStepName()            client/signup/utils.js:L85-L88 (for the duality demo)
 *   - addQueryArgs()                   client/lib/url/add-query-args.ts (PATH_ABSOLUTE + PATH_RELATIVE)
 *   - StepWrapper back_to guard        client/signup/step-wrapper/index.jsx:L274-L277
 *
 * It uses the REPO'S OWN lodash (filter/sortBy/get/includes) for fidelity.
 * It is NON-CANONICAL because it does not render <NavigationLink> through the
 * real React/Redux/page.js path; the canonical capture is the jest test run.
 */
'use strict';

const REPO = '/tmp/blitzy/wp-calypso/blitzy-7ddb2e12-bf8a-4204-9404-c1b0f7958385_645bc6';
const { filter, sortBy, get, includes } = require(REPO + '/node_modules/lodash');

// ---- Flow model: the REAL onboarding flow (default flow) --------------------
// flows.js:L243-L245 getDefaultFlowName() => 'onboarding'; :L250 defaultFlowName.
// flows-pure.js:L133 ONBOARDING_FLOW steps: [userSocialStep,'domains','plans']
//   userSocialStep => 'user' with signup/social-first disabled (default).
const defaultFlowName = 'onboarding';
const FLOWS = {
	onboarding: { steps: ['user', 'domains', 'plans'] },
	// onboarding-with-email exists as a separate flow; only its NAME matters for
	// getStepUrl in scenario E (getStepUrl does not consult getFlow).
	'onboarding-with-email': { steps: ['user', 'domains', 'plans'] },
};
const flows = { getFlow: (name) => FLOWS[name] };

// ---- addQueryArgs (client/lib/url/add-query-args.ts) ------------------------
const BASE_URL = 'http://__domain__.invalid';
function addQueryArgs(args, url) {
	if ('object' !== typeof args) throw new Error('addQueryArgs expects the first argument to be an object.');
	if ('string' !== typeof url) throw new Error('addQueryArgs expects the second argument to be a string.');
	const clean = {};
	for (const k of Object.keys(args)) if (args[k] != null) clean[k] = args[k]; // pickBy(!= null)
	const parsed = new URL(url, BASE_URL);
	const newSearch = new URLSearchParams(parsed.search);
	for (const k of Object.keys(clean)) newSearch.set(k, String(clean[k]));
	const pathRelative = !url.startsWith('/') && !url.startsWith('http'); // PATH_RELATIVE like 'mailbox-domain/'
	if (pathRelative) {
		let s = newSearch.toString();
		if (s !== '') s = '?' + s;
		if (parsed.search !== '') return url.replace(parsed.search, s);
		if (parsed.hash) return url.replace(parsed.hash, s + parsed.hash);
		return url + s;
	}
	parsed.search = newSearch.toString();
	return parsed.pathname + (parsed.search || '') + (parsed.hash || ''); // format PATH_ABSOLUTE
}

// ---- utils.js re-implementations -------------------------------------------
function isFirstStepInFlow(flowName, stepName /*, isUserLoggedIn */) {
	const { steps: stepsBelongingToFlow } = flows.getFlow(flowName);
	return stepsBelongingToFlow.indexOf(stepName) === 0;
}
function getFilteredSteps(flowName, progress /*, isUserLoggedIn */) {
	const flow = flows.getFlow(flowName);
	if (!flow) return [];
	return sortBy(
		filter(progress, (step) => includes(flow.steps, step.stepName)),
		({ stepName }) => flow.steps.indexOf(stepName)
	);
}
function getStepUrl(flowName, stepName, stepSectionName, localeSlug, params = {}, frameworkParam = null) {
	const flow = flowName ? '/' + flowName : '';
	const step = stepName ? '/' + stepName : '';
	const section = stepSectionName ? '/' + stepSectionName : '';
	const locale = localeSlug ? '/' + localeSlug : '';
	// No window in /tmp; canonical /start app resolves framework to '/start'.
	const framework =
		frameworkParam ||
		(typeof window !== 'undefined' && window.location.pathname.startsWith('/setup') ? '/setup' : '/start');
	const url =
		flowName === defaultFlowName && framework === '/start'
			? framework + step + section + locale // default flow name OMITTED in /start
			: framework + flow + step + section + locale;
	return addQueryArgs(params || {}, url);
}
function getPreviousStepName(flowName, currentStepName) {
	const flow = flows.getFlow(flowName);
	return flow.steps[flow.steps.indexOf(currentStepName) - 1];
}

// ---- NavigationLink.getPreviousStep (verbatim logic) -----------------------
function getPreviousStep(flowName, signupProgress, currentStepName, userLoggedIn) {
	const previousStep = { stepName: null };
	if (isFirstStepInFlow(flowName, currentStepName, userLoggedIn)) {
		return previousStep;
	}
	const filteredProgressedSteps = getFilteredSteps(flowName, signupProgress, userLoggedIn).filter(
		(step) => !step.wasSkipped
	);
	if (filteredProgressedSteps.length === 0) {
		return previousStep;
	}
	const currentStepIndexInProgress = filteredProgressedSteps.findIndex(
		(step) => step.stepName === currentStepName
	);
	if (currentStepIndexInProgress === -1) {
		return filteredProgressedSteps.pop();
	}
	return filteredProgressedSteps[currentStepIndexInProgress - 1] || previousStep;
}

// ---- NavigationLink.getBackUrl (verbatim logic) ----------------------------
function getBackUrl(props) {
	if (props.direction !== 'back') {
		return undefined;
	}
	if (props.backUrl) {
		return props.backUrl; // <-- HIGHEST PRECEDENCE early return (index.jsx:L83-L85)
	}
	const { flowName, signupProgress, stepName, userLoggedIn, queryParams } = props;
	const previousStep = getPreviousStep(flowName, signupProgress, stepName, userLoggedIn);
	const stepSectionName = get(signupProgress, [previousStep.stepName, 'stepSectionName'], '');
	const locale = !userLoggedIn ? '' /* getLocaleSlug() */ : '';
	return getStepUrl(
		previousStep.lastKnownFlow || props.flowName,
		previousStep.stepName,
		stepSectionName,
		locale,
		queryParams
	);
}

// ---- StepWrapper back_to guard (step-wrapper/index.jsx:L274-L277) -----------
function resolveBackUrlFromQuery(ownBackUrl, back_to) {
	const backToParam = back_to != null ? String(back_to) : undefined;
	const backTo = backToParam && backToParam.startsWith('/') ? backToParam : undefined; // GUARD L275
	return ownBackUrl != null ? ownBackUrl : backTo; // ownProps.backUrl ?? backTo  L277
}

// ---- Scenario data ----------------------------------------------------------
const mk = (stepName, extra = {}) => ({
	stepName,
	stepSectionName: '',
	wasSkipped: false,
	lastKnownFlow: 'onboarding',
	...extra,
});
// Full progress (object keyed by stepName, as the real Redux state stores it).
const fullProgress = {
	user: mk('user'),
	domains: mk('domains'),
	plans: mk('plans'),
};
// Progress WITHOUT plans (plans not yet finished => absent from progress).
const partialProgress = {
	user: mk('user'),
	domains: mk('domains'),
};
// Progress where 'user' was recorded under a DIFFERENT flow.
const crossFlowProgress = {
	user: mk('user', { lastKnownFlow: 'onboarding-with-email' }),
	domains: mk('domains', { lastKnownFlow: 'onboarding' }),
};

const base = { direction: 'back', flowName: 'onboarding', userLoggedIn: true };

function run() {
	const rows = [];
	const push = (scenario, condition, step, dest) => rows.push({ scenario, condition, step, dest });

	// A - step-by-step, full progress, no override
	for (const s of ['user', 'domains', 'plans']) {
		push('A', 'step-by-step, full progress, no override', s,
			getBackUrl({ ...base, signupProgress: fullProgress, stepName: s }));
	}
	// B - current step NOT in progress (findIndex === -1 -> pop())
	push('B', 'current step absent (findIndex===-1 -> pop())', 'plans (absent)',
		getBackUrl({ ...base, signupProgress: partialProgress, stepName: 'plans' }));
	// C - external ?back_to=/home (backUrl set) : every step
	for (const s of ['user', 'domains', 'plans']) {
		const backUrl = resolveBackUrlFromQuery(undefined, '/home');
		push('C', 'external ?back_to=/home (override)', s,
			getBackUrl({ ...base, signupProgress: fullProgress, stepName: s, backUrl }));
	}
	// D - hardcoded component backUrl on mailbox step
	push('D', "hardcoded backUrl 'mailbox-domain/'", 'mailbox',
		getBackUrl({ ...base, flowName: 'onboarding', signupProgress: fullProgress, stepName: 'mailbox', backUrl: 'mailbox-domain/' }));
	// E - previous step carries different lastKnownFlow
	push('E', "prev step lastKnownFlow='onboarding-with-email'", 'domains (prev user)',
		getBackUrl({ ...base, signupProgress: crossFlowProgress, stepName: 'domains' }));
	// F - query args carried into built URL
	push('F', 'query args {ref: logged-out-homepage}', 'plans',
		getBackUrl({ ...base, signupProgress: fullProgress, stepName: 'plans', queryParams: { ref: 'logged-out-homepage' } }));
	// G - back_to FAILS startsWith('/') guard (e.g. back_to=home) -> ignored
	{
		const backUrl = resolveBackUrlFromQuery(undefined, 'home'); // -> undefined
		push('G', "back_to='home' fails startsWith('/') guard => ignored", 'plans',
			getBackUrl({ ...base, signupProgress: fullProgress, stepName: 'plans', backUrl }) +
			'   [resolved backUrl=' + JSON.stringify(backUrl) + ']');
	}
	return rows;
}

// ---- Duality demo: getPreviousStep vs getPreviousStepName -------------------
function dualityDemo() {
	const gps = getPreviousStep('onboarding', fullProgress, 'plans', true).stepName;
	const gpsn = getPreviousStepName('onboarding', 'plans');
	const gps2 = getPreviousStep('onboarding', crossFlowProgress, 'domains', true);
	const gpsn2 = getPreviousStepName('onboarding', 'domains');
	return { gps, gpsn, gps2, gpsn2 };
}

function printRun(label) {
	const rows = run();
	console.log('\n================ ' + label + ' ================');
	console.log('SCENARIO | CONDITION | STEP | OBSERVED BACK DESTINATION');
	for (const r of rows) {
		console.log(r.scenario + ' | ' + r.condition + ' | ' + r.step + ' | ' + JSON.stringify(r.dest));
	}
	const d = dualityDemo();
	console.log('\n-- getPreviousStep vs getPreviousStepName (duality) --');
	console.log("full progress, current 'plans': getPreviousStep=" + JSON.stringify(d.gps) + '  getPreviousStepName=' + JSON.stringify(d.gpsn));
	console.log("cross-flow progress, current 'domains': getPreviousStep.stepName=" + JSON.stringify(d.gps2.stepName) + ' lastKnownFlow=' + JSON.stringify(d.gps2.lastKnownFlow) + '  getPreviousStepName=' + JSON.stringify(d.gpsn2));
	return rows;
}

// Run TWICE in-process to show determinism for the SAME input.
const r1 = printRun('RUN 1');
const r2 = printRun('RUN 2');
const identical = JSON.stringify(r1) === JSON.stringify(r2);
console.log('\nDETERMINISM (RUN 1 === RUN 2): ' + (identical ? 'IDENTICAL' : 'DIFFERENT'));
```


### 11.4 Read-only verification (`git status --porcelain`)

All temporary artifacts (the reproduction script and captured logs) were created under `/tmp/blitzy_obs/`, **outside** the repository checkout, so they never appear in the repository's `git status`; they are deleted during cleanup. The working tree shows **only** this one new document:

```text
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/wp-calypso_be7e5cc64162.md

# tracked-file modifications (expect none):
(none — no modified/added tracked files)
```

No existing source file was modified, no tests were added, and no dependencies were changed.

---

## 12. Coverage checklist

Final pass confirming every question and every named mechanism is addressed:

| Item | Where addressed | Status |
|------|-----------------|--------|
| **REQ-1** decider | §4 — `NavigationLink.getBackUrl` (`navigation-link/index.jsx:78-115`), href wiring `:183-193`, inert `handleClick` back branch `:125-127`, `main.jsx:805-806` omits `goToPreviousStep` | ✓ |
| **REQ-2** which input wins | §5 — precedence order backUrl > flow position > query args, with observed C/F evidence | ✓ |
| **REQ-3** external override source | §6 — `StepWrapper` connect() `back_to` → `backUrl` (`step-wrapper/index.jsx:273-283`) + all `backUrl` origins | ✓ |
| **REQ-4** precedence rule | §5 — early return `if ( this.props.backUrl ) return this.props.backUrl;` (`navigation-link/index.jsx:83-85`) + precedence table | ✓ |
| **REQ-5** bypassed step-by-step path | §7 — `getPreviousStep` (`navigation-link/index.jsx:47-76`) + helpers + duality | ✓ |
| **REQ-6** per-step observation | §8 — per-step table + canonical jest + NON-CANONICAL raw output + repeated-run determinism | ✓ |
| Named: `getBackUrl` | §4, §5 | ✓ |
| Named: `getPreviousStep` | §7, §8 | ✓ |
| Named: `getPreviousStepName` (duality) | §7.2, §8.3 | ✓ |
| Named: `getStepUrl` (framework prefix, default-flow omission) | §7.1 | ✓ |
| Named: `isFirstStepInFlow` | §7.1 | ✓ |
| Named: `getFilteredSteps` | §7.1 | ✓ |
| Named: `handleClick` | §4.1 | ✓ |
| Named: `StepWrapper.renderBack` + connect() | §6 | ✓ |
| Named: `allowBackFirstStep` first-step defeat | §6.1 | ✓ |
| Named: `saveSignupStep` / `submitSignupStep` `lastKnownFlow` origin | §9 | ✓ |
| Condition A (step-by-step per position) | §8.1, §8.3 | ✓ |
| Condition B (`findIndex === -1` → `pop()`) | §8.1, §8.3 | ✓ |
| Condition C (external `?back_to=/...` override) | §8.1, §8.3 | ✓ |
| Condition D (hardcoded mailbox `backUrl`) | §8.1, §8.3 | ✓ |
| Condition E (cross-flow `lastKnownFlow`) | §8.1, §8.3, §9 | ✓ |
| Condition F (query-arg decoration) | §8.1, §8.3 | ✓ |
| Condition G (`back_to` fails `startsWith('/')` guard) | §6, §8.1, §8.3 | ✓ |
| User phrase "snaps straight to the first step" | §9 | ✓ |
| User phrase "slips out into an entirely different flow" | §9 | ✓ |
| User phrase "never feels truly random" (determinism) | §8.4, §9 | ✓ |
| page.js anchor-click interception (cited) | §4.2 | ✓ |
| Secondary Stepper `/setup` framework | §10 | ✓ |
| Read-only, temp scripts removed, `git status` clean | §3.4, §11.4 | ✓ |

**All six questions and every named function, condition, and user phrase are addressed with `file:line` references and observed output.**

