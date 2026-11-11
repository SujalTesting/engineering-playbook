# Personal Engineering Playbook & Architecture Reference

This playbook is a practical reference for building, testing, operating, and
evolving enterprise web systems. It covers manual QA, Playwright and TypeScript
automation, split-stack networking, relational and serverless data systems,
performance telemetry, and AI-assisted engineering workflows.

## Module Index

### [Module 1: Manual QA Principles & Quality Analytics](./01-manual-qa-principles.md)

Test design methodologies, coverage matrices, requirement traceability, defect
RCA, DOM and locator knowledge, exploratory testing, and release readiness.

### [Module 2: Automation Framework Architecture](./02-automation-architecture.md)

Playwright and TypeScript patterns covering API/UI hybrids, POM, isolated
contexts, authentication caching, serverless pacing, fixtures, teardown, and
regression slicing.

### [Module 3: Web System Architecture & Networking](./03-split-stack-networking.md)

Monolith versus split-stack design, request routing, reverse proxies, SOP,
CORS, authentication, API contracts, error handling, resilience, and
cross-service observability.

### [Module 4: Database, Serverless & Concurrency Engineering](./04-serverless-database-handling.md)

Prisma and PostgreSQL modeling, migrations, foreign keys, cascading deletes,
serverless execution, connection pools, pacing, transactions, locking,
idempotency, and reliable messaging.

### [Module 5: Performance Engineering & Telemetry](./05-performance-telemetry.md)

Web Vitals, API latency, browser and backend profiling, waterfalls, caching,
optimistic UI updates, load testing, distributed tracing, alerting, and
telemetry governance.

### [Module 6: AI-Assisted Engineering & Agentic Workflows](./06-ai-assisted-engineering.md)

Repository rules, hallucination guardrails, bounded agent workflows, QA prompt
patterns, human-in-the-loop validation, information security, and AI outcome
measurement.

## How to use this playbook

Use the modules as working standards rather than static reference material:

1. Define observable acceptance criteria and identify the associated risk.
2. Select the smallest reliable test layer that proves each criterion.
3. Design isolated data, authentication, and cleanup before browser execution.
4. Document network, database, concurrency, and performance assumptions.
5. Capture evidence for failures, regressions, and production changes.
6. Use AI to accelerate bounded work, then verify every claim and artifact.

## Engineering principles

- **Evidence over assumption:** conclusions should be supported by source,
  telemetry, test output, or reproducible observations.
- **Risk-based coverage:** prioritize critical workflows, security boundaries,
  state transitions, and high-impact failure modes.
- **Explicit contracts:** define behavior between users, browsers, APIs,
  services, databases, queues, and deployment environments.
- **Isolation by default:** separate personas, tenants, test data, sessions,
  workers, and mutable resources.
- **Observable failure:** preserve useful diagnostics without exposing secrets or
  personal data.
- **Safe change:** prefer focused, reversible, validated changes over broad
  rewrites or hidden fallback behavior.

## Cross-module workflow

```text
requirements
  -> risk and coverage design
  -> API/data and environment setup
  -> focused implementation and automation
  -> functional, security, and performance validation
  -> observable release decision
  -> durable learning and follow-up
```

## Maintenance

Review the playbook when architecture, test tooling, deployment boundaries, or
security policy changes. Keep examples sanitized, update links when files move,
and record significant decisions in the relevant module or an architecture
record.
