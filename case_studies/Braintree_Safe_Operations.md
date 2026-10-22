# Case study - Braintree: Safe operations for high-volume PostgreSQL (2014)

Braintree (PayPal) engineering, via pgrs.net · [post](https://www.pgrs.net/2014/05/14/safe-operations-for-high-volume-postgresql/)

## Notes-

- Braintree runs payments on PostgreSQL and almost never takes scheduled downtime, so migrations run while the payment API is serving traffic. Their bar: "any operation that locks a table for updates for more than a few seconds means downtime for us."
- From "extensive testing, trial and error", they ended up with a list of what's safe and what isn't on a big, busy table.
- Safe: add a new column, drop a column (fast, though the space isn't reclaimed without `VACUUM FULL` or pg_repack), add an index `CONCURRENTLY`, drop a constraint, add a default to an existing column.
- Unsafe, with their workarounds:
  - Change a column's type: add a new column, write to both, backfill. This is our change, and their answer is my E3, done by hand.
  - Add a column with a default: add it, set the default separately, then backfill.
  - Add a `NOT NULL` column: create a new table with it, write to both tables, backfill, then switch. They call this "incredibly onerous" and say they rarely do it.
  - Add a unique column: add it, build a unique index concurrently, then add the constraint using that index.
  - `VACUUM FULL`: use pg_repack instead.
- The post is from the PostgreSQL 9.x era. Some of these have since got easier (adding a column with a default stopped rewriting the table in PG 11).
- For me: this is what every Postgres team ends up writing down sooner or later, a list of rules instead of a tool. Changing a type is "add, dual-write, backfill" every time, and the dual-write part is app code. Nothing here keeps the old app's queries working. That's left to whoever writes the migration.
