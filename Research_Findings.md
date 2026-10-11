# Research Findings

Last updated after reading InVerDa.

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

## Side by side

| | Phase 1 view (E3) | PRISM | InVerDa |
|---|---|---|---|
| Old app reads | Yes | Yes | Yes |
| Old app writes | Yes (trigger) | No | Yes |
| Old and new apps live together | Partly | Old app reads only | Yes |
| Text → int value mapping | Yes (lookup table) | One way only | One way only (drop uses a default) |
| Keeps the index on selective reads | No | Unclear (depends on the rewrite) | No, for non-materialized versions |
| Upserts / constraints | No | Unclear | No (constraints out of scope) |
| Behaviour during migration | Measured, brief lock with queueing risk | Not discussed | "Available", not measured |
| Storage | 2× after backfill | Old DB copy until deploy | Stored once + auxiliary tables |
| Prepared statements / drivers | Measured (E4, E7) | Not discussed | Not discussed |
| Tested on | PostgreSQL 16, 2M rows, live writes | MySQL 5.0 / DB2, offline | PostgreSQL 9.4, single thread |
