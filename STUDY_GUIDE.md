# DurableQ Study Guide

This document combines the prerequisite concepts, canonical readings, and deeper explanatory notes needed to understand DurableQ well enough to build and defend it.

Use this file to answer three questions:

1. What do I need to understand first?
2. What should I read from primary sources?
3. What does that material mean in the specific context of DurableQ?

## Recommended Learning Order

Start with these concepts in order:

1. transactions
2. MVCC
3. transaction isolation
4. row-level locking
5. `FOR UPDATE SKIP LOCKED`
6. idempotent submission
7. state machines and invariants
8. leases
9. stale completion prevention
10. retry classification
11. exponential backoff and jitter
12. crash recovery
13. partial indexes
14. multicolumn indexes
15. `EXPLAIN`
16. lock/activity inspection
17. node-postgres transaction discipline
18. structured logging and metrics
19. integration testing against PostgreSQL
20. serializable isolation theory
21. deadlocks
22. failure semantics as a design contract

## Prerequisite Concepts

### Tier 1: Core Logic Prerequisites

You should understand these before implementing the `jobs` table, `POST /jobs`, or the claim path:

- transactions:
  what must succeed or fail atomically
- MVCC:
  how PostgreSQL snapshot visibility actually works
- transaction isolation:
  especially `READ COMMITTED` in PostgreSQL
- row-level locking:
  especially `FOR UPDATE`
- `SKIP LOCKED`:
  why its intentionally incomplete view is acceptable for queue work distribution
- idempotent submission:
  uniqueness plus application semantics
- state machines and invariants:
  legal transitions and correctness conditions

### Tier 2: Worker, Retry, and Recovery Prerequisites

You should understand these before implementing execution, retry, and recovery:

- leases:
  why they exist in addition to row locks
- stale completion prevention:
  why completion must check the current lease token
- retry classification:
  retryable versus terminal failure
- exponential backoff and jitter:
  why retry timing affects system load
- crash recovery:
  why expired running jobs must be re-queued

### Tier 3: Evidence and Defensibility

You should understand these before claiming the design is strong:

- partial indexes:
  why the dequeue and recovery paths need them
- multicolumn index design:
  why index order must match query shape
- `EXPLAIN`:
  how to verify the planner is using the intended indexes
- `pg_stat_activity` and `pg_locks`:
  how to inspect contention and blocked workers
- node-postgres transaction discipline:
  why one transaction must stay on one client
- structured logging and metrics:
  how to observe queue health
- PostgreSQL integration testing:
  why unit tests alone are not enough
- serializable isolation theory and deadlocks:
  for deeper systems understanding and defensive reasoning

## Mastery Bar

The target is not to start coding as quickly as possible. The target is to understand the system well enough that, under detailed questioning, you can justify why the algorithm is safe, what its guarantees are, and where those guarantees stop.

You are approaching that bar if you can answer these without hand-waving:

1. Why is `SKIP LOCKED` acceptable for a queue claim path even though it returns an inconsistent view?
2. Why must the handler execute outside the claim transaction?
3. Why is the lease token needed if the claim transaction already acquired row locks?
4. Why is submission idempotency not the same as execution idempotency?
5. Why do the dequeue and recovery paths need partial indexes?
6. Why must every statement in a transaction use the same `node-postgres` client?
7. What evidence would show that your dequeue query is using the correct plan?
8. Why is `READ COMMITTED` sufficient for the chosen claim path, and what would break if the protocol were split incorrectly?
9. Under what circumstances can stale completion corrupt state, and why does lease-token compare-and-set prevent it?
10. Why is at-least-once execution the honest guarantee statement here?

## Primary Sources

Read these first:

1. [PostgreSQL 18: MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html)
2. [PostgreSQL 18: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
3. [PostgreSQL 18: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
4. [PostgreSQL 18: `SELECT` locking clause](https://www.postgresql.org/docs/current/sql-select.html)
5. [PostgreSQL 18: `INSERT` and `ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html)
6. [PostgreSQL 18: Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
7. [PostgreSQL 18: Unique Indexes](https://www.postgresql.org/docs/current/indexes-unique.html)
8. [PostgreSQL 18: Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
9. [PostgreSQL 18: Using `EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html)
10. [PostgreSQL 18: `pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html)
11. [PostgreSQL 18: Monitoring stats / `pg_stat_activity`](https://www.postgresql.org/docs/current/monitoring-stats.html)
12. [node-postgres: Transactions](https://node-postgres.com/features/transactions)
13. [AWS Builders’ Library: Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
14. [AWS Architecture Blog: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

Then read for deeper theory:

15. [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/abs/cs/0701157)
16. [Serializable Snapshot Isolation in PostgreSQL](https://arxiv.org/abs/1208.4179)

## What Each Source Is For

| Topic | Source(s) | Why it matters for DurableQ |
| --- | --- | --- |
| MVCC and snapshots | MVCC intro, Transaction Isolation | explains statement visibility and why claim/update behavior depends on snapshot timing |
| Safe claiming | Explicit Locking, `SELECT` locking clause | explains row locks, lock conflicts, and why `SKIP LOCKED` is appropriate for queue consumers |
| Idempotent submission | `INSERT`, Unique Indexes, Partial Indexes | explains how duplicate submissions converge to one logical job |
| Queue and recovery indexes | Partial Indexes, Multicolumn Indexes, `EXPLAIN` | explains why the hot-path index shape must match the query shape |
| Runtime transaction correctness | node-postgres Transactions | explains how to make transaction scope real in application code |
| Recovery and debugging | `pg_locks`, `pg_stat_activity` | explains how to observe blocked workers and lock contention |
| Retry semantics | AWS Builders’ Library, AWS jitter article | explains why retries need bounds, classification, and jitter |
| Isolation theory | ANSI critique paper, SSI paper | explains the deeper theory behind anomalies and serializability |

## Technical Notes

### PostgreSQL Uses MVCC Plus Locks

PostgreSQL is not well-modeled as "everything is locks." Reads observe snapshots of committed tuple versions, while conflicting writes and row claims are coordinated through locks and write conflicts. DurableQ’s claim path must therefore be reasoned about in two layers:

1. snapshot visibility
2. lock acquisition and write conflict behavior

### `READ COMMITTED` Can Be Enough For Claiming

The claim path can be correct under `READ COMMITTED` if it is shaped narrowly:

- select only runnable jobs
- lock them in the same statement with `FOR UPDATE SKIP LOCKED`
- transition them to `running` inside the same transaction
- commit immediately

The correctness comes from the claim transaction converting visible candidate rows into exclusively claimed rows before leaving the transaction.

### `SKIP LOCKED` Is Intentionally Inconsistent

That inconsistency is acceptable here because the claim query is not a reporting query. It is a work-distribution mechanism. The goal is not a globally coherent answer set. The goal is to reserve a safe subset of claimable work without blocking on already-claimed rows.

### The Database Lock Is Not The Lease

The row lock exists only for the duration of the claim transaction. The lease exists while the worker is executing the handler outside that transaction. Without a lease, the worker would have no durable ownership record after the claim transaction commits.

### Completion Must Be Compare-And-Set

Success, retry, and terminal failure updates must check:

- `id`
- `state = 'running'`
- current `lease_token`

That is the stale-worker defense. Without it, an expired or superseded worker can overwrite newer legitimate ownership.

### Idempotency Is More Than A Unique Index

The unique index is the arbitration mechanism. The application still has to define the semantics:

- repeated equivalent request:
  return the existing logical job
- repeated conflicting request:
  reject or surface conflict explicitly

Submission idempotency and execution idempotency are different problems. DurableQ aims only to guarantee the first.

### Partial Indexes Only Work If The Planner Can Prove Predicate Match

The dequeue path and recovery path depend on partial indexes, but PostgreSQL only uses a partial index when it can prove the query predicate implies the index predicate. Hot-path SQL should therefore match the partial-index predicate directly rather than obscure it behind generic forms.

### `EXPLAIN` Is Evidence

The queue should not merely define the right-looking indexes. It should verify, with `EXPLAIN`, that:

- the dequeue query uses the runnable index
- the recovery query uses the recovery index
- estimates are at least directionally sane

### `pg_stat_activity` And `pg_locks` Are Part Of The Design Workflow

They are not optional production extras. They are how you inspect worker races, blocked sessions, long-lived transactions, and lock ownership during adversarial testing.

### Retry Design Is Also Load Design

Retries increase the chance of eventual success, but they also increase load on the failing dependency. That is why DurableQ needs:

- retry classification
- bounded attempts
- exponential backoff
- jitter

Jitter is not cosmetic randomness. It reduces synchronized retry waves and therefore reduces overload amplification.

## Suggested Reading Passes

### Pass 1: Foundational Concurrency Model

Read:

1. MVCC intro
2. Transaction Isolation
3. Explicit Locking
4. `SELECT` locking clause
5. `INSERT ... ON CONFLICT`

Objective:
Build the base model for snapshots, row locks, `SKIP LOCKED`, and atomic idempotent submission.

### Pass 2: Schema And Hot-Path Design

Read:

1. Partial Indexes
2. Unique Indexes
3. Multicolumn Indexes
4. `EXPLAIN`
5. node-postgres Transactions

Objective:
Understand how to shape the dequeue SQL, idempotency index, recovery index, and transaction code in Node.js.

### Pass 3: Operational Diagnosis And Load Behavior

Read:

1. `pg_stat_activity`
2. `pg_locks`
3. AWS retries article
4. AWS jitter article

Objective:
Understand how to inspect contention and why retry policy is a systems load-control problem.

### Pass 4: Isolation Theory And Formal Depth

Read:

1. ANSI isolation critique paper
2. SSI paper

Objective:
Build a deeper conceptual model of anomalies, isolation taxonomies, and why PostgreSQL’s serializable mode behaves the way it does.

## Written Reconstruction Before Coding

Before you start implementing, write your own answers to these:

1. Why is `SKIP LOCKED` acceptable in a queue claim path?
2. Why must the handler run outside the claim transaction?
3. Why is the lease needed in addition to the row lock?
4. Why is submission idempotency different from execution idempotency?
5. Why can a partial index fail to be used even when it seems relevant?
6. Why must one transaction stay on one node-postgres client?
7. What evidence would convince you that the dequeue query is using the correct index?

If you can answer those under pressure, connect them to concrete SQL statements, and defend them with reference to PostgreSQL behavior, you are approaching the level of understanding this project is meant to produce.
