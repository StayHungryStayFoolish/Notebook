# Select Advance

## Keywords syntax

| Syntax                     | Description                                                                                                                       |
|:---------------------------|:----------------------------------------------------------------------------------------------------------------------------------|
| AS                         | Aliases are used to give a table, or a column in a table, a temporary name.                                                       |
| DISTINCT                   | Used to return only different values.                                                                                             |
| WHERE                      | Used to extract only those records that fulfill a specified condition.                                                            |
| GROUP BY                   | Groups rows that have the same values into summary rows.                                                                          |
| HAVING                     | The HAVING clause was added to SQL because the WHERE keyword cannot be used with `COUNT(), SUM(), AVG(), MAX(), MIN()` functions. |
| ORDER BY                   | Used to sort the result-set in ascending or descending order.                                                                     |
| LIMIT `n_1` [OFFSET `n_2`] | Specify the number of records(offset the numbers) to return.                                                                      |
| INNER JOIN ... ON ...      | Selects records that have matching values in `both` tables.                                                                       |
| LEFT JOIN ... ON ...       | Returns all records from the `left table`, and the matching records (if any) from the right table.                                |
| RIGHT JOIN ... ON ...      | Returns all records from the `right table`, and the matching records (if any) from the left table.                                |

### AS

1. Alias Column 

```sql
SELECT `column_name` AS `alias_name`
FROM `table_name`;
```

2. Alias Table

```sql
SELECT `column_name`
FROM `table_name` AS `alias_name`;
```

### DISTINCT

```sql
SELECT DISTINCT column_1, column_2, column_3, column_n 
FROM `table_name`;
```

### WHERE

**Operators in The WHERE Clause**

| Operator  | Example                                         |
|:----------|:------------------------------------------------|
| `=`       | `WHERE` column = value                          |
| `>`       | `WHERE` column > value                          |
| `<`       | `WHERE` column < value                          |
| `>=`      | `WHERE` column >= value                         |
| `<=`      | `WHERE` column <= value                         |
| `<>`      | `WHERE` column <> value                         |
| `BETWEEN` | `WHERE` column `BETWEEN` value_1 `AND` value_2  |
| `LIKE`    | `WHERE` column `LIKE` `regex_pattern`           |
| `IN`      | `WHERE` column `IN` (value_1, value_2, value_n) |

```sql
SELECT column_1, column_2, column_3, column_n 
FROM `table_name`
WHERE condition;
```

### GROUP BY

*GROUP BY clause often used with the aggregation functions(`COUNT(), SUM(), AVG(), MAX(), MIN()`) to group the result-set by one or more columns.*

> GROUP BY often used the columns declared by SELECT 

1. GROUP BY

```sql
SELECT column_1, column_2, column_3, column_n 
FROM `table_name`
WHERE condition
GROUP BY `column_name(s)`
```

2. GROUP BY with aggregation function

```sql
SELECT COUNT(column_1), column_2
FROM `table_name`
GROUP BY `column_name(s)`;
```

### HAVING

*HAVING clause often used with the aggregation functions(`COUNT(), SUM(), AVG(), MAX(), MIN()`).*

```sql
SELECT column_1, column_2, column_3, column_n
FROM `table_name`
[WHERE condition]
GROUP BY `column_name(s)`
HAVING condition
ORDER BY `column_name(s)`;
```

Example:

```sql
SELECT COUNT(column_1), column_2 
FROM `table_name`
GROUP BY `column_2`
HAVING COUNT(`column_1`) > 5;
```

### ORDER BY

*Default ASC*

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
ORDER BY column_1, column_2, column_n ASC|DESC;
```

### LIMIT

```sql
SELECT column_1, column_2, column_3, column_n
FROM `table_name`
WHERE condition
LIMIT `n_1` [OFFSET `n_2`];
```

### JOIN

![MySQL JOIN](https://raw.githubusercontent.com/StayHungryStayFoolish/notebook-img/master/img/MySQL/mysql_join.png?raw=true)

Table: `Customers`

| customer_id | customer_name | email           | country  |
|:------------|:--------------|:----------------|:---------|
| 1           | Alfred        | alfred@***.com  | America  |
| 2           | Ana           | ana@***.com     | France   |
| 3           | Antonio       | antonio@***.com | Mexico   |

Table: `Orders`

| order_id | customer_id | order_date      |
|:---------|:------------|:----------------|
| 20220101 | 1           | alfred@***.com  |
| 20220102 | 2           | ana@***.com     |
| 20220103 | 3           | antonio@***.com |

The relationship between the two tables above is the "customer_id" column.

#### INNER JOIN...ON...

```sql
SELECT column_1, column_2, column_3, column_n
FROM `table_1`
INNER JOIN `table_2`
ON table_1.column_name = table_2.column_name
[WHERE condition];
```

#### LEFT JOIN...ON...

```sql
SELECT column_1, column_2, column_3, column_n
FROM `table_1`
LEFT JOIN `table_2`
ON table_1.column_name = table_2.column_name
[WHERE condition];
```

#### RIGHT JOIN...ON...

```sql
SELECT column_1, column_2, column_3, column_n
FROM `table_1`
RIGHT JOIN `table_2`
ON table_1.column_name = table_2.column_name
[WHERE condition];
```

## SELECT Statement

```sql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
    [HIGH_PRIORITY]
    [STRAIGHT_JOIN]
    [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
    [SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr] ...
    [into_option]
    [FROM table_references
      [PARTITION partition_list]]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]]
    [HAVING where_condition]
    [WINDOW window_name AS (window_spec)
        [, window_name AS (window_spec)] ...]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [into_option]
    [FOR {UPDATE | SHARE}
        [OF tbl_name [, tbl_name] ...]
        [NOWAIT | SKIP LOCKED]
      | LOCK IN SHARE MODE]
    [into_option]

into_option: {
    INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
  | INTO DUMPFILE 'file_name'
  | INTO var_name [, var_name] ...
}
```