
# Isolation levels in Postgres

---

| Isolation Level | Dirty Read | Nonrepeatable Read | Phantom Read | Serialization Anomaly |
| --------------- | ---------- | ------------------ | ------------ | --------------------- |
| Read uncommitted | - (+) | + | +     | + |
| Read Committed  | -      | + | +     | + |
| Repeatable read | -      | - | - (+) | + |
| Serializable    | -      | - | -     | - |

---
## Read Committed

* In Postgres Read Uncommitted acts like Read Committed
* Select query returns data at the beginning of execution
``` sql
=> SELECT amount, pg_sleep(2) FROM accounts WHERE client = 'bob';
|  => BEGIN;
|  => UPDATE accounts SET amount = amount + 100 WHERE id = 2;
|  => UPDATE accounts SET amount = amount - 100 WHERE id = 3;
|  => COMMIT;
```
``` sql
 amount  | pg_sleep 
---------+----------
    0.00 | 
 1000.00 | 
(2 rows)
```
* Single query (whatever operators, CTE, etc wouldn't use in query) executes consistently.
---
### Examples inconsistency
* Concurrent executing UPDATE and UPDATE with SELECT. 
For example, with a query in Read Committed level will have Nonrepeatable Read anomaly
``` sql
=> SELECT * FROM accounts WHERE client = 'bob';
 id | number | client | amount 
----+--------+--------+--------
  2 | 2001   | bob    | 200.00
  3 | 2002   | bob    | 800.00
(2 rows)
```
``` sql
=> BEGIN;
=> UPDATE accounts SET amount = amount - 100 WHERE id = 3;
|  /* UPDATE is blocked by 1st transaction UPDATE */
|  => UPDATE accounts SET amount = amount * 1.01
|  WHERE client IN (
|  /* But SELECT isn't blocked and 
    returns rows versions without application UPDATE from 1st transaction 
    therefore bob is a valid client */
|    SELECT client
|    FROM accounts
|    GROUP BY client
|    HAVING sum(amount) >= 1000
|  );
=> COMMIT;
```
``` sql
=> SELECT * FROM accounts WHERE client = 'bob';
id | number | client |  amount  
----+--------+--------+----------
  2 | 2001   | bob    | 202.0000
  3 | 2002   | bob    | 707.0000
(2 rows)
```
* Functions with category *VOLATILE* (category by default) will act unexpectedly
``` sql
=> CREATE FUNCTION get_amount(id integer) RETURNS numeric AS $$
  SELECT amount FROM accounts a WHERE a.id = get_amount.id;
$$ VOLATILE LANGUAGE sql;

=> SELECT get_amount(id), pg_sleep(2)
FROM accounts WHERE client = 'bob';
|  => BEGIN;
|  => UPDATE accounts SET amount = amount + 100 WHERE id = 2;
|  => UPDATE accounts SET amount = amount - 100 WHERE id = 3;
|  => COMMIT;
```
``` sql
 get_amount | pg_sleep 
------------+----------
     100.00 | 
     800.00 | 
(2 rows)
```
 * Concurrent UPDATE and DELETE
``` sql
SELECT hits from website;
 hits |
------
    9 | 
   10 | 
(2 rows)
```
``` sql
=> BEGIN;
=> UPDATE website SET hits = hits + 1;
|  => DELETE FROM website WHERE hits = 10;
=> COMMIT;
```
The `DELETE` will have no effect even though there is a `website.hits = 10` row before and after the `UPDATE`. This occurs because the pre-update row value 9 is skipped, and when the `UPDATE` completes and `DELETE` obtains a lock, the new row value is no longer 10 but 11, which no longer matches the criteria.


---
### How we can deal with it?
- Add constraints
- Use 1 SQL-operator
- Use locks

---
## Repeatable Read and Serializable
 * On the same queries, we get an access error
`|  ERROR:  could not serialize access due to concurrent update`
## Serializable
 * False-positive errors depending on indexes, memory, and moons phase
 * Parallel query execution plan is unavailable
 * unable to use Serializable and other isolation levels in other transactions. Serializable level will be silently downgraded to a less strict level if it is used.
 For example, both transactions will be executed with REPEATABLE READ level.
 ``` sql
 => BEGIN ISOLATION LEVEL REPEATABLE READ;
 ... some query
 |   => BEGIN ISOLATION LEVEL SERIALIZABLE;
 |   ... some query
 ```
---
## Example of using Repeatable read level
 * check logical consistency in DB through 2 queries
 

---
# How PG stores data
## Layers
 * main - with data rows
 * init - using for UNLOGGED tables
 * fsm - free space map
 * vm (visibility map) - the bitmap tracks those pages that contain pretty old tuples, which are visible in all data snapshots for sure
## Pages
 * Default size is 8Kb
 * Data from pages reads to buffers "as is" therefore we can't transfer files for DB between different architectures:
     * bytes ordering in x86 and ARM is different
     * align in 32-bit and 64-bit systems is different (4 and 8 bytes respectively)
``` sql
       0  +-----------------------------------+
          | header                            |
      24  +-----------------------------------+
          | array of pointers to row versions |
   lower  +-----------------------------------+
          | free space                        |
   upper  +-----------------------------------+
          | row versions                      |
 special  +-----------------------------------+
          | special space                     |
pagesize  +-----------------------------------+
```
---
## TOAST
Oversized Attributes Storage Technique
 * PG tries to put at least 4 rows on the page (if the page size is 8K row with headers should be less than 2040 bytes)
 * Storage strategies by types:

| Strategy | Types   | Description |
| -------- | ------- | ----------- |
| plain    | integer | Don't use TOAST - store values in table |
| extended |text, jsonb| Stores in TOAST in zipped form |
| external |text, jsonb| Stores in TOAST in unzipped form |
| main     | numeric | Try to zip value if it doesn't help store in TOAST |

If we knew that zipping is useless for some field we can define storage as external.
``` sql
ALTER TABLE accounts ALTER COLUMN number SET STORAGE external;
```
 * Values in TOAST table are stored in N-rows (data is sliced by 2000 bytes)
 * Postgres is NOT a good BD for storing overweight objects

### Indexes for big-size fields
 * Values over 1/4 page size can't be stored in B-Tree indexes (PG11)
 * For JSONB should use GIN index by keys. GIN has operators to hash big data

---
# Rows Versions

## Headers of row version
 - t_xmin - transaction id which adds the row
 - t_xmax - transaction id which deletes the row
 - t_infomask - properties of version (committed, aborted)
 - t_ctid (page num, pointer_idx in page) - point to the actual version of the row
 - t_bits - bit-mask that shows which fields have NULL value. Yes NULL values don't store in fields
 - t_cmin - counter which is used for CURSORs consistency

The simple way to get `xmin` and `xmax`.
``` sql
SELECT xmin, xmax, * FROM t;
```

## Transaction statuses
 * All active transactions are in shared memory in the structure `ProcArray`.
 * Statuses (committed, aborted) of finished transactions are stored in XACT (CLOG for PG versions before 10).
 The list of all transactions is in PGDATA/pg_xact, but several last pages are stored in buffers.

## What is happening inside before and after transaction
``` sql
CREATE FUNCTION heap_page(relname text, pageno_from integer, pageno_to integer)
RETURNS TABLE(ctid tid, state text, xmin text, xmin_age integer, xmax text, t_ctid tid)
AS $$
SELECT (pageno,lp)::text::tid AS ctid,
       CASE lp_flags
         WHEN 0 THEN 'unused'
         WHEN 1 THEN 'normal'
         WHEN 2 THEN 'redirect to '||lp_off
         WHEN 3 THEN 'dead'
       END AS state,
       t_xmin || CASE
         WHEN (t_infomask & 256+512) = 256+512 THEN ' (f)'
         WHEN (t_infomask & 256) > 0 THEN ' (c)'
         WHEN (t_infomask & 512) > 0 THEN ' (a)'
         ELSE ''
       END AS xmin,
      age(t_xmin) xmin_age,
       t_xmax || CASE
         WHEN (t_infomask & 1024) > 0 THEN ' (c)'
         WHEN (t_infomask & 2048) > 0 THEN ' (a)'
         ELSE ''
       END AS xmax,
       t_ctid
FROM generate_series(pageno_from, pageno_to) p(pageno),
     heap_page_items(get_raw_page(relname, pageno))
ORDER BY pageno, lp;
$$ LANGUAGE SQL;
```
Create row in transaction
``` sql
BEGIN;
INSERT INTO t (1, 'FOO');
```
``` sql
SELECT * FROM heap_page('t',0);
 ctid  | state  | xmin | xmax  | t_ctid 
-------+--------+------+-------+--------
 (0,1) | normal | 3664 | 0 (a) | (0,1)
(1 row)
```
``` sql;
COMMIT;
```
Query `SELECT * FROM heap_page('t',0);` returns the same result as before `COMMIT`.

Command `COMMIT` or `ROLLBACK` **don't change rows** - only add a record to XACT with transaction status.
It is an optimization that helps don't load all changed rows and change their headers.
**Important!** `COMMIT` doesn't change row headers!

## SELECT changes rows headers!
``` sql
SELECT * FROM t;
```
``` sql
SELECT * FROM heap_page('t',0);

 ctid  | state  |   xmin   | xmax  | t_ctid 
-------+--------+----------+-------+--------
 (0,1) | normal | 3664 (c) | 0 (a) | (0,1)
(1 row)
```
### What does SELECT really do?
> Isolation levels change `SELECT` behavior. Here we consider `READ COMMITTED`.
1. Find row version with `xmax` aborted `(a)` and `xmin` with flag committed `(c)` where `t_ctid` point to itself or `xmin == current transaction`
2. Find with the same conditions but `xmin` doesn't have a committed flag:
    * skip rows from other active transactions (check in structure `ProcArray`)
    * check `xmin` is in finished transactions (from XACT)
    * **update headers** for rows versions and set corresponding flags: committed, aborted

## DELETE and ROLLBACK
`DELETE` sets `xmax` to the current transaction (that locks the row for updating from other transactions) and unsets aborting flag `(a)`.
``` sql
BEGIN;
DELETE FROM t;

SELECT * FROM heap_page('t',0);
 ctid  | state  |   xmin   | xmax | t_ctid 
-------+--------+----------+------+--------
 (0,1) | normal | 3664 (c) | 3665 | (0,1)
 
 ROLLBACK;
```
`ROLLBACK` as `COMMIT` doesn't change headers, but `SELECT` does that.
``` sql
SELECT * FROM t;
...

SELECT * FROM heap_page('t',0);
 ctid  | state  |   xmin   |   xmax   | t_ctid 
-------+--------+----------+----------+--------
 (0,1) | normal | 3664 (c) | 3665 (a) | (0,1)
```

## UPDATE is DELETE and INSERT
``` sql
BEGIN;
UPDATE t SET s = 'BAR';

SELECT * FROM heap_page('t',0);
 ctid  | state  |   xmin   | xmax  | t_ctid 
-------+--------+----------+-------+--------
 (0,1) | normal | 3664 (c) | 3666  | (0,2)
 (0,2) | normal | 3666     | 0 (a) | (0,2)
```

## CURSOR
After `CURSOR` declaration all new rows in all transactions start incrementing `cmin` row-header in order to CURSOR can return only rows versions on its (cursor) declaration.
``` sql
=> SELECT xmin, cmin, * FROM accounts;
 xmin | cmin | id | number | client  | amount  
------+------+----+--------+---------+---------
 3697 |    0 |  3 | 2002   | bob     |  900.00
 3698 |    0 |  4 | 3001   | charlie |  100.00
 
|  => DECLARE c CURSOR FOR SELECT count(*) FROM accounts;
 
=> INSERT INTO accounts(id, number, client, amount) VALUES (5, 3002, 'charlie', 200.00);
=> SELECT xmin, CASE WHEN xmin = 3698 THEN cmin END cmin, * FROM accounts;
 xmin | cmin | id | number | client  | amount  
------+------+----+--------+---------+---------
 3697 |    0 |  3 | 2002   | bob     |  900.00
 3698 |    0 |  4 | 3001   | charlie |  100.00
 3698 |    1 |  5 | 3002   | charlie |  200.00
 
|  => FETCH c;
 count 
-------
     2
```

## SAVEPOINTs
`SAVEPOINT` uses an ordinary transaction mechanism and each `SAVEPOINT` has its own transaction id and is saved in XACT.

## Indexes
Indexes hold all versions of row which exist in the table.
``` sql
SELECT itemoffset, ctid FROM bt_page_items('t_s_idx',1);
 itemoffset | ctid  
------------+-------
          1 | (0,2)
          2 | (0,1)
```
---
# Snapshots
 * With Read Committed snapshots create on every query
 * With Repeatable Read and Sirializable - on 1st query in a transaction

On creating a snapshot statuses of active transactions (from ProcArray) are stored in the snapshot. That means that each snapshot should get with **blocking** ProcArray for a copy. That's why sometimes even with a powerful machine we can see performance downgrading and a low level of using resources cause of ProcArray blocking. It is one of the reasons why hundreds of active connections are a bad idea.

 - snapshot.xmax - last transaction + 1
 - snapshot.xip - list of active transactions
 - snapshot.xmin - the earliest of active transactions

We can get this information by pattern `xmax:xmin:xip`
``` sql
=> SELECT txid_current_snapshot();
 txid_current_snapshot 
-----------------------
    3695:3697:3695
```
---
## Event horizon
![Event horizon](https://postgrespro.com/media/2020/12/04/snapshots8-en.png)

Rows with xid < snapshot.xmin can be vacuumed. Therefore long-living transactions are blocking vacuuming old rows versions.
Event horizon is shared across all transactions.

**We should try to use only short-living transactions**, but if we can not rule this on the application side we can use settings:
 - old_snapshot_threshold
 - idle_in_transaction_session_timeout
**Another good advice**: A bad idea is combining OLTP and OLAP approaches in Postgres because long analytics queries will block `VACUUM`ing rows for fast updating rows.

---
## Export snapshot
Sometimes concurrent transactions should see the same DB snapshot (for example `pg_dump` utility). It is available to export and apply a particular snapshot for a transaction.
``` sql
=> BEGIN ISOLATION LEVEL REPEATABLE READ;
=> SELECT pg_export_snapshot();
 pg_export_snapshot  
---------------------
 00000004-00000E7B-1
 
|  => BEGIN ISOLATION LEVEL REPEATABLE READ;
|  => SET TRANSACTION SNAPSHOT '00000004-00000E7B-1';
 ```
 ---
# In-page vacuum
 
 `fillfactor` percentage of page size after which the next `INSERT` will add a row on the next page. And the reminder (100% - fillfactor) will be used for `UPDATE`s of current rows.
By default `fillfactor` for tables = 100 and for indexes = 90.

**Best practice**: If you are going to insert rows without updates it is a good idea to set `fillfactor=100` for index. For example for Materialised Views .

In-page vacuuming executes in 2 cases:
 * when the previous `UPDATE` could not find space on the current page and mark the current page for cleaning and insert in the next page. Page is vacuuming in the next access
 * when fillfactor is achieved vacuuming executing right now

---
## Indexes
In-page vacuuming cleans only rows versions (before event horizon) but doesn't clean in-page pointers. Because the index is placed on other pages and can have links to pointers inside the page. Whereas rows are being moved to higher addresses and marked as `dead` and actual rows are moving to lower addresses.

**Record in indexes** page (which points to dead row) will be **corrected at the next access** (SELECT or UPDATE).

---
### HOT updates optimizations
HOT optimization will be applied when we update a field which doesn't mentioned in the index.

After updating fields that don't participate in the index, the index page won't have additional links to new row versions. Instead, the index will point to 1st version of the row in the table and have to jump to the actual version using `t_ctid`.
The new link will be added to the index page only in case when the new version of the row doesn't fit on the page and has to be added to the next page. In this case, the index will have 2 links to versions of the row.
 - Row which holds the link from the index is marked as HHU (**Heap Hot Updated**).
 - Row which doesn't hold this link is marked as HOT (**Heap-Only Tuple**).

``` sql
SELECT * FROM heap_page('hot',0);
 ctid  |     state     |   xmin   |   xmax   | hhu | hot | t_ctid 
-------+---------------+----------+----------+-----+-----+--------
 (0,1) | redirect to 2 |          |          |     |     | 
 (0,2) | normal        | 3993 (c) | 3994 (c) | t   | t   | (0,3)
 (0,3) | normal        | 3994 (c) | 3995 (c) | t   | t   | (0,4)
 (0,4) | normal        | 3995 (c) | 3996 (c) | t   | t   | (0,5)
 (0,5) | normal        | 3996 (c) | 3997     |     | t   | (1,1)
 
 SELECT * FROM index_page('hot_id',1);
 itemoffset | ctid  
------------+-------
          1 | (1,1)
          2 | (0,1)
 ```
**Best practice**: To reduce the amount of in-page vacuuming while updating non-indexing fields recommended increasing `fillfactor`.

---
# VACUUM

Vacuum blocks only change DLL queries, like `CREATE INDEX` and `CREATE TABLE`.

**`VACUUM`** remove links to non-actual row versions from indexes and mark row versions in the table as `unused`. But doesn't compact rows from different pages. Therefore:
- used space for tables and indexes doesn't change
- a percentage of useful information on pages is decreasing
- index scan and table full-scan has to load to buffer the same amount of pages

Additionally `VACUUM` runs only on pages that aren't presented in the visibility map and after running updates visibility map (using in index-only scan).

If all removing rows versions ids list don't fil in `maintenance_work_mem` `VACUUM` has to rescan indexes again. It happens when an enormous amount rows are updated or deleted.

## VACUUM FULL
`VACUUM FULL` is repacking table and indexes but gets table lock.
It can be a good idea to run `VACUUM FULL` (if you can afford significant downtime) after a big amount of deleting or updating.
`CLUSTER` is very similar but additionally places rows in special order useful for indexes.
`REINDEX` is a recreating index (it is used in `VACUUM FULL`).
If we don't want to lock the table for all query time we can use [pg_repack](https://github.com/reorg/pg_repack)

`TRUNCATE` creates a new file and deletes the old one.

---
## AUTOVACUUM

The rate of running `AUTOVACUUM` is defined by the settings:
- autovacuum_vacuum_threshold - amount of dead row versions. default = 200.
- autovacuum_vacuum_scale_factor - fraction dead row versions in the table. default = 0.2 (20%)

### Append-only tables
Any table used exclusively in append-only mode will not be vacuumed and therefore, the visibility map won't be updated for it. But this makes use of index-only scan impossible. Actually, vaccuming will be eventually run but very rarely. (That's no more the case since PostgreSQL 13. Now autovacuum also takes insertions into account.)



# Transaction id wraparound
 
 Transaction id is stored in 32 bits. This is a pretty large number (about 4 billion), but with the intensive work of the server, this number is unlikely to get exhausted. For example: with a workload of 1000 transactions a second, this will happen as early as one month and a half of continuous work.
 Postgres has the mechanism of freezing rows and even table pages. That allows the reuse of transaction ids.
 ![Transactions wraparound](https://postgrespro.com/media/2021/03/19/freeze4-en.png)
 
### Row freezing
 The row is described as frozen if both hint bits `committed` and `aborted` are set. If a row is marked as frozen, `xmin` does not mean anything.
 Here Postgres introduces the age of a row (instead of using transation id) which is calculated from the last frozen row tid.
 Rows freezing is provided by `VACUUM` based on settings `vacuum_freeze_min_age`, which defines the minimum age of the `xmin` transaction for which a tuple can be frozen.

### Table pages freezing
 The vacuuming looks only through pages not tracked in the visibility map. Remind that pages are placed in the visibility map when the page contains only the actual version of rows that are visible in all transactions.
 To freeze the tuples left on pages (presented in the visibility map) where vacuuming does not normally look, the second parameter is provided: `vacuum_freeze_table_age`. It defines the transaction age for which vacuuming ignores the visibility map and looks through all the table pages to freeze.
 Each page stores the transaction ID for which all the older transactions are known to be frozen for sure (all-frozen bit `pg_class.relfrozenxid`). And this is the age of this stored transaction that the value of the `vacuum_freeze_table_age` parameter is compared to.
 Thanks to all-frozen vacuuming has not to go through all pages but only for which the bit is not set yet.
 Anyway, all table pages get frozen once every (`vacuum_freeze_table_age` − `vacuum_freeze_min_age`) transactions.
 
#### Aggressive freezing
 VACUUM usually does not visit append-only tables because those tables do not have dead tuples. But VACUUM will be eventually executed when the age of the oldest unfrozen row has achieved the value of parameter `autovacuum_freeze_max_age`. By default `autovacuum_freeze_max_age` = 2 billion transactions, but we can change it for a table
``` sql
ALTER TABLE tfreeze SET (autovacuum_freeze_max_age = 1000000, fillfactor = 100);
```
