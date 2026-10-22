# Case study - GoCardless: Zero-downtime Postgres migrations, the hard parts

Chris Sinjakli, GoCardless blog · [post](https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts/)

## Notes-

- A planned migration took the GoCardless API down for about 15 seconds. It only added foreign keys from new, empty, unused tables. Run against a backup, the same migration took a few hundred milliseconds.
- The cause was the lock queue. On the Postgres version they ran, adding a foreign key took an `AccessExclusive` lock on both tables, even with `NOT VALID`. One of those tables was busy, and a slow `SELECT` was running on it. The `ALTER` queued behind the `SELECT`, and every later read and write queued behind the `ALTER`, until clients timed out.
- This is exactly my E9: a cheap `ALTER` stuck behind a long query, and a one-row read waiting three seconds behind it. GoCardless hit it in production, on a change that touched no data at all.
- Their fixes:
  - get rid of long-running queries on the primary, and send analytics to a replica;
  - turn on `log_lock_waits` and `log_min_duration_statement` to see trouble coming;
  - set `lock_timeout` in migrations, so a migration that can't get its lock fails instead of taking the app down;
  - split schema changes into small steps that each hold locks briefly;
  - keep Postgres up to date, since locking gets better with each release.
- Other rules from the same post: never rename a table or column the app still uses (copy and drop later), never rewrite a table under an exclusive lock (like adding a `NOT NULL` column with a default), build indexes `CONCURRENTLY`, and add constraints as `NOT VALID` then `VALIDATE` them separately with a weaker lock.
- Their bar for downtime: the database shouldn't stop answering for more than "a second or two".
- For me: the best real-world evidence that the problem isn't slow migrations, it's waiting. A migration that would take milliseconds caused 15 seconds of outage. No paper I read measures this. `lock_timeout` plus retry should be the default for every migration step, and the tooling should set it, not the developer.
