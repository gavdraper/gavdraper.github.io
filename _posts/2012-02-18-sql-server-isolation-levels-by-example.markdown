---
layout: post
title: SQL Server Isolation Levels By Example
date: '2012-02-18 12:57:28'
---

<p>Isolation levels in SQL Server control the way locking works between transactions.</p>  <p>SQL Server 2008 supports the following isolation levels</p>  <ul>   <li>Read Uncommitted </li>    <li>Read Committed (The default) </li>    <li>Repeatable Read </li>    <li>Serializable </li>    <li>Snapshot </li> </ul>  <p>Before I run through each of these in detail you may want to create a new database to run the examples, run the following script on the new database to create the sample data. <strong>Note</strong> : You’ll also want to drop the IsolationTests table and re-run this script before each example to reset the data.</p>  
```language-sql
CREATE TABLE IsolationTests
(
    Id INT IDENTITY,
    Col1 INT,
    Col2 INT,
    Col3 INT
)

INSERT INTO IsolationTests(Col1,Col2,Col3)
SELECT 1,2,3
UNION ALL SELECT 1,2,3
UNION ALL SELECT 1,2,3
UNION ALL SELECT 1,2,3
UNION ALL SELECT 1,2,3
UNION ALL SELECT 1,2,3
UNION ALL SELECT 1,2,3
```

<p><font style="font-weight: bold"></font></p>

<p>Also before we go any further it is important to understand these two terms….</p>

<ol>
  <li><strong>Dirty Reads</strong> – This is when you read uncommitted data, when doing this there is no guarantee that data read will ever be committed meaning the data could well be bad. </li>

  <li><strong>Phantom Reads</strong> – This is when data that you are working with has been changed by another transaction since you first read it in. This means subsequent reads of this data in the same transaction could well be different. </li>
</ol>

<h3><font style="font-weight: bold">Read Uncommitted</font></h3>

<p>This is the lowest isolation level there is. Read uncommitted causes no shared locks to be requested which allows you to read data that is currently being modified in other transactions. It also allows other transactions to modify data that you are reading.</p>

<p>As you can probably imagine this can cause some unexpected results in a variety of different ways. For example data returned by the select could be in a half way state if an update was running in another transaction causing some of your rows to come back with the updated values and some not to.</p>

<p>To see read uncommitted in action lets run Query1 in one tab of Management Studio and then quickly run Query2 in another tab before Query1 completes.</p>
<p><u>Query1</u></p>
```language-sql
BEGIN TRAN
UPDATE IsolationTests SET Col1 = 2
--Simulate having some intensive processing here with a wait
WAITFOR DELAY '00:00:10'
ROLLBACK
```

<p><u>Query2</u></p>
```language-sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED
SELECT * FROM IsolationTests
```

<p>Notice that Query2 will not wait for Query1 to finish, also more importantly Query2 returns dirty data. Remember Query1 rolls back all its changes however Query2 has returned the data anyway, this is because it didn't wait for all the other transactions with exclusive locks on this data it just returned what was there at the time.</p>

<p>There is a syntactic shortcut for querying data using the read uncommitted isolation level by using the NOLOCK table hint. You could change the above Query2 to look like this and it would do the exact same thing.</p>

```language-sql
SELECT * FROM IsolationTests WITH(NOLOCK)
```

<h3><font style="font-weight: bold">Read Committed</font></h3>

<p>This is the default isolation level and means selects will only return committed data. Select statements will issue shared lock requests against data you’re querying this causes you to wait if another transaction already has an exclusive lock on that data. Once you have your shared lock any other transactions trying to modify that data will request an exclusive lock and be made to wait until your Read Committed transaction finishes.</p>

<p>You can see an example of a read transaction waiting for a modify transaction to complete before returning the data by running the following Queries in separate tabs as you did with Read Uncommitted. </p>

<p><u>Query1</u></p>

```language-sql
BEGIN TRAN
UPDATE IsolationTests SET Col1 = 2
--Simulate having some intensive processing here with a wait
WAITFOR DELAY '00:00:10'
ROLLBACK
```

<p><u>Query2</u></p>

```language-sql
SELECT * FROM IsolationTests
```

<p>Notice how Query2 waited for the first transaction to complete before returning and also how the data returned is the data we started off with as Query1 did a rollback. The reason no isolation level was specified is because Read Committed is the default isolation level for SQL Server. If you want to check what isolation level you are running under you can run “DBCC useroptions”. Remember isolation levels are Connection/Transaction specific so different queries on the same database are often run under different isolation levels.</p>

<h3><font style="font-weight: bold">Repeatable Read</font></h3>

<p>This is similar to Read Committed but with the additional guarantee that if you issue the same select twice in a transaction you will get the same results both times. It does this by holding on to the shared locks it obtains on the records it reads until the end of the transaction, This means any transactions that try to modify these records are forced to wait for the read transaction to complete. </p>

<p>As before run Query1 then while its running run Query2</p>

<p><u>Query1</u></p>

```language-sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRAN
SELECT * FROM IsolationTests
WAITFOR DELAY '00:00:10'
SELECT * FROM IsolationTests
ROLLBACK
```


<p><u>Query2</u></p>

```language-sql
UPDATE IsolationTests SET Col1 = -1
```

<p>Notice that Query1 returns the same data for both selects even though you ran a query to modify the data before the second select ran. This is because the Update query was forced to wait for Query1 to finish due to the exclusive locks that were opened as you specified Repeatable Read.</p>

<p>If you rerun the above Queries but change Query1 to Read Committed you will notice the two selects return different data and that Query2 does not wait for Query1 to finish.</p>

<p>One last thing to know about Repeatable Read is that the data can change between 2 queries if more records are added. Repeatable Read guarantees records queried by a previous select will not be changed or deleted, it does not stop new records being inserted so it is still very possible to get Phantom Reads at this isolation level.</p>

<h3><font style="font-weight: bold">Serializable</font></h3>

<p>This isolation level takes Repeatable Read and adds the guarantee that no new data will be added eradicating the chance of getting Phantom Reads. It does this by placing range locks on the queried data. This causes any other transactions trying to modify or insert data touched on by this transaction to wait until it has finished.</p>

<p>You know the drill by now run these queries side by side…</p>

<p><u>Query1</u></p>

```language-sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
BEGIN TRAN
SELECT * FROM IsolationTests
WAITFOR DELAY '00:00:10'
SELECT * FROM IsolationTests
ROLLBACK
```

<p><u>Query2</u></p>

```language-sql
INSERT INTO IsolationTests(Col1,Col2,Col3)
VALUES (100,100,100)
```

<p>You’ll see that the insert in Query2 waits for Query1 to complete before it runs eradicating the chance of a phantom read. If you change the isolation level in Query1 to repeatable read, you’ll see the insert no longer gets blocked and the two select statements in Query1 return a different amount of rows. </p>

<h3><font style="font-weight: bold">Snapshot</font></h3>

<p>This provides the same guarantees as serializable. So what's the difference? Well it’s more in the way it works, using snapshot doesn't block other queries from inserting or updating the data touched by the snapshot transaction. Instead row versioning is used so when data is changed the old version is kept in tempdb so existing transactions will see the version without the change. When all transactions that started before the changes are complete the previous row version is removed from tempdb.  This means that even if another transaction has made changes you will always get the same results as you did the first time in that transaction.</p>

<p>So on the plus side your not blocking anyone else from modifying the data whilst you run your transaction but…. You’re using extra resources on the SQL Server to hold multiple versions of your changes.</p>

<p>To use the snapshot isolation level you need to enable it on the database by running the following command</p>

```language-sql
ALTER DATABASE IsolationTests
SET ALLOW_SNAPSHOT_ISOLATION ON
```

<p>If you rerun the examples from serializable but change the isolation level to snapshot you will notice that you still get the same data returned but Query2 no longer waits for Query1 to complete.</p>

<h3><font style="font-weight: bold">Summary</font></h3>

<p>You should now have a good idea how each of the different isolation levels work. You can see how the higher the level you use the less concurrency you are offering and the more blocking you bring to the table. You should always try to use the lowest isolation level you can which is usually read committed.</p>