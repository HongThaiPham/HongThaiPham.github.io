---
layout: post
title: 'Pagination with OFFSET / FETCH : A better way'
date: 2019-03-28 08:58:00 +0700
categories: [database]
tags: [database, SQL, pagination]
image: place_holder.jpeg
---

Pagination is a common use case throughout client and web applications everywhere. Google shows you 10 results at a time, your online bank may show 20 bills per page, and bug tracking and source control software might display 50 items on the screen.

Based on the indexing of the table, the columns needed, and the sort method chosen, paging can be relatively painless. If you're looking for the "first" 20 customers and the clustered index supports that sorting (say, a clustered index on an IDENTITY column or DateCreated column), then the query is going to be pretty efficient. If you need to support sorting that requires non-clustered indexes, and especially if you have columns needed for output that aren't covered by the index (never mind if there is no supporting index), the queries can get more expensive. And even the same query (with a different @PageNumber parameter) can get much more expensive as the @PageNumber gets higher – since more reads may be required to get to that "slice" of the data.

Some will say that progressing toward the end of the set is something that you can solve by throwing more memory at the problem (so you eliminate any physical I/O) and/or using application-level caching (so you're not going to the database at all). Let's assume for the purposes of this post that more memory isn't always possible, since not every customer can add RAM to a server that's out of memory slots, or just snap their fingers and have newer, bigger servers ready to go. Especially since some customers are on Standard Edition, so are capped at 64GB (SQL Server 2012) or 128GB (SQL Server 2014), or are using even more limited editions such as Express (1GB) or whatever they're calling Azure SQL Database this week (many different servicing tiers).

So I wanted to look at the common paging approach on SQL Server 2012 – OFFSET / FETCH – and suggest a variation that will lead to more linear paging performance across the entire set, instead of only being optimal at the beginning. Which, sadly, is all that a lot of shops will test.

## Setup

I'm going to borrow from a recent post, [Bad habits : Focusing only on disk space when choosing keys](https://sqlperformance.com/2015/01/sql-indexes/bad-habits-disk-space), where I populated the following table with 1,000,000 rows of random-ish (but not entirely realistic) customer data:

```sql
CREATE TABLE [dbo].[Customers_I]
(
  [CustomerID] [int] IDENTITY(1,1) NOT NULL,
  [FirstName] [nvarchar](64) NOT NULL,
  [LastName] [nvarchar](64) NOT NULL,
  [EMail] [nvarchar](320) NOT NULL,
  [Active] [bit] NOT NULL DEFAULT ((1)),
  [Created] [datetime] NOT NULL DEFAULT (sysdatetime()),
  [Updated] [datetime] NULL,
  CONSTRAINT [C_PK_Customers_I] PRIMARY KEY CLUSTERED ([CustomerID] ASC)
);
GO
CREATE NONCLUSTERED INDEX [C_Active_Customers_I]
  ON [dbo].[Customers_I]
  ([FirstName] ASC, [LastName] ASC, [EMail] ASC)
  WHERE ([Active] = 1);
GO
CREATE UNIQUE NONCLUSTERED INDEX [C_Email_Customers_I]
  ON [dbo].[Customers_I]
  ([EMail] ASC);
GO
CREATE NONCLUSTERED INDEX [C_Name_Customers_I]
  ON [dbo].[Customers_I]
  ([LastName] ASC, [FirstName] ASC)
  INCLUDE ([EMail]);
GO
```

Since I knew I would be testing I/O here, and would be testing from both a warm and cold cache, I made the test at least a little bit more fair by rebuilding all of the indexes to minimize fragmentation (as would be done less disruptively, but regularly, on most busy systems that are performing any type of index maintenance):

```sql
ALTER INDEX ALL ON dbo.Customers_I REBUILD WITH (ONLINE = ON);
```

After the rebuild, fragmentation comes in now at 0.05% – 0.17% for all indexes (index level = 0), pages are filled over 99%, and the row count / page count for the indexes are as follows:

| Index                                 | Page Count | Row Count |
| ------------------------------------- | ---------- | --------- |
| C_PK_Customers_I (clustered index)    | 19,210     | 1,000,000 |
| C_Email_Customers_I                   | 7,344      | 1,000,000 |
| C_Active_Customers_I (filtered index) | 13,648     | 815,235   |
| C_Name_Customers_I                    | 16,824     | 1,000,000 |

This obviously isn't a super-wide table, and I've left compression out of the picture this time. Perhaps I will explore more configurations in a future test.

## Paging Scenarios

Typically, users will formulate a paging query like this (I'm going to leave the old-school, pre-2012 methods out of this post):

```sql
SELECT [a_bunch_of_columns]
  FROM dbo.[some_table]
  ORDER BY [some_column_or_columns]
  OFFSET @PageSize * (@PageNumber - 1) ROWS
  FETCH NEXT @PageSize ROWS ONLY;
```

As I mentioned above, this works just fine if there is an index that supports the ORDER BY and that covers all of the columns in the SELECT clause (and, for more complex queries, the WHERE and JOIN clauses). However, the sort costs might be overwhelming with no supporting index, and if the output columns aren't covered, you will either end up with a whole bunch of key lookups, or you may even get a table scan in some scenarios.

Let's get more specific. Given the table and indexes above, I wanted to test these scenarios, where we want to show 100 rows per page, and output all of the columns in the table:

1. **Default** – `ORDER BY CustomerID` (clustered index).
2. **Phone book** – `ORDER BY LastName, FirstName` (supporting non-clustered index).
3. **User-defined** – `ORDER BY FirstName DESC, EMail` (no supporting index).

I wanted to test these methods and compare plans and metrics when – under both warm cache and cold cache scenarios – looking at page 1, page 500, page 5,000, and page 9,999. So I created these procedures (differing only by the ORDER BY clause):

```sql
CREATE PROCEDURE dbo.Pagination_Test_1 -- ORDER BY CustomerID
  @PageNumber INT = 1,
  @PageSize   INT = 100
AS
BEGIN
  SET NOCOUNT ON;

  SELECT CustomerID, FirstName, LastName,
      EMail, Active, Created, Updated
    FROM dbo.Customers_I
    ORDER BY CustomerID
    OFFSET @PageSize * (@PageNumber - 1) ROWS
    FETCH NEXT @PageSize ROWS ONLY OPTION (RECOMPILE);
END
GO

CREATE PROCEDURE dbo.Pagination_Test_2 -- ORDER BY LastName, FirstName
CREATE PROCEDURE dbo.Pagination_Test_3 -- ORDER BY FirstName DESC, EMail
```

In reality, you will probably just have one procedure that either uses dynamic SQL or a CASE expression to dictate the order. In either case, you may see best results by using OPTION (RECOMPILE) on the query to avoid reuse of plans that are optimal for one sorting option but not all. I created separate procedures here to take those variables away; I added OPTION (RECOMPILE) for these tests to stay away from parameter sniffing and other optimization issues without flushing the entire plan cache repeatedly.

## An alternate approach

A slightly different approach, which I don't see implemented very often, is to locate the "page" we're on using only the clustering key, and then join to that:

```sql
;WITH pg AS
(
  SELECT [key_column]
  FROM dbo.[some_table]
  ORDER BY [some_column_or_columns]
  OFFSET @PageSize * (@PageNumber - 1) ROWS
  FETCH NEXT @PageSize ROWS ONLY
)
SELECT t.[bunch_of_columns]
  FROM dbo.[some_table] AS t
  INNER JOIN pg ON t.[key_column] = pg.[key_column] -- or EXISTS
  ORDER BY [some_column_or_columns];
```

It's more verbose code, of course, but hopefully it's clear what SQL Server can be coerced into doing: avoiding a scan, or at least deferring lookups until a much smaller resultset is whittled down. Paul White (@SQL_Kiwi) investigated a similar approach back in 2010, before OFFSET/FETCH was introduced in the early SQL Server 2012 betas (I first [blogged about it](https://sqlblog.org/blogs/aaron_bertrand/archive/2010/11/10/sql-server-11-denali-using-the-offset-clause.aspx) later that year).

Given the scenarios above, I created three more procedures, with the only difference between the column(s) specified in the ORDER BY clauses (we now need two, one for the page itself, and one for ordering the result):

```sql
CREATE PROCEDURE dbo.Alternate_Test_1 -- ORDER BY CustomerID
  @PageNumber INT = 1,
  @PageSize   INT = 100
AS
BEGIN
  SET NOCOUNT ON;

  ;WITH pg AS
  (
    SELECT CustomerID
      FROM dbo.Customers_I
      ORDER BY CustomerID
      OFFSET @PageSize * (@PageNumber - 1) ROWS
      FETCH NEXT @PageSize ROWS ONLY
  )
  SELECT c.CustomerID, c.FirstName, c.LastName,
      c.EMail, c.Active, c.Created, c.Updated
  FROM dbo.Customers_I AS c
  WHERE EXISTS (SELECT 1 FROM pg WHERE pg.CustomerID = c.CustomerID)
  ORDER BY c.CustomerID OPTION (RECOMPILE);
END
GO

CREATE PROCEDURE dbo.Alternate_Test_2 -- ORDER BY LastName, FirstName
CREATE PROCEDURE dbo.Alternate_Test_3 -- ORDER BY FirstName DESC, EMail
```

_Note: This may not work so well if your primary key is not clustered – part of the trick that makes this work better, when a supporting index can be used, is that the clustering key is already in the index, so a lookup is often avoided._

## Testing the clustering key sort

First I tested the case where I didn't expect much variance between the two methods – sorting by the clustering key. I ran these statements in a batch in SQL Sentry Plan Explorer and observed duration, reads, and the graphical plans, making sure that each query was starting from a completely cold cache:

```sql
SET NOCOUNT ON;
-- default method
DBCC DROPCLEANBUFFERS;
EXEC dbo.Pagination_Test_1 @PageNumber = 1;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Pagination_Test_1 @PageNumber = 500;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Pagination_Test_1 @PageNumber = 5000;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Pagination_Test_1 @PageNumber = 9999;

-- alternate method
DBCC DROPCLEANBUFFERS;
EXEC dbo.Alternate_Test_1 @PageNumber = 1;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Alternate_Test_1 @PageNumber = 500;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Alternate_Test_1 @PageNumber = 5000;
DBCC DROPCLEANBUFFERS;
EXEC dbo.Alternate_Test_1 @PageNumber = 9999;
```

The results here were not astounding. Over 5 executions the average number of reads are shown here, showing negligible differences between the two queries, across all page numbers, when sorting by the clustering key:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_reads_ck.png)

The plan for the default method (as shown in [SQL Sentry Plan Explorer](https://sentryone.com/plan-explorer)) in all cases was as follows:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_1_def.png)

While the plan for the CTE-based method looked like this:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_1_alt.png)

Now, while I/O was the same regardless of caching (just a lot more read-ahead reads in the cold cache scenario), I measured the duration with a cold cache and also with a warm cache (where I commented out the DROPCLEANBUFFERS commands and ran the queries multiple times before measuring). These durations looked like this:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_ck_dur.png)

While you can see a pattern that shows duration increasing as the page number gets higher, keep the scale in mind: to hit rows 999,801 -> 999,900, we're talking half a second in the worst case and 118 milliseconds in the best case. The CTE approach wins, but not by a whole lot.

## Testing the phone book sort

Next, I tested the second case, where the sorting was supported by a non-covering index on LastName, FirstName. The query above just changed all instances of Test_1 to Test_2. Here were the reads using a cold cache:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_reads_pb.png)

(The reads under a warm cache followed the same pattern – the actual numbers differed slightly, but not enough to justify a separate chart.)

When we're not using the clustered index to sort, it is clear that the I/O costs involved with the traditional method of OFFSET/FETCH are far worse than when identifying the keys first in a CTE, and pulling the rest of the columns just for that subset.

Here is the plan for the traditional query approach:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_2_def.png)

And the plan for my alternate, CTE approach:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_2_alt.png)

Finally, the durations:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_pb_dur.png)

The traditional approach shows a very obvious upswing in duration as you march toward the end of the pagination. The CTE approach also shows a non-linear pattern, but it is far less pronounced and yields better timing at every page number. We see 117 milliseconds for the second-to-last page, versus the traditional approach coming in at almost two seconds.

## Testing the user-defined sort

Finally, I changed the query to use the _Test_3_ stored procedures, testing the case where the sort was defined by the user and did not have a supporting index. The I/O was consistent across each set of tests; the graph is so uninteresting, [I'm just going to link to it](https://sqlperformance.com/wp-content/uploads/2015/01/pag_reads_un.png). Long story short: there were a little over 19,000 reads in all tests. The reason is because every single variation had to perform a full scan due to the lack of an index to support the ordering. Here is the plan for the traditional approach:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_3_def.png)

And while the plan for the CTE version of the query looks alarmingly more complex…

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_plan_3_alt.png)

…it actually leads to lower durations in all but one case. Here are the durations:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_un_dur.png)

You can see that we can't get linear performance here using either method, but the CTE does come out on top by a good margin (anywhere from 16% to 65% better) in every single case except the cold cache query against the first page (where it lost by a whopping 8 milliseconds). Also interesting to note that the traditional method isn't helped much at all by a warm cache in the "middle" (pages 500 and 5000); only toward the end of the set is any efficiency worth mentioning.

## Higher volume

After individual testing of a few executions and taking averages, I thought it would also make sense to test a high volume of transactions that would somewhat simulate real traffic on a busy system. So I created a job with 6 steps, one for each combination of query method (traditional paging vs. CTE) and sort type (clustering key, phone book, and unsupported), with a 100-step sequence of hitting the four page numbers above, 10 times each, and 60 other page numbers chosen at random (but the same for each step). Here is how I generated the job creation script:

```sql
SET NOCOUNT ON;
DECLARE @sql NVARCHAR(MAX), @job SYSNAME = N'Paging Test', @step SYSNAME, @command NVARCHAR(MAX);

;WITH t10 AS (SELECT TOP (10) number FROM master.dbo.spt_values),
f AS (SELECT f FROM (VALUES(1),(500),(5000),(9999)) AS f(f))
SELECT @sql = STUFF((SELECT CHAR(13) + CHAR(10)
  + N'EXEC dbo.$p$_Test_$v$ @PageNumber = ' + RTRIM(f) + ';'
  FROM
  (
    SELECT f FROM
    (
      SELECT f.f FROM t10 CROSS JOIN f
      UNION ALL
      SELECT TOP (60) f = ABS(CHECKSUM(NEWID())) % 10000
	    FROM sys.all_objects
    ) AS x
  ) AS y ORDER BY NEWID()
  FOR XML PATH(''),TYPE).value(N'.[1]','nvarchar(max)'),1,0,'');

IF EXISTS (SELECT 1 FROM msdb.dbo.sysjobs WHERE name = @job)
BEGIN
  EXEC msdb.dbo.sp_delete_job @job_name = @job;
END

EXEC msdb.dbo.sp_add_job
  @job_name = @job,
  @enabled = 0,
  @notify_level_eventlog = 0,
  @category_id = 0,
  @owner_login_name = N'sa';

EXEC msdb.dbo.sp_add_jobserver
  @job_name = @job,
  @server_name = N'(local)';

DECLARE c CURSOR LOCAL FAST_FORWARD FOR
SELECT step = p.p + '_' + v.v,
    command = REPLACE(REPLACE(@sql, N'$p$', p.p), N'$v$', v.v)
  FROM
  (SELECT v FROM (VALUES('1'),('2'),('3')) AS v(v)) AS v
  CROSS JOIN
  (SELECT p FROM (VALUES('Alternate'),('Pagination')) AS p(p)) AS p
  ORDER BY p.p, v.v;

OPEN c; FETCH c INTO @step, @command;

WHILE @@FETCH_STATUS <> -1
BEGIN
  EXEC msdb.dbo.sp_add_jobstep
    @job_name   = @job,
    @step_name  = @step,
    @command    = @command,
    @database_name = N'IDs',
    @on_success_action = 3;

  FETCH c INTO @step, @command;
END

EXEC msdb.dbo.sp_update_jobstep
  @job_name = @job,
  @step_id  = 6,
  @on_success_action = 1; -- quit with success

PRINT N'EXEC msdb.dbo.sp_start_job @job_name = ''' + @job + ''';';
```

Here is the resulting job step list and one of the step's properties:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_job.png)

I ran the job five times, then reviewed the job history, and here were the average runtimes of each step:

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_job_runtimes.png)

I also correlated one of the executions on the [SQL Sentry Event Manager](https://sentryone.com/platform/sql-server-performance-monitoring) calendar…

![](https://sqlperformance.com/wp-content/uploads/2015/01/paging_em.png)

…with the [SQL Sentry](https://sentryone.com/platform/sql-server-performance-monitoring) dashboard, and manually marked roughly where each of the six steps ran. Here is the CPU usage chart from the Windows side of the dashboard:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_dash_cpu1.png)

And from the SQL Server side of the dashboard, the interesting metrics were in the Key Lookups and Waits graphs:

![](https://sqlperformance.com/wp-content/uploads/2015/01/pag_dash_wait1.png)

The most interesting observations just from a purely visual perspective:

CPU is pretty hot, at around 80%, during step 3 (CTE + no supporting index) and step 6 (traditional + no supporting index);
CXPACKET waits are relatively high during step 3 and to a lesser extent during step 6;
you can see the massive jump in key lookups, to almost 600,000, in about a one-minute span (correlating to step 5 – the traditional approach with a phone book-style index).
In a future test – as with my previous post on GUIDs – I'd like to test this on a system where the data doesn't fit into memory (easy to simulate) and where the disks are slow (not so easy to simulate), since some of these results probably do benefit from things not every production system has – fast disks and sufficient RAM. I also should expand the tests to include more variations (using skinny and wide columns, skinny and wide indexes, a phone book index that actually covers all of the output columns, and sorting in both directions). Scope creep definitely limited the extent of my testing for this first set of tests.

## Conclusion

Pagination doesn't always have to be painful; SQL Server 2012 certainly makes the syntax easier, but if you just plug the native syntax in, you might not always see a great benefit. Here I have shown that slightly more verbose syntax using a CTE can lead to much better performance in the best case, and arguably negligible performance differences in the worst case. By separating data location from data retrieval into two different steps, we can see a tremendous benefit in some scenarios, outside of higher CXPACKET waits in one case (and even then, the parallel queries finished faster than the other queries displaying little or no waits, so they were unlikely to be the "bad" CXPACKET waits everyone warns you about).

Still, even the faster method is slow when there is no supporting index. While you may be tempted to implement an index for every possible sorting algorithm a user might choose, you may want to consider providing fewer options (since we all know that indexes aren't free). For example, does your application absolutely need to support sorting by LastName ascending _and_ LastName descending? If they want to go directly to the customers whose last names start with Z, can't they go to the _last_ page and work backward? That's a business and usability decision more than a technical one, just keep it as an option before slapping indexes on every sort column, in both directions, in order to get the best performance for even the most obscure sorting options.
