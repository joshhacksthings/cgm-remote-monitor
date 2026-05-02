# Tech Debt Assessment

Stack-ranked from highest to lowest priority. All items are grounded in repository evidence.

---

## 1. .nvmrc Pins Node 16 While CI and Docker Use Node 20/22

**Why this matters**
- Local developers using `nvm use` get Node 16.16.0. CI runs 20 and 22. Dockerfile builds with 22-alpine. Local development and production are on different major Node versions, causing "works on my machine" failures.

**Evidence**
- `.nvmrc`: `16.16.0`
- `package.json` engines: `"node": ">=16.x"`
- `.github/workflows/main.yml` matrix: `[20, 22]`
- `Dockerfile` line 1: `FROM node:22-alpine`

**Why this is easy to review**
- Single-line change to `.nvmrc`. Reviewer can cross-reference CI matrix and Dockerfile instantly.

**Suggested smallest fix**
- Update `.nvmrc` to `22` (matching Dockerfile and latest CI target). Optionally update `package.json` engines to `">=20.x"`.

**How the community can validate it**
- Run `nvm use && node -v` and confirm it matches CI. Run `npm test` locally with the new version.

**Estimated lift**
- Tiny

---

## 2. Disabled Client Test Suite With No Documented Reason

**Why this matters**
- `tests/client.test.js.temporary_removed` is ~130 lines of legitimate client tests (initialization, data updates, unit conversion) preserved under a "temporary" rename. There is zero client-side test coverage. The name implies it was meant to be restored, but it has been left indefinitely.

**Evidence**
- File exists at `tests/client.test.js.temporary_removed` with full Mocha `describe`/`it` blocks
- Uses `benv` for browser environment mocking
- No other client-side test file exists in the test suite

**Why this is easy to review**
- Binary decision: either delete the dead file (tiny PR) or restore it with a fix for whatever broke it (small PR with clear scope).

**Suggested smallest fix**
- Option A: Delete the file and open a tracking issue for client test coverage.
- Option B: Rename back to `.test.js`, run tests, fix whatever broke, and restore coverage.

**How the community can validate it**
- `npm test` with the file renamed back to `.test.js` — either it passes or the failure reveals what needs fixing.

**Estimated lift**
- Tiny (delete) / Small (restore)

---

## 3. ESLint Only Covers `lib/` — Tests and Scripts Are Unlinted

**Why this matters**
- The lint script targets `lib/` only. The `tests/` directory (81 files), `bin/` (10 scripts), `testing/` (5 files), and `webpack/` config are never linted. The security plugin (`eslint-plugin-security`) only catches issues in `lib/`, leaving test utilities and build scripts unchecked.

**Evidence**
- `package.json` lint script: `"lint": "eslint lib"`
- `.eslintrc.js` has no `ignorePatterns` or multi-directory config
- Only override is for `lib/client/*.js` files

**Why this is easy to review**
- One-line change to the lint script. Any new lint errors in tests/bin are separate follow-up PRs.

**Suggested smallest fix**
- Change lint script to `"eslint lib tests bin testing"`. Add a `--max-warnings` flag or address any immediate errors in a follow-up.

**How the community can validate it**
- Run `npm run lint` before and after the change. Count how many new issues surface.

**Estimated lift**
- Tiny (script change) / Small (if fixing initial lint errors)

---

## 4. Deprecated `babel-eslint` Parser

**Why this matters**
- `babel-eslint` was deprecated and replaced by `@babel/eslint-parser`. The deprecation warning is explicit in `package-lock.json`: `"deprecated": "babel-eslint is now @babel/eslint-parser. This package will no longer receive updates."` Using a dead package means no bug fixes or security patches for the linting parser.

**Evidence**
- `package.json` devDependencies: `"babel-eslint": "^10.1.0"`
- `.eslintrc.js` line 9: `'parser': 'babel-eslint'`
- `package-lock.json` contains explicit deprecation message

**Why this is easy to review**
- Swap one dependency, change one line in `.eslintrc.js`. Run `npm run lint` to confirm parity.

**Suggested smallest fix**
- `npm uninstall babel-eslint && npm install --save-dev @babel/eslint-parser`. Update `.eslintrc.js`: `'parser': '@babel/eslint-parser'`.

**How the community can validate it**
- `npm run lint` passes with identical results before and after.

**Estimated lift**
- Tiny

---

## 5. Deprecated `request` HTTP Library Still in Production Dependencies

**Why this matters**
- The `request` npm package has been deprecated since February 2020. It is still used in production code paths for Pushover notifications and Maker/IFTTT webhooks. The project already has `axios` in devDependencies, so a modern alternative is available.

**Evidence**
- `package.json`: `"request": "^2.88.0"` in dependencies
- `lib/plugins/maker.js` line 5: `require('request')`
- `lib/plugins/pushover.js` line 5: `require('request')`
- `axios` already present in devDependencies

**Why this is easy to review**
- Two plugin files use `request`. Each is a narrow HTTP call. Migrating to `axios` or Node's built-in `fetch` (available in Node 18+) is a well-understood pattern.

**Suggested smallest fix**
- Replace `request` usage in `maker.js` and `pushover.js` with `axios` or built-in `fetch`. Remove `request` from dependencies.

**How the community can validate it**
- Unit tests for those plugins pass. Manual test: configure a Pushover or Maker webhook and trigger a notification.

**Estimated lift**
- Small

---

## 6. docker-compose.yml Pins MongoDB 4.4 (EOL Since February 2024)

**Why this matters**
- `docker-compose.yml` uses `image: mongo:4.4`, which reached end-of-life. CI already tests against 4.4, 5.0, and 6.0 — so the app is known to work on newer versions. Shipping a default compose file with an EOL database version exposes self-hosters to unpatched vulnerabilities.

**Evidence**
- `docker-compose.yml` line 12: `image: mongo:4.4`
- `.github/workflows/main.yml` matrix: `[4.4, 5.0, 6.0]` — proving compatibility with newer versions
- `docker-compose.yml` line 27: `image: traefik:latest` (unpinned)

**Why this is easy to review**
- Single-line version bump. CI already validates compatibility with 6.0.

**Suggested smallest fix**
- Change `image: mongo:4.4` to `image: mongo:6.0`. Pin Traefik to a specific version (e.g., `traefik:v2.11`).

**How the community can validate it**
- `docker-compose up` starts successfully. Run any test against the compose stack.

**Estimated lift**
- Tiny

---

## 7. Makefile Duplicates npm Scripts With Different Timeouts and Coverage Tools

**Why this matters**
- The Makefile defines test/coverage targets using `istanbul` with 30-second timeouts. `package.json` uses `nyc` with 5-second timeouts. Contributors don't know which is canonical. Different timeouts mean flaky tests may pass in one runner and fail in the other.

**Evidence**
- `package.json`: `"test": "env-cmd ... mocha --timeout 5000"`
- `Makefile`: `${MOCHA} --timeout 30000`
- `Makefile`: Uses `istanbul` for coverage; `package.json` uses `nyc`
- CI uses npm scripts (not Make targets)

**Why this is easy to review**
- Either remove the Makefile test targets (since CI uses npm scripts) or align them. Reviewer can verify that `npm test` and `make test` produce the same results.

**Suggested smallest fix**
- Add a comment to the Makefile noting it is legacy, or remove the duplicated targets and point to npm scripts.

**How the community can validate it**
- Run `npm test` and `make test` side-by-side. Confirm CI only uses npm scripts.

**Estimated lift**
- Tiny

---

## 8. Dockerfile Missing HEALTHCHECK Instruction

**Why this matters**
- The Dockerfile has no `HEALTHCHECK` instruction. Container orchestrators (Docker Compose, Kubernetes, ECS) cannot auto-detect when the app is unhealthy and restart it. The app already exposes `/api/v1/status.json` which is a natural health endpoint.

**Evidence**
- `Dockerfile`: No `HEALTHCHECK` instruction present
- `lib/api/status.js`: Status endpoint exists at `/api/v1/status.json`
- `Dockerfile` lines 8-10: Multiple TODO comments about production optimization

**Why this is easy to review**
- Single line addition. Uses an existing endpoint.

**Suggested smallest fix**
- Add `HEALTHCHECK --interval=60s --timeout=10s CMD wget -q --spider http://localhost:1337/api/v1/status.json || exit 1`

**How the community can validate it**
- `docker build` and `docker inspect --format='{{.Config.Healthcheck}}' <image>` confirms the health check is registered. `docker ps` shows health status.

**Estimated lift**
- Tiny

---

# Best First PR Candidates

1. **Update .nvmrc to match CI/Docker (Item 1)** — Single-line change, zero risk, immediately obvious value. Any contributor can do this.
2. **Replace deprecated babel-eslint (Item 4)** — Well-documented migration path, one dependency swap + one config line. Linting output stays identical.
3. **Bump MongoDB in docker-compose.yml (Item 6)** — Single-line version bump, CI already proves compatibility with newer versions.
