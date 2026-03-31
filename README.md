# DurableQ

DurableQ is a PostgreSQL-backed durable job queue service to be built in Node.js and TypeScript.

This repository is intended to be a technically rigorous backend project, not a generic app scaffold. The system is a transactional state machine under concurrency and failure. PostgreSQL is the source of truth for submission, claiming, retries, recovery, and operational inspection.

## Project Objective

Build a durable queue service that supports:

- transactional job submission
- safe multi-worker claiming
- lease-based ownership
- crash recovery
- retry handling with exponential backoff and jitter
- dead-lettering
- idempotent submission
- operational observability

The project should also force direct understanding of:

- transaction boundaries
- row-level locking
- `FOR UPDATE SKIP LOCKED`
- stale completion prevention
- retry classification
- index design
- query plan inspection
- lock inspection under contention
- metrics and structured logging

## Version 1 Goals

Version 1 must:

1. accept jobs over HTTP
2. store them durably in PostgreSQL
3. allow multiple workers to claim runnable jobs safely
4. execute handlers outside the claim transaction
5. retry retryable failures with bounded backoff and jitter
6. dead-letter jobs on terminal failure or retry exhaustion
7. recover jobs stranded by crashed workers
8. expose logs, queue stats, and metrics sufficient for debugging

## Explicit Guarantees

Version 1 provides:

- durable persistence of accepted jobs
- at-least-once execution
- idempotent submission
- lease-based crash recovery
- bounded retries
- dead-lettering
- basic observability

Version 1 does not provide:

- exactly-once execution
- fairness guarantees
- cancellation
- workflow orchestration
- distributed sharding

## Non-Goals

The following are intentionally deferred:

- Redis-first design
- Kafka, RabbitMQ, BullMQ, or other brokers
- Kubernetes
- microservices decomposition
- arbitrary workflow DAGs
- multi-region replication
- priority queues
- per-tenant fairness
- frontend-heavy dashboards

The initial bar is correctness under concurrency, not platform breadth.

## Technical Constraints

Required:

- Node.js
- TypeScript in strict mode
- PostgreSQL as the only durable source of truth
- Fastify or another minimal HTTP framework
- `pg` or a very thin SQL layer
- Docker Compose for local reproducibility
- real integration tests against PostgreSQL
- structured logging
- a machine-readable metrics endpoint

Forbidden in Version 1:

- heavy ORM abstractions
- Redis as the control plane
- workflow engines
- queue semantics hidden behind third-party infrastructure

## System Components

### API Server

Responsibilities:

- validate job submissions
- enforce idempotency rules
- create jobs transactionally
- expose job inspection and queue stats endpoints

### PostgreSQL

Responsibilities:

- store all durable job state
- coordinate safe claiming with transactions and row locks
- store retry timing and lease metadata
- support recovery and observability queries

### Worker Process

Responsibilities:

- poll for runnable jobs
- claim jobs transactionally
- execute handlers outside the claim transaction
- record success, retry, or dead-letter outcomes

### Recovery Loop

Responsibilities:

- find expired leases
- return stranded work to `queued`
- avoid mutating already terminal jobs

### Observability Surface

Responsibilities:

- emit structured logs on major state transitions
- expose queue-level stats
- expose machine-readable metrics

## Domain Model

Each job record contains:

- `id`
- `queue_name`
- `job_type`
- `payload`
- `state`
- `run_at`
- `attempts`
- `max_attempts`
- `lease_token`
- `leased_until`
- `idempotency_key`
- `last_error`
- `created_at`
- `updated_at`

## State Machine

States:

- `queued`
- `running`
- `succeeded`
- `dead_lettered`

Legal transitions:

- `new -> queued`
- `queued -> running`
- `running -> succeeded`
- `running -> queued` on retryable failure with attempts remaining
- `running -> dead_lettered` on terminal failure or retry exhaustion
- `running -> queued` on lease-expiry recovery

Illegal transitions:

- `succeeded -> queued`
- `dead_lettered -> queued`
- `succeeded -> running`
- completion by a stale lease holder
- claim of a non-runnable job

## Core Invariants

The implementation is only correct if all of the following remain true:

1. At most one currently valid lease token can authorize completion for a job at a time.
2. If a submission transaction commits, the job remains representable in PostgreSQL until terminal state.
3. A queued job is runnable only when `run_at <= now`.
4. A running job with an expired lease may eventually be returned to `queued`.
5. Duplicate submissions with the same idempotency key must not create duplicate logical jobs.
6. Terminal jobs are immutable with respect to queue execution semantics.

## Database Contract

Version 1 uses one main table: `jobs`.

Required columns:

- `id UUID PRIMARY KEY`
- `queue_name TEXT NOT NULL`
- `job_type TEXT NOT NULL`
- `payload JSONB NOT NULL`
- `state TEXT NOT NULL CHECK (state IN ('queued', 'running', 'succeeded', 'dead_lettered'))`
- `run_at TIMESTAMPTZ NOT NULL`
- `attempts INTEGER NOT NULL DEFAULT 0`
- `max_attempts INTEGER NOT NULL`
- `lease_token UUID`
- `leased_until TIMESTAMPTZ`
- `idempotency_key TEXT`
- `last_error TEXT`
- `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`

Required indexes:

- runnable-job index on `(queue_name, run_at)` where `state = 'queued'`
- recovery index on `(leased_until)` where `state = 'running'`
- unique idempotency index on `(queue_name, idempotency_key)` where `idempotency_key IS NOT NULL`

## API Contract

### `POST /jobs`

Request fields:

- `queueName`
- `jobType`
- `payload`
- `maxAttempts`
- optional `runAt`
- optional `idempotencyKey`

Required behavior:

- validate input
- open a transaction
- if an equivalent job already exists for the same idempotency key, return it
- otherwise insert a new job in `queued` state

Response must include:

- job ID
- state
- queue name
- job type
- run time

### `GET /jobs/:id`

Response should include:

- job metadata
- state
- attempts
- lease information when `running`
- timestamps
- last error

### `GET /queues/:queueName/stats`

Response should include:

- queued count
- running count
- succeeded count
- dead-lettered count
- retry backlog count
- oldest runnable age

### `GET /metrics`

Returns machine-readable metrics for operators and monitoring.

## Worker Protocol

Each worker loop must:

1. begin a transaction
2. select runnable jobs where `state = 'queued'` and `run_at <= now`
3. lock jobs safely using a protocol such as `SELECT ... FOR UPDATE SKIP LOCKED`
4. claim selected jobs by setting `state = 'running'`
5. assign a fresh `lease_token`
6. set `leased_until`
7. commit
8. execute handlers outside the claim transaction
9. record outcomes in a new transaction

Completion must only succeed when:

- `id` matches
- `state = 'running'`
- `lease_token` matches the currently stored lease token

This is the stale completion defense and it is mandatory.

## Retry Protocol

On retryable failure:

- increment `attempts`
- if attempts remain:
  - set `state = 'queued'`
  - schedule `run_at = now + backoff_with_jitter`
- otherwise:
  - set `state = 'dead_lettered'`
- clear lease fields
- store a summarized error

Version 1 retry policy:

- initial delay: 5 seconds
- cap: 15 minutes
- backoff: exponential
- jitter: required

The exact jitter formula may vary, but it must be documented in code and architecture notes.

## Recovery Protocol

A periodic recovery task must scan for jobs where:

- `state = 'running'`
- `leased_until < now`

It must return those jobs to:

- `state = 'queued'`
- `run_at = now`
- cleared lease metadata

Recovery must not mutate jobs that are already terminal.

## Idempotency Semantics

Submission idempotency scope in Version 1:

- `(queue_name, idempotency_key)`

Requirements:

- the same logical submission with the same idempotency key must create at most one logical job
- reuse of the same key with materially different input may be rejected as a conflict
- duplicate submission protection is distinct from duplicate execution protection

## Observability Contract

Structured logs for every major transition must include:

- event name
- job ID
- queue name
- job type
- attempt count
- lease token
- worker ID
- outcome
- error summary when present
- timestamp

Metrics must expose at least:

- queued jobs count
- running jobs count
- succeeded jobs count
- dead-lettered jobs count
- total jobs claimed
- total jobs succeeded
- total jobs retried
- total jobs dead-lettered
- stale completion attempts
- oldest runnable age
- claim latency
- handler duration

The system must allow an operator to answer:

- Is the queue growing?
- Are jobs stuck in `running`?
- Are retries spiking?
- Are workers making progress?

## Implementation Instructions

Build the system in this order. Do not skip correctness gates to chase extra features.

### Milestone 0: Repository Foundation

Goal:
Create the project skeleton and local development baseline.

Required work:

- initialize Node.js and TypeScript strict mode
- add Docker Compose with PostgreSQL
- establish migration workflow
- create repository layout for API, worker, DB, domain, observability, and tests
- add linting and test commands

Expected repository layout:

```text
src/
  api/
  db/
  domain/
  worker/
  recovery/
  observability/
tests/
migrations/
docker-compose.yml
README.md
ARCHITECTURE.md
```

Exit criteria:

- database boots locally through Docker Compose
- TypeScript compiles
- test runner executes
- migration system can create the schema

### Milestone 1: Durable Submission Path

Goal:
Accept jobs and persist them transactionally.

Required work:

- create the `jobs` table
- create all required indexes
- implement `POST /jobs`
- implement request validation
- implement idempotent insert behavior

Exit criteria:

- submitted jobs are durable and retrievable
- duplicate submissions with the same idempotency key do not create duplicate logical jobs
- integration test covers commit durability

### Milestone 2: Safe Claiming

Goal:
Allow multiple workers to claim runnable jobs without double ownership.

Required work:

- implement dequeue query using row locking
- implement lease assignment
- implement worker polling loop
- capture claim metrics and structured logs

Exit criteria:

- only one worker can obtain valid ownership of a job at a time
- claim path is transactionally safe
- integration test proves concurrent claim safety
- `EXPLAIN` on the dequeue query uses the runnable-job index

### Milestone 3: Completion and Stale Lease Defense

Goal:
Record successful completion safely and prevent stale workers from corrupting state.

Required work:

- implement success completion update guarded by `lease_token`
- reject stale or replaced lease holders
- expose job inspection via `GET /jobs/:id`

Exit criteria:

- valid lease holder can complete a job
- stale completion attempts do not mutate the row
- tests prove stale completion rejection

### Milestone 4: Retries and Dead-Lettering

Goal:
Handle retryable and terminal failures correctly.

Required work:

- implement failure classification
- implement bounded exponential backoff with jitter
- increment attempts on retry
- move exhausted jobs to `dead_lettered`

Exit criteria:

- retryable failures move jobs back to `queued` with a future `run_at`
- retry exhaustion moves jobs to `dead_lettered`
- tests prove retry scheduling and DLQ behavior

### Milestone 5: Lease Recovery

Goal:
Recover stranded work from crashed workers.

Required work:

- implement recovery loop for expired leases
- clear lease metadata on recovery
- emit recovery logs and metrics

Exit criteria:

- expired running jobs become runnable again
- recovery does not modify terminal jobs
- `EXPLAIN` on the recovery query uses the recovery index
- tests prove crash recovery end to end

### Milestone 6: Operational Visibility

Goal:
Make queue health inspectable during failures and load.

Required work:

- implement `GET /queues/:queueName/stats`
- implement `GET /metrics`
- emit structured transition logs
- add queue health metrics

Exit criteria:

- stats reflect actual state changes
- operators can identify stuck jobs and retry spikes
- metrics visibility test passes

### Milestone 7: Hardening and Documentation

Goal:
Prove the design and make the system defensible as a portfolio artifact.

Required work:

- write `ARCHITECTURE.md`
- document guarantees, non-goals, invariants, and failure cases
- inspect locks and activity under adversarial testing
- record query plan analysis for dequeue and recovery paths

Exit criteria:

- architecture document explains why PostgreSQL is the source of truth
- concurrency and recovery behavior are documented and tested
- project can be defended in terms of semantics, not just implementation

## Required Test Matrix

Version 1 is incomplete unless it includes tests for:

1. enqueue durability
2. worker claim flow
3. concurrent claim safety
4. successful completion by the valid lease holder
5. stale completion rejection
6. retry scheduling
7. dead-letter transition
8. crash recovery
9. idempotent submission
10. queue stats and metrics visibility

## Required Technical Knowledge Outcomes

By the end of the project, the implementation should leave you able to explain:

- where transaction boundaries belong and why
- how row locks prevent unsafe concurrent claims
- why `SKIP LOCKED` is appropriate here
- why leases are needed for crash recovery
- why stale completion protection is mandatory
- how retry backoff and jitter affect load behavior
- how submission idempotency differs from execution idempotency
- why the dequeue and recovery queries need indexes
- how to inspect `EXPLAIN`, `pg_stat_activity`, and lock behavior
- what metrics and logs are needed to debug queue health

## Definition of Done

The project is strong enough only if:

1. the contract is explicit
2. the state machine is explicit
3. transactions are intentional
4. worker crash is survivable
5. retries are bounded and observable
6. stale completion corruption is prevented
7. idempotent submission is enforced
8. logs and metrics are useful in practice
9. concurrency and failure tests exist
10. the architecture is written down clearly

## Current Repository Status

This repository is currently in specification-first mode. The next concrete implementation step is Milestone 0: create the service skeleton, migrations, Docker Compose setup, and integration test harness.
