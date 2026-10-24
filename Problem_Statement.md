# Problem Statement and Related Work

We consider a schema change that alters how a value is stored, for example converting `orders.status` from text (`pending`, `paid`, `shipped`, `cancelled`) to a small integer, in a relational database that must stay online while old and new versions of the application run side by side during a rolling deploy. A satisfactory solution must:

(R1) never make other queries wait behind a migration step for longer than an agreed limit; 

(R2) serve old and new application versions at the same time; 

(R3) preserve the meaning of every query, including pattern matches, ordering, comparisons and parameterised statements; 

(R4) keep selective reads on their indexes; 

(R5) accept all writes, including upserts, and keep old and new representations consistent; 

(R6) prevent a stale or paused client from writing in a shape the migration has moved past; 

(R7) keep writers that bypass the application consistent; 

(R8) state the mapping between representations once, in both directions; and 

(R9) let the application's queries be checked against every stage before deployment.

We review existing work against these requirements, grouped by approach.

## 1 Schema change with built-in database features

The simplest approach is an in-place `ALTER TABLE ... ALTER COLUMN ... TYPE`. In our measurements on PostgreSQL 16 with two million rows, this rewrote the whole table in 2.2 s while holding an `ACCESS EXCLUSIVE` lock, during which a nine-row primary-key read waited 2.7 s [E1, E2]. Converting to an enum instead keeps string literals working, but expressions that produce text (`LIKE`, `lower()`) fail, ordering and range comparisons silently follow the enum's declaration order, and statements prepared with text-typed parameters cannot be prepared at all [E4]. A cached plan also fails once after the change [E7]. In-place changes therefore violate R1–R3.

PostgreSQL offers building blocks for safer changes: indexes can be built concurrently, constraints can be added as `NOT VALID` and validated later under a weaker lock, adding a column with a constant default no longer rewrites the table since version 11, and `lock_timeout` bounds how long a statement waits for its lock [13]. None of these changes a column's representation without a rewrite. Moreover, even a metadata-only `ALTER` that must wait behind a long-running transaction blocks every later query on the table: in our measurements a one-row read waited 3 s, versus 0.7 ms when the `ALTER` used `lock_timeout` [E9].

The common practice is expand/contract: add a new column, keep it in sync, backfill, switch, and drop the old column [10, 11, 12]. Implemented inside the database with a synchronisation trigger and a compatibility view over the new storage, it stays online with a 36 ms cutover and no mapping inconsistencies, but doubles the table during backfill (88 → 184 MB) [E3]. The view hides the change from old queries at a cost: a selective read through it loses its index and becomes about 4,700× slower (70.4 ms vs 0.015 ms) [E5], writes through its trigger are about 5× slower than a plain table [E6], and `ON CONFLICT`, `TRUNCATE` and `ALTER` fail against it [E8]. Implemented in the application instead, as at Stripe, it avoids these problems but requires dual-write code, offline backfill pipelines and production verification of old and new code paths [10].

## 2 Co-existing schema versions

A line of work keeps the old schema available as a logical view over the new one. PRISM [1] expresses a change as a sequence of schema modification operators, from which it generates migration scripts, inverse operators and rewritings of old queries onto the new schema, automating 97.2% of query rewrites across Wikipedia's schema history. It addresses reads only (updates are left as future work), moves between finished versions rather than supporting a live transition, and can express value conversion only as a one-way per-row function. InVerDa [2] extends this to bidirectional operators, so that all versions are readable and writable simultaneously, with generated views and triggers that perform within about 4% of hand-written code. However, only one version is materialised as real tables; every other version is a view and inherits the costs measured in [E5, E6, E8]. BiDEL has no operator that states two columns encode the same value, so a row written through the new version appears in the old one with the dropped column's default, and integrity constraints are out of scope. Oracle Edition-Based Redefinition offers the same co-existence natively through editioning views and crossedition triggers [14], and pgroll automates expand/contract on PostgreSQL by exposing each schema version as views selected via `search_path` [15]. Both rely on the same view-and-trigger machinery, and the application must still choose its version.

## 3 Online physical migration

A second line of work focuses on moving data without blocking. QuantumDB [3] targets the mixed state of a rolling deploy directly. It copies affected tables into ghost tables kept in sync by forward and backward triggers, tracks which connections use which schema version, and drops a version only once no connection uses it. All 19 tested schema changes were non-blocking on PostgreSQL 9.4. The cost is two triggers per write, ghost copies of every table that references the changed one (in one deployment, most of the database), a mapping that must be written by hand since data changes are not supported within a changeset, and a JDBC-only client layer. Trigger-based tools for MySQL such as pt-online-schema-change suffer similar overheads; GitHub reported lock contention, near-lockdowns and multi-second stalls when removing triggers, and built gh-ost, which instead synchronises the shadow table from the replication log and can truly pause and throttle [9]. These tools move data, not meaning: if the representation changes, the application must change with it.

F1 [4] addresses schema change in a distributed database without global synchronisation. Every change passes through intermediate delete-only and write-only states in an order proven to avoid orphaned data and integrity violations, and schema leases with write fencing guarantee that no more than two schema versions are in use, rejecting writes from servers whose lease has expired. In production, read latency was nearly unaffected (p99 +4.5%) while writes slowed by 67% at the median. F1 is the only system that satisfies R6, but its guarantees cover database servers, not the applications above them, it does not rewrite queries, and its model covers adding and removing elements rather than re-encoding a value. CockroachDB inherits this approach and supports online column type changes only for columns without indexes or constraints [16]. Tesseract [7] takes a different route, treating a schema change as an ordinary MVCC transaction over a versioned catalog. Transactions see a schema consistent with their snapshot, a transaction that used an older schema cannot commit after a newer one, and migration copies data out of place and replays concurrent changes from the log, costing up to about 10% throughput when adding a column to 100 million rows. It requires changes to the storage engine (an in-memory research engine was used), switches all clients at one moment, and its evaluation does not include type changes.

## 4 Lazy migration

Lazy approaches defer data movement. BullFrog [5] makes the new schema active immediately and migrates old rows when a query first touches them, with background threads covering the rest and a per-row bitmap ensuring each row moves exactly once; at moderate load its throughput and latency were close to running without a migration. The mapping is ordinary SQL, which can join a lookup table. However, for changes that are not backward compatible the old schema is rejected at the switch, so all application instances must change at once; reads become writes, which fails on read-only transactions and replicas; and progress is held in memory without crash recovery. KVolve [6] applies the same idea to Redis with a durable version tag on every item, which makes progress crash-safe, but disconnects clients on the old version, leaves migration incomplete until every item is touched, and relies on Redis being single-threaded for atomicity.

## 5 Practice

An empirical study of six years of migrations in one Django application on PostgreSQL 9.6 found blocking operations in 87.5% of months but enough to plausibly cause visible downtime in only about 5.5%, with full-table rewrites as the main source of harm; over 70% of changes were additive and type changes were rare [8]. Industry reports are consistent with this. GoCardless lost 15 s of API availability to a migration that touched no data, because an `ALTER` queued behind a slow query and every later query queued behind it [11]. Braintree's operational rules reduce a type change to adding a column, writing to both, and backfilling [12]. Stripe moved hundreds of millions of records with application-side dual writes in four phases [10], and GitHub replaced trigger-based tools with log-based synchronisation [9]. None of these uses the research systems above; all perform expand/contract by hand, and all place the mapping in application code.

## 6 Summary

Table 1 summarises how each approach meets the requirements.

| | R1 | R2 | R3 | R4 | R5 | R6 | R7 | R8 | R9 |
|---|---|---|---|---|---|---|---|---|---|
| In-place `ALTER` [E1] | No | No | No | Yes | Yes | No | Yes | Partly | No |
| Enum [E4] | No | Partly | No | Yes | Partly | No | Yes | Partly | No |
| View + trigger [E3] | Partly | Partly | Yes | No | No | No | Yes | Partly | No |
| Manual expand/contract [10–12] | Partly | Partly | Yes | Yes | Yes | No | Partly | No | No |
| PRISM [1] | ? | Partly | Partly | ? | No | No | No | Partly | Partly |
| InVerDa [2] | ? | Yes | Yes | Partly | Partly | No | Yes | Partly | No |
| QuantumDB [3] | Partly | Yes | Yes | Partly | Partly | No | Yes | No | No |
| F1 [4] | Yes | No | ? | Yes | Yes | Yes | No | No | No |
| BullFrog [5] | Partly | No | Yes | Partly | Yes | No | ? | Partly | No |
| KVolve [6] | Partly | No | n/a | n/a | Yes | Partly | ? | Partly | No |
| Tesseract [7] | Yes | No | Yes | Yes | Yes | Yes | Yes | Partly | No |
| pgroll [15] | Partly | Yes | Partly | No | Partly | No | Yes | Partly | No |
| Oracle EBR [14] | Partly | Yes | Yes | ? | Partly | No | Yes | Partly | No |
| CockroachDB [16] | Yes | No | ? | Yes | Partly | Yes | Yes | No | No |

Table 1: Existing approaches against requirements R1–R9. "?" means the source does not report it.

Three observations follow. First, the logical approaches (PRISM, InVerDa, EBR, pgroll) keep old versions alive but do so through views and triggers, whose costs we measured directly. Second, the approaches that guard against stale writers (F1, Tesseract) do so inside the database engine, for the database's own servers or transactions, and do not extend to the application. Third, no approach states a value mapping once, in both directions, in a form that both the database and the application can use, and none checks the application's queries against each stage of a migration before it runs.

The problem we address is therefore the following. For an application on a relational database and a schema change that alters how a value is stored, run the change online, with old and new versions of the application live at the same time, such that at every stage of the migration every query keeps its meaning or is rejected before it ships, selective reads keep their indexes, all writes including upserts go to real tables and stay consistent, no client can write in a stage that has been left behind, and no step makes other queries wait beyond an agreed limit, while the application's only change is to declare the mapping once.

## 7 The gap and what is required

Taken together, the approaches above already contain most of the pieces a solution needs, but no single approach puts them together, and none puts them where the application can use them. We see five gaps, each pointing to a requirement for any future solution.

The mapping is never a first-class object. Every approach keeps the relationship between old and new representations somewhere implicit: a trigger body (E3, QuantumDB), a per-row function (PRISM, InVerDa), a C transformer (KVolve), a SQL query (BullFrog) or application code (Stripe, Braintree). A solution needs the mapping to be declared once, in both directions, and classified as reversible or lossy, so that the same declaration can drive data movement, query translation, consistency checks and rollback.

Translation happens in the wrong layer. Keeping old queries working inside the database, through views and triggers, costs indexes, upserts and write throughput (E5, E6, E8), while leaving it to the application costs dual-write code in every team (Stripe, Braintree). A solution needs translation to happen before a query reaches the database, so the database only ever sees plain SQL against real tables, with values it can use indexes for, and so queries that cannot be translated faithfully are rejected rather than silently changed.

The migration has no shared notion of its own progress. F1 shows that a schema change is safest as a sequence of explicit stages, and that clients must never fall more than one stage behind. Outside F1's own servers, nothing tracks which stage a migration is in or which clients know about it. A solution needs the stage to be explicit, durable, visible to every client, and enforced by the database, so that a stale or paused client's write is refused rather than applied.

Not every writer can be relied on to cooperate. Cron jobs, ad hoc sessions and other services write to the same tables. A solution needs a database-side safety net that keeps old and new representations consistent for those writers, and is active only while a migration needs it, so that its cost is not paid permanently.

Problems are found after deployment. Every failure in our experiments (E4, E5, E7, E8) was found by running the query, and the industry reports describe outages discovered in production (GoCardless). A solution needs the application's real queries to be checked against every stage of a migration before it runs, and every step to carry a bound on how long it may make other queries wait.

In short, what is required is a declared, two-way mapping; translation of application queries per migration stage, ahead of the database; a durable, enforced migration stage shared by all clients; a temporary database-side safety net; and checks and lock-wait bounds applied before and during every step. Existing work demonstrates each of these in isolation. Combining them for live applications is the open problem.

## References

[1] C. Curino, H. J. Moon, C. Zaniolo. Graceful Database Schema Evolution: the PRISM Workbench. PVLDB 1(1), 2008.

[2] K. Herrmann, H. Voigt, A. Behrend, J. Rausch, W. Lehner. Living in Parallel Realities: Co-Existing Schema Versions with a Bidirectional Database Evolution Language. SIGMOD 2017.

[3] M. de Jong, A. van Deursen, A. Cleve. Zero-Downtime SQL Database Schema Evolution for Continuous Deployment. ICSE SEIP 2017.

[4] I. Rae, E. Rollins, J. Shute, S. Sodhi, R. Vingralek. Online, Asynchronous Schema Change in F1. PVLDB 6(11), 2013.

[5] S. Bhattacherjee, G. Liao, M. Hicks, D. J. Abadi. BullFrog: Online Schema Evolution via Lazy Evaluation. SIGMOD 2021.

[6] K. Saur, T. Dumitraş, M. Hicks. Evolving NoSQL Databases Without Downtime. ICSME 2016.

[7] T. Hu, T. Wang, Q. Zhou. Online Schema Evolution is (Almost) Free for Snapshot Databases. PVLDB 16(2), 2022.

[8] W. Lawrence, E. Adachi. Assessing the Occurrence of Blocking Operations in Database Schema Evolution: A Case Study. VEM 2021.

[9] S. Noach. gh-ost: GitHub's online schema migration tool for MySQL. The GitHub Blog, 2016.

[10] J. Xu. Online migrations at scale. Stripe Engineering Blog, 2017.

[11] C. Sinjakli. Zero-downtime Postgres migrations: the hard parts. GoCardless Blog.

[12] Braintree. Safe operations for high-volume PostgreSQL. 2014.

[13] PostgreSQL Global Development Group. PostgreSQL 16 Documentation: ALTER TABLE.

[14] Oracle. Database Development Guide: Using Edition-Based Redefinition.

[15] Xata. pgroll documentation.

[16] Cockroach Labs. Online schema changes and ALTER COLUMN TYPE documentation.

[E1]–[E9] refer to our own experiments, with scripts and logs in the Benchmarks repository (experiment-phase-1) and notes in experiment-phase-1-notes.
