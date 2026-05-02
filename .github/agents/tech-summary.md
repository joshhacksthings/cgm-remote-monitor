# Technical Summary

## Project Overview
- **Nightscout** (cgm-remote-monitor) is a real-time web application that displays continuous glucose monitoring (CGM) data for remote caregivers of diabetes patients.
- Node.js (16+) / Express / MongoDB / Socket.io / D3.js
- Full-stack web application: REST API server + single-page dashboard + report generator
- Version 15.0.6, licensed AGPL-3.0, 30+ language translations

## Repository Structure
- `server.js` — Root entrypoint, delegates to `lib/server/server.js`
- `lib/server/` — Express app bootstrap, environment loading, WebSocket setup, boot event chain
- `lib/api/`, `lib/api2/`, `lib/api3/` — Three API versions (legacy REST, enhanced REST, modern CRUD)
- `lib/plugins/` — ~37 plugins (glucose display, insulin calculations, alarms, device integrations)
- `lib/client/` — Client-side app controller, D3 chart rendering, WebSocket data receiver
- `lib/data/` — Data loading, delta calculation, treatment curve fitting
- `lib/authorization/` — JWT + API_SECRET auth, role-based access (Shiro-Trie), rate limiting
- `lib/middleware/` — Content negotiation, JSON response, device privacy, body parsing
- `lib/storage/` — MongoDB driver with retry/backoff, collection management
- `lib/report_plugins/` — 11 report types (daily stats, percentile, distribution, loopalyzer)
- `lib/admin_plugins/` — 6 admin operations (user management, DB cleanup)
- `bundle/` — Webpack entry points (main dashboard, clocks, reports)
- `webpack/` — Build config with Babel, loaders, moment.js timezone support
- `tests/` — 81 test files (Mocha + Should.js + Supertest)
- `testing/` — Test data generators (cosine wave CGM simulator, REST populator)
- `static/` — HTML, CSS, JS, audio, images, fonts served directly
- `views/` — Server-side templates
- `translations/` — i18n language files
- `docs/` — Example env template, issue template, plugin docs
- `bin/` — Shell/JS utilities (setup, data posting, release prep, SVG generation)

## Architecture and Execution Flow
- Boot chain: `server.js` → `lib/server/server.js` → `lib/server/env.js` (environment) → `lib/server/bootevent.js` (validates node version, connects MongoDB, initializes plugins, sets up bus) → `lib/server/app.js` (Express middleware, routes, static files) → `lib/server/websocket.js` (Socket.io)
- HTTP requests flow through Express middleware (helmet, compression, body parsing, auth) → API routers (v1/v2/v3 mounted at `/api/v1`, `/api/v2`, `/api/v3`) → MongoDB storage layer
- Plugin lifecycle: register → `setProperties(sbx)` (calculate data) → `checkNotifications(sbx)` (server alarms) → `updateVisualisation(sbx)` (client rendering)
- All plugins communicate through a sandbox (`sbx`) object providing lazy-evaluated properties, unit scaling, settings, and notification access
- Real-time: WebSocket (Socket.io) broadcasts data deltas to connected clients; clients merge new data and re-render D3 charts
- Event bus (`lib/bus.js`) drives heartbeat ticks (60s default), notification dispatch, and teardown
- Alarms: `lib/notifications.js` manages URGENT/WARN/INFO levels with silence timers, acknowledgment, and snooze

## Notable Patterns and Decisions
- **Plugin factory pattern**: Every plugin exports `function init(ctx)` returning an object with lifecycle methods
- **Sandbox isolation**: Plugins never access global state directly; all data/settings/notifications go through `sbx`
- **Lazy property evaluation**: `sbx.offerProperty(name, fn)` defers computation until first access
- **Three API generations**: v1 (legacy REST with file-extension content negotiation), v2 (enhanced with properties/auth), v3 (modern CRUD with Swagger, deduplication, generic handlers)
- **Extended settings convention**: `PLUGIN_SETTING_NAME` env vars auto-convert to `plugin.settingName` via camelCase transformation
- **Content negotiation via file extension**: API v1 supports .json, .csv, .svg, .txt, .png, .html, .tsv responses
- **Dual auth**: API_SECRET hash-based (legacy) + JWT Bearer tokens (modern), both supported simultaneously
- **Environment-driven auto-registration**: Setting `PUSHOVER_API_TOKEN` automatically enables the pushover plugin
- **Require cache clearing in tests**: Global `afterEach` hook in `tests/hooks.js` clears `require.cache` to prevent state leakage
- **comma-first code style** per `.jsbeautifyrc`

## Developer Workflow
- `npm start` — Production server
- `npm run dev` — Development server with nodemon + inspector (port 1337)
- `npm test` — Mocha tests with 5s timeout, using `my.test.env`
- `npm run test-ci` — NYC coverage with `tests/ci.test.env` (MongoDB required)
- `npm run lint` — ESLint on `lib/` only
- `npm run bundle` / `npm run bundle-dev` — Webpack production/development builds
- `postinstall` hook auto-runs bundling and generates cache-busting random string
- MongoDB required for all tests (no mock-only option in CI)
- CI: GitHub Actions matrix — Node 20/22 × MongoDB 4.4/5.0/6.0
- Docker: `docker-compose.yml` provides Nightscout + MongoDB + Traefik reverse proxy
- Also deployable to Heroku (Procfile + app.json), Azure (azuredeploy.json + web.config), Vagrant

## Contributor Notes
- Start reading at `lib/server/bootevent.js` to understand the initialization chain, then `lib/plugins/index.js` for the plugin system
- Plugin development: follow the factory pattern in any existing plugin under `lib/plugins/`; register in `lib/plugins/index.js`
- Tests follow two styles: legacy callback-based (API v1/v2) and modern async/await (API v3 with fixture-based instance creation)
- API v3 test fixtures in `tests/fixtures/api3/` provide full app factories, auth helpers, and cache monitors
- ESLint uses `eslint-plugin-security` — security rules are enforced in `lib/`
- Code style: 2-space indentation, comma-first formatting
- The `testing/` directory contains data generators useful for local development (cosine-wave CGM simulator)

## Open Questions / Ambiguities
- `tests/client.test.js.temporary_removed` — client-side tests have been disabled with no documented reason; unclear if the `benv` browser mocking approach is viable or permanently abandoned
- `.nvmrc` pins Node 16.16.0 but CI tests against 20/22 and Docker uses 22 — unclear which version local developers should target
- `tests/ci.test.env` sets `NODE_ENV=production` — unclear if intentional (to validate production paths) or an oversight
- The Makefile duplicates several npm script commands with different timeouts and coverage tools — unclear which is canonical
- `event-stream` pinned to exact version 3.3.4 in dependencies — likely a response to the 2018 supply chain attack, but unclear if the dependency is still needed
