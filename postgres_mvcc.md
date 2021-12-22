
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

* In Postgres Read Uncommitted acts like Read Commited
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
For example with query in Read Committed level will has Nonrepeatable Read anomaly
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
|  /* UPDATE is blocked by 1st transation UPDATE */
|  => UPDATE accounts SET amount = amount * 1.01
|  WHERE client IN (
|  /* But SELECT isn't blocked and 
    returns rows versions without application UPDATE from 1st transaction 
    therefore bob is valid client */
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
 * False-positive errors depending on indexes, memory and moons phase
 * Parallel query execution plan is unavailable
 * unable to use Serializable and other isolation levels in other transactions. Serializable level will be silently downgraded to less strict level if it using.
 For example both transactions will be executed with REPEATABLE READ level.
 ``` sql
 => BEGIN ISOLATION LEVEL REPEATABLE READ;
 ... some query
 |   => BEGIN ISOLATION LEVEL SERIALIZABLE;
 |   ... some query
 ```
---
## Example of using Repeatable read level
 * check logical consistency in DB through 2 queries
 --placeholder for example--

---
# How PG stores data
## Layers
 * main - with data rows
 * init - using for UNLOGGED tables
 * fsm - free space map
 * vm (visibility map) - bitmap of pages with only actual rows
## Pages
 * Default size is 8Kb
 * Data from pages reads to buffers "as is" therefore we can't transfer files for DB between different architectures:
     * bytes ordering in x86 and ARM is different
     * align in 32-bit and 64-bit systems is different (4 and 8 byte respectively)
---
## TOAST
Oversized Attributes Storage Technique
 * PG tries to put at least 4 rows in page (if page size is 8K row with headers should be less 2040 bytes)
 * Storage strategies by types:

| Strategy | Types   | Description |
| -------- | ------- | ----------- |
| plain    | integer | Don't use TOAST - store values in table |
| extended |text, jsonb| Stores in TOAST in zipped form |
| external |text, jsonb| Stores in TOAST in unzipped form |
| main     | numeric | Try to zip value if it don't help store in TOAST |

If we knew that zipping is useless for some field we can define storage as external.
``` sql
ALTER TABLE accounts ALTER COLUMN number SET STORAGE external;
```
 * Values in TOAST table are stored in N-rows (data is sliced by 2000 bytes)
 * Postgres is NOT good BD for storing overweight objects

### Indexes for big-size fields
 * Values over 1/4 page size can't be stored in B-Tree indexes (PG11)
 * For JSONB should use GIN index by keys. GIN has operators to hash big data

---
# Rows Versions

## Headers of row version
 - t_xmin - transaction id which adds the row
 - t_xmax - transaction id which delete the row
 - t_infomask - properties of verion (commited, aborted)
 - t_ctid (page num, pointer_idx in page) - point to actual version of row
 - t_bits - bit-mask that show which fields have NULL value. Yes NULL values don't store in fields
 - t_cmin - counter which using for CURSORs consistency

Simple way to get `xmin` and `xmax`.
``` sql
SELECT xmin, xmax, * FROM t;
```

## Transaction statuses
 * All active transactions are in shared memory in structure `ProcArray`.
 * Statuses (committed, aborted) of finished transactions are storing in XACT (CLOG for PG versions before 10).
 List of all transactions is in PGDATA/pg_xact, but several last pages are stored in buffers.

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

Command `COMMIT` or `ROLLBACK` **don't change rows** - only add record to XACT with transaction status.
It is an optimization that helps don't load all changed rows and change their headers.
**Important!** `COMMIT` doesn't change rows headers!

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
1. Find row version with `xmax` aborted `(a)` and `xmin` with flag commited `(c)` where `t_ctid` point to itself or `xmin == current transaction`
2. Find with the same conditions but `xmin` doesn't have committed flag:
    * skip rows from other active transactions (check in structure `ProcArray`)
    * check `xmin` is in finished transactions (from XACT)
    * **update headers** for rows versions and set corresponding flags: committed, aborted

## DELETE and ROLLBACK
`DELETE` set `xmax` to current transaction (that locks row for updating from other transactions) and unset aborting flag `(a)`.
``` sql
BEGIN;
DELETE FROM t;

SELECT * FROM heap_page('t',0);
 ctid  | state  |   xmin   | xmax | t_ctid 
-------+--------+----------+------+--------
 (0,1) | normal | 3664 (c) | 3665 | (0,1)
 
 ROLLBACK;
```
`ROLLBACK` as `COMMIT` doesn't change headers, but `SELECT` do that.
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
`SAVEPOINT` uses an ordinary transaction mechanism and each `SAVEPOINT` has its own transaction id and save in XACT.

## Indexes
Indexes hold all versions of row which exist in table.
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
 * With Repeatable Read and Sirializable - on 1st query in transaction

On creating snapshot statuses of active transactions (from ProcArray) are stored in the snapshot. That means that each snapshot should get with **blocking** ProcArray for a copy. That's why sometimes even with a powerful machine we can see performance downgrading and a low level of using resources cause of ProcArray blocking. It is one of the reasons why hundreds of active connections are a bad idea.

 - snapshot.xmax - last transaction + 1
 - snapshot.xip - list of active transactions
 - snapshot.xmin - the earlies of active transactions

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

**We should try to use only short-living transactions**, but if we can not rule this on application side we can use settings:
 - old_snapshot_threshold
 - idle_in_transaction_session_timeout
**Another good advice**: bad idea is combine OLTP and OLAP approaches in Postgres because long analitics query will block `VACUUM`ing rows for fast updating rows.

---
## Export snapshot
Sometimes concurrent transactions should see the same DB snapshot (for example `pg_dump` utility). It is available export and apply particular snapshot for transaction.
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
 
 `fillfactor` percentage of page size after which the next `INSERT` will add row in next page. And the reminder (100% - fillfactor) will be used for `UPDATE`s of current rows.
By default `fillfactor` for tables = 100 and for indexes = 90.

**Best practice**: If you are going to insert rows without updates it is a good idea to set `fillfactor=100` for index. For example for Materialised Views (--placeholder for example --.

In-page vacuuming executes in 2 cases:
 * when previous `UPDATE` could not find space in current page and marked current page for cleaning and inserted in next page. Page is vacuuming in next access
 * when fillfactor is achieved vacuuming executing right now

---
## Indexes
In-page vacuuming cleans only rows versions (before event horizon) but doesn't clean in-page pointers. Because index is placed on other pages and can have links to pointers inside page. Whereas rows are being moved to higher addresses and marked as `dead` and actual rows are moving in lower addresses.

**Record in indexes** page (which point to dead row) will be **corrected at the next access** (SELECT or UPDATE).

---
### HOT updates optimizations
HOT optimization will be applied when we update field which doesn't mention in index.

After updating fields that don't participate in index, index page won't have additional links to new row versions. Instead, index will point to 1st version of row in the table and have to jump to actual version using `t_ctid`.
New link will be added to index page only in case when new version of row doesn't fit on page and has to be added to the next page. In this case, index will have 2 links to versions of row.
 - Row which holds link from index is marked as HHU (**Heap Hot Updated**).
 - Row which doesn't hold this link is marked as HOT (**Heap Hot Updated**).

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
**Best practice**: To reduce amount of in-page vacuuming while updating non-indexing fields recommended increasing `fillfactor`.

---
# VACUUM

Vacuum blocks only change DLL queries, like `CREATE INDEX` and `CREATE TABLE`.

**`VACUUM`** remove links to non-actual rows versions from indexes and mark rows versions in table as `unused`. But doesn't compact rows from different pages. Therefore:
- used space for tables and indexes doesn't change
- a percentage of useful information on pages is decreasing
- index scan and table full-scan has to load to buffer the same amount of pages

Additionally `VACUUM` runs only on pages that don't true in invisibility map and after running updates visibility map (using in index-only scan).

If all removing rows versions ids list don't fil in `maintenance_work_mem` `VACUUM` has to rescan indexes again. It is happening when enormous amount rows are updated or deleted.

## VACUUM FULL
`VACUUM FULL` is repacking table and indexes but gets table lock.
It can be a good idea to run `VACUUM FULL` (if you can afford significant downtime) after a big amount of deleting or updating.
`CLUSTER` is very similar but additionally places rows in special order useful for indexes.
`REINDEX` is recreating index (it is used in `VACUUM FULL`).
If we don't want to lock table for all query time we can use [pg_repack](https://github.com/reorg/pg_repack)

`TRUNCATE` create a new file and delete the old one.

---
## AUTOVACUUM

Rate of running `AUTOVACUUM` is defined by settings:
- autovacuum_vacuum_threshold - amount of dead row versions. default = 200.
- autovacuum_vacuum_scale_factor - fraction dead row versions in table. default = 0.2 (20%)

 # Transaction id wraparound
 
 Transaction id is stored in 32 bits.
 
 
