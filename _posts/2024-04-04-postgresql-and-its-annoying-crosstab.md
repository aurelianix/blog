---
layout: post
title: 'PostgreSQL and its annoying crosstab'
date: 2024-04-04 08:38 +0200
categories:
- technology
tags:
- postgres
- TIL
---
Today, I had to pivot (pun intended) from my usual tasks to help a colleague with a query. The task is deceptively simple: Collect metadata about all columns of a table in a single query. This was to be a function in PostgreSQL that would return a table with the following columns:

- `table_name`
- `column_name`
- `data_type`
- `total_tows`
- `not_null_count`
- `unique_count`
- `max_value` (for integers)
- `min_value` (for integers)
- `avg_value` (for integers)
- `max_length` (for strings)
- `min_length` (for strings)
- `avg_length` (for strings)
- `space_count_max` (for strings)
- `space_count_min` (for strings)
- `space_count_avg` (for strings)
- `max_date` (for dates)
- `min_date` (for dates)

You can can imagine how this would be slow if you did the query for each column individually since it requires a full table scan if we have no index on the column.

Let's create a test table:

```sql
CREATE TABLE my_table
(
    string_col text,
    int_col    int
);
-- create 1000 rows with strings '1 2 ... 10' or null for string_col
-- create 1000 rows with random values between 1 and 100 or null for int_col
INSERT INTO my_table
SELECT CASE
           WHEN RANDOM() < 0.9 THEN (SELECT STRING_AGG(words::text, ' ') FROM GENERATE_SERIES(1, 10) words)
           ELSE NULL END,
       CASE WHEN RANDOM() < 0.9 THEN FLOOR(RANDOM() * 100) ELSE NULL END
FROM GENERATE_SERIES(1, 1000);
```

## Get the stats, in wide format

The first part of the solution was to build a query that would return the metadata wide, something like (shortened for brevity):

```sql
SELECT
        COUNT(*) total_rows,
        -- stats for int_col
        COUNT(int_col) int_col_null_count,
        COUNT(DISTINCT int_col) int_col_unique_count,
        MAX(int_col) int_col_max_value,
        MIN(int_col) int_col_min_value,
        AVG(int_col) int_col_avg_value,
        -- stats for string_col
        COUNT(string_col) string_col_null_count,
        COUNT(DISTINCT string_col) string_col_unique_count,
        MAX(LENGTH(string_col)) string_col_max_length,
        MIN(LENGTH(string_col)) string_col_min_length,
        AVG(LENGTH(string_col)) string_col_avg_length,
        MAX(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_max,
        MIN(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_min,
        AVG(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_avg
FROM my_table;
```


## Get the stats, in tall format

The idea is to gather the stats of the table in a single go and then build some sort of pivot table from it. Since we are using dynamic SQL, it's not too hard. We can use a lateral join to make the data tall:

```sql
WITH stats AS (
    SELECT
        COUNT(*) total_rows,
        -- stats for int_col
        COUNT(int_col) int_col_not_null_count,
        COUNT(DISTINCT int_col) int_col_unique_count,
        MAX(int_col) int_col_max_value,
        MIN(int_col) int_col_min_value,
        AVG(int_col) int_col_avg_value,
        -- stats for string_col
        COUNT(string_col) string_col_not_null_count,
        COUNT(DISTINCT string_col) string_col_unique_count,
        MAX(LENGTH(string_col)) string_col_max_length,
        MIN(LENGTH(string_col)) string_col_min_length,
        AVG(LENGTH(string_col)) string_col_avg_length,
        MAX(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_max,
        MIN(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_min,
        AVG(LENGTH(string_col) - LENGTH(REPLACE(string_col, ' ', ''))) string_col_space_count_avg
    FROM my_table
)
SELECT l.* FROM stats
    CROSS JOIN LATERAL(
        VALUES
        -- int_col stats
        ('int_col', 'total_rows', total_rows::text),
        ('int_col', 'not_null_count', int_col_not_null_count::text),
        ('int_col', 'unique_count', int_col_unique_count::text),
        ('int_col', 'max_value', int_col_max_value::text),
        ('int_col', 'min_value', int_col_min_value::text),
        ('int_col', 'avg_value', int_col_avg_value::text),
        -- string_col stats
        ('string_col', 'total_rows', total_rows::text),
        ('string_col', 'not_null_count', string_col_not_null_count::text),
        ('string_col', 'unique_count', string_col_unique_count::text),
        ('string_col', 'max_length', string_col_max_length::text),
        ('string_col', 'min_length', string_col_min_length::text),
        ('string_col', 'avg_length', string_col_avg_length::text),
        ('string_col', 'space_count_max', string_col_space_count_max::text),
        ('string_col', 'space_count_min', string_col_space_count_min::text),
        ('string_col', 'space_count_avg', string_col_space_count_avg::text)
    ) AS l(column_name, meta_key, value);
```

This will yield results that look like this:

| column\_name | meta\_key | value |
| :--- | :--- | :--- |
| int\_col | total\_rows | 1000 |
| int\_col | not\_null\_count | 904 |
| int\_col | unique\_count | 100 |
| int\_col | max\_value | 99 |
| int\_col | min\_value | 0 |
| int\_col | avg\_value | 50.4115044247787611 |
| string\_col | total\_rows | 1000 |
| string\_col | not\_null\_count | 915 |
| string\_col | unique\_count | 1 |
| string\_col | max\_length | 20 |
| string\_col | min\_length | 20 |
| string\_col | avg\_length | 20.0000000000000000 |
| string\_col | space\_count\_max | 9 |
| string\_col | space\_count\_min | 9 |
| string\_col | space\_count\_avg | 9.0000000000000000 |


## Crosstab the data into a table

Now, we need to pivot this data. PostgreSQL's crosstab is what we need, but getting it to work is a bit of a pain. The first thing we need to do is to install the `tablefunc` extension:

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
```

`crosstab` takes a string as argument with the query that will return the data to pivot. We can use behemoth of generated SQL that we had above (just 10 times bigger, because what table has only 2 columns?). The query will look like this:

```sql
SELECT * FROM crosstab('SELECT column_name, meta_key, value FROM ([GIANT SUBQUERY HERE]) t ORDER BY 1, 2') AS ct (
    column_name text,
    total_rows text,
    not_null_count text,
    unique_count text,
    max_value text,
    min_value text,
    avg_value text,
    max_length text,
    min_length text,
    avg_length text,
    space_count_max text,
    space_count_min text,
    space_count_avg text
);
```

Now let's look at the results:

| column\_name | total\_rows | not\_null\_count | unique\_count | max\_value | min\_value | avg\_value | max\_length | min\_length | avg\_length | space\_count\_max | space\_count\_min | space\_count\_avg |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| int\_col | 1000 | 904 | 100 | 99 | 0 | 50.4115044247787611 | null | null | null | null | null | null |
| string\_col | 1000 | 915 | 1 | 20 | 20 | 20.0000000000000000 | 9 | 9 | 9.0000000000000000 | null | null | null |


Uh... WTF? THis is not what we wanted. The stats for `string_col` moved to the left. It didn't pivot by name, it just filled up the columns. When using `crosstab`, the first column is the one that will be pivoted. If we want to pivot into a table that has more columns than each value, (here, e.g. the `int_col` will pivot into 6 columns, but `string_col` will pivot into 9 columns), we need to supply crosstab with a second parameter. It is also recommended to order the rows, so that each parameter will be in order

```sql
SELECT * FROM crosstab(
    'SELECT column_name, meta_key, value, row_number() OVER (PARTITION BY column_name ORDER BY meta_key) FROM ([GIANT SUBQUERY HERE]) t ORDER BY 1, 2',
    -- list of all result columns, sorted by name
    'VALUES
        (''avg_length''),
        (''avg_value''),
        (''max_length''),
        (''max_value''),
        (''min_length''),
        (''min_value''),
        (''not_null_count''),
        (''space_count_avg''),
        (''space_count_max''),
        (''space_count_min''),
        (''total_rows''),
        (''unique_count'')
    ') AS ct (
    -- MUST BE IN SAME ORDER AS THE LIST ABOVE
    column_name text,
    avg_length text,
    avg_value text,
    max_length text,
    max_value text,
    min_length text,
    min_value text,
    not_null_count text,
    space_count_avg text,
    space_count_max text,
    space_count_min text,
    total_rows text,
    unique_count text
);
```

Let's have a look at the results:

| column\_name | avg\_length | avg\_value | max\_length | max\_value | min\_length | min\_value | not\_null\_count | space\_count\_avg | space\_count\_max | space\_count\_min | total\_rows | unique\_count |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| int\_col | null | 50.4115044247787611 | null | 99 | null | 0 | 904 | null | null | null | 1000 | 100 |
| string\_col | 20.0000000000000000 | null | 20 | null | 20 | null | 915 | 9.0000000000000000 | 9 | 9 | 1000 | 1 |


Ahhh, this looks more like it.

This approach has a couple of drawbacks:

1. All data types must be converted to text because our tall format requires static types.
2. Crosstab is really annoying and easy to get wrong
3. You can't include more columns than you have in the list of columns to pivot into. The crosstab function WILL complain if there is a column in the result that has no value in the pivot list.

But, you know what? It works. It's good enough for me at the moment. If you have _any_ suggestion on how to improve this, PLEASE let me know.

PS: I learned that you can also escape multiline strings in postgres using dollar-quoted strings.

```sql
SELECT $$This is
a long string with 'quotes'$$;
```

Ahh, feels so much better.