# Paper - BullFrog: Online Schema Evolution via Lazy Evaluation (SIGMOD, 2021)

Bhattacherjee, Liao, Hicks, Abadi · [PDF](https://par.nsf.gov/servlets/purl/10283168) · [code](https://github.com/umd-dslam/BullFrog)

## Section 1 - Introduction
* BullFrog's idea: switch to the new schema in one step, immediately, without moving any data yet. Data moves later, a little at a time, when it's actually needed.

## Section 2 - Request-Driven Lazy Migration
* The migration is written as ordinary DDL (schema statements like `CREATE TABLE`), with the new table defined by a query over the old ones: `CREATE TABLE new AS (SELECT ... FROM old ...)`. There's no special language like PRISM's operators or InVerDa's BiDEL. Their example renames a table, adds derived and timestamp columns, and builds it from a join of two old tables.
* So during the migration there are two tables: the old one, still holding rows that haven't moved yet, and the new one, filling up as rows are asked for. BullFrog also keeps a temporary view with the same query, used to work out which old rows a query needs. Once everything has moved, the old table can be dropped.

> Question - What does keeping two tables cost? Storage peaks at about two copies: stale rows stay in the old table, marked as migrated but not deleted, until the whole old table is dropped. So no space comes back until the very end, and the end depends on how fast the background threads get through rows nobody asks for. On 2 million rows that's fine. On a table that's most of the disk, it isn't. Then there's the time cost: the first query to touch an unmigrated row pays for migrating it (a read on the old table, a write to the new one, a bitmap update) before it gets its answer. And every migrated row is written again, which means extra WAL and extra load on any replicas. None of this goes away. It's the same work as E3's backfill, just spread over real requests instead of done up front.

* "The new schema becomes immediately active as soon as it is processed by BullFrog."
* When a query touches the new table, BullFrog first migrates the old rows that query could need, then runs the query on the new table.
  * Filters are pushed down to the old table, using Postgres's own view expansion, so only matching rows move.
  * UPDATEs and DELETEs are first turned into SELECTs on the old table, to migrate their rows.
  * INSERTs usually don't need anything migrated first, unless a new constraint (like uniqueness) has to be checked against old rows.

> Question - So is the latency cost no longer in the tail? It's no longer one big stall like E2, but it doesn't vanish either. Whichever query first touches an unmigrated row pays to move it, so the median barely changes while unlucky queries get slower. That's still a tail. A broad query, like counting every `status_id = 2`, could have to migrate thousands of rows before it answers. And two requests wanting the same row wait on each other. The cost also changes over time, high at the start and falling as the background threads catch up. I'll check part 2's numbers for where the tail is worst.

* Background threads send fake requests to move whatever no real query touches. When they finish, migration is done and the old tables can be deleted.
* A row never exists in both schemas at once. Stale copies can stay in the old table, but they're marked as migrated so nothing reads them.
* Constraints have to be declared again for the new table.
* Limits: filters inside user-defined functions can't be pushed down, and a new constraint can make some old rows fail to migrate.

* To extrapolate into what I am doing:
  ```sql
  CREATE TABLE orders_new AS (
    SELECT o.id, s.id AS status_id
    FROM orders o JOIN order_status s ON s.name = o.status
  );
  ```
  * The mapping is just SQL, so it can join E3's `order_status` lookup table instead of being stuck inside a function like in PRISM.
  * From the moment it runs, the new app reads and writes `orders_new`. Old rows move over the first time someone asks for them.

> Question - Every read can turn into a write: reading a row that hasn't moved yet migrates it first. What happens in a read-only transaction, or on a read replica, where writes aren't allowed?

## Section 3 - Lazy Migration, Concurrently
* The main job is making sure each row is migrated exactly once, even with many transactions touching the same rows at the same time.
* Migrations are grouped by how old rows map to new ones:
  * 1:1, one old row becomes one new row: adding or dropping columns, changing types, adding constraints, joins on a foreign key. Our change is this kind.
  * 1:n, one old row becomes several, like splitting a table.
  * n:1, several old rows become one, like a `GROUP BY` aggregate.
  * n:n, like many-to-many joins.
* For 1:1 and 1:n, a bitmap keeps two bits per old row: "locked" (someone is migrating it right now) and "migrated" (done).
* For n:1 and n:n, a hash table tracks each group, since a whole group has to move together.
* The migration runs in its own transaction just before the client's transaction. Splitting it that way makes aborts simpler and avoids deadlocks. If the migration transaction aborts, its bits are reset so someone else can try.
* A simpler option is Postgres's `ON CONFLICT DO NOTHING` on the new table, but it only works when there's a deterministic unique key.
* The bitmap and hash tables live in memory. After a crash they'd have to be rebuilt from the log, and that recovery "is not yet implemented".

> Question - In-memory tracking with no crash recovery is a big gap for something meant for production. If the database restarts halfway through, which rows have moved?

* After the switch:
  * For a change that isn't backward compatible, "the old schema becomes inactive, and all subsequent requests that access it are rejected." Every app instance has to switch at the same moment, which they call a "big flip".
  * For backward-compatible changes, old app versions can keep running while they're updated gradually.
