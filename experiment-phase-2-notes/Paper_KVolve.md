# Paper - Evolving NoSQL Databases Without Downtime (ICSME, 2016)

Saur, Dumitraş, Hicks (University of Maryland) · [PDF (arXiv)](https://arxiv.org/pdf/1506.08800) · the system is called KVolve, built on Redis

## Section 1 - Introduction
* The usual way to change a data format is to shut down, migrate everything, and start again. That's downtime: "Wikipedia was locked for editing during the upgrade to MediaWiki 1.5."
* The usual workaround is lazy migration by hand: the app checks each item's format when it reads it, and converts old ones. But that "results in code that mixes application and format-maintenance logic".
* KVolve moves that lazy migration into the database. The app always sees data in the newest format, and the database converts old items the first time they're touched.


## Section 2 - The Problem with Online Upgrades
* NoSQL and key-value stores have no fixed schema in the database. The structure lives in the app: in the key names (like `user:42:profile`) and in the format of the values (often JSON).
* So "changing the schema" means changing key names, value formats, or both, and the database doesn't know a change happened.
* Two common approaches, both bad:
  * Stop the world and convert everything eagerly: downtime that grows with the size of the data.
  * Convert lazily in the app: every piece of app code has to know about every old format, forever.
* They looked at 18 of the most active GitHub projects using Redis, and found none where converting one item needed another item's value. That's why update functions are only allowed to see the item they convert (Section 3.2). 
* In a key-value store the "schema" lives entirely in the app, which is the extreme version of my problem. Postgres at least knows `status` is text. Redis has no idea what's inside a value.

## Section 3 - KVolve
### 3.1 Design
* Apps always see data at the newest version, so they don't need any version-handling code.
* Versions are tracked per key prefix, not for the whole database. `user:` keys and `order:` keys can be at different versions.
* Every stored item carries a hidden version tag saying what version it's actually in.
* A change is described by a version spec and an update function written in C. The function takes the old key and value, and rewrites them into the new format.
* When a client reads or writes an item whose tag is older than the current version, KVolve first converts it with the update function, then answers. If an item is several versions behind, the functions are chained automatically.
* Clients say which version of each prefix they expect when they connect. A client on an old version is disconnected, and "will not be allowed to reconnect until it is upgraded to the new version."
* To avoid restarting the app, they pair it with dynamic software updating (Kitsune), which swaps the app's code while it keeps running.

> Question - Disconnecting old clients is BullFrog's "big flip" again. Every app instance has to switch at once, or be swapped live with Kitsune, which only works for C programs. A rolling deploy isn't possible.

> Question - A read can become a write: reading a stale item converts and stores it. What happens on a read replica, or when the stored value can't be written for some reason?

> Question - The mapping is C code compiled into the database. The mapping lives in the right place (one function, not spread through the app), but writing and shipping C for every data change is a big ask.

### 3.2 Logical consistency
* The conversion has to happen in the same atomic step as the read or write that triggered it. Otherwise two clients could both convert the same item, and one of their writes would be lost.
* An update function may only look at the old version of the item it's converting. It can't read other keys.
* Key renames must be a clean one-to-one mapping of prefixes, so a key can always be traced back to its old name.
* The version tags and the update spec are stored durably, so after a crash the database still knows which version each item is in.

> Question - Durable version tags fix BullFrog's biggest gap, progress lost on a crash. Every item knows its own version, so there's no separate bitmap to lose.

> Question - "Only the item itself" is the same limit as PRISM's same-row functions. Our mapping is self-contained, so that's fine, but a change that needs to look something up in another key isn't possible.

## Section 4 - Implementation
* KVolve is a library compiled into Redis. It looks at every incoming command before Redis runs it.
* Atomicity comes for free: "All of this is sure to be atomic because Redis is single-threaded."
* The version tag is stored on each object. For containers (sets, hashes, lists, sorted sets), the version is on the whole container, so it all converts at once.
* Renamed prefixes are handled with a table of old and new prefixes, so a key can still be found under its old name.
* 36 Redis commands are covered, including all the main data structures.

> Question - "Atomic because Redis is single-threaded" doesn't carry over to Postgres. With many concurrent transactions, BullFrog needed a lock bit per row to get the same guarantee.

> Question - A container converts all at once. A big hash or list converts in one go the first time it's touched, which could make one request very slow.

## Section 5 - Experimental Results

Setup

| | |
|---|---|
| Machine | 24-core Intel Xeon E5-2430, 32 GB RAM, Red Hat Linux 6.5 |
| Measurement | Median of 11 runs, clients on the same machine (about 0.03 ms network latency) |
| Steady-state test | Redis's own benchmark: 50 clients, 5 million operations, 1 million keys |
| Real apps | Redisfs, a file system stored in Redis (123,002 keys), and Amico, a social graph app run on the LiveJournal dataset (792,711 keys) |

Overhead when nothing is being migrated

| | Result |
|---|---|
| One command per round trip | Between −0.8% and +2.3%, within run-to-run noise (about ±3%) |
| 10 pipelined commands | Up to 5.74% for string sets, 1% or less for most others |
| Memory (1M small values, 5 prefixes) | 112.1 MB → 128.6 MB, about 15% more |

During an update

| App | Stop-and-convert pause | KVolve pause | Afterwards |
|---|---|---|---|
| Redisfs (version 5 → 7) | 12 s | Close to zero | Throughput dipped slightly and recovered by about 120 s. Lazy conversions peaked at ~3K keys per second, then tapered off |
| Amico (1.2.0 → 2.0.0) | 87 s | Close to zero | About 2K fewer queries per second at the switch, then recovered. Still ~500 lazy conversions per second at 1,100 s |

* For Redisfs, the update renamed 251 directory keys, retagged 112,752 inode keys, and compressed 10K data keys.
* For Amico, many keys hadn't been touched yet, so the paper doesn't say when everything had been converted.
* Kitsune swapped the redisfs driver's code without unmounting it, described as seamless, with no number given.

> Question - "Close to zero" pause, against 12 and 87 seconds for stop-and-convert. That's the same win BullFrog showed, and the same shape as E3's fast cutover.

> Question - The migration never finishes on its own. Amico was still converting 500 keys a second after 18 minutes, and the old update functions have to be kept forever unless everything is forced to convert. There are no background threads like BullFrog's.

> Question - 15% more memory just for version tags and bookkeeping. For an in-memory store like Redis, that's a real cost, all the time, not only during a migration.


## Summary of claims / benefits
* Data format changes in a key-value store with almost no pause: close to zero, against 12 s and 87 s for stop-and-convert.
* Apps always see the newest format and need no version-handling code.
* Each item carries its own version, stored durably, so a crash loses nothing about the migration.
* Several versions can be chained, and each key prefix can be at its own version.
* Almost no overhead when nothing is being migrated.

## My 2 cents

KVolve is the cleanest version of lazy migration so far, and it got there five years before BullFrog. Putting the version on every item, durably, is a better design than BullFrog's in-memory bitmap: nothing to rebuild after a crash, and no extra structure to keep in sync.

But it comes with the same two costs. Old clients are thrown off at the switch, so there's no rolling deploy. And every read can turn into a write, with the migration never quite finishing because nothing pushes the untouched items along. It also relies on Redis being single-threaded for its atomicity, which a relational database can't.

Some things I keep coming back to:

* Version tags belong with the data. Knowing exactly which version a row is in, durably, makes lazy migration crash-safe and easy to reason about. In Postgres that could be as simple as which of two columns is filled in, or a small version column.
* The mapping should be declared, not compiled. KVolve itself says writing C transformers is tedious and asks for a small language. Our `'paid'` ↔ 2 mapping is a table, and should be written as one, readable by both the database and whatever sits in front of it.
* Lazy needs a finish line. Without background work pushing untouched rows along, a migration can run forever, and the old format and its conversion code can never be dropped.
* Disconnecting old clients is the simple answer, and the wrong one for a rolling deploy. Their own future work asks for keeping clients running through compatible updates. That's the same gap every lazy approach has had.
* Pay for versioning only during a migration. 15% more memory all the time is a lot. The bookkeeping should exist while a change is in progress and disappear when it's done.
