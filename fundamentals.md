# DurableQ Fundamentals

This document is the prerequisite concept list for starting DurableQ.

It is not a list of tools to memorize. It is a list of concepts you need to understand well enough to design and justify the queue correctly. The order is intentional: top items are required before meaningful implementation, lower items can deepen understanding as the system matures.

## Tier 1: Must Understand Before Writing Core Logic

These are the concepts you should understand before implementing the `jobs` table, `POST /jobs`, or the worker claim path.

### 1. Transactions

You need to understand:

- what it means for multiple SQL statements to succeed or fail as one unit
- when a queue operation must be atomic
- why enqueue, claim, completion, retry, and recovery all depend on correct transaction boundaries
- why splitting a logically atomic operation across multiple independent statements creates race conditions

Why DurableQ needs it:
The queue is a state machine. Every state transition that matters for correctness must be committed atomically or rolled back atomically.

### 2. MVCC

You need to understand:

- that PostgreSQL reads from snapshots rather than from one globally current mutable row image
- that readers and writers interact differently under MVCC than in simple lock-only mental models
- that statement timing matters because visibility is snapshot-based

Why DurableQ needs it:
Worker races, claim visibility, and update behavior cannot be understood correctly without the PostgreSQL MVCC model.

### 3. Transaction Isolation

You need to understand:

- what `READ COMMITTED` means in PostgreSQL
- why each statement can see a different committed snapshot under `READ COMMITTED`
- what stronger isolation levels change
- why "using transactions" does not automatically imply serial behavior

Why DurableQ needs it:
The claim path often works under `READ COMMITTED`, but only if the query and lock protocol are designed correctly.

### 4. Row-Level Locking

You need to understand:

- what `FOR UPDATE` does
- how row locks conflict
- how lock lifetime is tied to the transaction
- how lock contention differs from snapshot visibility

Why DurableQ needs it:
Safe multi-worker claiming is impossible to reason about without understanding how one worker prevents another from acquiring the same runnable job.

### 5. `FOR UPDATE SKIP LOCKED`

You need to understand:

- that `SKIP LOCKED` gives an intentionally incomplete view
- why that is acceptable for work distribution but not for general reporting
- how it allows workers to make progress without waiting on already-claimed rows

Why DurableQ needs it:
This is the central primitive for concurrent dequeueing in PostgreSQL-backed worker systems.

### 6. Idempotent Submission

You need to understand:

- what an idempotency key is
- how duplicate submission differs from duplicate execution
- how uniqueness constraints and `INSERT ... ON CONFLICT` work together
- what should happen if the same key is reused with materially different inputs

Why DurableQ needs it:
The API must avoid creating duplicate logical jobs when clients retry requests.

### 7. State Machines and Invariants

You need to understand:

- that the queue is not "just rows in a table"
- that job behavior is defined by legal and illegal transitions
- that invariants are the real correctness contract

Why DurableQ needs it:
Without explicit state transitions and invariants, the implementation becomes a set of ad hoc updates that cannot be defended under concurrency and failure.

## Tier 2: Must Understand Before Writing Worker and Recovery Logic

These are required before implementing completion, retries, lease expiry recovery, and stale-worker protection.

### 8. Leases

You need to understand:

- why the database row lock during claim is not sufficient for long-running handler execution
- why ownership must persist after the claim transaction commits
- why lease expiry is necessary for crash recovery

Why DurableQ needs it:
The worker cannot hold a transaction open during handler execution. Lease fields carry ownership semantics across transactions.

### 9. Stale Completion Prevention

You need to understand:

- why a worker that once owned a job may later become stale
- how a recovered or re-claimed job invalidates old ownership
- why completion must check the current lease token

Why DurableQ needs it:
Without lease-token comparison on completion, an old worker can overwrite the result of a newer legitimate owner.

### 10. Retry Classification

You need to understand:

- the distinction between retryable and terminal failure
- that not every failure should be retried
- that retry semantics are part of system behavior, not just application convenience

Why DurableQ needs it:
The queue must encode whether failures lead to rescheduling or dead-lettering.

### 11. Exponential Backoff and Jitter

You need to understand:

- why immediate retry can amplify overload
- why deterministic backoff can synchronize retries into waves
- why jitter spreads retries over time

Why DurableQ needs it:
Retry scheduling affects queue pressure, downstream load, and operational stability.

### 12. Crash Recovery

You need to understand:

- what happens when a worker dies after claiming a job
- why recovery is needed for at-least-once systems
- why recovery must only target expired running jobs

Why DurableQ needs it:
Without recovery, a single worker crash can strand work indefinitely.

## Tier 3: Must Understand Before Optimizing or Defending the Design

These concepts become critical once the core flow exists and you need to validate performance and explain why the design is correct.

### 13. Partial Indexes

You need to understand:

- what a partial index is
- why it is appropriate when only subsets of rows are hot
- that the planner only uses a partial index when it can prove the query predicate matches the index predicate

Why DurableQ needs it:
The dequeue path, recovery path, and idempotency enforcement all depend on selective indexing.

### 14. Multicolumn Index Design

You need to understand:

- the leftmost-column rule for B-tree indexes
- why index column order should follow the actual query shape
- why a "technically related" index can still be the wrong index

Why DurableQ needs it:
The queue’s runnable-job query and recovery query must be backed by indexes designed for those specific filters and orderings.

### 15. `EXPLAIN`

You need to understand:

- how to inspect a query plan
- how to tell whether PostgreSQL is using the intended index
- the difference between estimated and actual row counts

Why DurableQ needs it:
You need evidence that the dequeue and recovery paths use the schema the way you intended.

### 16. Lock and Activity Inspection

You need to understand:

- what `pg_stat_activity` shows
- what `pg_locks` shows
- how to inspect blocked sessions and lock contention

Why DurableQ needs it:
Concurrency bugs often become obvious only when you observe real sessions under load or adversarial tests.

## Tier 4: Runtime and Implementation Discipline

These are implementation fundamentals rather than purely database concepts.

### 17. node-postgres Transaction Discipline

You need to understand:

- that a transaction exists on a specific client connection
- that `BEGIN`, all intermediate statements, and `COMMIT` must use the same client
- why pooled convenience calls can silently break transaction scope

Why DurableQ needs it:
If transaction boundaries are fake in application code, the correctness arguments collapse even if the SQL looks right.

### 18. Structured Logging

You need to understand:

- what fields belong in a state-transition log
- why free-form text is not enough for debugging distributed worker behavior
- how logs differ from metrics

Why DurableQ needs it:
You need to reconstruct claim, retry, completion, and recovery behavior across workers.

### 19. Metrics

You need to understand:

- the difference between counters, gauges, and latency distributions
- which queue behaviors need direct visibility
- why metrics must correspond to operational questions

Why DurableQ needs it:
You need to know whether the queue is growing, workers are stuck, retries are spiking, or progress has stalled.

### 20. Integration Testing Against PostgreSQL

You need to understand:

- why unit tests alone are insufficient for concurrency-sensitive database behavior
- why claim safety, recovery, and idempotency must be tested against a real PostgreSQL instance
- why adversarial testing matters

Why DurableQ needs it:
Most of the difficult behavior in this project exists at the boundary between application code and the database engine.

## Tier 5: Theoretical Depth and Defensive Understanding

These concepts matter when you need to defend design choices under sustained technical questioning, reason about anomalies precisely, and avoid shallow or slogan-based explanations.

### 21. Serializable Isolation Theory

You need to understand:

- that serializability is stronger than common production isolation defaults
- that snapshot-based systems admit subtleties not captured by simplistic isolation descriptions

Why DurableQ needs it:
It improves your ability to reason precisely about why the chosen isolation level is sufficient for some paths and not others.

### 22. Deadlocks

You need to understand:

- that deadlocks can still happen in short transactional systems
- how lock ordering and transaction length affect deadlock risk

Why DurableQ needs it:
As the queue evolves, especially if additional tables or richer transitions are introduced, deadlock awareness becomes necessary.

### 23. Failure Semantics as a Design Contract

You need to understand:

- the difference between durable acceptance, successful execution, retryable failure, terminal failure, and recovery eligibility
- that guarantees must be stated explicitly rather than inferred from implementation

Why DurableQ needs it:
The project’s quality depends as much on precise semantics as on code correctness.

## Mastery Bar

You should not aim merely to start the project. You should aim to understand it well enough that, under detailed questioning, you can justify why the chosen algorithm is safe, what its guarantees are, and where its guarantees stop.

You are approaching that bar if you can answer these without hand-waving:

1. Why is `SKIP LOCKED` acceptable for a queue claim path even though it returns an inconsistent view?
2. Why must the handler execute outside the claim transaction?
3. Why is the lease token needed if the claim transaction already acquired row locks?
4. Why is submission idempotency not the same as execution idempotency?
5. Why do the dequeue and recovery paths need partial indexes?
6. Why must every statement in a transaction use the same `node-postgres` client?
7. What evidence would show that your dequeue query is using the correct plan?

That is still not the full bar. A stronger standard is that you can also answer:

8. Why is `READ COMMITTED` sufficient for the claim path you chose, and what precisely would break if the claim protocol were split incorrectly?
9. Under what circumstances can stale completion corrupt state, and why does lease-token compare-and-set prevent it?
10. Why is the database row lock during claim not the same thing as the application-level lease during execution?
11. What anomaly are you preventing with `INSERT ... ON CONFLICT`, and why is application-level read-then-insert insufficient?
12. Why can a partial index fail to be used even when it appears logically relevant?
13. What is the relationship between MVCC snapshot visibility and row-lock acquisition in the claim path?
14. Why is at-least-once execution the correct guarantee statement here, and what would have to change to make a stronger claim?
15. How do retries interact with overload, and why is jitter a correctness-adjacent operational requirement rather than a tuning detail?
16. How would you inspect PostgreSQL during a worker race to prove which session holds locks and which session is waiting or skipping?
17. What evidence would distinguish a correct queue under concurrency from one that merely appears to work in single-worker tests?

## Defensive Understanding Checklist

If someone pushes on the design, you should be able to defend all of the following clearly:

1. The state machine:
   Why each transition is legal or illegal.
2. The invariants:
   Why at most one current lease holder can complete a job successfully.
3. The transaction boundaries:
   Why the enqueue path, claim path, completion path, retry path, and recovery path each begin and end where they do.
4. The concurrency control:
   Why two workers cannot both obtain valid ownership of the same job.
5. The recovery model:
   Why crashed workers do not strand jobs forever.
6. The idempotency model:
   Why duplicate submission does not create duplicate logical jobs.
7. The retry model:
   Why retryable and terminal failures are distinct semantic classes.
8. The observability model:
   Why the logs and metrics are sufficient to diagnose stuck jobs, retry storms, and worker stalls.
9. The indexing model:
   Why the chosen indexes support the hot paths and how you verified that with `EXPLAIN`.
10. The guarantee statement:
    Why the system is at-least-once rather than exactly-once.

## Recommended Order

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

## Relationship To `resources.md`

Use [resources.md](/Users/jensonphan/back_end/resources.md) for the source documents and deeper explanations.

Use this file to answer a different question:

"What do I need to understand, in the right order, to design DurableQ well enough to withstand deep technical scrutiny?"
