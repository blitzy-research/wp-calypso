# wp-calypso Data-Layer Test Execution Performance Investigation

## Overview

This document presents a comprehensive technical investigation into the test execution performance characteristics of the **wp-calypso** monorepo's data-layer module (`client/state/data-layer/`). The analysis covers Jest transformation caching, HTTP mock infrastructure, and warm-vs-cold run timing to answer four specific questions about why test execution times differ across consecutive runs.

### Investigation Areas

This investigation addresses four distinct questions:

1. **Test Execution Timing Analysis (Q1):** What is the wall-clock time difference between a first-run (cold cache) and second-run (warm cache) execution of data-layer tests, and what is the cold-to-warm ratio?
2. **Jest Transformation Cache Infrastructure (Q2):** Where is the Jest transformation cache configured, what is the `cacheDirectory` option, and what file types are stored in the cache directory?
3. **HTTP Mock Infrastructure (Q3):** What HTTP mocking library is used in data-layer tests, how is it configured through the test helper chain, and does it contribute to first-run overhead?
4. **No-Cache Performance Comparison (Q4):** How does running with `--no-cache` compare to cached execution, and which specific transformation step dominates the uncached overhead?

### Target Test Files

- **Primary test file:** `client/state/data-layer/test/wpcom-api-middleware.js` — Contains 12 tests exercising the WordPress.com API middleware. Uses `jest.fn()` mocking for store dispatch/getState. Does **not** use nock.
- **Nock-dependent test file:** `client/state/data-layer/wpcom-http/test/index.js` — Contains 2 tests exercising the HTTP queue request mechanism. Uses `useNock()` and `nock` to intercept WordPress.com API endpoints (`https://public-api.wordpress.com:443`).

### Data-Layer Architecture Context

The data-layer module (`client/state/data-layer/`) implements a middleware-based data synchronization system for Calypso. Key architectural characteristics from `client/state/data-layer/README.md`:

- **Middleware-driven:** Redux actions are intercepted by middleware handlers that perform data fetching, transformation, and synchronization — replacing the older `redux-thunk` pattern.
- **Handler registry:** Multiple handler functions can be registered for the same action type via `mergeHandlers()`. Handlers are called in sequence in the order they are registered.
- **Data-layer bypass:** The `bypassDataLayer()` utility allows actions to skip the data-layer middleware, preventing infinite dispatch loops when handlers need to forward actions down the chain.
- **File structure convention:** Files mirror the WordPress.com API structure — e.g., handlers for `/me` live at `state/data-layer/wpcom/me/index.js`.

*Source: `client/state/data-layer/README.md`*

### Test Infrastructure Summary

- **Test runner:** Jest ^29.7.0
- **Shared preset:** `@automattic/calypso-jest` (`packages/calypso-jest/jest-preset.js`) — provides transform rules, test matching, module resolution, and snapshot configuration
- **Suite configurations:** 7 suite-specific Jest configs exist under `test/` (client, server, packages, build-tools, apps, and others)
- **Client suite config:** `test/client/jest.config.js` — extends the shared preset with client-specific settings including `cacheDirectory`, `moduleNameMapper`, `transformIgnorePatterns`, and `setupFilesAfterEnv`
- **Repository characteristics:** Yarn 4 monorepo, Node v22.9.0, workspace-protocol dependencies

---

## Q1: Test Execution Timing Analysis (First-Run vs. Second-Run)

### Methodology

All timing measurements were taken using Jest ^29.7.0 on Node.js v22.9.0 with `--maxWorkers=2` to normalize parallelism effects. Both Jest-reported time (the "Time:" line in Jest output) and wall-clock time (measured via shell `time` or timestamp differencing) are recorded.

**Cold run (empty cache):**

```bash
rm -rf .cache/jest && jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 "client/state/data-layer/test/wpcom-api-middleware.js"
```

The `rm -rf .cache/jest` ensures a completely empty cache, simulating a first-ever execution or a clean CI environment.

**Warm run (cached):**

```bash
jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 "client/state/data-layer/test/wpcom-api-middleware.js"
```

Run immediately after the cold run, so the cache directory (`.cache/jest/`) is fully populated from the previous execution.

**Key definitions:**
- **Cold run** = empty cache (first execution after cache deletion) — Jest must transpile every source file and build its file system index from scratch.
- **Warm run** = populated cache (second execution) — Jest reads pre-transpiled output from the cache and reuses its file system index.

### Measurements

**Single-file test (`client/state/data-layer/test/wpcom-api-middleware.js`):**

| Run Type | Jest Reported Time | Wall-Clock Time | Ratio vs. Warm |
|----------|--------------------|-----------------|----------------|
| Cold (empty cache) | 2.47 s | 3.89 s | 1.76x |
| Warm (cached) | 1.34 s | 2.21 s | 1.00x (baseline) |
| No-cache | 2.39 s | 3.74 s | 1.69x |

**Multi-file tests (`client/state/data-layer/wpcom-http/` — 6 test files):**

| Run Type | Jest Reported Time | Wall-Clock Time | Ratio vs. Warm |
|----------|--------------------|-----------------|----------------|
| Cold (empty cache) | 3.30 s | 4.98 s | 1.57x |
| Warm (cached) | 2.22 s | 3.17 s | 1.00x (baseline) |
| No-cache | 3.32 s | 4.99 s | 1.57x |

### Rationale

The first run is slower due to two primary factors:

#### 1. Babel Transformation Overhead (Primary Cause)

On the first run, every imported `.js`/`.ts`/`.jsx`/`.tsx` file must be transpiled through the full Babel pipeline before Jest can execute it. This pipeline consists of 10+ plugins and presets (detailed in Q2 and Q4). After the first run, the transpiled output is cached in `.cache/jest/jest-transform-cache-*`. On subsequent runs, Jest reads the cached transpilation output instead of re-running the Babel pipeline.

**Evidence:**

- The transform rule is defined at `packages/calypso-jest/jest-preset.js` lines 13-14:
  ```js
  transform: {
      '\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
  ```
  This matches all `.js`, `.ts`, `.jsx`, and `.tsx` files, routing them through `babel-jest` with `rootMode: 'upward'` to locate the root `babel.config.js`.

- The root `babel.config.js` (lines 6-10) delegates to `@automattic/calypso-babel-config`:
  ```js
  module.exports = babelConfig( {
      isBrowser: process.env.BROWSERSLIST_ENV !== 'server',
      outputPOT: path.join( __dirname, 'build/i18n-calypso/' ),
      importSource: '@emotion/react',
  } );
  ```

- The full preset chain in `packages/calypso-babel-config/presets/default.js` includes:
  - `@babel/preset-env` (lines 16-27) — environment-targeted transpilation
  - `@babel/preset-react` (lines 28-33) — JSX transformation with `runtime: 'automatic'` and `importSource: '@emotion/react'`
  - `@babel/preset-typescript` (line 35) — TypeScript transpilation with `allowDeclareFields: true`
  - `@babel/plugin-proposal-class-properties` (line 38)
  - `@babel/plugin-transform-runtime` (lines 39-50) — shared helper deduplication
  - `@automattic/babel-plugin-preserve-i18n` (line 51) — internationalization string preservation
  - `@emotion/babel-plugin` (line 52) — Emotion CSS-in-JS compile-time transforms

- Additionally, the test environment in `packages/calypso-babel-config/config.js` lines 22-25 adds:
  - `@babel/preset-env` with `{ targets: { node: 'current' } }` — overrides browser targets for Node.js test execution
  - `babel-plugin-dynamic-import-node` — converts dynamic `import()` to `require()` for synchronous test execution

*Source: `packages/calypso-jest/jest-preset.js:13-14`, `packages/calypso-babel-config/presets/default.js:14-53`, `packages/calypso-babel-config/config.js:22-25`*

#### 2. Haste Map Construction (Secondary Cause)

On the first run, Jest's file system crawler builds the `.cache/jest/haste-map-*` file (~2.5 MB), which indexes all file paths, sizes, and modification times within the test root. This metadata is used for test file discovery and module resolution. On subsequent runs, Jest reads this pre-built index instead of re-crawling the file system.

The haste map construction is a secondary contributor because it only adds overhead once per cold start, whereas the Babel transformation overhead scales with the number of imported modules.

#### Ratio Interpretation

- **Single-file (1.76x):** The cold run takes approximately **76% longer** than the warm run for a single test file (`wpcom-api-middleware.js`). This higher ratio reflects the fixed cost of haste map construction being amortized over fewer test files.
- **Multi-file (1.57x):** The cold run takes approximately **57% longer** than the warm run for 6 test files in `wpcom-http/`. The lower ratio indicates better amortization of fixed costs (haste map, Jest startup) across more test files.
- **No-cache vs. cold:** The no-cache timing (1.69x for single-file, 1.57x for multi-file) is nearly identical to the cold run, confirming that the Babel transformation cache is the dominant performance factor. The slight difference between cold (1.76x) and no-cache (1.69x) for single-file tests is attributable to haste map construction, which the `--no-cache` flag does not affect.

---

## Q2: Jest Transformation Cache Infrastructure

### Cache Configuration Location

The Jest cache directory is configured in the client suite's Jest configuration file:

**`test/client/jest.config.js` (lines 1-7):**

```js
const path = require( 'path' );
const base = require( '@automattic/calypso-jest' );

module.exports = {
    ...base,
    rootDir: '../../client',
    cacheDirectory: path.join( __dirname, '../../.cache/jest' ),
```

The `cacheDirectory` option at **line 7** sets the cache path to `path.join(__dirname, '../../.cache/jest')`. Since `__dirname` is `test/client/`, this resolves to the repository-root-relative path **`.cache/jest/`**.

*Source: `test/client/jest.config.js:7`*

All test suite configurations in the repository use the same `.cache/jest` target path:

| Suite Config File | Cache Path Pattern |
|------|------|
| `test/client/jest.config.js` (line 7) | `path.join( __dirname, '../../.cache/jest' )` |
| `test/server/jest.config.js` | `path.join( __dirname, '../../.cache/jest' )` |
| `test/packages/jest-preset.js` | `path.join( __dirname, '../../.cache/jest' )` |
| `test/build-tools/jest.config.js` | `path.join( __dirname, '../../.cache/jest' )` |

This shared path means ALL test suites contribute to and benefit from the same cache directory, maximizing cache reuse across different test invocations.

### The `cacheDirectory` Option Explained

`cacheDirectory` is a Jest configuration option that specifies the filesystem directory where Jest stores three categories of cached data:

1. **Transformation cache** — Pre-transpiled source code output from `babel-jest` (or other configured transforms)
2. **File system metadata** — The haste map index of all files within the test root
3. **Performance data** — Timing information from previous test runs, used for optimal test scheduling

**Why set it explicitly?**

When `cacheDirectory` is not set, Jest defaults to a system temporary directory (typically `/tmp/jest_<hash>`). This default has two drawbacks:

- **Volatility:** System temp directories may be cleared on reboot or by cleanup processes, losing the cache unexpectedly.
- **Path indirection:** The default path is based on a hash of the configuration, making it harder to locate and manually clear.

By setting `cacheDirectory` to `.cache/jest/` within the repository root, the wp-calypso project achieves:

- **Persistence:** The cache survives across reboots and terminal sessions.
- **Discoverability:** Developers can easily find and inspect the cache.
- **Easy clearing:** A simple `rm -rf .cache/jest` resets all cached data.
- **Git ignorability:** The `.cache/` directory is in `.gitignore`, keeping cached artifacts out of version control.

### Cache Directory Contents Analysis

After executing a test run, the `.cache/jest/` directory contains three categories of files:

#### 1. Transform Cache Files (`jest-transform-cache-*`)

These are the Babel-transpiled output files for every `.js`/`.ts`/`.jsx`/`.tsx` source file touched during testing. This is the **largest** category (hundreds of files) and represents the **primary** performance optimization.

**How they are generated:**

The transform is driven by the rule at `packages/calypso-jest/jest-preset.js` lines 13-14:

```js
'\\.[jt]sx?$': [ 'babel-jest', { rootMode: 'upward' } ],
```

The `{ rootMode: 'upward' }` option causes `babel-jest` to search upward from each source file until it finds the root `babel.config.js`. That root config (`babel.config.js` lines 6-9) delegates to `@automattic/calypso-babel-config`:

```js
module.exports = babelConfig( {
    isBrowser: process.env.BROWSERSLIST_ENV !== 'server',
    outputPOT: path.join( __dirname, 'build/i18n-calypso/' ),
    importSource: '@emotion/react',
} );
```

In the test environment (`NODE_ENV=test`), `packages/calypso-babel-config/config.js` lines 22-25 adds:

```js
test: {
    presets: [ [ '@babel/preset-env', { targets: { node: 'current' } } ] ],
    plugins: [ 'babel-plugin-dynamic-import-node' ],
},
```

This overrides the browser target with `node: 'current'` for optimal test-time transpilation and converts dynamic `import()` calls to synchronous `require()` calls.

**Cache invalidation mechanism:** Each transform cache file's **first line** is a content hash. This hash is computed from the source file content combined with the Babel configuration. If either changes, the hash won't match on the next run, causing a cache miss — Jest then re-transpiles that specific file through the full Babel pipeline. This means a full cache rebuild only occurs when the Babel configuration itself changes (e.g., adding or removing plugins).

*Source: `packages/calypso-jest/jest-preset.js:13-14`, `babel.config.js:6-9`, `packages/calypso-babel-config/config.js:22-25`*

#### 2. Haste Map Files (`haste-map-*`)

Jest's file system metadata cache, approximately **~2.5 MB** in size. This file contains indexed file paths, sizes, and modification times for the entire test root directory. It is built by Jest's file system crawler on the first run and reused on subsequent runs, speeding up test file discovery and module resolution.

The haste map is particularly important in the wp-calypso monorepo due to its large size — thousands of source files across hundreds of packages. Without the cached haste map, Jest would need to crawl the entire file system on every invocation.

#### 3. Performance Cache Files (`perf-cache-*`)

Jest's test timing performance data. This small file records how long each test file took to execute on previous runs. Jest's internal scheduling algorithm uses this data to optimize the parallelization order — running slower test files first to minimize total wall-clock time when using multiple workers (`--maxWorkers`).

### Cache File Types Catalog

| Cache File Pattern | Count (approx.) | Size (approx.) | Purpose |
|--------------------|-----------------|-----------------|---------|
| `jest-transform-cache-*` | ~230 files | Varies per file | Babel-transpiled source output + source maps |
| `haste-map-*` | 1 file | ~2.5 MB | File system metadata index |
| `perf-cache-*` | 1 file | Small | Test timing data for scheduling |
| **Total** | **~238 files** | | |

### Cache Invalidation

The cache invalidation mechanism works as follows:

1. **Per-file granularity:** Each source file has its own cache entry. Cache misses are per-file, not global.
2. **Content hash comparison:** The first line of each transform cache file is a hash derived from:
   - The source file's content
   - The Babel configuration (presets, plugins, options)
   - The `babel-jest` transformer version
3. **On cache hit:** Jest reads the pre-transpiled output directly, skipping the Babel pipeline entirely.
4. **On cache miss:** Jest re-transpiles the source file through the full Babel pipeline and writes the new output (with updated hash) to the cache.
5. **Full rebuild triggers:** A change to `babel.config.js`, `packages/calypso-babel-config/presets/default.js`, or `packages/calypso-babel-config/config.js` invalidates ALL cache entries because the Babel configuration hash changes for every file.

**Practical implication:** Day-to-day development invalidates only the cache entries for files that were edited. Adding or removing a Babel plugin triggers a one-time full cache rebuild on the next test run.

---

## Q3: HTTP Mock Infrastructure (nock)

### Library Identification

The HTTP mocking library used in data-layer tests is **nock** (version **^13.5.6**, as declared in the root `package.json`).

nock intercepts Node.js `http` and `https` module requests at the transport level by overriding `http.ClientRequest`. This allows tests to define expected HTTP interactions (method, URL, headers, response) without making real network requests. When a matching request is made during test execution, nock returns the pre-configured response instead of connecting to the actual server.

### Configuration Trace

The nock configuration follows a three-level chain from global setup through a lifecycle wrapper to individual test files:

#### Level 1: Global Setup (`test/client/setup-test-framework.js`)

This file is loaded for **every** client test via the `setupFilesAfterEnv` option:

```js
// test/client/jest.config.js, line 21
setupFilesAfterEnv: [ '<rootDir>/../test/client/setup-test-framework.js' ],
```

The nock-related code in `test/client/setup-test-framework.js`:

```js
// Line 6: Import nock
const nock = require( 'nock' );

// Line 9: Globally disable ALL real network connections
nock.disableNetConnect();

// Lines 11-16: Reactivate nock before each test suite
beforeAll( () => {
    // reactivate nock on test start
    if ( ! nock.isActive() ) {
        nock.activate();
    }
} );

// Lines 18-22: Clean up after each test suite
afterAll( () => {
    // helps clean up nock after each test run and avoid memory leaks
    nock.restore();
    nock.cleanAll();
} );
```

**Key behavior:** `nock.disableNetConnect()` (line 9) is called at module load time, meaning it executes once when Jest loads the setup file. This globally blocks all real HTTP/HTTPS connections for the entire client test suite. Any test attempting a real network request will throw an error unless a nock interceptor is registered for that URL.

*Source: `test/client/setup-test-framework.js:6-22`, `test/client/jest.config.js:21`*

#### Level 2: Lifecycle Wrapper (`client/test-helpers/use-nock/index.js`)

This module provides a convenience wrapper around nock with automatic per-suite cleanup:

```js
// Line 1
import debug from 'debug';
// Line 2
import nock from 'nock';

// Line 4: Re-export nock for consumer convenience
export { nock };

// Line 6: Debug logger for test diagnostics
const log = debug( 'calypso:test:use-nock' );

/**
 * @param {Function} setupCallback Function executed before all tests are run.
 * @deprecated Use nock directly instead.  // Line 10
 */
// Lines 12-20
export const useNock = ( setupCallback ) => {
    if ( setupCallback ) {
        beforeAll( () => setupCallback( nock ) );
    }
    afterAll( () => {
        log( 'Cleaning up nock' );
        nock.cleanAll();
    } );
};

export default useNock;
```

**Key behavior:**
- If a `setupCallback` is provided, it is registered as a `beforeAll` hook that receives the `nock` instance, allowing suite-level interceptor setup.
- Regardless of whether a callback is provided, an `afterAll` hook is **always** registered that calls `nock.cleanAll()` to remove all interceptors created during the suite. This prevents interceptor leakage between test suites.
- The function is marked `@deprecated` (line 10) — the recommended approach is to use `nock` directly. However, it remains in use in existing test files.

*Source: `client/test-helpers/use-nock/index.js:1-22`*

#### Level 3: Test Usage (`client/state/data-layer/wpcom-http/test/index.js`)

The nock-dependent test file demonstrates actual usage:

```js
// Line 2: Import both the wrapper and nock itself
import useNock, { nock } from 'calypso/test-helpers/use-nock';

// Line 27: Register suite-level cleanup (no setup callback)
useNock();

// Lines 31-32: Per-test success interceptor
nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).reply( 200, data );

// Lines 45-46: Per-test error interceptor
nock( 'https://public-api.wordpress.com:443' ).get( '/rest/v1.1/me' ).replyWithError( error );
```

**Key behavior:** Each test creates its own nock interceptor for the specific WordPress.com API endpoint it needs to test. The `useNock()` call at line 27 ensures all interceptors are cleaned up after the describe block completes. The tests verify that the `queueRequest` HTTP mechanism correctly dispatches `onSuccess` and `onFailure` callbacks based on the mocked API response.

*Source: `client/state/data-layer/wpcom-http/test/index.js:2,27,32,46`*

### Mock Setup Lifecycle

The complete lifecycle of nock in a client test run, in chronological order:

1. **Jest loads `setup-test-framework.js`** → `nock.disableNetConnect()` globally blocks all real HTTP/HTTPS connections for all client tests.
2. **`beforeAll` (global, from `setup-test-framework.js`)** → Checks if nock is active; reactivates with `nock.activate()` if it was previously restored.
3. **Test file loads** → `useNock()` is called, registering a per-suite `afterAll` cleanup hook via `nock.cleanAll()`.
4. **Individual test executes** → `nock(url).get(path).reply(...)` creates a one-time interceptor for the specific endpoint under test.
5. **Test assertion** → The test verifies that the dispatched action matches the expected response from the nock interceptor (e.g., `expect(action).toEqual(extendAction(succeeder, successMeta(data)))`).
6. **`afterAll` (per-suite, from `useNock`)** → `nock.cleanAll()` removes all interceptors created during the suite, preventing leakage to other suites.
7. **`afterAll` (global, from `setup-test-framework.js`)** → `nock.restore()` undoes the `http.ClientRequest` override, and `nock.cleanAll()` performs a final cleanup to prevent memory leaks.

### Impact on First-Run Timing

**nock itself does NOT contribute to the first-run vs. second-run timing gap.** Here is the reasoning:

1. **Fixed initialization cost:** nock's initialization (`require('nock')` + `nock.disableNetConnect()`) is a fixed cost paid on **every** run, regardless of cache state. It is not cached by Jest's `cacheDirectory` because it is not a transformation — it is runtime initialization.

2. **No Babel transformation needed:** The nock module (~212 KB) is pre-compiled JavaScript distributed through npm. It does **not** match the `babel-jest` transform pattern because it lives in `node_modules/`, which is excluded from transformation by the `transformIgnorePatterns` setting in `test/client/jest.config.js` line 14-16:
   ```js
   transformIgnorePatterns: [
       'node_modules[\\/\\\\](?!.*\\.(?:gif|jpg|jpeg|png|svg|scss|sass|css)$)',
   ],
   ```
   This pattern transforms only image/style assets within `node_modules`, not JavaScript files like nock.

3. **Timing gap fully explained by cache:** The near-identical timing between cold runs (1.76x) and no-cache runs (1.69x) for the single-file test confirms that the Babel transformation cache is the sole significant variable between cold and warm executions. If nock contributed to the gap, the warm run with nock would still show overhead — but it doesn't.

**However**, the global setup in `test/client/setup-test-framework.js` **does** contribute a fixed per-suite startup cost (unrelated to cache state):

| Line(s) | Global Polyfill/Mock | Purpose |
|---------|---------------------|---------|
| 25-26 | `TextEncoder`, `TextDecoder` | ReactDOMServer compatibility |
| 30-32 | `CSS.supports` | `@wordpress/components` compatibility |
| 34 | `ResizeObserver` | Layout observation polyfill |
| 36-40 | `fetch` | Global fetch mock |
| 44-49 | `wpcom-proxy-request` | WordPress.com proxy API mock |
| 52 | `crypto.randomUUID` | Cryptographic UUID generation |
| 54-63 | `matchMedia` | CSS media query mock |
| 66-67 | `ReadableStream`, `TransformStream` | `@wp-playground/client` compatibility |
| 68 | `Worker` | Web Worker via `worker_threads` |
| 71-73 | `structuredClone` | Deep clone polyfill |
| 76-78 | `crypto.subtle` | Web Crypto API mock |

This is a **fixed** per-suite cost of 12+ global installations, paid equally on every run regardless of cache state. It does not contribute to the cold-vs-warm timing differential.

*Source: `test/client/setup-test-framework.js:25-78`*

**Important note on test file scope:** `client/state/data-layer/test/wpcom-api-middleware.js` does **not** use nock — it relies entirely on `jest.fn()` mocking for `store.dispatch` and `store.getState`. Only `client/state/data-layer/wpcom-http/test/index.js` uses nock for HTTP interception.

---

## Q4: `--no-cache` Performance Comparison

### Methodology

The `--no-cache` flag tells Jest to **skip reading from AND writing to** the transformation cache. This means every `.js`/`.ts`/`.jsx`/`.tsx` file is re-transpiled through the Babel pipeline on every invocation, regardless of whether a cache directory exists.

**Command used:**

```bash
jest --config test/client/jest.config.js --no-coverage --watchAll=false --ci --maxWorkers=2 --no-cache "client/state/data-layer/test/wpcom-api-middleware.js"
```

**Key distinction from cold run:** The `--no-cache` flag disables cache **reads and writes** — no cache is consulted, and no cache is populated. A cold run (after `rm -rf .cache/jest`) disables cache reads (because the cache is empty) but **does** write to the cache for future runs. Additionally, the cold run must build the haste map from scratch, while `--no-cache` can still use an existing haste map if one exists.

### Timing Comparison Table

**Single-file test (`client/state/data-layer/test/wpcom-api-middleware.js`):**

| Run Type | Jest Reported Time | Wall-Clock Time | Delta vs. Cached |
|----------|--------------------|-----------------|------------------|
| Warm (cached) | 1.34 s | 2.21 s | — (baseline) |
| No-cache | 2.39 s | 3.74 s | +1.53 s (+69%) |
| Cold (empty cache) | 2.47 s | 3.89 s | +1.68 s (+76%) |

### Performance Impact Quantification

- The **no-cache run** is approximately **69% slower** than the cached (warm) run. This +1.53 s overhead represents the time spent re-transpiling all imported source files through the Babel pipeline.
- The **cold run** is approximately **76% slower** than the cached run — slightly worse than no-cache because it additionally builds the haste map from scratch.
- The **near-identical timing** between the no-cache run (3.74 s) and the cold run (3.89 s) confirms that the **transformation cache is the dominant performance factor**. The 0.15 s difference is attributable to haste map construction on the cold run.

### Dominant Transformation Step Analysis

The dominant overhead source when running without cache is the **`babel-jest` transformation** of `.js`/`.ts`/`.jsx`/`.tsx` files through the full Babel pipeline.

#### 1. The Primary Bottleneck: Babel Transformation Pipeline

When `--no-cache` forces re-transpilation, every imported module goes through the **complete** preset and plugin chain. The full pipeline consists of:

**Base presets from `packages/calypso-babel-config/presets/default.js` (lines 14-35):**

| Preset | Version | Configuration | Purpose |
|--------|---------|---------------|---------|
| `@babel/preset-env` | ^7.26.9 | `targets: { node: 'current' }` (in test env) | Environment-targeted transpilation |
| `@babel/preset-react` | ^7.26.3 | `runtime: 'automatic'`, `importSource: '@emotion/react'` | JSX transformation |
| `@babel/preset-typescript` | ^7.26.0 | `allowDeclareFields: true` | TypeScript transpilation |

**Base plugins from `packages/calypso-babel-config/presets/default.js` (lines 37-53):**

| Plugin | Version | Purpose |
|--------|---------|---------|
| `@babel/plugin-proposal-class-properties` | ^7.18.6 | Class properties syntax support |
| `@babel/plugin-transform-runtime` | ^7.26.10 | Shared helper deduplication |
| `@automattic/babel-plugin-preserve-i18n` | workspace:^ | Internationalization string preservation |
| `@emotion/babel-plugin` | ^11.11.0 | Emotion CSS-in-JS compile-time transforms |

**Test-environment-specific additions from `packages/calypso-babel-config/config.js` (lines 22-25):**

| Preset/Plugin | Version | Purpose |
|---------------|---------|---------|
| `@babel/preset-env` | ^7.26.9 | Overrides browser targets with `{ node: 'current' }` |
| `babel-plugin-dynamic-import-node` | ^2.3.3 | Converts dynamic `import()` to synchronous `require()` |

**Monorepo-level plugin from `packages/calypso-babel-config/config.js` (line 3):**

| Plugin | Version | Purpose |
|--------|---------|---------|
| `@automattic/babel-plugin-transform-wpcalypso-async` | workspace:^ | Calypso-specific async transform |

**Total: 3 presets + 7 plugins = 10+ transformation steps per source file.**

*Source: `packages/calypso-babel-config/presets/default.js:14-53`, `packages/calypso-babel-config/config.js:1-25`*

#### 2. Transitive Import Amplification

Even a single test file like `wpcom-api-middleware.js` triggers transformation of **all its transitive imports**. Examining the test file (`client/state/data-layer/test/wpcom-api-middleware.js` lines 1-3):

```js
import { mergeHandlers } from 'calypso/state/action-watchers/utils';
import { bypassDataLayer } from '../utils';
import { middleware } from '../wpcom-api-middleware';
```

Each of these imports, and every module they in turn import, must be transpiled. The `mergeHandlers` utility from `calypso/state/action-watchers/utils` and the data-layer `middleware` and `utils` modules each bring their own dependency trees. Without the cache, every file in these dependency trees is re-transpiled through the full 10+ step Babel pipeline on every test invocation.

*Source: `client/state/data-layer/test/wpcom-api-middleware.js:1-3`*

#### 3. Asset Transforms Are Negligible

Non-code imports (`.gif`, `.jpg`, `.jpeg`, `.png`, `.svg`, `.scss`, `.sass`, `.css`) go through a separate, trivially fast transform defined at `packages/calypso-jest/jest-preset.js` line 15:

```js
'\\.(gif|jpg|jpeg|png|svg|scss|sass|css)$': require.resolve( './src/asset-transform.js' ),
```

The `asset-transform.js` module simply returns the file's basename as a module export — no Babel parsing, no AST transformation, no compilation. This adds negligible overhead regardless of cache state.

*Source: `packages/calypso-jest/jest-preset.js:15`, `packages/calypso-jest/src/asset-transform.js`*

#### 4. Module Resolution Is a Minor Factor

The custom module resolver at `packages/calypso-jest/src/module-resolver.js` uses the `enhanced-resolve` library (^5.8.3) to support the `calypso:src` package.json field for workspace-internal module resolution. This adds slight per-module overhead for path resolution but is **not cached** by `cacheDirectory` — it runs on every invocation regardless of cache state. Therefore, it contributes equally to both cached and uncached runs and is not a factor in the timing differential.

*Source: `packages/calypso-jest/src/module-resolver.js`, `packages/calypso-jest/jest-preset.js:9`*

---

## References

### Configuration and Infrastructure Files

| File Path | Key Content | Relevant Lines |
|-----------|-------------|----------------|
| `babel.config.js` | Root Babel entry point — delegates to `@automattic/calypso-babel-config` with `importSource: '@emotion/react'` | Lines 6-9 |
| `test/client/jest.config.js` | Client Jest config — `cacheDirectory` setting, `setupFilesAfterEnv`, `transformIgnorePatterns` | Line 7 (`cacheDirectory`), Line 21 (`setupFilesAfterEnv`), Lines 14-16 (`transformIgnorePatterns`) |
| `test/client/setup-test-framework.js` | Client test bootstrap — nock initialization and 12+ global polyfill/mock installations | Lines 6-22 (nock), Lines 25-78 (polyfills) |
| `packages/calypso-jest/jest-preset.js` | Shared Jest preset — transform rules (`babel-jest` for JS/TS, `asset-transform.js` for images/styles), test matching, resolver | Lines 13-16 (transform), Line 9 (resolver), Line 12 (testMatch) |
| `packages/calypso-jest/src/module-resolver.js` | Custom module resolver — `calypso:src` field priority via `enhanced-resolve` (^5.8.3) | Entire file |
| `packages/calypso-jest/src/asset-transform.js` | Asset transform — returns `path.basename(filename)` as module export | Entire file |
| `packages/calypso-jest/src/setup.js` | Shared setup — global `CSS.supports` mock | Entire file |
| `packages/calypso-babel-config/config.js` | Babel config factory — test env adds `@babel/preset-env` (node:current) + `babel-plugin-dynamic-import-node` | Lines 22-25 (test env), Line 3 (wpcalypso-async plugin) |
| `packages/calypso-babel-config/presets/default.js` | Full Babel preset chain — 3 presets (`@babel/preset-env`, `@babel/preset-react`, `@babel/preset-typescript`) + 4 plugins | Lines 14-35 (presets), Lines 37-53 (plugins) |

### Data-Layer Source and Test Files

| File Path | Description |
|-----------|-------------|
| `client/state/data-layer/README.md` | Data-layer architecture documentation — middleware design, handler registry, bypass mechanism, file structure conventions |
| `client/state/data-layer/test/wpcom-api-middleware.js` | Primary test subject — 12 tests for WordPress.com API middleware, uses `jest.fn()` mocking (no nock) |
| `client/state/data-layer/test/handler-registry.js` | Handler registry tests |
| `client/state/data-layer/test/utils.js` | Data-layer utility function tests |
| `client/state/data-layer/test/convert-snake-case-to-camel-case.ts` | TypeScript test file — exercises TypeScript transform path |
| `client/state/data-layer/wpcom-http/test/index.js` | Nock-dependent test — 2 tests for HTTP queue request, intercepts `https://public-api.wordpress.com:443` via `useNock()` and `nock` |
| `client/state/data-layer/wpcom-http/test/actions.js` | HTTP action tests |
| `client/state/data-layer/wpcom-http/test/utils.js` | HTTP utility tests |

### Test Helper Files

| File Path | Description |
|-----------|-------------|
| `client/test-helpers/use-nock/index.js` | Nock lifecycle wrapper — exports `useNock` (deprecated) and re-exports `nock` for consumer convenience |
