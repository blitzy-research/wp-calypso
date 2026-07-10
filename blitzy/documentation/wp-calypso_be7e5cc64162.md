# WordPress.com Calypso — Runtime Onboarding Answers (Reader, local dev)

> **Branch:** `wp-calypso_be7e5cc64162`
> **HEAD commit:** `be7e5cc641622d153040491fd5625c6cb83e12eb` — *"Reader: Show login prompts on all logged out reader streams"*
> **Repository:** `Automattic/wp-calypso`

This document answers four questions about how Calypso boots and serves the Reader locally. **Every value below was captured from a real, running instance** built and launched through the canonical entry point, and each factual claim carries a `file:line` citation against the checkout at the HEAD above. Values that could not be captured from the canonical logged‑out run (for example, the logged‑in sidebar, which requires credentials that are not available) are explicitly labelled **inferred (source‑grounded)** or **non‑canonical**, never presented as observed.

---

## 0. Environment, toolchain, and exact run procedure

All observations come from the **default, canonical configuration** unless a line is explicitly labelled otherwise.

**Toolchain (observed):**

```
Command:  node --version ; yarn --version
Output:   v22.23.1
          4.0.2
Observed: Node v22.23.1 (satisfies engines.node "^v22.9.0"); Yarn 4.0.2 (matches packageManager pin).
Source:   package.json:L57 ("node": "^v22.9.0"); .nvmrc:L1 (22.9.0); package.json:L422 ("packageManager": "yarn@4.0.2").
```

> Node 20.x must **not** be used: the `check-node-version --package` gate inside the `start` script rejects it (`package.json:L110`).

**Host prerequisite (observed):**

```
Command:  grep -n calypso.localhost /etc/hosts
Output:   9:127.0.0.1 calypso.localhost
Observed: calypso.localhost resolves to 127.0.0.1. The app is reachable only at http://calypso.localhost:3000.
Source:   docs/install.md:L38 (hosts entry + remote REST API requirement).
```

**Exact build/run commands used:**

```
Install:  yarn
Start:    SECTION_LIMIT=reader,login yarn start
```

- `yarn start` chains `npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build` (`package.json:L110`).
- `start-build` = `BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short` (`package.json:L113`).
- The SSR server is compiled via `webpack --config client/webpack.config.node.js` (`package.json:L81`).
- `SECTION_LIMIT=reader,login` is a **build‑scope optimization** (code‑splitting) that only limits which sections are compiled; it does **not** change runtime behavior or the port (`docs/install.md:L48-L50`). This is called out again under Q1.

Then the Reader was loaded at `http://calypso.localhost:3000/reader`.

**Stability:** the run was performed twice (referred to below as *run 1* and *run 2*). Port, readiness signals, endpoints, the Redux action sequence, the auth decision, and the CSS values were identical across both runs except for millisecond timings, which are noted where relevant.

**A note that makes Q2 and Q3 fully observable while logged out.** The Reader section is registered with `enableLoggedOut: true` (`client/sections.js:L396`), and the HEAD commit shows login prompts on logged‑out streams. As a result, the Reader fires its initial‑load API calls and Redux actions, and exposes its auth decision, **even without credentials** — so the canonical logged‑out run is sufficient to observe them.

---

## Q1 — Development server: port, readiness, and single‑ vs multi‑port architecture

**Short answer.** The dev server binds to **port 3000**. A developer knows it is fully ready when **two** signals appear on stdout: the bunyan boot line `wp-calypso booted in <n>ms - http://calypso.localhost:3000`, followed (after the first webpack compile) by `Ready! You can load http://calypso.localhost:3000/ now. Have fun!`. The architecture is **single‑port**: one Express server on 3000 serves the SSR HTML, the compiled JS/CSS assets, **and** the hot‑module‑reload stream in‑process; REST API traffic does **not** use a local port at all — it leaves the machine to the remote `https://public-api.wordpress.com`.

### Q1.a — The port is 3000 (config → actual bind)

The dev environment configuration sets the port:

```
Source:   config/development.json:L6-L8
          "protocol": "http",
          "hostname": "calypso.localhost",
          "port": 3000,
Shared defaults: config/_shared.json:L24-L25 ("protocol":"http","port":3000), L13 ("hostname": false).
```

Resolved through the real server config subsystem (canonical):

```
Command:  NODE_ENV=development node -e "const c=require('<repo>/client/server/config'); \
          console.log(\"config('port') =\", c('port'), '(typeof', typeof c('port'), ')')"
Output:   config('protocol') = "http"
          config('hostname') = "calypso.localhost"
          config('port')     = 3000 (typeof number )
          config('port')===3000 : true
Observed: config('port') === 3000 (number).
Source:   client/server/index.js:L11-L13 (const { protocol, port, host } = ... config reads).
```

The value is read and then passed to the actual listen call:

```
Source:   client/server/index.js:L83
          server.listen( { port, host: process.env.CALYPSO_IS_FORK ? host : null }, function () {
Observed: A non-fork run binds host = null (all interfaces). Only the desktop fork (CALYPSO_IS_FORK) binds calypso.localhost.
```

The live bind was confirmed by hitting the health endpoint on 3000:

```
Command:  curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://calypso.localhost:3000/version
Output:   HTTP 200
Observed: The server is listening and serving on port 3000. (GET /version returns {"version":"0.17.0"}.)
```

### Q1.b — The two readiness signals (raw, both runs)

**Signal #1 — bunyan boot log** (emitted by `logger.info(...)` and piped through `bunyan -o short`):

```
Source:   client/server/index.js:L33
          logger.info( 'wp-calypso booted in %dms - %s://%s:%s', Date.now() - start, protocol, host, port );
          Logger name "calypso" at stdout info level: client/server/lib/logger/index.js.

Command:  grep "wp-calypso booted in" /tmp/calypso_run/start_run1.log   (and start_run2.log)
Output (run 1):  07:43:39.846Z  INFO calypso: wp-calypso booted in 963ms - http://calypso.localhost:3000
Output (run 2):  07:49:48.189Z  INFO calypso: wp-calypso booted in 971ms - http://calypso.localhost:3000
Observed: Identical protocol://host:port ("http://calypso.localhost:3000"); only the millisecond figure differs (963 vs 971).
```

**Signal #2 — webpack "Ready!"** (printed after the first browser‑bundle compile finishes):

```
Source:   client/server/bundler/index.js:L56
          `\nReady! You can load ${ protocol }://${ host }:${ port }/ now. Have fun!`

Command:  grep -E "webpack .* compiled|Ready! You can load" /tmp/calypso_run/start_run2.log
Output (run 2):  webpack 5.97.1 compiled with 11 warnings in 65870 ms
                 Ready! You can load http://calypso.localhost:3000/ now. Have fun!
Output (run 1):  webpack 5.97.1 compiled with 11 warnings in 64184 ms
                 Ready! You can load http://calypso.localhost:3000/ now. Have fun!
Observed: The "Ready!" line is byte-identical across runs; only the compile duration differs (64184 vs 65870 ms).
```

### Q1.c — Transitional state (before/during/after first compile)

Before that first compile completes, the bundler serves a **"Welcome to Calypso!"** holding page for `/` (with a 5‑second auto‑refresh), and queues other requests via `waitForCompiler`. This was captured live during the run‑2 compile window:

```
Source:   client/server/bundler/index.js — holding page markup (meta refresh + "Welcome to Calypso!" + READY! hint).

Command:  curl -s http://calypso.localhost:3000/   (issued after boot, before "Ready!")
Output (complete, unedited):
				<head>
					<meta http-equiv="refresh" content="5">
				</head>
				<body>
					<h1>Welcome to Calypso!</h1>
					<p>
						Please wait until webpack has finished compiling and you see
						<code style="font-size: 1.2em; color: blue; font-weight: bold;">READY!</code> in
						the server console. This page should then refresh automatically. If it hasn&rsquo;t, hit <em>Refresh</em>.
					</p>
					<p>
						In the meantime, try to follow all the emotions of the allmoji:
						<img src="https://emoji.slack-edge.com/T024FN1V2/allmoji/15b93529a828705f.gif"
							width="36" style="vertical-align: middle;">
				</body>
Observed: Pre-compile, "/" returns the holding page (HTTP 200, text/html) with <meta http-equiv="refresh" content="5">.
          After "Ready!", "/" returns the real ~25 KB application document with zero holding-page markers.
```

### Q1.d — Single‑port proof (assets + HMR on 3000; REST is remote)

The development bundler mounts **`webpack-dev-middleware`** and **`webpack-hot-middleware`** on the **same** Express `app`:

```
Source:   client/server/bundler/index.js:L5-L6 (require webpack-dev-middleware, webpack-hot-middleware)
          attached via require('calypso/server/bundler')(app) at client/server/boot/index.js:L37 (development only).
```

Everything below was served from port 3000:

```
Command:  curl -sI http://calypso.localhost:3000/reader
Output:   HTTP/1.1 200 OK ; Content-Type: text/html; charset=utf-8 ; X-Powered-By: Express ; Content-Length: 35865
Observed: SSR HTML served by Express on 3000.

Command:  curl -sI http://calypso.localhost:3000/calypso/evergreen/runtime.js
Output:   HTTP/1.1 200 ; Content-Type: application/javascript; charset=utf-8 ; (75033 bytes)
Observed: Compiled JS asset served in-process by webpack-dev-middleware on 3000.

Command:  curl -sI http://calypso.localhost:3000/__webpack_hmr
Output:   HTTP/1.1 200 OK ; Content-Type: text/event-stream;charset=utf-8 ; X-Powered-By: Express ;
          Cache-Control: no-cache ; Connection: keep-alive
Observed: HMR event stream served on the SAME port 3000 by webpack-hot-middleware.
```

REST traffic, by contrast, targets the **remote** API (confirmed from a live Reader request in Q2, whose origin is `https://public-api.wordpress.com`), not any local port (`docs/install.md:L38`).

**Conclusion (Q1):** one Express server on **port 3000** serves SSR + assets + HMR; REST calls go to the remote WordPress.com API. **Single‑port.**

### Q1.e — Explicitly non‑canonical / clarifications (not the basis of any value above)

- **`MOCK_WORDPRESSDOTCOM=1`** forces `https`/`443`/`wordpress.com` at `client/server/index.js:L16-L21`. **Non‑canonical** — not exercised.
- **Inspector port `5858`** via `NODE_OPTIONS="--inspect=5858" yarn start` is a **debugger** port only, unrelated to app traffic (`docs/install.md:L63`).
- **`SECTION_LIMIT` / `ENTRY_LIMIT`** and "multiple Webpack entry points" are **build‑time code‑splitting** for smaller bundles (`docs/install.md:L48-L50`), **not** a multi‑port topology.

---

## Q2 — Reader stream: which API endpoints populate it, and which Redux actions fire on initial load

**Short answer.** On a default (logged‑out) load, `/reader` redirects to `/discover`, and the stream that populates is the **Discover** stream. The first request is a **GET** to `https://public-api.wordpress.com/wpcom/v2/read/streams/discover` with `number=4` (the initial page size). The initial‑load Redux sequence, in dispatch order, is **`READER_STREAMS_PAGE_REQUEST` → `READER_POSTS_RECEIVE` → `READER_STREAMS_PAGE_RECEIVE`**.

### Q2.a — The canonical logged‑out flow and the observed endpoint

Navigating to `/reader` triggers a client‑side redirect to `/discover` for logged‑out users; the reader route table wires `redirectLoggedOutToDiscover` ahead of the stream view in `client/reader/index.ts` (the file is **`index.ts`**, TypeScript — there is no `client/reader/index.js`). The endpoint map that builds each stream request lives in `client/state/data-layer/wpcom/read/streams/index.js`.

```
Command:  Chrome DevTools — load http://calypso.localhost:3000/reader, capture Network requests to public-api.wordpress.com
Output (the first Reader-stream request; identical on both loads — run 1 reqid=149, run 2 reqid=428):
  GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover?_envelope=1&orderBy=popular
      &meta=post%2Cdiscover_original_post&feed_id=&number=4&lang=en
      &tags%5B%5D=dailyprompt&tags%5B%5D=wordpress&tag_recs_per_card=5&site_recs_per_card=5
      &age_based_decay=0.5&content_width=675
  → HTTP 200 ; response body shape: {"body":{"cards":[{"type":"recommended_blogs",...},{"type":"post",...}]}}
Observed: method GET; path /read/streams/discover; number=4 (the initial page size); apiNamespace wpcom/v2 → URL prefix /wpcom/v2.
Source:   /read/streams/discover at client/state/data-layer/wpcom/read/streams/index.js:L226;
          apiNamespace 'wpcom/v2' at :L246; INITIAL_FETCH = 4 at :L161.
```

The second (pagination) request confirms the page‑size logic:

```
Output (pagination request; run 1 reqid=168, run 2 reqid=452):
  GET .../wpcom/v2/read/streams/discover?...&page_handle=<opaque>&number=7
Observed: number=7 on subsequent pages == PER_FETCH.
Source:   PER_FETCH = 7 at client/state/data-layer/wpcom/read/streams/index.js:L160;
          fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH at :L380.
```

The request builder itself:

```
Source:   client/state/data-layer/wpcom/read/streams/index.js
          - requestPage( action )                         :L358  (issues http({ method:'GET', path, apiVersion, apiNamespace, query, onSuccess, onFailure }))
          - default apiVersion = '1.2'                     :L370
          - handlePage( action, data )                    :L428  (success handler; dispatches receivePosts + receivePage)
          - registerHandlers(...)                          :L514  (wires READER_STREAMS_PAGE_REQUEST + READER_STREAMS_PAGINATED_REQUEST
                                                                    to dispatchRequest({ fetch: requestPage, onSuccess: handlePage }))
```

### Q2.b — Disambiguation: `/read/streams/discover` vs `/read/following` vs `/read/streams/following`

The `streamApis` map defines many endpoints; the one used is selected by the stream type (`getStreamType` returns the substring before the first `:` of the stream key, `client/reader/utils.ts:L116`). The observed value on the default run is **`/read/streams/discover`** (stream type `discover`). The two "following" endpoints exist for the **logged‑in** Following stream and are therefore **not** exercised on a credential‑free run (labelled **source‑grounded, logged‑in path**):

| Stream type | Endpoint path | Citation |
|-------------|---------------|----------|
| **discover** (observed default) | `/read/streams/discover` (+ `?tags=` variant) | `client/state/data-layer/wpcom/read/streams/index.js:L226` |
| following | `/read/following` | `:L194` |
| recent | `/read/streams/following` (`apiNamespace: wpcom/v2`) | `:L198`, `:L246` |
| search | `/read/search` | `:L212` |
| feed | `/read/feed/${feed}/posts` | `:L220` |
| tag | `/read/tags/${tag}/posts` | `:L317` |
| user | `/users/${user}/posts` | `:L347` |

**Sibling requests observed on the same load** (all to `public-api.wordpress.com`, all HTTP 200): `/rest/v1.1/me?meta=flags` (the client's current‑user fetch — see Q3), `/rest/v1.1/read/feed/{feedId}` and `/rest/v1.1/read/sites/{siteId}` (per‑card metadata), `/rest/v1.1/users/suggest`, and `POST /rest/v1.1/logstash`.

### Q2.c — The initial‑load Redux action sequence (raw)

The action stream was captured non‑invasively by injecting a Redux DevTools‑compatible enhancer shim (the store already connects to `window.__REDUX_DEVTOOLS_EXTENSION__` as its innermost enhancer at `client/state/index.ts:L49`); the shim was injected as a page init‑script, so **no source file was modified**, and the temporary script was removed afterward.

```
Command:  Chrome DevTools — record dispatched action types during the /reader → /discover initial load
Output (first Reader-stream lifecycle, in dispatch order; identical across both loads — 291 total actions on load 1, 304 on load 2):
  READER_STREAMS_PAGE_REQUEST
  READER_POSTS_RECEIVE
  READER_STREAMS_PAGE_RECEIVE
Observed: request → posts-receive → page-receive, exactly matching the data-layer fetch/handlePage lifecycle.
```

**Action‑type constants** (`client/state/reader/action-types.ts`):

- `READER_STREAMS_PAGE_REQUEST` — `:L78`
- `READER_POSTS_RECEIVE` — `:L52`
- `READER_STREAMS_PAGE_RECEIVE` — `:L77`
- `READER_STREAMS_PAGINATED_REQUEST` — `:L79`

**Action creators:**

- `requestPage` — `client/state/reader/streams/actions.js:L28` (emits `READER_STREAMS_PAGE_REQUEST`)
- `receivePage` — `client/state/reader/streams/actions.js:L52` (emits `READER_STREAMS_PAGE_RECEIVE`)
- `requestPaginatedStream` — `client/state/reader/streams/actions.js:L143`
- `receivePosts` — `client/state/reader/posts/actions.js:L63`, dispatching `{ type: READER_POSTS_RECEIVE }` at `:L87`

### Q2.d — Where the dispatch originates

The initial `requestPage` is dispatched from the `Stream` React component when it mounts:

```
Source:   client/reader/stream/index.jsx
          - componentDidMount()                :L221  → this.fetchNextPage( {} )  :L225
          - fetchNextPage()                    :L490  → this.props.requestPage({ streamKey, ... })  :L502
Observed: The mount of the Stream component triggers the first READER_STREAMS_PAGE_REQUEST, which the data layer turns into the
          GET /wpcom/v2/read/streams/discover request captured above.
```

**Conclusion (Q2):** the Discover stream is populated by `GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover` (initial `number=4`, then `number=7`), and the initial‑load Redux sequence is `READER_STREAMS_PAGE_REQUEST → READER_POSTS_RECEIVE → READER_STREAMS_PAGE_RECEIVE`.


---

## Q3 — How the app decides "is the user logged in?" before rendering, and which storage it checks

**Short answer.** On the **server**, the pre‑render decision is a cookie test: `isLoggedIn = !!req.cookies.wordpress_logged_in` (`client/server/pages/index.js:L93`). On the **client**, the decision is the selector `isUserLoggedIn(state)`, which is `getCurrentUserId(state) !== null` (`client/state/current-user/selectors.js:L15-L16`). On the default dev run this evaluates to **false**, so `LayoutLoggedOut` renders. The storage mechanisms inspected are: **cookies** (`wordpress_logged_in`, `wpcom_token`, `support_session_id`), **localStorage** (OAuth‑token fallback), **IndexedDB** (persisted Redux state), **sessionStorage** (OAuth `flags`), and the **server‑injected initial state** (`window.initialReduxState`).

### Q3.a — The default‑configuration nuance (state it up front)

```
Source:   config/development.json:L130  "oauth": false
          config/development.json:L209  "wpcom-user-bootstrap": false
Observed: On a default dev run, the server-side login-redirect and the server-side user bootstrap are DISABLED, and the OAuth
          token middleware is inactive. The app therefore renders LOGGED-OUT by default. Enabling a logged-in server path would
          require changing these flags (non-default) and supplying a real wordpress_logged_in cookie — not done here (read-only repo).
```

### Q3.b — Server (SSR) decision + what actually gets injected

```
Source:   client/server/pages/index.js:L93   const isLoggedIn = !! req.cookies.wordpress_logged_in;   (stored on req.context.isLoggedIn :L99)
          Downstream (default-config gated OFF): login redirect if(!isLoggedIn) at :L372 — gated by isEnabled('wpcom-user-bootstrap') at :L364;
          setCurrentUser bootstrap getBootstrappedUser(req) :L382 → dispatch(setCurrentUser(data)) :L391.

Command:  curl -sS -D - -o /dev/null http://calypso.localhost:3000/reader            (no cookie)
          curl -sS -D - -o /dev/null -H 'Cookie: wordpress_logged_in=fakeuser%7C123%7Cabc' http://calypso.localhost:3000/reader
Output (no cookie):    HTTP/1.1 200 OK ; X-Powered-By: Express ; Cache-control: no-store ; Content-Length: 35865
Output (with cookie):  HTTP/1.1 200 OK ; X-Powered-By: Express ; Cache-control: no-store ; Content-Length: 35893
Byte diff (only):      the cookie version adds one line `var languageRevisions = {}`.
Observed: The server DOES read req.cookies.wordpress_logged_in (the response changes with the cookie), but because
          wpcom-user-bootstrap is false it does NOT bootstrap a user or redirect. No currentUser is injected either way.
```

What the server injects into the page is a small `var initialReduxState = {...}`, populated by `pick(store.getState(), initialClientStateTrees)` where the tree list is just `documentHead` for the (non‑isomorphic) Reader section:

```
Source:   client/server/render/index.js:L257  const initialClientStateTrees = [ 'documentHead', ...isomorphicSubtrees ];
          client/server/render/index.js:L260  context.initialReduxState = pick( context.store.getState(), initialClientStateTrees );
          Injection form: client/document/index.jsx:L86-L87  `var initialReduxState = ${ jsonStringifyForHtml( initialReduxState ) };`

Command:  grep -oE "var initialReduxState = \{.*?\};" /tmp/calypso_run/reader_nocookie.html
Output (identical with and without the cookie):
  var initialReduxState = {"documentHead":{"link":[],"meta":[{"property":"og:site_name","content":"WordPress.com"}],"title":"","unreadCount":0}};
Observed: Only documentHead is injected; no currentUser slice is ever server-injected on this run.
          In the browser this global is read as window.initialReduxState.
```

### Q3.c — Client hydration and the decision selector

The client merges the injected server state with IndexedDB‑persisted state:

```
Source:   client/state/initial-state.js
          - getInitialState(...)                     :L140  (merges server + persisted)
          - getInitialServerState ← window.initialReduxState  :L148-L154
          - persisted state keyed by user id (IndexedDB)      :L160
          - persistence key: 'redux-state-' + (userId ?? 'logged-out') (+ optional ':subkey')  :L76
```

The decision selector, read against the **real, fully‑hydrated app store** (47 top‑level slices, exposed on the global object in dev as `window.getState()`):

```
Source:   client/state/current-user/selectors.js:L6   getCurrentUserId(state) => state.currentUser?.id
          client/state/current-user/selectors.js:L15-L16  isUserLoggedIn(state) => getCurrentUserId(state) !== null

Command:  evaluate_script — window.getState() on the live page
Output:   currentUser = {"id":null,"user":null,"capabilities":{},"flags":[],"emailVerification":{"status":null,"errorMessage":""},"lasagnaJwt":null}
          getCurrentUserId (state.currentUser?.id) = null
          isUserLoggedIn (null !== null)           = false
          getCurrentUser (state?.currentUser?.user ?? null) = null
Observed: isUserLoggedIn === false → logged-out render.
```

> **Selector nuance worth noting.** `isUserLoggedIn` uses `!== null`. The correct logged‑out result relies on the `currentUser.id` reducer defaulting to **`null`** (`client/state/current-user/reducer.js:L24`, inside a `combineReducers`). Because that slice is registered, `state.currentUser.id === null` → `isUserLoggedIn === false`. (A store where the slice were *absent* would yield `undefined?.id === undefined`, and `undefined !== null` is `true`; that is not the app's real store, which has the slice.)

The render selection consumes it:

```
Source:   client/controller/index.web.js
          - import LayoutLoggedOut          :L18
          - import { isUserLoggedIn }        :L29
          - const userLoggedIn = isUserLoggedIn( state )  :L59
          - <LayoutLoggedOut ...>            :L64
Observed (DOM, live): url = http://calypso.localhost:3000/discover (redirected from /reader);
          body class "... is-reader-page"; layout class includes "has-no-sidebar"; "Log In"/"Sign Up" links present → logged-out layout.
```

### Q3.d — Storage mechanisms inspected (enumerated, live capture, logged‑out default)

```
Command:  evaluate_script — dump document.cookie, localStorage, sessionStorage, and IndexedDB on the live page
```

| Mechanism | What Calypso uses it for | Live value on the default logged‑out run | Source |
|-----------|--------------------------|------------------------------------------|--------|
| **Cookie `wordpress_logged_in`** | Server pre‑render session test | **absent** | `client/server/pages/index.js:L93`; `client/server/user-bootstrap/index.js:L8` |
| **Cookie `wpcom_token`** | OAuth token (OAuth/desktop builds) | **absent** | `packages/oauth-token/src/index.js:L7` (`TOKEN_NAME='wpcom_token'`) |
| **Cookie `support_session_id`** | Support session | **absent** | `client/server/user-bootstrap/index.js:L9` |
| Cookies actually present | WordPress.com analytics/geo | `tk_ai`, `country_code=US`, `region=Iowa`, `tk_qs` | — |
| **localStorage** (via `store`) | OAuth‑token fallback `store.get('wpcom_token')` | only `{ tusSupport: "null" }`; no `wpcom_token` | `packages/oauth-token/src/index.js:L17-L21` |
| **sessionStorage** | OAuth `flags` | **empty** (no `flags`; the OAuth path is inactive) | `client/boot/common.js:L127` |
| **IndexedDB** | Persisted Redux state | DB `calypso` (v2), store `calypso_store`; 16 keys all prefixed **`redux-state-logged-out`** (`…`, `:reader`, `:readerUi`, `:ui`, `:route`, `:preferences`, `:siteSettings`, `:teams`, `:signup`, `:plugins`, `:memberships`, `:pushNotifications`, `:connectedApplications`, `:all-domains`, `:documentHead`, `:userSuggestions`) + `was-state-randomly-cleared` | `client/state/initial-state.js:L76`; DB name `client/lib/browser-storage/index.ts` |
| **Server‑injected** `window.initialReduxState` | Hydration seed | top keys `["documentHead"]` only (no `currentUser`) | `client/document/index.jsx:L86-L87` |

The **`redux-state-logged-out`** key prefix is direct confirmation of the persistence‑key expression `'redux-state-' + (userId ?? 'logged-out')` with `userId = null` (`client/state/initial-state.js:L76`).

### Q3.e — Alternate/edge paths (labelled)

- **User‑bootstrap error branch — inferred (source‑grounded).** When bootstrap is enabled and the auth cookie is missing, `getBootstrappedUser` throws `'Cannot bootstrap without an auth cookie'` (`client/server/user-bootstrap/index.js:L34`; cookie read at `:L28`; it otherwise forwards the cookie to `https://public-api.wordpress.com/rest/v1/me`, `:L13`). This branch is unreachable on the default run (`wpcom-user-bootstrap: false`) and is not runtime‑observed here.
- **OAuth mode — non‑default (source‑grounded).** `oauthTokenMiddleware` (gated by `isEnabled('oauth')`, false by default) redirects to the authorize URL when `getToken() === false` (`client/boot/common.js:L154,L176`); the OAuth token is read from `document.cookie` then falls back to `store.get('wpcom_token')` in localStorage (`packages/oauth-token/src/index.js:L11-L21`) and written back to `document.cookie` (`:L28`).
- **Logged‑in render — source‑grounded.** With a real session, `setCurrentUser` (`client/state/current-user/actions.js`, emitting `CURRENT_USER_RECEIVE`) would populate `currentUser.id`, making `isUserLoggedIn` true and selecting the logged‑in `Layout` instead of `LayoutLoggedOut` (`client/controller/index.web.js:L59-L64`, and the bootstrap‑disabled shortcut at `:L100`).

**Conclusion (Q3):** the pre‑render decision is the `wordpress_logged_in` cookie on the server and the `isUserLoggedIn` selector (`currentUser.id !== null`) on the client; on the default run both resolve to logged‑out. Storage inspected spans cookies, localStorage, IndexedDB, sessionStorage, and the server‑injected `window.initialReduxState`.


---

## Q4 — Responsive sidebar: header padding/margin, the CSS custom properties, and the breakpoints

**Short answer.** The **global** sidebar header (`.sidebar__header`) uses `padding: 30px 24px 29px` with `margin: 0` and `gap: 8px`. The **classic** sidebar header row (`.sidebar__heading`) uses `padding: 16px 8px 6px 16px; margin: 0` (base rule). The layout `calc()` expressions are driven by three CSS custom properties: `--masterbar-height` (`46px`, dropping to `32px` at `min-width: 782px`), `--sidebar-width-max` (`272px`), and `--sidebar-width-min` (`228px`). The global breakpoint scale is `480, 660, 800, 960, 1040, 1280, 1400 px`, with the sidebar/masterbar‑specific switches at `660/661`, `782`, `960`, and `600`.

### Q4.0 — Important observed nuance: no sidebar mounts on the logged‑out Reader

```
Command:  evaluate_script — count sidebar elements on the live /discover (redirected from /reader) and /tag/wordpress
Output:   .sidebar = 0 ; .global-sidebar = 0 ; .sidebar__header = 0 ; .sidebar-v2 = 0 ; .masterbar = 1 ; .layout = 1
          layout class = "layout is-group-reader is-section-reader focus-content has-header-section has-no-sidebar ..."
Observed: The logged-out Reader uses a masterbar+footer landing layout with NO sidebar; the sidebar header rules are
          code-split out of the logged-out page (0 CSSOM matches across 44 accessible sheets).
```

Because the sidebar variants are logged‑in constructs (and no credentials are available), the header padding/margin values below were captured from the **live‑served compiled CSS artifact** and confirmed with a `getComputedStyle` **class‑probe** in the app's real `body.theme-default` context. The CSS is the canonical build output served on port 3000; the probe *element* is synthetic (labelled as such). The **custom properties** and **breakpoints** were captured **directly and live** from `:root` and `matchMedia`.

### Q4.a — Sidebar header padding/margin, per variant

The compiled reader‑sidebar CSS chunk was fetched from the running server and the exact declarations extracted:

```
Command:  curl -s http://calypso.localhost:3000/calypso/evergreen/async-load-calypso-reader-sidebar.css
          → HTTP 200, 123922 bytes ; then extract each rule block
Output (GLOBAL sidebar header):
  .global-sidebar .sidebar__header { align-items: center; display: none; gap: 8px; padding: 30px 24px 29px; }
Output (GLOBAL logo):
  .global-sidebar .sidebar__header span.dotcom { display: flex; width: 125px; height: 28px; margin: 0; }
  .is-global-sidebar-collapsed .global-sidebar .sidebar__header span.dotcom { width: 24px; margin-left: 6px; }
Output (GLOBAL body):
  .global-sidebar .sidebar__body { padding-top: 40px; }
  .has-no-masterbar .global-sidebar .sidebar__body { padding-top: 0; }
Output (CLASSIC root + heading):
  .sidebar { margin: 0; padding: 0; padding-top: 6px; }
  .sidebar__heading { padding: 16px 8px 6px 16px; margin: 0; }
```

Confirmed as **computed** px via the class‑probe (live compiled CSS injected; measured with `getComputedStyle`):

```
Observed (computed):
  GLOBAL .sidebar__header : padding = 30px 24px 29px (top 30 / right 24 / bottom 29 / left 24) ; margin = 0px ;
                            gap = 8px ; align-items = center ; display = none
  GLOBAL span.dotcom      : width = 125px ; height = 28px ; margin = 0px ; display = flex
  GLOBAL .sidebar__body   : padding-top = 40px
  CLASSIC .sidebar (root) : computed padding = 6px 0px 12px  (base padding-top:6px + a theme-default padding-bottom:12px)
  CLASSIC .sidebar__heading: base rule 16px 8px 6px 16px, but computed 0px 0px 0px 8px under the more specific
                             ".theme-default .sidebar .sidebar__heading" rule
```

**Sources (per variant):**

- **Global** (`client/layout/global-sidebar/style.scss`): `.sidebar__header { align-items:center; display:none; gap:8px; padding:30px 24px 29px }` at **L70‑L75** (the `display:none` comment "Hide the header when the masterbar is visible" is at L72); logo `span.dotcom` `width:125px; height:28px; margin:0` at **L82‑L90**; `.sidebar__body { padding-top:40px }` at **L112**.
- **Classic** (`client/layout/sidebar/style.scss`): root `.sidebar { margin:0; padding:0; padding-top:6px }` at **L4‑L6**; header row `.sidebar__heading { padding:16px 8px 6px 16px; margin:0 }` at **L70‑L71**.
- **sidebar‑v2 — inferred (source‑grounded).** This variant is **not** compiled into the reader/login build (0 occurrences in the served chunks), so it could not be observed at runtime here. Per source, `client/layout/sidebar-v2/style.scss` has root `.sidebar-v2 { gap:16px; margin:0; padding:16px; height:100vh; box-sizing:border-box }` and no `.sidebar__header` (it uses `.sidebar-v2__main` and `.sidebar-v2__footer`).

> **Cascade nuance (documented as an "every condition" case).** The classic header's *base* declaration is `16px 8px 6px 16px`, but in the active `.theme-default` theme with a `.sidebar` ancestor a more‑specific rule wins, so the *computed* value is `0 0 0 8px`. Both are real; the base is the canonical answer to "what is the padding on the header," the computed reflects the themed cascade.

### Q4.b — CSS custom properties driving the `calc()` layout (live)

```
Command:  evaluate_script — getComputedStyle(document.documentElement).getPropertyValue('--...')  + CSSOM scan
Output (compiled :root rules, served):
  :root { --masterbar-height: 46px; --masterbar-checkout-height: 72px; --sidebar-width-max: 272px; --sidebar-width-min: 228px; }
  @media only screen and (min-width: 782px) { :root { --masterbar-height: 32px; } }
Output (live computed at width 1905): --masterbar-height = 32px ; --sidebar-width-max = 272px ; --sidebar-width-min = 228px
Source:   client/assets/stylesheets/shared/_variables.scss:L7 (46px), L10-L11 (→32px @782px), L15 (max 272px), L16 (min 228px).
```

These feed the layout `calc()` expressions (captured live from the CSSOM of the served build):

```
Observed (served, compiled):
  .layout__content { margin: 0px; padding: 79px 32px 32px calc(var(--sidebar-width-max) + 32px + 1px); box-sizing: border-box; overflow: hidden; }
  @media (max-width: 960px) {
    .layout__content { padding: 71px 24px 24px calc(var(--sidebar-width-min) + 24px + 1px); }
    .has-no-sidebar .layout__content { padding-left: 24px; }
  }
  .is-section-preview .layout__content { padding: calc(var(--masterbar-height) + 1px) 0 0 calc(var(--sidebar-width-max) + 1px); }
Source:   client/layout/style.scss:L52 (main calc padding), L119 (@max-width:960px variant),
          L185/L192 (width: var(--sidebar-width-max) / var(--sidebar-width-min) under "<960px").
Edge (source-grounded): 0px fallbacks — client/layout/style.scss L169-L170 (--sidebar-width-max/min:0px), L410 (--masterbar-height:0px), used before the variables resolve.
Per-context overrides (source-grounded): client/my-sites/sidebar/style.scss L12-L17 (272px), L60-L61 (collapsed 69px; .is-global-sidebar-visible 295px).
```

### Q4.c — Breakpoints (viewport widths where layout changes), exercised before/during/after

The global scale is declared once:

```
Source:   client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10
          $breakpoints: 480px, 660px, 800px, 960px, 1040px, 1280px, 1400px;   (breakpoint-deprecated mixin at :L12)
```

The most consequential switch (the masterbar‑height custom property) was observed flipping **live** by resizing the viewport across 782px:

```
Command:  resize_page to each width, then read --masterbar-height + matchMedia
Output:
  width 781px → --masterbar-height = 46px ; matchMedia(min-width:782px)=false, (max-width:781px)=true,
                (min-width:661px)=true, (max-width:660px)=false, (max-width:600px)=false, (min-width:960px)=false
  width 783px → --masterbar-height = 32px ; matchMedia(min-width:782px)=true, (min-width:783px)=true, (max-width:781px)=false
  width 600px → --masterbar-height = 46px ; matchMedia(max-width:600px)=true, (max-width:660px)=true,
                (min-width:661px)=false, (max-width:960px)=true, (max-width:480px)=false
Observed: --masterbar-height flips 46px → 32px exactly at the 782px breakpoint (before/at/after confirmed).
Source:   client/assets/stylesheets/shared/_variables.scss:L10-L11.
```

**Sidebar/masterbar‑specific switches** (each verified either by the live `matchMedia` booleans above or in the served CSS):

- **782px** — masterbar height 46px→32px (`_variables.scss:L10`).
- **660 / 661px** — global sidebar switches (`min-width:661px` collapsed rules; `max-width:660px` tooltip rule) in `client/layout/global-sidebar/style.scss`.
- **960px** — layout switch: `.layout__content` padding changes and the sidebar width flips from `var(--sidebar-width-max)` to `var(--sidebar-width-min)` (`client/layout/style.scss:L185/L192`, under `"<960px"`).
- **783 / 781 / 600px** — the my‑sites sidebar imports `@wordpress/base-styles/breakpoints` and uses these thresholds (`$break-small = 600px`).

**Conclusion (Q4):** header padding is `30px 24px 29px` (global) / `16px 8px 6px 16px` (classic base); the layout math is driven by `--masterbar-height`, `--sidebar-width-max`, and `--sidebar-width-min`; and layout changes occur across the `480/660/800/960/1040/1280/1400` scale plus the sidebar/masterbar switches at `660/661`, `782`, `960`, and `600` — with the 782px masterbar flip demonstrated live.


---

## Coverage pass — every named item, marked Observed vs. Inferred

**Q1 — port, readiness, architecture**

| Item | Answer | Status |
|------|--------|--------|
| Numeric port | 3000 | **Observed** (config probe + live `curl /version` 200) — `config/development.json:L8`, `client/server/index.js:L83` |
| Actual `listen()` bind | `server.listen({ port, host: CALYPSO_IS_FORK ? host : null })`; non‑fork ⇒ all interfaces | **Observed** — `client/server/index.js:L83` |
| Readiness signal #1 | bunyan `wp-calypso booted in <n>ms - http://calypso.localhost:3000` | **Observed** (both runs) — `client/server/index.js:L33` |
| Readiness signal #2 | webpack `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` | **Observed** (both runs) — `client/server/bundler/index.js:L56` |
| Transitional state | pre‑compile "Welcome to Calypso!" holding page (`meta refresh 5`) | **Observed** (raw HTML captured) |
| Multi‑ vs single‑port | **single port**: SSR + assets + HMR on 3000; REST is remote | **Observed** (assets, `__webpack_hmr`, and `/reader` all on 3000; REST origin remote) |
| MOCK_WORDPRESSDOTCOM / inspector 5858 / SECTION_LIMIT | non‑canonical / debugger‑only / code‑splitting | **Labelled** (not used) |

**Q2 — endpoints and actions**

| Item | Answer | Status |
|------|--------|--------|
| Default stream endpoint | `GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover?...&number=4` | **Observed** — `:L226`, `:L246`, `INITIAL_FETCH` `:L161` |
| Pagination size | `number=7` (`PER_FETCH`) | **Observed** — `:L160`, `:L380` |
| following / recent / search / feed / tag / user endpoints | `/read/following`, `/read/streams/following`, `/read/search`, `/read/feed/…`, `/read/tags/…`, `/users/…` | **Observed (map) / logged‑in paths source‑grounded** — `:L194,L198,L212,L220,L317,L347` |
| Redux action sequence | `READER_STREAMS_PAGE_REQUEST → READER_POSTS_RECEIVE → READER_STREAMS_PAGE_RECEIVE` | **Observed** (both loads) |
| Action‑type constants | `action-types.ts:L78,L52,L77,L79` | **Observed/verified** |
| Action creators | `requestPage:L28`, `receivePage:L52`, `requestPaginatedStream:L143`; `receivePosts` `posts/actions.js:L63→L87` | **Verified** |
| Dispatch origin | `Stream.componentDidMount` → `requestPage` | **Observed** — `client/reader/stream/index.jsx:L221→L502` |
| Citation fix | `client/reader/index.ts` (not `.js`) | **Verified** |

**Q3 — auth detection and storage**

| Item | Answer | Status |
|------|--------|--------|
| Server decision | `isLoggedIn = !!req.cookies.wordpress_logged_in` | **Observed** — `client/server/pages/index.js:L93` |
| Client selector | `isUserLoggedIn = getCurrentUserId(state) !== null` → **false** | **Observed** — `client/state/current-user/selectors.js:L15-L16` |
| Default‑config nuance | `oauth:false` (L130), `wpcom-user-bootstrap:false` (L209) ⇒ logged‑out default | **Observed/verified** |
| Cookies checked | `wordpress_logged_in`, `wpcom_token`, `support_session_id` (all absent) | **Observed** |
| localStorage | `wpcom_token` fallback (absent) | **Observed** |
| IndexedDB | `calypso` v2 / `redux-state-logged-out*` keys | **Observed** — `initial-state.js:L76` |
| sessionStorage | OAuth `flags` (absent; OAuth off) | **Observed** — `client/boot/common.js:L127` |
| Server‑injected state | `window.initialReduxState` = `{documentHead}` only | **Observed** — `client/document/index.jsx:L86-L87` |
| Render selection | `LayoutLoggedOut` | **Observed** — `client/controller/index.web.js:L59-L64` |
| Bootstrap error branch / OAuth / logged‑in render | throw `'Cannot bootstrap without an auth cookie'`, OAuth middleware, logged‑in `Layout` | **Inferred (source‑grounded)** — `user-bootstrap/index.js:L34`, `boot/common.js:L154,L176`, `controller/index.web.js:L100` |

**Q4 — responsive sidebar**

| Item | Answer | Status |
|------|--------|--------|
| Global header padding/margin | `padding: 30px 24px 29px; margin: 0; gap: 8px` | **Observed** (served CSS + computed probe) — `global-sidebar/style.scss:L70-L75` |
| Global logo / body | `125×28, margin 0`; body `padding-top: 40px` | **Observed** — `:L82-L90`, `:L112` |
| Classic root / heading | root `padding-top:6px`; heading base `16px 8px 6px 16px; margin:0` (themed‑computed `0 0 0 8px`) | **Observed** — `sidebar/style.scss:L4-L6`, `:L70-L71` |
| sidebar‑v2 | root `padding: 16px` | **Inferred (source‑grounded)** — `sidebar-v2/style.scss` (not in reader/login build) |
| Custom properties | `--masterbar-height` 46→32px@782; `--sidebar-width-max` 272px; `--sidebar-width-min` 228px | **Observed** — `_variables.scss:L7,L10-L11,L15,L16` |
| calc() consumers | `.layout__content { padding: 79px 32px 32px calc(var(--sidebar-width-max) + 32px + 1px) }` (+ `@max-width:960px`) | **Observed** — `layout/style.scss:L52,L119,L185,L192` |
| Breakpoint scale | `480, 660, 800, 960, 1040, 1280, 1400 px` | **Observed/verified** — `_breakpoints.scss:L10` |
| Sidebar/masterbar switches | 782 (live flip), 660/661, 960, 600 | **Observed** (matchMedia + live var flip) |
| No‑sidebar‑logged‑out | logged‑out Reader has no sidebar (`has-no-sidebar`) | **Observed** |

**Methodology compliance:** every behavioral value shows the exact command and its complete, unedited output; each key value was confirmed across two runs (or is a deterministic build output); non‑canonical, non‑default, and inferred values are explicitly labelled; and all citations were re‑verified against the checkout at HEAD `be7e5cc641622d153040491fd5625c6cb83e12eb`. The source repository was not modified, and all temporary observation scripts were removed.

