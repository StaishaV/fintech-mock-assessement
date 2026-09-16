# Fintech QA Automation Framework

A Playwright + TypeScript test automation framework built against a mock fintech
system (user accounts + money transfers), covering API tests, UI tests, test
data management, and multi-format reporting.

## Approach & assumptions

The assessment describes a User Service, Transaction Service, and Notification
Service, but provides no running system — only endpoint shapes and example
payloads. Rather than stub responses with route interception, I built a small
**real** mock backend (`mock-server/`) and a matching **real** mock frontend
(`mock-frontend/`) that the backend serves. The trade-off: more setup time,
but the test suite exercises actual HTTP requests and a real DOM instead of
pre-canned fixtures, which is a much closer approximation of testing a real
system.

Scope deliberately **not** covered in this exercise (see "What I'd add with
more time" below): a persistent database, a real auth provider, the
Notification Service, contract tests between services, and load/performance
testing.

## Architecture

```
mock-server/         Express API (users, transactions) + serves mock-frontend
mock-frontend/        Plain HTML/JS UI that calls the mock API
config/environments/  baseURL per environment (local/staging), switched via TEST_ENV
utils/factories/      Test data builders (buildUser, buildTransaction) using faker
utils/helpers/        ApiClient (HTTP wrapper + request logging) and a custom
                       `api` fixture that provisions it per test
utils/assertions/     Custom Playwright matcher (toBeValidationError)
pages/                Page Object Model classes for the UI tests
tests/api/            API test suite
tests/ui/             UI test suite (drives the real mock frontend)
```

Why a mock server instead of pure stubbing: it lets both the API suite and
the UI suite exercise the same real backend, which is what actually catches
integration bugs (a stubbed UI test can't tell you the API contract changed
underneath it).

Why Playwright: one framework covers both API testing (`request` fixture)
and browser UI testing (`page` fixture), plus built-in multi-format
reporting, trace/screenshot capture on failure, and an extensible `expect`
— everything the brief asks for, without stitching together separate tools.

## Mock API design decisions worth noting

- **Auth**: the bearer token *is* the user's own id (`Authorization: Bearer <userId>`),
  required on both transaction endpoints. Simple, but real enough to test
  401 (missing) and 403 (wrong user) meaningfully.
- **Idempotency**: `POST /api/transactions` accepts an `Idempotency-Key`
  header; replaying the same key returns the original transaction (`200`)
  instead of creating a duplicate (`201`). This is the single most
  fintech-specific behavior in the suite — payment retries must not double-charge.
- **Transaction history is directional**: `GET /api/transactions/:userId`
  returns both sent and received transactions with a computed `direction`
  field, from one endpoint.
- **Empty history is `200 []`, not `404`** — a user with no transactions is a
  normal state, not an error.

## Running it

```bash
npm install
npx playwright test
```

That's it — `playwright.config.ts` auto-starts the mock server (`webServer`)
and waits for its `/health` endpoint before running anything, so there's no
separate manual step.

Other useful commands:
```bash
npm run mock:server          # run the mock backend + frontend on their own, e.g. to poke around at http://localhost:4000
npx playwright test tests/api           # API suite only
npx playwright test tests/ui            # UI suite only
npx playwright show-report              # open the last HTML report
TEST_ENV=staging npx playwright test    # point the suite at a different environment
                                         # (PowerShell: $env:TEST_ENV="staging"; npx playwright test)
```

A GitHub Actions workflow (`.github/workflows/playwright.yml`) runs the full
suite on push/PR and uploads the HTML report as a build artifact.

## Test coverage

**API — Users** (`tests/api/users.spec.ts`): create (valid, invalid email,
missing name, missing/invalid accountType, duplicate email → 409), get by id
(found, not found → 404).

**API — Transactions** (`tests/api/transactions.spec.ts`): create (valid,
negative amount, invalid type, unknown recipient, self-transfer), auth
(missing token → 401, wrong user's token → 403), idempotent replay, five
concurrent creates with no lost/duplicated records, get history (sent vs.
received direction, empty history, unknown user → 404).

**UI** (`tests/ui/`): registration (success, invalid-email error), transaction
flow (success + appears in history, negative-amount error, unknown-recipient
error).

**78 tests total** (26 unique cases × 3 browser engines), all passing.

## Reporting

- **HTML** report (visual, default `npx playwright show-report`)
- **JUnit XML** (`test-results/junit.xml`) for CI ingestion
- **List** reporter for plain console output
- **Trace + screenshot** captured automatically for any failing test
  (`retain-on-failure` / `only-on-failure`)
- **API request logging**: `ApiClient` logs `METHOD url -> status` for every
  call, visible per-test in the report

## Test data management

Every user/transaction is generated via `faker`-backed factories
(`utils/factories/`) with unique emails, so parallel test runs never collide
on the mock server's duplicate-email or existing-id checks. No shared
fixtures or seed data — every test creates exactly what it needs and asserts
only on that.

## A real bug this process caught

Early in building this, the `api` fixture reset the mock server's entire
in-memory store before every test (`POST /test/reset`), intended to guarantee
a clean slate. Under Playwright's default parallel workers, this caused a
genuine race condition: one worker's reset could wipe out data another
worker's test had just created and hadn't yet read back, producing
intermittent 404s that had nothing to do with the actual test logic. The fix
was to stop resetting per-test entirely — every test already creates its own
uniquely-generated data and only asserts on that, so a global reset was never
actually necessary, and removing it eliminated the race. Verified stable
across repeated runs afterward, including under the 5-concurrent-request
idempotency-adjacent test added later.

## What I'd add with more time

- **Contract tests** between the mock services (e.g., Pact) — this exercise
  only has one "service," but a real microservices system needs to verify
  the User/Transaction/Notification services agree on shapes independently
  of any one team's test suite.
- **Load/performance testing** (k6 or Artillery) against the transaction
  endpoint — the brief's scenario is a fintech transaction system, and
  throughput/latency under load is a real production concern that unit-style
  functional tests don't cover.
- **Real auth** (JWT or similar) in place of the "token = user id" mock, and
  a persistent datastore in place of the in-memory arrays.
- Malformed-JSON-body and rate-limiting edge cases.
- Visual regression and accessibility (axe) checks on the UI.
