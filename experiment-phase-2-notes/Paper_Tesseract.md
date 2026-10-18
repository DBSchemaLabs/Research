# Paper - Online Schema Evolution is (Almost) Free for Snapshot Databases (VLDB, 2023)

Hu, Wang (Simon Fraser University), Zhou (Tencent) · [PDF](https://www.vldb.org/pvldb/vol16/p140-hu.pdf) · the system is called Tesseract

## Section 1 - Introduction
* Tesseract's idea: a schema change is really two ordinary updates. Converting `status` is "update every row" plus "update the catalog row that says what type `status` is". If the database treats both as normal updates, the schema change gets versions and snapshots like any other write. A query that started before it keeps seeing the old schema and old rows, nobody waits on a special lock, and a failed conversion just rolls back. They call it "data-definition-as-modification".
* Claimed cost: even a heavy change like adding a column drops ordinary query throughput by "only up to ~10%".


## Section 2 - MVCC Background
* Tesseract is designed for databases that keep several versions of every row, known as multi-version concurrency control (MVCC). Each transaction sees a consistent snapshot, made of the versions that were committed before it started. PostgreSQL works this way too.
* Tesseract is built on ERMIA, a research engine that keeps everything in memory. Each table has an array indexed by row ID, and each entry points to that row's chain of versions.

> Question - In ERMIA, old row versions are chains in memory. In Postgres they're dead rows in the table file, cleaned up later by vacuum. Rewriting every row means a second copy of the table on disk until vacuum catches up, which is E3's 88 → 184 MB again.

> Question - What happens to indexes? In ERMIA they point to row IDs, so swapping the array keeps them valid. In Postgres a new copy of the table needs every index rebuilt to point at it.

> Question - Logs on a RAM disk aren't durable. What happens if the machine crashes halfway through a migration? And with real disks and real write-ahead logging, how much of the "~10%" survives?

## Section 3 - Tesseract Overview

### 3.1 Four kinds of schema change
* Every schema change answers two questions: does it have to copy or change the table's data, and does it have to read through the data to check it? (Table 1)

![img.png](OperationTypes_Tesseract.png)

* A change that does neither only touches metadata, like adding or dropping a column.
* The same operation can land in a different box depending on how a database is built.

* To extrapolate into what I am doing: text → int is `MODIFY COLUMN`, in the copy-and-check box. Every row has to be rewritten, and every value checked to see if it converts. That's the most expensive kind, and it's what my E1 did.

> Question - "Check" for us means every status has to map to a number. What happens to a row with a value that doesn't, like `'refunded'`? Does the whole change fail, after all the copying is done?

### 3.2 Schema versioning
* A table's schema is stored as a row, a "schema record", in a catalog table. The catalog is an ordinary table.
* Schema records have versions, just like data rows. Each version carries a commit timestamp, and the paper calls them S1, S2 and so on.
* To read a table, a transaction first reads the schema version its snapshot can see, then reads the data that matches it.
* Data written under different schema versions is kept apart, in separate arrays (Figure 2).

* To extrapolate into what I am doing:
  * S1 says `orders.status` is text. S2 says it's an integer.
  * A query that started before S2 committed keeps seeing S1 and the text values, even while the change is running.
  * A query that starts after S2 commits sees integers.

> Question - "Data under different schema versions is kept apart" sounds like two copies of the table while both versions exist. Is that the 2× storage from E3 again? The paper only explains how the data gets there in Section 5.

> Question - This is the opposite of what I saw in E2, where a reader waited 2.7 s for the `ALTER` to finish. Here it would just read the old version. But it only helps queries that were already running. A query from the old app that starts after S2 commits gets integers, and nothing in Section 3 keeps the old app working.

> Question - Postgres also versions rows, but not every `ALTER TABLE` respects old snapshots. Local's sandbox saw an old snapshot go from 1,000 rows to 0 after a table rewrite. Worth checking on my Mac whether a reader that started before a rewrite really sees the old data in Postgres 16.

### 3.3 Schema changes and normal writes in one system
* One transaction can contain both schema changes and ordinary inserts and updates.
* A schema change is built out of ordinary writes: update the schema record, then do whatever inserts or updates are needed to change or check the data. It goes through the same concurrency control as any other transaction.
* An ordinary transaction reads the schema version its snapshot can see, and has to make sure the data and schema versions it uses match.
* Two ways they can clash:
  * A schema change has written a new schema record but not committed yet. An ordinary write that touches the same thing conflicts with it, and one of them aborts.
  * An ordinary transaction started under S1, and wants to commit after S2 has committed. If it did, later readers would read its old-shape row with the new schema. So this "must be forbidden". Their example: T2 began at timestamp 200, before S2 existed, so T2 must use S1 throughout.

> Question - "Forbidden" presumably means the old transaction is aborted. So any in-flight transaction from the old app that straddles the change fails, and the app has to retry it. That's F1's write fencing, on one machine. Is there a number for how many transactions get aborted this way?

> Question - In E9, one long transaction made everyone queue behind a waiting `ALTER`. Here, a long transaction that started before the change wouldn't block anyone, but it would be aborted if it tried to write after the change committed. Which is better depends on whether the long one or the many short ones matter more.

> Question - Postgres can also run DDL inside a transaction, and roll it back. MySQL can't. So the "DDL is just a transaction" part already exists in Postgres. What's missing there is the versioned schema that lets old snapshots keep reading.

## Section 4 - Data-Definition-as-Modification
This section tries the most direct version of Section 3's idea, then shows why it doesn't work.

### 4.1 How a normal transaction works (snapshot isolation)
* Read: walk a row's chain of versions from newest to oldest, and take the first one that was committed before the transaction started.
* Write: only allowed if the transaction can see the row's latest version. The new version is linked in front of the old one.
* Conflict: if someone else has written a newer version of that row since this transaction started, the write fails. Only one uncommitted version of a row is allowed at a time, so the first to write wins and the other aborts.
* Commit: every new version the transaction wrote is stamped with one commit timestamp, and becomes visible to transactions that start after it.

### 4.2 The basic version: a schema change as one normal transaction
* Step 1: write a new version of the table's schema record, using a normal write. If the transaction can't see the latest schema, it aborts.
* Step 2: run the migration with normal reads and writes. Every row gets a new version, converted to the new schema, added to the same version chain as the old data.
* Meanwhile, every ordinary transaction reads the schema version its snapshot can see, and a write also has to use the latest schema. Each transaction remembers which schema versions it used.
* At commit, every transaction checks that the schemas it used are still the latest. If not, it aborts. If they are, all its new versions, data and schema alike, become visible together.
* So the new schema and all the converted rows appear at the same moment, when the schema change commits, and a failure leaves nothing behind.

* To extrapolate into what I am doing: one transaction writes "S2: `status` is an integer" into the catalog, then writes a new version of all 2 million orders with `'paid'` stored as 2. Nothing is visible until it commits.

### 4.3 Why that doesn't work
The paper says following snapshot isolation strictly "is a double-edged sword that can make DDaM impractical", for three reasons:

1. It blocks ordinary writes. The schema change is long, a full scan plus a write for every row. Any ordinary transaction that updates a row the schema change has already rewritten hits a write-write conflict and aborts. Their example: T1 and T2 update the table while T3 runs `ALTER TABLE`, and lose. Transactions that touch other tables are fine.
2. The schema change itself can abort. If an ordinary transaction updates a row before the schema change reaches it, the schema change hits the conflict when it gets there, aborts, and all the work so far is wasted. Under "first committer wins", whether a schema change ever commits is "pure 'luck'".
3. It's expensive to track. Every row the migration writes goes into the transaction's write set, so for a big table the write set is huge and committing it is slow.

* In their evaluation, "almost no DDL transaction committed" this way once data had to be migrated.

> Question - This is E9's fight in a different form. In Postgres, the long `ALTER` waits for a lock and everyone queues behind it. Here, the long schema change and the short writes collide, and someone gets aborted. Either way, one long operation and many short ones can't share a table for free.

## Section 5 - Tesseract with Relaxed DDaM
In short: instead of updating every row where it is, copy every row, converted, into a hidden new table. Let live writes carry on against the old one, replay whatever they changed, then swap the new one in when the schema change commits.

* Out of place: the converted rows go into a new, invisible row array. The existing rows aren't touched, so ordinary transactions don't conflict with the copy.
* Two phases:
  1. Scan: copy every row that exists when the change starts.
  2. Change data capture (CDC): replay whatever ordinary transactions changed while the scan was running.
* "Relaxed snapshot": the scan copies the latest committed version of each row, not the version as of when the schema change started, and keeps that row's original commit timestamp. That's what lets the copy keep up with live writes.
* Ordinary transactions can keep committing during the migration. Their final commit is held back briefly until it's clear they don't conflict.
* The new schema is first visible in a "pending" state. When the schema change commits, the new array replaces the old one in one step.
* If a value can't be converted, the whole schema change aborts and the new copy is thrown away.
* A transaction that used the old schema can't commit after the new one (Section 3.3), so it's aborted.
* It can be combined with lazy migration (like BullFrog) for compatible changes. But lazy migration lets "incompatible schema changes" commit "without verification", leaving bad data to fix by hand.
* Aborted schema changes are called "very rare" with this design, with no numbers.

* To extrapolate into what I am doing:
  1. Start the change: S2 says `status` is an integer, but it's still pending.
  2. Scan: copy every order into the hidden array, with `'paid'` stored as 2, and so on.
  3. CDC: orders inserted or updated during the scan get replayed into the hidden array.
  4. Commit: the hidden array becomes `orders`. From then on everyone reads integers.

> Question - The scan-then-replay is gh-ost's idea, and my E3 backfill plus trigger, built into the database's own transaction log. No trigger on every write, and no separate tool. In Postgres, logical decoding could play the CDC part. Has anyone built that?

> Question - The hidden array is a full copy of the table until the swap, so storage is 2× for the duration, like E3 and BullFrog.

> Question - The swap is still one moment. Every query from the old app that starts after it gets integers. Tesseract keeps snapshots consistent, but it doesn't keep the old app working, and it doesn't rewrite queries.

## Section 6 - Evaluation

Results

| Schema change | Blocking | Lazy | Tesseract |
|---|---|---|---|
| Add a column, 100M rows | ~5 s near zero, then about half the normal throughput | Smaller dip, but slower for 5–6 s after | Up to ~10% drop, fully recovered after ~12–13 s |
| Add a constraint, and both together | Near zero during the change | — | Stayed near normal |
| Bigger tables (10M → 100M rows) | Much worse | Worse | Slightly bigger dip and longer, much less affected than the others |
| TPC-CD: split, aggregate, join tables | Highest overhead | — | About the same as Lazy, slightly better on some, slightly slower on adding a column |
| TPC-CD: create index, add constraint | Fewer transactions | — | More transactions than Blocking |

* CDC has to keep up: with 80% writes, 3–4 CDC threads fell behind, and 5 were enough.
* Not reported: abort rates, how long each migration took, memory or storage overhead.

> Question - None of the tested changes is ours. They added columns and constraints, split, aggregated and joined tables, and created indexes. `MODIFY COLUMN`, the copy-and-check change from Section 3.1, isn't in the list.

> Question - 40 cores, 375 GB of RAM, everything in memory, logs on a RAM disk, and 8 threads just for the migration. The "~10%" is from that machine. On a disk-based Postgres with real write-ahead logging, the copy and the replay both cost a lot more.

> Question - No abort numbers. Sections 3.3 and 5 both depend on aborting transactions that straddle the change. How many were there, and how often would an app have to retry?

## Summary of claims / benefits
* A schema change can be an ordinary transaction: versioned, snapshot-consistent, and rolled back on failure.
* Queries that started before the change keep seeing the old schema and the old data, without waiting.
* No views or triggers, and no separate migration tool.
* Copy plus replay lets ordinary writes keep going during the migration. Adding a column to 100M rows cost up to ~10% throughput, against near zero for a blocking change.
* Transactions that would mix old and new schemas are stopped at commit.

## My 2 cents

Tesseract is the most elegant answer to the database's half of the problem. No triggers, no views, no special lock: the schema is just another row with versions, and a migration is just a big transaction that copies in the background and swaps at the end. If Postgres worked like this, E2's 2.7-second read would have returned immediately.

But it's an engine change, on an in-memory research engine, and it still has a single moment where the schema flips for everyone. It also never tested a type change, which is the thing I actually need.

Some things I keep coming back to:

* Each request should see one consistent version. Tesseract pins every transaction to the schema its snapshot sees, and refuses to mix. The same rule makes sense one level up: each request from the app should be tied to one migration stage, and something should refuse writes that cross stages.
* Change capture beats triggers. Replaying from the log instead of firing a trigger on every write is what keeps Tesseract's cost low. In Postgres, logical decoding is already there and could keep a new column in sync without a trigger tax.
* An aborted write needs a plan. Tesseract and F1 both abort transactions that straddle a change. That's correct, but the app has to expect it and retry. Whatever coordinates the migration should make that retry automatic, not leave it to every developer.
* Consistent snapshots aren't the same as a working old app. Tesseract keeps old snapshots right, but the old app breaks the moment the swap commits. The translation between old and new shapes still has to live somewhere.
