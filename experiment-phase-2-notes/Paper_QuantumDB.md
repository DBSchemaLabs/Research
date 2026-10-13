# Paper - Zero-Downtime SQL Database Schema Evolution for Continuous Deployment (ICSE SEIP, 2017)

de Jong, van Deursen, Cleve · [PDF](https://pure.tudelft.nl/ws/files/11926973/TUD_SERG_2017_005.pdf) · the system is called QuantumDB

## Section 1 - Introduction
* Some schema changes lock the whole table while every row is adjusted. Teams either accept downtime in a quiet window, or do expand/contract by hand over several coordinated deploys.
* The goal here is fully automatic schema changes with zero downtime.
* Evaluated with 19 synthetic schema changes and about 95 real ones from a commercial application.

## Section 2 - Background
* Zero-downtime deploys (rolling upgrade, or a big flip between two sets of servers) all go through a "mixed state", where the old and new versions of the service handle requests at the same time.
* So the database has to serve both versions' schemas at once, not just switch from one to the other.


## Section 3 - Problem Statement
* Three problems: some DDL blocks queries, the old and new app expect different schemas during the mixed state, and foreign keys have to keep holding throughout.
* Seven requirements:
  * R1, non-blocking: no client's queries should be blocked by a schema change.
  * R2, several non-trivial changes applied in one go.
  * R3, several schemas active at the same time.
  * R4, foreign keys enforced at all times.
  * R5, each schema isolated from the others.
  * R6, non-invasive: few or no app changes.
  * R7, resilient: a failed change can be rolled back.

  
## Section 5 - Transparent Schema Evolution with Ghost Tables
* Every changeset produces a new schema version with its own ID (like `80bfa11`).
* Mixed state with ghost tables:
  1. Create a "ghost" copy of each table the change touches, plus any table that depends on it through foreign keys. In their example `customers` changes, so `customers` and `rentals` (which points to customers) get ghost copies, but `movies` doesn't.
  2. Forward triggers on the original tables copy every insert, update and delete into the ghost tables. Updates are upserts, because the row might not have been copied yet.
  3. Existing rows are copied over in small batches to go easy on disk I/O.
  4. Backward triggers on the ghost tables copy changes back to the originals, so both versions stay in sync.
  5. A wrapper around the database driver rewrites table names in each query to the right physical table for that app's version. The app picks its version in the connection URL, e.g. `jdbc:quantumdb:postgresql://...?version=80bfa11`.

* To extrapolate into what I am doing:
  * `orders` gets a ghost table with `status_id` instead of `status`.
  * The forward trigger turns `'paid'` into 2, and the backward trigger turns 2 back into `'paid'`.
  * The old app connects with the old version in its URL and keeps writing text. The new app connects with the new version and writes integers.

> This is E3 with a second trigger going the other way: a shadow copy, a sync trigger, and a batched backfill. So E6's write cost should apply twice, since a write fires a trigger on one side and then the other. How do the forward and backward triggers stop firing each other in a loop? The paper doesn't say.

> Ghost tables are full copies of the changed table and everything that points to it. In E3, just one extra column doubled my table. Here `orders` and every table with a foreign key to it would exist twice for the whole mixed state.

> The wrapper only rewrites table names. Our change is about values, `'paid'` vs 2. Where does the mapping go? Section 5 only shows structural changes, like a new nullable column the old side never sees. Can I even give it a conversion function?

> Choosing a version in the connection URL is a config change, like `search_path` for pgroll or InVerDa. And it's a JDBC URL, so only Java apps get it. What about other drivers, and prepared statements across the switch (E7)?

> Creating ghost tables and installing triggers on a live table takes locks. The paper says it wants to avoid locking (R1), but doesn't say which locks these steps take, or for how long they'd wait behind a long transaction (E9).

* Leaving the mixed state:
  * The wrapper records which schema version every connection uses. An old version is dropped, with its triggers and tables, only once no connections use it.
  * Rolling back is easy, because the original tables are never changed during the mixed state. Drop the ghost tables, foreign keys and triggers, and you're back where you started.

> Tracking connections per version answers a question I had about InVerDa: who knows when the old version can go? But connection pools keep connections open for a long time. Does the old version only go once every pool has been recycled?

* Implementation: QuantumDB, open source, written in Java, targets PostgreSQL. It supports 11 schema operations and has over 300 tests.

## Section 6 - Evaluation
* All 19 scenarios from Section 4 were re-run through QuantumDB, on PostgreSQL 9.4 only. "QuantumDB is nonblocking for all scenarios." Their figure for the first scenario shows no visible change in query latency before and after the DDL.
* Real history from Magnet.me: 95 changesets with 532 schema operations over 46 weeks, about two schema changes a week. The oldest backup had 77 tables and 3.8 million rows.
* 35 of the 95 changesets used DDL that blocks, about a third.
* QuantumDB could run 61 of the 95 as they were. 24 more worked only partly, because they also needed hand-written DML (data changes), and 10 couldn't be used at all. Views and custom functions aren't supported.
* Replaying the 85 usable changesets took 2 h 25 min in total, about 1.7 minutes each, with no load on the database.
* There are no numbers for query latency, throughput or storage overhead. The triggers "add a small overhead".


## Section 7 - Discussion
* Liquibase ran all 95 changesets in about 8 minutes, but 35 of them would have blocked. QuantumDB is slower, and doesn't block.
* Tested under simulated load with their own tool, Nemesis. Not yet used in production at Magnet.me.
* Only two schemas can be active at once. That covers a rolling upgrade, but not canary releases with several versions live.
* Scaling is expected to be linear (double the data, double the time to reach the mixed state), but that wasn't measured. "The busier the database system the slower the copying process becomes."
* Trigger overhead on write-heavy workloads still needs research.
* No data is lost if something fails, because the original tables are never modified.
* Limits: PostgreSQL and JVM languages only, and no DML inside changesets.
* Foreign key chains: at Magnet.me, changing the `users` table needs ghost copies of "the majority of tables in the database".
* Everything comes from one company, which "may not be representative".


## My 2 cents

QuantumDB is the first one that feels like it was written by someone who has deployed software. It starts from the mixed state, measures blocking before designing anything, keeps foreign keys, and knows when the old version can go. Leaving the original tables untouched is a lovely property: rollback is just "drop the new stuff".

But underneath, it's my E3 again, done twice. Copies of tables, triggers in both directions, a batched backfill. And it pays for that in the places Phase 1 already flagged: storage (a whole foreign key neighbourhood copied), write cost (two triggers per write), and, for our case, the value mapping it can't really express.

Some things I keep coming back to:

* The layer in the driver is the most interesting part. QuantumDB already sits between the app and the database, and already knows which version each connection is on. Right now it only swaps table names. If that layer also knew the mapping, it could rewrite `status = 'paid'` to `status_id = 2`, aim writes at the right column, and handle upserts against a real table.
* Copying tables shouldn't be the default. Our change is one column. Adding `status_id` next to `status` in the same table, and filling it in, touches far less than copying `orders` and everything that references it.
* The version should be a stage, not just a connection setting. Each connection picks a version once, in its URL. During a migration the right SQL changes as the migration moves along, so the layer should learn the current stage from the database, rather than be told once at connect time.
* Knowing who's connected is the right idea for coordination. It needs to cope with pools that never reconnect, though, so the stage change should reach existing connections, not only new ones.
* Some writers will always skip the driver. A cron job or a psql session won't use the wrapper, so something in the database still has to keep both sides in agreement. QuantumDB's triggers do that, but they're on all the time, not only while the migration needs them.
* Data changes are part of the schema change. A value mapping is the whole point of our change, so it can't be the part that's left for the developer to do by hand.
