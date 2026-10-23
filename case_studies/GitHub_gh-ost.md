# Case study - GitHub: gh-ost, triggerless online schema changes for MySQL (2016)

Shlomi Noach, The GitHub Blog · [post](https://github.blog/engineering/databases/gh-ost-github-s-online-migration-tool-for-mysql/) · [code](https://github.com/github/gh-ost)

## Notes-

- GitHub changes its MySQL schema "multiple times per day". MySQL's built-in online DDL caused replication lag and couldn't be paused or throttled, and migrating replica by replica needed a brief outage to promote one.
- The existing tools (pt-online-schema-change, Facebook's OSC, LHM, oak) all use triggers to keep a shadow table in sync. GitHub found that painful in production:
  - every write to the table pays for the trigger;
  - triggers compete for locks inside the app's own transactions, which caused "near or complete lockdowns";
  - removing the triggers at the end sometimes stalled on metadata locks for "many seconds to a minute";
  - you can pause the row copy, but not the triggers, so a throttled migration can still overload the database.
- gh-ost drops triggers entirely. It builds a ghost table, copies rows in chunks, and keeps it in sync by reading the database's replication log (the binlog), then swaps the tables at the end. All of its writes go through one connection, so the database sees one well-behaved writer.
- Because it controls every write, it can genuinely pause and throttle: on server load, on replication lag, on a flag file or a command. Settings like chunk size can be changed while it runs.
- They tested it with continuous no-op migrations on every production table, on replicas, comparing checksums, and ran "thousands of successful migrations" there before trusting it on the primary.
- The final table swap can be postponed with a flag file, so a person can choose when the cutover happens.
- By the time of the post, every production migration at GitHub ran through gh-ost, daily and sometimes several times a day.
- For me: this is the industry's answer to the trigger cost I measured in E6 and saw in QuantumDB. Reading the log instead of firing a trigger is the same idea Tesseract built into its engine. In Postgres, logical decoding could play that role. It still swaps tables at one moment, and it does nothing about the app's queries, so the old app and the new schema are still the app's problem.
