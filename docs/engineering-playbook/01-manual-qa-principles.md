# Module 1: Manual QA Principles, Test Design & Quality Analytics

## 1. Quality is an engineering responsibility

Quality assurance is not a final-stage activity or a synonym for clicking
through a checklist. It is the disciplined reduction of uncertainty about
user-visible behavior, data integrity, security, reliability, accessibility,
and operability.

QA should participate from requirement refinement through release verification.
The best test strategy is risk-based: spend the most effort where failure would
cause the greatest customer, security, financial, or operational impact.

### Core test-design principles

- Start with business rules and user journeys, not implementation details.
- Separate happy paths, validation paths, authorization paths, and recovery paths.
- Test observable outcomes: UI state, API response, persisted data, events, and
  user notifications.
- Make test data deterministic and document its owner, setup, and cleanup.
- Test boundaries explicitly: empty values, limits, duplicate actions, time
  zones, slow networks, expired sessions, and partial failures.
- Treat accessibility, privacy, and security behavior as acceptance criteria.
- Use the smallest reliable test layer that proves each requirement.

## 2. Advanced test design techniques

Test design should minimize redundant cases while maximizing the probability of
detecting meaningful defects.

### Equivalence Partitioning (EP)

Divide an input or state domain into classes that should behave equivalently,
then select representative values from each class. Every valid and invalid
partition needs coverage.

Example: a task priority field may have valid partitions of `Low`, `Medium`,
`High`, and `Urgent`, plus invalid partitions such as blank, unsupported text,
and a null value. A single representative from each partition is not enough if
the application has different behavior for API input and UI input; partition
the interfaces separately when their contracts differ.

### Boundary Value Analysis (BVA)

Test the edges where defects cluster. For a field with a range from 1 to 100,
use values such as:

| Boundary | Example |
|---|---:|
| Just below minimum | 0 |
| Minimum | 1 |
| Nominal value | 50 |
| Maximum | 100 |
| Just above maximum | 101 |

Apply the same approach to date pickers, pagination limits, string lengths,
upload sizes, retry counts, and concurrency limits. Include the representation
of each value, such as a date in the user's time zone versus UTC.

### Decision table testing

Use decision tables when multiple conditions combine to produce different
outcomes. For example:

| Role | Task status | Due date passed | Expected action |
|---|---|---|---|
| Member | In Review | Yes | Disable edit |
| Member | In Review | No | Allow permitted edit |
| Manager | In Review | Yes | Allow manager override |
| Member | Done | No | Disable edit |

The table should identify both valid combinations and combinations that must be
rejected. It is especially useful for pricing, permissions, workflow gates, and
feature flags.

### State transition testing

Model the system as states, events, guards, and outcomes. A task lifecycle may
be:

`To Do -> In Progress -> In Review -> Done`

Cover valid transitions, invalid transitions, repeated events, cancellation,
reopening, concurrent updates, and transitions attempted by unauthorized roles.
For each transition, verify the state shown in the UI, the API response, the
persisted value, and any emitted notification or audit record.

### Pairwise and combinatorial coverage

When many independent options exist, exhaustive testing can be impractical.
Pairwise testing covers every pair of option values at least once and can expose
interaction defects with fewer cases. Do not use pairwise coverage as a
replacement for mandatory business-rule combinations, security cases, or
high-risk end-to-end workflows.

## 3. Coverage matrices and requirement traceability

### Coverage matrix

A coverage matrix maps business requirements to test cases, execution status,
and defects. It is a release-control artifact, not merely a count of tests.

| Requirement ID | Requirement description | Test case ID | Test type | Execution status | Defect reference |
|---|---|---|---|---|---|
| REQ-PRJ-01 | Department auto-selects inside program view | TC-PRJ-01 | Functional/UI | Passed | None |
| REQ-TSK-04 | Member cannot edit task properties | TC-TSK-12 | Security/RBAC | Failed | BUG-1042 |
| REQ-TSK-07 | Done tasks cannot transition to In Progress | TC-TSK-18 | State transition | Not run | None |

At minimum, track requirement ID, risk, acceptance criteria, test case IDs,
execution status, environment/build, and defect references. Mark a requirement
as covered only when its required positive, negative, and authorization cases
have evidence.

### Requirement Traceability Matrix (RTM)

Traceability has two directions:

- **Forward traceability:** requirement -> acceptance criterion -> test case ->
  execution evidence -> defect or release decision.
- **Backward traceability:** test case or implementation behavior -> requirement,
  proving that the work is intentional and not an undocumented “ghost feature.”

When an API, UI component, permission rule, or workflow changes, perform impact
analysis by identifying the affected requirements, test data, fixtures, mocks,
browser journeys, API contracts, and monitoring signals. Update the matrix in
the same change as the requirement or implementation; stale traceability is
misleading evidence.

## 4. Exploratory testing

Exploratory testing combines learning, test design, and execution. Time-box a
session and document:

- charter, scope, and risk area;
- environment, build, browser/device, flags, and data;
- paths explored and notable observations;
- failures, hypotheses, and evidence captured;
- follow-up scenarios and defects;
- session duration and tester.

Useful tours include boundary-value, error-message, workflow, interoperability,
accessibility, recovery, security, localization, and high-volume data tours.
Exploration complements scripted regression; it does not replace traceability
for committed requirements.

## 5. Defects and root-cause analysis

### Defect reports

A useful defect report is reproducible and actionable:

1. Summarize the observed behavior and affected area.
2. Record environment, build, browser/device, account role, and feature flags.
3. Give exact preconditions and reproduction steps.
4. State expected and actual results separately.
5. Attach screenshots, traces, console output, network details, request IDs, or
   sanitized data identifiers.
6. Describe impact, frequency, severity, and priority.

Distinguish a confirmed root cause from a hypothesis. Do not infer a backend
failure from a UI symptom without checking the network response and logs.

### Five Whys example

**Symptom:** `SyntaxError: Unexpected token '<'` during global test setup.

1. Why did the JSON parser fail? `RuntimeApiClient` received HTML.
2. Why was HTML returned? The API endpoint returned a 404 page.
3. Why did the endpoint return 404? The request included a trailing slash.
4. Why was the slash accepted by the client but not the route? The URL
   construction contract differed between environments.
5. Why was the mismatch not detected earlier? URL normalization and API
   contract checks were missing from test setup.

**Root cause:** an environment configuration contract mismatch between base URL
formatting and backend route matching. A corrective action should address the
systemic gap, such as central URL normalization plus a contract test, rather
than only changing one test fixture.

### RCA methods

Use the method that fits the failure:

- **Five Whys:** process or configuration failures with a clear causal chain.
- **Fishbone analysis:** multi-factor failures across people, process, tools,
  environment, data, and design.
- **Fault tree analysis:** combinations of conditions that produce a critical
  failure.
- **Change-point analysis:** regressions introduced by a particular commit,
  release, flag, dependency, or environment change.

Categorize defects consistently, for example: UI/DOM drift, API contract
mismatch, RBAC permission gap, state-machine violation, data integrity,
environment configuration, flaky synchronization, or performance regression.
Use categories to improve upstream engineering practices, not to assign blame.

## 6. DOM and element-selection knowledge

QA engineers working with browser automation should understand the browser's
rendered DOM, accessibility tree, shadow DOM boundaries, iframe boundaries, and
asynchronous re-rendering.

### Locator hierarchy

Prefer selectors that express user intent and survive presentation changes:

1. Accessible role and accessible name.
2. Associated label or form control name.
3. Stable, purpose-built test ID such as `data-testid`.
4. Stable semantic attributes.
5. Text matching when the text is an intentional contract.

Avoid absolute XPath such as `/html/body/div[1]/div[2]/button` and brittle CSS
classes generated by a styling system. A locator should identify the intended
control, not its current position in the DOM.

### Asynchronous rendering

Modern SPAs may render a shell, hydrate client-side code, fetch data, and then
replace nodes. Tests must wait for meaningful state rather than arbitrary time:

- wait for the UI to become enabled or visible;
- wait for a specific response or URL;
- poll for a business state with a bounded timeout;
- use framework-supported assertions that retry;
- capture the DOM, console, and network state when a wait expires.

Race conditions often occur when a test interacts before hydration, data
fetching, or animation has completed. In Playwright, prefer web-first
assertions and bounded polling such as `expect(...).toPass()` where appropriate.

For shadow DOM and iframes, use framework-supported boundary handling and verify
that the target element is in the expected browsing context. Do not bypass
accessibility behavior merely to make a selector pass.

## 7. Quality engineering analytics

Metrics should support decisions and reveal systemic risk. Never optimize for
test-count growth or pass rate if doing so hides instability.

### Useful measures

- **Defect density:** confirmed defects per module, feature, or normalized size.
  Compare trends carefully; raw LOC is only a rough denominator.
- **Defect leakage:** production or later-stage defects divided by total
  confirmed defects for the release or period.
- **Flaky-test rate:** nondeterministic failures divided by total executions,
  tracked separately from legitimate product failures.
- **Automation ROI:** time saved and risk reduced compared with authoring,
  maintenance, infrastructure, and triage costs.
- **Requirement coverage:** high-risk requirements with passing evidence divided
  by high-risk requirements in scope.
- **Mean time to verify:** time from fix delivery to validated result.
- **Release Readiness Index:** a documented composite of critical-path coverage,
  automated sanity status, open critical/high defects, known-risk acceptance,
  and operational readiness.

Define formulas, data sources, time windows, and owners before publishing a
metric. A metric without a decision threshold is a report, not a control.

### Release readiness example

A release is not ready solely because the suite is green. Confirm that:

- critical and high-risk requirements have current evidence;
- critical workflows pass in the candidate build;
- no unaccepted critical defects remain;
- high defects have an explicit owner and release decision;
- flaky failures are investigated rather than silently retried;
- monitoring, rollback, migration, and support procedures are ready;
- test data and environment match the intended deployment conditions.

## 8. QA working checklist

Before testing:

- [ ] Requirements and acceptance criteria are unambiguous.
- [ ] Risk, role, state, data, and environment assumptions are documented.
- [ ] Positive, negative, boundary, and recovery cases are selected.
- [ ] Requirement IDs map to test cases in the coverage matrix.

During testing:

- [ ] Evidence identifies the exact build and environment.
- [ ] Failures include request, console, DOM, and server evidence where relevant.
- [ ] Defects distinguish symptoms, hypotheses, and confirmed causes.
- [ ] Repeated failures are categorized and checked for systemic patterns.

Before release:

- [ ] Critical-path requirements have passing, current evidence.
- [ ] Known risks and defects have explicit decisions and owners.
- [ ] Regression and exploratory findings are reviewed together.
- [ ] Monitoring, rollback, and post-release verification are defined.
