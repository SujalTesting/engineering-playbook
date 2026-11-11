# Module 5: Performance Engineering & Telemetry

This module covers the metrics, diagnostic tools, profiling methods, and
telemetry practices needed to evaluate and improve frontend rendering, backend
API latency, database behavior, and end-to-end user experience.

## 1. Start with a measurable target

Performance work should begin with a service-level objective and a repeatable
workload, not a subjective statement such as “the page feels slow.” Record:

- application version, browser, device, and environment;
- dataset size, user persona, feature flags, and cache state;
- traffic shape, concurrency, and test duration;
- metric definition, percentile, threshold, and measurement window;
- acceptable error rate and resource budget.

Use distributions rather than averages alone. Averages hide the slow requests
that often determine whether users experience a usable system.

## 2. Core metrics and Web Vitals

### Time to First Byte (TTFB)

TTFB measures the time from initiating a request until the first response byte
arrives. It includes connection setup, TLS, edge processing, server execution,
and often database or dependency latency. High TTFB may indicate slow backend
handlers, cache misses, network distance, or unoptimized queries.

Break TTFB into DNS, connection, TLS, request wait, and response phases where the
tooling supports it. A single TTFB number cannot identify which phase is slow.

### First Contentful Paint (FCP)

FCP is the time until the browser paints the first text, image, or canvas
content. It indicates when users receive initial visual feedback, but it does
not prove that the application is interactive or that business data is ready.

### DOMContentLoaded (DCL)

DOMContentLoaded occurs after the initial HTML is parsed and deferred scripts
have executed; it does not wait for all images and some other resources. It is
useful for document parsing analysis, but modern SPAs require additional
hydration and data-ready measurements.

### Complete rendering and record loading

For an SPA, define an explicit business-ready milestone, such as:

```text
navigation start -> route committed -> app hydrated
-> authenticated API response -> table or board rendered
-> loading indicator removed -> interactive
```

Measure the full elapsed time to load a representative task board or data table.
Record both empty and high-volume datasets, cold and warm caches, and success
and error paths.

### Other useful metrics

- **Largest Contentful Paint (LCP):** when the primary visible content renders.
- **Interaction to Next Paint (INP):** responsiveness of user interactions.
- **Cumulative Layout Shift (CLS):** unexpected visual movement.
- **API latency:** p50, p95, and p99 by route and status.
- **Throughput:** requests, jobs, or records processed per second.
- **Error rate:** failed requests and business failures, segmented by dependency.
- **Saturation:** CPU, memory, connection pools, queues, and rate-limit usage.
- **Startup and cold-start time:** especially for serverless functions.

## 3. Performance measurement workflow

1. Establish a representative baseline.
2. Verify the measurement itself and remove unrelated noise.
3. Identify the bottleneck using traces, profiles, and resource data.
4. Change one meaningful variable.
5. Repeat the same workload and compare distributions.
6. Verify correctness, error rate, resource cost, and behavior under saturation.
7. Keep the change only when the improvement is meaningful and durable.

Use a control run where possible. Warm the environment consistently, or
deliberately compare cold and warm behavior. Run enough iterations to account
for variance, and report confidence or spread rather than only the fastest run.

Do not trade correctness, security, or operability for a benchmark improvement
without an explicit decision, owner, and rollback plan.

## 4. Profiling and telemetry tools

### Browser developer tools

The Network panel shows DNS, connection, TLS, request wait, download, headers,
payload sizes, redirects, cache status, and failed requests. The Performance
panel reveals main-thread tasks, scripting, style calculation, layout, paint,
long tasks, and frame drops. The Application panel helps inspect cache,
storage, service workers, and cookies.

### HTTP analyzers

Tools such as HTTPWatch and browser network analyzers can inspect raw
HTTP/HTTPS traffic, headers, compression, redirects, SSL handshakes, payload
sizes, and blocked requests. Use them to validate what crossed the network, not
only what the application intended to send.

### Playwright Trace Viewer

Playwright traces capture test actions, DOM snapshots, screenshots, network
requests, console output, and timing. Use traces to distinguish:

- slow navigation from slow API data;
- selector waiting from application rendering;
- server latency from browser processing;
- a genuine product failure from a test synchronization problem.

Keep trace collection policy-based: retain rich traces for failures and
targeted diagnostics, while avoiding unnecessary sensitive data in artifacts.

### Backend profiling

Use structured request timings, distributed traces, database query plans,
application profilers, and runtime metrics to identify CPU, memory, lock, and
I/O bottlenecks. A frontend waterfall cannot explain a slow query unless the
backend emits correlated evidence.

## 5. Network waterfalls and blocking requests

A waterfall maps requests and their dependency relationships across a page
lifecycle. Look for:

- long DNS, TLS, or connection establishment;
- redirects and repeated authentication calls;
- large JavaScript or data payloads;
- requests blocked behind a single slow dependency;
- serial API calls that could run independently;
- cache misses and ineffective compression;
- long server wait time versus slow download or parsing.

### Sequential versus parallel requests

If Request B waits for Request A only because of client orchestration, the
waterfall becomes linear. Independent requests should run in parallel, or a
batched endpoint should return related data in one controlled response.

Do not parallelize calls that have a data dependency, exceed a rate limit, or
create database contention. Measure both latency and backend load after changing
request ordering.

### Synchronous handler auditing

Review endpoints that perform email delivery, file processing, multi-table
cascade writes, or slow external calls during the client request. Move
long-running work to a queue or asynchronous job where the business operation
allows it. Return a durable operation ID and expose status rather than holding
the request open indefinitely.

## 6. Frontend rendering and bundle performance

Profile the browser main thread for long tasks, excessive re-rendering, layout
thrashing, large component trees, and expensive serialization. Optimize with
measured changes such as:

- route and component code splitting;
- lazy loading for non-critical features;
- smaller dependencies and tree shaking;
- responsive image formats and dimensions;
- virtualization for large lists and boards;
- avoiding repeated parsing and unnecessary state updates;
- server or CDN caching for immutable assets.

Do not optimize a component in isolation if the dominant cost is API latency,
database work, or an oversized initial bundle. Keep source maps and build
analysis available for diagnosis without exposing them unnecessarily in
production.

## 7. Client-side caching and invalidation

Libraries such as React Query, SWR, and Redux-based data layers can retain
responses and avoid redundant round trips. A cache policy must define:

- query key and scope;
- freshness and stale time;
- persistence and eviction;
- cacheability of authenticated data;
- mutation invalidation;
- background refetch behavior;
- offline or failure behavior.

### Invalidation correctness

After a mutation, invalidate or update every affected query. For example,
changing a task status may affect a task detail view, board column, project
counter, filter result, and activity feed. A fast but stale interface is a
correctness defect, not a performance success.

Test cache hit, miss, stale, error, logout, persona switch, and multi-tab
behavior. Never allow one tenant, user, or role to reuse another's cached data.

### Optimistic UI updates

An optimistic update can move a task card immediately while the server processes
the mutation. It should include:

1. a local provisional change;
2. a request with an idempotency or conflict strategy;
3. success reconciliation with server state;
4. rollback and visible error handling on failure;
5. conflict handling if another user changed the record.

Optimistic rendering reduces perceived latency but must not claim success before
the server confirms the operation.

## 8. Backend and database performance

Inspect query plans, indexes, joins, N+1 access, unbounded result sets, locks,
connection pools, serialization, and transaction duration. Paginate large
responses and select only required fields. Measure database time separately
from application and network time.

For serverless services, include cold-start time, function memory, concurrency,
connection acquisition, and pool wait in the analysis. A larger pool or more
workers may reduce queueing briefly while exhausting the database sooner.

## 9. Load, stress, and capacity testing

Use realistic records and traffic shapes, including steady load, spikes,
ramp-up, long-running connections, retries, and dependency degradation. Define
pass/fail thresholds before the run:

- p95 and p99 latency by critical route;
- error and timeout rate;
- throughput at the target concurrency;
- database and queue saturation;
- memory and CPU ceilings;
- recovery time after the spike.

Watch dependencies and infrastructure during the run. A passing API metric can
hide a saturated database, queue, identity provider, or third-party service.
Never load test production without explicit authorization, safeguards, and
traffic isolation.

## 10. Observability design

Use the three pillars together:

- **Logs:** structured events with timestamp, severity, service, operation,
  environment, status, duration, and correlation ID.
- **Metrics:** request rate, error rate, latency distributions, saturation,
  queue depth, cache hit rate, and dependency health.
- **Traces:** spans across browser navigation, edge, API, database, queues, and
  external calls.

Correlation IDs must survive redirects, asynchronous jobs, and message
boundaries. Trace sampling should retain slow, failed, and high-value
transactions even when normal traffic is sampled down.

## 11. Alerting, privacy, and cost

Alert on sustained user impact or resource exhaustion rather than every
transient retry. Every alert needs an owner, severity, runbook, useful context,
and a recovery or escalation path. Review noisy alerts and remove alerts that
do not lead to action.

Redact tokens, passwords, personal data, and request bodies at the source. Set
retention, access controls, and sampling rules for telemetry. Monitor
observability volume and cloud cost so increased instrumentation remains
sustainable.

## 12. Performance review checklist

- [ ] Targets use defined workloads, percentiles, environments, and thresholds.
- [ ] TTFB, FCP, DCL, LCP, INP, CLS, and business-ready milestones are
      distinguished correctly.
- [ ] Network waterfalls identify serial calls, redirects, payloads, and cache
      behavior.
- [ ] Browser traces correlate with API, database, and dependency timings.
- [ ] Frontend rendering and backend/database bottlenecks are measured separately.
- [ ] Cache freshness, invalidation, tenant isolation, and rollback are tested.
- [ ] Optimistic updates reconcile success, failure, and conflicts.
- [ ] Load tests define pass/fail criteria before execution.
- [ ] Connection pools, queues, CPU, memory, and rate limits are observed.
- [ ] Logs, metrics, and traces share correlation IDs.
- [ ] Alerts have owners and actionable runbooks.
- [ ] Telemetry is redacted, access-controlled, retained, and cost-monitored.
