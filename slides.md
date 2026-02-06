---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Welcome to Slidev
info: |
  ## CST 363
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply UnoCSS classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Transactions

CST 363




---

## Everyday Transactions

<div class="p-5">

Which of these have you experienced?


- Tried to register for a class and the seat disappeared
- Two people editing the same cell of a spreadsheet in Google Sheets and one person’s change vanished
- Cart shows a price for $29. You hit ‘Pay’ and the system re-checks price → now $34.


</div>


---
layout: section
---


## Foundations
What is a transaction? $+$ demo


---

## What is a Transaction?

<div class="p-5">

- A **transaction** is a unit of work that transforms the database from one consistent state to another.
- Boundaries: `BEGIN ... COMMIT / ROLLBACK`
  - Default: each SQL statement runs in its own transaction and is committed automatically (autocommit)
- Groups multiple statements so they succeed or fail together (atomicity).
- Visibility: “reads” see a snapshot; “writes” become visible after `COMMIT`.

</div>


---

## Example: Bank Transfer

<div class="p-5">

- Transaction to transfer $50 from account A to account B
  - Debit A, Credit B <br>


<div class="ml-12 mt-4">

1. `read(A)`
2. `A := A –50`
3. `write(A)`
4. `read(B)`
5. `B := B + 50`
6. `write(B)`

</div>


What if something goes wrong between steps 3 and 6?



</div>


---

## Quick SQL Demo: BEGIN / COMMIT / ROLLBACK


<div class="p-5">

```
-- create isolated playground in PSQL
CREATE TEMP TABLE demo(x int);
```
<br>

```sql
BEGIN;  -- start a transaction
INSERT INTO demo VALUES (1),(2);
SELECT COUNT(*) FROM demo;  -- 2
ROLLBACK;   -- undo the whole transaction
SELECT COUNT(*) FROM demo;  -- 0
```

<br>

```sql
BEGIN;
INSERT INTO demo VALUES (1),(2);
COMMIT; -- make it durable
SELECT COUNT(*) FROM demo;   -- 2
```

</div>


---


## Statements vs Transactions vs Sessions

<div class="p-5">

- **Statement:** a single SQL command
  - by default (autocommit), each statement is its own transaction.
- **Transaction:** one or more statements executed atomically between `BEGIN` and `COMMIT` / `ROLLBACK`.
- **Session/connection:** a long‑lived context that can execute many transactions and hold settings (e.g., isolation level).
- PostgreSQL default: autocommit ON; explicit `BEGIN` groups statements.


</div>


---

## Savepoints (subtransactions)

<div class="p-5">


```sql
BEGIN;
INSERT INTO demo VALUES (1);
SAVEPOINT s1;  
UPDATE demo SET x = x + 1;
ROLLBACK TO s1;  -- undo the UPDATE only
COMMIT;     -- keeps the INSERT
```


</div>




---

## ACID in Practice

<div class="p-5">


- **Atomicity:** all‑or‑nothing; 
  - aborted/failed transactions leave no visible effects.
- **Consistency:** constraints & invariants must hold after commit.
- **Isolation:** concurrent execution should be equivalent to some serial (one-at-a-time) order 
  - each transaction appears to run back-to-back with no overlap.
- **Durability:** once commit is acknowledged, effects survive crashes.

</div>

---

## WAL (Write‑Ahead Logging) & Crash Recovery


<div class="p-5">

- “Log first, data later”: commit is durable when commit record is flushed to disk via WAL.
- On crash: REDO from WAL; uncommitted versions remain invisible.
- Checkpoints bound recovery time; full‑page writes avoid torn pages.

<br>

<img src="/images/wal.webp" class="w-100" />


</div>


---

## Transaction Lifecycle

<div class="p-5">

- States: Start → Active → Partially Committed $→$ Committed
- Error path: Failed $→$ Aborted
- Commit in PostgreSQL: commit record flushed to disk $→$ success reported; crash before = abort; after = REDO.


<br>

<div class="ml-12">

<img src="/images/lifecycle.webp" class="w-80" />

</div>

</div>

---
layout: section
---


## Schedules & Serializability
How we judge correctness




---


## Transaction Schedules

<div class="p-5">


A schedule shows how the operations of multiple transactions are interleaved when run concurrently.

- Serial: transactions run one after another (no interleaving)
- Concurrent: operations are interleaved to improve throughput
- A schedule represents the actual order of operations (reads, writes, commits)
- Goal: a concurrent schedule should be **as correct as** some serial order $→$ **serializability**

</div>


---

## *Example*


<div class="p-5">


Let **T1** and **T2** be two transactions:


```
T1: R(A), W(A)
T2: R(B), W(B)
```

<br>

```
Schedule:
R(A) (T1)
R(B) (T2)
W(B) (T2)
W(A) (T1)
```

This is an **interleaved schedule** — a specific ordering of the operations <br>from $T1$ and $T2$ during concurrent execution.

</div>


---

## Serializable Schedules



<div class="p-5">



A schedule is **serializable** if its effect is equivalent to some serial schedule (where transactions run one after the other).

- **Serial schedule:** T1 then T2 (or T2 then T1), no interleaving
- **Serializable schedule:** Interleaved, but behaves like one of the serial options


Serializable schedules ensure correctness, even with concurrency


</div>


---

## Conflict Serializability and Precedence Graphs


<div class="p-5">



- A precedence graph (also called a serializability graph) shows how transactions depend on one another based on conflicting operations.
- A schedule is conflict serializable if and only if its precedence graph has no cycles.
  - We can rearrange its non-conflicting operations to get an equivalent serial schedule.
  - We can prove that a schedule is equivalent to a serial one, by checking the order of conflicting operations.
- Use the graph to check: draw an edge from T₁ $→$ T₂ if T₁’s operation precedes and conflicts with T₂’s.


</div>


---


## Precedence graph to check for conflict serializability

<div class="p-5">


- Each node represents a transaction (T₁, T₂, T₃).
- Draw a directed edge from Tᵢ → Tⱼ if:
  - Tᵢ performs an operation before Tⱼ in the schedule,
  - Both access the same data item, and
  - At least one operation is a write (conflict).


</div>


---


<div class="ml-6">

<img src="/images/serializability.png" class="w-130" />

</div>

<br>

- T₁ $→$ T₂ because T₁ reads A before T₂ writes A
- T₁ $→$ T₃ because T₁ reads B before T₃ writes B
- T₃ $→$ T₂ because T₃ reads C before T₂ writes C


There are no cycles in the graph, so this schedule is conflict serializable.


---
layout: section
---


## Anomalies

Names, quick tests, fixes

---

## Examples of Concurrency problems

- Lost update
- Inconsistent Read
- Inconsistent Write
- Phantom


---

## Lost update problem

<div class="p-5">


- Two concurrent programs both read and increment the value of  record A which <br> has a starting value of 0.
- After programs run, the value of A in the database is 1, not 2 as expected.

<br>

<div class="ml-12 mr-12">


<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 4px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1</th><th>T2</th></tr></thead><tbody>
 <tr><td>1</td><td>Start<br><code>R(A) → 0</code></td><td>&nbsp;</td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start<br><code>R(A) → 0</code></td></tr>
 <tr><td>3</td><td><code>W(A:= 1)</code></td><td>&nbsp;</td></tr>
 <tr><td>4</td><td>&nbsp;</td><td><code>W(A:= 1)</code></td></tr>
</tbody></table>

</div>

<br>

- R(A) is a read operation
- W(A) is a write operation


</div>

---

## Inconsistent Read problem

<div class="p-5">


<div class="grid grid-cols-2 gap-8">
<div class="mt-5">
  

<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 1px;
		margin: 1px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1</th><th>T2</th></tr></thead><tbody>
 <tr><td>1</td><td>Start<br><code>R(A1) → 500</code></td><td>&nbsp;</td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start<br><code>R(A1) → 500</code></td></tr>
 <tr><td>3</td><td><code>W(A1 := 400)</code><br><code>R(A2) → 100</code><br><code>W(A2 := 200)</code></td><td>&nbsp;</td></tr>
 <tr><td>4</td><td>&nbsp;</td><td><code>R(A2) → 200</code></td></tr>
</tbody></table>


</div>
<div class="...">
  
  - Program `T1` is transferring 100 <br>from account 1 to account 2
- At start
  - Account 1 = 500
  - Account 2 = 100
  - Total = 600
- Program `T2` is read total amount in <br>accounts 1 and 2. Program `T2` gets  an incorrect total of 700.

  
</div>
</div>


</div>


---

## Inconsistent Write problem


<div class="p-5">


<div class="grid grid-cols-2 gap-4">

<div>

- Bob and Alice are “on-duty”.  <br>One can go “off-duty” provided the <br>other remains “on-duty”
- Alice reads Bob’s record and sees <br>he is on duty. She changes <br>her record to off.
- Bob does the same.
- They are now both “off-duty”


</div>

<div class="mt-5">


<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 4px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1 (alice)</th><th>T2 (bob)</th></tr></thead><tbody>
 <tr><td>1</td><td>Start R(B) → ON</td><td>&nbsp;</td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start R(A) → ON</td></tr>
 <tr><td>3</td><td>R(A) → ON W(A:=OFF)</td><td>&nbsp;</td></tr>
 <tr><td>4</td><td>&nbsp;</td><td>R(B) → ON W(B:=OFF)</td></tr>
</tbody></table>


</div>


</div>

</div>


<!-- 

An inconsistent write happens when two transactions:
Read overlapping data,


Each sees the same consistent state,


But then each writes to different parts of the data,


Together, their updates violate an invariant or consistency rule.


This usually shows up when there's a constraint that depends on the combined state of multiple values.

Suppose A and B represent two switches with a rule:
At least one of A or B must be ON at all times.
Both T1 and T2:
Read the initial state: A = ON, B = ON.


Each transaction believes it's okay to turn off one switch.


But since they act concurrently, both A and B get turned OFF, violating the constraint.

This is a textbook inconsistent write / write skew example:
Each transaction makes a change that is valid in isolation, but


Combined, they leave the system in an invalid state.

-->


---

## Phantom problem

<div class="p-5">


<div class="grid grid-cols-2 gap-4">

<div>

- Before inserting a record with key = 10, <br>a program checks that key 10 does <br> not exist.
- But during insert finds that key 10 <br>now exists. <br>Same problem can occur on delete.
- New rows might appear if a query is <br>repeated in a transaction.

</div>

<div>

<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 4px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1</th><th>T2</th></tr></thead><tbody>
 <tr><td>1</td><td>Start <br>R(10) not found</td><td>&nbsp;</td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start <br>W(10)</td></tr>
 <tr><td>3</td><td>W(10) DUP KEY</td><td></td></tr>
</tbody></table>

</div>

</div>

</div>


---


## Summary: Anomalies without ACID

<div class="p-5">

With multiple concurrent transactions:

- **Dirty write → lost update:** e.g. two people book the last flight seat at once.
- **Dirty read:** read uncommitted, possibly wrong data.
- **Unrepeatable read:** same row, different result.
- **Phantom read / incorrect summary:** *e.g.* average changes mid-query due to inserts.


</div>


---

## Solutions to Concurrency and Isolation

<div class="p-5">

- Performance gain by running concurrent transaction
  - hardware has multi CPU cores
  - I/O is slow. While one transaction is waiting for I/O to complete, another transaction can be running.
  - in memory databases:
    - there is no I/O. Run transactions serially? (e.g. Redis)
- When two transactions are only reading, or reading/writing different records, no problem.

</div>


---

## Ways to detect and prevent conflicts


<div class="p-5">


- pessimistic locking – usually just called locking.
  - assume conflicts are likely, proactive locking
- record timestamp (timestamp ordering protocol)
  - Not commonly used in modern production systems like PostgreSQL.
- weak levels of concurrency control (optimistic locking)
  - Assumes that conflicts are rare, so transactions run without locking.

</div>


---

## Lock-Based Protocols


<div class="p-5">

- A lock is a mechanism to control concurrent access to a data item
- Data items can be locked in two modes :
  1. **exclusive** (X) mode. Data item can be both read as well as written.
  2. **shared** (S) mode. Data item can only be read.
- Lock requests are made to concurrency-control manager. Done by the DBMS on behalf of the program.
  - Program must obtain S lock before reading and X lock before writing.
  - When are locks released?
    - for complete isolation, locks held until commit/rollback.
- Program can proceed only after request is granted.


</div>

---

## Lock-Based Protocols (cont.)

<div class="p-5">


Lock-compatibility matrix
<img src="/images/lock.png" class="w-80" />



- A transaction may be granted a lock on an item if the requested lock is compatible with locks already held on the item by other transactions
- Any number of transactions can hold shared locks on an item,
- But if any transaction holds an exclusive on the item no other transaction may hold any lock on the item.


</div>

<!-- 

If a transaction wants a shared (S) lock on a data item:
It can get it if others already have shared locks.
It cannot get it if someone already has an exclusive (X) lock.
If a transaction wants an exclusive (X) lock:
It cannot get it if anyone else holds a lock (S or X).

-->


---

## Lock-Based Protocols (cont.)

<div class="p-5">


- A transaction can upgrade an S lock to a X lock
  - if no other transaction has an S lock, then upgrade is granted
  - other the transaction must wait until the other S lock holders release their locks.


</div>



---

## Two-Phase Locking (2PL): Foundation for Isolation


<div class="p-5">

- Phase 1 (Growing): transaction acquires locks
- Phase 2 (Shrinking): transaction releases locks
- Ensures conflict serializability
- Used as the basis for stricter protocols like Strict 2PL and Conservative 2PL



</div>


---


## Strict Two-Phase Locking (2PL)

<div class="p-5">


- PostgreSQL uses **strict** 2PL, which requires:
  - Acquiring S-lock before reading, X-lock before writing.
  - Holding all locks until commit or abort.
- Guarantees conflict serializability and recoverability.
- But can lead to deadlocks, where two transactions wait on each other.


</div>


---

## Lost update problem with Locking

<div class="p-5">



- Two concurrent programs both  increment the value of record A  which has a starting  value of 0.
- After programs run, the value  of A in the database is 1, not 2  as expected.
- Neither T1 nor T2 can make progress.
- Deadlock
  - How to detect it? How to resolve it?
    - One of T1 or T2 must be aborted and rollbacked, releasing its locks.

<div class="mt-5 ml-6 mr-6">

<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 1px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1</th><th>T2</th><th>Lock</th></tr></thead><tbody>
 <tr><td>1</td><td>Start<br><code>R(A) → 0</code></td><td>&nbsp;</td><td><code>A, S, T1</code></td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start<br><code>R(A) → 0</code></td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
 <tr><td>3</td><td><code>W(A:= 1)</code><br>wait for X lock</td><td>&nbsp;</td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
 <tr><td>4</td><td>&nbsp;</td><td><code>W(A:= 1)</code><br>wait or X lock</td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
</tbody></table>

</div>

</div>

<!-- 

which transaction is locking what, and how.

-->

---

## Lost update problem with Locking


<div class="p-5">


<div class="grid grid-cols-2 gap-8">

<div>

- Two concurrent programs both  increment the value of record `A`  which has a starting<br>
value of 0.
- After programs run, the value 
of A in the database is 1, not 2 
as expected.
- `T2` is aborted and rolled back.
- `T1` completes
- `T2` can now retry. 
- `T2` will read the new value of `A=1`.

</div>

<div class="mt-5">


<style type="text/css">
	table.tableizer-table {
		font-size: 10px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 1px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Time</th><th>T1</th><th>T2</th><th>Lock</th></tr></thead><tbody>
 <tr><td>1</td><td>Start<br><code>R(A) → 0</code></td><td>&nbsp;</td><td><code>A, S, T1</code></td></tr>
 <tr><td>2</td><td>&nbsp;</td><td>Start<br><code>R(A) → 0</code></td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
 <tr><td>3</td><td><code>W(A:= 1)</code><br>wait for X lock</td><td>&nbsp;</td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
 <tr><td>4</td><td>&nbsp;</td><td><code>W(A:= 1)</code><br>deadlock rollback</td><td><code>A, S, T1</code><br><code>A, S, T2</code></td></tr>
 <tr><td>5</td><td>proceed</td><td>&nbsp;</td><td>&nbsp;</td></tr>
 <tr><td>6</td><td>commit</td><td>&nbsp;</td><td></td></tr>
</tbody></table>


</div>

</div>

</div>


---


## Avoiding Deadlock

<div class="p-5">

- use `SELECT . . . FOR UPDATE`
  - obtain X (exclusive) lock at time of read if the program will later do an update or delete
  - this will cause T2’s read to wait until T1 completes.
  - some deadlocks can be avoided.
- Programs should read/write rows from tables in the same order
  - program 1 updates rows from table A, then updates rows from table B
  - program 2 updates rows from table B, then updates rows from table A
  - `FOR UPDATE` will not fix this problem.
  - Both programs should update rows from table A first and table B second.


</div>


---


## Deadlock Strategies


<div class="p-5">

- Detection: PostgreSQL uses a waits-for graph to detect cycles and resolve deadlocks by aborting one transaction.
- Timeouts: Alternative strategy — abort if wait exceeds a threshold.
- Prevention protocols (conceptual):
  - Wait-Die: older waits, younger dies (timestamps)
  - Wound-Wait: older preempts (wounds) younger if conflict
  - Conservative 2PL: request all locks up front to avoid circular wait (less concurrency).

</div>

---
layout: section
---

## Isolation Levels in PostgreSQL

ANSI* vs PostgreSQL semantics<br>
<div class="text-xs">
  *American National Standards Institute
</div>


---


## ANSI SQL Isolation Levels


<div class="p-5">

<div class="ml-5 mr-5">


<style type="text/css">
	table.tableizer-table {
		font-size: 12px;
		border: 1px solid #CCC; 
		font-family: Arial, Helvetica, sans-serif;
	} 
	.tableizer-table td {
		padding: 1px;
		margin: 3px;
		border: 1px solid #CCC;
	}
	.tableizer-table th {
		background-color: #104E8B; 
		color: #FFF;
		font-weight: bold;
	}
</style>
<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Level Name</th><th>Phantom</th><th>Inconsistent Reads/Writes</th><th>Lost Update</th><th>How is it done?</th><th>Disadvantages</th></tr></thead><tbody>
 <tr><td>Serializable</td><td>Solved</td><td>Solved</td><td>Solved</td><td>- S lock all records read.<br>- Predicate locks.<br>- X lock all records written.<br>- Hold locks until commit.</td><td>- Locking overhead.<br>- Waits for locks.</td></tr>
 <tr><td>Repeatable Read</td><td>Possible</td><td>Solved</td><td>Solved</td><td>- S lock all records read.<br>- X lock all records written.<br>- Hold locks until commit.<br>- No predicate locks.</td><td>- Locking but does not guarantee serializability.</td></tr>
 <tr><td>Read Committed</td><td>Possible</td><td>Possible</td><td>Solved</td><td>- S lock all records read.<br>- X lock all records written.<br>- Hold X locks until commit.<br>- S locks released after read.</td><td>- Less locking.</td></tr>
 <tr><td>Read Uncommitted</td><td>Possible</td><td>Possible</td><td>Possible</td><td>- X lock all records written.<br>- Hold X locks until commit.<br>- No lock on read.</td><td></td></tr>
</tbody></table>


</div>


</div>


---

## Summary: Transactions & Concurrency in PostgreSQL

<div class="p-5">

<v-clicks>

- **ACID properties** ensure correctness: Atomicity, Consistency, Isolation, Durability
- Schedules represent the order of operations across transactions
    - Conflict serializable if precedence graph has no cycles
- Concurrency problems (e.g., lost updates, dirty reads) are avoided using:
  - Strict 2PL (locking: shared & exclusive)
- Deadlocks can occur under locking → PostgreSQL detects and breaks them via waits-for graph
- SELECT FOR UPDATE locks rows explicitly to prevent anomalies

</v-clicks>

</div>