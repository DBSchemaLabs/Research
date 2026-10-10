# Phase 1: What PostgreSQL alone can do

Date: Oct 10, 2026  
Covers: Entries #2 and #4–#9 (experiments E1–E9). Scripts and logs are in [Benchmarks · experiment-phase-1](https://github.com/DBSchemaLabs/Benchmarks/tree/main/experiment-phase-1)

### The question (again)

Can `orders.status` go from text to integers while the database stays live and the app's SQL doesnt change, using only what Postgres gives me? No tools, no papers, just me and psql.

### Short answer

Kind of. The app can keep its SQL, but only because something inside the database keeps pretending the old shape is still there - either the enum's input function or a view with a trigger. And that pretending is never complete. Wherever it falls short, the app is the one that finds out.

When I started I wrote down "avoiding app changes means a translation layer inside the database" and felt quite smart about it. Its true, but it's only half the sentence. The other half is: and the layer leaks.

## What I tried

**The enum (E1, E2, E4, E7).** One `ALTER` and done - or so I thought. It rewrote the whole table and blocked a nine-row read for 2.7 seconds, which is a very long time to wait for nine rows. Then E4 showed it only really translates string literals. `LIKE` and `lower()` error out, `ORDER BY` and `>` quietly give you different answers, and prepared statements fall over once after the change. In Entry #2 I wrote that "the basic queries still worked". Technically correct. The best kind of correct, and also the least useful.

**The view (E3, E5, E6, E8).** Store an integer underneath, put a view with the old name on top, and let a trigger translate the writes. This one actually stays online - cutover was 36 ms, and inserts, updates, selects, COPY and row locks all kept working. I was pretty happy for about a day. Then a selective read lost its index (70 ms instead of 0.015 ms), upserts and TRUNCATE stopped working, the table doubled in size and writes got slower. So the view works great, as long as your app doesn't do any of those things. Mine would.

**The thing underneath both (E9).** Any DDL that has to wait for its lock makes every new query on that table queue up behind it. A metadata-only `ALTER` - one that needs basically no time to do its actual job - made a one-row read wait three seconds. `lock_timeout` fixes that, but now the ALTER fails and you get to retry it. Pick your poison.

## Scorecard

| | Enum                                           | View + trigger                |
|---|------------------------------------------------|-------------------------------|
| Stays online | No                                             | Yes (brief lock at cutover)   |
| Old SQL keeps working | Only literals                                  | Most of it                    |
| Fails with an error | LIKE, `lower()`, text parameters, cached plans | Upserts, TRUNCATE, ALTER      |
| Fails silently | Sort order, comparisons                        | Index loss on selective reads |
| Cost | Full rewrite + blocked reads                   | ~2× storage, slower writes    |

## What I learnt

- The silent failures scare me more than the errors. An error at least shows up in a log somewhere. Wrong sort order and a sudden full table scan just show up as a confused user or a slow page, and nobody connects it back to the migration from last tuesday.
- "No app change" depends a lot on *how* the app talks to the database, not just what SQL it sends. Parameter typing, prepared statements and upserts decided more of these tests than the SQL did.
- Don't trust a single number. 36 ms cutover looked amazing until E9 showed how long readers can wait if the cutover itself has to wait for its lock. I nearly put "36 ms" in bold somewhere. Glad I didn't.

## Still open

- Can the view be written so the planner sees the integer and keeps using the index?
- Does `FOR UPDATE` through the join view lock the lookup row too? If so, two orders with the same status could block each other, which would be a fun one to explain in an incident review.
- What would MySQL, Oracle or SQL Server do with the same change? Entry #3 only covered them from the docs.
