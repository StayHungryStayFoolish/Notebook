# MySQL Syntax

## DDL(Data Definition Language)

**Keywords**
  - CREATE
  - ALTER
  - DROP

### CREATE

1. Create database

*replace database_name*

```sql
CREATE DATABASE IF NOT EXISTS `database_name`;
```

2. Create table in database

*replace database_name, table_name, column_n, column_definition, constraint*

```sql
USE `database_name`;

CREATE TABLE IF NOT EXISTS `table_name`
(
    column_1 column_definition [constraint],
    column_2 column_definition [constraint],
    column_3 column_definition [constraint],
    column_n column_definition [constraint]
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;
```

### ALTER

*replace new_column_name, column_definition, existing_column*

1. Add `new_column_name` to the end of table 

```sql
ALTER TABLE `table_name`
    ADD `new_column_name` column_definition;
```

2. Add `new_column_name` to before `existing_column`

> keywords: FIRST 

```sql
ALTER TABLE `table_name`
    ADD `new_column_name` column_definition FIRST `existing_column`;
```

3. Add `new_column_name` to after `existing_column`

> keywords: AFTER

```sql
ALTER TABLE `table_name`
    ADD `new_column_name` column_definition AFTER `existing_column`;
```

4. Change column definition

> keywords: MODIFY

```sql
ALTER TABLE `table_name`
    MODIFY COLUMN `existing_column_name` column_definition;
```

5. Drop column

> keywords: DROP

```sql
ALTER TABLE `table_name`
    DROP COLUMN `existing_column_name`;
```

### DROP

*replace database_name, table_name, column*

1. Drop database

```sql
DROP DATABASE IF EXISTS `database_name`;
```

2. Drop table

```sql
DROP TABLE IF EXISTS `table_name`;
```

## DML(Data Manipulate Language)

**Keywords**
 - INSERT
 - UPDATE
 - DELETE

### INSERT

1. Add values for all columns

*replace table_name, value_n*

```sql
INSERT INTO `table_name`
    VALUES (value_1, value_2, value_3, value_n);
```

2. Specify the columns and the values to be added 

*replace table_name, column_n, value_n*

```sql
INSERT INTO `table_name` (column_1, column_2, column_3, column_n)
    VALUES (value_1, value_2, value_3, value_n);
```

### UPDATE

1. Specify conditions and modify the existing records

*replace table_name, column_n, value_n*

```sql
UPDATE `table_name`
    SET 
        column_1 = value_1, 
        column_2 = value_2, 
        column_n = value_n
WHERE condition;
```
