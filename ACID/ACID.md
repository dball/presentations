# ACID

ACID is an acronym describing four classical database properties provided by
most common relational databases.

## What is a database?

A database is a system that maintains and provides projections of a set of data.

## What is a transaction?

An ordered sequence of database statements, including queries and commands

## What is ACID?

It's the properties you want from your shared persistent data storage.

* Atomicity guarantees that either commands of a transaction succeed, or none do.
  This is a requirement when performing coordinated data changes.

* Consistency: The data are consistent with the rules set forth in the schema.
  For example, there is a table named deals wherein is found the truth of all
  things.

* Isolation: Concurrent transactions are processed such that the subsequent
  database state is the result of a serial ordering of the transactions. In
  order to guarantee isolation, transactions may block or deadlock.

* Durability: A committed transaction may not be lost. This is a useful
  property for a database to have.

## Isolation

Of these, isolation is the most subtle and mysterious. Let's explore!

Mysql guarantees isolation using
MVCC, multi-version concurrency control. In a nutshell, this means it keeps track
of the portions of the database each transaction reads or writes. If another
transaction and blocks wants to read or write to the same portions concurrently,
it may join the lock (e.g. concurrent reads), block (wait until the lock is
released), or deadlock (if it holds a lock on which the other transaction is
blocked).

Of course, this elides much detail: what kind of locks are there, and what
do we mean by "portions of the database"?

## How do row locks work in mysql?

* Updates, deletes, and write-locking selects obtain write locks

* Read-locking selects obtain read locks; serializable transactions have the effect
  of obtaining read locks on all selects

* Locks aren't obtained on rows directly; they're obtained on the index values
  corresponding to the rows scanned in the query.

* Inserts are weird. They also obtain locks on the gap before the index value of the
  inserted rows.

* There's also intention locks, but we're going to move on now

## What are isolation levels?

* Serializable: Isolation is guaranteed. In mysql, this means all selects obtain
  write locks.

* Repeatable reads: Reads always return the same rows, but other transactions
  are not blocked from changing them. This means that a transaction may make
  changes based on a projection of the state of the database that is no longer true.

* Read committed: Reads are not consistent within the same transaction. I suppose
  this is useful if what you really care about is atomic rollback, but it seems
  ill-advised.

* Read uncommitted: Reads see uncommitted results from concurrent transactions.
  Hey, if you want to observe a state of the database that may never have been true,
  knock yourself out.

## Example

```
CREATE TABLE acid (
  id INT NOT NULL PRIMARY KEY,
  k INT NOT NULL UNIQUE KEY,
  v INT NOT NULL);

INSERT INTO acid (id, k, v) VALUES (1, 1, 1), (2, 2, 2);
```

```
BEGIN; t1
SELECT * FROM acid WHERE id = 1 FOR UPDATE; write locks id=1

BEGIN; t2
SELECT * FROM acid WHERE k = 1 FOR UPDATE; write locks k=1
```

t2 is now blocked trying to acquire a write lock on id=1

Interestingly! If t1 now tries to acquire a the write lock on k=1

```
SELECT * FROM acid WHERE k = 1 FOR UPDATE; deadlock!
```

This is counterintuitive. Naively, it would seem that t1 has a write lock on that
row already so this should be a noop. But mysql locks index values, not rows, and it
didn't need the k index in order to effect the first query in t1, so t2 was free to
acquire it.

It's important to realize that if we were to query on v:

```
SELECT * FROM acid WHERE v = 1 FOR UPDATE; write locks id=*
```

Because there is no key defined on v, it has to do a full-table scan and write lock
all of the primary key index values.

## Why do I even care about transactions?

You care if your data model requires the assistance of the database to enforce its
consistency constraints. For example, let's assume we record deals, consumers, and
purchases in a database. A constraint may be that when a purchase is recorded, the
deal and the customer must both exist and be active.

It's possible for this constraint to be satisfied by the data model itself. If deals
and customers are immutable upon becoming active, the constraint can be satisfied
simply by asserting the existence of such records when creating the purchase.
However, if the records are mutable or deletable, enforcing the constraint requires
help from the database. Specifically, asserting the existence of the active deal
and customer and recording the purchase must occur in a serializable transaction.

If you care about transactions, then you should care about the isolation levels and
the characteristics of the locks your transactions will acquire.

## Takeaways

* If you're confused about how concurrent transactions will work, open up two
  concurrent mysql transactions in the console and examine the behavior. Testing
  this stuff doesn't typically isn't timing-sensitive, it's order-sensitive.

* If your application can tolerate loosely enforced constraints, feel free to use a
  relaxed or non-existent level of isolation. My sense is that repeatable reads are
  adequate for most read-only transactions, but serializable should be strongly
  preferred for mutating transactions.

* Try to acquire your locks in a consistent order to avoid deadlocks. This means
  locking tables in a consistent order, and locking rows in a consistent order,
  probably in the order implied by the keys.

* Database design and lock acquisition order can prevent deadlocks, but it's difficult.
  Transactions should always be prepared to retry a number of times.