# MySQL 数据类型

## RDBMS(Relational Database Management System)

A relational database defines database relationships in the form of tables.

## Table Structure

> Table Name: aws_monthly_cost

| id  | invoice_id | product_name          | usage_start_date       | usage_end_date            | total_cost     |
|:----|:-----------|:----------------------|:-----------------------|:--------------------------|:---------------|
| 1   | Estimated  | AWS Data Transfer     | `2022-04-01 00:00:00`  | `2022-04-30 23:59:59`     | 17.32          |
| 2   | Estimated  | Amazon Athena         | `2022-04-01 00:00:00`  | `2022-04-30 23:59:59`     | 244.80         |
| 3   | Estimated  | Amazon QuickSight     | `2022-04-01 00:00:00`  | `2022-04-30 23:59:59`     | 3.44           |

## String Data Types

| Data Definition             | Description                                                                                                                                                                                                                                                             |
|:----------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CHAR(size)                  | A FIXED length string (can contain letters, numbers, and special characters). The size parameter specifies the column length in characters - can be from 0 to 255. Default is 1                                                                                         |
| BINARY(size)                | Equal to CHAR(), but stores binary byte strings. The size parameter specifies the column length in bytes. Default is 1                                                                                                                                                  |
| `VARCHAR(size)`             | A VARIABLE length string (can contain letters, numbers, and special characters). The size parameter specifies the maximum column length in characters - can be from `0 to 65535`                                                                                        |
| TINYBLOB                    | For BLOBs (Binary Large OBjects). Max length: 255 bytes                                                                                                                                                                                                                 |
| TINYTEXT                    | Holds a string with a maximum length of 255 characters                                                                                                                                                                                                                  |
| TEXT(size)                  | Holds a string with a maximum length of 65,535 bytes                                                                                                                                                                                                                    |
| BLOB(size)                  | For BLOBs (Binary Large OBjects). Holds up to 65,535 bytes of data                                                                                                                                                                                                      |
| MEDIUMTEXT                  | Holds a string with a maximum length of 16,777,215 characters                                                                                                                                                                                                           |
| MEDIUMBLOB                  | For BLOBs (Binary Large OBjects). Holds up to 16,777,215 bytes of data                                                                                                                                                                                                  |
| `LONGTEXT`                  | Holds a string with a maximum length of `4,294,967,295` characters                                                                                                                                                                                                      |
| LONGBLOB                    | For BLOBs (Binary Large OBjects). Holds up to 4,294,967,295 bytes of data                                                                                                                                                                                               |
| ENUM(val1, val2, val3, ...) | A string object that can have only one value, chosen from a list of possible values. You can list up to 65535 values in an ENUM list. If a value is inserted that is not in the list, a blank value will be inserted. The values are sorted in the order you enter them |
| SET(val1, val2, val3, ...)  | A string object that can have 0 or more values, chosen from a list of possible values. You can list up to 64 values in a SET list                                                                                                                                       |

## Numeric Types

| Data Definition      | Description                                                                                                                                                                                                                                                                                    |
|:---------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| BIT                  | A bit-value type. The number of bits per value is specified in size. The size parameter can hold a value from 1 to 64. The default value for size is 1.                                                                                                                                        |
| TINYINT(size)        | A very small integer. Signed range is from -128 to 127. Unsigned range is from 0 to 255. The size parameter specifies the maximum display width (which is 255)                                                                                                                                 |
| `BOOL`               | Zero is considered as false, nonzero values are considered as true.                                                                                                                                                                                                                            |
| BOOLEAN              | Equal to BOOL                                                                                                                                                                                                                                                                                  |
| SMALLINT(size)       | A small integer. Signed range is from -32768 to 32767. Unsigned range is from 0 to 65535. The size parameter specifies the maximum display width (which is 255)                                                                                                                                |
| MEDIUMINT(size)      | A medium integer. Signed range is from -8388608 to 8388607. Unsigned range is from 0 to 16777215. The size parameter specifies the maximum display width (which is 255)                                                                                                                        |
| `INT(size)`          | A medium integer. Signed range is from `-2147483648 to 2147483647`. Unsigned range is from 0 to 4294967295. The size parameter specifies the maximum display width (which is 255)                                                                                                              |
| INTEGER(size)        | Equal to INT(size)                                                                                                                                                                                                                                                                             |
| `BIGINT(size)`       | A large integer. Signed range is from `-9223372036854775808 to 9223372036854775807`. Unsigned range is from 0 to 18446744073709551615. The size parameter specifies the maximum display width (which is 255)                                                                                   |
| `DOUBLE(size, d)`    | A normal-size floating point number. The total number of digits is specified in size. The number of digits after the decimal point is specified in the d parameter                                                                                                                             |
| `DECIMAL(size, d)`   | An exact fixed-point number. The total number of digits is specified in size. The number of digits after the decimal point is specified in the d parameter. The maximum number for size is 65. The maximum number for d is 30. The default value for size is 10. The default value for d is 0. |
| `FLOAT(size, d)`     | A floating point number. The total number of digits is specified in size. The number of digits after the decimal point is specified in the d parameter. This syntax is deprecated in MySQL 8.0.17, and it will be removed in future MySQL versions                                             |

## Date and Time Data Types

| Data Definition        | Description                                                                                                                                                                                                                                                                                                                                                                                                         |
|:-----------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| DATE                   | A date. Format: `YYYY-MM-DD`. The supported range is from `1000-01-01` to `9999-12-31`                                                                                                                                                                                                                                                                                                                              |
| `DATETIME`             | A date and time combination. Format: `YYYY-MM-DD hh:mm:ss`. The supported range is from `1000-01-01 00:00:00` to `9999-12-31 23:59:59`. Adding DEFAULT and ON UPDATE in the column definition to get automatic initialization and updating to the current date and time                                                                                                                                             |
| `TIMESTAMP`            | A timestamp. TIMESTAMP values are stored as the number of seconds since the Unix epoch (`1970-01-01 00:00:00` UTC). Format: `YYYY-MM-DD hh:mm:ss`. The supported range is from `1970-01-01 00:00:01` UTC to `2038-01-09 03:14:07` UTC. Automatic initialization and updating to the current date and time can be specified using DEFAULT CURRENT_TIMESTAMP and ON UPDATE CURRENT_TIMESTAMP in the column definition |
| TIME                   | A time. Format: `hh:mm:ss`. The supported range is from `-838:59:59` to `838:59:59`                                                                                                                                                                                                                                                                                                                                 |
| YEAR                   | A year in four-digit format. Values allowed in four-digit format: 1901 to 2155, and 0000.MySQL 8.0 does not support year in two-digit format.                                                                                                                                                                                                                                                                       |

## Column Constraints

| Constraint Type | Description                                                                                      |
|:----------------|:-------------------------------------------------------------------------------------------------|
| PRIMARY KEY     | A combination of a `NOT NULL` and `UNIQUE`. Uniquely identifies each row in a table              |
| AUTO_INCREMENT  | Allows a unique number to be generated automatically when a new record is inserted into a table. |
| DEFAULT         | Sets a default value for a column if no value is specified                                       |
| NOT NULL        | Ensures that a column cannot have a NULL value                                                   |
| UNIQUE          | Ensures that all values in a column are different                                                |
| CREATE INDEX    | Used to create and retrieve data from the database very quickly                                  |
| CHECK           | Ensures that the values in a column satisfies a specific condition                               |
| FOREIGN KEY     | Prevents actions that would destroy links between tables                                         |

## Operators

### Arithmetic

| Operator | Description |
|:---------|:------------|
| `+`      | Add         |
| `-`      | Subtract    |
| `*`      | Multiply    |
| `/`      | Divide      |
| `%`      | Modulo      |

### Bitwise 

| Operator        | Description  |
|:----------------|:-------------|
| `&`             | And          |
| &VerticalLine;  | Or           |
| `^`             | exclusive OR |

`&` operator
```shell
1 & 1 = 1
1 & 0 = 1
0 & 0 = 0
```

`|` operator
```shell
1 | 1 = 1
1 | 0 = 1
0 | 0 = 0
```

`^` operator
```shell
1 ^ 1 = 0
1 ^ 0 = 1
0 ^ 0 = 0
```

### Comparison

| Operator | Description              |
|:---------|:-------------------------|
| `=`      | Equal to                 |
| `>`      | Greater than             |
| `<`      | Less than                |
| `>=`     | Greater than or equal to |
| `<=`     | Less than or equal to    |
| `<>`     | Not equal to             |

### Logical

| Operator  | Description                                                    |
|:----------|:---------------------------------------------------------------|
| ALL       | TRUE if all of the subquery values meet the condition          |
| ANY       | TRUE if any of the subquery values meet the condition          |
| `AND`     | TRUE if all the conditions separated by AND is TRUE            |
| `BETWEEN` | TRUE if the operand is within the range of comparisons         |
| EXISTS    | TRUE if the subquery returns one or more records               |
| `IN`      | TRUE if the operand is equal to one of a list of expressions   |
| `LIKE`    | TRUE if the operand matches a pattern                          |
| `NOT`     | Displays a record if the condition(s) is NOT TRUE              |
| `OR`      | TRUE if any of the conditions separated by OR is TRUE          |
| SOME      | TRUE if any of the subquery values meet the condition          |

Example:

*condition use comparison operator*
> general condition: column = value 

*replace column_name, column_n, value_n, conditions*

1. AND

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE condition_1 AND condition_2 AND condition_3;
```

2. BETWEEN

*Select values within a given range. The values can be number, date, text.*

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE `column_name` BETWEEN value_1 AND value_2;
```

3. IN

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE `column_name` IN (value_1, value_2, value_n);
```

4. LIKE

> Regex Pattern：
> 
> 'a%'  ->      Finds any values that start with "a"
> 
> '%a'  ->      Finds any values that end with "a"
> 
> '%a%' ->      Finds any values have "a"

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE `column_name` LIKE `regex_pattern`;
```

5. NOT

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE NOT condition;
```

6. OR

```sql
SELECT column_1, column_2, column_n
FROM `table_name`
WHERE condition_1 OR `condition_2 OR condition_n;
```

### Numeric(Aggregate) Functions

| Function  | Description                                                   |
|:----------|:--------------------------------------------------------------|
| `SUM()`    | Calculates the sum of a set of values                         |
| `AVG()`     | Returns the average value of an expression                    |
| `MAX()`     | Returns the maximum value in a set of values                  |
| `MIN()`     | 	Returns the minimum value in a set of values                 |
| `COUNT()`   | Returns the number of records returned by a select query      |

Example:

*replace column_n, aggregate_column, conditions, column_name*

1. SUM()

```sql
SELECT column_1, column_2, column_n,
       SUM(aggregate_column)
FROM `table_name`
[WHERE conditions]
GROUP BY column_name;
```
2. AVG()

```sql
SELECT column_1, column_2, column_n,
       SUM(aggregate_column)
FROM `table_name`
[WHERE conditions]
GROUP BY column_name;
```

3. MAX()

```sql
SELECT column_1, column_2, column_n,
       MAX(aggregate_column)
FROM `table_name`
[WHERE conditions]
GROUP BY column_name;
```
4. MIN()

```sql
SELECT column_1, column_2, column_n,
       MIN(aggregate_column)
FROM `table_name`
[WHERE conditions]
GROUP BY column_name;
```

5. COUNT()

```sql
SELECT column_1, column_2, column_n,
       COUNT(aggregate_column)
FROM `table_name`
[WHERE conditions]
GROUP BY column_name;
```
