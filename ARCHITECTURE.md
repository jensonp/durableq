# DurableQ Architecture

This document explains the intended DurableQ design at the level a reviewer would use to evaluate whether the queue is technically coherent.

It is not a speculative microservices design. It is a defense of the core single-database queue architecture and the specific failure semantics the system is meant to guarantee.

## System Summary

DurableQ is a PostgreSQL-backed durable job queue service implemented in Node.js and TypeScript.

Version 1 is intentionally small in component count:

- one API service
- one PostgreSQL database as the source of truth
- one worker process type
- one recovery loop
- one observability surface

The design goal is not breadth. The design goal is correctness under concurrency and failure.

## Guarantees

Version 1 is intended to provide:

- durable persistence of accepted jobs
- at-least-once execution
- idempotent submission
- lease-based crash recovery
- bounded retries
- dead-lettering
- structured observability

Version 1 does not provide:

- exactly-once execution
- fairness guarantees
- cancellation
- DAG workflow semantics
- broker-backed fanout
- sharding or multi-region behavior

## Core Design Decision

PostgreSQL is the only durable source of truth.

That means PostgreSQL is responsible for:

- durable job storage
- concurrency coordination for claiming
- idempotency enforcement
- retry scheduling metadata
- lease state
- recovery eligibility

This is the central architectural decision of the project. It keeps the concurrency semantics inspectable and forces the project to directly engage with transactions, row locks, indexes, and plans.

## Main Components

### API Server

Responsibilities:

- validate requests
- enforce idempotency semantics
- insert jobs transactionally
- expose job and queue inspection endpoints

Critical requirement:
Submission must not create duplicate logical jobs under client retries.

### PostgreSQL

Responsibilities:

- store job state
- arbitrate concurrent claim attempts
- enforce idempotency uniqueness
- support recovery scanning
- expose internal state through plans, locks, and activity views

Critical requirement:
The database schema and query shape must match the hot operational paths, not just represent the domain.

### Worker

Responsibilities:

- find claimable jobs
- claim them safely
- execute handlers outside the claim transaction
- record success, retry, or dead-letter outcome in a follow-up transaction

Critical requirement:
No two workers may both obtain valid ownership of the same job at the same time.

### Recovery Loop

Responsibilities:

- identify expired running jobs
- clear expired ownership
- return stranded work to `queued`

Critical requirement:
Recovery must not resurrect terminal jobs or allow stale owners to overwrite current ownership.

### Observability Surface

Responsibilities:

- logs for all important transitions
- queue stats
- metrics for claims, retries, dead-lettering, stale completions, and queue age

Critical requirement:
An operator must be able to determine whether work is progressing, stuck, or retrying pathologically.

## State Model

Each job has these execution states:

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
- completion without current lease ownership

## Core Invariants

The system is only correct if these invariants hold:

1. At most one currently valid lease token can authorize completion for a job at any time.
2. A job accepted by a committed submission transaction remains durable until terminal state.
3. A job is claimable only when `state = 'queued'` and `run_at <= now`.
4. Terminal jobs are immutable with respect to queue execution semantics.
5. Duplicate submission with the same idempotency key must not create duplicate logical jobs.
6. Expired running jobs must eventually become recoverable.

## Claim Algorithm

The worker claim path is the central concurrency protocol.

Intended shape:

1. begin transaction
2. find runnable rows where `state = 'queued'` and `run_at <= now`
3. lock them with `FOR UPDATE SKIP LOCKED`
4. update claimed rows to:
   - `state = 'running'`
   - new `lease_token`
   - new `leased_until`
5. commit
6. execute the handler outside the transaction

Why this shape matters:

- the transaction keeps selection and claim atomic
- `SKIP LOCKED` prevents workers from blocking each other on already-claimed rows
- handler execution stays outside the transaction so row locks are not held for handler duration

## Why The Lease Exists

The database row lock protects the claim transaction only while the transaction is open.

The worker still needs a durable notion of ownership after the transaction commits and before execution completes. That is why the design needs:

- `lease_token`
- `leased_until`

The lease is the application-level ownership record that survives after the row lock is released.

## Completion And Retry Safety

Completion and retry updates must be guarded by the current lease token.

Required condition for success or failure recording:

- `id` matches
- `state = 'running'`
- `lease_token` matches the current stored token

This is effectively a compare-and-set rule. It prevents stale workers from mutating rows after:

- a crash
- lease expiry
- recovery requeue
- re-claim by a different worker

## Recovery Model

Recovery is based on lease expiry, not worker self-reporting.

The recovery loop scans for:

- `state = 'running'`
- `leased_until < now`

And then transitions those rows back to:

- `state = 'queued'`
- `run_at = now`
- cleared lease metadata

This is what makes the queue crash-tolerant at the system level.

## Idempotency Model

Submission idempotency is scoped to:

- `(queue_name, idempotency_key)`

The intended mechanism is:

- unique partial index for non-null idempotency keys
- insert path that resolves duplicates through `INSERT ... ON CONFLICT`

The design problem is not only uniqueness. It is semantic interpretation of duplicate requests:

- equivalent repeated request returns the existing logical job
- conflicting repeated request may be rejected

## Schema And Indexing Rationale

The `jobs` table is intentionally the main table in Version 1.

Critical indexes:

- runnable-job index on `(queue_name, run_at)` where `state = 'queued'`
- recovery index on `(leased_until)` where `state = 'running'`
- unique idempotency index on `(queue_name, idempotency_key)` where `idempotency_key IS NOT NULL`

These indexes are not decorative:

- dequeue performance depends on the runnable partial index
- recovery performance depends on the recovery partial index
- submission correctness depends on the idempotency unique index

The queue should be defended with `EXPLAIN`, not just index definitions.

## Failure Modes The Design Must Handle

The architecture is only meaningful if it explains failure handling explicitly.

### Client Retries Submission

Required outcome:
No duplicate logical job when the same idempotency key is reused for the same logical request.

### Two Workers Race To Claim The Same Job

Required outcome:
At most one worker obtains valid ownership of the job.

### Worker Crashes After Claim

Required outcome:
The job becomes runnable again after lease expiry and recovery.

### Worker Completes After Losing Ownership

Required outcome:
The stale worker cannot overwrite current ownership or terminal state.

### Retryable Downstream Failure

Required outcome:
The job is re-queued with incremented attempts and delayed `run_at`.

### Retry Exhaustion Or Terminal Failure

Required outcome:
The job transitions to `dead_lettered`.

## Tradeoffs

### Why PostgreSQL Instead Of Redis Or A Broker In Version 1

Because the project is intended to exercise:

- transaction boundaries
- row-level locking
- MVCC reasoning
- index design
- plan inspection
- recovery semantics

Using a broker-first design would change the project into an integration exercise instead of a semantics exercise.

### Why At-Least-Once Instead Of Exactly-Once

Exactly-once claims are stronger than the project needs and are difficult to justify honestly in a queue with external side effects.

At-least-once is the honest guarantee because:

- a claimed job may be retried after crash or timeout
- downstream execution may happen more than once
- submission idempotency does not imply execution idempotency

### Why Short Transactions Instead Of Long-Lived Worker Transactions

Because long-lived transactions would:

- hold locks during handler execution
- increase contention
- increase operational fragility
- complicate recovery

The architecture deliberately separates:

- short claim transaction
- out-of-transaction execution
- short completion/retry transaction

## Evidence This Design Still Needs

To become a strong portfolio artifact, the architecture needs executable proof in the repository:

- integration tests against PostgreSQL
- concurrency tests for claim safety
- stale completion rejection tests
- crash recovery tests
- retry scheduling tests
- `EXPLAIN` output for hot queries
- lock inspection workflow

Until those exist, the design is coherent but not yet proven.
