---
layout: post
title: 'How to Calculate Multiple Aggregate Functions in a Single Query'
date: 2019-06-10 10:34:00 +0700
categories: [database]
tags: [sql, aggregate, query]
image: place_holder.jpeg
---

At a customer site, I’ve recently encountered a report where a programmer needed to count quite a bit of stuff from a single table. The counts all differed in the way they used specific predicates. The report looked roughly like this (as always, [I’m using the Sakila database for illustration](https://www.jooq.org/sakila)):

```sql
-- Total number of films
SELECT count(*)
FROM film

-- Number of films with a given length
SELECT count(*)
FROM film
WHERE length BETWEEN 120 AND 150

-- Number of films with a given language
SELECT count(*)
FROM film
WHERE language_id = 1

-- Number of films for a given rating
SELECT count(*)
FROM film
WHERE rating = 'PG'
```

And then, unsurprisingly, combinations of these predicates were needed as well, i.e.

```sql
-- Number of films with a given length / language_id
SELECT count(*)
FROM film
WHERE length BETWEEN 120 AND 150
AND language_id = 1

-- Number of films with a given length / rating
SELECT count(*)
FROM film
WHERE length BETWEEN 120 AND 150
AND rating = 'PG'

-- Number of films with a given language_id / rating
SELECT count(*)
FROM film
WHERE language_id = 1
AND rating = 'PG'

-- Number of films with a given length / language_id / rating
SELECT count(*)
FROM film
WHERE length BETWEEN 120 AND 150
AND language_id = 1
AND rating = 'PG'
```

In the end, there were 32 queries in total (or 8 in my example) with all the possible combinations of predicates. Needless to say that running them all took quite a while, because the table had around 200M records and only one predicate could profit from an index.

But in fact, the improvement is really easy. There are several options to calculate all these counts in a single query

## Simplest solution works in all databases: Filtered aggregate functions (or manual pivot)

This solution allows for calculating all results in a single query by using 8 different, explicit, [filtered aggregate functions](https://blog.jooq.org/2014/12/30/the-awesome-postgresql-9-4-sql2003-filter-clause-for-aggregate-functions/) and no [GROUP BY clause](https://blog.jooq.org/2014/12/04/do-you-really-understand-sqls-group-by-and-having-clauses/) (none in this example. More complex cases where GROUP BY persists are sill imaginable).

This is how it works on all databases:

```sql
SELECT
  count(*),
  count(length),
  count(language_id),
  count(rating),
  count(length + language_id),
  count(length + rating),
  count(language_id + rating),
  count(length + language_id + rating)
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
    CASE WHEN language_id = 1            THEN 1 END language_id,
    CASE WHEN rating = 'PG'              THEN 1 END rating
  FROM film
) film
```

Which yields:

```
col1  col2  col3  col4  col5  col6  col7  col8
1000  224   1000  194   224   43    194   43
```

How to read the above query?

Instead of evaluating the three different predicates in a WHERE clause, we pre-calculate it in a derived table (subquery in the FROM clause) and translate the predicate in some random value (e.g. 1) if TRUE and NULL if FALSE. Note, I omitted the ELSE clause from the CASE expression, which means that we get NULLs per default. Running the nested select on its own…

```sql
SELECT
  CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
  CASE WHEN language_id = 1            THEN 1 END language_id,
  CASE WHEN rating = 'PG'              THEN 1 END rating
FROM film
```

… yields something along the lines of:

```
length  language_id  rating
---------------------------
NULL    1            1
NULL    1            NULL
NULL    1            NULL
NULL    1            NULL
1       1            NULL
NULL    1            1
NULL    1            NULL
...
```

(Note, of course, we could have used actual BOOLEAN types, e.g. in PostgreSQL, but that wouldn’t work on all databases)

Now, in the outer query, we’re using once COUNT(\*), which simply counts all the rows regardless of any predicates in the CASE expressions. The other COUNT(expr) aggregate functions do something that surprisingly few people are aware of (yet a lot of people use this form “by accident”). They count only the number of non-NULL rows. For instance:

```sql
SELECT
  ...
  count(length),
  ...
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
    ...
  FROM film
) film
```

Or also:

```sql
SELECT
  count(CASE WHEN length BETWEEN 120 AND 150 THEN 1 END)
FROM
  film
```

These queries will count those films whose length is BETWEEN 120 AND 150 (because those rows produce the value 1, which is non-NULL, and thus counted), whereas all the other films are not being counted.

Finally, I just used a trick to combine nullable values to make sure they’re all non-NULL:

```sql
SELECT
  ...
  count(length + language_id),
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
    CASE WHEN language_id = 1            THEN 1 END language_id,
    ...
  FROM film
) film
```

This counts those rows whose length BETWEEN 120 AND 150 and whose language_id = 1, because if either predicate was FALSE, the number would be NULL and thus the sum is NULL as well.

**PostgreSQL and HSQLDB variant: FILTER**

In PostgreSQL and HSQLDB (and in the SQL standard), there’s a special syntax for this. We can use the FILTER clause instead of encoding values in NULL / non-NULL like this:

```sql
SELECT
  count(*),
  count(*) FILTER (WHERE length IS NOT NULL),
  count(*) FILTER (WHERE language_id IS NOT NULL),
  count(*) FILTER (WHERE rating IS NOT NULL),
  count(*) FILTER (WHERE length + language_id IS NOT NULL),
  count(*) FILTER (WHERE length + rating IS NOT NULL),
  count(*) FILTER (WHERE language_id + rating IS NOT NULL),
  count(*) FILTER (
    WHERE length + language_id + rating IS NOT NULL)
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
    CASE WHEN language_id = 1            THEN 1 END language_id,
    CASE WHEN rating = 'PG'              THEN 1 END rating
  FROM film
) film
```

Or even, writing out the entire predicates again:

```sql
SELECT
  count(*),
  count(*) FILTER (WHERE length BETWEEN 120 AND 150),
  count(*) FILTER (WHERE language_id = 1),
  count(*) FILTER (WHERE rating = 'PG'),
  count(*) FILTER (
    WHERE length BETWEEN 120 AND 150 AND language_id = 1),
  count(*) FILTER (
    WHERE length BETWEEN 120 AND 150 AND rating = 'PG'),
  count(*) FILTER (
    WHERE language_id = 1 AND rating = 'PG'),
  count(*) FILTER (
    WHERE length BETWEEN 120 AND 150
    AND language_id = 1 AND rating = 'PG')
FROM film
```

Usually, the FILTER clause is more convenient, but both approaches are equivalent, and we’re running only a single query!

I also call this “manual PIVOT“, because it really works like a PIVOT table. And the good news is… There is a PIVOT syntax!

## A more fancy solution: PIVOT

This solution is vendor-specific and only works in Oracle and with a bit less features in SQL Server. Here’s the Oracle version:

```sql
SELECT
  a + b + c + d + e + f + g + h,
                  e + f + g + h,
          c + d         + g + h,
      b     + d     + f     + h,
                          g + h,
                      f     + h,
              d             + h,
                              h
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150
         THEN 1 ELSE 0 END length,
    CASE WHEN language_id = 1
         THEN 1 ELSE 0 END language_id,
    CASE WHEN rating = 'PG'
         THEN 1 ELSE 0 END rating
  FROM film
) film
PIVOT (
  count(*) FOR (length, language_id, rating) IN (
    (0, 0, 0) AS a,
    (0, 0, 1) AS b,
    (0, 1, 0) AS c,
    (0, 1, 1) AS d,
    (1, 0, 0) AS e,
    (1, 0, 1) AS f,
    (1, 1, 0) AS g,
    (1, 1, 1) AS h
  )
)
```

How to read this solution? There are 3 steps:

**Step 1: The derived table**

As in the previous example, we’re translating the desired predicates for our report into three columns that produce values 1 and 0. That’s understood so I won’t repeat the explanation.

**Step 2: The PIVOT clause**

The PIVOT clause can be applied to a table expression to “pivot” it in a similar way as we’re used from Microsoft Excel’s powerful pivot tables. It takes three parts:

- A list of aggregate functions
- An expression (FOR clause)
- A list of expected values (IN clause)

The resulting table expression groups the PIVOT‘s input table by all the remaining columns (i.e. all the columns that are not part of the FOR clause, in our example, that’s no columns), and aggregates all the aggregate functions (in our case, only one) for all the values in the IN list.

If we SELECT \* from this PIVOT table:

```sql
SELECT *
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150
         THEN 1 ELSE 0 END length,
    CASE WHEN language_id = 1
         THEN 1 ELSE 0 END language_id,
    CASE WHEN rating = 'PG'
         THEN 1 ELSE 0 END rating
  FROM film
) film
PIVOT (
  count(*) FOR (length, language_id, rating) IN (
    (0, 0, 0) AS a,
    (0, 0, 1) AS b,
    (0, 1, 0) AS c,
    (0, 1, 1) AS d,
    (1, 0, 0) AS e,
    (1, 0, 1) AS f,
    (1, 1, 0) AS g,
    (1, 1, 1) AS h
  )
)
```

… we’ll get these values:

```
a    b    c    d    e    f    g    h
------------------------------------
0    0  625  151    0    0  181   43
```

As you can see, the column names are generated from the IN list of expected values and the values contained in these columns are aggregations for the different predicates. These aggregations are not exactly the ones we wanted. For instance, column G is all the films whose length BETWEEN 120 AND 150 and whose language_id = 1 and whose RATING != 'PG'.

**Step 3: Summing the count values**

So, in order to get the expected results, we have to sum all the partial counts as such:

```sql
SELECT
  a + b + c + d + e + f + g + h,
                  e + f + g + h,
          c + d         + g + h,
      b     + d     + f     + h,
                          g + h,
                      f     + h,
              d             + h,
                              h
FROM
  ...
```

The result is now the same.

## A more fancy solution: GROUPING SETS

[GROUPING SETS are a SQL standard](https://blog.jooq.org/2011/11/26/group-by-rollup-cube/) and they’re supported in at least:

- DB2
- HANA
- Oracle
- PostgreSQL
- SQL Server
- Sybase SQL Anywhere

Simply put, GROUPING SETS allow for grouping a table several times and creating a UNION of all the results. For example, the following two queries are the same, conceptually, although the GROUPING SETS one is usually faster:

```sql
-- Grouping once by language_id, then by rating
SELECT language_id, rating, count(*)
FROM film
GROUP BY GROUPING SETS (
  (language_id),
  (rating)
)

-- Grouping first by language_id
SELECT language_id, NULL, count(*)
FROM film
GROUP BY language_id
UNION ALL
SELECT NULL, rating, count(*)
FROM film
GROUP BY rating
```

Both queries yield:

```
language_id   rating   count
          1             1000 -- First grouping set / union subquery
              G          178 \
              PG         194  |
              PG-13      223  | Second grouping set / union subquery
              R          195  |
              NC-17      210 /


```

Clearly, the GROUPING SETS variant is more concise. Let’s imagine, we’d like to add more combinations of grouping columns, e.g.

```sql
SELECT language_id, rating, count(*)
FROM film
GROUP BY GROUPING SETS (
  (),
  (language_id),
  (rating),
  (language_id, rating)
)
```

Now, we’re grouping by all the combinations of columns, and the result is:

```
language_id   rating   count
                        1000 -- First grouping set: ()
          1             1000 -- Second grouping set: (language_id)
              G          178 \
              PG         194  |
              PG-13      223  | Third grouping set: (rating)
              R          195  |
              NC-17      210 /
          1   G          178 \
          1   PG         194  |
          1   PG-13      223  | Fourth grouping set: (language_id, rating)
          1   R          195  |
          1   NC-17      210 /

```

Of course, this would all be more impressive if we had more than one language in the system…

So, how do we solve the original problem with GROUPING SETS? Here’s how:

```sql
SELECT
  GROUPING_ID (length, language_id, rating),
  length,
  language_id,
  rating,
  count(*)
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150
         THEN 1 ELSE 0 END length,
    CASE WHEN language_id = 1
         THEN 1 ELSE 0 END language_id,
    CASE WHEN rating = 'PG'
         THEN 1 ELSE 0 END rating
  FROM film
) film
GROUP BY GROUPING SETS (
  (),
  (length),
  (language_id),
  (rating),
  (length, language_id),
  (length, rating),
  (rating, language_id),
  (length, language_id, rating)
)
HAVING COALESCE (length, 1) != 0
AND COALESCE (language_id, 1) != 0
AND COALESCE (rating, 1) != 0
ORDER BY GROUPING_ID (length, language_id, rating) DESC
```

Wow. How to read this? In 4 steps:

**Step 1: Again, the derived table**

This time, we’ll encode FALSE as 0, not NULL, because NULL already has a different meaning in GROUPING SETS. It means that for a given GROUPING SET, we didn’t group by that column. We’ll see that in step 3.

**Step 2: The GROUPING SETS**

In this section, we’re just listing all the possible combinations of GROUP BY columns that we want to use, which produces 8 distinct GROUPING SETS. I’ve already explained this in the previous introduction to GROUPING SETS, so this is no different.

**Step 3: Filter out unwanted groupings**

Just like in the PIVOT example, we’re also getting results for which the predicates are FALSE, but we don’t want those in the result. So we’re filtering them out in the HAVING clause:

```sql
SELECT
  ...
HAVING COALESCE (length, 1) != 0
AND COALESCE (language_id, 1) != 0
AND COALESCE (rating, 1) != 0
...
```

How to read this? E.g. LENGTH can be any of:

- 1: The length predicate was TRUE
- 0: The length predicate was FALSE
- NULL: The length column is not considered for a given GROUPING SET, e.g. () or (rating, language_id)

So, using COALESCE, we’re making sure that we include only 1 and NULL lengths, not 0 lengths.

**Step 4: Ordering the results**

This is optional, but in order to get the same output order as before, we can use the special GROUPING_ID() (or GROUPING() depending on the DB) function which returns an ID for each GROUPING SET. The output is:

```
grouping   length   language_id   rating   count
------------------------------------------------
       7     NULL          NULL     NULL    1000
       6     NULL          NULL        1     194
       5     NULL             1     NULL    1000
       4     NULL             1        1     194
       3        1          NULL     NULL     224
       2        1          NULL        1      43
       1        1             1     NULL     224
       0        1             1        1      43

```

Excellent! And hey, there’s even syntax sugar for “special” GROUPING SETS configurations like ours, where we list all the possible column permutations. In this case, we can use CUBE()!

```sql
SELECT
  GROUPING_ID (length, language_id, rating),
  length,
  language_id,
  rating,
  count(*)
FROM (
  SELECT
    CASE WHEN length BETWEEN 120 AND 150
         THEN 1 ELSE 0 END length,
    CASE WHEN language_id = 1
         THEN 1 ELSE 0 END language_id,
    CASE WHEN rating = 'PG'
         THEN 1 ELSE 0 END rating
  FROM film
) film
GROUP BY CUBE (length, language_id, rating)
HAVING COALESCE(length, 1) != 0
AND COALESCE(language_id, 1) != 0
AND COALESCE(rating, 1) != 0
ORDER BY GROUPING_ID (length, language_id, rating) DESC
```

## Performance

Such a comparison blog post wouldn’t be complete if we wouldn’t benchmark for performance. This time, I’ll be benchmarking only for Oracle, as PostgreSQL doesn’t support PIVOT and SQL Server’s PIVOT is more limited than Oracle’s.

Here’s the complete benchmark:

```sql
SET SERVEROUTPUT ON
DECLARE
  v_ts TIMESTAMP WITH TIME ZONE;
  v_repeat CONSTANT NUMBER := 2000;
BEGIN

  -- Repeat the whole benchmark several times to avoid warmup penalty
  FOR r IN 1..5 LOOP

    -- Individual statements
    v_ts := SYSTIMESTAMP;

    FOR i IN 1..v_repeat LOOP
      FOR rec IN (
        SELECT count(*) FROM film
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE length BETWEEN 120 AND 150
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE language_id = 1
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE rating = 'PG'
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE length BETWEEN 120 AND 150
        AND language_id = 1
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE length BETWEEN 120 AND 150
        AND rating = 'PG'
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE language_id = 1
        AND rating = 'PG'
      ) LOOP
        NULL;
      END LOOP;

      FOR rec IN (
        SELECT count(*) FROM film
        WHERE length BETWEEN 120 AND 150
        AND language_id = 1
        AND rating = 'PG'
      ) LOOP
        NULL;
      END LOOP;
    END LOOP;

    dbms_output.put_line('Run ' || r ||', Statement 2 : ' || (SYSTIMESTAMP - v_ts));

    -- Manual PIVOT
    v_ts := SYSTIMESTAMP;

    FOR i IN 1..v_repeat LOOP
      FOR rec IN (
        SELECT
          count(*),
          count(length),
          count(language_id),
          count(rating),
          count(length + language_id),
          count(length + rating),
          count(language_id + rating),
          count(length + language_id + rating)
        FROM (
          SELECT
            CASE WHEN length BETWEEN 120 AND 150 THEN 1 END length,
            CASE WHEN language_id = 1            THEN 1 END language_id,
            CASE WHEN rating = 'PG'              THEN 1 END rating
          FROM film
        ) film
      ) LOOP
        NULL;
      END LOOP;
    END LOOP;

    dbms_output.put_line('Run ' || r ||', Statement 2 : ' || (SYSTIMESTAMP - v_ts));

    -- PIVOT
    v_ts := SYSTIMESTAMP;

    FOR i IN 1..v_repeat LOOP
      FOR rec IN (
        SELECT
          a + b + c + d + e + f + g + h,
                          e + f + g + h,
                  c + d         + g + h,
              b     + d     + f     + h,
                                  g + h,
                              f     + h,
                      d             + h,
                                      h
        FROM (
          SELECT
            CASE WHEN length BETWEEN 120 AND 150 THEN 1 ELSE 0 END length,
            CASE WHEN language_id = 1            THEN 1 ELSE 0 END language_id,
            CASE WHEN rating = 'PG'              THEN 1 ELSE 0 END rating
          FROM film
        ) film
        PIVOT (
          count(*) FOR (length, language_id, rating) IN (
            (0, 0, 0) AS a,
            (0, 0, 1) AS b,
            (0, 1, 0) AS c,
            (0, 1, 1) AS d,
            (1, 0, 0) AS e,
            (1, 0, 1) AS f,
            (1, 1, 0) AS g,
            (1, 1, 1) AS h
          )
        )
      ) LOOP
        NULL;
      END LOOP;
    END LOOP;

    dbms_output.put_line('Run ' || r ||', Statement 3 : ' || (SYSTIMESTAMP - v_ts));

    -- GROUPING SETS
    v_ts := SYSTIMESTAMP;

    FOR i IN 1..v_repeat LOOP
      FOR rec IN (
        SELECT
          GROUPING_ID (length, language_id, rating),
          length,
          language_id,
          rating,
          count(*)
        FROM (
          SELECT
            CASE WHEN length BETWEEN 120 AND 150 THEN 1 ELSE 0 END length,
            CASE WHEN language_id = 1            THEN 1 ELSE 0 END language_id,
            CASE WHEN rating = 'PG'              THEN 1 ELSE 0 END rating
          FROM film
        ) film
        GROUP BY CUBE (length, language_id, rating)
        HAVING COALESCE (length, 1) != 0
        AND COALESCE (language_id, 1) != 0
        AND COALESCE (rating, 1) != 0
        ORDER BY GROUPING_ID (length, language_id, rating) DESC
      ) LOOP
        NULL;
      END LOOP;
    END LOOP;

    dbms_output.put_line('Run ' || r ||', Statement 4 : ' || (SYSTIMESTAMP - v_ts));
  END LOOP;
END;
/
```

And the results:

```
Run 1, Statement 1 : +000000000 00:00:01.928497000
Run 1, Statement 2 : +000000000 00:00:01.136341000
Run 1, Statement 3 : +000000000 00:00:02.751679000
Run 1, Statement 4 : +000000000 00:00:00.797529000

Run 2, Statement 1 : +000000000 00:00:01.695543000
Run 2, Statement 2 : +000000000 00:00:01.004073000
Run 2, Statement 3 : +000000000 00:00:02.490895000
Run 2, Statement 4 : +000000000 00:00:00.838979000

Run 3, Statement 1 : +000000000 00:00:01.634047000
Run 3, Statement 2 : +000000000 00:00:01.016266000
Run 3, Statement 3 : +000000000 00:00:02.566895000
Run 3, Statement 4 : +000000000 00:00:00.790159000

Run 4, Statement 1 : +000000000 00:00:01.669844000
Run 4, Statement 2 : +000000000 00:00:01.015502000
Run 4, Statement 3 : +000000000 00:00:02.574646000
Run 4, Statement 4 : +000000000 00:00:00.807804000

Run 5, Statement 1 : +000000000 00:00:01.653498000
Run 5, Statement 2 : +000000000 00:00:00.980375000
Run 5, Statement 3 : +000000000 00:00:02.556186000
Run 5, Statement 4 : +000000000 00:00:00.890283000
```

Very disappointingly, the PIVOT solution is the slowest every time. I’m assuming there’s some substantial temporary object overhead which wouldn’t be as severe if the table were much larger, but clearly, the manual PIVOT solution (COUNT(CASE ...)) and the GROUPING SETS solution heavily outperform the initial attempt, where we calculate 8 counts individually.

To get back to the original report where 32 counts were calculated: The report was roughly 20x as fast with manual PIVOT on 200M rows and imagine if you need to JOIN – you definitely want to avoid those 32 individual queries and calculate everything in one go.

Cheers!

Source: [How to Calculate Multiple Aggregate Functions in a Single Query](https://blog.jooq.org/2017/04/20/how-to-calculate-multiple-aggregate-functions-in-a-single-query/)
