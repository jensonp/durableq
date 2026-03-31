# DurableQ Resources

This document is not a generic backend reading list. It is a targeted resource map for building the DurableQ system described in [README.md](/Users/jensonphan/back_end/README.md).

The structure is deliberate:

1. precise source documents and why each one matters
2. a recommended reading order
3. a deeper technical explanation tying those sources back to DurableQ

The goal is not interview fluency. The goal is to understand the concurrency and failure semantics well enough to implement the queue correctly, defend each design choice precisely, and withstand deep technical questioning without collapsing into slogans or hand-waving.

## Read This First

If you want the first reading sequence that builds a serious conceptual base rather than a shallow implementation-only familiarity, read these in order:

1. [PostgreSQL 18: 13.1 Introduction (MVCC)](https://www.postgresql.org/docs/current/mvcc-intro.html)
2. [PostgreSQL 18: 13.2 Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
3. [PostgreSQL 18: 13.3 Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
4. [PostgreSQL 18: SELECT, locking clause (`FOR UPDATE ... SKIP LOCKED`)](https://www.postgresql.org/docs/current/sql-select.html)
5. [PostgreSQL 18: INSERT / `ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html)
6. [PostgreSQL 18: 11.8 Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
7. [PostgreSQL 18: 14.1 Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
8. [node-postgres: Transactions](https://node-postgres.com/features/transactions)

That first set is enough to begin reasoning rigorously about the submission path, claim path, and schema.

Then read:

9. [PostgreSQL 18: 11.6 Unique Indexes](https://www.postgresql.org/docs/current/indexes-unique.html)
10. [PostgreSQL 18: 11.3 Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
11. [PostgreSQL 18: `pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html)
12. [PostgreSQL 18: Monitoring stats / `pg_stat_activity`](https://www.postgresql.org/docs/current/monitoring-stats.html)
13. [AWS Builders’ Library: Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
14. [AWS Architecture Blog: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

If you want the theory behind the database behavior, read these after the official docs:

15. [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/abs/cs/0701157)
16. [Serializable Snapshot Isolation in PostgreSQL](https://arxiv.org/abs/1208.4179)

## Canonical Sources

### PostgreSQL Core Concurrency

- [PostgreSQL 18: 13.1 Introduction (MVCC)](https://www.postgresql.org/docs/current/mvcc-intro.html)
  Why it matters: establishes PostgreSQL’s concurrency model. DurableQ depends on understanding that readers see snapshots while writers create new versions and conflicting writers are coordinated separately.

- [PostgreSQL 18: 13.2 Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
  Why it matters: explains exactly what `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE` mean in PostgreSQL, including which anomalies remain possible and when retries are required.

- [PostgreSQL 18: 13.3 Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
  Why it matters: explains table-level locks, row-level locks, lock conflicts, deadlocks, and lock lifetime. DurableQ’s claim path is built on row locks, so this is mandatory.

- [PostgreSQL 18: SELECT](https://www.postgresql.org/docs/current/sql-select.html)
  Why it matters: the locking clause section defines `FOR UPDATE`, `NOWAIT`, and `SKIP LOCKED`. DurableQ uses queue-like access where `SKIP LOCKED` is explicitly appropriate.

### PostgreSQL Insert, Idempotency, and Index Design

- [PostgreSQL 18: INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
  Why it matters: `INSERT ... ON CONFLICT` is the core tool for idempotent submission. The docs also explain partial-index inference and the atomicity guarantee of `ON CONFLICT DO UPDATE`.

- [PostgreSQL 18: 11.6 Unique Indexes](https://www.postgresql.org/docs/current/indexes-unique.html)
  Why it matters: idempotency keys need a uniqueness mechanism. This explains what uniqueness means in PostgreSQL and how multicolumn uniqueness is enforced.

- [PostgreSQL 18: 11.8 Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
  Why it matters: DurableQ needs predicate-limited indexes such as "only queued rows" or "only running rows". The planner caveat here is extremely important for correctness and performance.

- [PostgreSQL 18: 11.3 Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
  Why it matters: the dequeue path needs a thoughtful index shape, not just any index. The leftmost-column rule determines whether `(queue_name, run_at)` actually serves the query shape you write.

### PostgreSQL Observability and Performance

- [PostgreSQL 18: 14.1 Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
  Why it matters: you need to verify that the planner is using the dequeue and recovery indexes you intended, and understand how row estimates and costs relate to your query shape.

- [PostgreSQL 18: `pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html)
  Why it matters: lets you inspect actual lock holders and waiters during contention testing.

- [PostgreSQL 18: Monitoring stats / `pg_stat_activity`](https://www.postgresql.org/docs/current/monitoring-stats.html)
  Why it matters: lets you see active queries, wait events, transaction start times, and stuck sessions.

### Node/TypeScript Runtime Surface

- [node-postgres: Transactions](https://node-postgres.com/features/transactions)
  Why it matters: node-postgres is intentionally low-level. If you misuse pooling around transactions, your transaction boundaries will be fake even if the SQL looks right.

### Retries, Backoff, and Jitter

- [AWS Builders’ Library: Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
  Why it matters: provides a production-grade explanation of retry safety, overload amplification, idempotency, and why retries must be bounded and jittered.

- [AWS Architecture Blog: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
  Why it matters: gives concrete intuition for why no-jitter backoff clusters retries and why jittered strategies reduce both work and contention.

### Theory and Research

- [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/abs/cs/0701157)
  Why it matters: this is one of the canonical papers for understanding why the textbook ANSI isolation taxonomy is incomplete and why snapshot-style systems need more precise reasoning.

- [Serializable Snapshot Isolation in PostgreSQL](https://arxiv.org/abs/1208.4179)
  Why it matters: explains how PostgreSQL implements true serializability on top of MVCC and why serializability is not "just stronger locking". This is not required to ship DurableQ v1, but it is extremely useful for conceptual depth.

## What Each Resource Contributes To DurableQ

| Topic | Source(s) | Why DurableQ needs it |
| --- | --- | --- |
| MVCC and snapshots | PostgreSQL MVCC intro, Transaction Isolation | explains what each SQL statement can see and why claim/update behavior depends on snapshot timing |
| Safe claiming | Explicit Locking, SELECT locking clause | explains `FOR UPDATE`, row-level lock conflicts, and why `SKIP LOCKED` is acceptable for queue consumers |
| Idempotent submission | INSERT, Unique Indexes, Partial Indexes | explains how to make duplicate requests converge to one logical job |
| Queue and recovery indexes | Partial Indexes, Multicolumn Indexes, EXPLAIN | explains why your index must match both the predicate and the query’s leftmost access path |
| Runtime transaction correctness | node-postgres Transactions | explains how to make transaction scope real in application code |
| Recovery and debugging | `pg_locks`, `pg_stat_activity` | explains how to observe blocked workers, stale sessions, and lock contention |
| Retry semantics | AWS Builders’ Library, AWS Architecture Blog | explains why retries need bounds, classification, and jitter rather than naive loops |
| Isolation theory | ANSI critique paper, SSI paper | explains the deeper theory behind anomalies and why "serializable" is a nontrivial property |

## My Technical Explanation

The explanations below are intentionally rigorous. They are not simplified into interview slogans, because the target is defensible understanding rather than conversational familiarity.

## Standard Of Understanding

Reading the sources is not enough. The actual bar is whether you can reconstruct the argument yourself.

For DurableQ, that means you should eventually be able to do all of the following from memory and from first principles:

1. explain why the claim algorithm is safe under concurrent workers
2. explain why the system provides at-least-once execution rather than exactly-once execution
3. explain why the lease exists in addition to the database lock
4. explain why completion must be guarded by current lease token
5. explain why idempotent submission and idempotent execution are different problems
6. explain why the hot-path indexes have the shapes they do
7. explain how PostgreSQL will behave under contention, not just how you hope it will behave

If you cannot reconstruct those arguments without leaning on vague phrases, you have not finished the reading in the way this project requires.

### 1. PostgreSQL Is Not "Locking Rows Instead of Versions"; It Is MVCC Plus Locks

The first conceptual mistake many people make is to think of PostgreSQL concurrency as "everything is locks." That is not the right model.

PostgreSQL is fundamentally an MVCC system. Reads observe a snapshot of committed tuple versions. Writes do not mutate a universal single copy of a row in-place in the conceptual sense; they create a new visible state and coordinate conflicts with other writers and lockers. This matters because:

- plain reads usually do not block writes
- writes still conflict with other writes to the same logical row
- row-level locks are not the whole isolation story; they are one mechanism layered on top of MVCC

For DurableQ, this means the claim path must be reasoned about in two layers:

1. snapshot visibility
2. lock acquisition and write conflicts

If you only think in terms of "worker A read row X first", you will miss the actual source of correctness, which is that only one transaction can successfully take the relevant row lock and commit the state transition to `running` for the same candidate row.

### 2. `READ COMMITTED` Is Often Enough for the Claim Path, But Only Because the Claim Path Is Narrow

PostgreSQL’s default isolation level is `READ COMMITTED`. At this level, each statement sees a snapshot as of the start of that statement, not as of the start of the entire transaction.

That sounds weak, and it is weaker than `REPEATABLE READ` or `SERIALIZABLE`, but it is often sufficient for a queue claim path if the algorithm is shaped correctly:

- select only rows that are claimable now
- lock them in the same statement with `FOR UPDATE SKIP LOCKED`
- update them to `running` within the same transaction
- commit immediately

Why this works:

- competing workers may see overlapping candidate sets at snapshot time
- but row-level lock acquisition linearizes which worker actually gets each row
- `SKIP LOCKED` causes a worker to move past rows already locked by another worker
- the state transition to `running` is committed before handler execution begins

The correctness does not come from all workers seeing the same global truth. It comes from the fact that the claim transaction turns "possibly visible candidate rows" into "exclusively claimed rows" before leaving the transaction.

This is a subtle but important distinction. Queue claiming is a resource allocation problem, not a general-purpose query problem.

### 3. `SKIP LOCKED` Is Intentionally Inconsistent, and That Is Exactly Why It Is Appropriate Here

The PostgreSQL docs explicitly warn that `SKIP LOCKED` gives an inconsistent view. That warning is correct.

In a normal business query, an inconsistent view is often unacceptable because you want the answer to correspond to some coherent state of the world. In a work queue, the claim query is not a report. It is a mechanism for distributing work among competitors.

The queue consumer does not need a globally consistent set of all runnable jobs. It needs a safe subset of currently claimable jobs such that:

- no two workers obtain valid ownership of the same job simultaneously
- work can continue without blocking on already-claimed rows

This is why `SKIP LOCKED` is a queue primitive rather than a general query primitive. It is acceptable precisely because the operation is selecting work to reserve, not trying to produce a semantically complete answer set.

### 4. The Database Lock Is Not the Lease

Another common mistake is to confuse the row lock taken during claim with the long-lived ownership of the job during execution.

They are different mechanisms with different lifetimes:

- the row lock exists only for the duration of the claim transaction
- the lease exists at the application level while the worker executes the handler outside that transaction

You cannot keep the database transaction open during handler execution without creating serious problems:

- long lock hold times
- increased deadlock probability
- blocked concurrent activity
- poor throughput
- operational fragility under slow handlers

So DurableQ must separate:

1. short transaction to claim the job
2. out-of-transaction handler execution
3. short transaction to record success, retry, or terminal failure

The application-level lease fields, such as `lease_token` and `leased_until`, are what bridge the gap between those short transactions.

### 5. Stale Completion Prevention Is a Compare-And-Set Problem

Suppose worker A claims a job and gets lease token `L1`.

Then worker A crashes or stalls.

Recovery observes that the lease expired and re-queues the job.

Worker B claims the job later and gets lease token `L2`.

If worker A wakes up late and tries to mark the job `succeeded` without checking the current lease token, it can corrupt the state by overwriting worker B’s legitimate ownership.

The correct completion update is therefore not "update by ID." It is "update by ID, current state, and current lease token."

In other words, completion must be a compare-and-set:

- compare the persisted ownership fields against the worker’s current lease
- set terminal or retry outcome only if they still match

That is the core stale-worker defense.

### 6. Idempotency Is Not Just a Unique Index

A unique index is necessary for request idempotency, but it is not the full semantics.

What the unique index gives you is arbitration: for a given idempotency scope, the database will ensure that two concurrent inserts do not both win.

What the application must still define is the meaning of a repeated request:

- if the request is materially identical, should you return the existing job?
- if the request uses the same idempotency key but different payload, should you reject it as a conflict?
- what fields determine "same logical submission"?

This is why idempotency is a semantic property enforced with database mechanisms, not a purely database property.

For DurableQ, a reasonable v1 design is:

- unique scope: `(queue_name, idempotency_key)` for non-null keys
- behavior for duplicate equivalent request: return existing job
- behavior for duplicate conflicting request: reject as conflict

Also note that submission idempotency and execution idempotency are different:

- submission idempotency stops duplicate job creation
- execution idempotency concerns what happens if a job runs more than once

Because DurableQ is explicitly at-least-once, downstream handlers must still be prepared for duplicate execution.

### 7. `INSERT ... ON CONFLICT` Is a Concurrency Primitive, Not Just a Convenience

For idempotent submission, `INSERT ... ON CONFLICT` should be thought of as an atomic arbitration primitive under concurrency.

The important property is not merely that it avoids an exception. The important property is that, under concurrency, it gives a single atomic statement that resolves the race between:

- "this key is new, insert it"
- "this key already exists, use the existing logical job"

Without this, an application-level read-then-insert sequence would be vulnerable to a classic race:

1. transaction A checks for existing key, sees none
2. transaction B checks for existing key, sees none
3. both insert

The unique index plus `ON CONFLICT` collapses that into one database-level decision point.

### 8. Partial Indexes Are Powerful, but the Predicate-Matching Rule Is a Real Constraint

Partial indexes are exactly the right tool for queue workloads because only a subset of rows are relevant to each hot path:

- dequeue path cares about rows where `state = 'queued'`
- recovery path cares about rows where `state = 'running'`
- idempotency uniqueness only applies where `idempotency_key IS NOT NULL`

However, PostgreSQL’s planner only uses a partial index if it can prove that the query predicate implies the index predicate at planning time.

This has a real practical consequence:

- a claim query written with a literal predicate such as `state = 'queued'` is planner-friendly
- a query written as `state = $1` may not be recognized as implying the partial index predicate in the generic planning case

That is not a cosmetic issue. It can be the difference between an efficient queue scan and an accidental sequential scan.

For DurableQ, this means the SQL for the hot path should be written to match the partial index predicate as directly as possible.

### 9. The Dequeue Index Should Reflect the Query Shape, Not Just the Columns You Happen to Filter On

The proposed runnable-job index is effectively:

```sql
CREATE INDEX jobs_runnable_idx
ON jobs (queue_name, run_at)
WHERE state = 'queued';
```

This is not arbitrary. It is justified by the shape of the intended query:

- equality on `queue_name`
- inequality on `run_at <= now()`
- optional ordering by `run_at`
- predicate on `state = 'queued'`

The multicolumn B-tree rule matters here:

- leading equality on `queue_name` narrows the search to a queue
- range condition on `run_at` then narrows within that queue

That is much more appropriate than, for example, putting `run_at` first for a query that always scopes by queue.

### 10. `EXPLAIN` Is Not Optional Documentation; It Is Evidence

You should treat `EXPLAIN` as a verification instrument, not as a performance curiosity.

For DurableQ, the critical questions are:

- does the dequeue query use the runnable partial index?
- does the recovery query use the recovery partial index?
- does the planner estimate row counts reasonably?
- do actual row counts diverge badly from the estimates?

If you never inspect the plan, then you are trusting the shape of the SQL and the shape of the indexes without evidence that PostgreSQL agrees with you.

That is not good enough for this project.

The minimal discipline is:

- inspect `EXPLAIN` before load testing
- inspect `EXPLAIN ANALYZE` against representative data
- compare estimated rows to actual rows
- check whether the chosen plan matches the intended hot path

### 11. `pg_stat_activity` and `pg_locks` Are Your Runtime Microscope

When workers contend, you need to answer questions like:

- which sessions are active?
- which queries are waiting?
- what are they waiting on?
- which locks are held and by whom?

`pg_stat_activity` gives session-level activity, query text, wait events, and transaction/query start times.

`pg_locks` gives lock records and can be joined with `pg_stat_activity` by PID.

This is not only for production. You should use these views during adversarial local tests where:

- two workers race to claim
- one worker holds a transaction open
- recovery overlaps with handler completion
- you intentionally create lock contention

Useful starter queries:

```sql
SELECT pid, state, wait_event_type, wait_event, xact_start, query_start, query
FROM pg_stat_activity
WHERE datname = current_database();
```

```sql
SELECT *
FROM pg_locks pl
LEFT JOIN pg_stat_activity psa
  ON pl.pid = psa.pid;
```

The point is not to memorize every column. The point is to become empirically fluent in what the database is actually doing under contention.

### 12. node-postgres Transaction Boundaries Are Bound to a Client, Not to Your Function Call

This is a nontrivial implementation detail that can silently invalidate your reasoning.

In node-postgres, a transaction exists on a specific client connection. If you:

- call `BEGIN` on one client
- then accidentally issue the next query through `pool.query(...)`
- and later `COMMIT` through yet another path

you do not have one transaction. You have unrelated statements on potentially different clients.

So for any DurableQ operation that depends on transactionality, especially:

- submission with idempotency
- claim path
- completion/retry transition
- recovery transition

you must:

- acquire a client
- run every statement of that transaction on that same client
- `COMMIT` or `ROLLBACK`
- release the client

### 13. Retry Design Is About Load Control as Much as Failure Recovery

Many beginner retry loops assume that if a request failed, sending it again quickly improves availability. That is often false.

Retries do two things at once:

- they increase the chance of eventual success under transient failure
- they increase load on the failing dependency

If the failure mode is overload, naive retries worsen the system precisely when it is already least able to absorb extra traffic.

That is why retry design must include:

- classification: which failures are retryable
- bounds: how many times to retry
- spacing: exponential backoff
- decorrelation: jitter

For DurableQ, retry behavior is part of the job semantics, not a helper utility. It changes queue pressure, worker behavior, and operational load shape.

### 14. Jitter Is Not Cosmetic Randomness

Exponential backoff without jitter still tends to synchronize retries into waves. The AWS material is useful because it shows that backoff alone reduces frequency but still creates clustered contention windows.

Jitter changes the retry distribution from "many clients retry together after similar delays" to "retries are smeared over time."

For DurableQ, this matters in two places:

- job retry scheduling
- any periodic worker polling or recovery cadence

If every retryable failure lands on the same deterministic schedule, a queue can create its own retry storms.

A reasonable v1 policy is:

- exponential backoff with a cap
- full jitter or decorrelated jitter
- bounded max attempts

### 15. Deadlocks Are Still Possible Even in a Queue System

A queue claim path is often simple enough to avoid deadlocks if you keep transactions short and lock in a consistent order. But deadlocks are not impossible just because you are "only using row locks."

They become more likely when:

- transactions touch multiple jobs or tables
- transaction duration expands
- completion and recovery logic lock different objects in inconsistent order

The defensive strategy is:

- keep transactions short
- acquire locks in consistent order
- use the most restrictive needed mode first
- be prepared to retry transactions aborted due to deadlock

This is one reason not to let the claim transaction leak into handler execution.

### 16. Why `SERIALIZABLE` Is Useful to Understand Even If You Do Not Use It

DurableQ v1 can likely be built correctly without running the whole system at `SERIALIZABLE`. That does not make serializability irrelevant.

Understanding serializability teaches you:

- what kinds of anomalies weaker levels can admit
- why snapshot isolation is strong but not omnipotent
- why "I used transactions" is not a complete argument

The PostgreSQL SSI paper is especially useful because it shows that true serializability in an MVCC system is not merely "block more often." It requires tracking read/write dependencies and may abort transactions whose combination would violate serializability.

Even if you ship DurableQ using `READ COMMITTED`, understanding why that is sufficient for some paths and insufficient for others is part of the systems knowledge this project is meant to build.

## Suggested Reading Passes

### Pass 1: Foundational Concurrency Model

Read:

1. PostgreSQL MVCC intro
2. Transaction Isolation
3. Explicit Locking
4. SELECT locking clause
5. INSERT / `ON CONFLICT`

Objective:
Build the base model for snapshots, row locks, `SKIP LOCKED`, and atomic insert-or-conflict semantics.

### Pass 2: Schema and Hot-Path Design

Read:

1. Partial Indexes
2. Unique Indexes
3. Multicolumn Indexes
4. Using EXPLAIN
5. node-postgres Transactions

Objective:
Understand how to shape the dequeue SQL, idempotency index, recovery index, and transaction code in Node.js.

### Pass 3: Operational Diagnosis and Load Behavior

Read:

1. `pg_stat_activity`
2. `pg_locks`
3. AWS Builders’ Library retries article
4. AWS jitter article

Objective:
Understand how to inspect contention and why retry policy is a systems load-control problem.

### Pass 4: Isolation Theory and Formal Depth

Read:

1. ANSI isolation critique paper
2. SSI paper

Objective:
Build a deeper conceptual model of anomalies, isolation taxonomies, and why PostgreSQL’s serializable mode behaves the way it does.

### Pass 5: Written Reconstruction

After the reading, write your own explanations of:

1. the claim protocol
2. the stale-completion defense
3. the idempotent submission path
4. the retry and dead-letter semantics
5. the recovery protocol
6. the queue and recovery indexes
7. the system guarantee statement

Objective:
Convert recognition into durable understanding. If you can only recognize the right words when reading the docs, you are not ready to defend the design.

## Recommended Next Step After Reading

After the reading passes, write down your own answers to these questions before coding:

1. Why is `SKIP LOCKED` acceptable in a queue claim path even though it gives an inconsistent view?
2. Why must the handler run outside the claim transaction?
3. Why is the lease token needed if the claim transaction already used row locks?
4. Why is submission idempotency not the same as execution idempotency?
5. Why can a partial index fail to be used even if it looks logically relevant?
6. Why must every statement in a transaction use the same node-postgres client?
7. What evidence would convince you that the dequeue query is using the correct index?

If you can answer those in writing, you understand enough to begin implementing DurableQ correctly.

If you can answer them under pressure, extend them, connect them to specific SQL statements, and defend them with reference to PostgreSQL behavior, then you are approaching the level of understanding this project is meant to produce.
