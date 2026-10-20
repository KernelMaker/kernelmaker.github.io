---
layout: post
title: MySQL vs PostgreSQL Internals (Part 3) -- Row Locking
---

As the third post in the MySQL vs PostgreSQL series, this post compares their row locks. Row locks are involved throughout transaction execution, and there is a lot to cover. It is not possible to discuss everything in a single post. This post starts with a specific query and compares how the two systems design and implement its execution.

## The Question

Consider the following table:

```sql
CREATE TABLE t (id int PRIMARY KEY, v int);
```

Insert 100,000 rows into both PostgreSQL and MySQL, with `id` ranging from 1 to 100000 and arbitrary values for `v` (for example, `v = id * 7919 % 1000`). Then execute:

```sql
SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;
```

How do the two systems execute this statement, and what are the differences?

## 1. Overall Execution Flow

### PostgreSQL

PostgreSQL generates the following execution plan for this statement:

```
LockRows
  ->  Sort
        Sort Key: t.v
        ->  Index Scan using t_pkey on t
              Index Cond: (t.id > 99990)
```

The executor pulls rows from the top node one at a time: whenever an upper node needs a row, it requests one from the node below. Execution of this statement has three stages:

1. **Scan**: When LockRows first requests a row from Sort, Sort needs to collect all its input first, so it repeatedly requests rows from Index Scan. Index Scan finds each index entry satisfying `id > 99990` in the primary-key B+Tree index `t_pkey`, then uses its TID to fetch the version of the row visible to the current snapshot from the heap, producing `(id, v, ctid)`. Sort copies each row into its own memory. No row locks are taken at this stage.

2. **Sort**: After Index Scan has returned all rows, Sort sorts them by `v`. No row locks are taken at this stage either.

3. **Lock and return**: LockRows then fetches the next row from Sort, uses its `ctid` to access the heap again, locks the row, and returns it to the client. This continues until all rows from Sort have been consumed.

In other words, PostgreSQL does not lock rows during scanning or sorting. It locks them one by one only after sorting, in the order of `v`.

The following figure shows these three stages.

<img src="/public/images/2026-10-20/1.png" alt="image-1"/>

### MySQL

MySQL generates the following execution plan for this statement:

```
-> Sort: t.v
    -> Filter: (t.id > 99990)
        -> Index range scan on t using PRIMARY over (99990 < id)
```

MySQL's executor also pulls rows one at a time: whenever an upper iterator needs a row, it calls `Read()` on the iterator below. Unlike PostgreSQL, there is no dedicated locking node in the execution plan. Locking happens when InnoDB reads the data. Execution of this statement has two stages:

1. **Scan and lock**: When SortingIterator is called for the first time, it needs to collect all its input, so it repeatedly calls IndexRangeScanIterator through FilterIterator to read rows. Each read goes through the handler interface into InnoDB's `row_search_mvcc`: it locates the next record satisfying `id > 99990` in the primary key (clustered index), locks that record, and then copies its latest version to the upper layer. Locking reads do not use a snapshot (read view); they read the latest committed version. If another transaction has locked the record, execution waits here. Under the default REPEATABLE READ isolation level, InnoDB takes a next-key lock (covering the record itself and the gap before it), and when the scan reaches the end of the index, it also locks the supremum pseudo-record. FilterIterator checks `id > 99990` again for each returned row. Since the range scan has already ensured this condition, every row passes the filter. filesort copies each row into the sort buffer.

2. **Sort and return**: Once scanning is complete, filesort sorts the rows by `v`. SortingIterator then returns the next row from the sort buffer to the client each time, without accessing InnoDB again. All these rows were already locked in stage 1.

In other words, MySQL locks rows as it scans them. By the time it sorts and returns the rows, they are already locked. The locking order follows `id`.

The following figure shows these two stages.

<img src="/public/images/2026-10-20/2.png" alt="image-2"/>

### Comparison

Looking at the overall flow, PostgreSQL scans first, sorts next, and locks last, while MySQL locks during scanning and then sorts. This leads to several differences: PostgreSQL locks each row after sorting and before returning it, so the locking order follows `v`; MySQL locks rows during scanning, so the locking order follows `id`. PostgreSQL reads the versions visible to its snapshot during scanning, while MySQL reads the latest committed versions. During sorting, PostgreSQL has not yet locked these rows, while MySQL has already locked them.

## 2. Index Scan

Scanning is the first step in both execution flows, but their internal implementations differ.

### PostgreSQL

The execution function for the Index Scan node is `ExecIndexScan`. Like other scan nodes, it is driven by the general scan loop `ExecScanExtended`, while `IndexNext` implements the actual fetching of the next row. Each row is fetched in two steps:

1. Scan the B+Tree index. For each index tuple that satisfies the condition, use its TID to fetch the corresponding heap tuple from the heap and check its visibility against the snapshot. If it is not visible, continue to the next index tuple; if it is visible, proceed to the next step.
2. Build the result row from the columns required by the query, and append the heap tuple's TID as the `ctid` column. This is necessary because the row is not locked during scanning. When the row reaches the upper LockRows node, LockRows will use this TID to access the page containing the row again and lock it (by then, another transaction may have modified the row, as discussed later).

The following figure shows this flow:

<img src="/public/images/2026-10-20/3.png" alt="image-3"/>

If we look at this scan from the data perspective, it looks like the following:

<img src="/public/images/2026-10-20/4.png" alt="image-4"/>

### MySQL

The `Read()` method of IndexRangeScanIterator enters InnoDB through the handler interface: the first call goes through `read_range_first` to `ha_innobase::index_read`, and subsequent calls go through `read_range_next` to `ha_innobase::index_next`. Both eventually call `row_search_mvcc`. This example scans the primary key, which is the clustered index. Each call to `row_search_mvcc` involves three steps:

1. Move the cursor to the next record in the clustered index. The leaf nodes of an InnoDB clustered index store the entire row, so there is no need to use a TID to fetch the row from a separate structure, as PostgreSQL does.
2. Lock the record. If another transaction has locked it, wait here. After acquiring the lock, if the record is found to be delete-marked, skip it and continue to the next record.
3. Copy the latest version of the record into `table->record[0]` and return it to the upper layer. Locking reads do not use a snapshot, so there is no visibility check here. Since the record was already locked during scanning, the returned row does not need to carry its location as PostgreSQL's does.

When the scan reaches the end of the range (the end of the index in this example), under REPEATABLE READ it also locks the supremum pseudo-record, then returns `HA_ERR_END_OF_FILE` to end the scan.

The following figure shows this flow:

<img src="/public/images/2026-10-20/5.png" alt="image-5"/>

If we look at this scan from the data perspective, it looks like the following:

<img src="/public/images/2026-10-20/6.png" alt="image-6"/>

### Comparison

There are two main differences in index scanning. First, MySQL can directly obtain the latest version of the entire row from the clustered index. PostgreSQL first finds a TID in the B+Tree index, then fetches the row from the heap and checks its visibility against the snapshot. Second, MySQL locks a record immediately after finding it and waits if another transaction has locked it. PostgreSQL does not lock rows during scanning. Instead, it builds the result row, attaches `ctid`, and leaves locking to the upper LockRows node.

## 3. Locking Flow

### PostgreSQL

This is one of the biggest differences between the two systems. In PostgreSQL, scanning and locking happen at different stages: a row read during scanning may have been locked, updated, or deleted by another transaction by the time it is locked. Therefore, every time LockRows fetches a row from Sort, it must check the row's current state before deciding what to do: wait for another transaction to finish, lock the updated version, or skip the row. The following figure shows this process:

<img src="/public/images/2026-10-20/7.png" alt="image-7"/>

`table_tuple_lock` is an interface of the table access method. For heap tables, it is implemented by `heapam_tuple_lock`, while the actual locking is performed by `heap_lock_tuple`. `heap_lock_tuple` locates the row using `ctid`, checks its current state, and handles the following cases:

1. **No conflict**: No other transaction holds a conflicting lock, and the row has not been modified. Lock it directly (the locking implementation is discussed in the next section).
2. **Another transaction holds a conflicting lock or is modifying the row**: Wait for that transaction to finish, then check again.
3. **The row has been updated by a committed transaction**: Under READ COMMITTED, `heapam_tuple_lock` follows the update chain to find and lock the latest version, and marks it as traversed. Under REPEATABLE READ, it directly returns `TM_Updated`.
4. **The row has been deleted by a committed transaction**: Return `TM_Deleted`.

LockRows decides how to handle the row based on the return value:

- `TM_Ok`, not traversed: The locked version is the same one read during scanning, so return the row directly.
- `TM_Ok`, traversed (only under READ COMMITTED): The locked version is a newer, updated version, which may no longer satisfy the query condition. LockRows re-executes the lower plan using the new version (a mechanism called EvalPlanQual) to check whether it still satisfies `id > 99990`. If it does, return the values from the new version. Otherwise, skip the row, but keep it locked. Note that the rows are not sorted again. If the new version has a different `v`, the returned results may no longer be ordered by `v`.
- `TM_Deleted`: Under READ COMMITTED, skip the row. Under REPEATABLE READ, report an error (`could not serialize access due to concurrent update`).
- `TM_Updated` (only under REPEATABLE READ): Report the same error.
- `TM_WouldBlock`: This occurs only when `SKIP LOCKED` is used. Skip the row.

PostgreSQL's default isolation level is READ COMMITTED, so by default, rows modified by other transactions after scanning are rechecked through EvalPlanQual as described above.

Let's use the row with `id = 99995` to see how EvalPlanQual works. The version read during scanning has `v = 405`. Before LockRows locks it, another transaction updates it to `v = 7` and commits. When LockRows tries to lock the row, it finds that the row has been updated, follows the update chain to lock the new version, and returns `TM_Ok` with the traversed flag set, triggering EvalPlanQual.

EvalPlanQual re-executes a separate copy of the lower plan (Sort and Index Scan), while the remaining rows in the main plan's Sort stay unchanged. In this copy of the plan, Index Scan no longer scans the index. Instead, it returns only the newly locked version, and does so only once. It rechecks `id > 99990` against the new version, and Sort only needs to sort this single row. Since the new version satisfies the condition, LockRows returns its values, `(99995, 7)`. If it did not satisfy the condition, LockRows would skip the row but keep it locked. LockRows then returns to the main plan and continues fetching the next row from Sort.

Because only this row is re-executed, it is not sorted again together with the other rows: it was sorted using `v = 405`, but is returned with `v = 7`, still in its original position. The following figure shows this process:

<img src="/public/images/2026-10-20/8.png" alt="image-8"/>

### MySQL

MySQL has already completed locking during scanning, so its flow is much simpler. InnoDB locks a record immediately after reading it, and reads the latest version rather than a snapshot. Therefore, there is no window in which a record can be modified between being read and being locked. Conflicts are also handled during scanning:

- **The record is locked by another transaction**: Wait for that transaction to finish, then read the latest version of the record again.
- **The record has been updated by another transaction**: After waiting for the transaction to finish, InnoDB reads the record again at the same position in the index and gets the updated version. The record is still within the scan range, so there is no need to recheck it through a mechanism like EvalPlanQual. (If the update changes the primary key, InnoDB marks the old record as deleted and inserts a new record. When the scan encounters the old record, it skips it as described next.)
- **The record has been deleted by another transaction**: After acquiring the lock, InnoDB finds that the record is delete-marked, skips it, and continues scanning.

Rows that are successfully locked are passed directly to the upper layer, without the need to handle the locking result as PostgreSQL does. Under REPEATABLE READ, there is also no error caused by a concurrent modification after scanning.

### Comparison

The locking flows differ significantly. PostgreSQL reads from a snapshot during scanning and locks rows only after sorting. When locking, it must handle modifications made since the scan, resulting in several possible outcomes: waiting, locking and rechecking a new version, skipping a row, or reporting an error. It also needs a dedicated mechanism like EvalPlanQual to recheck whether the new version still satisfies the query condition. MySQL locks the latest version during scanning. When there is a conflict, it waits and then reads the latest version. There is no window between scanning and locking, which makes the flow much simpler.

## 4. Lock Implementation

The two systems also implement row locks differently: PostgreSQL stores row-lock information in the row itself, while InnoDB stores row locks in its in-memory lock system, organized by page and looked up through a hash table.

### PostgreSQL

PostgreSQL stores row-lock information in the heap tuple header. The `xmin` and `xmax` fields in the header record the transactions that inserted and deleted (or updated) the row, respectively. Row locks reuse `xmax`: when a transaction locks a row, its transaction ID is also written into `xmax`. Flags in `t_infomask` and `t_infomask2` distinguish whether `xmax` represents a deleting transaction or only a locking transaction, as well as the lock mode. The details are as follows:

**Whether xmax represents a locker or a deleter.** If `HEAP_XMAX_LOCK_ONLY` is set in `t_infomask`, the transaction in `xmax` has only locked the row, without deleting or updating it. Otherwise, `xmax` represents the transaction that deleted or updated the row.

**Lock modes.** PostgreSQL has four row-lock modes, represented by different flags in `t_infomask`:

- `FOR KEY SHARE`: `HEAP_XMAX_KEYSHR_LOCK`
- `FOR SHARE`: `HEAP_XMAX_SHR_LOCK`, which is not a separate flag but the combination of `HEAP_XMAX_EXCL_LOCK` and `HEAP_XMAX_KEYSHR_LOCK`
- `FOR NO KEY UPDATE`: `HEAP_XMAX_EXCL_LOCK`
- `FOR UPDATE`: `HEAP_XMAX_EXCL_LOCK`, together with `HEAP_KEYS_UPDATED` in `t_infomask2`

Both `FOR UPDATE` and `FOR NO KEY UPDATE` are exclusive locks, so `HEAP_XMAX_EXCL_LOCK` alone cannot distinguish them. PostgreSQL therefore uses `HEAP_KEYS_UPDATED` to indicate the stronger `FOR UPDATE` lock. This flag is also used for deletes and updates that modify key columns. In all these cases, it means that the transaction in `xmax` holds the strongest row lock.

**The example in this post.** After executing `SELECT ... FOR UPDATE`, each of the 10 locked rows has the current transaction's ID in `xmax`, `HEAP_XMAX_LOCK_ONLY | HEAP_XMAX_EXCL_LOCK` set in `t_infomask`, and `HEAP_KEYS_UPDATED` set in `t_infomask2`. Because the tuple header is modified, the pages containing these rows are marked dirty, and each row lock also generates a WAL record of type `XLOG_HEAP_LOCK`. In other words, taking a row lock in PostgreSQL is a write operation.

**Conflict detection.** When another transaction wants to update or lock the row, it reads `xmax` and the flags, then checks whether the transaction recorded in `xmax` is still running. If it is still running and the lock modes conflict, it waits for that transaction to finish. This is the waiting process in `heap_lock_tuple` discussed in Section 3.

**Multiple transactions holding locks on the same row.** If multiple transactions hold locks on a row at the same time (for example, they all hold `FOR SHARE`), a single `xmax` cannot store multiple transaction IDs. In this case, `xmax` stores a MultiXactId, with `HEAP_XMAX_IS_MULTI` set in `t_infomask`. Each member transaction and its lock mode are recorded in separate MultiXact storage.

**Lock release.** When a transaction commits or rolls back, PostgreSQL does not modify the rows it locked. The locks simply become invalid when the transaction ends. Later, when another transaction tries to update or lock the row (or when VACUUM processes the page), it finds that the transaction in `xmax` has finished, treats that `xmax` as invalid, and sets the hint bit `HEAP_XMAX_INVALID`. The `xmax` value and other flags remain until the row is locked, updated, or deleted again, or until they are overwritten or cleared by VACUUM freezing.

### MySQL

InnoDB does not store row locks in the rows themselves. Instead, they are maintained in the in-memory lock system, `lock_sys`. I will only give a brief overview here. For more details, see my previous posts on InnoDB locking: [MySQL Lock (1)](https://kernelmaker.github.io/MySQL-Lock-1), [MySQL Lock (2)](https://kernelmaker.github.io/MySQL-Lock-2), and [MySQL Implicit Locks](https://kernelmaker.github.io/MySQL-implicit-locks).

**Lock organization.** Record locks are organized by page. Locks taken by the same transaction, on the same page, in the same mode share one `lock_t` structure. The `lock_t` stores the page number and is followed by a bitmap. Each bit in the bitmap corresponds to a record on that page (identified by its record number within the page, `heap_no`), and a bit set to 1 indicates that the record is locked. `lock_sys` uses a hash table keyed by page ID (tablespace number + page number) to look up these `lock_t` structures. As a result, if a transaction locks many records on the same page, it only needs one `lock_t`. These locks exist only in memory: acquiring them does not modify records or generate redo logs, unlike PostgreSQL.

**Lock types.** Each record lock has a mode, `S` (shared) or `X` (exclusive), together with a range:

- next-key lock (`LOCK_ORDINARY`): Locks the record itself and the gap before it.
- `LOCK_REC_NOT_GAP`: Locks only the record itself.
- `LOCK_GAP`: Locks only the gap before the record.
- `LOCK_INSERT_INTENTION`: An insert intention lock taken on a gap before insertion.

`SELECT ... FOR UPDATE` takes an `X` lock. Under REPEATABLE READ, range scans generally take next-key locks. Under READ COMMITTED, they take only `LOCK_REC_NOT_GAP` locks, without locking gaps.

**The example in this post.** Under REPEATABLE READ, this statement takes an `X` next-key lock on each of the 10 records in the primary key, and also locks the supremum pseudo-record. Under READ COMMITTED, it takes only `X,REC_NOT_GAP` locks on the 10 records.

**Conflict detection.** When a transaction wants to lock a record, InnoDB uses the page number to find all `lock_t` structures for that page in `lock_sys`, then checks whether any other transaction's lock conflicts on the bit corresponding to that record. If there is a conflict, it creates a waiting `lock_t` for the requesting transaction and waits for the transaction holding the lock to release it.

**Implicit locks.** There is also a type of lock that is not stored in `lock_sys`. For a newly inserted, uncommitted record, the hidden column `DB_TRX_ID` contains the ID of the transaction that inserted it. InnoDB treats this as an implicit exclusive lock held by that transaction on the record, without creating a `lock_t`. Only when another transaction tries to lock the record does InnoDB first convert the implicit lock into an explicit lock in `lock_sys` and then perform conflict detection.

**Lock release.** When a transaction commits or rolls back, InnoDB removes all the locks held by that transaction from `lock_sys` and wakes up transactions waiting for them. No changes to the records themselves are needed.

### Comparison

The main difference in lock implementation is where the locks are stored. PostgreSQL writes row locks into tuple headers, so **acquiring a lock modifies data pages and generates WAL**, but does not consume additional memory. There is no limit on the number of rows that can be locked. InnoDB stores row locks in the in-memory `lock_sys`, so acquiring a lock neither modifies records nor generates logs, but it consumes memory. Locked records are tracked with bitmaps organized by page.

## 5. Lock Waiting

If a row has already been locked by another transaction, the current transaction must wait for the lock holder to finish before it can lock the row. But how does this waiting work? The two systems also differ significantly here.

### PostgreSQL

PostgreSQL knows the transaction ID of the row-lock holder from `xmax` and waits for that transaction to finish. This waiting uses PostgreSQL's regular lock manager (the heavyweight lock manager), which maintains various locks in shared memory using a hash table. Table locks are managed by the same lock manager. When a transaction is assigned a transaction ID (when it first modifies data or first takes a row lock), it acquires an exclusive lock (`ExclusiveLock`) on its own transaction ID in the lock manager and holds it until the transaction ends. Another transaction that needs to wait for it requests a shared lock (`ShareLock`) on that transaction ID. This request blocks until the holder finishes. Once the request succeeds, the shared lock is immediately released, and the waiting transaction returns to the row to check its state again. If the row's `xmax` contains a MultiXactId, it waits for each conflicting member transaction in turn.

But if multiple transactions are waiting for the same row, which one gets the row lock first? If they all just wait on the holder's transaction ID, they will be woken up together when the holder finishes, then race to acquire the row lock. There is no guarantee of the order, and one transaction might keep losing the race.

To avoid this, before waiting on the transaction ID, `heap_lock_tuple` first acquires a lock on the row itself in the lock manager (a tuple lock), which determines the order of the waiters:

1. The first waiter acquires the tuple lock, then waits on the holder's transaction ID.
2. Later waiters queue on the tuple lock instead of waiting on the holder's transaction ID.
3. When the holder finishes, the first waiter wakes up, locks the row (by writing its own transaction ID into `xmax`), and releases the tuple lock.
4. The next waiter acquires the tuple lock, finds that the row is now locked by the first waiter, and starts waiting on the first waiter's transaction ID instead.

In this way, transactions waiting for the same row acquire the row lock in the order in which they queued for the tuple lock. The tuple lock is held only while waiting and is released once the row has been locked. The row lock itself is still recorded only in `xmax`.

Let's use an example to illustrate this process. T1 has already locked the row (`xmax = T1`). T2, T3, and T4 arrive in that order and all want to lock it with `FOR UPDATE`. Before T1 finishes, the state is shown below:

<img src="/public/images/2026-10-20/9.png" alt="image-9"/>

There are two relationships worth noting in the figure: the waiters know from the row's `xmax` that the holder is T1, so they wait on the lock for transaction ID T1; the tuple lock is keyed by the row's location (page number and offset within the page), and all transactions that want to lock the row queue on it. T2 arrives first, acquires the tuple lock, and requests a shared lock on T1's transaction ID lock, waiting for T1 to finish. T3 and T4 queue behind T2 on the tuple lock.

After T1 commits, the state changes to the following:

<img src="/public/images/2026-10-20/10.png" alt="image-10"/>

T1's transaction ID lock disappears when T1 finishes. T2 wakes up, writes its transaction ID into the row's `xmax` to acquire the row lock, and then releases the tuple lock. T3, now at the front of the queue, acquires the tuple lock. It finds through `xmax` that the row is locked by T2, so it starts waiting on T2's transaction ID lock instead. T4 remains queued on the tuple lock.

The process above assumes that the waiters' lock modes conflict with one another (for example, they all use `FOR UPDATE`). Tuple locks use the same lock modes and conflict rules as table locks. Therefore, if the waiters' lock modes are compatible (for example, they all use `FOR SHARE`), they can hold the tuple lock at the same time and wait together for the holder to finish. When the holder finishes, they can all lock the row, and `xmax` becomes a MultiXactId containing them. However, if an incompatible waiter is queued between them (for example, a `FOR UPDATE` waiter), later waiters must queue behind that waiter even if they are compatible with the current holder, to prevent that waiter from being starved.

Since all waiting goes through the lock manager, PostgreSQL's deadlock detection is also handled by the lock manager. After a transaction has been waiting longer than `deadlock_timeout` (1 second by default), the lock manager checks whether the lock-wait relationships contain a cycle. If there is one, the transaction that detects the deadlock reports an error and exits, allowing other transactions to continue.

### MySQL

InnoDB waits on the record lock itself, rather than on the transaction holding it. When a transaction wants to lock a record and finds a conflicting lock held by another transaction in `lock_sys`, it creates a waiting `lock_t` for itself, adds it to the lock queue for the page containing that record, and suspends the current thread. The waiting `lock_t` is kept in the same structure as granted locks, so `lock_sys` directly shows which transaction is waiting for which record and which other transaction it is waiting on.

When the holder commits or rolls back, InnoDB removes its locks from `lock_sys`, then checks the waiting locks on those records, grants the locks that can now be granted, and wakes up the corresponding transactions. If multiple transactions are waiting for the same record, InnoDB does not grant locks strictly in arrival order. Instead, it gives priority to transactions with higher weights. A transaction's weight roughly reflects how many other transactions it is blocking; when weights are equal, arrival order is used. This scheduling method is called CATS (Contention-Aware Transaction Scheduling).

Deadlock detection is performed by a background thread. Whenever a transaction starts waiting, this thread is notified to check whether the lock-wait relationships contain a cycle. If it finds a deadlock, it chooses one transaction to roll back. In addition, if the wait exceeds `innodb_lock_wait_timeout` (50 seconds by default), the waiting statement reports an error (`ERROR 1205: Lock wait timeout exceeded`).

### Comparison

There are three main differences in lock waiting. First, they wait on different objects. PostgreSQL records row locks only in `xmax`, so there is no lock object on which waiters can queue. Instead, a waiter waits on the holder's transaction ID, and a tuple lock is used to queue transactions waiting for the same row. InnoDB stores row locks directly in `lock_sys`, so waiters queue directly on the record's lock queue. Second, the granting order differs. PostgreSQL follows the order of the tuple-lock queue (compatible locks can be granted together), while InnoDB uses CATS weights to prioritize transactions that block more other transactions. Third, deadlock detection happens at different times. In PostgreSQL, after a transaction has waited longer than `deadlock_timeout` (1 second by default), that transaction checks for deadlocks itself. InnoDB uses a background thread to check when a transaction starts waiting. Also, PostgreSQL does not limit lock-wait time by default (`lock_timeout` defaults to 0), while InnoDB reports an error after 50 seconds by default.

## 6. Repeatable Read

The previous sections were based on the default isolation levels of the two systems: READ COMMITTED for PostgreSQL and REPEATABLE READ for MySQL. Under REPEATABLE READ, executing the same statement multiple times in one transaction should produce the same results. The two systems take completely different approaches to achieve this.

### PostgreSQL

Under REPEATABLE READ, PostgreSQL uses the same snapshot for the entire transaction: the snapshot taken when the first statement in the transaction starts. All subsequent statements read data according to this snapshot.

The effect on locking was mentioned in Section 3. When LockRows tries to lock a row and finds that the version read during scanning has been updated or deleted by another transaction, it does not lock the latest version and recheck it through EvalPlanQual as it would under READ COMMITTED. Instead, it directly reports an error (`could not serialize access due to concurrent update`), requiring the application to retry the entire transaction. This is because locking and returning the latest version would cause the transaction to see data that is not part of its own snapshot.

On the other hand, PostgreSQL locks only existing rows, not ranges. After a transaction executes this statement, another transaction can still insert a new row with `id = 100001` without waiting. This new row is not in the first transaction's snapshot, so when that transaction executes the statement again, it still cannot see the new row and the result stays unchanged. But the new row is not locked by the first transaction either.

```
T1: BEGIN ISOLATION LEVEL REPEATABLE READ;
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Locks 10 rows
T2: INSERT INTO t VALUES (100001, 5);                         -- Succeeds immediately
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Still returns 10 rows
```

### MySQL

Under MySQL's REPEATABLE READ, a regular `SELECT` uses the read view established by the first read after the transaction starts. But `SELECT ... FOR UPDATE` is a locking read: it does not use a read view and reads the latest committed version. Since a locking read always reads the latest data, to keep its results unchanged when it is executed multiple times in the same transaction, MySQL must prevent other transactions from inserting new rows into the scanned range.

This is the purpose of next-key locks and the lock on the supremum. In the example from Section 4, the statement takes next-key locks on 10 records and locks the supremum, covering the range (99990, +∞). When another transaction tries to insert `id = 100001`, it must acquire an insert intention lock on the gap before the supremum. This conflicts with those locks, so it waits until the first transaction finishes.

```
T1: SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; START TRANSACTION;
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Locks 10 records and gaps
T2: INSERT INTO t VALUES (100001, 5);                         -- Waits
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Still returns 10 rows
T1: COMMIT;                                                   -- T2's insert completes now
```

Under READ COMMITTED, InnoDB does not take gap locks, only locks on the records themselves. T2's insert in the example above can succeed immediately. After T2 commits, when T1 executes the statement again, it sees and locks the newly inserted row (before T2 commits, T1 would wait because of T2's implicit lock on that row).

When InnoDB encounters a record locked or being modified by another transaction during scanning, it handles the record under REPEATABLE READ the same way as under READ COMMITTED: it waits for the lock holder to finish, then reads the latest version, without reporting an error.

### Comparison

The two systems ensure repeatable reads in completely different ways. PostgreSQL relies on snapshots: a transaction always reads from the same snapshot and does not lock ranges, so other transactions can insert new rows at any time. If a row to be locked has been modified since the snapshot, PostgreSQL reports an error and requires the application to retry. InnoDB relies on locks: locking reads always read the latest version, and next-key locks and the lock on the supremum prevent other transactions from inserting new rows into scanned ranges. When there is a conflict, other transactions wait instead of getting an error. The former allows higher concurrency, but the application must handle serialization errors and retry. The latter does not require retries, but locks larger ranges, makes other transactions' inserts wait, and is also more likely to cause deadlocks.

## 7. Serializable

Under SERIALIZABLE, the difference between the two systems becomes even greater: PostgreSQL adds conflict detection on top of snapshots, while MySQL turns regular reads into locking reads as well.

### PostgreSQL

PostgreSQL's REPEATABLE READ is actually Snapshot Isolation. Snapshot Isolation guarantees repeatable reads, but still allows anomalies such as write skew, so it is not truly serializable. PostgreSQL implements Serializable Snapshot Isolation (SSI) on top of Snapshot Isolation for its SERIALIZABLE level.

Under SERIALIZABLE, the locking flow of this statement is exactly the same as under REPEATABLE READ: the entire transaction uses the same snapshot, only existing rows are locked, and a row modified after scanning still causes an error. The difference is that, during scanning, PostgreSQL also records which data the transaction has read. These are called predicate locks (shown as `SIReadLock` in `pg_locks`). In this example, the B+Tree leaf pages accessed by Index Scan and the heap tuples it reads are recorded.

Predicate locks are not real locks: they do not block any transactions. They are used only to detect read-write dependencies between transactions. If another transaction writes into a range read by the current transaction (for example, inserting `id = 100001` into a leaf page that the current transaction has read), PostgreSQL records a dependency indicating that the current transaction read data that the other transaction wrote. Only when these dependencies form a structure that could violate serializability (for example, two transactions each read a range that the other transaction writes to) does PostgreSQL report an error for one of the transactions (`could not serialize access due to read/write dependencies among transactions`), requiring the application to retry.

```
T1: BEGIN ISOLATION LEVEL SERIALIZABLE;
T2: BEGIN ISOLATION LEVEL SERIALIZABLE;
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Locks 10 rows, records predicate locks
T2: SELECT * FROM t WHERE id > 99990;                         -- Regular read, does not wait, records predicate locks
T1: INSERT INTO t VALUES (100002, 1);                         -- Succeeds immediately
T2: INSERT INTO t VALUES (100001, 1);                         -- Does not wait
T1: COMMIT;                                                   -- Succeeds
T2: COMMIT;                                                   -- T2 errors: each transaction read a range written by the other
```

No transaction needs to wait during this process. In the order shown above, T1 has not yet committed when T2 inserts its row, so PostgreSQL only records the dependency. When T1 commits, it detects the structure of mutual dependencies and marks T2 as a transaction that must fail. T2 then reports the error at commit. If T1 had already committed before T2's insert, the error would occur directly on T2's `INSERT`. If only T2 inserted a row and T1 did not, there would be only a one-way dependency, and both transactions could commit successfully.

### MySQL

MySQL's SERIALIZABLE makes just one change on top of REPEATABLE READ: inside non-autocommit transactions, a regular `SELECT` is also executed as `SELECT ... FOR SHARE`, taking `S` next-key locks on the records and gaps it reads.

For the statement in this post, `FOR UPDATE` is already a locking read, so locking is exactly the same as under REPEATABLE READ: `X` next-key locks on the 10 records, plus a lock on the supremum. The difference appears in regular reads performed by other transactions. They must take `S` locks, which conflict with these `X` locks, so they have to wait.

```
T1: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; START TRANSACTION;
T2: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; START TRANSACTION;
T1: SELECT * FROM t WHERE id > 99990 ORDER BY v FOR UPDATE;   -- Locks 10 records and gaps
T2: SELECT * FROM t WHERE id > 99990;                         -- Regular read also needs S locks, waits
T1: COMMIT;                                                   -- T2's read returns only now
```

In other words, InnoDB also locks reads to place potentially conflicting transactions into a definite order. It does not need to detect conflicts at commit time, and serialization errors do not occur (although deadlocks and lock-wait timeouts can still cause errors). The cost is that reads and writes block one another, reducing concurrency.

### Comparison

The two systems take completely different approaches to serializability. PostgreSQL uses SSI: reads do not acquire blocking locks; it only records the data that was read, and writes do not have to wait because of these records. During reads, writes, and commits, it checks for dependency structures that could violate serializability, and only then makes a transaction fail and retry. This is an optimistic approach. InnoDB uses locking: on top of REPEATABLE READ, it also makes regular reads acquire shared locks, so conflicting transactions wait directly. This is a pessimistic approach. For the `SELECT ... FOR UPDATE` in this post, both systems use the same locking flow under SERIALIZABLE as under REPEATABLE READ. The difference lies in other transactions running concurrently with it: in PostgreSQL, other transactions' reads and inserts do not need to wait, but may fail (as late as commit time); in InnoDB, other transactions' reads and inserts must wait, but do not fail except in cases such as deadlocks or lock-wait timeouts.

## 8. Summary

Row locking is one of the core components of a transactional database. The trade-offs in its design and implementation directly affect the database's concurrency and how applications use it. By looking at a single `SELECT ... FOR UPDATE` statement, we can see several major differences between PostgreSQL and MySQL.

First, they differ in when they lock rows and which versions they read. PostgreSQL scans according to a snapshot and locks rows one by one only after sorting. Therefore, it must handle rows modified after scanning, with different possible outcomes: waiting, locking and rechecking a new version through EvalPlanQual, skipping a row, or reporting an error. MySQL locks the latest versions while scanning, making the flow much simpler.

Second, they differ in where locks are stored. PostgreSQL writes row locks into tuple headers, so locking modifies data pages and generates WAL, but does not consume additional memory. InnoDB stores row locks in the in-memory `lock_sys`, so locking neither modifies records nor generates logs.

Third, they differ in how they wait. PostgreSQL waits on the holder's transaction ID and uses tuple locks to queue waiters. InnoDB waits directly on the record's lock queue and uses CATS weights to determine granting order.

Fourth, they differ in how they enforce isolation levels. PostgreSQL relies on snapshots and does not lock ranges. Under REPEATABLE READ, it reports an error when it encounters a conflicting modification. Under SERIALIZABLE, it uses SSI to detect conflicts during reads, writes, and commits. In both cases, the application must retry. InnoDB relies on locks, uses next-key locks to lock ranges, and also locks regular reads under SERIALIZABLE, making conflicting transactions wait directly.

Behind these differences are two different approaches. PostgreSQL locks only the rows that actually need to be locked, not ranges. It mainly relies on snapshots and conflict detection to enforce isolation levels, and when a conflict cannot be handled, it reports an error and lets the application retry. InnoDB locks ranges in addition to rows, using locks to prevent potential conflicts in advance, and makes other transactions wait when conflicts occur.

There is much more to row locking, such as the locking flows for UPDATE and DELETE, and the details of MultiXact and implicit locks. I have not covered those topics here and will leave them for future posts.
