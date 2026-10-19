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
