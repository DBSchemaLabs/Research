# Case studies

What real teams ran into when changing a database schema under live traffic. These sit next to the papers in `experiment-phase-2-notes/`: the papers say what's possible, these say what people actually do.

| Case study | What it's about |
|---|---|
| [Lawrence & Adachi (2021)](LawrenceAdachi_Blocking_Operations.md) | Six years of one Django/PostgreSQL app's migrations: how often schema changes block, and how often it actually hurts |
| [Stripe (2017)](Stripe_Online_Migrations.md) | Moving hundreds of millions of records to a new table with app-side dual writes, in four phases |
| [GitHub gh-ost (2016)](GitHub_gh-ost.md) | Why GitHub dropped trigger-based schema change tools for MySQL, and built one that reads the replication log |
| [GoCardless](GoCardless_Lock_Queue.md) | A 15-second API outage from a migration that touched no data, caused by the Postgres lock queue |
| [Braintree (2014)](Braintree_Safe_Operations.md) | The list of safe and unsafe schema changes a payments company runs PostgreSQL by |

## What they have in common

- Nobody uses the research systems. Every team runs some form of expand/contract by hand: add the new thing, fill it gradually, switch over, remove the old.
- Changing a column's type is always "add a new column, write to both, backfill", and the "write to both" part is app code.
- The outages come from waiting, not from slow work. GoCardless lost 15 seconds to a change that took milliseconds. `lock_timeout` and small steps are everyone's fix.
- Triggers are a known pain at scale. GitHub replaced them with reading the replication log.
- Safety comes from verification and small steps, not from one clever mechanism: Stripe compared old and new code paths in production, GitHub ran thousands of test migrations on replicas.
- None of them keeps the old app's queries working without changing the app. Either the app changes in many small steps (Stripe), or the change is designed so the old app never notices (Braintree's rules).
