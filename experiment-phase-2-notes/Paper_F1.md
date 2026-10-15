# Paper - Online, Asynchronous Schema Change in F1 (VLDB, 2013)

Rae, Rollins, Shute, Sodhi, Vingralek (Google) · [PDF](https://www.vldb.org/pvldb/vol6/p1045-rae.pdf)

## Section 1 - Introduction
* F1 is Google's globally distributed relational database, built on Spanner, with strong consistency. It's what Google's ads business runs on.
* Schema changes have to happen with the database fully available, with little impact on performance, and without knowing which servers are currently running. That last one means "explicit global synchronization is not possible": there's no moment where everyone switches together.
* So different servers switch to a new schema at different times, and for a while several schema versions are in use at once.
* The example of why that's dangerous:
  * Server M2 already has the new schema, which includes index I. When it inserts row r, it also writes r's index entry.
  * Server M1 is still on the old schema, which has no index I. When it deletes r, it doesn't know to remove the index entry.
  * The index now points to a row that doesn't exist.


## Section 2 - Background
This section is closer to what I deal with day to day than the rest of the papers so far: rows spread across a distributed store, writes checked at commit, and servers that never all deploy at the same moment.

* F1 sits on a key-value store. Every non-key column of a row is stored as its own key-value pair, and so is every index entry.
* A schema is tables, columns, indexes, constraints and optimistic locks.
* Concurrency is optimistic: each column is covered by exactly one lock, and transactions check the lock's timestamp at commit.

## Section 3 - Schema Changes
* Rule: every server uses either the newest schema or the one just before it. Never more than two versions at once.
* Normally a schema element (a column, an index, a constraint) is either absent, meaning it doesn't exist yet, or public, meaning it exists and users can read and write it. F1 adds two in-between states that users never see:
  * Delete-only: deletes remove its key-value pairs, but nothing new is created, and users can't read it.
  * Write-only: inserts, updates and deletes all maintain it, but users still can't read it.

> Question - These states only cover adding and removing things: tables, columns, indexes, constraints. Nothing in the paper covers changing a type, renaming, or a change where old and new schemas read the same data differently. So our change has to be "add `status_id`, then drop `status`", and the link between them (`'paid'` = 2) is outside the model. The paper also says "we do not automatically rewrite queries to conform to the schema in use", so keeping the app working through the change is left entirely to the app. F1 handles database servers being on different versions, not the app being on a different version.

* Two kinds of things can go wrong while versions are mixed:
  * Orphan data: data the schema doesn't account for, like an index entry for a row that's gone (the example above).
  * Integrity problems: something that should be there is missing, or a constraint is broken.
* A change is safe only if it's safe against both the old and the new schema at all times. Going straight from absent to public is never safe: servers still on the old schema can't see the new element, so they'll leave orphans or skip writes it depends on.
* So every change goes through the in-between states, one step at a time:
  * Optional element (e.g. a nullable column): absent → delete-only → public. Dropping it goes the other way.
  * Required element (e.g. an index): absent → delete-only → write-only → backfill existing rows → public.
  * Constraint: absent → write-only → public. While write-only, it's checked for every new write, but old data may still break it.
* Changing which optimistic lock covers a column can let conflicting transactions through. That's fixed with a "dual coverage" step, where the column is covered by both locks for a while.

* To extrapolate into what I am doing:
  1. Add `status_id` as delete-only, then write-only: every new write sets it, nothing reads it.
  2. Backfill existing rows.
  3. Make it public: reads switch to `status_id`.
  4. Drop `status`: public → write-only → delete-only → absent.

> Question - In write-only, "F1 servers will ensure that it is properly maintained for new data". But for an index the server can compute the entry from the row. For `status_id`, someone has to know that `'paid'` is 2. F1's model is about structure (columns, indexes, constraints), not about one column being a function of another. Who supplies the value? In E3 it was my trigger.

> Question - If the old app keeps writing only `status` while `status_id` is write-only, every one of its writes needs translating. F1 assumes every writer is an F1 server running the protocol. What about writers that aren't, like a cron job or a psql session?

## Section 4 - Implementation
* Only one Spanner feature is needed for correctness: write fencing. A write can be given a deadline, and it can't commit once the schema it was written under has expired.

> Question - Why not just let a stalled write commit under the older schema? Because each step is only safe between neighbouring versions. A write from one version behind is fine; that's what the states are for. A write from two versions behind (a server that paused before the index was write-only, and woke after it went public) leaves a row with no index entry, which queries then silently miss. Fencing turns that into a retry. In our case it would be a client that wrote only `status` waking up after reads moved to `status_id`.

* The schema is a file in version control. Changes are applied about twice a week. A tool works out which in-between schemas and backfills a change needs, then applies them in order.
* Schema leases:
  * Every F1 server holds a lease on the schema, typically a few minutes long, and renews it by re-reading the schema from one well-known place.
  * A server that can't renew shuts itself down, and the cluster manager restarts it.
  * A new schema is written at most once per lease period. So by the time the next one comes, every server is either on the latest schema or can't commit at all. That's what keeps it to two versions.
* That waiting adds up. "A typical schema change involving no reorganization takes 10 to 20 minutes."
* Backfill runs as a background MapReduce job. It scans the rows as of when the change started, can be stopped and resumed, and is safe to run twice. Rows a user has already changed since then are skipped.
* Each server keeps a few schema versions in memory. New writes use the newest. A write still running on the old version can finish, unless its lease runs out, in which case it's cut off and has to be resubmitted.

## Section 5 - Impact on User Transactions
* Production numbers: about 1,500 F1 servers in five datacenters, 19 schema changes over about 10 weeks. Over a billion operations with no schema change running, and over 50 million during one. Roughly 75% reads, 25% writes.
* Latency, in milliseconds, without and during a schema change:

  | | Reads, normal | Reads, during | Writes, normal | Writes, during |
  |---|---:|---:|---:|---:|
  | p50 | 2.84 | 2.94 (+3.5%) | 107.16 | 179.26 (+67.3%) |
  | p90 | 79.82 | 74.23 (−7.0%) | 497.27 | 616.08 (+23.9%) |
  | p99 | 170.33 | 178.05 (+4.5%) | 965.69 | 1076.69 (+11.5%) |

* Fenced-off writes are resubmitted, which "causes wasted work", but there's no number for how often that happens.


## Summary of claims / benefits
* Schema changes can run with no global switch-over moment, and no downtime, across a large distributed database.
* Every change goes through delete-only and write-only states in a proven-safe order, so mixed versions never corrupt data.
* Leases and write fencing guarantee no more than two schema versions are ever in use.
* Backfill runs in the background, resumable and safe to repeat.
* Production impact is small for reads and moderate for writes, measured over 19 real changes on about 1,500 servers.

## My 2 cents

F1 is the most convincing paper so far, because it's the only one that's clearly been through production. It doesn't try to keep old queries working. It makes sure that while servers disagree about the schema, nothing gets corrupted, and it proves it.

What it doesn't do is exactly my problem. It covers the database's own servers being on different versions. It doesn't cover the app: there's no query rewriting, no value mapping, and no help for an old client that still writes `status`. So it's a very good answer to half the question.

Some things I keep coming back to:

* The states are the right shape for a migration. Add, write-only, backfill, switch reads, contract. That's what I did in E3 by instinct. Having names for them, and knowing which moves are safe, should apply to the app as well as the database.
* Leases and fencing are the missing coordination piece. Every other paper either ignored "who's on which version" or tracked it loosely. If app clients held a lease on the migration stage, and the database refused writes from clients on an expired stage, a stale client couldn't corrupt anything, even if it paused for a minute.
* The fence has to be in the database. A lease the client checks by itself isn't enough: a client that stalls can wake up and write after its lease has run out. F1 relies on Spanner to refuse the write.
* Slow but safe is a fair trade. 10 to 20 minutes for a change that nobody notices is better than 36 ms that makes everyone queue (E9).
* The value mapping still has nowhere to live. F1 knows how to add `status_id` and drop `status` safely, but not that one is the other in a different encoding. Something has to own that mapping and apply it during the write-only stage, for every writer.
