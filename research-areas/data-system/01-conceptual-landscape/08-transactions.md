# Transactions

Motivation
- implementing **fault-tolerance** mechanisms is a lot of work
  - it requires careful thought about all the things that can go wrong and rigorous testing to ensure that the solutions that are implemented actually work
- **transactions** have been the mechanism of choice for **simplifying these issues**
  - conceptually, all the reads and writes in a transaction are **executed as one operation**
  - either 
    - the entire transaction succeeds, resulting in a **commit**
    - or it fails, resulting in an **abort** or **rollback**
  - transactions were created with a purpose: 
    - **to simplify the programming model for applications accessing a database**
    - using transactions allow applications to ignore certain potential error scenarios and concurrency issues, because the **database takes care of them** instead


## What Exactly Is a Transaction?

NoSQL movements
- nonrelational (NoSQL) databases aimed to **improve upon the relational** status quo by 
  - offering a choice of new data models
  - including replication and sharding by default
- **transacctions** were the main **casualty** of this movement
  - many of this generation of databases abandoned transactions entirely
  - or refined the word to describe a much weaker set of guarantees than had previously been understood
- the **hype around NoSQL distributed databases** led to a popular belief that 
  - **transactions were fundamentally unscalable** 
  - any large-scale system would have to abandon them in order to maintain good performance and high availability
- but recently, that belief has turned out to be **wrong**
  - "NewSQL" databases have shown that transactional systems **can scale to large data volumes and high throughput**
    - such as CockroachDB, TiDB, Spanner, FoundationDB, and YogabyteDB
  - these systems combine **sharding** with **consensus protocols** to provide strong ACID guarantees at scale

---
### The Meaning of ACID

- ACID stands for **atomocity, consistency, isolation, durability**
- one databases's impplementation of ACID does not equal another's
  - e.g., there is a lot of ambiguity around the meaning of **isolation**

Atomicity
- **in general**, atomic refers to something that cannot be broken into smaller parts
- has subtly different meanings in **different branches of computing**
  - **multithreaded programming**
    - one thread executes an atomic operation means there is no way that another thread could see the half-finished result of the operation
    - the system can be only in the state it was before the operation or after the operation, not something in between
- in the context of **ACID**
  - atomicity is not about concurrency
    - it does not describe what happens if several processes try to access the same data at the same time, that's **isolation**
  - it describes **what happens if a client wnats to make several writes, but a fault occurs after some of the writes have been processed**
- if the **writes are grouped** together into an **atomic transaction** 
  - the transaction cannot be completed (committed) because of a fault
  - the transaction is **aborted** and the database must discard or **undo any writes** it has made so far in that transaction
  - if a transaction was aborted, the application can be sure that it didn't change anything, so it **can safely be retried**

Consistency
- you have certain statements about your data (**invariants**) that must always be true
  - e.g., in an accounting system, credits and debts across all accounts must always be balanced
- rule
  - transaction **starts** with a database that is **valid** according to the invariants
  - any writes during the transaction **preserve the validity**
  - then you can be sure that the invariants are always satisfied
  - (an invariant may be **temporarily violated** during transaction execution, but it should be satisfied again at transaction commit)
- enforce
  - you need to declare them as **constraints** as part of the schema
    - e.g., foreign-key constraints, uniqueness constraints, check constraints (restrict the values that can appear in an individual row)
    - more complex consistency requirements can sometimes be modeled using **triggers** or **materialized views**
  - **not a property of the database alone**
    - if you write bad data that violates your invariants, but haven't declared those invariants, the database can't stop you
    - consistency often depends on how the application uses the database

Isolation
- **concurrency problems** (race conditions)
  - most databases are accessed by several clients at the same time
  - that's no problem if they are reading and writing different parts of the database
  - but if they are accessing same database records, you can run into concurrency problems
- isolation means that concurrently executing transactions are isolated from each other
  - they cannot step on each other's toes
- the classic database textbooks formalize isolation as **serializability**
  - means that each transaction can pretend that it is the only transaction running on the entire database
  - database ensure that when the transactions have committed, the result is the same as if they had **run serially**, even though in reality they may have **run concurrently**
- **reduce performance cost** of serializability
  - many databases use forms of isolation weaker than serializability
    - i.e., they allow concurrent transactions to interfere with each other in limited ways
  - some popular databases (Oracle) don't implement it
    - Oracle has an isolation level called "serializable" but it actually implements **snapshot isolation**, which is weaker guarantee than serializability


Durability
- durability is the promise that **after a transaction has committed successfully, any data it has written will not be forgotten**, even if there's a hardware fault or the data crashes
- in a **single-node database**, durability usually means the data has been written to nonvolatile storage such as a hard drive or SSD
  - regular file writes are usually buffered in memory before being sent to the disk sometime later
    - which means they may be lost if there is a sudden power failure
  - many databases therefore use the **fsync system call** to ensure that the data really has been written to disk
  - databases usually also have a **write-ahead log** or similar feature
    - which allows them to recover in the event that a crash occurs partway through a write
  - many databases store their data with a **checksum**
    - which allows them to detecct corrupted or incomplete log entries and thus help restore the database to a consistent snapshot after a crash
- in a **replicated database**, durability may mean that the data has been successfully copied to a certain number of nodes
  - to provide a durability guarantee, a database must wait until these writes or replications are complete before reporting a transaction as successfully committed


---
### Single-Object and Multi-Object Operations

Quick recap
- atomicity and isolation in ACID describe what the database should do if a client makes several writes within the same transaction
  - atomicity says if an error occurs halfway through a sequence of writes, the transaction should be aborted
  - isolation says concurrently running transactions shouldn't interfere with each other
- these definitions assume that you want to **modify several objects** (rows, documents, records) **at once**
- such **multi-object transactions** are often needed if several pieces of data need to be **kept in sync**

Multi-object transactions
- they require some way of determining w**hich read and write operations belong to the same transactions**
  - in **relational databases**, that is typically done based on the client's TCP connection to the database server
    - on any particular connection, everything between a `BEGIN TRANSACTION` and a `COMMIT` statement is considered to be part of the same transaction
    - if the TCP connection is interrupted, the transaction must be aborted
  - many **nonrelational databases** don't have such a way of grouping operations together
    - even if there is a multi-object API doesn't necessarily mean it has transaction semantics
      - meaning, the command may succeed for some keys and fail for others, leaving the database in a partially updated state
      - an example of multi-object API is: a key-value store that may have a multi-put operation that updates several keys in one operation

Single-object writes
- atomicity and isolation also apply when a single object is being changed
  - e.g., writing a 20 kB JSON document to a database
    - if the network connection is interrupted after the first 10 kB have been sent, does the database store that unparseable 10 kB fragment of JSON?
    - if the power fails while the database is in the middle of overwriting the previous value on disk, do you end up with the old and new values spliced together?
- storage engines hence almost universally aim to **provide atomicity and isolation** on the level of a **single object** (such as a key-value pair) **on one node**
  - **atomicity** can be implemented using a **log for crash recovery**
  - **isolation** can be implemented using a **lock** on each object (allowing only one thread to access an object at one time)
- some databases also provide **more complex atomic operations**
  - **increment operation** removes the need for a read-modify-write cycle
  - **conditional write** operation allows a write to happen only if the value has not been concurrently changed by someone else
  - **compare-and-set** or **compare-and-swap**(CAS) operation in shared-memory concurrency
- single-object operations can prevent **lost updates** when several clients try to write to the same object concurrently
  - but they are not transactions in the usual sense of the word
  - e.g., Aerospike's "strong consistency" mode and "lightweight transactions" feature of Cassandra and ScyllaDB offers **linearizable** reads and **conditional writes** on a single object
    - but no guarantees across multiple objects

The need for multi-object transactions
- in some cases, single-object inserts, updates, and deletes are sufficient
- but, in many other cases, **writes to several objects** need to be coordinated
  - relational data model and graph-like data model
    - in a relational data model, a row in one table often has a **foreign-key reference** to a row in another table
    - in a graph-like data model, a vertex has edges to other vertices
    - multi-object transactions allow you to ensure that these references remain valid
      - when inserting several records that refer to one another, the foreign keys have to be correct and up-to-date
  - document data model
    - the fields that need to be updated together are often within the same coument, which is treated as a **single object**
    - but the lack of join functionality also encourage **denormalization**
      - when denormalized information needs to be updated, you need to update several documents in one go
      - transactions are very useful in this situation to prevent denormalized data from going out of sync
  - database with **secondary indexes**
    - the indexes also need to be updated every time you change a value

Handling errors and aborts
- ACID databases are based on the transaction philosophy
  - if the database is in danger of violating its guarantee of atomicity, isolation, or durability, it would rather abandon the transaction entirely than allow it to remain half-finished
- not all systems follow this philosophy though
  - in particular, datastores with leaderless replication:
    - "the database will do as much as it can, and if it runs into an error, it won't undo something it has already done"
  - so it is the application's responsibility to recover from errors
- although retrying an aborted transaction is a simple and effective error-handling mechanism, it isn't perfect
  - if the transaction successded, but the **network was interrupted while the server tried to acknowledge the successful commit** to the client
    - so it timed out from the client's point of view
    - in this case, retrying the transaction causes it to be performed twice unless having additional application-level dedup mechanism in place
  - if the error is **due to overload** or high contention between concurrent transactions
    - retrying the transaction will make the problem worse
    - you can limit the number of retries, use exponential backoff, and handle overload-related errors differently from other errors
  - it is worth retrying only after **transient errors**, and after a **permanent error** would be pointless
    - transient errors: due to deadlock, isolation violation, temporary network interruptions, or failover
    - permanent error: constraint violation
  - if the transaction also has **side effects** outside of the database, those side effects may happen even if the transaction is aborted
    - e.g., you wouldn't want to repeatedly send the email every time you retry the transaction
    - **two-phase commit** can help make sure that several systems either commit or abort together

## Weak Isolation Levels

Motivation
- concurrency issue
  - if two transactions don't access the same data, or if both are read-only, they can safely be run in parallel, because neither depends on the other
  - concurrency issues (race conditions) come into play when 
    - one transaction reads data that is concurrently modified by another transactions
    - or when two transactions try to modify the same data
- unique difficulties
  - concurrency bugs are hard to find by testing, because such bugs are triggered only when you get unlucky with the timing
  - concurrency is also difficult to reason about, especially in a large application where you don't necessarily know which other pieces of code are accessing the db
- **transaction isolation**
  - because of the above difficulties, databases have tried to **hide concurrency issues** from application developers by providing transaction isolation
  - in theory, isolation should make life easier by letting you pretend that no concurrency is happening
    - **serializable isolation** means that the database guarantees that transactions have the same effect as if they ran serially — one at a time, without any concurrency
  - in practice, isolation is not that simple
    - serializable isolation has a **performance cost**, and many databases don't want to pay that price
    - hence, systems commonly use **weaker levels of isolation**, which protect against some concurrency issues but not all
  - concurrency bugs caused by weak transaction isolation and race conditions are not just a theoretical problem, they caused substantial loss of money
    - e.g., bankrupting a Bitcoin exchange led to investigation by financial auditors
    - caused customer data to be corrupted
  - also have to consider the possibility that an attacker might deliberately send a burst of highly concurrent requests to your API in an attempt to exploit concurrency bugs

---
### Read Commited
- it makes **two guarantees**
  - when reading from the database, you will only see data that has been commited (no dirty reads)
  - when writing to the database, you will only overwrite data that has been commited (no dirty writes)

No dirty reads
- definition
  - imagine a transaction has written some data to the database, but the transaction has not yet committed or aborted, can another transaction see that uncommitted data?
  - if so, that's called a **dirty read**
- read-committed isolation level guarantees
  - any writes by a transaction become visible to others only when that transaction commits (and then all its writes become visible at once)
- reason to prevent
  - if a transaction needs to update several rows, a dirty read means that another transaction may see some of the updates but not others
    - e.g., user 2 sees the new unread email but not the updated counter  
![alt text](images/0801.png)
  - if a transaction aborts, many writes it has made need to be rolled back
    - if the database allows dirty reads, a transaction may see data that is later rolled back
    - any transaction that read uncommitted data would also need to be aborted, leading to a problem called **cascading aborts**

No dirty writes
- scenario and definition
  - what happens if two transactions concurrently try to update the same row in a database?
    - we don't know which order the writes will happen, but we normally assume that the later write overwrites the earlier one
  - what happens if the earlier write is part of a transaction that has not yet committed?
    - so the later write overwrites an uncommitted value
    - this is called a **dirty write**
- rule of thumb
  - read-commit isolation level must prevent dirty writes, usually by delaying the second write until the first write's transaction has committed or aborted
- concurrency problems avoided
  - if transactions update multiple rows, dirty writes can lead to a bad outcome
    - e.g., consider the situation where a used car sales website on which two people, Aaliyah and Bryce, are simultaneously trying to buy the same car  
![alt text](images/0802.png)
      - buying a car requires two database writes
        - listing on the website needs to be updated to reflect the buyer
        - the sales invoice needs to be sent to the buyer
      - here
        - the sale is awarded to Bryce (because he performs the winning update to the listings table)
        - the invoice is sent to Aaliyah (because she performs the winning update to the invoice table)
    - read-committed isolation prevents such mishaps
  - however, read-committed isolation does not prevent the race condition between two counter increments here  
![alt text](images/0803.png)
    - in this case, the second write happens after the first transaction has committed, so it is not a dirty write
    - it's still incorrect, this is an example of **lost updates**

Implementing read-committed
- read-committed is a very popular isolation level, it is the default setting in Oracle Database, PostgreSQL, SQL Server, and many other databases
- **row-level locks** — prevent **dirty writes**
  - when a transaction wants to modify a particular row, it must first acquire **a lock on that row**
    - it must then hold that lock until the transaction is committed or aborted
  - only one transaction can hold the lock for any given row
    - if another transaction wants to write to the same row, it must wait until the first transaction is committed or aborted before it can acquire the lock and continue
  - this **locking is done automatically** by databases in read-committed mode (or at stronger isolation levels)
- **dirty reads**
  - one option would be to use the same lock and to require any transaction that wants to read a row to briefly acquire the lock and then release it again immediately after reading
    - this ensure that a read couldn't happen while a row had a dirty, uncommited value
      - because during that time the lock would be held by the transaction that was making the write
      - and databases don't simply tell this read transaction that "the row is locked under earlier write transactions", the read transaction has to issue a read-lock first
    - however, requiring read locks does not work well in practice
      - one long-running write transaction can force many other transactions to wait until the long-running transaction has completed
        - even if the other transactions only read and do not write anything to the database
      - this harms the response time of read-only transactions and is bad for operability
        - a slowdown in one part of an application can have a knock-on effect in a completely different part of the application, due to waiting for locks 
  - a more commonly used approach
    - for every row that is written, the database remembers both the old committed value and the new value set by the transaction that currently holds the write block
    - while the transaction is ongoing, any other transactions that read the row are simply given the old value
    - only when the new value is committed do transactions switch over to reading the new value
  - some databases support an even weaker isolation level called **read uncommitted**
    - it prevents dirty writes but does not prevent dirty reads
    - in other words, it immediately returns the latest written value, even if the writing transaction hasn't committed yet
    - this can provide better performance, since the database does not need to store two versions of the row
    - it can also reduce the probability of (but not prevent) lost updates


---
### Snapshot Isolation and Repeatable Read

Motivation
- superficially, read-committed isolation seems to have done everything that a transaction needs to do
  - it allows aborts (required for atomicity)
  - it prevents reading the incomplete results of transactions
  - it prevents concurrent writes from getting intermingled
- however, there are many **other ways to have concurrency bugs** when using read-committed isolation level
  - scenario  
![alt text](images/0804.png)
    - Aaliyah has 1k of savings at a bank, split across two accounts with 500 each
    - a transaction transfers 100 from one of her accounts to the other
    - if she looks at her list of account balances in the same moment as that transaction is being processed, she may see
      - one account balance before the incoming payment has arrived (500)
      - the other account after the outgoing transfer has been made (the new balance being 400)
    - to Aaliyah, it now appears as though she has only a total of 900 in her accounts
  - this anomaly is called **read skew**, and it is an example of **nonrepeatable read**
    - if Aaliyah were to read the balance of account 1 again at the end of the transaction, she would see a different value (600) than she saw in her previou query
  - **read skew** is considered **acceptable** under **read-committed isolation**
- in Aaliyah's case, this is not a lasting problem, but such **teporary inconsistency is not tolerable** in the following, for example
  - backups
    - taking a backup requires making a copy of the entire database, which may take hours for a large database
    - during the time that the backup process is running, writes will continue to be made to the database
    - thus, you could end up with some parts of the backup containing an older version of the data and other parts containing a newer version
    - if you need to restore from such a backup, the inconsistencies (such as disappearing money) become permanent
  - analytical queries and integrity checks
    - sometimes you may want to run a query that scans over large parts of the database
      - e.g., they may be part of a periodic integrity check that everything is in order (monitoring for data corruption)
    - these queries are likely to return nonsensical results if they observe parts of the database at different points in time
- **snapshot isolation** is the most common solution to this problem
  - the idea is that **each transaction** reads from a **consistent snapshot** of the database
    - i.e., it sees all the data that was committed in the database at the start of the transaction
    - even if the data is subsequently changed by another transaction, each transaction sees only the old data from that particular point in time
  - snapshot isolation is a boon for long-running, read-only queries such as backups and analytics
    - when a transaction can see a consistent snapshot of the database, forzen at a particular point in time, it is much easier to understand
  - snapshot isolation is a popular feature
    - variants of it are supported by PostgreSQL, MySQL with the InnoDB storage engine, Oracle, SQL Server, and others
    - although the detailed behavior varies from one system to the next


Multiversion concurrency control (MVCC)
- implementation of snpashot isolation typically use **write locks** to prevent dirty writes, however, reads do not require aby locks
- principle (from a performance point of view)
  - **readers never block writers, and writers never block readers**
- high level implementation
  - database potentially keeps several committed versions of a row, because various in-progress transactions may need to see the state of the database at different points in time
  - and because it maintains several versions of a row side by side, this technique is known as **multiversion concurrency control (MVCC)**
- MVCC-based snapshot isolation implementation in PostgreSQL  
![alt text](images/0805.png)
  - when a transaction is started, it is given a unique, always-increasing **transaction ID (txid)**
  - whenever a transaction writes anything to the database, the data it writes is tagged with the transaction ID of the writer
    - to be precise, transaction IDs in PostgreSQL are 32-bit integers, so they overflow after approximately 4 billion transactions
    - the **vacuum** process performs cleanup to ensure that overflow does not affect the data
  - each row in a table has an `inserted_by` field, containing the ID of the transaction that inserted that row into the table
  - each row also has a `deleted_by` field, which is initially empty
    - if a transaction deletes a row, the row isn't removed from the database 
    - instead is marked for deletion by setting the `deleted_by` field to the ID of the transaction that requested the deletion
    - at a later time, when it is certain that no transaction can any longer access the deleted data, a **garbage collecction (GC)** process in the db removes those rows
  - an **update** is internally translated into a delete and an insert
    - e.g., in the example, transaction 13 deducts $100 from account 2, changing the balance from $500 to $400
      - the accounts table now contains two rows for account 2
        - a row with a balance of $500 that was marked as deleted by transaction 13
        - and a row with a balance of $400 that was inserted by transaction 13
  - all the versions of a row are stored within the same database heap, regardless of whether the transactions that wrote them have committed
    - the versions of thes ame row form a linked list, going either from newest version to oldest or the other way round
    - so that queries can internally interate over all versions of a row


Visbility rules for observing a consistent snapshot
- when a transaction reads from the database, transaction IDs are used to decide which row versions it can see and which are invisible
  - by carefully defining visibility rules, the database can present a consistent snapshot of its contents to the application
- rules
  - at the **start** of each transaction, the database makes a **list of all the other transactions that are in progress** (not yet committed or aborted) at that time
    - any **writes** that those transactions have made are **ignored**, even if the transactions subsequently commit
    - this ensures that the application sees a consistent snapshot that is not affected by another transaction committing
  - any **writes** made by transactions with a **later transaction ID** (started after the current transaction started) are **ignored**
  - any **writes** made by **aborted transactions** are **ignored**, regardless of when the abort happened
    - this has the advantage that when a transaction aborts, we don't need to immediately remove the rows it wrote from storage, since the visibility rule filters them out
    - the GC process can remove them later
  - all other writes are visible to the application's queries
- these rules apply to both insertion and deletion of rows
  - e.g., in the example, transaction 12 reads from account 2, it sees a balance of 500 because the deletion of the $500 balance was made by transaction 13
- put another way, a row is **visible** if both of the following conditions are true
  - at the time when the reader's transaction started, the transaction that inserted the row had already committed
  - the row is not marked for deletion, or if it is, the transaction that requested deletion had not yet committed at the time when the reader's transaction started
- by never updating values in place but instead inserting a new version every time a value is changed, the database can provide a consistent snapshot while incurring only a small overhead


Indexes and snapshot isolation
- how do indexes work in a multiversion database?
  - most common approach is that each index entry **points at one of the versions** of a row that matches the entry (either the oldest or the newest version)
    - each row version may contain a reference to the next-oldest or next-newest version
    - a query that uses the index must then iterate over the rows to find one 
      - that is visible and 
      - where the value matches what the query is looking for
    - when GC removes old row versions that are no longer visible to any transaction, the corresponding index entries can also be removed
  - many implementation details affect the **performance** of multiversion concurrency control
    - e.g., PostgreSQL has optimizations for avoiding index updates if different versions of the same row can fit on the same page
    - some other databases avoid storing full copies of modified rows and **store only differences between versions**, to save space
  - another approach is used in CouchDB, Datomic, and LMDB, although they also use B-trees
    - they use an **immutable** (copy-on-write) variant that does not overwrite pages of the tree when they are updated but instead creates a **new copy of each modified page**
      - **parent pages**, up to the root of the tree, are **copied** and updated to point to the new versions of their child pages
      - any **pages** that are **not affected** by a write do not need to be copied and can be **shared** with the new tree
    -  with immutable B-trees, every write transaction (or batch of transactions) **creates a new B-tree root**, and a particular root is **a consistent snapshot** of the database at the point in time when it was created
       -  there is no need to filter out rows based on transaction IDs because subsequent writes cannot modify an existing B-tree; they can only create new tree roots
       -  this approach also requires a backgroud process for compaction and GC


Snapshot isolation, repeatable read, and naming confusion
- MVCC is a commonly used implementation technique for databases, and often it is used to implement snapshot isolation
- however, different databases sometimes **use different terms to refer to the same thing**
  - e.g., snapshot isolation is called "repeatable read" in PostgreSQL and "serializable" in Oracle
- also, sometimes different systems use the **same term but with a different meaning**
  - e.g., "repeatable read" means snapshot isolation in PostgreSQL, but it means an implementation of MVCC with weaker consistency than snapshot isolation in MySQL

---
### Preventing Lost Updates

Motivation
- **read-committed** and **snapshot isolation** levels has primarily focused on guarantees about what a read-only transaction can see in the presence of concurrent writes
- what about **concurrent writes**?
  - dirty writes is one particular type of write-write conflict that can occur
  - several other interesting kinds of conflicts can occur between concurrently writing transactions, such as **lost update**

Lost update
- the problem occur if an application reads a value from the database, modifies it, and writes back the modified value (the read-modify-write cycle)
- if two transactions do this concurrently, one of the modifications can be lost, because the second writes does not include the first modification
- this pattern occurs in various scenarios
  - incrementing a counter or updating an account balance (requires reading the current value, calculating the new value, and writing back the updated value)
  - making a local change to a complex value
    - e.g., adding an element to a list within a JSON document (requires parsing the document, making the change, and writing back the modified document)
  - two users editing a wiki page at the same time, where each user saves their changes by sending the entire page contents to the server, overwriting whatever is currently in the database
- a variety of solutions have been developed
  - atomic write operations
  - explicit locking
  - automatically detecting lost updates
  - conditional writes

Atomic write operations
- atomic update operations removes the need to implement read-modify-write cycles in application code
  - they are usually the best solution if your code can be expressed in terms of these operations
  - for example, the following instruction is concurrency-safe in most relational databases:  
    ```SQL
    UPDATE counters
    SET value = value + 1
    WHERE key = 'foo';
    ```
  - similarly, **document databases** such as MongoDB provide atomic operations for making local modifications to a part of a JSON document
  - Redis provides atomic operations for modifying data structures such as priority queues
- not all writes can easily be expressed in terms of atomic operations
  - e.g., updates to a wiki page involve arbitrary text editing
  - which can be handled using algorithms
- atomic operations are usually implemented by **exclusively locking** the object on the object when it is read so that no other transaction can read it until the update has been applied
  - another option is to simply force all atomic operations to be **executed on a single thread**
- unfortunately, ORM frameworks make it easy to accidentally write code that performs unsafe read-modify-write cycles instead of using atomic operations provided by the database

Explicit locking
- another option for preventing lost updates is for the application to **explicitly lock objects** that are going to be updated
  - then the application can perform a read-modify-write cycle, and if any other transaction tries to concurrently update or lock the same object, it is forced to wait until the first read-modify-write cycle has completed
- consider a multiplayer game in which several players can move the same figure concurrently
  - in this case, an atomic operation may not be sufficient, 
    - because the application also needs to ensure that a player's move abides by the fules of the game, 
    - which **involves some logic that you cannot sensibly implement as a database query**
  - instead, you may use a lock to prevent two players from concurrently moving the same piece  
    ```SQL
    BEGIN TRANSACTION;

    SELECT * FROM figures
      WHERE name = 'robot' AND game_id = 222
      FOR UPDATE;
    
    --Check whether move is valid, then update the position
    --of the piece that was returned by the previous SELECT
    UPDATE figures SET position = 'c4' WHERE id=1234;

    COMMIT;
    ```
      - the `FOR UPDATE` clause indicates that the database should lock all rows returned by this query
  - this works, but to get it right, you need to carefully think about your application logic
    - it is easy to forget to add a necessary lock somewhere in this code and thus introduce a race condition
- locking multiple objects carries a risk of **deadlock**, where two or more transactions are waiting for each other to release their locks
  - many databases automatically detect deadlocks and abort one of the involved transactions so that the system can make progress
  - you can handle this situation at the application level by retrying the aborted transaction

Automatically detecting lost updates
- atomic operations and locks are ways of preventing lost updates by **forcing** the read-modify-write cycles to happen **sequentially**
- an alternative is to allow them to execute **in parallel** and, if the transaction manager detects a lost update, abort the transaction in question and force it to retry its read-modify-write cycle
- an advantage of this approach is that databases can perform this check **efficiently** in conjunction **with snapshot isolation**
  - PostgreSQL's repeatable read, Oracle's serializable, and SQL Server's snapshot isolation levels automatically detect when a lost update has occurred and abort the offending transaction
  - however, MySQL/InnoDB's repeatable read isolation level does not detect lost updates
- a big advantage of lost update detection is that it doesn't require application code to use any special database features
  - you may forget to use a lock or an atomic operation and thus introduce a bug, but lost update detection happens automatically and is thus less error-prone
  - however, you also have to retry aborted transactions at the application level

Conditional writes (compare-and-set)
- in databases that **don't provide transactions**, conditional write operation can prevent lost updates by allowing an update to happen only if the value has not changed since you last read it
  - if the current value does not **match what you previously read**, the update has no effect, and the read-modify-write cycle must be retired
- for example, to prevent two users concurrently updating the same wiki page, you might try something like  
  ```SQL
  -- This may or may not be safe, depending on the database implementation
  UPDATE wiki_pages SET content='new content'
    WHERE id=1234 AND content='old content';
  ```
  - if the content has changed and no longer matches old content, this update will have no effect
- instead of **comparing the full content**, you could also use a **version number** column 
  - you increment on every update and apply the update only if the current version number hasn't changed
  - this approach is sometimes called **optimistic locking**
- note that if another transaction has concurrently modified content, the new content may **not be visible** under the **MVCC visibility rules**
  - many implementations of MVCC have an **exception** to the visibility rules for this scenario
  - where values written by other transactions are visible to the evaluation of the `WHERE` clause of `UPDATE` and `DELETE` queries
  - even though those writes are not otherwise visible in the snapshot

Conflict resolution and replication
- in **replicated databases**, preventing lost updates takes on another dimension
  - because these databases have copies of the data on multiple nodes, and the data can potentially be modified concurrently on different nodes, additional steps need to be taken
- locks and conditional write operations assume that there's a **single up-to-date copy** of the data
  - however, databases with multi-leader or leaderless replication usually allow several writes to happen concurrently and replicate them asynchronously, so they cannot guarantee a single up-to-date copy of the data
  - thus, techniques based on locks or conditional writes do not apply in this context
- a common approach in such replicated database is to **allow** concurrent writes to create several **conflicting versions** of a value (also known as siblings)
  - and to use application code or special data structures to **resolve and merge** these versions after the fact
  - merging conflicting values can prevent lost updates if the updates are **commutative**
    - i.e., you can apply them in a different order on different replicas and stil get the same result

---
### Write Skew and Phantoms
