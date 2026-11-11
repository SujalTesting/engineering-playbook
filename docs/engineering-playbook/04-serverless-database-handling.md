# Module 4: Database, Serverless & Concurrency Engineering

This module covers relational data modeling, Prisma and PostgreSQL access,
serverless execution limits, dataset pacing, and concurrency controls needed
to keep systems stable under automated testing and production traffic.

## 1. Relational database and ORM architecture

### Schema definitions and migrations

An ORM such as Prisma provides typed models and a controlled path from domain
objects to relational tables. A schema should make entities, columns, indexes,
relations, nullability, defaults, and uniqueness rules explicit.

For example, a `Task` may require a valid `ProjectId`, while a `Project` may
require a valid `ProgramId` and `DepartmentId`. Fixture setup must follow those
dependencies or use a transaction that creates the complete valid graph.

Migrations should be:

- reviewed and committed with the application change;
- deterministic and safe to run once;
- tested against a representative data volume;
- observable in deployment logs;
- compatible with the rollback or roll-forward strategy;
- separated from destructive cleanup unless explicitly approved.

Prefer expand-and-contract migrations:

1. Add the new column, table, index, or relation.
2. Deploy code that can read the old and new representations.
3. Backfill in controlled batches.
4. Switch writes and reads.
5. Verify production data and query performance.
6. Remove obsolete structures in a later, deliberate migration.

Do not treat changing an ORM model as equivalent to safely migrating an
existing database. The database is the source of truth for constraints and
the migration history is part of the deployable system.

### Referential integrity

Foreign keys, unique constraints, check constraints, and not-null rules prevent
invalid states at the database boundary. Application validation improves user
feedback, but it must not be the only protection because concurrent writers,
scripts, and other services may bypass the original code path.

Use indexes for foreign keys and common query predicates where the workload
requires them. Review query plans after adding relations or changing filters.

## 2. Referential cascading and deletion rules

Deletion policy is a domain decision, not just an ORM option.

### Cascade delete

`onDelete: Cascade` removes dependent records automatically when a parent is
deleted. It can be appropriate for truly owned, disposable data, such as
temporary test artifacts or comments that have no independent retention need.

Use cascade carefully for auditable, financial, shared, or legally retained
records. A single parent deletion can remove a large graph and create an
irreversible operational event.

### Restrict and protect rules

Restricting deletion when active children exist forces callers to make the
dependency decision explicitly. This is appropriate when child records must be
archived, transferred, or reviewed before the parent disappears.

The cleanup order must follow reverse dependencies. For a test hierarchy, an
example is:

```text
Tasks -> Comments/Boards -> Projects -> Teams -> Programs -> Departments
```

The exact order depends on foreign keys and should be derived from the schema,
not guessed in teardown code. Cleanup should be idempotent and should report
the IDs of records that could not be removed.

### Soft deletion and archival

Soft deletion can preserve history by setting an archive marker or timestamp,
but every read path must consistently exclude archived records unless it is an
administrative or reporting query. Define retention, uniqueness, restoration,
and indexing behavior before adopting it. A soft delete is not automatically
equivalent to privacy erasure.

## 3. Prisma access patterns

Create and manage the Prisma client according to the runtime model. In a
long-lived process, one shared client per process usually avoids unnecessary
connections. In serverless code, use the platform's supported reuse strategy,
configure pool limits, and avoid constructing a new client for every operation.

Keep data-access code behind a service or repository boundary where it improves
testability and ownership. Use:

- parameterized Prisma queries or safe query builders;
- explicit `select` fields to avoid accidental sensitive-data exposure;
- pagination for unbounded collections;
- transactions only around the smallest atomic unit;
- consistent error translation at the API boundary;
- query logging that redacts values and secrets.

Do not solve every performance issue with a larger connection pool. A pool is a
concurrency budget, and increasing it can move the failure to PostgreSQL.

## 4. Serverless execution model

Serverless functions are short-lived, independently scaled execution units.
They may be cold-started, reused, terminated, retried, or run concurrently.
Treat each invocation as stateless:

- do not depend on process memory for durable state;
- do not assume one invocation follows another on the same instance;
- persist work progress and idempotency records durably;
- keep handlers thin and delegate domain logic;
- validate input before opening expensive resources;
- return bounded, structured outcomes.

### Cold starts

Cold starts can add latency while the platform provisions a runtime, loads
dependencies, and establishes connections. Automation may interpret this as a
failure if it uses an unrealistically small timeout.

Use measured, bounded thresholds and distinguish cold-start latency from
dependency unavailability. Reduce avoidable startup work, keep bundles focused,
and initialize reusable clients safely. Do not mask a service outage with an
unbounded timeout.

### Retries and partial completion

Events and requests may be delivered more than once, and a timeout does not
prove that the server made no change. Design handlers for duplicate delivery
and partial completion. Store durable progress or operation state where a
workflow can resume safely.

## 5. Connection pools and concurrency bottlenecks

### The failure pattern

If automated setup sends 50 rapid API requests, each request may be handled by
a separate function instance. Each instance can open database connections, and
the aggregate demand can exceed PostgreSQL or a proxy pool limit.

Symptoms include:

- connection acquisition timeouts;
- transaction allocation failures;
- blocked requests and rising latency;
- PostgreSQL “too many clients” errors;
- Prisma transaction or pool errors such as `P2028`;
- HTTP 500 responses during otherwise valid setup.

The correct response is not always to add retries. Measure active connections,
pool wait time, function concurrency, transaction duration, and query latency.
Then tune the application, platform concurrency, pool size, and database
capacity as one system.

### Connection management controls

- Reuse supported clients rather than creating one per query.
- Set pool limits appropriate to the database's total connection budget.
- Keep transactions short and avoid network calls while holding locks.
- Use a connection proxy or serverless-aware pooling strategy where available.
- Limit worker concurrency in constrained environments.
- Apply backpressure rather than allowing unlimited bursts.
- Monitor pool exhaustion before it becomes an outage.

## 6. Dataset pacing and throttling

Test data generation can unintentionally behave like a denial-of-service attack
against its own backend. Introduce a configuration value such as
`DATASET_TASK_GAP_MS` between dependent or batch creations when the environment
needs time to release connections and commit transactions.

Pacing should be deliberate and observable:

- centralize it in the dataset fixture or API helper;
- log the configured policy without exposing credentials;
- skip unnecessary delays when a safe bulk endpoint exists;
- keep the gap bounded and environment-specific;
- measure whether it reduces pool waits and transient failures.

Use Playwright worker restrictions such as `workers: 1` in staging or
pre-production when serial transactions are required for stability. Increase
parallelism only after measuring connection usage, rate limits, test duration,
and failure rates. Pacing must not hide an unsafe production endpoint.

## 7. Transactions and isolation levels

Use a transaction when multiple database operations must commit or roll back
as one unit. Keep the transaction narrow and avoid calling external APIs,
waiting for user input, or performing slow computation inside it.

Isolation levels control what concurrent transactions can observe. The right
level depends on the invariant:

- **Read committed:** common default for independent operations.
- **Repeatable read:** useful when a transaction must reread stable rows.
- **Serializable:** strongest protection for conflicting invariants, with
  possible retries and reduced throughput.

Higher isolation is not automatically safer for every workload. Test deadlocks,
serialization failures, lock waits, and retry behavior under realistic load.

## 8. Race conditions and locking strategies

A race occurs when two operations read or update shared state without a
guarantee that preserves the business invariant. For example, two personas may
move a task or change its status simultaneously, causing lost updates or an
invalid transition.

### Optimistic concurrency

Use a version, revision number, or `updatedAt` condition:

```text
UPDATE task
SET status = 'DONE', version = version + 1
WHERE id = ? AND version = ?
```

If no row is updated, the caller knows the record changed and can surface a
conflict or reload. This works well when collisions are uncommon.

### Pessimistic concurrency

Lock a narrowly scoped row or set of rows while the critical transaction runs.
This can protect high-contention invariants, but increases lock waits and
deadlock risk. Establish a consistent lock order and keep the critical section
short.

### Atomic operations and state machines

Prefer a single atomic database operation for counters, reservations, and
guarded state transitions. Enforce valid transitions in the service and, where
possible, with database constraints. A read followed by a write is not atomic
unless the transaction or statement guarantees it.

## 9. Idempotency and reliable messaging

Retries and duplicate events are normal in distributed systems. For an
externally visible write:

1. Require or derive an idempotency key.
2. Store the key with a unique constraint and operation result.
3. Return the original result for a duplicate request.
4. Expire records only after the retry window is safe.

When database state and event publication must agree, use an outbox pattern:
write the business change and an event record in one transaction, then publish
the outbox record asynchronously. Consumers must tolerate duplicates,
out-of-order delivery, and redelivery from a dead-letter queue.

## 10. Test data lifecycle and cleanup

A test fixture should create only the data it needs, register every successful
resource immediately, and clean it up in reverse dependency order. Use unique
names or tenant scopes so parallel runs cannot claim each other's records.

If setup fails halfway through, cleanup must still run for resources already
created. If cleanup fails, publish IDs and the environment for safe follow-up.
Never use an unscoped production deletion command as a test cleanup shortcut.

Test the cleanup path itself, including:

- parent deletion with active children;
- already-deleted records;
- permission failures;
- transient database errors;
- partial teardown and retry;
- expired test data from abandoned runs.

## 11. Operational controls and telemetry

Monitor:

- active, idle, and waiting database connections;
- pool acquisition time and transaction duration;
- query latency and slow-query plans;
- lock waits, deadlocks, and serialization retries;
- function duration, cold starts, memory, and invocation failures;
- API rate limits and retry counts;
- queue age, throughput, and dead-letter volume;
- fixture setup and teardown duration.

Alert on sustained user impact or capacity exhaustion, not every transient
retry. Include correlation IDs, operation keys, function version, and sanitized
resource IDs in diagnostics.

## 12. Recovery and migration readiness

Document backup, restore, rollback, roll-forward, and replay procedures. Test
restores against a non-production environment and verify that application
versions can read restored data. For destructive migrations, define a recovery
window and an owner-approved rollback or compensation plan.

## 13. Database and concurrency review checklist

- [ ] Models, foreign keys, uniqueness, indexes, and migrations are reviewed.
- [ ] Fixture creation follows the dependency graph.
- [ ] Cascade, restrict, archive, and retention rules are intentional.
- [ ] Prisma clients and connection pools match the runtime model.
- [ ] Serverless handlers do not depend on in-memory durable state.
- [ ] Cold starts, retries, and partial completion have bounded behavior.
- [ ] Dataset pacing and worker limits are configured per environment.
- [ ] Transactions are short and use an appropriate isolation level.
- [ ] Shared writes use optimistic, pessimistic, or atomic concurrency control.
- [ ] Non-idempotent operations have durable idempotency protection.
- [ ] Database changes and event publication use an outbox or equivalent.
- [ ] Teardown is reverse-ordered, scoped, observable, and safe to retry.
- [ ] Pool, lock, query, function, queue, and cleanup metrics are monitored.
