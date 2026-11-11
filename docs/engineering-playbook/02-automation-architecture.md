# Module 2: Automation Framework Architecture & Design Patterns

This module describes a maintainable Playwright and TypeScript framework for
enterprise suites such as TaskFlow or Pro-Lan. The goal is fast, trustworthy
feedback over a small set of stable user journeys, not a browser script for
every line of application code.

## 1. Architecture goals and test layers

An automation framework should provide:

- fast feedback at the appropriate test layer;
- isolated data and authentication;
- readable tests that express business intent;
- deterministic synchronization rather than arbitrary delays;
- actionable failure diagnostics;
- repeatable local and CI execution;
- safe cleanup and clear ownership of test infrastructure.

Use the smallest layer that proves the behavior:

- **Unit tests:** deterministic domain logic and transformations.
- **API/component tests:** validation, service contracts, and persistence behavior.
- **Integration tests:** real database, queue, identity, and external boundaries
  where mocks would hide important behavior.
- **End-to-end tests:** a small number of high-value browser journeys.
- **Manual/exploratory testing:** usability, visual judgment, and scenarios that
  are expensive or unsuitable for automation.

The layers are complementary. A passing browser test should not be used to
justify omitting focused API or domain tests.

## 2. Hybrid API-UI architecture

### Principle

Use authenticated API clients to provision heavy prerequisites, then use the UI
only for the behavior under test. Departments, user personas, teams, programs,
projects, and baseline tasks should normally be created through a typed
`RuntimeApiClient`, not through a sequence of slow setup forms.

This keeps a UI test focused on user interactions, state transitions, board
filtering, permissions, and business workflows. It also makes failures easier
to classify: setup failures are separated from product interaction failures.

### Recommended flow

```text
fixture -> API setup -> storage state -> browser journey -> API assertions
       -> resource registry -> reverse-order teardown
```

Use the UI for setup only when the UI itself is the feature being tested, or
when the API path would bypass an important business rule.

### Typed API client boundaries

Keep request and response models explicit. Centralize authentication, base URL
normalization, headers, timeouts, error parsing, and correlation IDs in the
client. A failed setup call should include method, sanitized URL, status,
response body summary, and the test data identifier; it should not silently
return a success-shaped fallback.

## 3. Page Object Model and component encapsulation

The Page Object Model (POM) separates test scenarios from page-specific
locators and mechanics. A `LoginPage`, `DashboardPage`, or `ProjectPage`
encapsulates the controls and actions for that surface. If a button role or
test ID changes, the page object is updated instead of every spec.

### Design rules

- Keep locators private to the page or component that owns them.
- Expose meaningful actions such as `createProject()` or `moveTaskToReview()`.
- Put business assertions in domain-aware helpers when they are reused.
- Keep generic browser mechanics in fixtures or utilities.
- Avoid page objects that become giant “god classes.”
- Return the next page or component when navigation is guaranteed.
- Keep API fixtures and browser page objects separate.

### Scoped locators

Scope locators to semantic boundaries such as `main`, `aside`, a dialog, or a
specific card. This prevents duplicate-element collisions and makes intent
clear. Prefer role, label, and stable test IDs over absolute XPath or generated
CSS classes. A selector should identify the intended control, not its current
position in the DOM.

## 4. Isolated storage contexts and persona partitioning

Each test should receive an isolated Playwright browser context:

```ts
const context = await browser.newContext({
  storageState: authFile,
});
const page = await context.newPage();
```

Context isolation prevents cookies, local storage, permissions, and session
tokens from leaking between tests. Use separate storage states for personas
such as Member, Manager, Org Head, and Super Admin. A Member must never inherit
an administrative session from a previous test or worker.

Isolation also applies to:

- unique database records and resource names;
- feature flags and tenant identifiers;
- temporary files and download directories;
- service ports and mock servers;
- API rate-limit budgets.

Parallel execution is safe only when all mutable dependencies are isolated or
explicitly partitioned.

## 5. Authentication setup and session caching

Authentication is infrastructure, not the business behavior of every test.
`global-setup.ts` can perform a baseline login once and write storage states
(cookies and local storage) to a controlled `.auth/` directory. Functional
tests then consume those states rather than repeating UI login.

### Cache policy

Use an explicit switch such as `REUSE_AUTH=1` to reuse valid state and
`REUSE_AUTH=0` to force refresh. Cached state must be:

- excluded from source control;
- scoped to the environment and persona;
- protected from accidental publication in artifacts;
- refreshed when credentials, roles, or application builds change;
- validated for expiry before use.

Do not treat a cached session as proof that authentication is healthy. Keep a
small dedicated login test and refresh-state path in CI.

## 6. Serverless pacing and concurrency controls

Serverless databases and Prisma-backed functions may have strict connection,
transaction, and invocation limits. Fast API setup can overwhelm a connection
pool even when each individual request is valid.

### Dataset pacing

Introduce a controlled gap such as `DATASET_TASK_GAP_MS` between dependent or
batch entity creations when the environment requires it. The value should be
configuration, not a hidden sleep scattered through tests. Prefer batching,
idempotent setup endpoints, and bounded polling where the API supports them.

### Worker policy

Use conservative concurrency in pre-production, for example `workers: 1`,
when transactional stability matters more than throughput. Increase workers
only after measuring connection usage, rate limits, test duration, and failure
rates. Document environment-specific worker settings in the Playwright
configuration and CI pipeline.

Pacing must not mask a production race or make a fundamentally unsafe endpoint
appear reliable. Record setup failures and investigate repeated 500, timeout,
deadlock, or Prisma transaction errors.

## 7. Commit-level navigation and explicit polling

### Standard navigation

Centralize navigation behavior in configuration or helpers. A slow application
may benefit from `waitUntil: 'commit'` and an extended, bounded timeout such as
45 seconds when waiting for the initial document response. This confirms that
navigation committed without requiring every asset, websocket, or long-running
request to finish.

Do not use a large timeout to hide an unavailable service. Pair it with clear
failure diagnostics and a health check.

### Dynamic polling

Modern client-side applications may render a shell, hydrate, fetch data, and
replace nodes. Use Playwright's retrying assertions and bounded polling:

```ts
await expect(async () => {
  await expect(page.getByTestId('task-count')).toHaveText('3');
}).toPass({ timeout: 15_000 });
```

Prefer waiting for a meaningful condition: response, URL, visibility, enabled
state, state transition, or business value. Avoid fixed sleeps because they
are either unnecessarily slow or unreliable under load.

## 8. Stateful resource registry and teardown

Test environments degrade when generated projects, tasks, teams, and users are
never removed. A run-level resource registry should record every created entity:

```text
organization -> program -> project -> task
                       -> team membership
```

Register resources immediately after successful creation and persist enough
metadata to clean them up if a later setup step fails. Global teardown should
traverse the registry in strict reverse dependency order. Cleanup should be:

- bounded and observable;
- idempotent when possible;
- tolerant of already-removed records;
- explicit about failures;
- isolated from production data by tenant and environment safeguards.

Never use an unscoped “delete all” cleanup operation. If cleanup fails, publish
the resource IDs and environment as diagnostics so an owner can remove them
safely.

## 9. Centralized configuration and message dictionaries

### Environment mapping

Keep base URLs, API URLs, viewport settings, timeouts, worker policies, pacing,
and feature flags in typed environment mapping such as `environments.ts`.
Validate required variables at startup and fail clearly for an unknown
`TEST_ENV`. Avoid silently falling back to a development endpoint.

Use one canonical URL builder to normalize trailing slashes and route joining.
This prevents mismatches such as `/api/departments/` versus
`/api/departments` across environments.

### Message dictionaries

Centralize intentional UI copy, labels, validation messages, and alert banners
in a typed dictionary such as `config/messages.ts`. Keep selectors separate
from copy when localization or wording is expected to change. Assertions should
verify user-visible contracts without duplicating strings across dozens of specs.

Do not centralize values merely to hide unclear test intent. A local assertion
is appropriate when the text is unique to one scenario.

## 10. Modular regression slicing and tags

Large suites should be split by risk and execution purpose. A practical layout
may include:

- `00-06`: core platform and authorization specs;
- `07-10`: work management and insights;
- `99-e2e`: complete cross-module journeys.

Use tags such as `@sanity`, `@regression`, `@rbac`, and `@destructive` to select
appropriate CI slices. Tags should describe behavior or risk, not individual
developers. A test may belong to more than one slice.

Every slice should document:

- its purpose and owning team;
- expected duration;
- required environment and data;
- whether it can run in parallel;
- the command used locally and in CI;
- its release gate status.

Keep a small, fast sanity slice for pull requests and run broader regression
before merge or release. Do not let slicing hide failures by excluding a
relevant test from the default pipeline.

## 11. Known-issue control with `test.fixme`

Use `test.fixme` only for a confirmed upstream product defect that prevents
meaningful execution. Each exclusion must include:

- a defect or issue reference;
- the affected behavior;
- an owner;
- the date added;
- an expiry or review date;
- the expected removal condition.

Do not use `fixme` for flaky tests, missing setup, poor selectors, or failures
that the test owner has not investigated. A known issue should remain visible
in reporting and backlog metrics even when it is excluded from a gate. Review
the list regularly so temporary masking does not become permanent coverage loss.

## 12. Fixtures and dependency ownership

Use fixtures to compose stable setup and cleanup boundaries:

- browser and context fixtures own lifecycle;
- persona fixtures own authentication state;
- API fixtures own domain setup;
- data fixtures own unique records and registry registration;
- page fixtures own page-object construction.

Avoid fixtures with hidden side effects. Name fixtures by the capability they
provide and keep their scope explicit (`test`, `worker`, or global). A worker
fixture must not mutate data that another worker can observe.

## 13. Failure diagnostics and artifact hygiene

On failure, preserve enough evidence to reproduce the issue:

- trace and video for browser failures;
- screenshot and DOM snapshot at the failing step;
- console errors and failed network requests;
- request correlation ID and API response summary;
- persona, environment, worker, and test data IDs;
- application logs for the corresponding time window.

Redact tokens, cookies, personal data, and connection strings before publishing
artifacts. Keep artifacts tied to the exact build, browser version, and commit.

## 14. CI execution strategy

Run formatting, linting, type checks, and unit tests on every change. Run
targeted API and browser tests for affected areas, then run broader regression
on merge or before release. Pin browser and dependency versions where
reproducibility matters, and publish JUnit or Playwright reports as artifacts.

CI should fail clearly for:

- missing required configuration;
- invalid or expired authentication state;
- setup or teardown failures;
- infrastructure health failures;
- test assertion failures.

Retries may help diagnose transient infrastructure problems, but a retry must
not convert a product failure into a pass. Track retry outcomes separately.

## 15. Framework governance checklist

Before adding a test:

- [ ] The requirement and risk are identified.
- [ ] A lower test layer cannot prove the behavior more reliably.
- [ ] Setup uses typed APIs where UI setup is not the subject of the test.
- [ ] Data, persona, and environment isolation are explicit.
- [ ] Locators are semantic, scoped, and maintainable.
- [ ] Assertions verify business outcomes rather than implementation details.

Before merging framework changes:

- [ ] Authentication and cleanup paths are tested.
- [ ] Worker, pacing, timeout, and retry behavior is documented.
- [ ] Failure artifacts contain useful evidence without secrets.
- [ ] Tags and suite slices still provide the intended release gates.
- [ ] Known issue exclusions have owners and review dates.
- [ ] Targeted tests, type checks, and linting pass.
