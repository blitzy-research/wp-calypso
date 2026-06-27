# How the Calypso Signup “Back” Control Chooses Its Destination

> Investigative, code-grounded answer for branch `wp-calypso_be7e5cc64162`.
> Every factual claim below cites `path:line` anchored to repository
> `HEAD = be7e5cc641622d153040491fd5625c6cb83e12eb`. Where behavior is
> *observed*, it was produced by a temporary Node harness that faithfully
> reproduces the real functions (see the Appendix, section **(g)**); the
> repository itself was never modified.

## TL;DR — the one-sentence answer

The Back control’s destination is **not random**; it is a **pure function** of
`(backUrl prop, back_to query arg, signupProgress, flowName, stepName)` computed
by a single method, `NavigationLink.getBackUrl()`
[`client/signup/navigation-link/index.jsx:L78-L115`]. When the destination
“snaps to the first step” it is because the progress-derived previous step is
`null` and the URL builder emits a **step-less** `/start` URL; when it “slips
into a different flow” it is because an **external back target** (an explicit
`backUrl` prop or a `back_to` query argument) is returned **unconditionally**, or
because the resolved previous step carries a different `lastKnownFlow`.

The strict precedence is:

1. **Explicit step `backUrl` prop** (wins unconditionally, even on step 0),
2. **`back_to` query argument** (only if it starts with `/`),
3. **Progress-derived previous step** (the normal step-by-step path),
4. **Static flow position** (a fallback that *never wins on its own*).

The rest of this document proves each clause from the code and confirms the
pattern with a per-step trace.

---

## The anchor: one function decides, and it drives an `<a href>` — not a click handler

Two facts frame everything that follows.

**1. One function computes the destination.** The Back control is the
`NavigationLink` component. When it is rendered with `direction="back"`, the URL
it points at is whatever `getBackUrl()` returns
[`client/signup/navigation-link/index.jsx:L78-L115`].

**2. Navigation happens through the anchor’s `href`, not its `onClick`.** The
rendered button’s `href` is set to the result of `getBackUrl()`:

```jsx
// client/signup/navigation-link/index.jsx:L183-L193
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

The `onClick` handler only performs *programmatic* back-navigation **if a
`goToPreviousStep` callback was passed in**:

```jsx
// client/signup/navigation-link/index.jsx:L117-L132 (back branch)
} else if ( this.props.goToPreviousStep ) {
    this.props.goToPreviousStep();
}
```

…but the flow controller **never passes `goToPreviousStep` to steps.** In
`client/signup/main.jsx` the step component is created with `goToNextStep`,
`goToStep`, `positionInFlow`, and `queryParams`, but `goToPreviousStep` is
absent from the entire file (the step prop wiring is around
[`client/signup/main.jsx:L766-L820`]; `getPositionInFlow()` is at
[`client/signup/main.jsx:L733-L736`]). Because the callback is never provided,
the `else if` is falsy and the Back navigation is driven **entirely** by the
anchor `href` — i.e. by `getBackUrl()`. This is *why* understanding
`getBackUrl()` is sufficient to understand the behavior.

---

## (a) Scope: there are two onboarding frameworks; this is about the legacy one

Calypso ships **two** distinct signup/onboarding frameworks, and only one of
them exhibits the centralized “prop vs. query vs. position” precedence the
question describes:

- **Legacy signup framework — `client/signup`, served at `/start`.** This is the
  subject of this document. Its route controller lives at
  [`client/signup/controller.js:L70`], its flow-controller component at
  [`client/signup/main.jsx:L113`], its shared step chrome at
  [`client/signup/step-wrapper/index.jsx:L15`], and the actual Back/Skip control at
  [`client/signup/navigation-link/index.jsx:L17`]. In this framework a **single**
  function — `NavigationLink.getBackUrl()` — centralizes the decision, taking
  the step’s props, the query string, and the recorded progress as inputs.

- **Newer “stepper” framework — `client/landing/stepper`, served at `/setup`.**
  This framework is **out of the primary subject**. It implements navigation
  declaratively: each flow supplies its own `useStepNavigation` hook with a
  bespoke `goBack()`, so there is no single centralized precedence among
  prop/query/position. (`client/signup/utils.js` even branches on the `/setup`
  pathname when assembling URLs — see `getStepUrl` framework detection at
  [`client/signup/utils.js:L57-L61`] — but the Back *decision* logic discussed
  here is the legacy framework’s.)

**Why this distinction matters:** the user described “flow position, component
props, and query string arguments” competing for a single decision. That
description matches the legacy framework’s `getBackUrl()` precisely, so the
remainder of this document analyzes `client/signup`.

---

## (b) The precedence rule, with annotated code

The effective destination is resolved in **two layers**.

### Layer 1 — `connect()` resolves the *effective* `backUrl` prop

`StepWrapper` is connected to Redux. Its `mapStateToProps` decides what the
`backUrl` prop will be before `NavigationLink` ever runs:

```jsx
// client/signup/step-wrapper/index.jsx:L273-L283
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

Two precedence facts come straight from these lines:

- An **explicit `backUrl` prop passed by a step** (`ownProps.backUrl`) wins over
  the query string, because of the `??` at
  [`client/signup/step-wrapper/index.jsx:L277`].
- The **`back_to` query argument is accepted only if it starts with `/`**
  [`client/signup/step-wrapper/index.jsx:L274-L275`]. A non-path value (e.g. an
  absolute `https://…` URL) is discarded (`backTo` becomes `undefined`).

### Layer 2 — `getBackUrl()` makes the final decision

```jsx
// client/signup/navigation-link/index.jsx:L78-L115
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

The decisive line is the **unconditional early return** at
[`client/signup/navigation-link/index.jsx:L83-L85`]: *if any* `backUrl` is set,
it is returned immediately, **with no eligibility check** — no test of step
position, progress, or flow. Only when `backUrl` is absent does the function
fall through to the progress-derived computation
[`client/signup/navigation-link/index.jsx:L98-L114`], whose `previousStep.lastKnownFlow || this.props.flowName`
at [`client/signup/navigation-link/index.jsx:L109`] is the mechanism that can
send Back into a *different* flow.

### The rule, stated unambiguously

| Priority | Input | Where it comes from | Behavior |
|---|---|---|---|
| **1 (highest)** | Explicit step `backUrl` prop | A step renders `<StepWrapper backUrl=… />` (e.g. WooCommerce transfer [`client/signup/steps/woocommerce-install/transfer/index.tsx:L75`]) | Returned immediately by `getBackUrl()`; **no eligibility check** [`client/signup/navigation-link/index.jsx:L83-L85`] |
| **2** | `back_to` query-string arg | `getCurrentQueryArguments(state)?.back_to`, kept **only if** it starts with `/` [`client/signup/step-wrapper/index.jsx:L274-L275`] | Becomes the `backUrl` prop via `ownProps.backUrl ?? backTo` [`client/signup/step-wrapper/index.jsx:L277`], then behaves exactly like priority 1 |
| **3** | Progress-derived previous step | `getPreviousStep()` over filtered/sorted `signupProgress` [`client/signup/navigation-link/index.jsx:L47-L76`] | The normal step-by-step Back; may cross flows via `previousStep.lastKnownFlow` [`client/signup/navigation-link/index.jsx:L109`] |
| **4 (lowest)** | Static flow position | `flow.steps` order via `getFilteredSteps`/`isFirstStepInFlow` [`client/signup/utils.js:L137-L150`, `client/signup/utils.js:L28-L31`] | **Fallback only.** Position never wins by itself; it participates *only* through `signupProgress`, and when progress yields no previous step the result collapses to a step-less URL → the first step |

**Rationale / why this ordering is real and not assumed:** the ordering is a
direct reading of control flow. Layer 1’s `??` makes the prop beat the query
arg; Layer 2’s `getBackUrl()` returns the prop before it ever calls
`getPreviousStep()`; and `getPreviousStep()` is the only place flow position is
consulted, and only via the `signupProgress` array. There is no branch in which
a bare position index overrides a present `backUrl`.

---

## (c) Why Back sometimes “snaps straight to the first step”

This happens when the **progress-derived previous step is `null`** and the URL
builder therefore emits a **step-less** URL.

### Step 1 — `getPreviousStep()` returns `{ stepName: null }`

```jsx
// client/signup/navigation-link/index.jsx:L47-L76
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

`previousStep.stepName` is `null` in three situations:

1. **The current step is the first step in the flow** —
   `isFirstStepInFlow(...)` is true [`client/signup/navigation-link/index.jsx:L50-L52`].
2. **There is no usable recorded progress** — after filtering to the flow’s
   steps and removing skipped ones, the array is empty
   [`client/signup/navigation-link/index.jsx:L61-L63`].
3. **Index underflow** — the current step is found at index `0` of progress, so
   `filteredProgressedSteps[ -1 ]` is `undefined` and the `|| previousStep`
   fallback yields `{ stepName: null }` [`client/signup/navigation-link/index.jsx:L75`].

### Step 2 — `getStepUrl()` builds a *step-less* URL from a null step

```js
// client/signup/utils.js:L45-L69
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

The crucial line is [`client/signup/utils.js:L54`]:
`const step = stepName ? '/'+stepName : '';`. When `stepName` is `null`/falsy,
the step segment is the **empty string**. For the default `onboarding` flow the
flow name is omitted on `/start` [`client/signup/utils.js:L63-L67`], so the
whole URL collapses to just **`/start`**. Routing `/start` (with no step
segment) lands the user on the **first step** of the flow. That is exactly the
“snaps to the first step” symptom.

> **Rationale:** this is not a special “go to first step” instruction anywhere;
> it is an emergent consequence of building a URL from a `null` step. The system
> didn’t *decide* to jump to step 1 — it produced a URL that *happens* to route
> there.

### A separate “lands on the first step” vector (context — NOT the Back button)

So the reader does not conflate mechanisms: the flow controller **also**
redirects on mount when a **non-resumable** flow is deep-linked at a non-zero
position:

```jsx
// client/signup/main.jsx:L171-L194
if (
    canResumeFlow( this.props.flowName, this.props.progress, this.props.isLoggedIn ) &&
    ! this.isCurrentStepRemovedFromFlow()
) {
    return;
}

if ( this.getPositionInFlow() !== 0 ) {
    // Flow is not resumable; redirect to the beginning of the flow.
    const destinationStep = flows.getFlow( this.props.flowName, this.props.isLoggedIn )
        .steps[ 0 ];
    this.setState( { resumingStep: destinationStep } );
    const locale = ! this.props.isLoggedIn ? this.props.locale : '';
    return page.redirect(
        getStepUrl(
            this.props.flowName,
            destinationStep,
            undefined,
            locale,
            this.getCurrentFlowSupportedQueryParams()
        )
    );
}
```

This is a **distinct** mechanism (a mount-time redirect, not the Back control),
but it can produce a similar “I ended up on step 1” experience, which is part of
why the overall behavior *feels* unpredictable.

---

## (d) Why Back sometimes “slips out into an entirely different flow”

There are **two** code paths that produce this, both reachable from
`getBackUrl()`.

### Path 1 — an external back target overrides everything (the “quiet override”)

This is the unconditional early return again
[`client/signup/navigation-link/index.jsx:L83-L85`]:

```jsx
if ( this.props.backUrl ) {
    return this.props.backUrl;
}
```

Because there is **no eligibility check**, a `backUrl` that points outside the
current signup flow is honored verbatim — even on step 0. Two things make this
an effective “quiet override even when the step shouldn’t be eligible”:

- The `backUrl` can be supplied **externally** via the `back_to` query argument,
  which `connect()` turns into the prop [`client/signup/step-wrapper/index.jsx:L274-L277`].
- The first-step render **guard** is *defeated* whenever a `backUrl` exists,
  because `StepWrapper` forces `allowBackFirstStep` true:

```jsx
// client/signup/step-wrapper/index.jsx:L50-L70 (renderBack, excerpt)
<NavigationLink
    direction="back"
    …
    backUrl={ this.props.backUrl }
    rel={ this.props.isExternalBackUrl ? 'external' : '' }
    …
    allowBackFirstStep={ this.props.allowBackFirstStep || !! this.props.backUrl }
    …
/>
```

The render guard normally hides Back on step 0
[`client/signup/navigation-link/index.jsx:L154-L161`]:

```jsx
if (
    this.props.positionInFlow === 0 &&
    this.props.direction === 'back' &&
    ! this.props.stepSectionName &&
    ! this.props.allowBackFirstStep
) {
    return null;
}
```

…but `allowBackFirstStep` is `true` whenever `!! this.props.backUrl` is true
[`client/signup/step-wrapper/index.jsx:L65`], so the button **renders and
overrides even at position 0** — the precise “shouldn’t be eligible, yet it
takes control” situation.

**A concrete example of leaving the flow** is the WooCommerce transfer step,
which hardcodes an external `backUrl`:

```tsx
// client/signup/steps/woocommerce-install/transfer/index.tsx:L70-L80 (excerpt)
<StepWrapper
    className="transfer__step-wrapper"
    flowName="woocommerce-install"
    hideBack={ ! hasFailed }
    backUrl={ `/woocommerce-installation/${ domain }` }
    …
/>
```

Other steps that pass an explicit `backUrl` (and therefore exercise the Tier-1
override) include `difm-site-picker` [`client/signup/steps/difm-site-picker/index.tsx:L124`],
`new-or-existing-site` [`client/signup/steps/new-or-existing-site/index.tsx:L77`],
`site-options` [`client/signup/steps/site-options/index.tsx:L129`], `domains`
[`client/signup/steps/domains/index.jsx:L1488`, `:L1521`], `emails`
[`client/signup/steps/emails/index.jsx:L144`], and `woocommerce-install/step-store-address`
[`client/signup/steps/woocommerce-install/step-store-address/index.tsx:L217`].

The `back_to` query argument is also explicitly threaded into signup
dependencies for the WooCommerce flow by the route controller
[`client/signup/controller.js:L226-L230`]:

```js
// client/signup/controller.js:L226-L230
if ( 'woocommerce-install' === flowName ) {
    if ( context?.query?.back_to ) {
        // forces back_to update
        context.store.dispatch( updateDependencies( { back_to: context.query.back_to } ) );
    }
```

### Path 2 — the previous step belongs to a different flow (`lastKnownFlow`)

Even without an external override, Back can change flows. When `getBackUrl()`
builds the URL it uses the previous step’s **own** recorded flow if present
[`client/signup/navigation-link/index.jsx:L109`]:

```jsx
return getStepUrl(
    previousStep.lastKnownFlow || this.props.flowName,
    previousStep.stepName,
    …
);
```

If the previous progress entry was recorded under a different `lastKnownFlow`,
the Back URL is built for **that** flow, not the current one — so the user is
sent into a different flow’s step.

---

## (e) The normal step-by-step path that the override bypasses

The “expected” one-step-backward behavior is the **progress-derived** path, and
it is exactly what the Tier-1/Tier-2 overrides short-circuit:

1. `getBackUrl()` calls `getPreviousStep(flowName, signupProgress, stepName)`
   [`client/signup/navigation-link/index.jsx:L98`].
2. `getPreviousStep()` builds the candidate list with `getFilteredSteps()`,
   which keeps only progress entries whose `stepName` is part of the current
   flow and **sorts them into flow order**:

```js
// client/signup/utils.js:L137-L150
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

3. It then picks the entry **before** the current step (or `pop()`s the
   furthest-progressed entry when the current step is not yet recorded
   [`client/signup/navigation-link/index.jsx:L71-L73`]).
4. Finally `getStepUrl()` turns that previous step into a path
   [`client/signup/utils.js:L45-L69`].

(There is also a simpler, position-based helper, `getPreviousStepName()`
[`client/signup/utils.js:L85-L88`], which returns the flow-array neighbor; it is
not the path the Back control uses, but it shows the “pure position” notion the
question contrasts against.)

The key point for the question: **this entire path is skipped** the moment a
`backUrl` prop or a valid `back_to` query arg is present, because `getBackUrl()`
returns before reaching `getPreviousStep()` [`client/signup/navigation-link/index.jsx:L83-L85`].

---

## (f) Per-step computed-destination trace (empirical confirmation)

The table below is the **observed** output of running a faithful reproduction of
the real functions (`getStepUrl`, `getPreviousStep`, `getFilteredSteps`,
`isFirstStepInFlow`, plus the `connect()`/render-guard logic) against the
default `onboarding` flow. See section **(g)** for the harness and its raw
output; the values here are copied from that output verbatim.

**Flow under test:** `onboarding`, whose steps are
`[ userSocialStep, 'domains', 'plans' ]`
[`client/signup/config/flows-pure.js:L132-L133`]. With the `signup/social-first`
feature flag **disabled**, `userSocialStep === 'user'`
[`client/signup/config/flows-pure.js:L13-L14`], so the steps are
`['user','domains','plans']`. (See the note after the table about the bundled
default.)

| Scenario | pos 0 `user` | pos 1 `domains` | pos 2 `plans` |
|----------|--------------|-----------------|---------------|
| **A.** Normal (progress = prior steps) | hidden | `/start/user` | `/start/domains` |
| **B.** Empty progress | hidden | `/start` (first step) | `/start` (first step) |
| **C.** Explicit `backUrl=/woocommerce-installation/example.com` | `/woocommerce-installation/example.com` | `/woocommerce-installation/example.com` | `/woocommerce-installation/example.com` |
| **D.** `?back_to=/home` | `/home` | `/home` | `/home` |
| **E.** Prior step `lastKnownFlow='other-flow'` | hidden | `/start/other-flow/user` | `/start/other-flow/domains` |
| **F.** `?back_to=https://evil.example` (not `/`-prefixed) | hidden | `/start/user` | `/start/domains` |

Reading the table against the symptoms:

- **Scenario B confirms “snaps to the first step.”** With empty progress,
  `getPreviousStep()` returns `{ stepName: null }`, `getStepUrl()` emits the
  step-less `/start` [`client/signup/utils.js:L54`], and `/start` routes to the
  first step.
- **Scenarios C and D confirm “leaves the flow entirely.”** The external target
  is returned for **every** position — *including position 0* — proving the
  unconditional override and the defeated first-step guard.
- **Scenario E confirms a flow change without any external target**, via
  `previousStep.lastKnownFlow` [`client/signup/navigation-link/index.jsx:L109`].
- **Scenario F confirms the `startsWith('/')` guard.** A non-path `back_to` is
  rejected [`client/signup/step-wrapper/index.jsx:L275`], so Back falls back to
  the normal progress-derived destination. (It also never leaks into the URL,
  because `back_to` is not among the `onboarding` flow’s supported query params —
  `getCurrentFlowSupportedQueryParams()` at [`client/signup/main.jsx:L633-L652`]
  only forwards `siteId`, `siteSlug`, `flags`, and the flow’s declared
  `providesDependenciesInQuery`, which for `onboarding` is just `['coupon']`
  [`client/signup/config/flows-pure.js:L138`].)
- **“hidden”** marks the positions where the first-step render guard suppressed
  the button [`client/signup/navigation-link/index.jsx:L154-L161`] — note it is
  hidden at position 0 in every scenario *except* C and D, where a `backUrl`
  forces it to render.

> **Note on the bundled default flow.** The AAP-mandated trace uses
> `['user','domains','plans']` (the `signup/social-first`-disabled case). In the
> repository’s committed environment configs the flag is **enabled**
> (`"signup/social-first": true` in `config/development.json:L184`,
> `config/horizon.json:L119`, `config/production.json:L152`,
> `config/stage.json:L149`, `config/test.json:L113`, `config/wpcalypso.json:L154`),
> which makes `userSocialStep === 'user-social'`
> [`client/signup/config/flows-pure.js:L13-L14`]. The **logic and the pattern are
> identical** either way — only the literal first-step slug changes (`user` →
> `user-social`, and the position-1 destination `/start/user` → `/start/user-social`).
> The trace is presented with `user` to match the AAP and to keep the
> precedence story slug-agnostic.

---

## (g) Reproducible observation appendix

Per the engagement’s rules, behavior was observed with a **temporary** Node
script kept **entirely outside** the repository tree, at
`/tmp/aap_probe/back_probe.js`. It was executed on **Node `v22.23.1`** (which
satisfies the repository’s `engines.node` `^v22.9.0` and `.nvmrc` `22.9.0`); to keep the verbatim output below stable across
patch/minor releases in that range, the harness prints only the **major**
version line (`Node v22.x`). It uses **Node built-ins only** (`URL`, `URLSearchParams`), installs nothing, and
was **deleted after use** — the repository was never modified. Its content is
preserved here verbatim as a reproducible artifact.

> **Fidelity note.** The harness reproduces the *pure* routing logic: `getStepUrl`
> (including the L54 step-less rule and the `/start` default-flow omission),
> `getPreviousStep`, `getFilteredSteps`, and `isFirstStepInFlow`, plus the
> `connect()` `backUrl` resolution, the `allowBackFirstStep` rule, and the
> first-step render guard. lodash helpers (`filter`/`sortBy`/`findIndex`) are
> expressed with native array equivalents; the results are identical for these
> inputs. Running in Node, `window` is undefined, so `getStepUrl`’s framework
> detection resolves to `/start` exactly as it does on the server.

### Harness script (`/tmp/aap_probe/back_probe.js`)

```js
#!/usr/bin/env node
/*
 * back_probe.js — TEMPORARY observation harness (NOT part of the repository).
 *
 * Purpose: empirically confirm the destination computed for the legacy Calypso
 * signup "Back" control for every step position of the default `onboarding`
 * flow, under six input scenarios (A-F).
 *
 * It FAITHFULLY REPRODUCES the real, pure functions from client/signup at
 * HEAD be7e5cc641622d153040491fd5625c6cb83e12eb:
 *   - getStepUrl            (client/signup/utils.js:L45-L69)  incl. the L54 step-less rule
 *                                                             and the default-flow omission on /start (L63-L67)
 *   - getFilteredSteps      (client/signup/utils.js:L137-L150)
 *   - isFirstStepInFlow     (client/signup/utils.js:L28-L31)
 *   - getPreviousStep       (client/signup/navigation-link/index.jsx:L47-L76)
 *   - effective backUrl     (client/signup/step-wrapper/index.jsx:L274-L277)
 *   - first-step render guard (client/signup/navigation-link/index.jsx:L154-L161)
 *   - getBackUrl            (client/signup/navigation-link/index.jsx:L78-L115)
 *
 * Node built-ins only (URL, URLSearchParams). Run: `node back_probe.js`.
 */

'use strict';

// ---- Flow definition (client/signup/config/flows-pure.js:L132-L133) ---------
// onboarding steps = [ userSocialStep, 'domains', 'plans' ]; with
// `signup/social-first` DISABLED, userSocialStep === 'user' (client/signup/config/flows-pure.js:L13-L14).
// NOTE: the bundled config enables the flag, making the live first step
// 'user-social'; the destination LOGIC/pattern below is identical either way —
// only the literal first-step slug changes.
const ONBOARDING_STEPS = ['user', 'domains', 'plans'];
const DEFAULT_FLOW_NAME = 'onboarding'; // client/signup/config/flows.js:L243-L245,L250

function getFlow(flowName) {
	if (flowName === DEFAULT_FLOW_NAME) {
		return { steps: ONBOARDING_STEPS };
	}
	// Any non-default flow used only for the lastKnownFlow scenario; its steps
	// are irrelevant to URL assembly (only the flow name segment is used).
	return { steps: [] };
}

// ---- addQueryArgs (client/lib/url/add-query-args.ts) -------------------------
// Faithful, dependency-free reproduction for path-relative URLs: drop null/
// undefined args (pickBy, L28); when no args remain, return the URL unchanged.
function addQueryArgs(args, url) {
	const clean = {};
	for (const k of Object.keys(args || {})) {
		if (args[k] != null) clean[k] = String(args[k]);
	}
	const keys = Object.keys(clean);
	if (keys.length === 0) {
		return url; // no query string appended
	}
	const search = new URLSearchParams();
	for (const k of keys) search.set(k, clean[k]);
	return `${url}?${search.toString()}`;
}

// ---- getStepUrl (client/signup/utils.js:L45-L69) ----------------------------
function getStepUrl(flowName, stepName, stepSectionName, localeSlug, params = {}, frameworkParam = null) {
	const flow = flowName ? `/${flowName}` : '';
	const step = stepName ? `/${stepName}` : ''; // L54: step-less when stepName is falsy
	const section = stepSectionName ? `/${stepSectionName}` : '';
	const locale = localeSlug ? `/${localeSlug}` : '';
	// In Node there is no `window`, so this resolves to '/start' exactly as the
	// real expression does on the server / when not under /setup.
	const framework =
		frameworkParam ||
		(typeof window !== 'undefined' && window.location.pathname.startsWith('/setup') ? '/setup' : '/start');

	const url =
		flowName === DEFAULT_FLOW_NAME && framework === '/start'
			? framework + step + section + locale // default flow name omitted on /start
			: framework + flow + step + section + locale;
	return addQueryArgs(params, url);
}

// ---- getFilteredSteps (client/signup/utils.js:L137-L150) --------------------
function getFilteredSteps(flowName, progress) {
	const flow = getFlow(flowName);
	if (!flow) return [];
	return progress
		.filter((step) => flow.steps.includes(step.stepName))
		.slice()
		.sort((a, b) => flow.steps.indexOf(a.stepName) - flow.steps.indexOf(b.stepName));
}

// ---- isFirstStepInFlow (client/signup/utils.js:L28-L31) ---------------------
function isFirstStepInFlow(flowName, stepName) {
	return getFlow(flowName).steps.indexOf(stepName) === 0;
}

// ---- getPreviousStep (client/signup/navigation-link/index.jsx:L47-L76) ------
function getPreviousStep(flowName, signupProgress, currentStepName) {
	const previousStep = { stepName: null }; // L48

	if (isFirstStepInFlow(flowName, currentStepName)) {
		return previousStep; // L50-L52
	}

	const filteredProgressedSteps = getFilteredSteps(flowName, signupProgress).filter(
		(step) => !step.wasSkipped
	); // L56-L60
	if (filteredProgressedSteps.length === 0) {
		return previousStep; // L61-L63
	}

	const currentStepIndexInProgress = filteredProgressedSteps.findIndex(
		(step) => step.stepName === currentStepName
	); // L66-L68

	if (currentStepIndexInProgress === -1) {
		return filteredProgressedSteps.pop(); // L71-L73: current step not yet in progress
	}

	return filteredProgressedSteps[currentStepIndexInProgress - 1] || previousStep; // L75
}

// ---- Effective backUrl + render guard + getBackUrl --------------------------
// Models step-wrapper connect (client/signup/step-wrapper/index.jsx:L274-L277), allowBackFirstStep (client/signup/step-wrapper/index.jsx:L65), the
// first-step render guard (client/signup/navigation-link/index.jsx:L154-L161) and getBackUrl (client/signup/navigation-link/index.jsx:L78-L115).
function computeBack({ ownBackUrl, backToQueryArg, signupProgress, flowName, stepName, positionInFlow }) {
	// client/signup/step-wrapper/index.jsx:L274-L275 — back_to accepted only if it starts with '/'
	const backTo = backToQueryArg && backToQueryArg.startsWith('/') ? backToQueryArg : undefined;
	// client/signup/step-wrapper/index.jsx:L277 — explicit prop wins over query arg
	const backUrl = ownBackUrl ?? backTo;

	const stepSectionName = ''; // no sub-steps in this trace
	// client/signup/step-wrapper/index.jsx:L65 — allowBackFirstStep || !!backUrl
	const allowBackFirstStep = false || !!backUrl;

	// client/signup/navigation-link/index.jsx:L154-L161 — first-step render guard
	if (positionInFlow === 0 && !stepSectionName && !allowBackFirstStep) {
		return 'hidden';
	}

	// getBackUrl (direction === 'back'): L83-L85 unconditional override
	if (backUrl) {
		return backUrl;
	}

	// L98 + L108-L114 — normal progress-derived previous step
	const previousStep = getPreviousStep(flowName, signupProgress, stepName);
	const locale = ''; // userLoggedIn assumed true (L106) -> no locale segment
	// queryParams = onboarding flow-supported params (getCurrentFlowSupportedQueryParams,
	// client/signup/main.jsx:L633-L652). `back_to` is NOT supported by onboarding -> dropped.
	const queryParams = {};
	return getStepUrl(
		previousStep.lastKnownFlow || flowName, // L109 — cross-flow via lastKnownFlow
		previousStep.stepName,
		stepSectionName,
		locale,
		queryParams
	);
}

// ---- Build the "prior steps" progress for a given position ------------------
// Forward navigation state: steps before the current one are completed; the
// current step is not yet in `signupProgress`.
function priorProgress(position, { lastKnownFlow } = {}) {
	return ONBOARDING_STEPS.slice(0, position).map((stepName) =>
		lastKnownFlow ? { stepName, lastKnownFlow } : { stepName }
	);
}

// ---- Scenarios A-F ----------------------------------------------------------
const scenarios = [
	{
		id: 'A',
		label: 'Normal (progress = prior steps)',
		inputs: (pos) => ({ signupProgress: priorProgress(pos) }),
	},
	{
		id: 'B',
		label: 'Empty progress',
		inputs: () => ({ signupProgress: [] }),
	},
	{
		id: 'C',
		label: 'Explicit backUrl=/woocommerce-installation/example.com',
		inputs: () => ({ ownBackUrl: '/woocommerce-installation/example.com', signupProgress: [] }),
	},
	{
		id: 'D',
		label: '?back_to=/home',
		inputs: (pos) => ({ backToQueryArg: '/home', signupProgress: priorProgress(pos) }),
	},
	{
		id: 'E',
		label: "Prior step lastKnownFlow='other-flow'",
		inputs: (pos) => ({ signupProgress: priorProgress(pos, { lastKnownFlow: 'other-flow' }) }),
	},
	{
		id: 'F',
		label: '?back_to=https://evil.example (not /-prefixed)',
		inputs: (pos) => ({ backToQueryArg: 'https://evil.example', signupProgress: priorProgress(pos) }),
	},
];

const FLOW = DEFAULT_FLOW_NAME;

// Major Node version only, so this block stays byte-stable across patch/minor
// releases within the repo's pinned ^v22.9.0 range (.nvmrc 22.9.0).
console.log('Node ' + process.version.split('.')[0] + '.x');
console.log('Flow: ' + FLOW + ' steps=' + JSON.stringify(ONBOARDING_STEPS));
console.log('');

// Columns are separated by ' | ' so that long values (e.g. scenario C) stay
// unambiguous regardless of width.
const COL = 24;
function cell(v) {
	const s = String(v);
	return s.length < COL ? s.padEnd(COL) : s;
}

console.log(
	[cell('Scenario'), cell('pos 0 (user)'), cell('pos 1 (domains)'), cell('pos 2 (plans)')].join(' | ')
);
console.log('-'.repeat(96));

for (const sc of scenarios) {
	const cells = ONBOARDING_STEPS.map((stepName, pos) =>
		computeBack({
			ownBackUrl: undefined,
			backToQueryArg: undefined,
			flowName: FLOW,
			stepName,
			positionInFlow: pos,
			...sc.inputs(pos),
		})
	);
	console.log(
		[cell(sc.id + '. ' + sc.label)].concat(cells.map(cell)).join(' | ')
	);
}
console.log('-'.repeat(96));
console.log('Legend: "hidden" = first-step render guard suppressed the button');
console.log('        (client/signup/navigation-link/index.jsx:L154-L161).');
```

### Harness output (verbatim)

```text
Node v22.x
Flow: onboarding steps=["user","domains","plans"]

Scenario                 | pos 0 (user)             | pos 1 (domains)          | pos 2 (plans)           
------------------------------------------------------------------------------------------------
A. Normal (progress = prior steps) | hidden                   | /start/user              | /start/domains          
B. Empty progress        | hidden                   | /start                   | /start                  
C. Explicit backUrl=/woocommerce-installation/example.com | /woocommerce-installation/example.com | /woocommerce-installation/example.com | /woocommerce-installation/example.com
D. ?back_to=/home        | /home                    | /home                    | /home                   
E. Prior step lastKnownFlow='other-flow' | hidden                   | /start/other-flow/user   | /start/other-flow/domains
F. ?back_to=https://evil.example (not /-prefixed) | hidden                   | /start/user              | /start/domains          
------------------------------------------------------------------------------------------------
Legend: "hidden" = first-step render guard suppressed the button
        (client/signup/navigation-link/index.jsx:L154-L161).
```

To reproduce: save the script to `/tmp/aap_probe/back_probe.js` and run
`node /tmp/aap_probe/back_probe.js`. The printed table matches section **(f)**.

---

## (h) Conclusion: the behavior is deterministic, not random

The destination of the Back control is a **pure function** of five inputs:

```
backDestination = f( backUrl prop, back_to query arg, signupProgress, flowName, stepName )
```

- There is **no randomness** anywhere in `getBackUrl()`, `getPreviousStep()`,
  `getFilteredSteps()`, or `getStepUrl()` — they read props, the query string,
  and the recorded progress array, and compute a string.
- The reason it *feels* unpredictable is that two of those inputs are
  **invisible** at the moment of clicking: the accumulated `signupProgress`
  (which determines whether a previous step even exists, and which flow it
  belongs to) and **externally supplied back targets** (`backUrl` props and the
  `back_to` query argument).
- Made explicit, the rule is simple and total:
  1. an explicit `backUrl` prop wins unconditionally
     [`client/signup/navigation-link/index.jsx:L83-L85`];
  2. otherwise a `/`-prefixed `back_to` query arg becomes that prop
     [`client/signup/step-wrapper/index.jsx:L274-L277`];
  3. otherwise the previous step is derived from `signupProgress` and may carry
     its own `lastKnownFlow` [`client/signup/navigation-link/index.jsx:L98-L109`];
  4. and when no previous step can be derived, the URL is **step-less** and
     routes to the first step [`client/signup/utils.js:L54`].

So: “snaps to the first step” = a `null` previous step → step-less `/start`;
“slips into a different flow” = an external override or a previous step from a
different `lastKnownFlow`. Both are deterministic outcomes of the single
`getBackUrl()` decision, confirmed end-to-end by the per-step trace in section
**(f)**.

---

### Appendix — reference files consulted (read-only)

| File | Role in the Back decision |
|------|---------------------------|
| `client/signup/navigation-link/index.jsx` | `getBackUrl()` (`client/signup/navigation-link/index.jsx:L78-L115`), `getPreviousStep()` (`client/signup/navigation-link/index.jsx:L47-L76`), render guard (`client/signup/navigation-link/index.jsx:L154-L161`), `href`-driven nav (`client/signup/navigation-link/index.jsx:L183-L193`) |
| `client/signup/step-wrapper/index.jsx` | effective `backUrl` via `connect()` (`client/signup/step-wrapper/index.jsx:L273-L283`); `allowBackFirstStep \|\| !!backUrl` (`client/signup/step-wrapper/index.jsx:L65`) |
| `client/signup/utils.js` | `getStepUrl` (`client/signup/utils.js:L45-L69`, step-less `client/signup/utils.js:L54`), `getFilteredSteps` (`client/signup/utils.js:L137-L150`), `isFirstStepInFlow` (`client/signup/utils.js:L28-L31`), `getPreviousStepName` (`client/signup/utils.js:L85-L88`) |
| `client/signup/main.jsx` | `getPositionInFlow` (`client/signup/main.jsx:L733-L736`), non-resumable first-step redirect (`client/signup/main.jsx:L171-L194`), step prop wiring without `goToPreviousStep` (`client/signup/main.jsx:L766-L820`) |
| `client/signup/controller.js` | `back_to` dependency dispatch for `woocommerce-install` (`client/signup/controller.js:L226-L230`) |
| `client/signup/config/flows-pure.js` | `onboarding` steps (`client/signup/config/flows-pure.js:L132-L133`), `userSocialStep` (`client/signup/config/flows-pure.js:L13-L14`, `client/signup/config/flows-pure.js:L30`) |
| `client/signup/config/flows.js` | `defaultFlowName === 'onboarding'` (`client/signup/config/flows.js:L243-L245`, `client/signup/config/flows.js:L250`) |
| `client/signup/steps/woocommerce-install/transfer/index.tsx` | concrete external `backUrl` (`client/signup/steps/woocommerce-install/transfer/index.tsx:L75`) |
| `client/lib/url/add-query-args.ts` | query-arg serialization used by `getStepUrl` (`client/lib/url/add-query-args.ts:L12`) |
| `docs/routing.md`, `docs/isomorphic-routing.md` | background on Calypso routing conventions |

*All citations anchored to `HEAD = be7e5cc641622d153040491fd5625c6cb83e12eb`.*
