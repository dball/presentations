# Big Migrations

* How don't we migrate?
  * Many many records
  * Can't just walk the old records and create new records, underlying data are mutable
  * Can't lock the records while migrating
  * Also can't (easily) atomically switch the code to use the new tables when committing data migration

* How do we transition?
  1. Build an eventually consistent synchronization process:
     1. Write an (idempotent) function to reconcile an entity from old to new
     1. Begin maintaining a queue of entity changes recorded atomically when entities change
     1. Begin asynchronously processing said queue using that fn
     1. Walk the old entities and reconcile them
     1. Validate the results offline, e.g. using data warehouse
  1. Record changes to both databases instead of maintaining the queue of changes
     1. This requires transactions to two databases, so can fail
     1. Failure means we either manually sync the failure or redo phase 1
     1. Validate the results again
  1. Switch the reading code to the new database
     1. The new database is now the system of record
     1. Probably switch the order in which we commit the transactions to reflect this
  1. Stop writing to the old tables
  1. Begin writing to the new tables correctly (respecting domain immutability)
