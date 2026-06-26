# Calypso Reader — Onboarding Q&A

This document answers four concrete questions a new engineer asked while getting the **Reader** section of [Automattic/wp-calypso](https://github.com/Automattic/wp-calypso) (Calypso) running locally and reasoning about its behavior. Every system claim below is grounded in the actual source code and carries a `[path:locator]` citation (repo‑relative path plus `L<line>` or `L<start>-L<end>`). Where it strengthens an answer, the *rationale* — the chain of reasoning from code to conclusion — is spelled out, not just the value.

The four questions are:

1. **Development server & ports** — what port does the dev server bind to, how do you know when it is *fully* ready, and is the local architecture single‑port or multi‑port (separate ports for hot reloading vs. API)?
2. **Reader stream API & Redux actions** — once the Reader view loads, which REST endpoint(s) populate the stream, and which Redux action *types* fire during the initial load?
3. **Authentication detection & storage** — how does the app decide whether a user is logged in *before* rendering, and which storage mechanisms (cookies, `localStorage`, IndexedDB, server‑injected state) does it inspect?
4. **Sidebar responsive design** — what are the exact margin/padding values on the sidebar header, which CSS custom properties drive the layout `calc()`s, and at which viewport widths does the layout change?

> **Reading the citations.** A citation such as `[config/_shared.json:L25]` means line 25 of `config/_shared.json`, relative to the repository root. All locators were verified by reading the files directly in this checkout.

---

## Table of Contents

- [1. Environment & Prerequisites](#1-environment--prerequisites)
- [2. Q1 — Development Server & Ports](#2-q1--development-server--ports)
- [3. Q2 — Reader Stream API & Redux Actions](#3-q2--reader-stream-api--redux-actions)
- [4. Q3 — Authentication Detection & Storage](#4-q3--authentication-detection--storage)
- [5. Q4 — Sidebar Responsive Design](#5-q4--sidebar-responsive-design)
- [6. Gotchas for New Engineers](#6-gotchas-for-new-engineers)

---

## 1. Environment & Prerequisites

Before reproducing any of the observations below, provision the runtime the repository mandates. Calypso enforces these versions at start time, so getting them right is a prerequisite rather than a suggestion.

### Runtime versions

- **Node.js `^v22.9.0`** is required by the `engines` field: `"node": "^v22.9.0"` `[package.json:L57]`.
- **Yarn `4.0.2`** is the declared package manager: `"packageManager": "yarn@4.0.2"` `[package.json:L422]`; the looser `engines.yarn` floor is `"yarn": "^4.0.0"` `[package.json:L58]`. Activate it through Corepack:

```bash
corepack enable
corepack prepare yarn@4.0.2 --activate
yarn --version   # → 4.0.2
```

### The version is *enforced*, not advisory

The `start` script runs `check-node-version` **first**, before anything is built:

```json
"start": "npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build"
```

— `[package.json:L110]`. Because `check-node-version --package` reads the `engines` block `[package.json:L56-L59]`, launching with an incompatible Node version aborts the whole chain before the build runs. The `check-node-version` dependency itself is pinned at `"check-node-version": "^4.0.2"` `[package.json:L267]`.

> **Runtime‑version reconciliation (call‑out).** Some environment setup notes mention Node 20.x. The repository requires **Node 22.x**, and because of the `check-node-version` gate above, **Node 22.x takes precedence** for building and running. Node 20 will fail the engines check and never reach the build.

### Correct local entry point: `calypso.localhost`, not bare `localhost`

Reach the running app at **`http://calypso.localhost:3000`**, not `http://localhost:3000`. The install guide spells out the two required steps: add `127.0.0.1 calypso.localhost` to your hosts file `[docs/install.md:L9]` and open `calypso.localhost:3000` `[docs/install.md:L11]`. The reason is an authentication/origin constraint, not a vanity hostname: the locally‑running app talks to the **remote** WordPress.com REST API, which "allows only certain origins via our current authentication methods" `[docs/install.md:L38]`. The README quick‑start repeats the same three‑step ritual — install prerequisites, add the hosts entry, run `yarn`/`yarn start`, then open the URL `[README.md:L19-L21]`.

### What `yarn start` actually does (relevant to "readiness" in Q1)

The `start` script chains four steps `[package.json:L110]`:

1. `npx check-node-version --package` — enforce the Node engine.
2. `node bin/welcome.js` — print a welcome banner (a cyan "calypso" ASCII logo `[bin/welcome.js:L6-L11]`; it only pauses for input in the `MOCK_WORDPRESSDOTCOM` path `[bin/welcome.js:L13-L25]`).
3. `yarn run build` — build static assets, CSS, devdocs, server, and (in prod) the client bundle `[package.json:L64]`.
4. `yarn run start-build` — launch the server and pipe its logs through `bunyan`: `BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short` `[package.json:L113]` (bunyan is pinned at `"bunyan": "^1.8.15"` `[package.json:L265]`).

The full command graph behind `yarn start` is documented as a Mermaid diagram in `[docs/yarn-start.md:L5-L20]`.

---

## 2. Q1 — Development Server & Ports

> **Question.** *What port does the development server bind to? How do you know when it's fully ready? Does the architecture use multiple ports (hot reloading, API calls) or is everything served from one place?*

### Short answer

- **Port `3000`, protocol `http`, all network interfaces.** Overridable via the `PORT` env var (and `PROTOCOL`/`HOST`).
- **"Fully ready"** has two distinct meanings: (a) the Express process is *listening* — signalled by the `server.listen(...)` callback / `sendBootStatus('ready')`; and (b) the client bundle has *compiled* — signalled by the Webpack "compiled" log that streams through `bunyan`. A first‑time engineer should wait for the Webpack "compiled" message, not just the boot line.
- **Single‑port.** Hot Module Replacement (HMR) is served in‑process on the *same* Express server (port 3000). There is **no** separate HMR port and **no** local API port; data/API calls go to the **remote** WordPress.com REST API.

### Port, protocol, and host: the defaults

The base config sets all three:

```jsonc
// config/_shared.json
"hostname": false,   // L13
"protocol": "http",  // L24
"port": 3000,        // L25
```

— `"hostname": false` `[config/_shared.json:L13]`, `"protocol": "http"` `[config/_shared.json:L24]`, `"port": 3000` `[config/_shared.json:L25]`.

**Why `hostname: false` means "all interfaces."** The server reads the three values from config — `let protocol = config( 'protocol' ); let port = config( 'port' ); let host = config( 'hostname' );` `[client/server/index.js:L11-L13]` — and then passes `host` into `server.listen` only when running in a fork:

```js
// client/server/index.js:L83-L86
server.listen( { port, host: process.env.CALYPSO_IS_FORK ? host : null }, function () {
	// Tell the parent process that Calypso has booted.
	sendBootStatus( 'ready' );
} );
```

For a normal (non‑fork) `yarn start`, `CALYPSO_IS_FORK` is unset, so the ternary passes `host: null` to `server.listen`, and Node binds **all** interfaces. The falsy `hostname: false` from config is therefore the right default — the configured hostname (`calypso.localhost` in development, see below) is used for *logging/URLs*, not for restricting the bind address.

### Overriding the port

The config parser lets environment variables override protocol, host, and port:

```js
// client/server/config/parser.js:L61-L63
data.protocol = process.env.PROTOCOL || data.protocol;
data.hostname = process.env.HOST || data.hostname;
data.port = process.env.PORT || data.port;
```

So `PORT=4000 yarn start` (or `PORT=4000 node build/server.js`) moves the server off 3000 `[client/server/config/parser.js:L63]`.

### The one case the port is *not* 3000

If you opt into the mock backend, the server overrides protocol/port/host to talk to a local `wordpress.com` mock over HTTPS:

```js
// client/server/index.js:L16-L21
if ( process.env.MOCK_WORDPRESSDOTCOM === '1' ) {
	protocol = 'https';
	port = 443;
	host = 'wordpress.com';
	logger.warn( 'Ignoring protocol, port, and hostname configs to mock WordPress.com' );
}
```

This is the only built‑in path that changes the port away from 3000 `[client/server/index.js:L16-L21]`.

### How you know it's "fully ready"

There are two readiness signals, and they answer different questions:

1. **Process is listening.** The authoritative "server is up" signal is the `server.listen(...)` callback, which calls `sendBootStatus( 'ready' )` `[client/server/index.js:L83-L86]`. `sendBootStatus` is an IPC message to a parent process — it no‑ops unless Calypso was spawned as a fork: `function sendBootStatus( status ) { if ( ! process.send ) { return; } process.send( { boot: status } ); }` `[client/server/index.js:L25-L31]`. This is how the desktop app (which runs Calypso in a fork) learns the server is ready.
2. **Boot log line.** Once the server module initializes, it logs the bind URL: `logger.info( 'wp-calypso booted in %dms - %s://%s:%s', ... )` `[client/server/index.js:L33]`. With development config this renders as `http://calypso.localhost:3000`.
3. **Client bundle compiled.** In dev, the app cannot actually render until Webpack finishes its first compile. Requests are intentionally *held* until that point: the dev bundler mounts a `waitForCompiler` middleware **before** the Webpack middleware so early requests queue until the initial build completes `[client/server/bundler/index.js:L100]`. The Webpack "compiled" output is streamed through `bunyan` (the `start-build` script pipes `node build/server.js | bunyan -o short` `[package.json:L113]`).

**Rationale to internalize:** the boot log / `sendBootStatus('ready')` only proves the Express process is *accepting connections*; it does **not** prove the client bundle is ready. The app is "fully ready" for a human only after the Webpack **"compiled successfully"** message appears. Wait for that line.

### Single‑port vs. multi‑port: the architecture

**It is single‑port.** This is the crux of the question, and the proof is structural — both the bundle middleware *and* the HMR middleware are mounted on the **same** Express `app`:

```js
// client/server/bundler/index.js
const webpackMiddleware = require( 'webpack-dev-middleware' ); // L5
const hotMiddleware = require( 'webpack-hot-middleware' );     // L6
// ...
app.use( waitForCompiler );              // L100
app.use( webpackMiddleware( compiler ) ); // L101 — serves the JS bundle
app.use( hotMiddleware( compiler ) );     // L102 — serves HMR (Hot Module Replacement)
```

— the two `require`s at `[client/server/bundler/index.js:L5-L6]` and the two `app.use(...)` mounts at `[client/server/bundler/index.js:L101-L102]`. Because `webpack-hot-middleware` is attached to the same `app` that serves the application, **HMR rides on port 3000 alongside the app HTML and JS**. There is no second dev‑server port for hot reloading.

**Where do API/data calls go, then?** Not to any local port. The WordPress.com REST transport hard‑codes the remote origin: `proxyOrigin: 'https://public-api.wordpress.com'` `[packages/wpcom-xhr-request/src/index.js:L27]`, and the final request URL is assembled as `settings.url = proxyOrigin + basePath + settings.path` `[packages/wpcom-xhr-request/src/index.js:L254]`. So data traffic targets `https://public-api.wordpress.com` (see Q2), never a local API server.

**Conclusion.** Everything you load in the browser — HTML, the JS bundle, and HMR updates — comes from **one** Express process at `http://calypso.localhost:3000`. The only "other" endpoint in the picture is the remote WordPress.com API, not a second local port.

### Canonical entry point and the development hostname

The documented local URL is **`http://calypso.localhost:3000`** `[docs/install.md:L11]`. In development, `config/development.json` overrides the base config so the boot log and generated URLs use the `calypso.localhost` hostname: `"protocol": "http"` `[config/development.json:L6]`, `"hostname": "calypso.localhost"` `[config/development.json:L7]`, `"port": 3000` `[config/development.json:L8]`. As explained above, this configured hostname is used for URLs/logging; the server still binds all interfaces for non‑fork runs (the `host: ... ? host : null` ternary `[client/server/index.js:L83]`).

| Aspect | Value | Source |
| --- | --- | --- |
| Default port | `3000` | `[config/_shared.json:L25]` |
| Protocol | `http` | `[config/_shared.json:L24]` |
| Bind host | all interfaces (`hostname: false` → `host: null`) | `[config/_shared.json:L13]`, `[client/server/index.js:L83]` |
| Port override | `PORT` env var | `[client/server/config/parser.js:L63]` |
| Dev hostname (for URLs) | `calypso.localhost` | `[config/development.json:L7]` |
| "Listening" signal | `server.listen(...)` → `sendBootStatus('ready')` | `[client/server/index.js:L83-L86]` |
| "Renderable" signal | Webpack "compiled" log (via `bunyan`) | `[client/server/bundler/index.js:L100]`, `[package.json:L113]` |
| HMR transport | in‑process, same port 3000 | `[client/server/bundler/index.js:L102]` |
| Data/API transport | remote `https://public-api.wordpress.com` | `[packages/wpcom-xhr-request/src/index.js:L27]` |
| Only non‑3000 case | `MOCK_WORDPRESSDOTCOM=1` → `https`/`443` | `[client/server/index.js:L16-L21]` |

---

## 3. Q2 — Reader Stream API & Redux Actions

> **Question.** *Once Reader loads, what API endpoints get called to populate the stream? What Redux actions fire during the initial load?*

### Short answer

Mounting the Reader's `<Stream>` triggers a fetch that dispatches **`READER_STREAMS_PAGE_REQUEST`**; the data layer resolves the default ("following") stream to **`GET /read/following`** at REST **`v1.2`** with **`number=4`** (the `INITIAL_FETCH` size) against **`https://public-api.wordpress.com`**; on success it dispatches **`READER_POSTS_RECEIVE`** (the posts) followed by **`READER_STREAMS_PAGE_RECEIVE`** (the page of stream items).

```
mount <Stream>
  → dispatch READER_STREAMS_PAGE_REQUEST
    → data layer: GET https://public-api.wordpress.com/rest/v1.2/read/following?number=4&…
      → on success: dispatch READER_POSTS_RECEIVE
                    dispatch READER_STREAMS_PAGE_RECEIVE
```

### Step 1 — The route

The Reader's default route registers the `following` middleware in its chain (guarded by the `reader` feature flag `[client/reader/index.ts:L53]`):

```js
// client/reader/index.ts:L54-L62
page(
	[ '/reader', '/reader/recent/:feed_id' ],
	redirectLoggedOutToDiscover,
	sidebar,
	setSelectedSiteIdByOrigin,
	following,
	makeLayout,
	clientRender
);
```

So visiting `/reader` runs the `following` controller `[client/reader/index.ts:L54-L62]`.

### Step 2 — The controller sets the primary view to the Following stream

```js
// client/reader/controller.js:L84-L99
context.primary = createElement( StreamComponent, {
	key: 'following',
	listName: i18n.translate( 'Followed Sites' ),
	streamKey: 'following',
	startDate,
	recsStreamKey: 'custom_recs_posts_with_images',
	// …
	feedId: context.params.feed_id,
} );
```

The `following` controller (`export function following( context, next )` `[client/reader/controller.js:L50]`) mounts `<Stream streamKey="following">` as the primary view `[client/reader/controller.js:L84-L99]`. The `streamKey: 'following'` `[client/reader/controller.js:L87]` is the value that later selects the REST endpoint.

### Step 3 — Component mount triggers the first fetch

```js
// client/reader/stream/index.jsx
componentDidMount() {           // L221
	const { streamKey } = this.props;
	this.props.resetCardExpansions();
	this.props.viewStream( streamKey, window.location.pathname );
	this.fetchNextPage( {} );   // L225
	// …
}
```

`componentDidMount()` `[client/reader/stream/index.jsx:L221]` calls `this.fetchNextPage( {} )` `[client/reader/stream/index.jsx:L225]`. `fetchNextPage` `[client/reader/stream/index.jsx:L489]` computes a `pageHandle` — which is `null` on the very first load because there is no prior `stream` page — and then dispatches the request:

```js
// client/reader/stream/index.jsx:L501-L502
const pageHandle = stream ? this.getPageHandle( stream.pageHandle, startDate ) : null;
props.requestPage( { feedId: selectedFeedId, streamKey, pageHandle, localeSlug } );
```

The `pageHandle: null` on first load `[client/reader/stream/index.jsx:L501]` is what later selects the *initial* fetch size (see Step 6).

### Step 4 — The action dispatched on load: `READER_STREAMS_PAGE_REQUEST`

The `requestPage` action creator returns a plain action whose **type string** is `READER_STREAMS_PAGE_REQUEST`:

```js
// client/state/reader/streams/actions.js:L28-L50
export function requestPage( { streamKey, feedId, pageHandle, isPoll = false, gap = null, localeSlug = null } ) {
	const streamType = getStreamType( streamKey );
	return {
		type: READER_STREAMS_PAGE_REQUEST,           // L39
		payload: { streamKey, pageHandle, streamType, isPoll, gap, localeSlug, feedId },
	};
}
```

— creator at `[client/state/reader/streams/actions.js:L28-L50]`, type field at `[client/state/reader/streams/actions.js:L39]`. The literal type string is defined as `export const READER_STREAMS_PAGE_REQUEST = 'READER_STREAMS_PAGE_REQUEST';` `[client/state/reader/action-types.ts:L78]`.

### Step 5 — Endpoint resolution: `following` → `/read/following`

The data layer maps each stream type to a REST path. For `following`:

```js
// client/state/data-layer/wpcom/read/streams/index.js:L192-L195
const streamApis = {
	following: {
		path: () => '/read/following',
		// …
	},
	// …
};
```

So `streamKey: 'following'` resolves to path `/read/following` `[client/state/data-layer/wpcom/read/streams/index.js:L194]`.

### Step 6 — The HTTP request: `GET`, `v1.2`, `number = 4`

The data‑layer `requestPage` builder turns the action into an `http(...)` effect:

```js
// client/state/data-layer/wpcom/read/streams/index.js:L358-L405 (excerpted)
export function requestPage( action ) {                       // L358
	const { payload: { streamKey, streamType, feedId, pageHandle, isPoll, gap, … } } = action;
	const api = streamApis[ streamType ];
	const { apiVersion = '1.2', path, query = defaultQueryFn, … } = api;   // L370
	const fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH;             // L380
	let number;
	if ( page ) { number = perPage; } else { number = gap ? PER_GAP : fetchCount; } // L386
	return http( {
		method: 'GET',                          // L396
		path: path( { ...action.payload } ),    // L397 → "/read/following"
		apiVersion,                             // L398 → "1.2"
		query: isPoll ? … : query( { …, number, lang, page }, action.payload ),
		onSuccess: action,
		onFailure: action,
	} );
}
```

Key facts: the request is a **`GET`** `[client/state/data-layer/wpcom/read/streams/index.js:L396]` at default **`apiVersion = '1.2'`** `[client/state/data-layer/wpcom/read/streams/index.js:L370]`. The number of items requested is governed by `const fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH;` `[client/state/data-layer/wpcom/read/streams/index.js:L380]`. On the first page `pageHandle` is `null` (Step 3), so `fetchCount === INITIAL_FETCH`. The fetch‑size constants are:

```js
// client/state/data-layer/wpcom/read/streams/index.js:L160-L163
export const PER_FETCH = 7;
export const INITIAL_FETCH = 4;
const PER_POLL = 40;
const PER_GAP = 40;
```

so the very first load requests **4** items (`INITIAL_FETCH`), and subsequent pages request 7 (`PER_FETCH`) `[client/state/data-layer/wpcom/read/streams/index.js:L160-L163]`.

### Step 7 — The REST base URL

The transport prepends the remote origin `proxyOrigin: 'https://public-api.wordpress.com'` `[packages/wpcom-xhr-request/src/index.js:L27]`, assembling `settings.url = proxyOrigin + basePath + settings.path` `[packages/wpcom-xhr-request/src/index.js:L254]`. The full conceptual call on first load is therefore:

```
GET https://public-api.wordpress.com/rest/v1.2/read/following?number=4&…
```

### Step 8 — The handler and the success actions

A handler is registered for the request type, wiring the fetch builder to the success handler:

```js
// client/state/data-layer/wpcom/read/streams/index.js:L514-L527 (excerpted)
registerHandlers( 'state/data-layer/wpcom/read/streams/index.js', {
	[ READER_STREAMS_PAGE_REQUEST ]: [
		dispatchRequest( { fetch: requestPage, onSuccess: handlePage, onError: noop } ),
	],
	// …
} );
```

— `[client/state/data-layer/wpcom/read/streams/index.js:L514-L527]`. On success, `handlePage( action, data )` `[client/state/data-layer/wpcom/read/streams/index.js:L428]` pushes two actions: first `receivePosts( streamPosts )` `[client/state/data-layer/wpcom/read/streams/index.js:L470]`, then `receivePage( { … } )` `[client/state/data-layer/wpcom/read/streams/index.js:L496-L497]`. Concretely:

- **`receivePosts`** dispatches **`READER_POSTS_RECEIVE`**: `dispatch( { type: READER_POSTS_RECEIVE, posts: normalizedPosts } )` `[client/state/reader/posts/actions.js:L86-L89]` (creator at `[client/state/reader/posts/actions.js:L63]`; literal string at `[client/state/reader/action-types.ts:L52]`).
- **`receivePage`** dispatches **`READER_STREAMS_PAGE_RECEIVE`**: `{ type: READER_STREAMS_PAGE_RECEIVE, payload: { … } }` `[client/state/reader/streams/actions.js:L52-L74]` (type field at `[client/state/reader/streams/actions.js:L63]`; literal string at `[client/state/reader/action-types.ts:L77]`).

### Redux actions during the initial load (summary)

| Order | Action type | Where it's created | Type constant |
| --- | --- | --- | --- |
| 1 | `READER_STREAMS_PAGE_REQUEST` | `requestPage()` on mount | `[client/state/reader/action-types.ts:L78]` |
| 2 | `READER_POSTS_RECEIVE` | `receivePosts()` on success | `[client/state/reader/action-types.ts:L52]` |
| 3 | `READER_STREAMS_PAGE_RECEIVE` | `receivePage()` on success | `[client/state/reader/action-types.ts:L77]` |

> **Note.** `receivePosts` may dispatch `READER_POSTS_RECEIVE` twice — once after "fast" normalization rules and again after "slow" rules resolve `[client/state/reader/posts/actions.js:L86-L98]`. Both carry the same type string.


---

## 4. Q3 — Authentication Detection & Storage

> **Question.** *How does the app know whether someone is logged in before deciding what to render? What storage mechanisms does it check (cookies, `localStorage`, etc.)?*

### Short answer

"Logged in" is, at the moment of rendering, simply **`state.currentUser.id !== null`** in the Redux store. That value is resolved **before** routing/rendering starts (`page.start()`). *How* it gets populated depends on the `wpcom-user-bootstrap` feature flag: in **development/test** (flag **false**) the app does a client‑side `GET /me`; in **production/stage/horizon/wpcalypso** (flag **true**) the server bootstraps the user and injects `window.currentUser`. The browser‑side identity signals it inspects are the **`wordpress_logged_in`** cookie, the **`wpcom_token`** OAuth token (cookie → `localStorage`), the **`wpcom_user_id`** in `localStorage`, and the **IndexedDB** `calypso` persisted‑state cache.

### The selector that answers "logged in?"

```js
// client/state/current-user/selectors.js
export function getCurrentUserId( state ) {       // L6
	return state.currentUser?.id;                  // L7
}
export function isUserLoggedIn( state ) {          // L15
	return getCurrentUserId( state ) !== null;     // L16
}
export function getCurrentUser( state ) {          // L24
	return state?.currentUser?.user ?? null;       // L25
}
```

`isUserLoggedIn(state)` returns `getCurrentUserId(state) !== null`, i.e. `state.currentUser.id !== null` `[client/state/current-user/selectors.js:L6-L17]`; `getCurrentUser(state)` returns the user object or `null` `[client/state/current-user/selectors.js:L24-L26]`. So the entire notion of "logged in" reduces to whether `state.currentUser.id` is set.

### Identity is resolved *before* render

The boot entry point awaits the current user *before* it boots routing:

```js
// client/boot/common.js:L340-L343
export const bootApp = async ( appName, registerRoutes ) => {
	const user = await initializeCurrentUser();
	debug( `Starting ${ appName }. Let's do this.` );
	await boot( user, registerRoutes );
};
```

`bootApp` first `await`s `initializeCurrentUser()` `[client/boot/common.js:L341]`, then calls `boot(user, …)` `[client/boot/common.js:L343]`. Inside `boot`, the store is created with the resolved user and rehydrated from cache (`setStore( reduxStore, getStateFromCache( currentUser?.ID ) )` `[client/boot/common.js:L325]`), the user is written into the store — but only when present:

```js
// client/boot/common.js:L217-L223
const configureReduxStore = ( currentUser, reduxStore ) => {
	debug( 'Executing Calypso configure Redux store.' );
	if ( currentUser && currentUser.ID ) {
		// Set current user in Redux store
		reduxStore.dispatch( setCurrentUser( currentUser ) );
	}
	// …
};
```

`setCurrentUser(currentUser)` is dispatched only if `currentUser && currentUser.ID` `[client/boot/common.js:L220-L222]`. Finally, **`page.start()`** kicks off routing/rendering `[client/boot/common.js:L337]`.

**Rationale:** routing/rendering (`page.start()` at `[client/boot/common.js:L337]`) runs only *after* `initializeCurrentUser()` has resolved and the store has been configured. That ordering is what guarantees the app always knows the logged‑in state before it decides what to render.

### The path depends on the `wpcom-user-bootstrap` flag

`initializeCurrentUser` branches on the flag:

```js
// client/lib/user/shared-utils/initialize-current-user.js:L28-L49 (excerpted)
if ( ! skipBootstrap && config.isEnabled( 'wpcom-user-bootstrap' ) ) {
	if ( window.currentUser ) {
		return window.currentUser;   // server‑injected user (SSR)
	}
	return false;                     // bootstrap enabled but no injected user → logged out
}

let userData;
try {
	userData = await rawCurrentUserFetch();   // client‑side GET /me
} catch ( error ) { /* … */ }
if ( ! userData ) { return false; }
return filterUserObject( userData );
```

— `[client/lib/user/shared-utils/initialize-current-user.js:L28-L49]`. When the flag is **on**, it trusts the server‑injected `window.currentUser` `[client/lib/user/shared-utils/initialize-current-user.js:L29-L31]`. When **off**, it client‑fetches via `rawCurrentUserFetch()`, which is a plain `GET /me`:

```js
// client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L7
export function rawCurrentUserFetch() {
	return wpcom.me().get( { meta: 'flags' } );
}
```

— `[client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L7]`.

The flag's value per environment (verified):

| Environment | `wpcom-user-bootstrap` | Source |
| --- | --- | --- |
| development | **false** | `[config/development.json:L209]` |
| test | **false** | `[config/test.json:L124]` |
| production | **true** | `[config/production.json:L177]` |
| stage | **true** | `[config/stage.json:L173]` |
| horizon | **true** | `[config/horizon.json:L141]` |
| wpcalypso | **true** | `[config/wpcalypso.json:L178]` |

> **Dev‑vs‑prod divergence (a common onboarding surprise).** Because the flag is **false in development/test**, your *local* app determines login by a **client‑side `GET /me`** to the remote API (relying on browser cookies/token), **not** by server bootstrap. In **production/stage/horizon/wpcalypso** the flag is **true**, so the server bootstraps the user and injects `window.currentUser` for SSR. When running locally, expect the client‑fetch path — and remember the `calypso.localhost` origin requirement from §1, since that `GET /me` is a cross‑origin call to `public-api.wordpress.com`.

### Storage mechanisms inspected (exact keys)

1. **Cookie `wordpress_logged_in`** — the *server* bootstrap path reads this to identify the user:
   - `const AUTH_COOKIE_NAME = 'wordpress_logged_in';` `[client/server/user-bootstrap/index.js:L8]`
   - read via `request.cookies[ AUTH_COOKIE_NAME ]` `[client/server/user-bootstrap/index.js:L28]`
   - the server then calls the WP.com `/me` endpoint: `const API_PATH = 'https://public-api.wordpress.com/rest/v1/me';` `[client/server/user-bootstrap/index.js:L13]`.
2. **OAuth token `wpcom_token`** — checked in the cookie first, then `localStorage`:
   - `const TOKEN_NAME = 'wpcom_token';` `[packages/oauth-token/src/index.js:L7]`
   - `getToken()` parses `document.cookie` and returns the cookie value if present, otherwise falls back to `store.get( TOKEN_NAME )` (a `localStorage` abstraction) `[packages/oauth-token/src/index.js:L10-L24]`.
3. **`wpcom_user_id` in `localStorage`** — the persisted user id used to key caches:
   - `getStoredUserId() => store.get( 'wpcom_user_id' )` and `setStoredUserId( userId ) => store.set( 'wpcom_user_id', userId )` `[client/lib/user/store.js:L12-L18]`, where `store` is the `localStorage` abstraction imported at `[client/lib/user/store.js:L1]`.
4. **IndexedDB‑persisted Redux state** — the offline cache of the Redux tree:
   - the module targets `window.indexedDB`; `supportsIDB` bails out if it is missing `[client/lib/browser-storage/index.ts:L36-L56]` and otherwise opens the database `[client/lib/browser-storage/index.ts:L59]`.
   - the database is `const DB_NAME = 'calypso';` `[client/lib/browser-storage/index.ts:L20]` with object store `const STORE_NAME = 'calypso_store';` `[client/lib/browser-storage/index.ts:L22]`; low‑level access is via `idbGet` `[client/lib/browser-storage/index.ts:L105]` and `idbSet` `[client/lib/browser-storage/index.ts:L162]`.
   - this cache is keyed by user id and rehydrated at boot through `getStateFromCache` `[client/state/initial-state.js:L233]`, which `boot()` invokes at `[client/boot/common.js:L325]`.
5. **SSR‑injected `window.currentUser`** — in the bootstrap‑enabled environments the server injects the user object onto `window.currentUser`, which `initializeCurrentUser` returns directly `[client/lib/user/shared-utils/initialize-current-user.js:L29-L31]`.

### Synthesis

The app's notion of "logged in" is the Redux value `state.currentUser.id` `[client/state/current-user/selectors.js:L6-L17]`, populated **before** `page.start()` `[client/boot/common.js:L337,L340-L343]`. *How* it is populated depends on `wpcom-user-bootstrap`: a client‑side `GET /me` in dev/test, or server bootstrap + `window.currentUser` in production. The browser‑side identity signals it inspects are the `wordpress_logged_in` cookie, the `wpcom_token` (cookie → `localStorage`), `wpcom_user_id` (`localStorage`), and the IndexedDB `calypso` persisted‑state cache.


---

## 5. Q4 — Sidebar Responsive Design

> **Question.** *What are the specific margin and padding values on the sidebar header? What CSS custom properties drive layout calculations? At what viewport widths do things change (breakpoints)?*

### Short answer

There are **three** distinct sidebar headers in the codebase; the one the Reader renders is `.sidebar-header` with **`margin: 0 12px 44px;`** and **`padding: 0 10px;`**. Layout offsets are computed with `calc()` over a small set of `:root` custom properties — chiefly **`--masterbar-height`** (`46px`, narrowing to `32px` at `min-width: 782px`), **`--sidebar-width-max: 272px`**, and **`--sidebar-width-min: 228px`**. Responsiveness comes from two coexisting breakpoint systems — a **deprecated** SCSS set and the JS **`@automattic/viewport`** package — with the sidebar's behavior actually flipping at **`<660px`** (narrow), **`>=782px`** (desktop; masterbar height drops to 32px), and **`>800px`** (collapsed sidebar allowed).

### Sidebar header spacing (exact values)

These are *literal* SCSS declarations, so the values are exact (not computed). Be careful which sidebar each belongs to:

**1) Reader sidebar header** — the one the question most likely means. It is rendered as `<li className="sidebar-header">` containing an `<h3>Reader</h3>` `[client/reader/sidebar/index.jsx:L168-L170]`, and styled under the `.is-section-reader` scope:

```scss
// client/reader/sidebar/style.scss:L112-L117
.is-section-reader {
	.sidebar-header {
		display: flex;
		justify-content: space-between;
		margin: 0 12px 44px;   // L116  → top:0  right/left:12px  bottom:44px
		padding: 0 10px;       // L117  → top/bottom:0  right/left:10px
```

→ **`margin: 0 12px 44px;`** `[client/reader/sidebar/style.scss:L116]` and **`padding: 0 10px;`** `[client/reader/sidebar/style.scss:L117]`.

**2) Modern global sidebar header** — `.sidebar__header`, hidden when the masterbar is visible:

```scss
// client/layout/global-sidebar/style.scss:L70-L75
.sidebar__header {
	align-items: center;
	// Hide the header when the masterbar is visible.
	display: none;
	gap: 8px;
	padding: 30px 24px 29px;   // L75
}
```

→ **`padding: 30px 24px 29px;`** `[client/layout/global-sidebar/style.scss:L75]` (note `display: none` `[client/layout/global-sidebar/style.scss:L73]`).

**3) Classic sidebar heading** — `.sidebar__heading`, used for static headings and expandable menus:

```scss
// client/layout/sidebar/style.scss:L66-L71
.sidebar__heading {
	color: var(--color-sidebar-text-alternative);
	font-size: $font-body;
	font-weight: 600;
	padding: 16px 8px 6px 16px;   // L70
	margin: 0;                    // L71
}
```

→ **`padding: 16px 8px 6px 16px;`** `[client/layout/sidebar/style.scss:L70]` and **`margin: 0;`** `[client/layout/sidebar/style.scss:L71]`.

| Sidebar | Selector | Margin | Padding | Source |
| --- | --- | --- | --- | --- |
| Reader | `.is-section-reader .sidebar-header` | `0 12px 44px` | `0 10px` | `[client/reader/sidebar/style.scss:L116-L117]` |
| Modern global | `.sidebar__header` | — | `30px 24px 29px` | `[client/layout/global-sidebar/style.scss:L75]` |
| Classic | `.sidebar__heading` | `0` | `16px 8px 6px 16px` | `[client/layout/sidebar/style.scss:L70-L71]` |

### CSS custom properties that drive layout `calc()`s

The `:root` block defines the variables the layout math depends on:

```scss
// client/assets/stylesheets/shared/_variables.scss:L5-L17
:root {
	// Masterbar
	--masterbar-height: 46px;            // L7
	--masterbar-checkout-height: 72px;   // L8

	@media only screen and (min-width: 782px) {
		--masterbar-height: 32px;        // L11
	}

	// Sidebar size limits
	--sidebar-width-max: 272px;          // L15
	--sidebar-width-min: 228px;          // L16
}
```

- **`--masterbar-height`** is `46px`, narrowing to **`32px`** at `min-width: 782px` `[client/assets/stylesheets/shared/_variables.scss:L7,L10-L12]`.
- **`--masterbar-checkout-height: 72px`** `[client/assets/stylesheets/shared/_variables.scss:L8]`.
- **`--sidebar-width-max: 272px`** `[client/assets/stylesheets/shared/_variables.scss:L15]` and **`--sidebar-width-min: 228px`** `[client/assets/stylesheets/shared/_variables.scss:L16]`.

These feed `calc()` expressions for content padding and sidebar width. For example, the Reader sidebar offsets its content with `padding: calc(var(--masterbar-height) + var(--content-padding-top)) calc(var(--sidebar-width-max)) var(--content-padding-bottom) 16px;` `[client/reader/sidebar/style.scss:L70]` and sizes a scroll area with `height: calc(100vh - var(--masterbar-height) - var(--content-padding-top) - var(--content-padding-bottom));` `[client/reader/sidebar/style.scss:L107]`. The main layout likewise offsets content by the masterbar/sidebar variables, e.g. `padding: calc(var(--masterbar-height) + 1px) 0 0 calc(var(--sidebar-width-max) + 1px);` `[client/layout/style.scss:L114]`.

**Contextual overrides** (worth knowing so the math makes sense):

- `--masterbar-height` is set to `0px` when there is no masterbar — in the main layout `[client/layout/style.scss:L410]` and in the masterbar stylesheet `[client/layout/masterbar/style.scss:L23]` (which also zeroes `--masterbar-checkout-height` `[client/layout/masterbar/style.scss:L21]`).
- In My Sites, the sidebar widths are re‑pinned: base `--sidebar-width-max/min: 272px` `[client/my-sites/sidebar/style.scss:L12-L13]`; `295px` when the global sidebar is visible `[client/my-sites/sidebar/style.scss:L16-L17]`; `69px` when collapsed `[client/my-sites/sidebar/style.scss:L60-L61]`; with `--content-padding-top/bottom: 16px` `[client/my-sites/sidebar/style.scss:L50-L51]`.

### Breakpoints — two coexisting systems (one deprecated)

**System A — deprecated SCSS mixin.** The legacy breakpoint set is:

```scss
// client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10
$breakpoints: 480px, 660px, 800px, 960px, 1040px, 1280px, 1400px;
```

→ `480 / 660 / 800 / 960 / 1040 / 1280 / 1400 px` `[client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10]`. The file header explicitly marks this **deprecated** in favor of Gutenberg breakpoints `[client/assets/stylesheets/shared/mixins/_breakpoints.scss:L4]` — prefer System B for new code.

**System B — JS `@automattic/viewport`.** The package exports named breakpoints and a `mediaQueryOptions` map:

```ts
// packages/viewport/src/index.ts
const SERVER_WIDTH = 769;                       // L41 (SSR fallback width)
export const MOBILE_BREAKPOINT = '<480px';      // L43
export const DESKTOP_BREAKPOINT = '>960px';     // L44
export const WIDE_BREAKPOINT = '>1280px';       // L45
// …
const mediaQueryOptions = {                      // L96
	'<480px': { max: 480 },   // L97
	'<660px': { max: 660 },   // L98
	'<800px': { max: 800 },   // L100
	// …
	'>=782px': { min: 781 },  // L108
	'>782px': { min: 782 },   // L109
	'>800px': { min: 800 },   // L110
	// …
};
```

→ `MOBILE_BREAKPOINT = '<480px'` `[packages/viewport/src/index.ts:L43]`, `DESKTOP_BREAKPOINT = '>960px'` `[packages/viewport/src/index.ts:L44]`, `WIDE_BREAKPOINT = '>1280px'` `[packages/viewport/src/index.ts:L45]`, the `mediaQueryOptions` map at `[packages/viewport/src/index.ts:L96-L119]` (e.g. `'<660px': { max: 660 }` `[packages/viewport/src/index.ts:L98]`, `'<800px': { max: 800 }` `[packages/viewport/src/index.ts:L100]`, `'>=782px': { min: 781 }` `[packages/viewport/src/index.ts:L108]`, `'>800px': { min: 800 }` `[packages/viewport/src/index.ts:L110]`), and the SSR fallback `SERVER_WIDTH = 769` `[packages/viewport/src/index.ts:L41]`.

**Where the sidebar's behavior actually changes.** The layout component reads System B at runtime to toggle state and body classes:

```js
// client/layout/index.jsx
const isNarrow = useBreakpoint( '<660px' );                  // L76
// …
isDesktop: isWithinBreakpoint( '>=782px' ),                  // L146
// …
if ( this.props.sidebarIsCollapsed && isWithinBreakpoint( '>800px' ) ) { // L221
	bodyClass.push( 'is-sidebar-collapsed' );                // L222
}
```

- `useBreakpoint( '<660px' )` drives the narrow state (`isNarrow`) `[client/layout/index.jsx:L76]`.
- `isWithinBreakpoint( '>=782px' )` drives the desktop state (`isDesktop`) `[client/layout/index.jsx:L146]` (and is also subscribed for live updates `[client/layout/index.jsx:L151]`).
- `isWithinBreakpoint( '>800px' )` gates the collapsed‑sidebar body class `[client/layout/index.jsx:L221-L222]`.
- Independently, `--masterbar-height` narrows from `46px` to `32px` at `min-width: 782px` via the `:root` media query `[client/assets/stylesheets/shared/_variables.scss:L10-L12]`.

| Breakpoint | Effect on the sidebar/layout | Source |
| --- | --- | --- |
| `< 660px` | `isNarrow` → narrow sidebar behavior | `[client/layout/index.jsx:L76]` |
| `min-width: 782px` | `--masterbar-height` drops `46px → 32px` | `[client/assets/stylesheets/shared/_variables.scss:L10-L12]` |
| `>= 782px` | `isDesktop` → desktop layout state | `[client/layout/index.jsx:L146]` |
| `> 800px` | allows the collapsed‑sidebar body class | `[client/layout/index.jsx:L221-L222]` |

### Rationale

The header *spacing* values are literal SCSS declarations, so they are exact and not computed — `margin: 0 12px 44px; padding: 0 10px;` for the Reader header `[client/reader/sidebar/style.scss:L116-L117]`. The *layout* (content offset and sidebar column width), by contrast, is computed via `calc()` over the `:root` custom properties `[client/assets/stylesheets/shared/_variables.scss:L5-L17]`. Responsiveness then arrives from two independent places: (a) the `:root` `@media (min-width: 782px)` rule that shrinks `--masterbar-height` `[client/assets/stylesheets/shared/_variables.scss:L10-L12]`, and (b) JS breakpoint checks in `client/layout/index.jsx` that toggle classes/state at `<660px`, `>=782px`, and `>800px` `[client/layout/index.jsx:L76,L146,L221]`. Because two breakpoint systems coexist and the SCSS one is deprecated, reach for `@automattic/viewport` when adding new responsive logic.


---

## 6. Gotchas for New Engineers

A short list of things that trip people up on day one, each cross‑referenced to the relevant answer above:

1. **Use `calypso.localhost:3000`, not bare `localhost`.** The local app calls the *remote* WordPress.com REST API, which only permits certain origins, so you must add `127.0.0.1 calypso.localhost` to your hosts file and open `http://calypso.localhost:3000` `[docs/install.md:L9,L38]`. (§1, §2)
2. **Auth behaves differently locally than in production.** `wpcom-user-bootstrap` is **false** in development/test, so login is detected by a client‑side `GET /me` `[config/development.json:L209]`, `[client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L7]`; it is **true** in production, where the server bootstraps the user and injects `window.currentUser` `[config/production.json:L177]`, `[client/lib/user/shared-utils/initialize-current-user.js:L28-L31]`. Don't expect SSR‑injected identity locally. (§4)
3. **The dev server is single‑port.** HMR is mounted in‑process on the same Express app `[client/server/bundler/index.js:L101-L102]`, and data goes to the remote API `[packages/wpcom-xhr-request/src/index.js:L27]` — there is no separate hot‑reload or local API port. (§2)
4. **"Ready" is two signals.** The boot log / `sendBootStatus('ready')` only means the process is listening `[client/server/index.js:L83-L86]`; wait for the Webpack **"compiled"** message (piped through `bunyan` `[package.json:L113]`) before the app actually renders. (§2)
5. **Two breakpoint systems exist; the SCSS one is deprecated.** Prefer the JS `@automattic/viewport` breakpoints `[packages/viewport/src/index.ts:L43-L45]` over the deprecated SCSS `$breakpoints` set `[client/assets/stylesheets/shared/mixins/_breakpoints.scss:L4,L10]` for new responsive code. (§5)
6. **Node 22.x is mandatory.** `yarn start` runs `check-node-version --package` first `[package.json:L110]`, and `engines.node` is `^v22.9.0` `[package.json:L57]`; Node 20 will abort before building. (§1)

---

### Appendix — Source map of every cited file

| Question | Files consulted (read‑only) |
| --- | --- |
| §1 / Q1 | `config/_shared.json`, `config/development.json`, `client/server/config/parser.js`, `client/server/index.js`, `client/server/bundler/index.js`, `package.json`, `bin/welcome.js`, `docs/install.md`, `README.md`, `docs/yarn-start.md`, `packages/wpcom-xhr-request/src/index.js` |
| §3 / Q2 | `client/reader/index.ts`, `client/reader/controller.js`, `client/reader/stream/index.jsx`, `client/state/reader/streams/actions.js`, `client/state/reader/action-types.ts`, `client/state/data-layer/wpcom/read/streams/index.js`, `client/state/reader/posts/actions.js`, `packages/wpcom-xhr-request/src/index.js` |
| §4 / Q3 | `client/state/current-user/selectors.js`, `client/boot/common.js`, `client/lib/user/shared-utils/initialize-current-user.js`, `client/lib/user/shared-utils/raw-current-user-fetch.js`, `config/{development,test,production,stage,horizon,wpcalypso}.json`, `client/server/user-bootstrap/index.js`, `packages/oauth-token/src/index.js`, `client/lib/user/store.js`, `client/lib/browser-storage/index.ts`, `client/state/initial-state.js` |
| §5 / Q4 | `client/assets/stylesheets/shared/_variables.scss`, `client/reader/sidebar/style.scss`, `client/reader/sidebar/index.jsx`, `client/layout/global-sidebar/style.scss`, `client/layout/sidebar/style.scss`, `client/layout/style.scss`, `client/layout/masterbar/style.scss`, `client/my-sites/sidebar/style.scss`, `client/assets/stylesheets/shared/mixins/_breakpoints.scss`, `packages/viewport/src/index.ts`, `client/layout/index.jsx` |

*All files above were read for evidence only; none were modified. The sole artifact produced by this task is this document.*

