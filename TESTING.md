# DurableQ Testing Plan

This document defines what must be proven for DurableQ to count as a serious backend/systems project.

The emphasis is not on unit-test volume. The emphasis is on verification of concurrency, failure handling, and database behavior against a real PostgreSQL instance.

## Testing Philosophy

DurableQ is a transactional state machine under concurrency and failure.

That means the most important tests are the ones that prove:

- state transitions are correct
- transaction boundaries are correct
- concurrent workers cannot both own the same job
- stale workers cannot corrupt state
- retries and recovery behave as specified

Pure unit tests are useful for helpers, but they are not sufficient evidence for the core semantics of this system.

## Required Test Categories

### 1. Submission Durability

Proof target:
If the enqueue transaction commits, the job exists durably in PostgreSQL and can be retrieved.

Minimum test:

- submit a job
- verify the row exists in `jobs`
- verify returned metadata matches stored metadata

### 2. Idempotent Submission

Proof target:
The same logical request with the same idempotency key does not create duplicate logical jobs.

Minimum tests:

- sequential duplicate submission returns one logical job
- concurrent duplicate submission returns one logical job
- conflicting reuse of the same key is handled explicitly

### 3. Worker Claim Safety

Proof target:
A runnable job can be claimed and moved to `running` with a valid lease.

Minimum test:

- insert runnable job
- run claim path
- verify `state = 'running'`
- verify `lease_token` and `leased_until` are populated

### 4. Concurrent Claim Safety

Proof target:
Two workers racing do not both obtain valid ownership of the same job.

Minimum test:

- create one runnable job
- run two workers concurrently
- verify only one successful claim result
- verify the row ends with one lease token and one owner

This is one of the most important tests in the entire repository.

### 5. Success Completion

Proof target:
The valid current lease holder can complete a job successfully.

Minimum test:

- claim a job
- complete it with the current lease token
- verify `state = 'succeeded'`
- verify lease metadata is cleared or made irrelevant per implementation

### 6. Stale Completion Rejection

Proof target:
A worker that no longer holds the current lease cannot mutate the job.

Minimum test:

- claim job with lease token `L1`
- expire or replace the lease
- attempt completion using `L1`
- verify no terminal overwrite occurs

This is the proof that stale-worker corruption is prevented.

### 7. Retry Scheduling

Proof target:
Retryable failure increments attempts and moves the next run time into the future.

Minimum test:

- claim job
- simulate retryable failure
- verify `attempts` increments
- verify `state = 'queued'`
- verify `run_at > now`
- verify lease metadata is cleared

### 8. Dead-Letter Transition

Proof target:
Retry exhaustion or terminal failure moves the job to `dead_lettered`.

Minimum test:

- claim job with low `max_attempts`
- force exhaustion or terminal failure
- verify `state = 'dead_lettered'`
- verify error metadata is recorded

### 9. Crash Recovery

Proof target:
A worker crash after claim does not strand the job forever.

Minimum test:

- claim job
- do not complete it
- advance time or use short leases
- run recovery loop
- verify job returns to `queued`
- verify it can be claimed again

### 10. Queue Stats And Metrics Visibility

Proof target:
Operational endpoints reflect actual state changes.

Minimum test:

- create jobs in multiple states
- query stats and metrics
- verify counts and gauges match database reality

## Required Verification Matrix

| Behavior | What must be proven | Evidence type |
| --- | --- | --- |
| Durable submission | committed enqueue persists | integration test |
| Idempotent submission | duplicate submission does not duplicate jobs | integration test |
| Claim safety | one worker can move queued job to running with lease | integration test |
| Concurrent claim safety | two workers cannot both own same job | adversarial integration test |
| Success completion | valid lease holder can succeed job | integration test |
| Stale completion defense | old lease holder cannot overwrite state | adversarial integration test |
| Retry scheduling | retryable failure re-queues with delayed `run_at` | integration test |
| Dead-lettering | exhausted or terminal jobs enter `dead_lettered` | integration test |
| Crash recovery | expired running jobs become claimable again | integration test |
| Queue observability | stats and metrics reflect reality | integration test |
| Hot-path performance assumptions | intended indexes are used | `EXPLAIN` evidence |
| Locking behavior under contention | worker races behave as expected | `pg_locks` and `pg_stat_activity` inspection |

## Execution Environment

The critical tests must run against a real PostgreSQL instance.

Recommended setup:

- Docker Compose for local PostgreSQL
- isolated test database per suite or test run
- migrations applied before tests
- fixtures kept minimal and explicit

Why this matters:
Mocked databases do not reproduce MVCC behavior, lock conflicts, `SKIP LOCKED`, or planner/index behavior.

## Adversarial Tests

Some tests should be deliberately adversarial rather than purely functional.

Required adversarial scenarios:

- two workers race on one runnable job
- a worker attempts completion after lease expiry
- recovery overlaps with late completion attempt
- duplicate submissions happen concurrently

These are the tests that separate a queue that "works" from a queue that is actually safe under concurrency.

## Database Evidence

The test suite should not be the only proof artifact.

The repository should eventually include:

- `EXPLAIN` for dequeue query
- `EXPLAIN` for recovery query
- example `pg_stat_activity` inspection during a worker race
- example `pg_locks` inspection during contention

These should live in either:

- this file
- `PERFORMANCE.md`
- `RUNBOOK.md`

## Acceptance Bar

DurableQ should not be presented as complete until:

1. all core semantic tests exist
2. at least one concurrent claim test passes
3. at least one stale completion rejection test passes
4. at least one recovery test passes
5. stats and metrics are validated against database state
6. hot-path query plans have been inspected

Without those, the queue may appear functional, but it has not yet earned strong backend/systems credibility.
