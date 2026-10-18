# Research Findings

Last updated after reading Tesseract.

## The problem

I want to store `orders.status` (`pending`, `paid`, `shipped`, `cancelled`) as a small integer instead of text, while the database stays live and the application's SQL doesn't change. Phase 1 showed that PostgreSQL alone can get part of the way there, but every approach I tried left problems behind. These are the ones I'm now looking for answers to in the papers:

* Old queries need to keep their meaning, not just their syntax. The enum kept `status = 'paid'` working, but `LIKE` and `lower()` broke and `ORDER BY` silently changed (E4).
* Translation shouldn't cost performance. The view kept old queries working, but a selective read lost its index and became ~4,700× slower (E5), and writes got slower (E6).
* Writes matter as much as reads. Upserts, `TRUNCATE` and `ALTER` failed once `orders` was a view (E8).
* The mapping has to work both ways. The old app writes text and the new one writes integers, and both have to agree on every row.
* The change has to stay online for real. An in-place change blocked reads for the whole rewrite (E2), and even an instant `ALTER` stalled readers while it waited for its lock (E9).
* Drivers matter. Parameter typing and prepared statements decided whether a query survived the change (E4, E7).
* Storage matters. A backfill doubled the table, and dropping the old column didn't give the space back (E3).

None of these are new problems. Researchers have been working on changing schemas under live applications for well over a decade, and some of their ideas made it into real tools. So before building anything myself, I'm reading the main papers to see which of these problems already have good answers, which ones only work on paper, and which ones nobody has really solved yet. The goal is to borrow what works and to find the gaps that are actually worth my time.


## Papers read

### PRISM (VLDB 2008) · [notes](experiment-phase-2-notes/Paper_PRISM.md)

PRISM describes a schema change as a sequence of small named operators (SMOs). Because each step is known exactly, PRISM can generate the migration SQL and inverses, and rewrite old queries onto the new schema automatically.

What it solves
* Old queries keep working after a schema change, by rewriting them or through generated views.
* 97.2% of Wikipedia's 170 schema changes needed no manual query rewriting.
* Lossy steps are flagged before they happen.

What it lacks
* It only supports old queries that read. Updates were future work.
* It says nothing about running live, such as locks, downtime, or old and new apps running at the same time. It goes from one finished version to the next.
* There's no two-way mapping. Our change fits `add column status_id as f(status)`, but `drop column status` counts as lossy unless I write the reverse function myself.
* It's unclear whether `status = 'paid'` becomes `status_id = 2` (index-friendly) or something the planner can't use.
* By default it keeps an untouched copy of the old database until deploy.
* It was tested on MySQL 5.0 and DB2, with Wikipedia only and no live traffic.

### InVerDa / BiDEL (SIGMOD 2017) · [notes](experiment-phase-2-notes/Paper_InVerDa.md)

InVerDa keeps several schema versions alive at once, all readable and writable, with a write in one showing up in the others. Each change is a few lines of BiDEL, a bidirectional version of PRISM's operators. InVerDa generates the views and triggers between versions, and the DBA can move the physical data to a different version with one command.

What it solves
* It adds the writes PRISM was missing. Every version is fully writable, and round trips are proven not to lose data.
* Old and new apps can run side by side, so the database and the app don't have to change in lockstep.
* 3 lines of BiDEL replace ~359 lines of hand-written SQL, within ~4% of hand-optimized performance.
* Where the data physically lives can follow the workload. Writes were 49× faster when the physical layout matched the version being written.

What it lacks
* Only one version is a real table. Every other version is views and triggers, which is my E3 setup, generated. So E5 (index loss), E6 (write cost) and E8 (no upserts, TRUNCATE or ALTER) should all apply.
* It can't re-encode a value. There's no operator for changing a type, only add and drop. If I'm reading the rules right, a row the new app writes would show up in the old version with the drop's default (`'pending'`), not its real status.
* It doesn't carry constraints across versions, so no primary keys, unique constraints or foreign keys, which is what upserts depend on. Listed as future work.
* It doesn't define online. Versions "stay available" during `MATERIALIZE`, but there's nothing on locks or what live traffic sees, and zero-downtime is future work.
* It has no plan for the deployment. Apps still have to pick a version, and someone has to track which app is on which one before the old version can be dropped.
* It was tested on PostgreSQL 9.4 with a single thread and 100,000 rows, so there's nothing on concurrency or tail latency.

### QuantumDB (ICSE SEIP 2017) · [notes](experiment-phase-2-notes/Paper_QuantumDB.md)

QuantumDB starts from the in-between period, where old and new versions of the app run at the same time during a rolling deploy. It makes a "ghost" copy of every table the change touches, keeps the copies in sync with triggers in both directions, and backfills existing rows in small batches. A wrapper around the JDBC driver points each connection at its version's tables, and the old version is dropped once no connection uses it.

What it solves
* Old and new apps run side by side, each seeing its own schema, which is what a rolling deploy needs.
* None of the 19 measured schema changes blocked queries through QuantumDB, on PostgreSQL 9.4.
* Foreign keys hold the whole time, unlike InVerDa.
* The original tables are never modified, so a failed change rolls back cleanly with no data loss.
* It tracks which connections use which version, so it knows when the old version can go.
* About two thirds of a real company's changesets (61 of 95) ran through it as they were.

What it lacks
* It's E3 done twice. A write fires a trigger on one side and then the other, and there are no numbers on what that costs.
* It copies a lot. Every table that points to the changed one gets a ghost copy too, and at Magnet.me that meant most of the database for a change to `users`.
* It can't express our value mapping cleanly. The wrapper only rewrites table names, and data changes (DML) inside a changeset aren't supported, so the `'paid'` ↔ 2 mapping would be written by hand.
* The version is fixed per connection, in a JDBC URL. That's a config change, Java-only, and long-lived connection pools could keep an old version alive.
* Only two schemas can be live at once.
* It was evaluated on PostgreSQL 9.4, the replay ran with no load, it hadn't been used in production yet, and everything came from one company.

### F1 (VLDB 2013) · [notes](experiment-phase-2-notes/Paper_F1.md)

F1 is Google's distributed database, where there's no moment when every server switches schema together. So each change walks through two hidden in-between states, delete-only and write-only, in an order that's proven never to corrupt data while servers disagree. Leases on the schema and write fencing guarantee that no more than two schema versions are ever in use.

What it solves
* No global switch-over and no downtime, across about 1,500 servers in five datacenters.
* A proven-safe order for every change: add, write-only, backfill, then public, and the reverse for drops. It's the order I used in E3, with names and a proof.
* Coordination is guaranteed rather than hoped for. After one lease period, every server is on the new schema or can't write.
* Long-running writes don't hold everyone up. A write that outlives its lease is fenced off and retried, which is the opposite of E9's lock queue.
* Real production numbers with tail latency. Reads barely changed (p99 +4.5%), writes got slower (p50 +67%).

What it lacks
* It doesn't help the app at all. There's no query rewriting, and the paper says so.
* It only adds and removes things. There's no type change, rename, or value mapping, so `'paid'` = 2 is outside the model.
* It assumes every writer is an F1 server running the protocol. A cron job or psql session isn't covered.
* It's slow. A change with no backfill takes 10 to 20 minutes of waiting for leases.
* There are no numbers on how many writes were fenced off and retried.
* It's built on Spanner. Write fencing needs the storage layer to refuse late writes, which PostgreSQL doesn't do on its own.

### BullFrog (SIGMOD 2021) · [notes](experiment-phase-2-notes/Paper_BullFrog.md)

BullFrog switches to the new schema the moment the DDL runs, without moving any data first. The new table is defined by a plain SQL query over the old one, and rows move only when a query asks for them, with background threads moving the rest. A bitmap in memory makes sure each row moves exactly once, even with many transactions touching it.

What it solves
* The cutover is instant, with no backfill to wait for.
* The mapping is ordinary SQL, so it can join a lookup table like E3's `order_status`. Our change is the simplest case they support, one old row to one new row.
* At moderate load (450 TPS on TPC-C), throughput and latency were close to running with no migration at all, and up to an order of magnitude better than eager or multi-step migration.
* It's a Postgres extension, with no changes to Postgres itself.

What it lacks
* It cuts off the old app. For a change that isn't backward compatible, like ours, every old instance is rejected the moment the DDL runs, so everything has to switch at once.
* Reads turn into writes. Reading a row that hasn't moved yet migrates it first, which is a problem for read-only transactions and replicas.
* Progress is tracked only in memory, and crash recovery isn't implemented.
* There are still two copies of the data until the old table is dropped, and the migration only ends when the background threads finish.
* Near capacity (700 TPS), the migration work tipped the system over and latency went to seconds.
* It was tested on one 8-core machine with TPC-C, only the new app running, and no replicas.

### Tesseract (VLDB 2023) · [notes](experiment-phase-2-notes/Paper_Tesseract.md)

Tesseract treats a schema change as an ordinary transaction. The schema is a row in a catalog table, versioned like any other row, so every transaction reads the schema its snapshot can see. A migration copies every row, converted, into a hidden new table, replays whatever live writes changed during the copy, and swaps the new table in when it commits.

What it solves
* Queries that started before a change keep seeing the old schema and old data, without waiting. E2's 2.7-second read would return immediately.
* No views, no triggers, no separate tool. Changes made during the copy are replayed from the database's own log.
* A transaction that would mix old and new schemas is stopped at commit, like F1's fencing on one machine.
* Adding a column to 100M rows cost up to ~10% throughput, against near zero for a blocking change.

What it lacks
* It needs a different engine. It's built on ERMIA, an in-memory research database, and its key trick (swapping a row-ID array) has no direct equivalent in Postgres.
* The schema still flips for everyone at one moment. The old app gets the new schema as soon as the change commits, and there's no query rewriting.
* Our kind of change wasn't tested. The evaluation added columns and constraints, split, aggregated and joined tables, and created indexes, but never changed a column's type.
* The hidden copy means 2× storage until the swap.
* Transactions that straddle a change are aborted, and there are no numbers on how often.
* The ~10% comes from a 40-core, 375 GB machine with everything in memory and logs on a RAM disk.

## Side by side

| | Phase 1 view (E3) | PRISM | InVerDa | QuantumDB | F1 | BullFrog | Tesseract |
|---|---|---|---|---|---|---|---|
| Old app reads | Yes | Yes | Yes | Yes | Not addressed (no query rewriting) | No, rejected after the switch | Until the swap; after it, sees the new schema |
| Old app writes | Yes (trigger) | No | Yes | Yes | Not addressed | No, rejected after the switch | Aborted if it straddles the change |
| Old and new apps live together | Partly | Old app reads only | Yes | Yes, two at most | Database servers yes, two versions at most; apps not addressed | No (big flip), unless the change is backward compatible | No, one schema at a time; old snapshots stay consistent |
| Text → int value mapping | Yes (lookup table) | One way only | One way only (drop uses a default) | Hand-written (no DML in changesets) | Not in the model (add and drop only) | Yes, as SQL (can join a lookup table); one way only | In the model (`MODIFY COLUMN`), not evaluated |
| Keeps the index on selective reads | No | Unclear (depends on the rewrite) | No, for non-materialized versions | Likely, each version reads a real table (not tested in the paper) | Yes, no views involved | On the new table yes; finding unmigrated rows may scan the old one | Yes, no views involved |
| Upserts / constraints | No | Unclear | No (constraints out of scope) | Foreign keys kept; upserts not discussed | Foreign key and unique constraints go through a write-only state | Constraints redeclared on the new table; checking them means migrating more | Yes, still a real table; constraints checked as part of the change |
| Behaviour during migration | Measured, brief lock with queueing risk | Not discussed | "Available", not measured | Non-blocking in 19 tests; lock queues not tested | Measured in production: reads p99 +4.5%, writes p50 +67%; late writes fenced | Instant switch; near-invisible at 450 TPS, latency in seconds at 700 TPS | Up to ~10% throughput drop on 100M rows; readers never wait |
| Storage | 2× after backfill | Old DB copy until deploy | Stored once + auxiliary tables | Ghost copies of the table and everything referencing it | Both columns during the change; leftovers garbage-collected | Old and new tables until the old one is dropped | Full hidden copy until the swap |
| Prepared statements / drivers | Measured (E4, E7) | Not discussed | Not discussed | JDBC wrapper only; prepared statements not discussed | Not discussed | Not discussed | Not discussed |
| Tested on | PostgreSQL 16, 2M rows, live writes | MySQL 5.0 / DB2, offline | PostgreSQL 9.4, single thread | PostgreSQL 9.4, simulated load, one company | Google production, ~1,500 servers, 19 changes | PostgreSQL 11, one 8-core machine, TPC-C | ERMIA (in-memory), 40 cores, YCSB and TPC-C variant |
