# Usage Matrix

Which approach to reach for, depending on what you can and can't live with. Based on my Phase 1 experiments (E1–E9) and the six papers read so far: PRISM, InVerDa, QuantumDB, F1, BullFrog and Tesseract. The running example is still `orders.status` going from text to a small integer.

Last updated after reading Tesseract.

## The approaches

| Approach | In one line | Where it comes from | Can I run it today? |
|---|---|---|---|
| In-place `ALTER` | Change the column type in one statement and let the database rewrite the table | E1, E2 | Yes, any PostgreSQL |
| Enum | `ALTER` to an enum so string literals keep working | E1, E4 | Yes, any PostgreSQL |
| Expand/contract with a view | New column, sync trigger, batched backfill, then a view with the old name | E3, E5–E8 | Yes, by hand |
| Query rewriting | Rewrite old queries onto the new schema | PRISM | No public code |
| Co-existing versions | Several schema versions live at once, all writable, through generated views and triggers | InVerDa | No public code |
| Ghost tables | Copy the changed tables, sync both ways with triggers, pick a version per connection | QuantumDB | Yes, Java and PostgreSQL, research prototype |
| Staged states with leases | Every change walks through delete-only and write-only states, with leases so servers are never more than one version apart | F1 | Not in PostgreSQL. The ideas live on in Spanner and CockroachDB |
| Lazy migration | Switch instantly, move rows only when they're asked for | BullFrog | PostgreSQL 11 extension, research prototype |
| Schema change as a transaction | Versioned schema; copy into a hidden table, replay live changes, swap at commit | Tesseract | Not in PostgreSQL. Research engine (ERMIA) |

## The matrix

Yes means it fits. Partly means it works with a catch. No means it breaks or doesn't apply.

| Your situation | In-place `ALTER` | Enum | View (E3) | Query rewriting | Co-existing versions | Ghost tables | Staged + leases | Lazy | Transaction |
|---|---|---|---|---|---|---|---|---|---|
| A short outage is acceptable | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| The database must stay online | No, reads block for the whole rewrite | No | Yes, brief lock at cutover | Not discussed | "Available", not measured | Yes, non-blocking in tests | Yes | Yes, instant switch | Yes, ~10% throughput drop in tests |
| Old and new app versions run at the same time (rolling deploy) | No | Partly, literals only | Partly, old app only | Old app reads only | Yes | Yes, two versions at most | Database servers yes, app not covered | No, old app rejected | No, one schema at a time |
| Every app instance can switch at once | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| The change re-encodes values (`'paid'` → 2) | Partly, via `USING` | Partly, literals only | Yes, lookup table | One way only | One way only | Hand-written | Not in the model | Yes, as SQL, one way | In the model, not tested |
| The change only adds something (a column, an index) | Yes, often instant | n/a | Overkill | Yes | Yes | Yes | Yes, this is its home ground | Yes | Yes |
| The table is write-heavy | Blocks writes while it runs | Blocks writes | Partly, ~5× slower writes through the view | n/a, reads only | Partly, triggers on every write | Partly, two triggers per write | Partly, writes +67% at p50 | Yes at moderate load | Partly, change capture has to keep up |
| Selective reads rely on an index | Yes after the rewrite | Yes | No, view loses the index | Unclear | No, except the materialized version | Likely yes | Yes | Yes on the new table, maybe not while migrating | Yes |
| The app uses upserts, `TRUNCATE` or ORM-driven `ALTER` | Yes | Partly, text-typed params fail | No | Not covered | No, constraints out of scope | Not discussed | Yes for constraints | Constraints redeclared | Yes, still a real table |
| The app uses prepared statements | Partly, one error per session | Partly | Yes, if types stay the same | Not discussed | Not discussed | Not discussed | Not discussed | Not discussed | Not discussed |
| Storage is tight | Yes | Yes | No, 2× after backfill | No, keeps a copy of the old DB | Yes, stored once | No, copies every referencing table | Partly, both columns during the change | No, two tables until the end | No, full copy until the swap |
| There are read replicas or read-only transactions | Yes after the rewrite | Yes | Yes | Not discussed | Not discussed | Not discussed | Yes | No, reads turn into writes | Not discussed |
| Long-running transactions are common | No, everyone queues (E9) | No | Partly, needs `lock_timeout` | Not discussed | Not discussed | Not tested | Yes, late writes are fenced | Not discussed | Partly, they don't block anyone but get aborted if they write across the change |
| Something writes outside the app (cron, psql) | Yes | Yes | Yes, triggers catch it | No | Yes, triggers | Yes, triggers | No, assumes every writer follows the protocol | Unclear, direct writes to the old table aren't tracked | Yes, every write goes through the engine |
| You need an easy rollback | No | No | Partly | Yes, inverse scripts | Yes | Yes, originals untouched | Yes, step back a state | No | Before commit yes, after it no |
| The database is distributed across many servers | No | No | No | No | No | No | Yes | No | No, single machine |

