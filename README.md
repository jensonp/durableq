# DurableQ Project Specification

## Purpose

This project has two simultaneous purposes:

1. **Portfolio signal**: produce a serious backend/systems artifact that demonstrates strong engineering judgment.
2. **Knowledge closure**: deliberately close missing applied systems/database gaps in:
   - transactions
   - locking
   - concurrent worker coordination
   - recovery
   - retries
   - idempotency
   - observability
   - production-style debugging

This is not a generic app project. It is a transactional state machine under concurrency and failure, implemented as a backend service.

---

## Project Statement

Build a **PostgreSQL-backed durable job queue service** in **Node.js + TypeScript** that supports:

- transactional job submission
- safe multi-worker claiming
- lease-based crash recovery
- retryable failure handling with exponential backoff and jitter
- dead-lettering
- idempotent submission
- operational observability

The system must serve both as a strong backend portfolio project and as an explicit vehicle for learning applied systems/database concepts.

---

## Core Framing

The project is:

- a transactional state machine
- under concurrent access
- with explicit failure semantics
- backed by PostgreSQL as the source of truth

The project is **not**:

- a CRUD app
- a frontend-heavy portfolio piece
- a framework exercise
- a microservices demo
- a Redis/Kafka showcase
- a queue built just because queues sound advanced

---

## Version 1 Goals

Version 1 must:

1. accept jobs through an HTTP API
2. store them durably in PostgreSQL
3. allow multiple workers to safely claim runnable jobs
4. execute handlers
5. retry retryable failures with exponential backoff and jitter
6. move exhausted jobs to dead-letter state
7. recover jobs stranded by crashed workers
8. expose logs, metrics, and admin visibility sufficient for debugging

---

## Version 1 Non-Goals

The following are explicitly excluded from version 1:

- Redis as required infrastructure
- Kafka / RabbitMQ / BullMQ
- Kubernetes
- microservices decomposition
- arbitrary workflow DAGs
- distributed sharding
- multi-region replication
- priority queues
- per-tenant fairness
- cancellation
- visual dashboard frontend
- exactly-once processing claims

These are deferred because they inflate scope or hide the concepts this project is supposed to teach.

---

## Technology Constraints

### Required

- **Node.js**
- **TypeScript** with strict mode
- **PostgreSQL** as the only durable source of truth
- **Fastify** or similarly minimal HTTP framework
- **pg** (`node-postgres`) or a very thin SQL layer
- **Docker Compose** for local reproducibility
- **real integration tests** against PostgreSQL
- **structured logging**
- **metrics endpoint**

### Forbidden in Version 1

- heavy ORM abstraction
- Redis-first design
- message brokers
- workflow frameworks
- Kubernetes deployment complexity
- large frontend work before semantics are proven

---

## Learning Constraints

The project must force direct engagement with the following concepts:

- explicit transaction boundaries
- row-level locking
- concurrent worker race conditions
- lease-based ownership
- stale completion prevention
- retry classification
- exponential backoff and jitter
- idempotency semantics
- index design
- query plan inspection
- lock/activity inspection
- structured logs and metrics

If the project can be built while avoiding those, the project specification is too weak.

---

## System Architecture

Version 1 consists of:

1. **API Server**
   - accepts submissions
   - validates requests
   - enforces idempotency rules
   - exposes job inspection and stats

2. **PostgreSQL**
   - stores all durable job state
   - coordinates claiming
   - stores retry timing and lease metadata

3. **Worker Process**
   - polls for runnable jobs
   - claims them transactionally
   - executes handlers
   - records success / retry / dead-letter outcome

4. **Recovery Loop**
   - finds expired running jobs
   - returns them to runnable state

5. **Observability Surface**
   - structured logs
   - queue stats
   - metrics endpoint

---

## Exact System Guarantee Statement

Version 1 provides:

- **durable persistence** of accepted jobs
- **at-least-once execution**
- **idempotent submission**
- **lease-based crash recovery**
- **bounded retries**
- **dead-lettering**
- **basic observability**

Version 1 does **not** provide exactly-once execution.

---

## Core Domain Model

A job is a record:

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

---

## Job State Machine

### States

- `queued`
- `running`
- `succeeded`
- `dead_lettered`

Retryable failure is represented by transition back to `queued` with updated attempt and schedule metadata.

### Legal Transitions

- `∅ -> queued`
- `queued -> running`
- `running -> succeeded`
- `running -> queued` on retryable failure with attempts remaining
- `running -> dead_lettered` on terminal failure or retry exhaustion
- `running -> queued` on lease expiry recovery

### Illegal Transitions

- `succeeded -> queued`
- `dead_lettered -> queued`
- `succeeded -> running`
- completion by a stale lease holder
- claim of a non-runnable job

---

## Core Invariants

1. For any job, at most one **currently valid lease token** may authorize completion at a given time.
2. If a job insert commits, the job remains representable in PostgreSQL until terminal state.
3. A queued job is runnable only when `run_at <= now`.
4. A running job whose lease expires may eventually be returned to `queued`.
5. Duplicate submissions with the same idempotency key must not create duplicate logical jobs.
6. Terminal jobs are immutable with respect to queue execution semantics.

---

## Database Schema

### Main Table: `jobs`

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

### Required Indexes

1. Runnable-job index:
   - `(queue_name, run_at)` where `state = 'queued'`

2. Recovery index:
   - `(leased_until)` where `state = 'running'`

3. Idempotency index:
   - unique `(queue_name, idempotency_key)` where `idempotency_key IS NOT NULL`

---

## API Specification

### `POST /jobs`

Accepts:

- `queueName`
- `jobType`
- `payload`
- `maxAttempts`
- optional `runAt`
- optional `idempotencyKey`

Behavior:

- validate request
- within a transaction:
  - if the idempotency key already exists for the same logical submission, return the existing job
  - otherwise insert a new job in `queued` state

Returns:

- job ID
- state
- queue name
- job type
- run time

### `GET /jobs/:id`

Returns:

- job metadata
- state
- attempts
- lease information if running
- timestamps
- last error

### `GET /queues/:queueName/stats`

Returns:

- queued count
- running count
- succeeded count
- dead-lettered count
- retry backlog count
- oldest runnable age

### `GET /metrics`

Returns machine-readable metrics.

---

## Worker Protocol

Each worker repeatedly:

1. begins a transaction
2. selects runnable jobs
3. locks them
4. claims them by setting `state = running`
5. writes a fresh `lease_token`
6. writes `leased_until`
7. commits
8. executes handlers outside the claim transaction
9. records outcome in a new transaction

### Claim Rule

A job is claimable iff:

- `state = 'queued'`
- `run_at <= now`

Claiming must use a safe row-locking protocol such as:

- `SELECT ... FOR UPDATE SKIP LOCKED`

### Completion Rule

Successful completion must update a job only if:

- `id` matches
- `state = 'running'`
- `lease_token` matches the current lease

This prevents stale workers from overwriting current ownership.

### Retry Rule

On retryable failure:

- increment `attempts`
- if attempts remain:
  - set `state = 'queued'`
  - schedule `run_at = now + backoff_with_jitter`
- else:
  - set `state = 'dead_lettered'`
- clear lease fields
- store error summary

### Terminal Failure Rule

On terminal failure:

- set `state = 'dead_lettered'`
- clear lease fields
- record error
- require matching current lease token

---

## Recovery Protocol

A periodic recovery task scans for jobs where:

- `state = 'running'`
- `leased_until < now`

and returns them to:

- `state = 'queued'`
- `run_at = now`
- cleared lease fields

Recovery must not affect any job already recorded as terminal.

---

## Retry Policy

Use bounded exponential backoff with jitter.

If the failure number is `a >= 1`, define:

- `base(a) = min(cap, initial * 2^(a-1))`

Then choose a delay in a bounded interval around `base(a)`.

Version 1 constants:

- initial delay: 5 seconds
- cap: 15 minutes

The exact formula can be chosen later, but the policy must be explicit and documented.

---

## Idempotency Semantics

### Meaning

If the same logical request is submitted multiple times with the same idempotency key, the system must create at most one logical job.

### Scope

Version 1 uniqueness scope:

- `(queue_name, idempotency_key)`

### Conflict Rule

If the same idempotency key is reused with materially different input, Version 1 may reject it as a conflict.

---

## Observability Requirements

### Structured Logs

Every major transition must log:

- event name
- job ID
- queue name
- job type
- attempt count
- lease token
- worker ID
- outcome
- error summary if any
- timestamp

### Metrics

Expose at least:

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

### Admin Visibility Requirement

The system must allow you to answer:

- Is the queue growing?
- Are jobs stuck in running?
- Are retries spiking?
- Are workers making progress?

If those cannot be answered, observability is insufficient.

---

## Required Knowledge Outcomes

By the end of the project, you must be able to explain and use:

### 1. Transaction Boundaries

You must know:

- what operations must be atomic
- where `BEGIN / COMMIT / ROLLBACK` belong
- what breaks if claim/completion is split incorrectly

### 2. Concurrency Control

You must know:

- why workers race
- how row locks prevent unsafe concurrent claims
- why `FOR UPDATE SKIP LOCKED` is useful

### 3. Recovery

You must know:

- why crashed workers strand jobs
- why leases exist
- how recovery restores stranded work

### 4. Retry Semantics

You must know:

- retryable vs terminal failure
- why immediate retry is dangerous
- how backoff and jitter change load behavior

### 5. Idempotency

You must know:

- API submission idempotency
- why at-least-once execution requires downstream idempotency awareness
- why duplicate submission and duplicate execution are different problems

### 6. Observability

You must know:

- what to log
- what to measure
- how to inspect queue health
- how to identify stuck jobs, lock contention, and retry storms

### 7. Query Performance

You must know:

- why the dequeue and recovery paths need indexes
- how to inspect `EXPLAIN`
- how to reason about plan quality

These are not optional side effects. They are explicit deliverables.

---

## Required Database/System Concepts the Project Must Exercise

The implementation must directly exercise:

- transactions
- row locking
- isolation reasoning at the level necessary to justify claim safety
- runnable-job indexing
- recovery indexing
- idempotency indexing
- `EXPLAIN` on dequeue query
- `EXPLAIN` on recovery query
- lock/activity inspection during adversarial testing

If the implementation never makes you inspect plans or locks, the project is under-specified.

---

## Required Test Suite

Version 1 is incomplete unless it includes the following tests:

1. **Enqueue test**
   - submitted job is stored durably and retrievable

2. **Worker claim test**
   - a runnable job can be claimed and moved to running with lease fields

3. **Concurrent claim test**
   - two workers racing do not both obtain valid ownership of the same job

4. **Success completion test**
   - correct lease holder can complete a job

5. **Stale completion rejection test**
   - expired/replaced lease holder cannot overwrite current ownership

6. **Retry scheduling test**
   - retryable failure increments attempts and moves `run_at` forward

7. **DLQ test**
   - retry exhaustion moves a job to `dead_lettered`

8. **Crash recovery test**
   - worker crash after claim eventually makes the job runnable again

9. **Idempotency test**
   - duplicate submission with the same idempotency key does not create duplicate logical jobs

10. **Metrics visibility test**
   - queue stats reflect actual state changes

These are proof obligations, not optional extras.

---

## Repository Artifacts Required

The project is incomplete unless the repository contains:

### `ARCHITECTURE.md`

Must include:

- goals
- guarantees
- non-goals
- component responsibilities
- schema summary
- state machine
- invariants
- failure scenarios
- tradeoff discussion
- why PostgreSQL is the source of truth

### `README.md`

Must include:

- what the system does
- how to run it
- API summary
- semantics summary
- known limitations

### Testing Documentation

Must include:

- required test categories
- concurrency tests
- recovery tests
- known limits

---

## Required Implementation Boundaries

### Keep in Version 1

- one main `jobs` table
- one worker type
- one recovery loop
- one handler registry
- one minimal admin/metrics surface

### Defer Until After Core Correctness Is Proven

- priorities
- fairness
- cancellation
- cron scheduling
- DAG workflows
- Redis
- `LISTEN/NOTIFY`
- sharding
- multi-region ideas
- frontend-heavy dashboard work

---

## Definition of Strong Enough

The project is strong enough only if:

1. it has a clear contract
2. it has an explicit state machine
3. it uses transactions intentionally
4. it survives worker crash
5. it handles retries rigorously
6. it prevents stale completion corruption
7. it supports idempotent submission
8. it exposes useful metrics and logs
9. it includes concurrency and failure tests
10. it contains a written architecture explanation

If any of these are missing, the system may run, but it is not the intended project.

---

## Knowledge Plan

### Must Know Before Starting

- TypeScript strict basics
- async/await and error propagation
- basic SQL
- schema basics
- Docker Compose basics
- HTTP route basics

### Must Learn During the Project

- transactions
- row locking
- `SKIP LOCKED`
- lease semantics
- retry design
- idempotency design
- index design
- query plan reading
- structured logs
- metrics design
- concurrency testing

### Can Defer Until After Version 1

- Redis
- notification-based wake-up
- advanced NoSQL comparisons
- DBMS internals below the query/lock/plan level
- broad distributed systems coverage

---

## Immediate Deliverables

### Deliverable 1
One-page design/spec containing:

- goals
- guarantees
- non-goals
- state machine
- invariants
- failure cases

### Deliverable 2
Initial schema:

- `jobs` table
- runnable-job index
- recovery index
- idempotency index

### Deliverable 3
Minimal repository skeleton:

- API
- worker
- DB
- domain
- observability
- tests

### Deliverable 4
Week 1 implementation:

- `POST /jobs`
- `GET /jobs/:id`
- one worker loop
- one integration test

---

## Final Project Statement

Build a PostgreSQL-backed durable job queue service in Node/TypeScript that supports transactional job submission, safe multi-worker claiming, lease-based crash recovery, retryable failure handling with exponential backoff and jitter, dead-lettering, idempotent submission, and operational observability, with the explicit goal of both producing a serious backend portfolio artifact and closing missing applied systems/database knowledge gaps.
