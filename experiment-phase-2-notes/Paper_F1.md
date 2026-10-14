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


