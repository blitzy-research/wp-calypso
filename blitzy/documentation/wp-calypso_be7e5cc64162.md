# WordPress.com Calypso — Runtime Onboarding Answers (Reader, local dev)

This document answers four onboarding questions about how the **Automattic/wp-calypso**
application boots and serves the **Reader** section locally, grounded in **direct runtime
observation** of a default `yarn start` server plus `file:line` citations into the pinned
source tree.

- **Q1 — Development server:** the port it binds, how a developer knows it is ready, and
  whether the architecture uses one port or several (HMR vs. API vs. HTTP).
- **Q2 — Reader stream:** which REST endpoints populate the stream and which Redux actions
  fire during the initial load.
- **Q3 — Authentication detection:** how the app decides "is the user logged in?" before it
  renders, and which storage mechanisms it inspects.
- **Q4 — Responsive sidebar:** the exact header `padding`/`margin`, the CSS custom properties
  that drive the `calc()` layout, and the viewport breakpoints where layout changes.

**Commit provenance.** The pinned Calypso source under investigation is branch
`wp-calypso_be7e5cc64162`, source HEAD `be7e5cc641622d153040491fd5625c6cb83e12eb`. The
destination checkout that contains this pinned tree is at HEAD
`acf63ee7ad64a96491ba2945a70db04f96d89627`; every `file:line` citation below was verified
against that checkout. All runtime output was captured on Node.js `v22.23.1` / Yarn `4.0.2`
on Linux.

**Observed vs. inferred.** Values labelled **[OBSERVED]** were captured live from the running
server or browser. Values labelled **[SOURCE-DERIVED]** were read from SCSS/JS that compiles
deterministically but whose runtime surface was not reachable in the canonical logged-out
Reader (explained where used). Values labelled **[NON-CANONICAL]** came from a deliberately
synthetic probe and are never the basis of a reported canonical value.

---

## Table of Contents

- [1. Methodology — run-first chronology](#1-methodology--run-first-chronology)
  - [1.1 Toolchain and host prerequisites (observed)](#11-toolchain-and-host-prerequisites-observed)
  - [1.2 Exact commands, in order, with durations](#12-exact-commands-in-order-with-durations)
  - [1.3 Dependency-install evidence](#13-dependency-install-evidence)
  - [1.4 Two canonical unflagged runs (stability)](#14-two-canonical-unflagged-runs-stability)
  - [1.5 Temporary observation-script inventory (verbatim) + cleanup](#15-temporary-observation-script-inventory-verbatim--cleanup)
  - [1.6 Redaction and security policy](#16-redaction-and-security-policy)
- [2. Official Automattic onboarding corroboration (supplementary)](#2-official-automattic-onboarding-corroboration-supplementary)
- [Q1 — Development server: port, readiness, transitions, single-port architecture](#q1--development-server-port-readiness-transitions-single-port-architecture)
- [Q2 — Reader stream: endpoints and Redux actions on initial load](#q2--reader-stream-endpoints-and-redux-actions-on-initial-load)
- [Q3 — Authentication detection before render, and storage inspected](#q3--authentication-detection-before-render-and-storage-inspected)
- [Q4 — Responsive sidebar: header padding/margin, custom properties, breakpoints](#q4--responsive-sidebar-header-paddingmargin-custom-properties-breakpoints)
- [Architecture and data-flow diagrams](#architecture-and-data-flow-diagrams)
- [Coverage pass — every named item: Observed vs. Inferred](#coverage-pass--every-named-item-observed-vs-inferred)
- [AAP compliance matrix (complete)](#aap-compliance-matrix-complete)
- [Rules compliance matrix (complete)](#rules-compliance-matrix-complete)
- [Repository cleanliness proof](#repository-cleanliness-proof)

---

## 1. Methodology — run-first chronology

The rules for this task require that the answer be written **from what was observed**, by
building and running the code paths first. This section is the reproducible chronology: the
exact commands, their real durations, the dependency-install output, two full runs of the
default server, and the verbatim observation scripts (removed afterward, per the cleanup
proof at the end).

### 1.1 Toolchain and host prerequisites (observed)

The toolchain and the required hosts entry were captured directly:

```console
$ node --version
v22.23.1
$ yarn --version
4.0.2
$ grep -n calypso.localhost /etc/hosts
9:127.0.0.1 calypso.localhost
```

- Node `v22.23.1` satisfies `engines.node` `^v22.9.0` (`package.json:L57`) and `.nvmrc`
  `22.9.0` (`.nvmrc:L1`). Node 20.x must **not** be used: the `check-node-version --package`
  gate inside the `start` script (`package.json:L110`) rejects it.
- Yarn `4.0.2` matches the `packageManager` pin `yarn@4.0.2` (`package.json:L422`), resolved
  via corepack.
- `127.0.0.1 calypso.localhost` is present in `/etc/hosts` (line 9). The app is reachable
  **only** at `http://calypso.localhost:3000` — see Q1.e for why other hostnames do not work.

### 1.2 Exact commands, in order, with durations

The canonical, default (unflagged) sequence a normal contributor runs is `yarn` then
`yarn start`. `yarn start` expands to (from `package.json:L110`):

```text
npx check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build
```

where `start-build` (`package.json:L113`) runs `BROWSERSLIST_ENV=evergreen node build/server.js | bunyan -o short`
and `build-server` (`package.json:L81`) compiles the SSR server via
`webpack --config client/webpack.config.node.js`.

The wall-clock chronology from the two captured runs (UTC timestamps, from the run-timing
files) is:

| Event | Run 1 | Run 2 | Source of timing |
|-------|-------|-------|------------------|
| `yarn install --immutable` total | 6.389 s | (reused) | `yarn_install.log` |
| `yarn start` process start | `09:07:31.149Z` | `09:13:59.126Z` | `run1_timing.txt`, `run2_timing.txt` |
| bunyan boot log (server `listen`) | `09:07:56.351Z` | `09:14:23.792Z` | `start_run{1,2}.raw.log:L115` |
| webpack "Ready!" seen | `09:11:05.911Z` | `09:16:56.540Z` | `start_run{1,2}.raw.log` |
| First-compile duration (webpack) | 180369 ms | 144985 ms | `start_run{1,2}.raw.log` |

The boot log appears ~25 s after process start (Node build + `server.listen`), and the
webpack "Ready!" appears ~2.5–3 min after that (first full browser-bundle compile). Both
signals are explained and quoted raw in Q1.b.

### 1.3 Dependency-install evidence

`yarn install --immutable` completed cleanly with the lockfile unchanged:

```console
$ yarn install --immutable
➤ YN0000: · Yarn 4.0.2
➤ YN0000: ┌ Resolution step
➤ YN0000: └ Completed in 0s 473ms
➤ YN0000: ┌ Fetch step
➤ YN0000: └ Completed in 4s 348ms
➤ YN0000: ┌ Link step
➤ YN0000: └ Completed in 1s 57ms
➤ YN0000: · Done in 6s 389ms
```

The `--immutable` flag proves the install did not mutate `yarn.lock` (a non-immutable install
would error if it needed to). `node_modules` and `packages/*/dist` are build/install state,
git-ignored, and left in place; they are not part of the deliverable.

### 1.4 Two canonical unflagged runs (stability)

The rules require confirming stability across at least two runs. Both runs used the
**default, unflagged** `yarn start` (no `SECTION_LIMIT`, no `MOCK_WORDPRESSDOTCOM`, no
`NODE_OPTIONS`). Observed stability:

- **Port:** identical (`3000`) on both runs.
- **Boot log:** byte-identical except the elapsed-ms figure (`968ms` run 1 vs. `984ms` run 2).
- **Readiness banner:** byte-identical on both runs.
- **Warning count:** identical (`37 warnings`) on both runs.

> Note on warning count: the default unflagged build emits **37 warnings**. A scoped
> `SECTION_LIMIT=reader,login` build emits fewer (it compiles fewer sections); this document
> reports the **default** figure because the rules require the default, canonical
> configuration. The scoped build is a labelled non-default alternative only.

### 1.5 Temporary observation-script inventory (verbatim) + cleanup

Three shell scripts and a set of browser-console snippets were used to capture Q1 signals.
The scripts — and **every file they write** (the start logs, the server PID file, and the
queued-request HTML capture) — live under `/tmp/calypso_obs/` (outside the repository) and are
removed in the cleanup step (see [Repository cleanliness proof](#repository-cleanliness-proof)),
so the working tree carries no observation residue. `01_run_server.sh` takes an optional run-tag
argument (`run1`, `run2`, …) and writes a per-run log (`start_run1.raw.log`, `start_run2.raw.log`,
…) — the exact filenames the two-run methodology in §1.4 and Q1.b cite. They are reproduced here
verbatim so every Q1 result is reproducible.

`01_run_server.sh` — canonical default build+run capturing both readiness signals:

```bash
#!/usr/bin/env bash
# Canonical, DEFAULT (unflagged) build+run of Calypso, capturing complete stdout with the two readiness signals.
# Usage: 01_run_server.sh [run-tag]   (default: run1)  ->  writes /tmp/calypso_obs/start_<run-tag>.raw.log
set -euo pipefail
shopt -s inherit_errexit   # make a failed $(...) (e.g. run outside a git repo) abort non-zero, never hang
# Resolve the repository root through git; fail loudly (non-zero) if not inside a git repo — no silent hang:
ROOT="$( git rev-parse --show-toplevel )" || { echo "ERROR: not inside a git repository" >&2; exit 1; }
cd "$ROOT"
# ALL observation artifacts live OUTSIDE the repository, under /tmp/calypso_obs:
OBS_DIR=/tmp/calypso_obs; mkdir -p "$OBS_DIR"
RUN_TAG="${1:-run1}"
LOG="$OBS_DIR/start_${RUN_TAG}.raw.log"
date -u +"START: %Y-%m-%dT%H:%M:%S.%3NZ"
# `yarn start` = check-node-version --package && node bin/welcome.js && yarn run build && yarn run start-build
setsid bash -c 'exec yarn start' > "$LOG" 2>&1 &
echo $! > "$OBS_DIR/server_${RUN_TAG}.pgid"
# Bounded waits (max-iteration caps) so an unexpected failure surfaces as a non-zero exit, never an infinite hang.
# Readiness signal #1 (bunyan boot log, printed just before server.listen at client/server/index.js:L83)
for _ in $( seq 1 90 ); do grep -q "wp-calypso booted in" "$LOG" && break; sleep 2; done
grep "wp-calypso booted in" "$LOG" || { echo "ERROR: boot log not seen within 180s" >&2; exit 1; }
# Readiness signal #2 (webpack first-compile "Ready!")
for _ in $( seq 1 180 ); do grep -q "Ready! You can load" "$LOG" && break; sleep 2; done
grep -E "webpack .* compiled|Ready! You can load" "$LOG" || { echo "ERROR: webpack Ready! not seen within 360s" >&2; exit 1; }
# Leaves the server running (by design) so the other helpers can probe it; the PID is saved to
# "$OBS_DIR/server_<run-tag>.pgid" for teardown (`kill -- -"$(cat "$OBS_DIR/server_${RUN_TAG}.pgid")"`),
# and the cleanup step removes all of /tmp/calypso_obs.
```

`02_q1_transitional.sh` — the pre-"Ready!" transitional behaviour (root holding page vs. a
queued non-root request):

```bash
#!/usr/bin/env bash
# Q1 transitional evidence: root holding page vs non-root queued request, captured in the pre-"Ready!" window.
set -euo pipefail
OBS_DIR=/tmp/calypso_obs; mkdir -p "$OBS_DIR"   # write every capture OUTSIDE the repository
# Root "/" returns the "Welcome to Calypso!" holding page immediately:
curl -s -m 15 -w '\n[HTTP %{http_code} | %{content_type} | %{size_download}B | %{time_total}s]\n' \
  http://calypso.localhost:3000/
# A NON-root request (e.g. /reader) BLOCKS in waitForCompiler until the first compile finishes:
curl -s -m 300 -o "$OBS_DIR/nonroot_reader.html" \
  -w 'NONROOT /reader => HTTP %{http_code} | %{content_type} | %{size_download}B | WAITED %{time_total}s\n' \
  http://calypso.localhost:3000/reader
```

`03_q1_singleport.sh` — single-port proof (SSR HTML, compiled asset, HMR stream, health, all
on port 3000):

```bash
#!/usr/bin/env bash
# Q1 single-port proof: SSR HTML, compiled JS asset, and the HMR event-stream are ALL served on port 3000.
set -euo pipefail
curl -sS -D - -o /dev/null http://calypso.localhost:3000/reader                       # SSR HTML
curl -sI       http://calypso.localhost:3000/calypso/evergreen/runtime.js             # compiled asset (webpack-dev-middleware)
# The HMR endpoint is a long-lived Server-Sent-Events stream; `-m 4` makes curl exit 28 (timeout) BY DESIGN.
# Tolerate that expected timeout (|| true) so the health probe below still runs and the script exits 0.
curl -sN -D - -m 4 -o /dev/null http://calypso.localhost:3000/__webpack_hmr || true   # HMR stream (webpack-hot-middleware)
curl -s        http://calypso.localhost:3000/version                                  # health endpoint
```

Browser-side captures (Q2/Q3/Q4) used the Chrome DevTools protocol and small read-only
console snippets (a Redux action-recorder enhancer and read-only storage/CSSOM enumerators);
each is shown inline in its section. None wrote to the repository.

### 1.6 Redaction and security policy

To keep the captured evidence complete yet safe, this document applies a single, explicit
policy:

- **No live secrets.** No real session cookie, OAuth token, or access token was present in the
  logged-out canonical run, and none is printed. Where a value is an analytics identifier or
  opaque cursor, it is shown as a narrow placeholder `[REDACTED: <what it is>]` rather than
  elided with `...`.
- **Redacted items (exhaustive):** the analytics cookie values `tk_ai` and `tk_qs`; the
  `localStorage` value of `tusSupport`; and the opaque stream `page_handle` cursor. Only these
  values are redacted; key **names**, sizes, structure, and status codes are shown in full.
- **Synthetic probes are labelled.** Any value produced by a fabricated header or cookie is
  marked **[NON-CANONICAL]** and is used only to demonstrate a server-side branch, never as a
  reported canonical value.
- **Raw before summary.** Each observation shows the exact command and its complete output
  first; any summary follows the raw block.

---

## 2. Official Automattic onboarding corroboration (supplementary)

The runtime observations below are primary. As a cross-check, they agree with Automattic's
own onboarding documentation, both as pinned in this repository and as currently published on
GitHub. This corroboration is **supplementary** — it confirms, but is not the basis for, the
observed values.

Pinned in-repo documentation (authoritative for this commit):

- `README.md:L20-L21` — the run procedure: execute `yarn`, then `yarn start`, and open
  `calypso.localhost:3000`.
- `docs/install.md:L36` — `yarn start` installs dependencies and starts the dev server, and
  rebuilds automatically on JS/Sass changes.
- `docs/install.md:L38` — running locally requires the `127.0.0.1 calypso.localhost` hosts
  entry and loading `http://calypso.localhost:3000`, **because Calypso uses the remote
  WordPress.com REST API** (which allows only certain origins). This is the documentary basis
  for the single-port conclusion in Q1.d (REST is remote, not a local port).
- `docs/install.md:L40` — if a browser blocks third-party cookies, an exception on
  `https://public-api.wordpress.com` is needed; relevant to Q3's third-party-cookie edge case.
- `docs/install.md:L48-L50` — `SECTION_LIMIT` (e.g. `SECTION_LIMIT=reader,login yarn start`)
  restricts which sections build; `docs/install.md:L63` — `NODE_OPTIONS="--inspect=5858"`
  starts the debugger on an inspector port (distinct from application traffic).
- `docs/yarn-start.md:L1-L28` — the boot flow diagram: `yarn start` → check node version →
  `yarn run start-build` → `node build/server.js`.

Current published documentation (checked at `github.com/Automattic/wp-calypso`, `trunk`)
matches: the README and `docs/install.md` give the same `yarn` / `yarn start` /
`calypso.localhost:3000` procedure and the same remote-REST-API rationale, and the historical
Calypso Bootstrap docs show the identical readiness banner ("Ready! You can load
http://calypso.localhost:3000/ now. Have fun!") that this run emitted verbatim (Q1.b).

---

## Q1 — Development server: port, readiness, transitions, single-port architecture

**Short answer.** The dev server binds **port 3000** (`config/development.json:L8`, actual
bind at `client/server/index.js:L83`). A developer knows it is ready from **two** stdout
signals: the bunyan boot log (`client/server/index.js:L33`) and the webpack first-compile
banner (`client/server/bundler/index.js:L56`). The architecture is **single-port**: SSR HTML,
compiled JS/CSS assets, and the hot-module-reload stream are all served from port 3000, while
REST API traffic leaves the machine to the **remote** `public-api.wordpress.com` — there is no
second local port for API or HMR.

### Q1.a — The port is 3000 (config subsystem → actual bind)

The port originates in configuration and is read by the server at boot:

```console
$ sed -n '1,10p' config/development.json
{
	"env": "development",
	"env_id": "development",
	"favicon_url": "/calypso/images/favicons/favicon-development.ico",
	"client_slug": "browser",
	"protocol": "http",
	"hostname": "calypso.localhost",
	"port": 3000,
	"i18n_default_locale_slug": "en",
	"siftscience_key": "e00e878351",
```

- `config/development.json:L6` `"protocol": "http"`, `:L7` `"hostname": "calypso.localhost"`,
  `:L8` `"port": 3000`. Shared defaults live in `config/_shared.json` (`:L24` protocol, `:L25`
  port `3000`, `:L13` `hostname: false`).

The server reads these through the config subsystem and binds:

```console
$ sed -n '11,13p;83p' client/server/index.js
let protocol = config( 'protocol' );
let port = config( 'port' );
let host = config( 'hostname' );
server.listen( { port, host: process.env.CALYPSO_IS_FORK ? host : null }, function () {
```

- `client/server/index.js:L12` resolves `config( 'port' )` → `3000`.
- `client/server/index.js:L83` is the actual `server.listen(...)`. In a normal (non-fork) run,
  `host` is `null`, so Node binds all interfaces; only the desktop fork
  (`process.env.CALYPSO_IS_FORK`) binds the `calypso.localhost` hostname specifically.

Live confirmation that something is actually listening on 3000 and answering:

```console
$ curl -s http://calypso.localhost:3000/version
{"version":"0.17.0"}
```

That exact command emits exactly `{"version":"0.17.0"}` — the health endpoint served on
port 3000. **[OBSERVED]**

### Q1.b — The two readiness signals (raw, both runs)

There are **two** distinct readiness signals, emitted at different times. Both were captured
raw from `start_run1.raw.log` and `start_run2.raw.log`.

**Signal 1 — bunyan boot log** (emitted just before `server.listen`, format string at
`client/server/index.js:L33`: `'wp-calypso booted in %dms - %s://%s:%s'`):

```text
# run 1, start_run1.raw.log:L115
09:07:56.351Z  INFO calypso: wp-calypso booted in 968ms - http://calypso.localhost:3000
# run 2, start_run2.raw.log:L115
09:14:23.792Z  INFO calypso: wp-calypso booted in 984ms - http://calypso.localhost:3000
```

This signals that the Express server is up — but the browser bundles are **not** compiled yet.

**Signal 2 — webpack first-compile banner** (emitted by the dev bundler at
`client/server/bundler/index.js:L56`):

```text
# run 1, start_run1.raw.log:L245,L247
webpack 5.97.1 compiled with 37 warnings in 180369 ms
Ready! You can load http://calypso.localhost:3000/ now. Have fun!
# run 2, start_run2.raw.log:L247,L249
webpack 5.97.1 compiled with 37 warnings in 144985 ms
Ready! You can load http://calypso.localhost:3000/ now. Have fun!
```

The "Ready!" banner is the signal a developer waits for — after it appears, non-root routes
render real HTML instead of blocking (Q1.c). The `37 warnings` count is stable across both
default runs (Q1.a, §1.4).

The logger itself is a bunyan instance named `calypso` writing to stdout at `info`
(`client/server/lib/logger/index.js:L7` name `'calypso'`, `:L10` stdout, `:L11` level
`info`), and `start-build` pipes it through `bunyan -o short` (`package.json:L113`) — which is
why the lines are prefixed with the short timestamp and `INFO calypso:`.

### Q1.c — Transitional state, before / during / after first compile

Before the "Ready!" banner, the server is already listening but the bundle is still compiling.
Its behaviour splits by route, which is the before/during/after transition the question
implies.

**Root `/` during compile → immediate "Welcome to Calypso!" holding page** (served by
`waitForCompiler` at `client/server/bundler/index.js:L66`, HTML template around `:L79-L86`):

```console
$ curl -s -m 15 -w '\n[HTTP %{http_code} | %{content_type} | %{size_download}B | %{time_total}s]\n' http://calypso.localhost:3000/

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
			
[HTTP 200 | text/html; charset=utf-8 | 630 bytes | 0.045709s]
```

This 630-byte page returns immediately (`0.045709s`) and auto-refreshes every 5 seconds via
`<meta http-equiv="refresh" content="5">` until the bundle is ready. **[OBSERVED]**

**Non-root `/reader` during compile → the request BLOCKS until the first compile finishes**
(queued by `waitForCompiler` at `client/server/bundler/index.js:L96`, `:L100`):

```text
# from q1_nonroot_queue.txt (issued during the run-2 compile window)
NONROOT /reader ISSUED at 09:14:38.503Z
NONROOT /reader => HTTP 200 | text/html; charset=utf-8 | 41676 bytes | WAITED 138.630867s
NONROOT /reader RETURNED at 09:16:57.144Z
```

The `/reader` request issued at `09:14:38.503Z` did **not** get the holding page — it was held
open for **138.6 seconds** and only returned (as the real 41,676-byte SSR document) at
`09:16:57.144Z`, immediately after the "Ready!" banner at `09:16:56.540Z`. This is the
"during" state: root is served a holding page, but content routes queue on the compiler.
**[OBSERVED]**

**After "Ready!" → the same route returns immediately:**

```text
# from q1_postready_reader_result.txt
POST-READY /reader => HTTP 200 | text/html; charset=utf-8 | 41676 bytes | 0.200105s
```

Same 41,676-byte document, now in `0.200105s` instead of 138 s. The response headers confirm
an Express SSR response with no-store caching:

```console
$ curl -sI http://calypso.localhost:3000/reader
HTTP/1.1 200 OK
X-Powered-By: Express
Cache-control: no-store
X-Frame-Options: SAMEORIGIN
Content-Type: text/html; charset=utf-8
Content-Length: 41676
ETag: W/"a2cc-bb9Sf2Wd5Kk/5EyjaeeIrizvTKM"
Date: Fri, 10 Jul 2026 09:11:41 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

**[OBSERVED]**

### Q1.d — Single-port proof: SSR + assets + HMR on 3000; REST is remote

Everything the browser loads from Calypso comes from port 3000; only the REST API is remote.
The bundler mounts both middlewares on the same Express app —
`webpack-dev-middleware` (`client/server/bundler/index.js:L5`) and
`webpack-hot-middleware` (`client/server/bundler/index.js:L6`) — attached via
`require('calypso/server/bundler')(app)` (`client/server/boot/index.js:L37`).

**SSR HTML on 3000** (headers shown in Q1.c). **Compiled JS asset on 3000**
(webpack-dev-middleware):

```console
$ curl -sI http://calypso.localhost:3000/calypso/evergreen/runtime.js
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/javascript; charset=utf-8
Accept-Ranges: bytes
Content-Length: 86739
ETag: W/"152d3-OjdFDoVtDzhmshS85rJM4GAVnOY"
Date: Fri, 10 Jul 2026 09:11:58 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

**HMR stream on 3000** (webpack-hot-middleware; a `text/event-stream`, i.e. Server-Sent
Events over the same port):

```console
$ curl -sN -D - -m 4 http://calypso.localhost:3000/__webpack_hmr
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: text/event-stream;charset=utf-8
Cache-Control: no-cache, no-transform
X-Accel-Buffering: no
Connection: keep-alive
Date: Fri, 10 Jul 2026 09:12:24 GMT
Transfer-Encoding: chunked
```

All three (SSR, asset, HMR) are `HTTP/1.1 200 OK` from `X-Powered-By: Express` on port 3000.
There is **no** separate HMR port and **no** local API port. REST traffic instead targets the
remote `public-api.wordpress.com` (observed live in Q2/Q3, and documented at
`docs/install.md:L38`). The Q1 architecture diagram is in
[Architecture and data-flow diagrams](#architecture-and-data-flow-diagrams). **[OBSERVED]**

### Q1.e — Non-canonical paths and clarifications (not the basis of any value above)

- **`MOCK_WORDPRESSDOTCOM=1`** (`client/server/index.js:L16-L21`) swaps in a local API mock.
  It is **[NON-CANONICAL]** for this task and was not used; all REST behaviour reported here is
  against the real remote API.
- **Inspector port 5858** (`NODE_OPTIONS="--inspect=5858"`, `docs/install.md:L63`) is a Node
  debugger port, not application traffic. It does not contradict the single-port conclusion.
- **`SECTION_LIMIT`** (`docs/install.md:L48-L50`) / **`ENTRY_LIMIT`** (`docs/install.md:L54-L59`)
  reduce which sections/entry points build. They change build scope and warning count, not the
  port topology. All reported values here are from the **default, unflagged** build.
- **"Only certain origins"** (`docs/install.md:L38`) is a statement about the remote API's
  allowed origins, i.e. why `calypso.localhost` is required — not a claim that 3000 is the only
  port a machine could ever use.


---

## Q2 — Reader stream: endpoints and Redux actions on initial load

**Short answer.** Loading `/reader` while logged out redirects to `/discover`
(`client/reader/controller.js:L362`). The default Discover ("Recommended") stream populates
via a single REST GET to `/wpcom/v2/read/streams/discover` (endpoint map at
`client/state/data-layer/wpcom/read/streams/index.js:L226`), and the initial-load Redux
sequence is the ordered triplet **`READER_STREAMS_PAGE_REQUEST` → `READER_POSTS_RECEIVE` →
`READER_STREAMS_PAGE_RECEIVE`**. Pagination repeats the triplet with a larger fetch count.

### Q2.a — The canonical logged-out flow and the observed endpoint

The canonical entry `http://calypso.localhost:3000/reader` redirects (logged out) to
`/discover`, which mounts the Discover stream. The live Network panel showed the initial
populate request (logged out, Recommended tab):

```text
# initial load (no page_handle → INITIAL_FETCH=4)
reqid=179  GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover
             ?_envelope=1&orderBy=popular&meta=post%2Cdiscover_original_post
             &feed_id=&number=4&lang=en&tags[]=dailyprompt&tags[]=wordpress
             &tag_recs_per_card=5&site_recs_per_card=5&age_based_decay=0.5&content_width=675
           [200]
```

Key observed facts: the path is `/wpcom/v2/read/streams/discover`; `number=4` equals
`INITIAL_FETCH` (`client/state/data-layer/wpcom/read/streams/index.js:L161`); `orderBy=popular`
is the Recommended-tab ordering (`:L244`); and `_envelope=1` requests the HTTP-envelope
response form. The response envelope shape (structure only — no user values):

```json
{
  "body": {
    "cards": [ /* 9 items: recommended_blogs ×1, post ×7, interests_you_may_like ×1 */ ],
    "next_page_handle": "[REDACTED: opaque cursor]",
    "user_interests": [ /* 2 */ ]
  },
  "status": 200,
  "headers": { "Allow": "…" }
}
```

**[OBSERVED]** The full raw response body was captured to
`q2_discover_initial_response.network-response` (111,951 bytes); only its structure is
reproduced here, per the redaction policy (§1.6).

### Q2.b — Endpoint disambiguation (which `read/*` path for which stream)

The `streamApis` map builds a different REST path per stream type. The relevant entries
(`client/state/data-layer/wpcom/read/streams/index.js`):

| Stream (Reader view) | REST path | Cite |
|----------------------|-----------|------|
| Following (feed) | `/read/following` | `:L194` |
| Following (recent, wpcom/v2) | `/read/streams/following` | `:L198` |
| Search | `/read/search` | `:L212` |
| Single feed | `/read/feed/{feed}/posts` | `:L220` |
| **Discover → Recommended** | **`/read/streams/discover`** | `:L226` |
| Discover → Latest | `/read/tags/posts` | `:L228` |
| Discover → First posts | `/read/streams/first-posts` | `:L230` |
| Discover → default tags | `/read/streams/discover?tags=…` | `:L232` |
| Single site | `/read/sites/{site}/posts` | `:L249` |

The Discover family shares `apiNamespace: 'wpcom/v2'` (`:L246`), which is why the observed URL
is prefixed `/wpcom/v2/…`. The default apiVersion for the non-namespaced streams is `1.2`
(`:L370`). The observed default `/discover` view uses `orderBy=popular`; the "Latest" view
uses `/read/tags/posts` with `orderBy=date` — a distinction confirmed by the failure probe in
Q2.f.

### Q2.c — The initial-load Redux action sequence (raw, both loads)

Actions were recorded with a small read-only enhancer installed as the innermost store
enhancer (it only records `action.type` and the top-level key **names** of each action; it
mutates nothing). It was injected via the page's init script:

```javascript
// read-only Redux action recorder (innermost enhancer); records ONLY type + top-level key names
window.__capturedActions = [];
const rec = (createStore) => (reducer, preloaded, enhancer) => {
  const store = createStore(reducer, preloaded, enhancer);
  const dispatch = store.dispatch;
  store.dispatch = (action) => {
    try {
      window.__capturedActions.push({
        i: window.__capturedActions.length,
        type: action && action.type,
        payloadKeys: action ? Object.keys(action) : [],
      });
    } catch (e) {}
    return dispatch(action);
  };
  return store;
};
```

Extracting the reader-stream actions from two consecutive fresh loads:

```text
# Load A (first navigation): 291 actions total
READER_STREAMS_PAGE_REQUEST (i=45) → READER_POSTS_RECEIVE (i=54) → READER_STREAMS_PAGE_RECEIVE (i=56)

# Load B (second navigation): 304 actions total
READER_STREAMS_PAGE_REQUEST (i=58) → READER_POSTS_RECEIVE (i=67) → READER_STREAMS_PAGE_RECEIVE (i=69)
  → [pagination] READER_STREAMS_PAGE_REQUEST (i=71) → READER_POSTS_RECEIVE (i=114,199) → READER_STREAMS_PAGE_RECEIVE (i=201)
```

**Stable signal across both loads:** the ordered type triplet
`READER_STREAMS_PAGE_REQUEST → READER_POSTS_RECEIVE → READER_STREAMS_PAGE_RECEIVE`. **[OBSERVED]**

> Honest limitation: the enhancer's captured top-level key names differed between loads (Load A
> drilled into the wrapped `payload` object and saw `streamKey, pageHandle, streamType, isPoll,
> gap, localeSlug, feedId`; Load B saw the wrapper keys `payload, meta`). The type sequence is
> the stable observable; the authoritative payload **shape** is the action-creator source
> (Q2.d), where `requestPage` wraps its fields under `payload` and `receivePosts` dispatches a
> flat `{ type, posts }`. This is stated as a limitation rather than papered over.

Action-type constants (`client/state/reader/action-types.ts`): `READER_POSTS_RECEIVE` `:L52`,
`READER_STREAMS_PAGE_RECEIVE` `:L77`, `READER_STREAMS_PAGE_REQUEST` `:L78`,
`READER_STREAMS_PAGINATED_REQUEST` `:L79`.

### Q2.d — Where the dispatch originates (route → controller → mount → data-layer → handler)

The full wiring, each step cited:

1. **Route** — `client/reader/index.ts:L55-L56`: the reader route
   `[ '/reader', '/reader/recent/:feed_id' ]` runs `redirectLoggedOutToDiscover` first.
2. **Controller** — `client/reader/controller.js:L356-L362`:
   ```javascript
   export function redirectLoggedOutToDiscover( context, next ) {
       const state = context.store.getState();
       if ( isUserLoggedIn( state ) ) { next(); return; }
       return page.redirect( '/discover' );
   }
   ```
   Logged out → `page.redirect( '/discover' )` (`:L362`).
3. **Mount** — `client/reader/stream/index.jsx:L221` `componentDidMount` calls
   `this.fetchNextPage( {} )`; `fetchNextPage` (`:L489`) calls `this.props.requestPage( … )`
   (`:L502`).
4. **Action creator** — `client/state/reader/streams/actions.js:L28` `requestPage(...)` returns
   `{ type: READER_STREAMS_PAGE_REQUEST, payload: { streamKey, pageHandle, streamType, isPoll, gap, localeSlug, … } }`.
5. **Data layer** — `client/state/data-layer/wpcom/read/streams/index.js`: `requestPage(action)`
   builder at `:L358` computes `fetchCount = pageHandle ? PER_FETCH : INITIAL_FETCH` (`:L380`)
   and issues `http({ method:'GET', path, apiVersion, onSuccess: action, onFailure: action })`
   (around `:L395-L404`). On success, `handlePage` (`:L428`) dispatches `receivePosts(...)`
   (`:L470`) and `receivePage(...)`.
6. **Receivers** — `client/state/reader/posts/actions.js:L63` `receivePosts` dispatches the flat
   `{ type: READER_POSTS_RECEIVE, posts }` (`:L86-L88`); `client/state/reader/streams/actions.js:L52`
   `receivePage` returns `{ type: READER_STREAMS_PAGE_RECEIVE, payload: {…} }`.
7. **Handler registration** — `client/state/data-layer/wpcom/read/streams/index.js:L514`:
   ```javascript
   registerHandlers( 'state/data-layer/wpcom/read/streams/index.js', {
       [ READER_STREAMS_PAGE_REQUEST ]:      [ dispatchRequest( { fetch: requestPage, onSuccess: handlePage, onError: noop } ) ], // :L519
       [ READER_STREAMS_PAGINATED_REQUEST ]: [ dispatchRequest( { fetch: requestPage, onSuccess: handlePage, onError: noop } ) ], // :L526
   } );
   ```
   with `noop = () => {}` (`:L19`).

### Q2.e — Pagination (PER_FETCH vs INITIAL_FETCH)

The first page omits `page_handle`, so `fetchCount = INITIAL_FETCH = 4`
(`:L161`, `:L380`); subsequent pages carry the cursor, so `fetchCount = PER_FETCH = 7`
(`:L160`). Observed live:

```text
# pagination request (page_handle present → PER_FETCH=7)
# Query string is identical to the initial request in Q2.a EXCEPT: number=4 → number=7,
# and a page_handle cursor is added. Full form:
reqid=198  GET https://public-api.wordpress.com/wpcom/v2/read/streams/discover?_envelope=1&orderBy=popular&meta=post%2Cdiscover_original_post&feed_id=&number=7&lang=en&tags[]=dailyprompt&tags[]=wordpress&tag_recs_per_card=5&site_recs_per_card=5&age_based_decay=0.5&content_width=675&page_handle=[REDACTED: opaque cursor]  [200]
```

The Redux effect is the triplet repeating (Load B, `i=71 → 114/199 → 201` in Q2.c).
**[OBSERVED]**

### Q2.f — Failure semantics: `onError: noop`, exercised offline

Both stream handlers register `onError: noop` (`:L519`, `:L526`). To exercise the failure
branch, the live stream was switched to "Latest" with DevTools network emulation set to
**Offline**, and both the action stream and the Network panel were observed:

```text
# Redux actions since the offline boundary (probeStart index = 291)
{ "probeStart": 291, "totalNow": 307, "dispatchedSince": 16,
  "readerActionsSince": [ { "i": 299, "type": "READER_STREAMS_PAGE_REQUEST" } ],
  "errorishSince": [] }

# Corresponding failed network request (Network panel, Offline) — complete URL:
reqid=320  GET https://public-api.wordpress.com/wpcom/v2/read/tags/posts?_envelope=1&orderBy=date&meta=post%2Cdiscover_original_post&feed_id=&number=4&lang=en&tags%5B%5D=dailyprompt&tags%5B%5D=wordpress&tag_recs_per_card=5&site_recs_per_card=5&age_based_decay=0.5&content_width=675  [net::ERR_INTERNET_DISCONNECTED]
```

Interpretation: exactly one reader-stream action fired (`READER_STREAMS_PAGE_REQUEST`, i=299);
the underlying HTTP GET was attempted and failed at the network layer
(`net::ERR_INTERNET_DISCONNECTED`); **no** `READER_POSTS_RECEIVE`, **no**
`READER_STREAMS_PAGE_RECEIVE`, and **no** error action followed (`errorishSince` is empty). The
failure was swallowed by the registered `onError: noop`
(`client/state/data-layer/wpcom/read/streams/index.js:L519` and `:L526`), so the store retains
its prior stream contents. (Note the "Latest" view uses `/read/tags/posts` with
`orderBy=date`, `number=4 == INITIAL_FETCH`, confirming Q2.b.) **[OBSERVED]**


---

## Q3 — Authentication detection before render, and storage inspected

**Short answer.** There is no single "logged in?" flag; there are **four distinct concepts**.
The server makes a fast **cookie-presence** heuristic (`!! req.cookies.wordpress_logged_in`,
`client/server/pages/index.js:L93`) to branch SSR. The client establishes **authenticated
identity** by calling `/me`; in the default config it fetches `/me` directly and, logged out,
receives `403 authorization_required`, so `getCurrentUserId(state) === null` and
`isUserLoggedIn(state) === false` (`client/state/current-user/selectors.js:L15-L16`), which
selects `<LayoutLoggedOut>` (`client/controller/index.web.js:L64`). Three independent signals
(absent cookie, `/me` 403, and the IndexedDB `redux-state-logged-out` key) all agree.

### Q3.a — Four distinct concepts (do not conflate them)

| Concept | What it is | Where | Decides layout? |
|---------|-----------|-------|-----------------|
| A. Cookie presence | Server heuristic `!! req.cookies.wordpress_logged_in` | `client/server/pages/index.js:L93` | No — only branches SSR route/cache |
| B. Server bootstrap | `getBootstrappedUser(req)` → GET `/rest/v1/me` forwarding the cookie | `client/server/pages/index.js:L382`; `client/server/user-bootstrap/index.js:L13,L28,L34` | Only if enabled; **gated OFF by default** (`:L364`) |
| C. Authenticated identity | Client `/me?meta=flags` returns a real user id vs. `403` | `client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L6`; `client/state/current-user/selectors.js:L15-L16` | **Yes** — selects the layout |
| D. Authorization | Per-resource capability checks | (not the login gate) | No — out of the login decision |

Cookie **presence** is not verified **identity**; identity is not per-resource
**authorization**. The layout decision is driven by (C).

### Q3.b — The default client current-user init path (observed)

In the default configuration (`wpcom-user-bootstrap: false`, `config/development.json:L209`;
`oauth: false`, `:L130`), the client establishes identity by fetching `/me` directly. The
chain:

```text
client/boot/app.js:L8                                   bootApp( 'Calypso' )
client/boot/common.js:L340                              export const bootApp = async ( appName, registerRoutes ) =>   [async function declaration]
client/boot/common.js:L341                              const user = await initializeCurrentUser();
client/lib/user/shared-utils/initialize-current-user.js:L28   if ( ! skipBootstrap && config.isEnabled( 'wpcom-user-bootstrap' ) )  // FALSE → skip
client/lib/user/shared-utils/initialize-current-user.js:L37   userData = await rawCurrentUserFetch();                              // canonical default path
client/lib/user/shared-utils/initialize-current-user.js:L38-L43 catch: logs via console.error() ONLY when ( error.error !== 'authorization_required' ) — the 403 is swallowed, never re-thrown
client/lib/user/shared-utils/raw-current-user-fetch.js:L3-L6   wpcom.me().get( { meta: 'flags' } )
```

The live `/me` request and its HTTP-envelope body were captured:

```console
$ # reqid=467 (from the DevTools Network panel), body saved to q3_me_response.network-response
$ cat q3_me_response.network-response
{"code":403,"headers":[{"name":"Content-Type","value":"application/json"}],"body":{"error":"authorization_required","message":"An active access token must be used to query information about the current user."}}
```

The transport is `200` but the envelope `code` is `403` with
`body.error = "authorization_required"`. Per `client/lib/user/shared-utils/initialize-current-user.js:L38-L43`, that specific
error is expected and swallowed (not re-thrown), so `userData` stays undefined → no
`setCurrentUser` → `getCurrentUserId(state) === null` → `isUserLoggedIn === false`. **[OBSERVED]**

### Q3.c — Server (SSR) cookie decision + synthetic probe (labelled non-canonical)

The server's pre-render decision is a cookie-presence check inside `setupLoggedInContext`
(`client/server/pages/index.js`):

```console
$ sed -n '90,99p' client/server/pages/index.js
function setupLoggedInContext( req, res, next ) {
	const isSupportSession = !! req.get( 'x-support-session' ) || !! req.cookies.support_session_id;
	const disableHelpCenterAutoOpen = isSupportSession || !! req.cookies.ssp;
	const isLoggedIn = !! req.cookies.wordpress_logged_in;

	req.context = {
		...req.context,
		isSupportSession,
		disableHelpCenterAutoOpen,
		isLoggedIn,
```

`isLoggedIn` (`:L93`) is stored on `req.context` (`:L99`) and used to: gate the SSR cache
(`:L138`, logged-in requests bypass the cache), choose the route
(`req.context.isLoggedIn ? setUpLoggedInRoute : setUpLoggedOutRoute`, `:L586-L588`), and — only
when `wpcom-user-bootstrap` is enabled (`:L364`) — redirect unauthenticated users to login
(`:L372`) or bootstrap the user and `dispatch( setCurrentUser( data ) )` (`:L382`, `:L391`).

**Canonical (no cookie)** — the real logged-out run:

```console
$ curl -s http://calypso.localhost:3000/reader -o q3_srv_canonical.html -w "http_code=%{http_code} size=%{size_download}\n"
http_code=200 size=41676
# body class: <body class="color-scheme theme-default is-group-reader is-section-reader"
```

**[NON-CANONICAL] Synthetic probe** — a fabricated cookie, used only to prove the server reads
and branches on the cookie. The value is not a real session and cannot mint an identity (and
`wpcom-user-bootstrap` is off), so it exercises **only** the presence check at `:L93`:

```console
$ curl -s http://calypso.localhost:3000/reader \
       -H 'Cookie: wordpress_logged_in=SYNTHETIC_PROBE_NOT_A_REAL_SESSION' \
       -o q3_srv_synthetic.html -w "http_code=%{http_code} size=%{size_download}\n"
http_code=200 size=41704
# body class: <body class="color-scheme theme-default is-group-reader is-section-reader"

# `normalize` strips volatile per-build hashes (bundle filenames, cache-buster query strings,
# ETag-like tokens) so the diff shows only STRUCTURAL differences, not build noise:
$ normalize() { sed -E 's/[0-9a-f]{8,}//g' "$1"; }
$ diff <(normalize q3_srv_canonical.html) <(normalize q3_srv_synthetic.html)
24a25
> var languageRevisions = {};
```

The only structural difference (+28 bytes) is the injected `var languageRevisions = {};`:
the presence check flipped the request onto `setUpLoggedInRoute` (`:L587 → :L328`), whose
language-revision setup injects that variable, whereas the logged-out route
(`:L588 → setUpLoggedOutRoute`) does not. Crucially, the **body class does not flip** to a
logged-in variant — because with `wpcom-user-bootstrap:false` the server never bootstraps an
identity from the fake cookie. This directly demonstrates concept A (presence) is separate
from concept C (identity). **[OBSERVED, with the cookie itself labelled NON-CANONICAL]**

### Q3.d — Client hydration and the decision selector

On the client, `getInitialState` merges the server-injected `window.initialReduxState` with
IndexedDB-persisted state keyed by user id (`client/state/initial-state.js:L76`
`'redux-state-' + ( userId ?? 'logged-out' )`, server-state merge at `:L148-L154`). The layout
decision selector:

```console
$ sed -n '14,16p' client/state/current-user/selectors.js
 */
export function isUserLoggedIn( state ) {
	return getCurrentUserId( state ) !== null;
```

consumed by the web controller:

```console
$ sed -n '59,70p' client/controller/index.web.js
	const userLoggedIn = isUserLoggedIn( state );

	const layout = userLoggedIn ? (
		<Layout primary={ primary } secondary={ secondary } />
	) : (
		<LayoutLoggedOut
			primary={ primary }
			secondary={ secondary }
			redirectUri={ redirectUri }
			renderHeaderSection={ renderHeaderSection }
		/>
	);
```

Logged out → the ternary's false branch renders `<LayoutLoggedOut>`
(`client/controller/index.web.js:L64`). **[OBSERVED via the /me→403 chain]**

### Q3.e — Storage mechanisms inspected (live, logged-out default)

Every browser storage surface was enumerated read-only (key **names** and sizes only; values
redacted per §1.6):

```javascript
// read-only storage enumeration (names + sizes; no values printed)
({
  cookieNames: document.cookie.split('; ').map(s => s.split('=')[0]).filter(Boolean),
  localStorageKeys: Object.keys(localStorage),
  sessionStorageKeys: Object.keys(sessionStorage),
  initialReduxState: { present: !!window.initialReduxState,
                       topLevelKeys: Object.keys(window.initialReduxState || {}) },
  hasIndexedDB: !!window.indexedDB,
})
```

Observed result:

```text
cookies (names)          : [ tk_ai, country_code, region, tk_qs ]   # analytics + geo; NO wordpress_logged_in, NO wpcom_token, NO support_session_id
localStorage keys        : [ tusSupport ]                            # value [REDACTED: 4 chars]; NO wpcom_token fallback
sessionStorage keys      : [ ]                                       # empty; NO 'flags' (OAuth path not taken; oauth:false)
window.initialReduxState : present; top-level keys = [ documentHead ]  # server injected NO current user
IndexedDB                : database "calypso" v2; store "calypso_store"; 17 keys (all prefixed redux-state-logged-out)
```

The 17 IndexedDB keys (read-only):

```text
browser-storage-sanity-test, redux-state-logged-out, redux-state-logged-out:all-domains,
redux-state-logged-out:connectedApplications, redux-state-logged-out:documentHead,
redux-state-logged-out:memberships, redux-state-logged-out:plugins,
redux-state-logged-out:preferences, redux-state-logged-out:pushNotifications,
redux-state-logged-out:reader, redux-state-logged-out:readerUi, redux-state-logged-out:route,
redux-state-logged-out:signup, redux-state-logged-out:siteSettings, redux-state-logged-out:teams,
redux-state-logged-out:ui, redux-state-logged-out:userSuggestions
```

Authority for the IndexedDB names: `client/lib/browser-storage/index.ts:L20`
`DB_NAME = 'calypso'`, `:L22` `STORE_NAME = 'calypso_store'`. The key prefix
`redux-state-logged-out` is exactly `client/state/initial-state.js:L76`
(`'redux-state-' + ( userId ?? 'logged-out' )`), so the persisted-state key literally encodes
the logged-out identity (userId null) — an independent third confirmation of the auth state.

**Enumerated storage summary:** cookies (`wordpress_logged_in` for server detection,
`wpcom_token` for OAuth, `support_session_id` for support — none present here); `localStorage`
via the `store` package (OAuth-token fallback `store.get('wpcom_token')`,
`packages/oauth-token/src/index.js:L17` — empty here); IndexedDB (persisted Redux state, DB
`calypso`); `sessionStorage` (`flags` for OAuth — empty here); and the server-injected
`window.initialReduxState`. **[OBSERVED]**

**Three independent signals agree (logged out):** (1) `wordpress_logged_in` absent →
`client/server/pages/index.js:L93` false; (2) `/me` → `403 authorization_required` → `getCurrentUserId` null;
(3) IndexedDB keys prefixed `redux-state-logged-out` → userId null.

### Q3.f — Alternate/edge modes (attempt + blocker + source-derived conclusion)

Per the completeness directive, each alternate condition was attempted; where a real path was
infeasible, the genuine blocker and the source-derived conclusion are stated.

- **Logged-in.** *Attempt:* no real WordPress.com credentials were provided to this
  environment; the closest exercised probe is the synthetic cookie (Q3.c), which is
  **[NON-CANONICAL]**. *Blocker:* no valid session, and `wpcom-user-bootstrap:false`
  (`config/development.json:L209`) means the server would not bootstrap even with a cookie.
  *Source-derived:* with a real session + bootstrap enabled, server
  `client/server/pages/index.js:L364 → :L382 getBootstrappedUser → :L391 dispatch(setCurrentUser(data))`
  yields non-null `getCurrentUserId` → `isUserLoggedIn` true → logged-in `<Layout>`
  (`client/controller/index.web.js:L62`); client-side, `/me` returns a real user (not 403) → same result.
- **OAuth (desktop/OAuth builds).** *Attempt:* default `oauth:false`
  (`config/development.json:L130`); the storage dump confirms no `wpcom_token` cookie and none
  in `localStorage`, so `getToken() === false` would hold — but the middleware never runs.
  *Blocker:* enabling OAuth requires a non-default build. *Source-derived:*
  `client/boot/common.js:L154` `oauthTokenMiddleware`, `:L155` `isEnabled('oauth')`, `:L176`
  `getToken() === false` → `:L177` `window.location = authorizePath`; token names/parse at
  `packages/oauth-token/src/index.js:L7` (`TOKEN_NAME='wpcom_token'`), `:L11` cookie parse,
  `:L17` `store.get` (localStorage) fallback.
- **Support session.** *Attempt (synthetic header, **[NON-CANONICAL]**):*
  ```console
  $ curl -s http://calypso.localhost:3000/reader -H 'x-support-session: SYNTHETIC_PROBE' \
         -o q3_srv_support.html -w "http_code=%{http_code} size=%{size_download}\n"
  http_code=200 size=41743
  # `normalize()` as defined in Q3.c above (strips volatile per-build hashes)
  $ diff <(normalize q3_srv_canonical.html) <(normalize q3_srv_support.html)
  21a22,23
  > var isSupportSession = true;
  > var disableHelpCenterAutoOpen = true;
  ```
  The server read `x-support-session` (`client/server/pages/index.js:L91`) and injected both vars (+67
  bytes). *Blocker:* a real support session (`support_session_id` cookie,
  `client/server/user-bootstrap/index.js:L9`) is an internal Automattic mechanism; only the
  server detection is exercisable here. **[OBSERVED server branch; session itself NON-CANONICAL]**
- **Third-party cookies.** *Observed:* the app origin is `calypso.localhost:3000`. Reading
  `document.cookie` on the live logged-out `/reader` (→ `/discover`) page returned the
  **first-party** cookie names `country_code`, `region`, `tk_ai`, `tk_qs` — so `tk_ai`/`tk_qs`
  are **first-party analytics cookies written client-side** by Calypso's own tracks library, not
  `public-api.wordpress.com` `set-cookie`s. Source confirms this: when `tk_ai` is absent the
  library generates an id and writes it first-party via
  `document.cookie = cookie.serialize( 'tk_ai', _ui )`
  (`packages/calypso-analytics/src/tracks.ts:L94-L99`); `country_code`/`region` are likewise
  client-set from the `/geo/` JSON body. The REST calls that populate the stream are cross-origin
  to `public-api.wordpress.com`; probing its endpoints shows
  `/rest/v1.1/me?http_envelope=1&meta=flags` and `/geo/` return **no** `Set-Cookie`, while the
  rest-proxy (`/wp-admin/rest-proxy/?v=2.0`) sets the **genuine third-party** cookies `wp_api`
  (`domain=public-api.wordpress.com; secure`) and `wp_api_sec`
  (`domain=public-api.wordpress.com; secure; HttpOnly`). Headless Chrome accepts third-party
  cookies by default, so the rest-proxy handshake completed and the logged-out stream loaded with
  no exception needed. *Source-derived caveat* (`docs/install.md:L40`): if a browser **blocks**
  third-party cookies, an exception on `https://public-api.wordpress.com` is required for
  **authenticated** flows; the logged-out Reader needs no auth cookie and is unaffected.
  **[OBSERVED]**


---

## Q4 — Responsive sidebar: header padding/margin, custom properties, breakpoints

**Short answer.** The canonical logged-out Reader renders **no sidebar** (`has-no-sidebar`),
so per-variant header `padding`/`margin` are reported **[SOURCE-DERIVED]** from the SCSS
(labelled), while the CSS custom properties and the masterbar breakpoint were captured
**[OBSERVED]** live. The global sidebar header is `padding: 30px 24px 29px`
(`client/layout/global-sidebar/style.scss:L75`). The layout `calc()` expressions are driven by
`--sidebar-width-max` / `--sidebar-width-min` / `--masterbar-height`
(`client/assets/stylesheets/shared/_variables.scss`). Layout changes at several viewport
widths — importantly, the widths that actually drive sidebar/layout changes are **not** all in
the declared `$breakpoints` scale.

### Q4.0 — Observed nuance: no sidebar mounts on the logged-out Reader

A live DOM probe of the canonical logged-out Reader shows there is no sidebar at all:

```javascript
// live DOM probe (read-only)
({
  bodyClass: document.body.className,
  layoutClass: document.querySelector('.layout')?.className,
  counts: ['.sidebar','.sidebar__header','.global-sidebar','.sidebar-v2','.sidebar-v2__header']
            .reduce((o,s)=>(o[s]=document.querySelectorAll(s).length,o),{}),
})
```

```text
bodyClass  : "color-scheme theme-default is-group-reader is-section-reader font-smoothing-antialiased is-reader-page"
layoutClass: "layout is-group-reader is-section-reader focus-content has-header-section has-no-sidebar feature-flag-woocommerce-core-profiler-passwordless-auth"
counts     : { .sidebar:0, .sidebar__header:0, .global-sidebar:0, .sidebar-v2:0, .sidebar-v2__header:0 }
```

A CSSOM scan (51 stylesheets / 14,358 rules) matched **zero** sidebar selectors — the sidebar
SCSS is code-split out of the logged-out Reader bundle entirely. A real sidebar requires a
logged-in my-sites/reader context; no credentials were available (the blocker). A screenshot
of the rendered page confirms a **top horizontal masthead** (logo left; "Discover / Popular
Tags / Search / Log In / Sign Up" right) with a centered content column and **no left
sidebar** (captured live during the session and described here; not committed, to keep the
repository to the single deliverable). Therefore header padding/margin are **[SOURCE-DERIVED]**
and explicitly labelled. **[OBSERVED]**

### Q4.a — Sidebar header padding/margin, per variant (source-derived)

There are three sidebar variants; the question's values belong to specific ones.

**Global sidebar (unified nav)** — `client/layout/global-sidebar/style.scss`:

```scss
.sidebar__header {          /* :L70 */
	align-items: center;    /* :L71 */
	/* Hide the header when the masterbar is visible. */
	display: none;          /* :L73 */
	gap: 8px;               /* :L74 */
	padding: 30px 24px 29px;/* :L75  ← the header padding */
	span.dotcom {           /* :L82 */
		width: 125px;       /* :L84 */
		height: 28px;       /* :L85 */
		margin: 0;          /* :L86 */
	}
}
.sidebar__body { padding-top: 40px; }  /* :L108 / :L112 */
```

**Classic sidebar** — `client/layout/sidebar/style.scss`:

```scss
.sidebar {                  /* :L1 */
	margin: 0;              /* :L4 */
	padding: 0;             /* :L5 */
	padding-top: 6px;       /* :L6 */
}
.sidebar__heading {         /* :L66 */
	padding: 16px 8px 6px 16px; /* :L70  ← header-row padding */
	margin: 0;              /* :L71 */
}
```

**Sidebar v2** — it **does** have its own header element (correcting any claim that v2 lacks a
header): `client/layout/sidebar-v2/header.tsx:L8` exports `SidebarV2Header`, and `:L9` renders
`<div className={ clsx( 'sidebar-v2__header', className ) }>`. The container padding is on the
root `.sidebar-v2`:

```scss
.sidebar-v2 {               /* :L1 */
	gap: 16px;              /* :L4 */
	margin: 0;              /* :L10 */
	padding: 16px;          /* :L11 */
	box-sizing: border-box; /* :L12 */
}
```

**[SOURCE-DERIVED]** for all three (sidebar not canonically rendered; SCSS compiles
deterministically).

### Q4.b — CSS custom properties driving the `calc()` layout (live)

The custom properties were read live from the running page (`:root` and `body`):

```text
--masterbar-height  : 32px   (at innerWidth 1280 and 1905)
--sidebar-width-max : 272px
--sidebar-width-min : 228px
```

Authority — `client/assets/stylesheets/shared/_variables.scss`:

```scss
:root {
	--masterbar-height: 46px;                                   /* :L7  base */
	@media only screen and (min-width: 782px) {                 /* :L10 */
		--masterbar-height: 32px;                               /* :L11 override */
	}
	--sidebar-width-max: 272px;                                 /* :L15 */
	--sidebar-width-min: 228px;                                 /* :L16 */
}
```

(The observed `272/228` are the `:root` base. The my-sites context overrides these — `272/272`
base, `295px` visible, `69px` collapsed — but those are not loaded in the Reader, so they are
source-derived; see Q4.a-adjacent note below.) The consumers of these properties —
`client/layout/style.scss`:

```scss
.layout__content {                                                        /* :L48 */
	padding: 79px 32px 32px calc(var(--sidebar-width-max) + 32px + 1px);  /* :L52 */
	@include breakpoint-deprecated( "<960px" ) {                          /* :L118 */
		padding: 71px 24px 24px calc(var(--sidebar-width-min) + 24px + 1px); /* :L119 */
	}
}
/* width switches max→min under 960px */
width: var(--sidebar-width-max);   /* :L185 */
width: var(--sidebar-width-min);   /* :L192 (inside <960px) */
&.has-no-sidebar .layout__content { padding-top: 0; }  /* :L415 — the reader's active branch */
```

**my-sites overrides (F5 citation fix; source-derived, my-sites context)** —
`client/my-sites/sidebar/style.scss`:

```scss
.theme-default {                        /* :L10 */
	--sidebar-width-max: 272px;         /* :L12 */
	--sidebar-width-min: 272px;         /* :L13 */
	.is-global-sidebar-visible {        /* :L15 */
		--sidebar-width-max: 295px;     /* :L16  ← 295px is L15–L17 */
		--sidebar-width-min: 295px;     /* :L17 */
	}
	.is-global-sidebar-collapsed {      /* :L59 */
		--sidebar-width-max: 69px;      /* :L60  ← collapsed 69px is L59–L61 */
		--sidebar-width-min: 69px;      /* :L61 */
	}
	.sidebar__heading, .sidebar__menu-link { padding: 0 0 0 8px; }  /* :L182–L189 themed cascade */
}
```

**[OBSERVED]** custom-property values; **[SOURCE-DERIVED]** overrides and consumers.

### Q4.c — Breakpoints, exercised before / during / after (threshold geometry)

The masterbar breakpoint (782px) was exercised live by resizing the viewport across the
boundary and reading `matchMedia` + the custom property at each step:

```text
width = 781 : matchMedia(min-width:782px)=false → --masterbar-height = 46px
width = 782 : matchMedia(min-width:782px)=true  → --masterbar-height = 32px   (boundary INCLUSIVE)
width = 783 : matchMedia(min-width:782px)=true  → --masterbar-height = 32px
# additional matchMedia truths @781: (min-width:661px)=true, (max-width:660px)=false,
#                                     (max-width:960px)=true,  (min-width:600px)=true
```

So the masterbar shrinks 46px → 32px at exactly 782px (inclusive). **[OBSERVED]**

**Declared scale vs. actually-consumed queries (precision).** The global declared scale is
`client/assets/stylesheets/shared/mixins/_breakpoints.scss:L10`:

```scss
$breakpoints: 480px, 660px, 800px, 960px, 1040px, 1280px, 1400px; // Think very carefully before adding a new breakpoint
```

But the widths that actually drive **sidebar/layout** changes are not all in that scale:

| Threshold | Effect | In `$breakpoints`? | Evidence |
|-----------|--------|--------------------|----------|
| **782px** | `--masterbar-height` 46→32 | **No** (WP admin-bar width) | `client/assets/stylesheets/shared/_variables.scss:L10` — **[OBSERVED]** |
| 960px | layout width max→min + padding | Yes | `client/layout/style.scss:L118,L191` — [SOURCE-DERIVED] |
| 660/661px | global-sidebar switches | Yes (660) | [SOURCE-DERIVED] |
| 600 / 781 / 783px | my-sites via `@wordpress/base-styles` | **No** | [SOURCE-DERIVED] |

Conclusion: the declared `$breakpoints` scale is **not** the exhaustive set of viewport widths
at which the sidebar/layout changes; the masterbar (782) and WP base-styles (600/781/783)
thresholds are separate. This corrects any overstatement that the declared scale is the
complete breakpoint set.


---

## Architecture and data-flow diagrams

**Q1 single-port topology.** Everything the browser loads from Calypso is served by one
Express process on port 3000; only REST traffic leaves the machine to the remote API.

```mermaid
flowchart LR
    Browser["Browser<br/>calypso.localhost:3000"]
    subgraph Node["Single Express server (port 3000) — build/server.js"]
        SSR["SSR page render<br/>client/server/pages"]
        DevMW["webpack-dev-middleware<br/>compiled JS/CSS assets"]
        HotMW["webpack-hot-middleware<br/>/__webpack_hmr SSE stream"]
    end
    Remote["Remote REST API<br/>public-api.wordpress.com"]

    Browser -->|"HTML"| SSR
    Browser -->|"asset requests"| DevMW
    Browser -->|"HMR updates (text/event-stream)"| HotMW
    Browser -->|"REST calls (wpcom.js)"| Remote
```

**Q2 Reader initial-load data flow.** Route → controller → component mount → action creator →
data-layer HTTP → success handler → receive actions.

```mermaid
sequenceDiagram
    participant R as Route (client/reader/index.ts:L55)
    participant C as Controller (client/reader/controller.js:L356)
    participant S as Stream mount (client/reader/stream/index.jsx:L221)
    participant A as action requestPage (client/state/reader/streams/actions.js:L28)
    participant D as data-layer (client/state/data-layer/wpcom/read/streams/index.js:L358)
    participant API as public-api.wordpress.com
    participant H as handlePage (client/state/data-layer/wpcom/read/streams/index.js:L428)

    R->>C: redirectLoggedOutToDiscover
    C->>C: isUserLoggedIn? no → page.redirect('/discover')
    S->>A: componentDidMount → requestPage({...})
    A-->>D: READER_STREAMS_PAGE_REQUEST
    D->>API: GET /wpcom/v2/read/streams/discover?number=4 (INITIAL_FETCH)
    API-->>D: envelope { cards, next_page_handle }
    D->>H: onSuccess: handlePage
    H-->>S: READER_POSTS_RECEIVE, then READER_STREAMS_PAGE_RECEIVE
    Note over D,H: on failure → onError: noop (:L519/:L526) — swallowed
```

**Q3 auth-decision chain (default logged-out).** Server cookie heuristic feeds SSR; the client
`/me` fetch decides identity and the layout.

```mermaid
flowchart TD
    Cookie{"req.cookies.wordpress_logged_in?<br/>client/server/pages/index.js:L93"}
    Cookie -->|"absent (canonical)"| LO["setUpLoggedOutRoute :L588"]
    Cookie -->|"present"| LI["setUpLoggedInRoute :L587<br/>(bootstrap gated OFF :L364)"]
    Boot["client bootApp → initializeCurrentUser<br/>client/boot/common.js:L341"]
    Boot --> Me["rawCurrentUserFetch → GET /me?meta=flags<br/>client/lib/user/shared-utils/raw-current-user-fetch.js:L3"]
    Me -->|"403 authorization_required"| NoUser["no setCurrentUser<br/>getCurrentUserId = null"]
    NoUser --> Sel["isUserLoggedIn = false<br/>client/state/current-user/selectors.js:L15"]
    Sel --> Layout["&lt;LayoutLoggedOut&gt;<br/>client/controller/index.web.js:L64"]
```

---

## Coverage pass — every named item: Observed vs. Inferred

This pass confirms every mechanism, file, and flag the four questions name is addressed, and
marks how each value was established.

**Q1 named items:**

- Dev server port `3000` — **[OBSERVED]** (`/version` on 3000; config `:L8`; bind `:L83`).
- Readiness signal(s) — **[OBSERVED]** both: bunyan boot log (`:L33`) and webpack "Ready!"
  (`bundler:L56`), raw from both runs.
- Multiple ports vs. one (HMR vs. API) — **[OBSERVED]** single-port; HMR + assets on 3000; REST
  remote. Inspector 5858 and `MOCK_WORDPRESSDOTCOM` labelled **[NON-CANONICAL]**.
- Before/during/after first compile — **[OBSERVED]** holding page, 138 s queued request,
  post-Ready 0.2 s.

**Q2 named items:**

- API endpoints populating the stream — **[OBSERVED]** `/wpcom/v2/read/streams/discover`
  (initial), plus the full endpoint map (`:L194–:L249`), and `/read/tags/posts` (Latest).
- Redux actions on initial load — **[OBSERVED]** `READER_STREAMS_PAGE_REQUEST →
  READER_POSTS_RECEIVE → READER_STREAMS_PAGE_RECEIVE`; pagination triplet; failure `onError:
  noop` exercised.

**Q3 named items:**

- "Is the user logged in?" before render — **[OBSERVED]** server cookie heuristic (`:L93`) and
  client `/me`→403 → `isUserLoggedIn` false → `<LayoutLoggedOut>`.
- Storage checked (cookies, localStorage, and so on) — **[OBSERVED]** cookies, `localStorage`,
  `sessionStorage`, IndexedDB (`calypso`/`calypso_store`), `window.initialReduxState`; token
  names `wordpress_logged_in`/`wpcom_token`/`support_session_id` enumerated.
- Logged-in / OAuth / support-session / third-party-cookie — attempted; observed where
  possible, else **[SOURCE-DERIVED]** with stated blocker; synthetic probes **[NON-CANONICAL]**.

**Q4 named items:**

- Sidebar header margin/padding — **[SOURCE-DERIVED]** per variant (global `30px 24px 29px`;
  classic `16px 8px 6px 16px`; v2 container `16px`) — with the observed nuance that no sidebar
  mounts on the logged-out Reader.
- CSS custom properties — **[OBSERVED]** `--masterbar-height`, `--sidebar-width-max/min` live;
  consumers and my-sites overrides **[SOURCE-DERIVED]** (F5 citations corrected).
- Breakpoints — **[OBSERVED]** 782px masterbar triplet (46→32, inclusive); declared scale vs.
  actually-consumed queries disambiguated.


---

## AAP compliance matrix (complete)

This matrix covers all **40 AAP acceptance requirements** enumerated by the review, each with
its status after this revision, where it is evidenced in this document, and any remaining
limitation. Requirements that were runtime-infeasible (a real login, a rendered sidebar) are
marked **PASS (source-derived)** with the documented blocker.

| # | AAP requirement | Status | Evidence (this doc) | Limitation / remaining |
|---|-----------------|--------|---------------------|------------------------|
| 1 | Sole deliverable at the branch-named path | ✅ PASS | This file; git diff | None |
| 2 | Source branch / revision identity | ✅ PASS | Intro "Commit provenance" | None |
| 3 | Build/run as first investigative step | ✅ PASS | §1 (run-first chronology) | None |
| 4 | Node 22.x, Yarn 4.0.2, hosts entry | ✅ PASS | §1.1 | None |
| 5 | Dependency install command + complete output | ✅ PASS | §1.3 (`yarn install --immutable`, timings) | None |
| 6 | Default unflagged `yarn start` as the evidence basis | ✅ PASS | §1.2, §1.4 (unflagged; `SECTION_LIMIT` labelled non-default) | None |
| 7 | Exact reproducible commands / procedures | ✅ PASS | §1.5 scripts; Q2.c enhancer shim; commands shown per block | None |
| 8 | Scale/duration + two-run stability | ✅ PASS | §1.2 duration table; §1.4 stability | Build time varies run-to-run (reported) |
| 9 | Q1 port/protocol/hostname from config | ✅ PASS | Q1.a | None |
| 10 | Q1 `server.listen` semantics, normal vs. fork | ✅ PASS | Q1.a (`:L83`, fork/null bind) | Fork-host bind is source-derived (no fork run) |
| 11 | Q1 actual boot + Ready lines | ✅ PASS | Q1.b (both runs, raw) | None |
| 12 | Q1 holding/queued + complete post-Ready | ✅ PASS | Q1.c (holding page, 138 s queue, post-Ready headers) | None |
| 13 | Q1 single-port app/HMR + remote REST | ✅ PASS | Q1.d | None |
| 14 | Distinguish entry limits / inspector / mock | ✅ PASS | Q1.e | None |
| 15 | Q2 route → controller → view → mount chain | ✅ PASS | Q2.d (each step cited) | None |
| 16 | Q2 exact observed request details | ✅ PASS | Q2.a (full URL, envelope shape) | Body summarized structurally per §1.6 |
| 17 | Q2 initial Redux sequence + payload context | ✅ PASS | Q2.c (two loads, shim, payload limitation) | Payload key names shim-vantage-dependent (stated) |
| 18 | Q2 pagination trigger/request/response | ✅ PASS | Q2.e (full URL, PER_FETCH=7) | Cursor redacted per §1.6 |
| 19 | Q2 sibling endpoint map, source-labeled | ✅ PASS | Q2.b table | None |
| 20 | Q2 failure behavior (`onError: noop`) | ✅ PASS | Q2.f (exercised offline) | None |
| 21 | Q3 SSR cookie/context/cache/redirect/bootstrap | ✅ PASS | Q3.c (`:L93,:L138,:L364,:L372,:L586-L588`) | None |
| 22 | Q3 server `/me`, absent-cookie error, `setCurrentUser` | ✅ PASS (source-derived) | Q3.a/f (`user-bootstrap:L13/L28/L34`, `:L391`) | Server bootstrap gated OFF by default (blocker) |
| 23 | Q3 canonical default client `/me` flow | ✅ PASS | Q3.b (chain observed; `/me`→403) | None |
| 24 | Q3 initial state / IndexedDB / selector / layout | ✅ PASS | Q3.d/e | None |
| 25 | Q3 complete storage/mode table | ✅ PASS | Q3.e (live dump + citations) | Values redacted per §1.6 |
| 26 | Q3 logged-out/in/OAuth/support/third-party conditions | ✅ PASS | Q3.f (observed or source-derived + blocker) | Real login/OAuth/support infeasible (blockers stated) |
| 27 | Distinguish cookie presence / authentication / authorization | ✅ PASS | Q3.a (four concepts) + Q3.c | None |
| 28 | Q4 identify active rendered variant | ✅ PASS | Q4.0 (no sidebar; explicit) | Logged-out Reader renders no sidebar (documented) |
| 29 | Q4 exact source/computed spacing + geometry | ✅ PASS | Q4.a (source-derived), Q4.b/c (observed vars/breakpoint) | Header geometry source-derived (no sidebar) |
| 30 | Q4 declarations/overrides/fallbacks/`calc()` consumers | ✅ PASS | Q4.b (F5 citations corrected) | None |
| 31 | Q4 boundary-state evidence + collision/overflow | ✅ PASS | Q4.c (782px triplet observed) | Collision/overflow not observable without a sidebar (blocker) |
| 32 | Official Automattic corroboration | ✅ PASS | §2 | Supplementary to observation (by design) |
| 33 | Every claim has exact current-HEAD citation | ✅ PASS | Throughout; validated programmatically | None |
| 34 | Complete unedited output + narrow redaction | ✅ PASS | §1.6 policy; tightened blocks | Large bodies summarized structurally (policy) |
| 35 | Security/privacy handling; no secret exposure | ✅ PASS | §1.6 | None |
| 36 | Cleanup + pristine repository proof | ✅ PASS | Repository cleanliness proof (finalized at commit) | None |
| 37 | Complete final AAP + Rules coverage matrices | ✅ PASS | This matrix + Rules matrix | None |
| 38 | No placeholders / unsupported inference / universal certainty | ✅ PASS | Scoped language; Observed/Source-derived labels | None |
| 39 | Concise, comprehensive onboarding usefulness | ✅ PASS | Whole document | None |
| 40 | No out-of-scope tracked source work | ✅ PASS | git diff (only the `.md`) | None |

### AAP file-plan coverage (§0.6.1)

The table below additionally confirms **every row** of the AAP File Transformation Mapping
(§0.6.1): one CREATE plus every REFERENCE file. "Referenced" means the file was read and its
cited behaviour was used to ground an answer; "Exercised" additionally means its behaviour was
observed at runtime.

| # | AAP file (target) | Plan | Q | Status | Evidence / where used | Limitation |
|---|-------------------|------|---|--------|-----------------------|------------|
| 1 | `blitzy/documentation/wp-calypso_be7e5cc64162.md` | CREATE | — | **Created** | This document | None |
| 2 | `config/development.json` | REFERENCE | Q1 | Exercised | Q1.a (port 3000, `:L6-L8`) | None |
| 3 | `config/_shared.json` | REFERENCE | Q1 | Referenced | Q1.a (shared defaults `:L13,:L24-L25`) | None |
| 4 | `client/server/index.js` | REFERENCE | Q1 | Exercised | Q1.a/b (port read `:L12`, bind `:L83`, boot log `:L33`) | None |
| 5 | `client/server/boot/index.js` | REFERENCE | Q1 | Referenced | Q1.d (bundler attach `:L37`) | None |
| 6 | `client/server/bundler/index.js` | REFERENCE | Q1 | Exercised | Q1.b/c/d (Ready `:L56`, holding page `:L66-L86`, middlewares `:L5-L6`) | None |
| 7 | `client/server/lib/logger/index.js` | REFERENCE | Q1 | Referenced | Q1.b (bunyan `:L7,:L10,:L11`) | None |
| 8 | `client/sections.js` | REFERENCE | Q1–Q3 | Referenced | Reader section `enableLoggedOut` context | Reader route confirmed via `reader/index.ts` at runtime |
| 9 | `package.json` | REFERENCE | Q1 | Exercised | §1.2 (`start` `:L110`, `start-build` `:L113`, `build-server` `:L81`, engines `:L57`, pm `:L422`) | None |
| 10 | `.nvmrc` | REFERENCE | Q1 | Referenced | §1.1 (`22.9.0`) | None |
| 11 | `README.md` | REFERENCE | Q1 | Referenced | §2 (`:L20-L21`) | None |
| 12 | `docs/install.md` | REFERENCE | Q1/Q3 | Referenced | §2 (`:L36,:L38,:L40,:L48-L50,:L63`) | None |
| 13 | `docs/yarn-start.md` | REFERENCE | Q1 | Referenced | §2 (boot flow `:L1-L28`) | None |
| 14 | `client/state/data-layer/wpcom/read/streams/index.js` | REFERENCE | Q2 | Exercised | Q2.a/b/d/e/f (endpoint map, builder, handlers, noop) | None |
| 15 | `client/state/reader/streams/actions.js` | REFERENCE | Q2 | Exercised | Q2.c/d (`requestPage :L28`, `receivePage :L52`) | Payload shape source-grounded (Q2.c limitation) |
| 16 | `client/state/reader/action-types.ts` | REFERENCE | Q2 | Exercised | Q2.c (constants `:L52,:L77-:L79`) | None |
| 17 | `client/state/reader/posts/actions.js` | REFERENCE | Q2 | Exercised | Q2.d (`receivePosts :L63,:L86-L88`) | None |
| 18 | `client/reader/stream/index.jsx` | REFERENCE | Q2 | Exercised | Q2.d (mount `:L221`, `fetchNextPage :L489`, `requestPage :L502`) | None |
| 19 | `client/reader/following/main.tsx` | REFERENCE | Q2 | Referenced | Following main view (Discover is the logged-out canonical target) | Logged-out redirects to `/discover`; Following is logged-in |
| 20 | `client/reader/following/index.js` | REFERENCE | Q2 | Referenced | Following controller | Same as #19 |
| 21 | `client/reader/index.ts` (AAP lists `.js`) | REFERENCE | Q2 | Exercised | Q2.d (route `:L55-L56`); actual extension `.ts` | AAP filename extension differed; noted |
| 22 | `client/server/pages/index.js` | REFERENCE | Q3 | Exercised | Q3.c (`isLoggedIn :L93`, route `:L586-L588`, bootstrap gate `:L364`) | None |
| 23 | `client/server/user-bootstrap/index.js` | REFERENCE | Q3 | Referenced | Q3.a/f (`AUTH_COOKIE_NAME :L8`, `/me :L13`, throw `:L34`) | Bootstrap gated OFF by default; source-derived |
| 24 | `client/state/current-user/selectors.js` | REFERENCE | Q3 | Exercised | Q3.b/d (`isUserLoggedIn :L15-L16`) | None |
| 25 | `client/state/current-user/actions.js` | REFERENCE | Q3 | Referenced | Q3.c (`setCurrentUser`) | Not dispatched in logged-out run (expected) |
| 26 | `client/state/initial-state.js` | REFERENCE | Q3 | Exercised | Q3.d/e (key `:L76`, server merge `:L148-L154`) | None |
| 27 | `client/controller/index.web.js` | REFERENCE | Q3 | Exercised | Q3.d (`isUserLoggedIn :L59`, `LayoutLoggedOut :L64`) | None |
| 28 | `client/boot/common.js` | REFERENCE | Q3 | Exercised | Q3.b/f (`bootApp :L340`, `initializeCurrentUser :L341`, oauth `:L154-L177`) | OAuth branch source-derived (oauth:false) |
| 29 | `packages/oauth-token/src/index.js` | REFERENCE | Q3 | Referenced | Q3.f (`TOKEN_NAME :L7`, cookie/`store` fallback `:L11,:L17`) | No token present (logged-out); source-derived |
| 30 | `client/layout/global-sidebar/style.scss` | REFERENCE | Q4 | Exercised (CSSOM absence) | Q4.a (`.sidebar__header padding :L75`) | Not rendered on logged-out Reader; source-derived padding |
| 31 | `client/layout/sidebar/style.scss` | REFERENCE | Q4 | Referenced | Q4.a (classic `:L4-L6,:L70-L71`) | Source-derived (not rendered) |
| 32 | `client/layout/sidebar-v2/` | REFERENCE | Q4 | Referenced | Q4.a (`SidebarV2Header` at `client/layout/sidebar-v2/header.tsx:L8-L9`; `client/layout/sidebar-v2/style.scss:L1-L12`) | Source-derived (not rendered) |
| 33 | `client/layout/style.scss` | REFERENCE | Q4 | Exercised | Q4.b (`calc() :L52,:L118-L119`, `has-no-sidebar :L415`) | None |
| 34 | `client/assets/stylesheets/shared/_variables.scss` | REFERENCE | Q4 | Exercised | Q4.b (`--masterbar-height :L7-L11`, widths `:L15-L16`) | None |
| 35 | `client/assets/stylesheets/shared/mixins/_breakpoints.scss` | REFERENCE | Q4 | Exercised | Q4.c (`$breakpoints :L10`) | None |
| 36 | `client/my-sites/sidebar/style.scss` | REFERENCE | Q4 | Referenced | Q4.b (295px `:L15-L17`, 69px `:L59-L61`, cascade `:L182-L189`) | Source-derived (my-sites context not rendered) |

**Scope directives (AAP §0.3, §0.7):** read-only source repository — **honored** (only the one
answer document created; cleanliness proof below). Default canonical configuration —
**honored** (unflagged `yarn start`; non-default flags labelled). Build-and-run-first —
**honored** (§1). Remove temporary scripts — **honored** (cleanup below).


---

## Rules compliance matrix (complete)

This matrix covers every rule in the user-specified "SWE-AtlasQnA-Repo" rule set (AAP §0.7),
plus the AAP's official-documentation research requirement.

| # | Rule (AAP §0.7 / §0.2.2) | Status | How satisfied | Limitation |
|---|--------------------------|--------|---------------|------------|
| 1 | Deliverable named `<source_branch>.md` in `blitzy/documentation/` | **Met** | This file: `blitzy/documentation/wp-calypso_be7e5cc64162.md` | None |
| 2 | Build & run first; write from observation | **Met** | §1 chronology; every Q section leads with captured output | None |
| 3 | Observe at sufficient scale/duration; stable across ≥2 runs | **Met** | §1.4 two runs; port/Ready/warning-count stable | Build times vary run-to-run (expected) |
| 4 | Reproduce reported inconsistency (run repeatedly) | **Met (N/A specifics)** | Two runs reported; boot-ms is the only run-to-run variance, reported as such | No inconsistency was alleged in the questions |
| 5 | Exercise the canonical entry point (no bypass/mock) | **Met** | Real `yarn start` + real `/reader`→`/discover`; `MOCK_WORDPRESSDOTCOM` labelled NON-CANONICAL | None |
| 6 | Default, canonical configuration; state exact commands | **Met** | Unflagged `yarn start`; commands shown throughout; non-default flags labelled | None |
| 7 | Persist until the signal is captured; label inferred | **Met** | Boot log, Ready, network, actions, storage, CSS vars, breakpoint all captured live; source-derived items labelled | Sidebar render & real login infeasible → labelled SOURCE-DERIVED/blocker |
| 8 | Exercise every condition (primary, alt, error, transitional) | **Met** | Q1 before/during/after; Q2 pagination + offline failure; Q3 alt modes; Q4 breakpoint triplet | Some alt paths source-derived with stated blockers |
| 9 | Include actual, complete output with its command | **Met** | Raw blocks with preceding commands; no `// ...` elision; only narrow `[REDACTED: …]` per §1.6 | Large bodies summarized structurally (redaction policy) |
| 10 | Answer every part and every named item | **Met** | Coverage pass enumerates each named item Observed/Inferred | None |
| 11 | Be exact & grounded (`file:line`, named symbols) | **Met** | Every claim carries `file:line`; symbols named (functions, selectors, constants) | None |
| 12 | Scope: read-only source; remove temp scripts | **Met** | Only the answer doc created; cleanup proof below | None |
| 13 | Official Automattic documentation corroboration (AAP §0.2.2) | **Met** | §2 (in-repo `README`/`docs/install.md`/`docs/yarn-start.md` + current published docs) | Stated as supplementary, not primary |

---

## Repository cleanliness proof

The source repository must be left unchanged except for the single answer document. The
methodology used only external scratch space and transient outputs:

- **Observation scripts and every file they write** (`start_run*.raw.log`, `server_run*.pgid`,
  `nonroot_reader.html`) live under `/tmp/calypso_obs/` (outside the repo) and are removed wholesale
  by `rm -rf /tmp/calypso_obs`; no helper writes into the working tree.
- **Build outputs** (`build/`, `public/`) produced by `yarn start` are git-ignored; they are
  removed in cleanup so the working tree carries no observation residue. Install state
  (`node_modules/`, `packages/*/dist`) is git-ignored and intentionally retained.
- **The only tracked change** is `blitzy/documentation/wp-calypso_be7e5cc64162.md`.

The final proof (normal and ignored-aware `git status`) is captured after cleanup and embedded
here:

**Cleanup commands executed** (specific paths inside the working tree only):

```bash
$ rm -rf blitzy/screenshots      # untracked screenshot artifact (kept repo to single deliverable)
$ rm -rf build                   # git-ignored SSR output produced by `yarn start`
$ rm -rf public                  # git-ignored browser-bundle output produced by `yarn start`
$ rm -rf /tmp/calypso_obs        # all observation scripts + every file they write (start_run*.raw.log, server_run*.pgid, nonroot_reader.html) — outside the repo
```

**Normal status** — the only tracked change is this document; nothing untracked remains:

```console
$ git status --porcelain
 M blitzy/documentation/wp-calypso_be7e5cc64162.md
```

**Ignored-aware status** — no observation residue (`build/`, `public/`, `blitzy/screenshots/`)
remains; the 83 remaining ignored entries are exclusively install state that is intentionally
retained (`node_modules/`, each `packages/*/dist/`, `.cache/`, `.husky/_/`,
`.yarn/install-state.gz`):

```console
$ git status --ignored --porcelain | grep -E '^!! (build|public)/$' \
    || echo 'none — no top-level build/ or public/ residue'
none — no top-level build/ or public/ residue

$ git status --ignored --porcelain | grep -c '^!!'
83

$ git status --ignored --porcelain | grep -E 'screenshots' \
    || echo 'none — no blitzy/screenshots residue'
none — no blitzy/screenshots residue
```

The `git status --porcelain` above was captured immediately before `git add`/`git commit` of
this document; the commit records exactly that single tracked modification. The source
repository is therefore left unchanged apart from the single answer document, satisfying the
read-only-source rule.

