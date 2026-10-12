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

