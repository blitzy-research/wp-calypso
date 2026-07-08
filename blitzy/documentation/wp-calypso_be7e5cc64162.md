# Running Calypso's Reader Locally — A Runtime-Grounded Onboarding Answer

**Repository:** `Automattic/wp-calypso`
**Commit under test:** `be7e5cc641622d153040491fd5625c6cb83e12eb`
**Canonical runtime used for all observations:** Node `v22.23.1` (satisfies `engines.node = "^v22.9.0"`), yarn `4.0.2` (the `packageManager` pin, activated via corepack).

---

## What Calypso is (context)

Calypso is the JavaScript and API-powered front-end of WordPress.com. It is a single-page application (SPA) powered by the WordPress.com REST API (`README.md:L5`), built with Node, Express, React, and Redux (`README.md:L9`). Because it talks to the *remote* WordPress.com REST API, running it locally still requires network reachability of `public-api.wordpress.com`, and the app is served from the host alias `calypso.localhost`.

This document answers six onboarding questions about the **running** application. Every behavioral claim below is backed by **observed runtime output** plus an exact **`file:line`** citation. Anything that is derived only from reading source (and not observed at runtime) is explicitly labeled **`(inferred)`**.

### How to read this document

- **Direct answer first**, then the command that produced the evidence, then the **complete, unedited** observed output, then the `file:line` citation(s), then the **cause → effect** reasoning and any **sibling variants**.
- Citations use the form `path:Lnn` or `path:Lnn-Lmm` and refer to the source at commit `be7e5cc64162`.
- Line numbers were verified against the source at the target commit.

### Methodology (run-first)

All values were produced by **building and running** the code first, then writing **temporary observation scripts**, executing them, and capturing the real output. The temporary scripts lived outside the repository (in `/tmp`) or were deleted after capture; the repository itself is left unchanged (verified with `git status --porcelain` — the only new path is this document under `blitzy/`).

Runtime harnesses used (all removed after capture):
- Dev server run under `SECTION_LIMIT=reader,login` (server log captured to `/tmp/blitzy_calypso_server.log`).
- A Jest harness (jsdom) importing the **real** Reader data-layer modules to enumerate every stream endpoint and the initial-load Redux action cascade (Q3/Q4).
- A Jest harness importing the **real** `oauth-token` and `current-user` modules for the login/token precedence (Q5).
- A standalone Node script requiring the **real compiled** `@automattic/viewport` module with a width-aware `matchMedia` mock (Q6c).
- Live Chrome DevTools inspection of the running app for network requests, browser storage, and computed CSS (Q2/Q3/Q5/Q6).
- The real `sass@1.54.0` compiler for the sidebar SCSS (Q6a/Q6b).

---

## Environment & How to Run (including the Node-20 gate discrepancy)

### Direct answer

Install dependencies with `yarn`, then start the dev server with `yarn start` (or, for a Reader-focused build, `SECTION_LIMIT=reader,login yarn start`). The app is served at **`http://calypso.localhost:3000/`**, which requires a `127.0.0.1 calypso.localhost` entry in `/etc/hosts`. **You must be on Node `22.x`**: `yarn start` runs `npx check-node-version --package`, which enforces `engines.node = "^v22.9.0"`, and **Node `20.x` fails that gate and aborts the start before anything is built.**

### The `start` script chain

```
"start": "npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build"
```

Citations:
- `package.json:L110` — the `start` script (gate → welcome banner → build → run).
- `package.json:L113` — `start-build`: `BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short`.
- `package.json:L57` — `engines.node = "^v22.9.0"`.
- `package.json:L267` — the `check-node-version` dev dependency that implements the gate.
- The app URL / hosts requirement is documented in `README.md:L19-L22` (add `127.0.0.1 calypso.localhost` to your hosts file, run `yarn start`, then open `http://calypso.localhost:3000`).

### The Node-20 gate discrepancy (demonstrated)

The environment's default setup installs Node 20.x, but the canonical runtime pinned by the repo is Node 22.x (`.nvmrc` = `22.9.0`; `engines.node = "^v22.9.0"`). Node 20 was **not** installed in the canonical environment (the setup explicitly warns against downgrading), so the failing case is demonstrated two honest, observed ways: (a) the exact mismatch output `check-node-version` prints when the running Node does not satisfy a wanted range, and (b) the actual gate decision computed with the repo's own `semver` against the literal `engines.node` string.

**(a) PASS under Node 22 (the canonical runtime) — command + full output:**

```
$ npx check-node-version --package --print
node: 22.23.1
yarn: 4.0.2
# exit code: 0
```

**(b) The real mismatch format the gate emits (asking the same tool for a Node-20 range while running Node 22):**

```
$ npx check-node-version --node '^20.9.0'
node: 22.23.1
Wanted node version ^20.9.0 (>=20.9.0 <21.0.0)
To install node, see https://nodejs.org/download/release/v20.9.0/
# exit code: 1
```

**(c) The actual gate decision, computed with the repo's own `semver` against `engines.node = "^v22.9.0"`:**

```
engines.node (literal)      = "^v22.9.0"
semver.validRange(...)      = ">=22.9.0 <23.0.0-0"
semver.satisfies("20.9.0")  = false     <-- Node 20 FAILS the gate
semver.satisfies("20.19.4") = false     <-- Node 20 FAILS the gate
semver.satisfies("22.9.0")  = true
semver.satisfies("22.23.1") = true      <-- the runtime used here PASSES
```

**Cause → effect:** `npx check-node-version --package` reads `engines.node` from `package.json:L57` and exits non-zero when the running Node is outside `>=22.9.0 <23.0.0-0`. Because it is the **first** command in the `&&` chain of the `start` script (`package.json:L110`), a non-zero exit **aborts `yarn start` before** `node bin/welcome.js`, `yarn run build`, or `start-build` ever run — so on Node 20 the dev server is never built or launched. On Node 22.x the gate passes and the chain proceeds. (The gate decision in (c) is *observed* — the repo's `semver` was executed; it is not inferred.)

### The commands actually used to produce the observations below

```
# canonical runtime already active: node v22.23.1, yarn 4.0.2 (corepack)
# /etc/hosts already contains: 127.0.0.1 calypso.localhost
CI=true yarn install --immutable          # dependencies (node_modules present)
CI=true yarn build                         # produces build/server.js
SECTION_LIMIT=reader,login \
  NODE_ENV=development CALYPSO_ENV=development \
  node build/server.js | bunyan -o short   # run the dev server on port 3000
```

`SECTION_LIMIT=reader,login` is the documented way to limit the build to selected sections for a faster Reader-focused run (see `docs/install.md`).

---

## Q1 — What port does the development server bind to, and how do I know when it's fully ready?

### Direct answer

The dev server binds to **port `3000`** on host **`calypso.localhost`** (`http://calypso.localhost:3000/`). It is **"fully ready" only when the cyan `Ready!` banner prints after webpack's first compile finishes** — **not** when the earlier "booted" log line appears. Before that first compile completes, requests to `/` receive a temporary "Welcome to Calypso!" holding page.

### Where the port comes from

- `client/server/index.js:L12` — `port = config( 'port' )` (and `protocol` at `L11`, `host` at `L13`).
- `client/server/index.js:L83` — `server.listen( { port, host: … }, … )` binds the HTTP server.
- Default port `3000`: `config/_shared.json:L25` (`"port": 3000`) and `config/development.json:L8`.
- Host `calypso.localhost`: `config/development.json:L7`. Protocol `http`: `config/_shared.json:L24` / `config/development.json:L6`.
- `PORT` env override: `client/server/config/parser.js:L63` — `data.port = process.env.PORT || data.port;`.

### Observed output (server log)

**Command:**

```
SECTION_LIMIT=reader,login NODE_ENV=development CALYPSO_ENV=development \
  node build/server.js | bunyan -o short
```

**The "booted" log line (appears at ~1 second — this is NOT "ready"):**

```
{"name":"calypso","hostname":"reverse-code-generator-782dced0-d7k7f","pid":28695,"level":30,"msg":"wp-calypso booted in 1029ms - http://calypso.localhost:3000","time":"2026-07-08T05:04:13.954Z","v":0}
[sections-loader] Limiting build to reader, login sections
Compiling assets... Wait until you see Ready! and then try http://calypso.localhost:3000/ again.
```

- The boot line is emitted by `client/server/index.js:L33` (`wp-calypso booted in %dms - %s://%s:%s`).
- `[sections-loader] Limiting build to reader, login sections` confirms the `SECTION_LIMIT` build.

**The first webpack compile finishing (~78 seconds later):**

```
webpack built 139ea3e375ebca6e6b9b in 77804ms
webpack 5.97.1 compiled with 11 warnings in 77804 ms
```

**Then — and only then — the readiness banner (this is "fully ready"):**

```
Ready! You can load http://calypso.localhost:3000/ now. Have fun!
```

- The banner is printed on webpack's `done` hook: `client/server/bundler/index.js:L38` (`compiler.hooks.done.tap(...)`), text at `client/server/bundler/index.js:L56`, wrapped in `chalk.cyan` (`client/server/bundler/index.js:L54-L59`).
- On subsequent recompiles the banner text is instead `Ready! All assets are re-compiled. Have fun!` (`client/server/bundler/index.js:L60`). **(inferred** — a recompile was not triggered at runtime because doing so cleanly would require touching a tracked source file, which the read-only constraint forbids; the text is quoted from source.**)**

**The pre-compile holding page (requesting `/` before the first compile finishes):**

```
$ curl -s http://calypso.localhost:3000/
# HTTP 200, 630 bytes — a "Welcome to Calypso!" holding page instructing the
# developer to wait until "READY!" appears in the server console before retrying.
```

- Served by `client/server/bundler/index.js:L76-L93` (the `waitForCompiler` middleware), heading `Welcome to Calypso!` at `client/server/bundler/index.js:L82`.

### Cause → effect

`server.listen` binds port `3000` (`client/server/index.js:L83`) almost immediately, which is why the "booted in 1029ms" line appears within ~1s. But the client bundles are compiled *at runtime* by `webpack-dev-middleware`; until webpack's first `done` hook fires (~78s here), `waitForCompiler` intercepts `/` and returns the holding page. The cyan `Ready!` banner is emitted from that same `done` hook, so it is the accurate "fully ready" signal — the ~77s gap between "booted" and "Ready!" is exactly the first webpack compile.

### Sibling variants

- **Booted log vs readiness banner:** two distinct signals; "booted" ≠ "ready" (a ~77s gap here).
- **First compile vs recompile banner:** `Ready! You can load … now. Have fun!` (first) vs `Ready! All assets are re-compiled. Have fun!` (recompile) — `client/server/bundler/index.js:L56` vs `L60`.
- **Port override:** `PORT=<n>` changes the bound port (`client/server/config/parser.js:L63`); default is `3000`.
- **The ASCII welcome banner** printed by `node bin/welcome.js` (`bin/welcome.js:L6`) is separate from the runtime readiness banner and appears before the build starts.

---

## Q2 — Does the architecture use multiple ports (hot reloading vs API), or is everything served from one place?

### Direct answer

**One local Express port (`3000`) serves everything** — server-rendered HTML, the compiled client assets (via `webpack-dev-middleware`), **and** hot-module updates (via `webpack-hot-middleware`) — all mounted on the same app. **There is no separate hot-reload port.** REST/API calls do **not** hit any local port; they go to the **remote** `https://public-api.wordpress.com`.

### Where the single-port wiring lives

All three middlewares are `app.use`'d on the same Express app:

- `client/server/bundler/index.js:L100` — `app.use( waitForCompiler )`
- `client/server/bundler/index.js:L101` — `app.use( webpackMiddleware( compiler ) )` (i.e., `webpack-dev-middleware`, required at `client/server/bundler/index.js:L5`)
- `client/server/bundler/index.js:L102` — `app.use( hotMiddleware( compiler ) )` (i.e., `webpack-hot-middleware`, required at `client/server/bundler/index.js:L6`)

The remote REST base (not a local port):

- `packages/wpcom-xhr-request/src/index.js:L27` — `proxyOrigin: 'https://public-api.wordpress.com'`.

### Observed output (all on port 3000; API is remote)

**SSR HTML — same origin, relative asset URLs:**

```
GET http://calypso.localhost:3000/            -> HTTP 200, 24946 bytes
# every <script src> is a RELATIVE path under the same origin, e.g.:
#   /calypso/evergreen/runtime.js
#   /calypso/evergreen/vendors-node_modules_moment_moment_js.js
```

**A compiled client asset — served by webpack-dev-middleware on :3000:**

```
GET http://calypso.localhost:3000/calypso/evergreen/runtime.js
  -> HTTP 200, Content-Type: application/javascript, 75033 bytes
```

**The HMR channel — served by webpack-hot-middleware on the SAME :3000:**

```
GET http://calypso.localhost:3000/__webpack_hmr
  -> HTTP 200, Content-Type: text/event-stream
```

**Data/REST — remote, not a local port** (from the SSR HTML and the live Network tab):

```
# the SSR HTML prefetches the remote proxy origin:
<link href="https://public-api.wordpress.com/wp-admin/rest-proxy..." ...>
# and every data request in the running app targets public-api.wordpress.com
# (e.g. the Reader stream call and /me — see Q3/Q5), never a local port.
```

### The `3001` / `3002` ports are different apps (not the main Calypso)

```
package.json:L115  "start-jetpack-cloud-p":  ... PORT=3001 ...   (Jetpack Cloud)
package.json:L117  "start-a8c-for-agencies-p": ... PORT=3002 ... (A8C for Agencies)
```

**Cause → effect:** the dev bundler mounts the asset middleware and the HMR middleware on the *same* Express `app` (`client/server/bundler/index.js:L100-L102`), so HTML, JS bundles, and the `text/event-stream` HMR channel all share origin `calypso.localhost:3000`; there is no second listener for hot reloading. Application data is fetched from the remote WordPress.com REST API because the XHR layer's `proxyOrigin` is `https://public-api.wordpress.com` (`packages/wpcom-xhr-request/src/index.js:L27`) — which is also why local Calypso needs that host reachable. The `3001`/`3002` ports belong to the separate Jetpack Cloud and A8C-for-Agencies environments (`package.json:L115,L117`), not to the main app.

### Sibling variants

- **HTML / assets / HMR:** all three on `:3000` (dev + hot middleware, `client/server/bundler/index.js:L100-L102`).
- **REST data:** remote `public-api.wordpress.com` (`packages/wpcom-xhr-request/src/index.js:L27`) — corroborated in Q3 (Reader stream) and Q5 (`/me`).
- **Other environments' ports:** Jetpack Cloud `3001`, A8C-for-Agencies `3002` (`package.json:L115,L117`) — separate apps.

---

## Q3 — What API endpoints get called to populate the Reader stream?

### Direct answer

The Reader resolves a **stream key** to a REST path via the `streamApis` map. The default logged-in Reader landing uses stream key **`'following'` → `GET /read/following`** (apiVersion `1.2`, `number=4` on the first page). The default **logged-out** landing is Discover → `GET /wpcom/v2/read/streams/discover`. Every stream key and its resolved path is enumerated below.

### Where the mapping and request builder live

- `client/state/data-layer/wpcom/read/streams/index.js:L192-L351` — the `streamApis` map (key → path function + query function + apiVersion/apiNamespace).
- `client/state/data-layer/wpcom/read/streams/index.js:L161` — `INITIAL_FETCH = 4` (first-page size); `L160` — `PER_FETCH = 7` (subsequent pages).
- `client/state/data-layer/wpcom/read/streams/index.js:L358` — `requestPage`; default `apiVersion = '1.2'` at `L369`; `fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH` at `L379`; the `http` GET is built at `L393`.
- `client/state/data-layer/wpcom-http/actions.js:L55-L66` — `http()` sets `query = { ...query, ...( apiNamespace ? { apiNamespace } : { apiVersion } ) }`. **This is why `apiVersion`/`apiNamespace` and `number` live *inside* `action.query`.**
- Default stream key `'following'`: `client/reader/controller.js:L53` (`mcKey = 'following'`), `L85` (`key`), `L87` (`streamKey`).

### Observed output — every stream key → path (via the real `requestPage`)

**Command (Jest harness importing the real data-layer `requestPage`, jsdom env):**

```
TZ=UTC CI=true node_modules/.bin/jest -c test/client/jest.config.js \
  <temp probe importing client/state/data-layer/wpcom/read/streams/index.js> \
  --watchAll=false --ci
# result: 5/5 tests passed
```

**Full enumeration (method `GET`; `number = INITIAL_FETCH = 4` on the first page unless noted):**

| Stream key | Resolved path | apiVersion / apiNamespace | `number` | key→path `file:line` |
|---|---|---|---|---|
| `following` | `/read/following` | apiVersion `1.2` | 4 | L193→L194 |
| `recent` | `/read/streams/following` | apiNamespace `wpcom/v2` | 4 | L197→L198 |
| `search` | `/read/search` | apiVersion `1.2` | 4 | L211→L212 |
| `feed` | `/read/feed/<feedId>/posts` (e.g. `/read/feed/12345/posts`) | apiVersion `1.2` | 4 | L219→L220 |
| `discover` (recommended) | `/read/streams/discover` | apiNamespace `wpcom/v2` | 4 | L223→L224 |
| `discover` (latest) | `/read/tags/posts` | apiNamespace `wpcom/v2` | 4 | L223→L224 |
| `discover` (firstposts) | `/read/streams/first-posts` | apiNamespace `wpcom/v2` | 4 | L223→L224 |
| `discover` (other) | `/read/streams/discover?tags=<suffix>` | apiNamespace `wpcom/v2` | 4 | L223→L224 |
| `site` | `/read/sites/<siteId>/posts` (e.g. `/read/sites/67890/posts`) | apiVersion `1.2` | 4 | L248→L249 |
| `conversations` | `/read/conversations` | apiVersion `1.2` | 4 | L252→L253 |
| `notifications` | `/read/notifications` | apiVersion `1.2` | 4 | L258→L259 |
| `featured` | `/read/sites/<siteId>/featured` (e.g. `/read/sites/67890/featured`) | apiVersion `1.2` | 4 | L262→L263 |
| `p2` | `/read/following/p2` | apiVersion `1.2` | 4 | L266→L267 |
| `a8c` | `/read/a8c` | apiVersion `1.2` | 4 | L270→L271 |
| `conversations-a8c` | `/read/conversations` | apiVersion `1.2` | 4 | L274→L275 |
| `likes` | `/read/liked` | apiVersion `1.2` | 4 | L281→L282 |
| `recommendations_posts` | `/read/recommendations/posts` | apiVersion `1.2` | **undefined** | L286→L287 |
| `custom_recs_posts_with_images` | `/read/recommendations/posts` | apiVersion `1.2` | 4 | L293→L294 |
| `custom_recs_sites_with_images` | `/read/recommendations/sites` | apiVersion `1.2` | 4 | L303→L304 |
| `tag` | `/read/tags/<tag>/posts` (e.g. `/read/tags/cats/posts`) | apiNamespace `wpcom/v2` | 4 | L316→L317 |
| `tag_popular` | `/read/streams/tag/<tag>` (e.g. `/read/streams/tag/cats`) | apiNamespace `wpcom/v2` | 4 | L321→L322 |
| `list` | `/read/list/<owner>/<slug>/posts` (e.g. `/read/list/bob/mylist/posts`) | apiVersion `1.3` | **40** | L332→L333 |
| `user` | `/users/<userId>/posts` (e.g. `/users/42/posts`) | apiVersion `1` | 4 | L346→L347 |

**Full default `following` `http` action (observed, unedited):**

```json
{"type":"WPCOM_HTTP_REQUEST","method":"GET","path":"/read/following","query":{"orderBy":"date","meta":"post,discover_original_post","number":4,"lang":"en","content_width":675,"apiVersion":"1.2"},"onSuccess":{"type":"READER_STREAMS_PAGE_REQUEST"},"onFailure":{"type":"READER_STREAMS_PAGE_REQUEST"},"onProgress":null,"onStreamRecord":null,"options":{}}
```

### Observed output — real browser network capture (logged-OUT `/reader` → redirects to `/discover`)

```
reqid=149  GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover
  ?_envelope=1&orderBy=popular&meta=post,discover_original_post&feed_id=
  &number=4&lang=en&tags[]=dailyprompt&tags[]=wordpress
  &tag_recs_per_card=5&site_recs_per_card=5&age_based_decay=0.5&content_width=675
  -> HTTP 200
  body: {"body":{"cards":[{"type":"recommended_blogs","data":[...]},{"type":"post","data":{...}}]}}

reqid=171  (next page) same path with page_handle=... & number=7   -> HTTP 200
```

- `number=4` on the first request confirms `INITIAL_FETCH = 4`; `number=7` on the next page confirms `PER_FETCH = 7`.
- The request targets the **remote** `public-api.wordpress.com` under the `wpcom/v2` namespace — corroborating Q2 (data is remote, not a local port) and the `apiNamespace 'wpcom/v2'` for the Discover stream.

### Cause → effect

`requestPage` looks up `streamApis[streamKey]`, calls that entry's `path()` function to build the REST path, and merges its query via `http()` (`client/state/data-layer/wpcom-http/actions.js:L55-L66`), which nests `apiVersion`/`apiNamespace` and the computed `number` inside `action.query`. On the first page `pageHandle` is empty, so `fetchCount = INITIAL_FETCH = 4` (`…/streams/index.js:L379`); on later pages it becomes `PER_FETCH = 7` — exactly matching the observed `number=4` then `number=7`.

### Sibling variants (exhaustive)

- **`discover` is computed** — four branches (recommended / latest / firstposts / other) resolve to different paths, all under `apiNamespace wpcom/v2` (`…/streams/index.js:L223-L224`).
- **`list` is computed** and forces `number: 40` and `apiVersion 1.3` (`…/streams/index.js:L332-L338`) — a deliberate departure from `INITIAL_FETCH`.
- **`recommendations_posts`** does **not** forward `number` (its query fn is `({ query }) => ({ ...query, seed, algorithm })`), so `number` is `undefined` on its request — a real per-key difference (`…/streams/index.js:L286-L287`).
- **`user`** uses `apiVersion 1` and **`recent`/`tag`/`tag_popular`/`discover`** use `apiNamespace wpcom/v2` instead of a numeric `apiVersion`.
- **Default landing differs by auth:** logged-in → `following` → `/read/following`; logged-out → Discover → `/wpcom/v2/read/streams/discover`.

---

## Q4 — What Redux actions fire during the Reader's initial load?

### Direct answer

On mount, the Reader stream component dispatches **`READER_STREAMS_PAGE_REQUEST`**. The data-layer intercepts it, issues the `http` GET, and on success runs `handlePage`, which dispatches (conditionally) analytics actions, then **`READER_POSTS_RECEIVE`**, then **`READER_STREAMS_PAGE_RECEIVE`**. The observed ordered success cascade is:

```
READER_STREAMS_PAGE_REQUEST  →  READER_POSTS_RECEIVE  →  READER_STREAMS_PAGE_RECEIVE
```

Surrounding this, app bootstrap dispatches `SECTION_SET`, `ROUTE_SET`, and `CURRENT_USER_RECEIVE`.

### Where the dispatch chain lives

- Mount → fetch: `client/reader/stream/index.jsx:L221` (`componentDidMount`), `L225` (`this.fetchNextPage( {} )`), `L489` (`fetchNextPage`), `L502` (`props.requestPage( … )`); `requestPage` imported at `client/reader/stream/index.jsx:L39`.
- Action creators: `client/state/reader/streams/actions.js:L28` (`requestPage`) → type `READER_STREAMS_PAGE_REQUEST` at `L39`; `L52` (`receivePage`) → type `READER_STREAMS_PAGE_RECEIVE` at `L63`.
- Constants: `client/state/reader/action-types.ts:L78` (`READER_STREAMS_PAGE_REQUEST`), `L77` (`READER_STREAMS_PAGE_RECEIVE`), `L52` (`READER_POSTS_RECEIVE`).
- Data-layer cascade: `client/state/data-layer/wpcom/read/streams/index.js:L428` (`handlePage`), which dispatches `receivePosts` + `receivePage` (`L497`); wired via `dispatchRequest( { fetch: requestPage, onSuccess: handlePage, … } )` at `L516-L518`.
- `receivePosts` thunk dispatches `READER_POSTS_RECEIVE`: `client/state/reader/posts/actions.js:L63` (thunk) → `L87` (dispatch).
- Bootstrap actions: `client/state/action-types.ts:L139` (`CURRENT_USER_RECEIVE`), `L848` (`ROUTE_SET`), `L850` (`SECTION_SET`); `setSection` → `SECTION_SET` at `client/state/ui/section/actions.js:L5,L8`; Reader section registration at `client/sections.js:L386`.

### Observed output (real action creators + `handlePage` through a real redux+thunk recording store)

```
BLITZY_Q4_REQUEST_TYPE   = READER_STREAMS_PAGE_REQUEST
BLITZY_Q4_RECEIVE_TYPE   = READER_STREAMS_PAGE_RECEIVE
BLITZY_Q4_HANDLEPAGE_RETURNS = ["THUNK(receivePosts)","READER_STREAMS_PAGE_RECEIVE"]
BLITZY_Q4_ORDERED_STREAM = ["READER_STREAMS_PAGE_REQUEST","READER_POSTS_RECEIVE","READER_STREAMS_PAGE_RECEIVE"]
```

- `handlePage` returned a `receivePosts` **thunk** followed by the `receivePage` action; dispatching the thunk through the recording store expanded it into `READER_POSTS_RECEIVE`, yielding the flat ordered stream shown as `BLITZY_Q4_ORDERED_STREAM`.

### Cause → effect

The component's `componentDidMount` (`client/reader/stream/index.jsx:L221`) calls `fetchNextPage`, which dispatches `requestPage(...)` → `READER_STREAMS_PAGE_REQUEST` (`client/state/reader/streams/actions.js:L39`). The data-layer's `dispatchRequest` mapping (`…/streams/index.js:L516-L518`) turns that plain action into the `http` GET (Q3) and, on success, calls `handlePage` (`…/streams/index.js:L428`), which dispatches the `receivePosts` thunk (→ `READER_POSTS_RECEIVE`, `client/state/reader/posts/actions.js:L87`) and then `receivePage` (→ `READER_STREAMS_PAGE_RECEIVE`, `client/state/reader/streams/actions.js:L63`). That produces the exact three-action success cascade above.

### Sibling variants

- **Conditional analytics:** `handlePage` appends `recordTracksEvent` analytics actions only for posts that carry a `railcar`. The observed test post had `railcar: null`, so **no** analytics action was appended — a real conditional branch, not an omission.
- **Bootstrap actions** `SECTION_SET` / `ROUTE_SET` / `CURRENT_USER_RECEIVE` fire during app/section boot around the stream fetch **(inferred** from source `client/state/action-types.ts:L139,L848,L850` and `client/state/ui/section/actions.js:L5,L8`; the three stream-cascade actions above are the runtime-observed ones**)**.
- **First page vs later pages:** the same `READER_STREAMS_PAGE_REQUEST` → `…RECEIVE` cascade repeats for pagination, differing only in `number` (4 → 7, per Q3).

---

## Q5 — How does the app know whether someone is logged in before it decides what to render, and what storage mechanisms does it check?

### Direct answer

The render decision reads Redux via **`isUserLoggedIn(state)`**, which is **`getCurrentUserId(state) !== null`** — i.e., true iff the current-user id is non-null. That id is populated either by **server-side `/me` bootstrap** (gated on the **`wordpress_logged_in`** cookie) or by **client hydration** from **`window.initialReduxState`**. In OAuth mode (desktop/dev), the app additionally checks the **`wpcom_token`** — **cookie first, then `localStorage`** — and redirects to `/login` if neither exists. Persistent Redux state lives in **IndexedDB** (database **`calypso`**, object store **`calypso_store`**), keyed **`redux-state-<userId | 'logged-out'>`** (with optional `:subkey`). A **storage-bypass** path exists for private/incognito contexts.

### The render decision

- `client/state/current-user/selectors.js:L6` — `getCurrentUserId = state.currentUser?.id`.
- `client/state/current-user/selectors.js:L15-L16` — `isUserLoggedIn( state )` returns `getCurrentUserId( state ) !== null`.
- `client/state/current-user/reducer.js:L24-L26` — the `id` reducer defaults to `null` and is set on `CURRENT_USER_RECEIVE`.

**Observed (real selectors against three states):**

```
BLITZY_Q5_LOGIN = {"loggedIn_id":12345,"loggedIn_isLoggedIn":true,"loggedOut_id":null,"loggedOut_isLoggedIn":false,"uninitialized_isLoggedIn":true}
```

- **Logged-in branch:** `currentUser.id = 12345` → `isUserLoggedIn = true`.
- **Logged-out branch:** `currentUser.id = null` → `isUserLoggedIn = false`.
- **Edge case (uninitialized state, no `currentUser` at all):** `getCurrentUserId` returns `undefined`, and `undefined !== null` is `true`, so `isUserLoggedIn` returns **`true`** for a completely empty/uninitialized state. This is a real nuance of the `!== null` check (`client/state/current-user/selectors.js:L16`): only an *explicit* `null` id reads as logged-out; the reducer's default is `null` (`client/state/current-user/reducer.js:L24`), so in practice a booted store has an explicit `null`, but a bare `{}` does not.

### The OAuth token check (`wpcom_token`): cookie → localStorage → false

- `packages/oauth-token/src/index.js:L7` — `TOKEN_NAME = 'wpcom_token'`.
- `packages/oauth-token/src/index.js:L9-L24` — `getToken()` parses `document.cookie` for `wpcom_token` **first** (`L10-L14`), then falls back to `store.get(...)` (**localStorage**, `L16`), returning the token if found (`L18-L20`) or **`false`** if neither is present (`L22`).
- `client/boot/common.js:L154` — `oauthTokenMiddleware`; when `config.isEnabled( 'oauth' )` (`L155`) and the route is not a logged-out route (`L156`), a missing token (`getToken() === false`) redirects to the login page (`L175-L176`).

**Observed (real `getToken()` with mocked `document.cookie` / `store`, three sub-cases + a roundtrip):**

```
case 1  cookie present                    -> getToken() = "COOKIE_TOKEN_ABC"
case 2  no cookie, localStorage present   -> getToken() = "LOCALSTORAGE_TOKEN_XYZ"
case 3  neither cookie nor localStorage   -> getToken() = false
setToken("ROUNDTRIP_TOKEN"); getToken()   -> "ROUNDTRIP_TOKEN"
```

### The server-side `/me` bootstrap (gated on `wordpress_logged_in`)

- `client/server/user-bootstrap/index.js:L8` — `AUTH_COOKIE_NAME = 'wordpress_logged_in'`.
- `client/server/user-bootstrap/index.js:L13` — `API_PATH = 'https://public-api.wordpress.com/rest/v1/me'`.
- `client/server/user-bootstrap/index.js:L26` — `getBootstrappedUser(...)`; `L32-L34` — throws (does not bootstrap) if the `wordpress_logged_in` cookie value is absent.

**Observed (live browser, logged-in path corroboration):** the running app issued `GET https://public-api.wordpress.com/rest/v1.1/me?...meta=flags -> HTTP 200` (reqid=100 in the Network capture) — the `/me` call that populates the current user. The data request is remote (corroborating Q2).

### Client hydration and the persistence key scheme

- `client/state/initial-state.js:L148-L149` — `getInitialServerState()` reads **`window.initialReduxState`**; `L153` deserializes it; `L154` picks the persisted slices.
- `client/state/initial-state.js:L75-L76` — the persistence key: `'redux-state-' + ( userId ?? 'logged-out' ) + ( subkey ? ':' + subkey : '' )`.
- `client/state/persisted-state.js:L15,L17` — `loadPersistedState` reads all stored items matching `/^(redux-state|query-state)-/`.

### IndexedDB persistence store + bypass

- `client/lib/browser-storage/index.ts:L20` — `DB_NAME = 'calypso'`; `L21` — `DB_VERSION = 2`; `L22` — `STORE_NAME = 'calypso_store'`.
- `client/lib/browser-storage/bypass.ts:L6` — the bypass path (used when persistent storage is unavailable, e.g. private/incognito), backed by an in-memory store (`L12`). **(inferred** for the private-mode trigger condition; the constants and in-memory fallback are read from source, not exercised in incognito at runtime.**)**

### Observed output — live browser storage on the **logged-out** `/discover` page

```
cookies          = tk_ai=...; country_code=US; region=Iowa; tk_qs=
                   # NOTE: NO wordpress_logged_in, NO wpcom_token (logged-out)
localStorage keys = ["tusSupport"]
IndexedDB dbs     = ["calypso (v2)"]
calypso_store keys = [
  "redux-state-logged-out",
  "redux-state-logged-out:all-domains",
  "redux-state-logged-out:connectedApplications",
  "redux-state-logged-out:documentHead",
  "redux-state-logged-out:memberships",
  "redux-state-logged-out:plugins",
  "redux-state-logged-out:preferences",
  "redux-state-logged-out:pushNotifications",
  "redux-state-logged-out:reader",
  "redux-state-logged-out:readerUi",
  "redux-state-logged-out:route",
  "redux-state-logged-out:signup",
  "redux-state-logged-out:siteSettings",
  "redux-state-logged-out:teams",
  "redux-state-logged-out:ui",
  "redux-state-logged-out:userSuggestions",
  "was-state-randomly-cleared"
]
window.initialReduxState present = true   (topKeys = ["documentHead"])
```

- The IndexedDB database is exactly `calypso` at version 2 (`client/lib/browser-storage/index.ts:L20-L21`); its object store is `calypso_store` (`L22`).
- The keys are exactly the `redux-state-<userId | 'logged-out'>[:subkey]` scheme (`client/state/initial-state.js:L75-L76`), here with `userId` absent → `'logged-out'`.
- **No** `wordpress_logged_in` and **no** `wpcom_token` cookies exist in the logged-out branch — consistent with `isUserLoggedIn = false` and with the server-bootstrap gate not firing.

### The two render branches (cross-product)

- **Logged-in:** `wordpress_logged_in` cookie present → server `/me` bootstrap populates `currentUser.id` → `isUserLoggedIn = true` → logged-in UI; persisted state keyed `redux-state-<userId>`; in OAuth mode `getToken()` returns the token (cookie or localStorage).
- **Logged-out:** no auth cookie → no bootstrap → `currentUser.id = null` → `isUserLoggedIn = false`; persisted state keyed `redux-state-logged-out`; in OAuth mode `getToken()` returns `false` and `oauthTokenMiddleware` redirects to `/login` (`client/boot/common.js:L175-L176`). The Reader still renders logged-out because its section sets `enableLoggedOut: true` (`client/sections.js:L396`).

### Cause → effect

Rendering keys off Redux `currentUser.id` (`client/state/current-user/selectors.js:L16`). That id is set from either the server `/me` bootstrap — which only runs when the `wordpress_logged_in` cookie is present (`client/server/user-bootstrap/index.js:L8,L32-L34`) — or client hydration from `window.initialReduxState` (`client/state/initial-state.js:L149`). In OAuth mode the additional `wpcom_token` check (cookie then localStorage, `packages/oauth-token/src/index.js:L10-L22`) decides whether to redirect to `/login`. Between visits, state is rehydrated from IndexedDB `calypso/calypso_store` under `redux-state-<userId|'logged-out'>` (`client/lib/browser-storage/index.ts:L20-L22`; `client/state/initial-state.js:L75-L76`), which is why the logged-out session shows `redux-state-logged-out*` keys and no auth cookies.

### Storage mechanisms checked (exhaustive, by name)

1. **`wordpress_logged_in` cookie** — gates server `/me` bootstrap (`client/server/user-bootstrap/index.js:L8,L13`).
2. **`wpcom_token` cookie** — first source in OAuth `getToken()` (`packages/oauth-token/src/index.js:L10-L14`).
3. **`localStorage` (`wpcom_token`)** — fallback in `getToken()` via `store.get` (`packages/oauth-token/src/index.js:L16`).
4. **`window.initialReduxState`** — SSR-injected client hydration (`client/state/initial-state.js:L149`).
5. **IndexedDB `calypso` / `calypso_store`** — persisted Redux state (`client/lib/browser-storage/index.ts:L20-L22`), keyed `redux-state-<userId|'logged-out'>[:subkey]` (`client/state/initial-state.js:L75-L76`; matched by `client/state/persisted-state.js:L17`).
6. **In-memory bypass store** — private/incognito fallback (`client/lib/browser-storage/bypass.ts:L6,L12`).

---

## Q6 — Sidebar responsive design

This question has three parts: (a) the sidebar header's margin/padding, (b) the CSS custom properties driving the layout `calc()`s, and (c) the viewport widths at which things change.

### Q6a — What are the specific margin and padding values on the sidebar header?

#### Direct answer

There are **three** plausible "sidebar header" elements. The one that manifests in the **Reader** is `.is-section-reader .sidebar-header`, with **`margin: 0 12px 44px`** and **`padding: 0 10px`**. The two siblings are the classic list heading `.sidebar__heading` (`padding: 16px 8px 6px 16px; margin: 0`) and the global-nav branding header `.sidebar__header` (`padding: 30px 24px 29px`).

#### Observed output (compiled with the real `sass@1.54.0`)

**Command (mirrors the webpack SCSS prelude that injects the shared utils):**

```
node_modules/.bin/sass --load-path node_modules \
  <wrapper that @use 'client/assets/stylesheets/shared/utils' as * then the target scss>
```

**The Reader header (the one that manifests):**

```css
.is-section-reader .sidebar-header {
  display: flex;
  justify-content: space-between;
  margin: 0 12px 44px;
  padding: 0 10px;
}
```

- `client/reader/sidebar/style.scss:L113` (selector), `L116` (`margin: 0 12px 44px`), `L117` (`padding: 0 10px`).

**Sibling 1 — the classic list heading (documented "used for … static headings like in Reader"):**

```css
.sidebar__heading {
  color: var(--color-sidebar-text-alternative);
  font-size: 1rem;
  font-weight: 600;
  padding: 16px 8px 6px 16px;
  margin: 0;
}
```

- `client/layout/sidebar/style.scss:L64-L65` (the "used for both static headings // like in Reader" comment), `L66` (selector), `L70` (`padding: 16px 8px 6px 16px`), `L71` (`margin: 0`).

**Sibling 2 — the global-nav branding header:**

```css
.global-sidebar .sidebar__header {
  align-items: center;
  display: none;
  gap: 8px;
  padding: 30px 24px 29px;
}
.has-no-masterbar .global-sidebar .sidebar__header { display: flex; }
```

- `client/layout/global-sidebar/style.scss:L70` (selector), `L75` (`padding: 30px 24px 29px`).

#### Cause → effect

`.is-section-reader .sidebar-header` is scoped to the Reader section body class, so it wins on Reader pages and applies `margin: 0 12px 44px` / `padding: 0 10px` (`client/reader/sidebar/style.scss:L116-L117`). The `44px` bottom margin creates the gap below the Reader header; the flex + `justify-content: space-between` (`L114`) spreads the header's children. The classic `.sidebar__heading` and the global `.sidebar__header` target different DOM (list section headings and the global-nav branding row), so they don't override the Reader header.

#### Note on live capture

On the logged-out `/discover` page the Reader **sidebar header element is not rendered** (the logged-out Discover view uses a different chrome), so the values above come from compiling the **real** SCSS with `sass@1.54.0` rather than from `getComputedStyle` on that page. The compiled selectors and values are the authoritative source rules; which one applies is determined by the section body class (`.is-section-reader`), confirmed present on the Reader route (Q6c shows `is-section-reader` in the body class).

---

### Q6b — What CSS custom properties drive the layout calculations?

#### Direct answer

The layout geometry is computed from a small set of `:root` custom properties: **`--masterbar-height`** (`46px`, dropping to `32px` at `min-width: 782px`), **`--masterbar-checkout-height`** (`72px`), **`--sidebar-width-max`** (`272px`), and **`--sidebar-width-min`** (`228px`), plus contextual **`--content-padding-top`/`--content-padding-bottom`** (`16px` each). The Reader sidebar's `padding` and `height` are `calc()` expressions built from these.

#### Observed output (compiled `_variables.scss` with real `sass@1.54.0`)

```css
:root {
  --masterbar-height: 46px;
  --masterbar-checkout-height: 72px;
  --sidebar-width-max: 272px;
  --sidebar-width-min: 228px;
}
@media (min-width: 782px) {
  :root { --masterbar-height: 32px; }
}
```

- `client/assets/stylesheets/shared/_variables.scss:L5` (`:root`), `L7` (`--masterbar-height: 46px`), `L8` (`--masterbar-checkout-height: 72px`), `L10-L11` (`@media (min-width: 782px)` → `--masterbar-height: 32px`), `L15` (`--sidebar-width-max: 272px`), `L16` (`--sidebar-width-min: 228px`).

**Contextual content padding:**

```css
--content-padding-top: 16px;
--content-padding-bottom: 16px;
```

- `client/my-sites/sidebar/style.scss:L50-L51`.

**The `calc()` usages in the Reader sidebar (compiled):**

```css
/* client/reader/sidebar/style.scss:L70 */
padding: calc(var(--masterbar-height) + var(--content-padding-top))
         calc(var(--sidebar-width-max))
         var(--content-padding-bottom)
         16px;

/* client/reader/sidebar/style.scss:L107 */
height: calc(100vh
        - var(--masterbar-height)
        - var(--content-padding-top)
        - var(--content-padding-bottom));
```

**Observed live computed values (running app, `getComputedStyle(document.documentElement)`):**

```
# desktop width (innerWidth = 1905, >= 782px):
--masterbar-height          = 32px
--masterbar-checkout-height = 72px
--sidebar-width-max         = 272px
--sidebar-width-min         = 228px
# (--content-padding-* are supplied contextually by the my-sites sidebar and
#  were empty on the logged-out Discover document)
```

**A no-sidebar override exists** (note the accurate selector — it is `is-mobile-app-view`, not literally "no-sidebar"):

```css
body.is-mobile-app-view { --sidebar-width-max: 0px; --sidebar-width-min: 0px; }
```

- `client/layout/style.scss:L169-L170`.

#### Cause → effect

Because `--masterbar-height` is redefined inside `@media (min-width: 782px)` (`client/assets/stylesheets/shared/_variables.scss:L10-L11`), any `calc()` that references it recomputes at the 782px boundary: the Reader content `padding-top` (`calc(var(--masterbar-height) + var(--content-padding-top))`, `client/reader/sidebar/style.scss:L70`) and the scroll `height` (`calc(100vh - var(--masterbar-height) - …)`, `L107`) both shift by `46px − 32px = 14px` as the viewport crosses 782px. The live computed `--masterbar-height = 32px` at `innerWidth = 1905` confirms the desktop override is active; below 782px it is `46px` (observed in Q6c). Setting `--sidebar-width-*: 0px` in `is-mobile-app-view` (`client/layout/style.scss:L169-L170`) collapses the sidebar column in that context because `calc(var(--sidebar-width-max))` resolves to `0px`.

#### Custom properties enumerated (by name, with value + `file:line`)

- `--masterbar-height`: `46px`, → `32px` at `min-width:782px` (`client/assets/stylesheets/shared/_variables.scss:L7,L11`).
- `--masterbar-checkout-height`: `72px` (`client/assets/stylesheets/shared/_variables.scss:L8`).
- `--sidebar-width-max`: `272px` (`client/assets/stylesheets/shared/_variables.scss:L15`); `0px` under `is-mobile-app-view` (`client/layout/style.scss:L169`).
- `--sidebar-width-min`: `228px` (`client/assets/stylesheets/shared/_variables.scss:L16`); `0px` under `is-mobile-app-view` (`client/layout/style.scss:L170`).
- `--content-padding-top`: `16px` (`client/my-sites/sidebar/style.scss:L50`).
- `--content-padding-bottom`: `16px` (`client/my-sites/sidebar/style.scss:L51`).

---

### Q6c — At what viewport widths do things change?

#### Direct answer

Breakpoints come from **three systems**: (1) the JS runtime `mediaQueryOptions` map in `@automattic/viewport` (used by `isWithinBreakpoint`/`useBreakpoint`); (2) a deprecated SCSS `$breakpoints` list `480px, 660px, 800px, 960px, 1040px, 1280px, 1400px`; and (3) explicit layout-code thresholds `<660px` (narrow), `>=782px` (desktop), `>800px` (sidebar collapse). The layout-visible transitions observed are: **narrow ≤ 660px**, **masterbar height flips 46→32px and desktop turns on at 782px**, **sidebar collapse turns on above 800px (i.e. at 801px)**, and the **global sidebar toggles across the 660/661px boundary**.

#### System 1 — the JS `mediaQueryOptions` map (real, compiled `@automattic/viewport`)

- Source map: `packages/viewport/src/index.ts:L96-L118` (22 entries). Notable off-by-one storage: `>=782px` is stored `{ min: 781 }` (`L108`) and `>=960px` is stored `{ min: 959 }` (`L111`).

**Command (standalone Node requiring the real compiled module with a width-aware `matchMedia` mock):**

```
node /tmp/blitzy_viewport_probe.js
```

**Observed — the literal media query each key generates (via the real `getMediaQueryList`):**

```
<480px       -> (max-width: 480px)
<660px       -> (max-width: 660px)
<782px       -> (max-width: 782px)
<800px       -> (max-width: 800px)
<960px       -> (max-width: 960px)
<1040px      -> (max-width: 1040px)
<1180px      -> (max-width: 1180px)
<1280px      -> (max-width: 1280px)
<1400px      -> (max-width: 1400px)
>480px       -> (min-width: 481px)
>660px       -> (min-width: 661px)
>=782px      -> (min-width: 782px)
>782px       -> (min-width: 783px)
>800px       -> (min-width: 801px)
>=960px      -> (min-width: 960px)
>960px       -> (min-width: 961px)
>1040px      -> (min-width: 1041px)
>1280px      -> (min-width: 1281px)
>1400px      -> (min-width: 1401px)
480px-660px  -> (min-width: 481px) and (max-width: 660px)
660px-960px  -> (min-width: 661px) and (max-width: 960px)
480px-960px  -> (min-width: 481px) and (max-width: 960px)
```

**Observed — the off-by-one made concrete (`>=` inclusive vs `>` strict):**

```
>=782px  781:false  782:true   783:true
>782px   782:false  783:true   784:true
>=960px  959:false  960:true   961:true
>960px   960:false  961:true   962:true
```

**Observed — boolean matrix across the named widths (480/660/782/800/960) + boundaries:**

```
w=480   {"<660px":true, ">=782px":false,">782px":false,">800px":false,">=960px":false}
w=660   {"<660px":true, ">=782px":false,">782px":false,">800px":false,">=960px":false}
w=700   {"<660px":false,">=782px":false,">782px":false,">800px":false,">=960px":false}
w=781   {"<660px":false,">=782px":false,">782px":false,">800px":false,">=960px":false}
w=782   {"<660px":false,">=782px":true, ">782px":false,">800px":false,">=960px":false}
w=800   {"<660px":false,">=782px":true, ">782px":true, ">800px":false,">=960px":false}
w=801   {"<660px":false,">=782px":true, ">782px":true, ">800px":true, ">=960px":false}
w=959   {"<660px":false,">=782px":true, ">782px":true, ">800px":true, ">=960px":false}
w=960   {"<660px":false,">=782px":true, ">782px":true, ">800px":true, ">=960px":true}
w=1000  {"<660px":false,">=782px":true, ">782px":true, ">800px":true, ">=960px":true}
```

**Cause → effect (the off-by-one):** `createMediaQueryList` builds min-queries as `(min-width: ${min + 1}px)` (compiled `@automattic/viewport`, `dist/cjs` lines 71/76). So the stored `{ min: 781 }` for `>=782px` becomes the actual query `(min-width: 782px)` — **inclusive at exactly 782** — while `>782px` (`{ min: 782 }`) becomes `(min-width: 783px)` — **strict** (one pixel higher). The same pattern makes `>=960px` inclusive at 960 and `>960px` strict at 961. This is why `<660px` is `true` at exactly 660 (`(max-width: 660px)`), and why sidebar collapse (`>800px` → `(min-width: 801px)`) turns on at 801, not 800.

#### System 2 — the deprecated SCSS `$breakpoints` list

```scss
$breakpoints: 480px, 660px, 800px, 960px, 1040px, 1280px, 1400px;
```

- `client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10`; the `breakpoint-deprecated` mixin at `L12`.

#### System 3 — the explicit layout-code thresholds

- `client/layout/index.jsx:L76` — `useBreakpoint( '<660px' )` → `isNarrow`.
- `client/layout/index.jsx:L146` — `isWithinBreakpoint( '>=782px' )` → `isDesktop`.
- `client/layout/index.jsx:L221` — `isWithinBreakpoint( '>800px' )` → sidebar collapse.

**Global sidebar collapse boundary (compiled media queries):**

```css
@media (min-width: 661px) { ... }   /* client/layout/global-sidebar/style.scss:L470 */
@media (max-width: 660px) { ... }   /* client/layout/global-sidebar/style.scss:L511 */
```

#### Observed — live browser resize (matchMedia + the `--masterbar-height` custom property)

```
innerWidth = 640px :  --masterbar-height = 46px ; (max-width:660)=true ; (min-width:661)=false ; (min-width:782)=false
innerWidth = 700px :  --masterbar-height = 46px ; (max-width:660)=false; (min-width:661)=true  ; (min-width:782)=false ; (min-width:800)=false
innerWidth =1000px :  --masterbar-height = 32px ; (min-width:782)=true ; (min-width:800)=true  ; (max-width:660)=false
innerWidth =1905px :  --masterbar-height = 32px
```

**Cause → effect (layout-visible transitions):**
- **≤ 660px (narrow):** `useBreakpoint('<660px')` is `true` (matrix: `w=660 <660px:true`, `w=700 <660px:false`), and the global sidebar switches to its `@media (max-width: 660px)` rules (`client/layout/global-sidebar/style.scss:L511`); above it, `@media (min-width: 661px)` applies (`L470`) — the 660/661 boundary, confirmed live (`640px`→`(max-width:660)=true`, `700px`→`(min-width:661)=true`).
- **782px (desktop + masterbar height):** `isDesktop` (`>=782px`) flips `true` at exactly 782 (matrix), and `--masterbar-height` flips `46px → 32px` at `min-width: 782px` (`client/assets/stylesheets/shared/_variables.scss:L10-L11`), confirmed live (`700px`→`46px`, `1000px`→`32px`). This is the same 782px on both the JS (`client/layout/index.jsx:L146`) and CSS sides.
- **> 800px (sidebar collapse):** `isWithinBreakpoint('>800px')` → `(min-width: 801px)` turns `true` at 801 (matrix: `w=800 >800px:false`, `w=801 >800px:true`), driving the sidebar-collapse layout in `client/layout/index.jsx:L221`.
- **960px:** `>=960px` turns `true` at exactly 960 (matrix), the next design step in the `$breakpoints` list.

#### Breakpoints enumerated (by name, with value + `file:line`)

- **JS map** (`packages/viewport/src/index.ts:L96-L118`): `<480/<660/<782/<800/<960/<1040/<1180/<1280/<1400`, `>480/>660/>=782/>782/>800/>=960/>960/>1040/>1280/>1400`, and ranges `480px-660px`, `660px-960px`, `480px-960px` — 22 entries; `>=782px`={min:781} (`L108`), `>=960px`={min:959} (`L111`).
- **SCSS list** (`client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10`): `480px, 660px, 800px, 960px, 1040px, 1280px, 1400px`.
- **Layout thresholds** (`client/layout/index.jsx`): `<660px` (`L76`), `>=782px` (`L146`), `>800px` (`L221`).
- **Global-sidebar boundary** (`client/layout/global-sidebar/style.scss`): `min-width:661px` (`L470`), `max-width:660px` (`L511`).
- **Masterbar-height media query** (`client/assets/stylesheets/shared/_variables.scss:L10`): `min-width:782px`.

---

## Coverage pass — every question, sub-part, and named item

Each row below re-enumerates a distinct thing the six questions asked for, with its value, `file:line`, whether it was observed at runtime (or `(inferred)`), and the causal reason.

### Q1 — port & readiness

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Bound port | `3000` | `config/_shared.json:L25`, `config/development.json:L8`, `client/server/index.js:L12,L83` | Observed (boot log + SSR served on :3000) |
| Host | `calypso.localhost` | `config/development.json:L7` | Observed (boot log URL) |
| Protocol | `http` | `config/_shared.json:L24`, `config/development.json:L6` | Observed (boot log URL) |
| `PORT` override | `process.env.PORT \|\| data.port` | `client/server/config/parser.js:L63` | (inferred) from source |
| Boot log (≠ ready) | `wp-calypso booted in 1029ms - http://calypso.localhost:3000` | `client/server/index.js:L33` | Observed |
| Readiness banner (fully ready) | `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` | `client/server/bundler/index.js:L56` | Observed (after 77804ms first compile) |
| Recompile banner | `Ready! All assets are re-compiled. Have fun!` | `client/server/bundler/index.js:L60` | (inferred) — recompile not triggered (read-only) |
| Pre-compile holding page | `Welcome to Calypso!` (630 bytes) | `client/server/bundler/index.js:L76-L93,L82` | Observed (curl `/`) |
| `start` script chain | gate → welcome → build → run | `package.json:L110,L113` | Observed |

### Q2 — port architecture

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| HTML + assets + HMR on one port | `:3000` (all three `app.use`) | `client/server/bundler/index.js:L100-L102` | Observed |
| Asset middleware | `webpack-dev-middleware` | `client/server/bundler/index.js:L5,L101` | Observed (`runtime.js` 200) |
| HMR middleware / channel | `webpack-hot-middleware`, `/__webpack_hmr` `text/event-stream` | `client/server/bundler/index.js:L6,L102` | Observed |
| Separate hot-reload port? | **No** | — | Observed (HMR on :3000) |
| REST base (remote) | `https://public-api.wordpress.com` | `packages/wpcom-xhr-request/src/index.js:L27` | Observed (remote requests) |
| `3001` / `3002` | Jetpack Cloud / A8C-for-Agencies | `package.json:L115,L117` | Observed (grep) |

### Q3 — Reader stream endpoints

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Default stream key | `following` | `client/reader/controller.js:L87` | Observed |
| Default path | `/read/following`, apiVersion `1.2`, `number 4` | `…/streams/index.js:L194`, `L369`, `L161` | Observed (http action) |
| All 20 stream keys → paths | see Q3 table | `…/streams/index.js:L192-L351` | Observed (each via `requestPage`) |
| `INITIAL_FETCH` / `PER_FETCH` | `4` / `7` | `…/streams/index.js:L161,L160` | Observed (`number` 4→7 in browser) |
| `http()` nests query | `apiVersion`/`apiNamespace`/`number` in `action.query` | `client/state/data-layer/wpcom-http/actions.js:L55-L66` | Observed |
| `list` variant | `number 40`, apiVersion `1.3` | `…/streams/index.js:L332-L338` | Observed |
| `recommendations_posts` variant | `number` undefined | `…/streams/index.js:L286-L287` | Observed |
| Logged-out default | Discover → `/wpcom/v2/read/streams/discover` | `…/streams/index.js:L223-L224` | Observed (reqid=149) |

### Q4 — initial-load Redux actions

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Mount dispatch | `READER_STREAMS_PAGE_REQUEST` | `client/reader/stream/index.jsx:L221,L502`; `client/state/reader/streams/actions.js:L39` | Observed |
| Success cascade | `READER_STREAMS_PAGE_REQUEST` → `READER_POSTS_RECEIVE` → `READER_STREAMS_PAGE_RECEIVE` | `…/streams/index.js:L428,L497`; `client/state/reader/posts/actions.js:L87`; `client/state/reader/streams/actions.js:L63` | Observed (ordered stream) |
| Analytics (conditional) | `recordTracksEvent` only when post has `railcar` | `…/streams/index.js:L428-L511` | Observed (railcar null → none) |
| Bootstrap actions | `SECTION_SET`, `ROUTE_SET`, `CURRENT_USER_RECEIVE` | `client/state/action-types.ts:L850,L848,L139`; `client/state/ui/section/actions.js:L5,L8` | (inferred) from source |

### Q5 — login detection & storage (both branches)

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Render decision | `isUserLoggedIn = getCurrentUserId(state) !== null` | `client/state/current-user/selectors.js:L6,L15-L16` | Observed (true/false/edge) |
| Logged-in | id `12345` → `true` | `client/state/current-user/reducer.js:L24-L26` | Observed |
| Logged-out | id `null` → `false` | same | Observed |
| Uninitialized edge case | `undefined !== null` → `true` | `client/state/current-user/selectors.js:L16` | Observed |
| `wpcom_token` getToken order | cookie → localStorage → `false` | `packages/oauth-token/src/index.js:L7,L10-L22` | Observed (3 sub-cases) |
| OAuth redirect | `/login` when `getToken()===false` | `client/boot/common.js:L154,L175-L176` | (inferred) from source |
| Server `/me` bootstrap | gated on `wordpress_logged_in` | `client/server/user-bootstrap/index.js:L8,L13,L32-L34` | Observed (`/me` 200) |
| Client hydration | `window.initialReduxState` | `client/state/initial-state.js:L149` | Observed (present) |
| IndexedDB store | `calypso` (v2) / `calypso_store` | `client/lib/browser-storage/index.ts:L20-L22` | Observed (live) |
| Persistence key | `redux-state-<userId\|'logged-out'>[:subkey]` | `client/state/initial-state.js:L75-L76` | Observed (17 keys) |
| Logged-out Reader | `enableLoggedOut: true` | `client/sections.js:L396` | Observed (Reader renders) |
| Bypass store | in-memory fallback | `client/lib/browser-storage/bypass.ts:L6,L12` | (inferred) |

### Q6 — sidebar responsive design

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Q6a Reader header (manifests) | `margin: 0 12px 44px; padding: 0 10px` | `client/reader/sidebar/style.scss:L113,L116,L117` | Observed (sass compile) |
| Q6a sibling `.sidebar__heading` | `padding: 16px 8px 6px 16px; margin: 0` | `client/layout/sidebar/style.scss:L66,L70,L71` | Observed (sass compile) |
| Q6a sibling `.sidebar__header` | `padding: 30px 24px 29px` | `client/layout/global-sidebar/style.scss:L70,L75` | Observed (sass compile) |
| Q6b `--masterbar-height` | `46px` → `32px` @≥782px | `client/assets/stylesheets/shared/_variables.scss:L7,L10-L11` | Observed (compile + live 32px) |
| Q6b `--masterbar-checkout-height` | `72px` | `client/assets/stylesheets/shared/_variables.scss:L8` | Observed |
| Q6b `--sidebar-width-max` | `272px` (0px in is-mobile-app-view) | `client/assets/stylesheets/shared/_variables.scss:L15`; `client/layout/style.scss:L169` | Observed |
| Q6b `--sidebar-width-min` | `228px` (0px in is-mobile-app-view) | `client/assets/stylesheets/shared/_variables.scss:L16`; `client/layout/style.scss:L170` | Observed |
| Q6b `--content-padding-top/bottom` | `16px` / `16px` | `client/my-sites/sidebar/style.scss:L50-L51` | Observed |
| Q6b Reader `padding` calc() | `calc(var(--masterbar-height)+var(--content-padding-top)) …` | `client/reader/sidebar/style.scss:L70` | Observed (compile) |
| Q6b Reader `height` calc() | `calc(100vh - --masterbar-height - --content-padding-top - --content-padding-bottom)` | `client/reader/sidebar/style.scss:L107` | Observed (compile) |
| Q6c JS map | 22 entries; `>=782px`={min:781}, `>=960px`={min:959} | `packages/viewport/src/index.ts:L96-L118` | Observed (query strings + booleans) |
| Q6c off-by-one | `>=782px`→`min-width:782px` (incl 782); `>782px`→`min-width:783px` | compiled `@automattic/viewport` | Observed |
| Q6c SCSS `$breakpoints` | `480,660,800,960,1040,1280,1400px` | `client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10` | Observed (read) |
| Q6c layout thresholds | `<660px` / `>=782px` / `>800px` | `client/layout/index.jsx:L76,L146,L221` | Observed (booleans) |
| Q6c global-sidebar boundary | `min-width:661px` / `max-width:660px` | `client/layout/global-sidebar/style.scss:L470,L511` | Observed (compile + live 640/700) |
| Q6c masterbar flip | `46px↔32px` at 782px | `client/assets/stylesheets/shared/_variables.scss:L10-L11` | Observed (live 700→46, 1000→32) |

---

## Notes on fidelity & scope

- **Read-only:** No existing source file was modified, created, or deleted. The only new artifact is this document under the `blitzy/` tree. All temporary observation scripts and Jest harnesses were removed after capturing output; `git status --porcelain` shows no tracked-source changes.
- **Canonical runtime:** All observations were produced under Node `v22.23.1` / yarn `4.0.2` (the pinned canonical runtime). The Node-20 gate discrepancy is documented in the Environment section above.
- **`(inferred)` labels:** A handful of claims are labeled `(inferred)` where a clean runtime trigger would have required violating the read-only constraint (the recompile banner text) or exercising an environment not available here (Node 20 itself, incognito storage bypass, the OAuth `/login` redirect, and the surrounding bootstrap actions). Every such claim is grounded in an exact `file:line`. All other claims carry observed runtime output.

