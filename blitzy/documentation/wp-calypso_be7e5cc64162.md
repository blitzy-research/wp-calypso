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

Runtime harnesses used (all kept under `/tmp/blitzy_evidence/`, outside the repo, and removed after capture):
- **Dev server** run under `SECTION_LIMIT=reader,login` (two runs) — for the boot log, readiness banner, pre-compile holding page, SSR HTML, `runtime.js`, and the `/__webpack_hmr` stream (Q1/Q2).
- **Live Chrome DevTools** inspection of the running Reader — for the live network requests to `public-api.wordpress.com` (Q3), the cookie / `localStorage` / IndexedDB reads and the client `/me` call (Q5), and the `matchMedia`/resize probes plus the **injected-DOM real-CSS** `getComputedStyle` of `.is-section-reader .sidebar-header` and the live `:root` custom-property reads (Q6).
- **Boot-time Redux action capture** — a fake `__REDUX_DEVTOOLS_EXTENSION__` hook installed **before** boot recorded the ordered 378-action dispatch stream, giving the real initial-load action order (Q4).
- **Standalone Node harnesses** — one running the **verbatim** `isUserLoggedIn`/`getCurrentUserId` selector and `id` reducer code, one running the **verbatim** `getToken`/`setToken` code against the real `cookie` package, and one that `require`s the **real compiled** `@automattic/viewport` module with a width-aware `matchMedia` mock (Q5/Q6c). These *replicate* code (they do not import the real modules, which use internal `calypso/*` aliases that a standalone script cannot resolve) and are labeled **Synthetic-harness**.
- **Source enumeration** where a value cannot be observed at runtime: the 20-entry `streamApis` map is a non-exported module-local `const`, and the read-only constraint forbids adding a Jest file, so the full stream-key→path table is enumerated exhaustively from source (with the default stream confirmed live); the logged-in render branch could not be driven without WordPress.com credentials and is labeled `(inferred)`.

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

The environment's default setup installs Node 20.x, but the canonical runtime pinned by the repo is Node 22.x (`.nvmrc:L1` = `22.9.0`; `engines.node = "^v22.9.0"` at `package.json:L57`). Node 20 was **not** installed in the canonical environment (the setup explicitly warns against downgrading), so the failing case is demonstrated two honest, observed ways: (a) the exact mismatch output `check-node-version` prints when the running Node does not satisfy a wanted range, and (b) the actual gate decision computed with the repo's own `semver` against the literal `engines.node` string.

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
semver.validRange(literal)  = ">=22.9.0 <23.0.0-0"
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
- `client/server/index.js:L83-L86` — `server.listen()` binds the HTTP server with `{ port, host: process.env.CALYPSO_IS_FORK ? host : null }` (`L83`); its callback calls `sendBootStatus( 'ready' )` (`L85`).
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
{"name":"calypso","hostname":"reverse-code-generator-782dced0-d7k7f","pid":71802,"level":30,"msg":"wp-calypso booted in 1005ms - http://calypso.localhost:3000","time":"2026-07-08T06:06:27.230Z","v":0}
[sections-loader] Limiting build to reader, login sections
Compiling assets... Wait until you see Ready! and then try http://calypso.localhost:3000/ again.
```

- The boot line is emitted by `client/server/index.js:L33` (`wp-calypso booted in %dms - %s://%s:%s`).
- `[sections-loader] Limiting build to reader, login sections` confirms the `SECTION_LIMIT` build.

**The first webpack compile finishing (run 1, ~69 seconds later):**

```
webpack built f28b7fd1267ece85e128 in 68652ms
webpack 5.97.1 compiled with 11 warnings in 68652 ms
```

**Then — and only then — the readiness banner (this is "fully ready"):**

```
Ready! You can load http://calypso.localhost:3000/ now. Have fun!
```

**Timing stability across two runs (R3).** The same unchanged command was executed twice. The readiness **signal** is invariant: the cyan banner text `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` was byte-identical in both runs, and both compiles finished with exactly `11 warnings`. The elapsed **milliseconds** are *not* a stable value — they vary run-to-run with machine load — so they are reported as a distribution rather than a single figure:

| Run | "booted in" (Express listen) | webpack first-compile (ms) | build content-hash |
|-----|------------------------------|-----------------------------|--------------------|
| 1   | `1005ms`                     | `68652ms`                   | `f28b7fd1267ece85e128` |
| 2   | `1021ms`                     | `74062ms`                   | `c27eacc081a88d695668` |

Boot ≈ 1 s (1005–1021 ms); first compile ≈ 69–74 s. The build content-hash differs each run (it embeds module order and timestamps), which is expected and has no bearing on the readiness signal.

- The banner is printed on webpack's `done` hook: `client/server/bundler/index.js:L38` (`compiler.hooks.done.tap( 'Calypso', fn )`), text at `client/server/bundler/index.js:L56`, wrapped in `chalk.cyan` (`client/server/bundler/index.js:L54-L59`).
- On subsequent recompiles the banner text is instead `Ready! All assets are re-compiled. Have fun!` (`client/server/bundler/index.js:L60`). *(inferred — a recompile was not triggered at runtime because doing so cleanly would require touching a tracked source file, which the read-only constraint forbids; the text is quoted from source.)*

**The pre-compile holding page (requesting `/` before the first compile finishes).**

Command:

```
$ curl -s -i http://calypso.localhost:3000/
```

Response headers reported `HTTP/1.1 200 OK` with `Content-Length: 630`. The complete, unedited response body (630 bytes) is:

```html
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
```

- Served by `client/server/bundler/index.js:L76-L93` (the `waitForCompiler` middleware, `function waitForCompiler` at `client/server/bundler/index.js:L66`). The `<meta http-equiv="refresh" content="5">` auto-retry is emitted at `client/server/bundler/index.js:L79`, the `Welcome to Calypso!` heading at `client/server/bundler/index.js:L82`, and the allmoji `<img>` at `client/server/bundler/index.js:L90`. The `content="5"` refresh is what makes the page reload itself every 5 seconds until the bundle is ready.

### Cause → effect

`server.listen` binds port `3000` (`client/server/index.js:L83`) almost immediately, which is why the "booted in 1005ms" line (run 1) appears within ~1 s. But the client bundles are compiled *at runtime* by `webpack-dev-middleware`; until webpack's first `done` hook fires (~69 s in run 1, ~74 s in run 2), `waitForCompiler` intercepts `/` and returns the holding page. The cyan `Ready!` banner is emitted from that same `done` hook, so it is the accurate "fully ready" signal — the ~68–73 s gap between "booted" and "Ready!" is exactly the first webpack compile.

### Sibling variants

- **Booted log vs readiness banner:** two distinct signals; "booted" ≠ "ready" (a ~68–73 s gap across the two runs).
- **First compile vs recompile banner:** `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` (first) vs `Ready! All assets are re-compiled. Have fun!` (recompile) — `client/server/bundler/index.js:L56` vs `L60`.
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
  -> HTTP 200, Content-Type: application/javascript; charset=utf-8, Content-Length: 75033
```

**The HMR channel — served by webpack-hot-middleware on the SAME :3000:**

```
GET http://calypso.localhost:3000/__webpack_hmr
  -> HTTP 200
  -> Content-Type: text/event-stream;charset=utf-8
  -> Cache-Control: no-cache, no-transform
  -> Connection: keep-alive
```

The HMR channel is a long-lived Server-Sent-Events stream. Captured with `curl -sN --max-time 3 http://calypso.localhost:3000/__webpack_hmr`, its first event is (complete relevant fields, verbatim):

```
data: {"name":"","action":"sync","time":68652,"hash":"f28b7fd1267ece85e128","warnings":[],"errors":[],"modules":{
[the "modules" map and subsequent keep-alive events continue beyond the 200-byte capture window]
```

The event's `hash` (`f28b7fd1267ece85e128`) and `time` (`68652`) are **identical** to the run-1 boot-log build line `webpack built f28b7fd1267ece85e128 in 68652ms` — proving the HMR event-stream and the SPA bundles are produced by the **same** webpack instance served on `:3000`, not a separate hot-reload port.

**Data/REST — remote, not a local port** (from the SSR HTML and the live Network tab):

```
# the SSR HTML prefetches the remote proxy origin (complete, unedited <link>,
# and the ONLY reference to public-api.wordpress.com in the SSR <head>):
<link rel="prefetch" as="document" href="https://public-api.wordpress.com/wp-admin/rest-proxy/?v=2.0"/>
# and every data request in the running app targets public-api.wordpress.com
# (e.g. the Reader stream call and /me — see Q3/Q5), never a local port.
```

### The `3001` / `3002` ports are different apps (not the main Calypso)

```
package.json:L115  "start-jetpack-cloud-p": "PORT=3001 CALYPSO_ENV=jetpack-cloud-development yarn run build-server && PORT=3001 CALYPSO_ENV=jetpack-cloud-development yarn run start-build",
package.json:L117  "start-a8c-for-agencies-p": "PORT=3002 CALYPSO_ENV=a8c-for-agencies-development yarn run build-server && PORT=3002 CALYPSO_ENV=a8c-for-agencies-development yarn run start-build",
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
- `client/state/data-layer/wpcom/read/streams/index.js:L358` — `requestPage`; default `apiVersion = '1.2'` at `L370`; `fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH` at `L380`. The `http` GET is built at `L395-L405` (complete, verbatim — no elision):

```js
	return http( {
		method: 'GET',
		path: path( { ...action.payload } ),
		apiVersion,
		apiNamespace: api.apiNamespace ?? null,
		query: isPoll
			? pollQuery( [], commonQueryParams )
			: query( { ...commonQueryParams, ...pageHandle, number, lang, page }, action.payload ),
		onSuccess: action,
		onFailure: action,
	} );
```
- `client/state/data-layer/wpcom-http/actions.js:L43,L60` — `http()` first computes `const version = apiNamespace ? { apiNamespace } : { apiVersion }` (`L43`), then merges it into the query object: `query: { ...query, ...version }` (`L60`). **This is why `apiVersion`/`apiNamespace` and `number` live *inside* `action.query`.**
- Default stream key `'following'`: `client/reader/controller.js:L53` (`mcKey = 'following'`), `L85` (`key`), `L87` (`streamKey`).

### Stream-key → path enumeration (exhaustive, from the `streamApis` source map)

**Methodology.** `streamApis` is a **module-local `const`** — it is *not* exported — declared at `client/state/data-layer/wpcom/read/streams/index.js:L192`, and the module imports internal `calypso/*` path aliases, so it cannot be `import`ed by a standalone probe; the read-only constraint (Section 0.3.2 of the AAP) also forbids adding a Jest test file to the repository. The complete key→path mapping is therefore enumerated **exhaustively from the source object literal** (`client/state/data-layer/wpcom/read/streams/index.js:L192-L351`), with each row cited to its exact `file:line`; the **default stream actually exercised at runtime is confirmed by the live browser network capture** shown below (logged-out Discover, `number=4`). Each resolved path is what that entry's `path()` function returns; the `number` / `apiVersion` / `apiNamespace` columns are the values `requestPage` (`client/state/data-layer/wpcom/read/streams/index.js:L358-L406`) attaches.

**Full enumeration (method `GET`; `number = INITIAL_FETCH = 4` on the first page unless noted; every `file:line` below is in `client/state/data-layer/wpcom/read/streams/index.js`):**

| Stream key | Resolved path | apiVersion / apiNamespace | `number` | key→path `file:line` |
|---|---|---|---|---|
| `following` | `/read/following` | apiVersion `1.2` | 4 | L193→L194 |
| `recent` | `/read/streams/following` | apiNamespace `wpcom/v2` | 4 | L197→L198 |
| `search` | `/read/search` | apiVersion `1.2` | 4 | L211→L212 |
| `feed` | `/read/feed/<feedId>/posts` (e.g. `/read/feed/12345/posts`) | apiVersion `1.2` | 4 | L219→L220 |
| `discover` (recommended) | `/read/streams/discover` | apiNamespace `wpcom/v2` | 4 | L223→L226 |
| `discover` (latest) | `/read/tags/posts` | apiNamespace `wpcom/v2` | 4 | L223→L228 |
| `discover` (firstposts) | `/read/streams/first-posts` | apiNamespace `wpcom/v2` | 4 | L223→L230 |
| `discover` (other) | `/read/streams/discover?tags=<suffix>` | apiNamespace `wpcom/v2` | 4 | L223→L232 |
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
| `list` | `/read/list/<owner>/<slug>/posts` (e.g. `/read/list/bob/mylist/posts`) | apiVersion `1.3` | **40** | L332→L335 |
| `user` | `/users/<userId>/posts` (e.g. `/users/42/posts`) | apiVersion `1` | 4 | L346→L347 |

**The default `following` `http` action (inferred from source — see note).** A **live logged-in session could not be run** — no WordPress.com credentials were available in the environment — so the logged-in `following` request was **not** observed at runtime (logged-out visitors are redirected to Discover instead; see the live capture below). The action shown is the one `requestPage` **constructs from source** for `streamType='following'`: `method`/`path`/`apiVersion`/`apiNamespace` from the `http()` builder (`client/state/data-layer/wpcom/read/streams/index.js:L395-L405`, reproduced verbatim in the Q3 citations above); the `query` from the default `getQueryString` (`client/state/data-layer/wpcom/read/streams/index.js:L166-L168` → `orderBy:'date'`, `content_width:675`) with `meta` = `QUERY_META` (`client/state/data-layer/wpcom/read/streams/index.js:L165` → `'post,discover_original_post'`), `number=4` = `INITIAL_FETCH` (`client/state/data-layer/wpcom/read/streams/index.js:L161`), and default `apiVersion='1.2'` (`client/state/data-layer/wpcom/read/streams/index.js:L370`):

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
```

The response is an `_envelope=1` wrapper. Its structure is reproduced verbatim below; volatile post content (bodies, image URLs, IDs) and the opaque pagination cursor are redacted, but the shape, card count, and card-type order are exact:

```
top-level keys : ["body","status","headers"]        (envelope status: 200)
body keys      : ["cards","next_page_handle","user_interests"]
body.cards     : 9 cards; type order (exact) =
                 ["post","recommended_blogs","post","post","post","post","post","post","interests_you_may_like"]
first "post" card -> "data" object has 47 keys; the first 25 are:
                 ["ID","site_ID","author","date","modified","title","URL","short_URL",
                  "content","excerpt","slug","guid","status","has_password","discussion",
                  "likes_enabled","sharing_enabled","like_count","i_like","is_reblogged",
                  "is_following","global_ID","featured_image","post_thumbnail","format"]
"recommended_blogs" card -> "data": list of 8 recommended blogs
"interests_you_may_like" card -> "data": interest tags
body.next_page_handle : "[REDACTED:pagination-cursor]"   (360-char opaque cursor string)
body.user_interests   : list (2 entries)

reqid=168  (next page) GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover
           with page_handle=[REDACTED:pagination-cursor] & number=7   -> HTTP 200
```

- `number=4` on the first request confirms `INITIAL_FETCH = 4`; `number=7` on the next page confirms `PER_FETCH = 7`.
- The request targets the **remote** `public-api.wordpress.com` under the `wpcom/v2` namespace — corroborating Q2 (data is remote, not a local port) and the `apiNamespace 'wpcom/v2'` for the Discover stream.

### Cause → effect

`requestPage` looks up `streamApis[streamKey]`, calls that entry's `path()` function to build the REST path, and merges its query via `http()` (`client/state/data-layer/wpcom-http/actions.js:L55-L66`), which nests `apiVersion`/`apiNamespace` and the computed `number` inside `action.query`. On the first page `pageHandle` is empty, so `fetchCount = INITIAL_FETCH = 4` (`client/state/data-layer/wpcom/read/streams/index.js:L380`); on later pages it becomes `PER_FETCH = 7` — exactly matching the observed `number=4` then `number=7`.

### Sibling variants (exhaustive)

- **`discover` is computed** — four branches (recommended / latest / firstposts / other) resolve to different paths, all under `apiNamespace wpcom/v2` (`client/state/data-layer/wpcom/read/streams/index.js:L224-L233`; apiNamespace at `L246`).
- **`list` is computed** and forces `number: 40` and `apiVersion 1.3` (`client/state/data-layer/wpcom/read/streams/index.js:L332-L338`) — a deliberate departure from `INITIAL_FETCH`.
- **`recommendations_posts`** does **not** forward `number` (its query fn at `client/state/data-layer/wpcom/read/streams/index.js:L289-L291` is `( { query } ) => ({ ...query, seed, algorithm })`), so `number` is `undefined` on its request — a real per-key difference.
- **`user`** uses `apiVersion 1` and **`recent`/`tag`/`tag_popular`/`discover`** use `apiNamespace wpcom/v2` instead of a numeric `apiVersion`.
- **Default landing differs by auth:** logged-in → `following` → `/read/following`; logged-out → Discover → `/wpcom/v2/read/streams/discover`.

---

## Q4 — What Redux actions fire during the Reader's initial load?

### Direct answer

During boot the app dispatches (in order) `SECTION_SET` then `ROUTE_SET`; then, on the Reader stream component's mount, **`READER_STREAMS_PAGE_REQUEST`**. The data-layer intercepts that request, issues the `http` GET, and on success runs `handlePage`, which dispatches — for each non-external post — a `POST_LIKES_RECEIVE`, then **`READER_POSTS_RECEIVE`**, then (for Discover's recommended blogs) `READER_RECOMMENDED_SITES_RECEIVE`, and finally **`READER_STREAMS_PAGE_RECEIVE`**. The **observed** boot-to-stream sequence (logged-out `/discover`, real browser session) is:

```
SECTION_SET  →  ROUTE_SET  →  READER_STREAMS_PAGE_REQUEST
             →  POST_LIKES_RECEIVE ×7  →  READER_POSTS_RECEIVE
             →  READER_RECOMMENDED_SITES_RECEIVE  →  READER_STREAMS_PAGE_RECEIVE
```

`CURRENT_USER_RECEIVE` is the **logged-in-branch** bootstrap action: it did **not** fire in the observed logged-out session (there is no user to receive), and with no credentials available it could not be observed live either — see the note under *Observed output*. All other actions above are runtime-observed and in the exact order captured.

### Where the dispatch chain lives

- Mount → fetch: `client/reader/stream/index.jsx:L221` (`componentDidMount`), `L225` (`this.fetchNextPage( {} )`), `L489` (`fetchNextPage`), `L502` (`props.requestPage( { feedId: selectedFeedId, streamKey, pageHandle, localeSlug } )`); `requestPage` imported at `client/reader/stream/index.jsx:L39`.
- Action creators: `client/state/reader/streams/actions.js:L28` (`requestPage`) → type `READER_STREAMS_PAGE_REQUEST` at `L39`; `L52` (`receivePage`) → type `READER_STREAMS_PAGE_RECEIVE` at `L63`.
- Constants: the initial load uses `READER_STREAMS_PAGE_REQUEST` (`client/state/reader/action-types.ts:L78`) and `READER_STREAMS_PAGE_RECEIVE` (`client/state/reader/action-types.ts:L77`), plus `READER_POSTS_RECEIVE` (`client/state/reader/action-types.ts:L52`). For completeness, the **full `READER_STREAMS_*` block** (`client/state/reader/action-types.ts:L76-L86`) and which members fire on initial load:

  | Constant | Line | Fires on initial load? |
  |---|---|---|
  | `READER_STREAMS_CLEAR` | L76 | no — stream reset |
  | `READER_STREAMS_PAGE_RECEIVE` | L77 | **yes** — success cascade |
  | `READER_STREAMS_PAGE_REQUEST` | L78 | **yes** — mount fetch |
  | `READER_STREAMS_PAGINATED_REQUEST` | L79 | no — explicit paginated-request path |
  | `READER_STREAMS_REMOVE_ITEM` | L80 | no — item removal |
  | `READER_STREAMS_SELECT_ITEM` | L81 | no — keyboard/selection |
  | `READER_STREAMS_SELECT_NEXT_ITEM` | L82 | no — keyboard/selection |
  | `READER_STREAMS_SELECT_PREV_ITEM` | L83 | no — keyboard/selection |
  | `READER_STREAMS_SHOW_UPDATES` | L84 | no — live-updates banner |
  | `READER_STREAMS_UPDATES_RECEIVE` | L85 | no — polling updates |
  | `READER_STREAMS_NEW_POST_RECEIVE` | L86 | no — new-post insertion |

  Only `READER_STREAMS_PAGE_REQUEST` (L78) and `READER_STREAMS_PAGE_RECEIVE` (L77) participate in the initial-load cascade; the other nine are siblings driven by pagination, keyboard selection, live updates, or item mutation.
- Data-layer cascade: `client/state/data-layer/wpcom/read/streams/index.js:L428` (`handlePage`) collects actions and dispatches them — `receivePosts( streamPosts )` (`L470`), `receiveRecommendedSites` (`L474`, `L479`), and `receivePage` (`L497`); wired via `dispatchRequest( { fetch: requestPage, onSuccess: handlePage, onError: noop } )` at `L516-L518` (registered for `READER_STREAMS_PAGE_REQUEST` at `L515`).
- `receivePosts` thunk (`client/state/reader/posts/actions.js:L63`) **first** dispatches a `receiveLikes` for each non-external post (`L76-L81`; the comment at `L73` reads "dispatch post like additions before the posts") → `POST_LIKES_RECEIVE` (`client/state/posts/likes/actions.js:L80-L81`), **then** dispatches `READER_POSTS_RECEIVE` (`client/state/reader/posts/actions.js:L86-L88`). This ordering is exactly what the observed stream shows (7 `POST_LIKES_RECEIVE` before `READER_POSTS_RECEIVE`).
- Bootstrap actions: `client/state/action-types.ts:L139` (`CURRENT_USER_RECEIVE`), `L848` (`ROUTE_SET`), `L850` (`SECTION_SET`); `setSection` → `SECTION_SET` at `client/state/ui/section/actions.js:L5,L8`; Reader section registration at `client/sections.js:L386`.

### Observed output — real boot action stream (live browser, logged-out `/discover`)

**Method.** Calypso's Redux store attaches the Redux-DevTools enhancer `window.__REDUX_DEVTOOLS_EXTENSION__` (`client/state/index.ts:L49`, applied inside `createStore` via the `compose`d enhancers at `client/state/index.ts:L54`). Before navigating, a tiny fake devtools enhancer was installed through the page's init-script hook; it records the `type` of **every** dispatched action into `window.__BLITZY_ACTIONS__` without altering behavior. Navigating to `http://calypso.localhost:3000/reader` while logged-out redirected to `/discover` and loaded real posts. **378 actions (38 unique types)** were captured in dispatch order. The contiguous slice spanning section-boot through the stream fetch (0-based indices) is reproduced verbatim:

```
25 PREFERENCES_FETCH
26 SECTION_LOADING_SET
27 SECTION_SET
28 LAYOUT_NEXT_FOCUS_ACTIVATE
29 SECTION_SET
30 LAYOUT_NEXT_FOCUS_ACTIVATE
31 ROUTE_SET
32 WPCOM_HTTP_REQUEST
33 USER_SETTINGS_REQUEST
34 SECTION_LOADING_SET
35 SECTION_LOADING_SET
36 SECTION_SET
37 LAYOUT_NEXT_FOCUS_ACTIVATE
38 ROUTE_SET
39 DOCUMENT_HEAD_TITLE_SET
40 DOCUMENT_HEAD_META_SET
41 PREFERENCES_FETCH_FAILURE
42 DOCUMENT_HEAD_UNREAD_COUNT_SET
43 READER_RESET_CARD_EXPANSIONS
44 READER_VIEW_STREAM
45 WPCOM_HTTP_REQUEST
46 READER_STREAMS_PAGE_REQUEST
47 POST_LIKES_RECEIVE
48 POST_LIKES_RECEIVE
49 POST_LIKES_RECEIVE
50 POST_LIKES_RECEIVE
51 POST_LIKES_RECEIVE
52 POST_LIKES_RECEIVE
53 POST_LIKES_RECEIVE
54 READER_POSTS_RECEIVE
55 READER_RECOMMENDED_SITES_RECEIVE
56 READER_STREAMS_PAGE_RECEIVE
57 WPCOM_HTTP_REQUEST
58 READER_STREAMS_PAGE_REQUEST
59 WPCOM_HTTP_REQUEST
```

**First-dispatch index and total count for the Q4 actions of interest:**

| Action type | First index | Count | Observed? |
|---|---|---|---|
| `SECTION_SET` | 27 | 3 | ✅ observed |
| `ROUTE_SET` | 31 | 2 | ✅ observed |
| `READER_STREAMS_PAGE_REQUEST` | 46 | 2 | ✅ observed |
| `POST_LIKES_RECEIVE` | 47 | 14 (7 contiguous at 47–53) | ✅ observed |
| `READER_POSTS_RECEIVE` | 54 | 4 | ✅ observed |
| `READER_RECOMMENDED_SITES_RECEIVE` | 55 | 2 | ✅ observed |
| `READER_STREAMS_PAGE_RECEIVE` | 56 | 2 | ✅ observed |
| `CURRENT_USER_RECEIVE` | — | **0** | ❌ did not fire (logged-out; the logged-in branch could not be observed live — no credentials) |

- The 7 `POST_LIKES_RECEIVE` at indices 47–53 correspond one-to-one to the 7 non-external `post` cards in the Discover response (Q3); they are dispatched *before* `READER_POSTS_RECEIVE` by the `receivePosts` thunk, exactly as its source prescribes.

### Cause → effect

The component's `componentDidMount` (`client/reader/stream/index.jsx:L221`) calls `fetchNextPage` (`L225` → `L489`), which dispatches `requestPage( { feedId: selectedFeedId, streamKey, pageHandle, localeSlug } )` (`L502`) → `READER_STREAMS_PAGE_REQUEST` (`client/state/reader/streams/actions.js:L39`) — observed at index 46. The data-layer's `dispatchRequest` mapping (`client/state/data-layer/wpcom/read/streams/index.js:L516-L518`) turns that plain action into the `http` GET (the `WPCOM_HTTP_REQUEST` at index 45; Q3) and, on success, calls `handlePage` (`client/state/data-layer/wpcom/read/streams/index.js:L428`). `handlePage` pushes the `receivePosts` thunk (`client/state/data-layer/wpcom/read/streams/index.js:L470`); that thunk dispatches a `POST_LIKES_RECEIVE` for each of the 7 non-external posts (indices 47–53; `client/state/reader/posts/actions.js:L76-L81`) **before** dispatching `READER_POSTS_RECEIVE` (index 54; `client/state/reader/posts/actions.js:L86-L88`). Because Discover's payload also carries a `recommended_blogs` card, `handlePage` then dispatches `receiveRecommendedSites` → `READER_RECOMMENDED_SITES_RECEIVE` (index 55; `client/state/data-layer/wpcom/read/streams/index.js:L474`), and finally `receivePage` → `READER_STREAMS_PAGE_RECEIVE` (index 56; `client/state/data-layer/wpcom/read/streams/index.js:L497` → `client/state/reader/streams/actions.js:L63`). This matches the observed order exactly.

### Sibling variants

- **Conditional analytics:** `handlePage` appends `recordTracksEvent` analytics actions only for items that carry a `railcar` (`client/state/data-layer/wpcom/read/streams/index.js:L51-L53`, filter `!! item.railcar`). No traintracks analytics action appeared between indices 46 and 56 in the observed stream (the Discover cards carried no `railcar`), so **none** was appended — a real conditional branch, not an omission.
- **Bootstrap actions.** `SECTION_SET` (`client/state/action-types.ts:L850`; dispatched by `setSection`, `client/state/ui/section/actions.js:L5,L8`) and `ROUTE_SET` (`client/state/action-types.ts:L848`) were **observed** at indices 27 and 31 respectively — before the stream fetch. `CURRENT_USER_RECEIVE` (`client/state/action-types.ts:L139`) is the logged-in-branch bootstrap action; it did **not** fire in the logged-out session and the logged-in branch could not be observed live (no credentials), so that single action is *(inferred)* from source while `SECTION_SET` / `ROUTE_SET` are observed.
- **First page vs later pages:** the same `READER_STREAMS_PAGE_REQUEST` → `READER_STREAMS_PAGE_RECEIVE` cascade repeats for pagination — observed again beginning at index 57 (`WPCOM_HTTP_REQUEST` at 57, `READER_STREAMS_PAGE_REQUEST` at 58) — differing only in `number` (4 → 7, per Q3).

---

## Q5 — How does the app know whether someone is logged in before it decides what to render, and what storage mechanisms does it check?

### Direct answer

The render decision reads Redux via **`isUserLoggedIn(state)`**, which is **`getCurrentUserId(state) !== null`** — i.e., true iff the current-user id is non-null. That id is populated either by **server-side `/me` bootstrap** (gated on the **`wordpress_logged_in`** cookie) or by **client hydration** from **`window.initialReduxState`**. In OAuth mode (desktop/dev), the app additionally checks the **`wpcom_token`** — **cookie first, then `localStorage`** — and redirects to `/login` if neither exists. Persistent Redux state lives in **IndexedDB** (database **`calypso`**, object store **`calypso_store`**), keyed **`redux-state-<userId | 'logged-out'>`** (with optional `:subkey`); when IndexedDB is unavailable it **falls back to `window.localStorage`** under the same `redux-state-*` keys (`client/lib/browser-storage/index.ts:L280-L364`). A separate in-memory **storage-bypass** path exists for the **support-user** sandbox (not private/incognito).

### The render decision

- `client/state/current-user/selectors.js:L6-L7` — `getCurrentUserId( state )` returns `state.currentUser?.id`.
- `client/state/current-user/selectors.js:L15-L16` — `isUserLoggedIn( state )` returns `getCurrentUserId( state ) !== null`.
- `client/state/current-user/reducer.js:L24,L26-L27` — the `id` reducer defaults to `null` (`L24`) and returns `action.user.ID` on `CURRENT_USER_RECEIVE` (`L26-L27`).

**Synthetic-harness output** (labeled **synthetic** — no WordPress.com credentials were available to observe a *live* logged-in browser session, so the render-decision logic is exercised through a Node harness that replicates the selector code **verbatim** from `client/state/current-user/selectors.js:L7,L16` and the id reducer from `client/state/current-user/reducer.js:L24,L26-L27`; the logged-out branch is *additionally* confirmed live below). Command: `node /tmp/blitzy_evidence/q5_login_selector_harness.js`. Complete, unedited output:

```
=== Q5 SYNTHETIC login-detection selector harness ===

id reducer BEFORE (default)            => null   (reducer.js:L24)
id reducer AFTER CURRENT_USER_RECEIVE  => 12345   (reducer.js:L26-27)

[1] LOGGED-IN   { currentUser: { id: 12345 } }
  state={"currentUser":{"id":12345}}
  getCurrentUserId => 12345
  isUserLoggedIn   => true

[2] LOGGED-OUT  { currentUser: { id: null } }  (canonical initialized store)
  state={"currentUser":{"id":null}}
  getCurrentUserId => null
  isUserLoggedIn   => false

[3] UNINITIALIZED {} (currentUser slice absent) — EDGE
  state={}
  getCurrentUserId => undefined
  isUserLoggedIn   => true
```

- **State transition (before → after):** the `id` reducer starts at its default `null` (`client/state/current-user/reducer.js:L24`) and becomes `12345` after a `CURRENT_USER_RECEIVE` carrying `user.ID` (`client/state/current-user/reducer.js:L26-L27`).
- **Logged-in branch (synthetic):** `currentUser.id = 12345` → `isUserLoggedIn = true`. *Not observed in a live logged-in session* — no credentials; the value is produced by the verbatim-logic harness above.
- **Logged-out branch:** `currentUser.id = null` → `isUserLoggedIn = false`. This branch **was** observed live (see the logged-out storage capture below, where no auth cookie exists and the client `/me` returns 403).
- **Edge case (uninitialized state, no `currentUser` at all):** `getCurrentUserId` returns `undefined`, and `undefined !== null` is `true`, so `isUserLoggedIn` returns **`true`** for a completely empty/uninitialized state. This is a real nuance of the `!== null` check (`client/state/current-user/selectors.js:L16`): only an *explicit* `null` id reads as logged-out; the reducer's default is `null` (`client/state/current-user/reducer.js:L24`), so in practice a booted store has an explicit `null`, but a bare `{}` does not.

### The OAuth token check (`wpcom_token`): cookie → localStorage → false

- `packages/oauth-token/src/index.js:L7` — `TOKEN_NAME = 'wpcom_token'`.
- `packages/oauth-token/src/index.js:L10-L24` — `getToken()` parses `document.cookie` for `wpcom_token` **first** (`L11-L14`), then falls back to `store.get( TOKEN_NAME )` (**localStorage**, `L17`; `TOKEN_NAME` = `'wpcom_token'`), returning the token if found (`L19-L20`) or **`false`** if neither is present (`L23`).
- `client/boot/common.js:L154` — `oauthTokenMiddleware`; when `config.isEnabled( 'oauth' )` (`L155`) and the route is not a logged-out route (`L156`), a missing token (`getToken() === false`, `L176`) triggers the redirect `window.location = authorizePath()` (`L177`).

**Synthetic-harness output** (verbatim `getToken()`/`setToken()` logic from `packages/oauth-token/src/index.js:L10-L28`, run against the **real `cookie` package** — the same dependency the module imports at `L1` — with a mocked `store` and a settable `document.cookie`; the browser OAuth flow was not driven live). Command: `NODE_PATH=<repo>/node_modules node /tmp/blitzy_evidence/q5_oauth_token_harness.js`. Complete, unedited output:

```
=== Q5 SYNTHETIC oauth getToken() harness (verbatim logic, real `cookie` pkg) ===

case 1  cookie present                    -> getToken() = "COOKIE_TOKEN_ABC"
case 2  no cookie, localStorage present   -> getToken() = "LOCALSTORAGE_TOKEN_XYZ"
case 3  neither cookie nor localStorage   -> getToken() = false
setToken("ROUNDTRIP_TOKEN"); getToken()   -> "ROUNDTRIP_TOKEN"
```

### Two distinct `/me` calls: the server-side **v1** bootstrap vs. the client-side **v1.1** fetch

There are **two different `/me` endpoints**, and they must not be conflated:

- **Server-side bootstrap → `rest/v1/me`** (gated on `wordpress_logged_in`):
  - `client/server/user-bootstrap/index.js:L8` — `AUTH_COOKIE_NAME = 'wordpress_logged_in'`.
  - `client/server/user-bootstrap/index.js:L13` — `API_PATH = 'https://public-api.wordpress.com/rest/v1/me'` (note: **v1**, not v1.1).
  - `client/server/user-bootstrap/index.js:L14-L17` — appends `?meta=flags`.
  - `client/server/user-bootstrap/index.js:L27` — `export default async function getBootstrappedUser( request )`; `L28` reads `request.cookies[ AUTH_COOKIE_NAME ]`; `L33-L34` — throws `'Cannot bootstrap without an auth cookie'` (does **not** bootstrap) if that cookie is absent. *(inferred — this server path only runs during SSR with a `wordpress_logged_in` cookie, which requires credentials that were not available, so it was not exercised at runtime.)*
- **Client-side fetch → `rest/v1.1/me`** (the browser's current-user request):
  - `client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L6` — `rawCurrentUserFetch()` calls `wpcom.me().get( { meta: 'flags' } )`, which resolves to the WordPress.com REST **v1.1** `/me` endpoint. This is the call the running browser actually issues.

**Observed (live browser, logged-out `/discover`) — the client v1.1 `/me` fetch:** the running app issued

```
GET https://public-api.wordpress.com/rest/v1.1/me?http_envelope=1&meta=flags
```

and, because the session was logged-out (no `wordpress_logged_in` / `wpcom_token`), the response was an HTTP-envelope **403** (`code:403`, 210 bytes, no PII). Complete, unedited response body (`/tmp/blitzy_evidence/q5_me_response.network-response`):

```json
{"code":403,"headers":[{"name":"Content-Type","value":"application/json"}],"body":{"error":"authorization_required","message":"An active access token must be used to query information about the current user."}}
```

This 403 is exactly why `CURRENT_USER_RECEIVE` never fires in the logged-out session (Q4): there is no user to receive. The request is remote (`public-api.wordpress.com`), corroborating Q2. The logged-in `200` variant of this call could not be observed live (no credentials).

### Client hydration and the persistence key scheme

- `client/state/initial-state.js:L148-L149` — `getInitialServerState()` reads **`window.initialReduxState`**; `L153` deserializes it; `L154` picks the persisted slices.
- `client/state/initial-state.js:L75-L76` — the persistence key: `'redux-state-' + ( userId ?? 'logged-out' ) + ( subkey ? ':' + subkey : '' )`.
- `client/state/persisted-state.js:L15,L17` — `loadPersistedState` reads all stored items matching `/^(redux-state|query-state)-/`.

### IndexedDB persistence store + bypass

- `client/lib/browser-storage/index.ts:L20` — `DB_NAME = 'calypso'`; `L21` — `DB_VERSION = 2`; `L22` — `STORE_NAME = 'calypso_store'`.
- `client/lib/browser-storage/bypass.ts:L6-L10` — this module's own docstring (quoted verbatim, `sic` typo included):

  ```
  // This module defines a series of methods which bypasse all persistent storage.
  // Any calls to read/write data using browser-storage instead access a temporary
  // in-memory store which is lost on page reload. This driver is used to sandbox
  // a user's data while support-user is active, ensuring it does not contaminate
  // the original user, and vice versa.
  ```

  It is therefore **not** a private/incognito path — it is the **support-user** sandbox. It is backed by an in-memory `Map` (`client/lib/browser-storage/bypass.ts:L12`, `const memoryStore = new Map()`) that is reset by `activate()` (`client/lib/browser-storage/bypass.ts:L47`).
- The bypass is switched on by the support-user flow: `client/lib/user/support-user-interop.js:L90` and `L108` both call `bypassPersistentStorage( true )` (imported at `client/lib/user/support-user-interop.js:L2`). *(inferred — the support-user path was not exercised at runtime; the docstring, in-memory store, and both trigger sites are read from source.)*

### Observed output — live browser storage on the **logged-out** `/discover` page

Captured live via Chrome DevTools at `http://calypso.localhost:3000/discover` (source: `/tmp/blitzy_evidence/q5_loggedout_storage.json`). Volatile analytics cookie values are redacted; everything else is verbatim:

```
cookie names        = ["tk_ai", "country_code", "region", "tk_qs"]   # values [REDACTED:analytics-cookie]
hasAuthCookie(wordpress_logged_in) = false
hasWpcomTokenCookie                = false
localStorage keys   = ["tusSupport"]
IndexedDB databases = ["calypso (v2)"]
calypso_store keys  = [
  "browser-storage-sanity-test",
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
  "redux-state-logged-out:userSuggestions"
]
window.initialReduxState present = true   (topKeys = ["documentHead"])
bodyClass   = "color-scheme theme-default is-group-reader is-section-reader font-smoothing-antialiased is-reader-page"
layoutClass = "layout is-group-reader is-section-reader focus-content has-header-section has-no-sidebar feature-flag-woocommerce-core-profiler-passwordless-auth"
```

- The IndexedDB database is exactly `calypso` at version 2 (`client/lib/browser-storage/index.ts:L20-L21`); its object store is `calypso_store` (`L22`).
- `browser-storage-sanity-test` is the availability-probe key `SANITY_TEST_KEY` written by the storage layer itself (`client/lib/browser-storage/index.ts:L24`), not a persisted Redux slice.
- The remaining keys are exactly the `redux-state-<userId | 'logged-out'>[:subkey]` scheme (`client/state/initial-state.js:L75-L76`), here with `userId` absent → `'logged-out'`. Persisted subkeys accumulate as their state subtrees are first written, so the exact set is **timing-dependent**; the enumeration above is the reproducible set observed here, stable across two runs — **16** `redux-state-logged-out*` keys (the base key plus 15 subkeys), i.e. 17 `calypso_store` keys once the `browser-storage-sanity-test` probe is included.
- **No** `wordpress_logged_in` and **no** `wpcom_token` cookies exist in the logged-out branch (`hasAuthCookie = false`, `hasWpcomTokenCookie = false`) — consistent with `isUserLoggedIn = false` and with the server-bootstrap gate not firing.
- `layoutClass` contains `has-no-sidebar` and no Reader sidebar header is present in the DOM — because the Reader sidebar mounts only when logged in (`client/reader/controller.js:L37-L44`); see Q6.

### The two render branches (cross-product)

- **Logged-in** *(inferred from source — not observed live; no credentials were available to drive a logged-in session):* `wordpress_logged_in` cookie present → server `/me` (`rest/v1/me`) bootstrap populates `currentUser.id` → `isUserLoggedIn = true` → logged-in UI; persisted state keyed `redux-state-<userId>`; in OAuth mode `getToken()` returns the token (cookie or localStorage).
- **Logged-out** *(observed live):* no auth cookie (`hasAuthCookie = false`) → no bootstrap → the client `rest/v1.1/me` fetch returns 403 → `currentUser.id = null` → `isUserLoggedIn = false`; persisted state keyed `redux-state-logged-out` (observed 16 such keys above — the base key + 15 subkeys); in OAuth mode `getToken()` returns `false` and `oauthTokenMiddleware` redirects via `window.location = authorizePath()` (`client/boot/common.js:L176-L177`). The Reader still renders logged-out because its section sets `enableLoggedOut: true` (`client/sections.js:L396`).

### Cause → effect

Rendering keys off Redux `currentUser.id` (`client/state/current-user/selectors.js:L16`). That id is set from either the server `/me` bootstrap — which only runs when the `wordpress_logged_in` cookie is present (`client/server/user-bootstrap/index.js:L8,L32-L34`) — or client hydration from `window.initialReduxState` (`client/state/initial-state.js:L149`). In OAuth mode the additional `wpcom_token` check (cookie then localStorage, `packages/oauth-token/src/index.js:L10-L23`) decides whether to redirect to `/login`. Between visits, state is rehydrated from IndexedDB `calypso/calypso_store` under `redux-state-<userId|'logged-out'>` (`client/lib/browser-storage/index.ts:L20-L22`; `client/state/initial-state.js:L75-L76`; or, when `supportsIDB()` returns `false`, from `window.localStorage` under the same `redux-state-*` keys — `client/lib/browser-storage/index.ts:L307-L310,L339-L342`), which is why the logged-out session shows `redux-state-logged-out*` keys and no auth cookies.

### Storage mechanisms checked (exhaustive, by name)

1. **`wordpress_logged_in` cookie** — gates server `/me` bootstrap (`client/server/user-bootstrap/index.js:L8,L13`).
2. **`wpcom_token` cookie** — first source in OAuth `getToken()` (`packages/oauth-token/src/index.js:L10-L14`).
3. **`localStorage` (`wpcom_token`)** — fallback in `getToken()` via `store.get` (`packages/oauth-token/src/index.js:L17`).
4. **`window.initialReduxState`** — SSR-injected client hydration (`client/state/initial-state.js:L149`).
5. **IndexedDB `calypso` / `calypso_store`** — persisted Redux state (`client/lib/browser-storage/index.ts:L20-L22`), keyed `redux-state-<userId|'logged-out'>[:subkey]` (`client/state/initial-state.js:L75-L76`; matched by `client/state/persisted-state.js:L17`).
6. **In-memory bypass store** — the **support-user** sandbox (not private/incognito): an in-memory `Map` used while support-user is active (`client/lib/browser-storage/bypass.ts:L6-L10,L12`), switched on by `bypassPersistentStorage( true )` at `client/lib/user/support-user-interop.js:L90,L108`.
7. **`window.localStorage` (persisted Redux-state fallback)** *(non-canonical — not exercised in this environment, where IndexedDB was available)* — when `supportsIDB()` returns `false` (i.e. `window.indexedDB` is absent (`L37`), `shouldDisableIDB` is set (`L42`), or the IDB sanity-write throws (`L52-L54`); all in `client/lib/browser-storage/index.ts:L36-L56`), the **same `redux-state-*` keys** are read/written through `window.localStorage` instead of IndexedDB: `getStoredItem` (`client/lib/browser-storage/index.ts:L283`), `getAllStoredItems` (`L310`), `setStoredItem` (`L342`), `clearStorage` (`L364`). In the observed logged-out session `supportsIDB()` was `true` (the `calypso`/`calypso_store` database was present — mechanism 5), so this fallback path did not run.

---

## Q6 — Sidebar responsive design

This question has three parts: (a) the sidebar header's margin/padding, (b) the CSS custom properties driving the layout `calc()`s, and (c) the viewport widths at which things change.

### Q6a — What are the specific margin and padding values on the sidebar header?

#### Direct answer

There are **three** plausible "sidebar header" elements. The one that manifests in the **Reader** is `.is-section-reader .sidebar-header`, with **`margin: 0 12px 44px`** and **`padding: 0 10px`**. The two siblings are the classic list heading `.sidebar__heading` (`padding: 16px 8px 6px 16px; margin: 0`) and the global-nav branding header `.sidebar__header` (`padding: 30px 24px 29px`).

#### Observed output — injected-DOM real-CSS (live browser `getComputedStyle`)

**Why injection is required:** the Reader sidebar (and therefore its `.sidebar-header`) mounts **only when logged in** — `client/reader/controller.js:L37` gates `context.secondary = <ReaderSidebar>` on `isUserLoggedIn( state )`, and the element is `<li className="sidebar-header">` (`client/reader/sidebar/index.jsx:L168`). With no credentials, a live logged-in sidebar could not render, so an element matching `.is-section-reader .sidebar-header` was **injected into the live Reader document** (whose `<body>` already carries `is-section-reader` — confirmed in Q5) and read back with `getComputedStyle`. This is the **real compiled Reader stylesheet** applied to a real element in the running page — not a CLI compile. Source: `/tmp/blitzy_evidence/q6a_computed_sidebar_header.json`. Complete, unedited values:

```
.is-section-reader .sidebar-header   (getComputedStyle, live Reader document)
  display          = flex
  justify-content  = space-between
  margin           = 0px 12px 44px      (top 0, right 12, bottom 44, left 12)
  padding          = 0px 10px           (top 0, right 10, bottom 0, left 10)
```

- These runtime values match the source rule exactly: `.sidebar-header` (`client/reader/sidebar/style.scss:L113`) nested under `.is-section-reader` (`L112`) — `display: flex` (`L114`), `justify-content: space-between` (`L115`), `margin: 0 12px 44px` (`L116`), `padding: 0 10px` (`L117`).

The two siblings do **not** render on the Reader page (they belong to the classic list chrome and the global-nav branding row), so they are quoted **verbatim from source** rather than from `getComputedStyle`.

**Sibling 1 — the classic list heading (documented as used "like in Reader"), verbatim SCSS:**

```scss
.sidebar__heading {
  color: var(--color-sidebar-text-alternative);
  font-size: $font-body;
  font-weight: 600;
  padding: 16px 8px 6px 16px;
  margin: 0;
  outline: 0;
}
```

- `client/layout/sidebar/style.scss:L64-L65` (the comment reads "Sidebar Headings, used for both static headings like in Reader, and for the expandable menus."), `L66` (selector), `L67` (`color`), `L68` (`font-size: $font-body`), `L69` (`font-weight: 600`), `L70` (`padding: 16px 8px 6px 16px`), `L71` (`margin: 0`), `L72` (`outline: 0`), `L73` (closing `}`) — this is the **complete** rule. `$font-body` is `rem(16px)` (the `@automattic/typography` workspace package, `packages/typography/styles/variables.scss:L45`), i.e. `1rem` at the 16px root — so the header's padding/margin (the values Q6a asks for) are the literal `16px 8px 6px 16px` / `0`.

**Sibling 2 — the global-nav branding header (the header's own box declarations, verbatim):**

```scss
.sidebar__header {
  align-items: center;
  // Hide the header when the masterbar is visible.
  display: none;
  gap: 8px;
  padding: 30px 24px 29px;
```

- `client/layout/global-sidebar/style.scss:L70` (selector, nested under `.global-sidebar`), `L71` (`align-items: center`), `L73` (`display: none`), `L74` (`gap: 8px`), `L75` (`padding: 30px 24px 29px`). The `.sidebar__header` rule declares **no `margin` of its own** — its only box value relevant to Q6a is `padding: 30px 24px 29px`. The rule then continues with nested child selectors — `a` (`L77-L80`), `span.dotcom` (`L82-L90`), `.link-logo` (`L92-L94`) through ~`L100` — which style the header's **children** (logo image, links), not the header's own box, so they are omitted here as out-of-scope for the margin/padding question. The header itself is shown (`display: flex`) only when there is no masterbar: `.has-no-masterbar .global-sidebar .sidebar__header` (`client/layout/global-sidebar/style.scss:L452,L458-L460`).

#### Cause → effect

`.is-section-reader .sidebar-header` is scoped to the Reader section body class, so it wins on Reader pages and applies `margin: 0 12px 44px` / `padding: 0 10px` (`client/reader/sidebar/style.scss:L116-L117`). The `44px` bottom margin creates the gap below the Reader header; the flex (`L114`) + `justify-content: space-between` (`L115`) spreads the header's children. The classic `.sidebar__heading` and the global `.sidebar__header` target different DOM (list section headings and the global-nav branding row), so they don't override the Reader header.

#### Note on live capture

The Reader sidebar mounts **only when logged in** (`client/reader/controller.js:L37`), and no WordPress.com credentials were available, so the header does not appear on the logged-out `/discover` page. To obtain a **real runtime** value rather than a source-only reading, an element matching `.is-section-reader .sidebar-header` was injected into the **live** Reader document (its `<body>` carries `is-section-reader`, confirmed in Q5) and measured with `getComputedStyle` — the injected-DOM real-CSS capture shown above. Its `margin`/`padding` match the source rule (`client/reader/sidebar/style.scss:L116-L117`) exactly. The two sibling headers are quoted verbatim from source because they do not render on the Reader page.

---

### Q6b — What CSS custom properties drive the layout calculations?

#### Direct answer

The layout geometry is computed from a small set of `:root` custom properties: **`--masterbar-height`** (`46px`, dropping to `32px` at `min-width: 782px`), **`--masterbar-checkout-height`** (`72px`), **`--sidebar-width-max`** (`272px` at `:root`, overridden to `295px` under `.theme-default .is-global-sidebar-visible`), and **`--sidebar-width-min`** (`228px` at `:root`, likewise overridden), plus contextual **`--content-padding-top`/`--content-padding-bottom`** (`16px` each, defined only under `.theme-default .is-global-sidebar-visible`). The Reader sidebar's `padding` and `height` are `calc()` expressions built from these, so their resolved values depend on which of those contexts is active.

#### The `:root` custom properties (verbatim source)

```scss
:root {
  // Masterbar
  --masterbar-height: 46px;
  --masterbar-checkout-height: 72px;

  @media only screen and (min-width: 782px) {
    --masterbar-height: 32px;
  }

  // Sidebar size limits
  --sidebar-width-max: 272px;
  --sidebar-width-min: 228px;
}
```

- `client/assets/stylesheets/shared/_variables.scss:L5` (`:root`), `L7` (`--masterbar-height: 46px`), `L8` (`--masterbar-checkout-height: 72px`), `L10-L11` (`@media (min-width: 782px)` → `--masterbar-height: 32px`), `L15` (`--sidebar-width-max: 272px`), `L16` (`--sidebar-width-min: 228px`). The **live-observed** values of these appear below.

**Contextual content padding** (defined only inside `.theme-default .is-global-sidebar-visible`, verbatim source):

```scss
.theme-default {                      /* L10 */
  .is-global-sidebar-visible {        /* L15 */
    --content-padding-top: 16px;      /* L50 */
    --content-padding-bottom: 16px;   /* L51 */
  }
}
```

- `client/my-sites/sidebar/style.scss:L10` (`.theme-default`), `L15` (`.is-global-sidebar-visible`), `L50-L51` (the two custom properties). Because they live under that selector — not `:root` — they are unset on any page without the global sidebar (e.g., logged-out Discover).

**The `calc()` usages in the Reader sidebar (verbatim source; note the RTL/LTR split):**

```scss
/* RTL branch — body.is-section-reader.rtl .layout__content — L69-L70 */
padding: calc(var(--masterbar-height) + var(--content-padding-top)) calc(var(--sidebar-width-max)) var(--content-padding-bottom) 16px;

/* LTR base — body.is-section-reader .layout__content — L77-L78 */
padding-top: calc(var(--masterbar-height) + var(--content-padding-top));
padding-bottom: var(--content-padding-bottom);

/* LTR desktop — @media (min-width: 782px) — L79-L80 */
padding: calc(var(--masterbar-height) + var(--content-padding-top)) 16px var(--content-padding-bottom) calc(var(--sidebar-width-max)) !important;

/* scroll height — @media (max-width: 781px) — L102 / L107 */
height: calc(100vh - var(--masterbar-height) - var(--content-padding-top) - var(--content-padding-bottom));
```

- **RTL vs LTR (the key difference):** the sidebar gutter sits on the **right** in RTL (`client/reader/sidebar/style.scss:L70` — padding-**right** = `calc(var(--sidebar-width-max))`) and on the **left** in LTR desktop (`L80` — padding-**left** = `calc(var(--sidebar-width-max))`). The LTR non-desktop base sets only `padding-top`/`padding-bottom` (`L77-L78`); the full four-side padding with the sidebar gutter applies at `@media (min-width: 782px)` (`L79-L80`). The scroll `height` calc lives inside `@media (max-width: 781px)` (`L102`), so it applies only **below** 782px (`L107`).

**Observed live root custom properties** (running app, `getComputedStyle(document.documentElement)` at `innerWidth = 1905`, `>= 782px`; source `/tmp/blitzy_evidence/q6b_resolved_calc.json`):

```
--masterbar-height          = 32px      (desktop override active at >= 782px)
--masterbar-checkout-height = 72px
--sidebar-width-max         = 272px
--sidebar-width-min         = 228px
--content-padding-top       = (empty on :root)
--content-padding-bottom    = (empty on :root)
```

`--content-padding-*` are **empty on `:root`** because they are defined only under `.theme-default .is-global-sidebar-visible` (`client/my-sites/sidebar/style.scss:L10,L15,L50-L51`); the logged-out Discover document carried `has-no-sidebar`, not `is-global-sidebar-visible`, so they were unset there.

**Resolved `calc()` values** — the LTR-desktop padding (`client/reader/sidebar/style.scss:L80`) and the scroll height (`L107`) resolve differently depending on which context supplies `--sidebar-width-max` and `--content-padding-*`. The self-consistent **logged-in Reader chrome** is `.theme-default .is-global-sidebar-visible` — the **only** context that defines `--content-padding-* = 16px` (`client/my-sites/sidebar/style.scss:L50-L51`), and in that *same* context `--sidebar-width-max` is overridden to **`295px`** (`client/my-sites/sidebar/style.scss:L16`), **not** the `:root` `272px`. Using the observed `--masterbar-height` (`32px` @≥782px, `46px` @<782px) with that contextual `295px` / `16px` (measured viewport `innerHeight = 2053`):

```
LTR desktop padding (>= 782px; .is-global-sidebar-visible → --sidebar-width-max = 295px, --content-padding = 16px):
  padding-top    = calc(32px + 16px)              = 48px
  padding-right  = 16px
  padding-bottom = var(--content-padding-bottom)  = 16px
  padding-left   = calc(var(--sidebar-width-max)) = 295px

scroll height (< 782px, --masterbar-height = 46px):
  height = calc(100vh - 46px - 16px - 16px) = 2053 - 46 - 16 - 16 = 1975px
```

Below 782px the `--masterbar-height` reverts to `46px`, so the LTR base `padding-top` (`L77`) becomes `calc(46px + 16px) = 62px`. These are arithmetic resolutions of the `calc()` expressions — **not** a direct `getComputedStyle` of `.layout__content`, which could not be captured in the logged-in chrome without WordPress.com credentials. On the **logged-out** page the live-observed `--sidebar-width-max` was the `:root` base **`272px`** and `--content-padding-*` were unset, so the gutter padding did not resolve there. The gutter (`padding-left`) is therefore context-dependent: `295px` under `.is-global-sidebar-visible` (where content-padding is also set), `272px` at the `:root`/`.theme-default` base, `69px` under `.is-global-sidebar-collapsed` (`client/my-sites/sidebar/style.scss:L60`), and `0px` under `is-mobile-app-view` (`client/layout/style.scss:L169`) — see the enumeration below.

**A no-sidebar override exists** (note the accurate selector — it is `is-mobile-app-view`, not literally "no-sidebar"):

```css
body.is-mobile-app-view { --sidebar-width-max: 0px; --sidebar-width-min: 0px; }
```

- `client/layout/style.scss:L169-L170`.

#### Cause → effect

Because `--masterbar-height` is redefined inside `@media (min-width: 782px)` (`client/assets/stylesheets/shared/_variables.scss:L10-L11`), any `calc()` that references it recomputes at the 782px boundary. The Reader content `padding-top` (`calc(var(--masterbar-height) + var(--content-padding-top))`, LTR base at `client/reader/sidebar/style.scss:L77`) resolves to `calc(46px + 16px) = 62px` below 782px and `calc(32px + 16px) = 48px` at/above 782px — a `14px` shift (`46px − 32px`). Likewise the scroll `height` (`calc(100vh - var(--masterbar-height) - var(--content-padding-top) - var(--content-padding-bottom))`, `L107`, applied under `@media (max-width: 781px)` at `L102`) resolves to `2053 − 46 − 16 − 16 = 1975px` at the measured `innerHeight = 2053`. The live computed `--masterbar-height = 32px` at `innerWidth = 1905` confirms the desktop override is active; below 782px it is `46px` (observed in Q6c). Setting `--sidebar-width-*: 0px` in `is-mobile-app-view` (`client/layout/style.scss:L169-L170`) collapses the sidebar gutter in that context because `calc(var(--sidebar-width-max))` resolves to `0px`.

#### Custom properties enumerated (by name, with value + `file:line`)

- `--masterbar-height`: `46px`, → `32px` at `min-width:782px` (`client/assets/stylesheets/shared/_variables.scss:L7,L11`).
- `--masterbar-checkout-height`: `72px` (`client/assets/stylesheets/shared/_variables.scss:L8`).
- `--sidebar-width-max`: `272px` at `:root` (`client/assets/stylesheets/shared/_variables.scss:L15`); overridden to `272px` under `.theme-default` (`client/my-sites/sidebar/style.scss:L12`), **`295px`** under `.theme-default .is-global-sidebar-visible` (`client/my-sites/sidebar/style.scss:L16`), `69px` under `.is-global-sidebar-collapsed` (`client/my-sites/sidebar/style.scss:L60`), and `0px` under `is-mobile-app-view` (`client/layout/style.scss:L169`).
- `--sidebar-width-min`: `228px` at `:root` (`client/assets/stylesheets/shared/_variables.scss:L16`); overridden to `272px` under `.theme-default` (`client/my-sites/sidebar/style.scss:L13`), **`295px`** under `.theme-default .is-global-sidebar-visible` (`client/my-sites/sidebar/style.scss:L17`), `69px` under `.is-global-sidebar-collapsed` (`client/my-sites/sidebar/style.scss:L61`), and `0px` under `is-mobile-app-view` (`client/layout/style.scss:L170`).
- `--content-padding-top`: `16px` (`client/my-sites/sidebar/style.scss:L50`).
- `--content-padding-bottom`: `16px` (`client/my-sites/sidebar/style.scss:L51`).

---

### Q6c — At what viewport widths do things change?

#### Direct answer

Breakpoints come from **three systems**: (1) the JS runtime `mediaQueryOptions` map in `@automattic/viewport` (used by `isWithinBreakpoint`/`useBreakpoint`); (2) a deprecated SCSS `$breakpoints` list `480px, 660px, 800px, 960px, 1040px, 1280px, 1400px`; and (3) explicit layout-code thresholds `<660px` (narrow), `>=782px` (desktop), `>800px` (sidebar collapse). The layout-visible transitions observed are: **narrow ≤ 660px**, **masterbar height flips 46→32px and desktop turns on at 782px**, **sidebar collapse turns on above 800px (i.e. at 801px)**, and the **global sidebar toggles across the 660/661px boundary**.

#### System 1 — the JS `mediaQueryOptions` map (real, compiled `@automattic/viewport`)

- Source map: `packages/viewport/src/index.ts:L96-L118` (22 entries). Notable off-by-one storage: `>=782px` is stored `{ min: 781 }` (`L108`) and `>=960px` is stored `{ min: 959 }` (`L111`).

**Command** — a standalone temporary Node script that `require`s the **real compiled** `@automattic/viewport` module (the workspace build output for `packages/viewport`, generated from the tracked source `packages/viewport/src/index.ts`; the built bundle is a gitignored artifact, so the citation points to that tracked source) with a width-aware `matchMedia` mock, then prints the literal media query each key generates via the real `getMediaQueryList`:

```
node /tmp/blitzy_evidence/viewport_probe.js
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

**Cause → effect (the off-by-one):** the `createMediaQueryList` function (`packages/viewport/src/index.ts:L71`) builds min-queries with a `min + 1` offset. The source (verbatim, `L76`/`L82`/`L88`):

```ts
// L76 — range branch (both min and max):
: window.matchMedia( `(min-width: ${ min + 1 }px) and (max-width: ${ max }px)` );
// L82 — min-only branch:
: window.matchMedia( `(min-width: ${ min + 1 }px)` );
// L88 — max-only branch (no offset):
: window.matchMedia( `(max-width: ${ max }px)` );
```

(The runtime probe exercises the module's *compiled* output; the source that compiles to that behavior is `packages/viewport/src/index.ts`, since the repository checks in only the TypeScript source under `packages/viewport/src/` and not the compiled bundle.) So the stored `{ min: 781 }` for `>=782px` (`L108`) becomes the actual query `(min-width: 782px)` — **inclusive at exactly 782** — while `>782px` (`{ min: 782 }`, `L109`) becomes `(min-width: 783px)` — **strict** (one pixel higher). The same pattern makes `>=960px` (`{ min: 959 }`, `L111`) inclusive at 960 and `>960px` (`{ min: 960 }`, `L112`) strict at 961. Conversely the max-only branch (`L88`) has no offset, which is why `<660px` (`{ max: 660 }`) is `true` at exactly 660 (`(max-width: 660px)`), and why sidebar collapse (`>800px` → `(min-width: 801px)`) turns on at 801, not 800.

#### System 2 — the deprecated SCSS `$breakpoints` list

```scss
$breakpoints: 480px, 660px, 800px, 960px, 1040px, 1280px, 1400px;
```

- `client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10`; the `breakpoint-deprecated` mixin at `L12`.

#### System 3 — the explicit layout-code thresholds

- `client/layout/index.jsx:L76` — `useBreakpoint( '<660px' )` → `isNarrow`.
- `client/layout/index.jsx:L146` — `isWithinBreakpoint( '>=782px' )` → `isDesktop`.
- `client/layout/index.jsx:L221` — `isWithinBreakpoint( '>800px' )` → sidebar collapse.

**Global sidebar collapse boundary (verbatim source, not elided):**

The two boundary media queries and their complete bodies, `client/layout/global-sidebar/style.scss:L470-L509` and `L511-L515`:

```scss
/* client/layout/global-sidebar/style.scss:L470-L509 */
@media (min-width: 661px) {
	.is-global-sidebar-collapsed {
		.global-sidebar {
			.sidebar__header,
			.sidebar__footer {
				flex-direction: column;
			}

			.sidebar__header {
				span.dotcom {
					background-position: left;
					width: 24px;
					margin-left: 6px;

					.rtl & {
						background-position: right;
						margin-right: 6px;
					}
				}
			}

			.sidebar__footer {
				.sidebar__footer-language-switcher {
					font-size: 0;
					gap: 0;
					margin-inline-start: unset;
				}
			}

			.sidebar__body {
				overflow-y: visible;
				.sidebar__menu-item-parent .sidebar__menu-link {
					> *:not(:first-child) {
						display: none;
					}
				}
			}
		}
	}
}

/* client/layout/global-sidebar/style.scss:L511-L515 */
@media (max-width: 660px) {
	.global-sidebar .tooltip:hover::after {
		display: none;
	}
}
```

**Cause → effect:** above the boundary (`min-width: 661px`, `L470`) the *collapsed* global sidebar (`.is-global-sidebar-collapsed`) reflows its header and footer to `flex-direction: column` (`L475`), shrinks the `span.dotcom` logo to `24px` with a `6px` inline offset (`L481-L482`, mirrored for RTL at `L485-L486`), zeroes the footer language-switcher text (`font-size: 0`, `L493`), and hides all-but-first children of nested menu links (`display: none`, `L503`). Below the boundary (`max-width: 660px`, `L511`) the only rule is suppressing the sidebar tooltip hover pseudo-element (`.global-sidebar .tooltip:hover::after { display: none; }`, `L512-L514`) — i.e. on narrow widths the hover tooltip is turned off.

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
| Boot log (≠ ready) | `wp-calypso booted in 1005ms - http://calypso.localhost:3000` (run 1; run 2 = 1021 ms) | `client/server/index.js:L33` | Observed (2 runs) |
| Readiness banner (fully ready) | `Ready! You can load http://calypso.localhost:3000/ now. Have fun!` | `client/server/bundler/index.js:L56` | Observed (invariant across 2 runs; after ~69–74 s first compile) |
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
| Default stream key | `following` | `client/reader/controller.js:L87` | Source (module default; logged-out visitor redirected to Discover) |
| Default path | `/read/following`, apiVersion `1.2`, `number 4` | `client/state/data-layer/wpcom/read/streams/index.js:L194,L370,L161` | *(inferred)* — source (logged-in `following` not run; no creds); `number=4` observed live on Discover |
| All 20 stream keys → paths | see Q3 table | `client/state/data-layer/wpcom/read/streams/index.js:L192-L351` | Source-enumerated (module-local `const`; default stream confirmed live) |
| `INITIAL_FETCH` / `PER_FETCH` | `4` / `7` | `client/state/data-layer/wpcom/read/streams/index.js:L161,L160` | Observed (`number` 4→7 in browser) |
| `http()` nests query | `apiVersion`/`apiNamespace`/`number` in `action.query` | `client/state/data-layer/wpcom-http/actions.js:L55-L66` | Source; query params observed on live Discover request |
| `list` variant | `number 40`, apiVersion `1.3` | `client/state/data-layer/wpcom/read/streams/index.js:L332-L338` | Source (variant not exercised live) |
| `recommendations_posts` variant | `number` undefined | `client/state/data-layer/wpcom/read/streams/index.js:L286-L287` | Source (variant not exercised live) |
| Logged-out default | Discover → `/wpcom/v2/read/streams/discover` | `client/state/data-layer/wpcom/read/streams/index.js:L223-L224` | Observed (reqid=149) |

### Q4 — initial-load Redux actions

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Mount dispatch | `READER_STREAMS_PAGE_REQUEST` | `client/reader/stream/index.jsx:L221,L502`; `client/state/reader/streams/actions.js:L39` | Observed |
| Success cascade | `READER_STREAMS_PAGE_REQUEST` → `READER_POSTS_RECEIVE` → `READER_STREAMS_PAGE_RECEIVE` | `client/state/data-layer/wpcom/read/streams/index.js:L428,L497`; `client/state/reader/posts/actions.js:L87`; `client/state/reader/streams/actions.js:L63` | Observed (ordered stream) |
| Analytics (conditional) | `recordTracksEvent` only when post has `railcar` | `client/state/data-layer/wpcom/read/streams/index.js:L428-L511` | Observed (railcar null → none) |
| Bootstrap actions | `SECTION_SET` (idx 27), `ROUTE_SET` (idx 31); `CURRENT_USER_RECEIVE` | `client/state/action-types.ts:L850,L848,L139`; `client/state/ui/section/actions.js:L5,L8` | `SECTION_SET`/`ROUTE_SET` **Observed** (in the 378-action stream, before the fetch); `CURRENT_USER_RECEIVE` (inferred) — did not fire logged-out |

### Q5 — login detection & storage (both branches)

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Render decision | `isUserLoggedIn = getCurrentUserId(state) !== null` | `client/state/current-user/selectors.js:L6-L7,L15-L16` | Synthetic-harness (true/false/edge) |
| Logged-in | id `12345` → `true` | `client/state/current-user/reducer.js:L24,L26-L27` | Synthetic-harness (no live creds) |
| Logged-out | id `null` → `false` | `client/state/current-user/reducer.js:L24` | Synthetic-harness (corroborated live) |
| Uninitialized edge case | `undefined !== null` → `true` | `client/state/current-user/selectors.js:L16` | Synthetic-harness |
| `wpcom_token` getToken order | cookie → localStorage → `false` | `packages/oauth-token/src/index.js:L7,L10-L23` | Synthetic-harness (3 sub-cases) |
| OAuth redirect | `/login` when `getToken()===false` | `client/boot/common.js:L154,L176-L177` | (inferred) from source |
| Server `/me` bootstrap | gated on `wordpress_logged_in` | `client/server/user-bootstrap/index.js:L8,L13,L32-L34` | (inferred) — server path not exercised (no creds); client `/me` observed **403** |
| Client hydration | `window.initialReduxState` | `client/state/initial-state.js:L149` | Observed (present in SSR HTML) |
| IndexedDB store | `calypso` (v2) / `calypso_store` | `client/lib/browser-storage/index.ts:L20-L22` | Observed (live) |
| Persistence key | `redux-state-<userId\|'logged-out'>[:subkey]` | `client/state/initial-state.js:L75-L76` | Observed (16 `redux-state-logged-out*` keys, stable across 2 runs) |
| Logged-out Reader | `enableLoggedOut: true` | `client/sections.js:L396` | Observed (Reader renders) |
| Bypass store | support-user sandbox (in-memory) | `client/lib/browser-storage/bypass.ts:L6,L12` | (inferred) |

### Q6 — sidebar responsive design

| Item | Value | `file:line` | Evidence |
|---|---|---|---|
| Q6a Reader header (manifests) | `margin: 0 12px 44px; padding: 0 10px` | `client/reader/sidebar/style.scss:L112-L117` | Observed (injected-DOM real-CSS `getComputedStyle`) |
| Q6a sibling `.sidebar__heading` | `padding: 16px 8px 6px 16px; margin: 0` | `client/layout/sidebar/style.scss:L66,L70,L71` | Source (verbatim; does not render on Reader) |
| Q6a sibling `.sidebar__header` | `padding: 30px 24px 29px` | `client/layout/global-sidebar/style.scss:L70,L75` | Source (verbatim; does not render on Reader) |
| Q6b `--masterbar-height` | `46px` → `32px` @≥782px | `client/assets/stylesheets/shared/_variables.scss:L7,L10-L11` | Observed (live `:root`; `32px` @≥782px) |
| Q6b `--masterbar-checkout-height` | `72px` | `client/assets/stylesheets/shared/_variables.scss:L8` | Observed (live `:root`) |
| Q6b `--sidebar-width-max` | `272px` `:root` → `295px` (is-global-sidebar-visible) / `69px` (collapsed) / `0px` (mobile-app) | `client/assets/stylesheets/shared/_variables.scss:L15`; `client/my-sites/sidebar/style.scss:L16,L60`; `client/layout/style.scss:L169` | Observed (live `:root` 272px); overrides from source |
| Q6b `--sidebar-width-min` | `228px` `:root` → `295px` (is-global-sidebar-visible) / `69px` (collapsed) / `0px` (mobile-app) | `client/assets/stylesheets/shared/_variables.scss:L16`; `client/my-sites/sidebar/style.scss:L17,L61`; `client/layout/style.scss:L170` | Observed (live `:root` 228px); overrides from source |
| Q6b `--content-padding-top/bottom` | `16px` / `16px` | `client/my-sites/sidebar/style.scss:L10,L15,L50-L51` | Source (empty on `:root` live; set only under `.theme-default .is-global-sidebar-visible`) |
| Q6b Reader `padding` calc() | `calc(--masterbar-height + --content-padding-top) 16px --content-padding-bottom calc(--sidebar-width-max)` (LTR @≥782px, `L80`); base `padding-top: calc(--masterbar-height + --content-padding-top)` (`L77`) | `client/reader/sidebar/style.scss:L77,L80` (LTR; `L70` is the RTL branch) | Source + resolved (`padding-top` 48px @≥782px / 62px @<782px) |
| Q6b Reader `height` calc() | `calc(100vh - --masterbar-height - --content-padding-top - --content-padding-bottom)` | `client/reader/sidebar/style.scss:L107` (inside `@media (max-width:781px)` at `L102`) | Source + resolved (`1975px` @<782px) |
| Q6c JS map | 22 entries; `>=782px`={min:781}, `>=960px`={min:959} | `packages/viewport/src/index.ts:L96-L118` | Observed (query strings + booleans via probe) |
| Q6c off-by-one | `>=782px`→`min-width:782px` (incl 782); `>782px`→`min-width:783px` | `packages/viewport/src/index.ts:L76,L82,L108` | Observed (probe) + source |
| Q6c SCSS `$breakpoints` | `480,660,800,960,1040,1280,1400px` | `client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10` | Source (read) |
| Q6c layout thresholds | `<660px` / `>=782px` / `>800px` | `client/layout/index.jsx:L76,L146,L221` | Observed (booleans) |
| Q6c global-sidebar boundary | `min-width:661px` / `max-width:660px` | `client/layout/global-sidebar/style.scss:L470,L511` | Observed (live 640/700) |
| Q6c masterbar flip | `46px↔32px` at 782px | `client/assets/stylesheets/shared/_variables.scss:L10-L11` | Observed (live 700→46, 1000→32) |

---

## Notes on fidelity & scope

- **Read-only:** No existing source file was modified, created, or deleted. The only new artifact is this document under the `blitzy/` tree. All temporary observation scripts were kept **outside** the repository (under `/tmp/blitzy_evidence/`) and removed after capturing output; `git status --porcelain` shows only this document.
- **Observation harnesses (how the runtime evidence was captured):** (1) the **live dev server** (two runs) for the boot log, readiness banner, holding page, SSR HTML, `runtime.js`, and the `/__webpack_hmr` stream; (2) a **Chrome DevTools** session on the running Reader for the live network calls (Q3), the storage/cookie/IndexedDB reads (Q5), the `--masterbar-height`/`matchMedia` resize probes and the **injected-DOM real-CSS** `getComputedStyle` of `.is-section-reader .sidebar-header` (Q6); (3) a **boot-time Redux action capture** via a fake `__REDUX_DEVTOOLS_EXTENSION__` hook installed before boot, yielding the ordered 378-action stream (Q4); and (4) small **standalone Node harnesses** that run the *verbatim* selector / `getToken` code and `require` the real compiled `@automattic/viewport` module (Q5/Q6c). Harnesses that replicate code rather than observe a live flow are labeled **Synthetic-harness**; source-only reads are labeled **`(inferred)`**.
- **Logged-in limitation (honest scope):** No WordPress.com credentials were available, so the **live logged-in** branch (server `/me` bootstrap, logged-in render, `CURRENT_USER_RECEIVE`, the logged-in `/me` `200`) could not be observed. Those items are labeled `(inferred)` or **Synthetic-harness**; the **logged-out** branch was observed live end-to-end (client `/me` returns **403**, no auth cookie, sidebar absent).
- **Canonical runtime:** All observations were produced under Node `v22.23.1` / yarn `4.0.2` (the pinned canonical runtime). The Node-20 gate discrepancy is documented in the Environment section above.
- **`(inferred)` labels:** A handful of claims are labeled `(inferred)` where a clean runtime trigger would have required violating the read-only constraint (the recompile banner text) or exercising an environment/flow not available here (Node 20 itself, the **support-user** storage bypass, the server-side `/me` bootstrap and OAuth `/login` redirect, the logged-in render branch, and the surrounding bootstrap actions — none of which could be driven without WordPress.com credentials or a source edit). Every such claim is grounded in an exact `file:line`, and the runtime-observable siblings (the logged-out branch, the client `/me` 403, the `SECTION_SET`/`ROUTE_SET` dispatches) were observed. All other claims carry observed runtime output.

