# Case study - Stripe: Online migrations at scale (2017)

Jacqueline Xu, Stripe Engineering Blog · [post](https://stripe.com/se/blog/online-migrations)

## Notes-

- Stripe had hundreds of millions of subscription records stored inside customer records, first as a single subscription, later as an array. Queries and updates on that array had become slow, so subscriptions needed their own table. The Subscriptions service couldn't be paused, so the move had to happen at full availability.
- They did it in four phases, all in the application, not the database: write to both stores, switch reads to the new one, switch writes to the new one (archiving to the old), then remove the old data. It's expand/contract, done by the app.
- Dual writes were ramped up slowly by percentage while they watched database metrics. Old records were copied lazily when they were next updated, then backfilled in bulk.
- The backfill avoided touching production for the search: a database snapshot was loaded into Hadoop, a MapReduce job listed what still needed copying, and a fleet of workers copied it in parallel. The same job was run again afterwards to check nothing was missed.
- Every read and write change was checked in production with GitHub's Scientist library, which runs the old and new code paths side by side and reports mismatches. Reads only moved over once the results matched.
- Code was changed in small pieces: they "never attempted to change more than a few hundred lines of code at one time". Once reads had moved, touching the old array raised a hard error, which flushed out any code still depending on it.
- No timeline or rollback procedure is given.
- For me: this is the opposite answer to "no app change". Stripe changed the app on purpose, in many small steps, and made the app do the migration. It worked, but it took dual-write code, a Hadoop pipeline, a verification library and a lot of engineering time. Most teams can't afford that for every schema change, which is exactly the gap a tool should fill.
