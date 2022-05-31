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

> condition: column = value

*replace table_name, column_n, value_n*

```sql
UPDATE `table_name`
    SET 
        column_1 = value_1, 
        column_2 = value_2, 
        column_n = value_n
WHERE condition;
```

### DELETE

1. Delete existing records in a table.

```sql
DELETE FROM `table_name` WHERE condition;
```

## DTL(Data Transaction Language)

**Keywords**
- START TRANSACTION
- SAVEPOINT
- ROLLBACK
- COMMIT

### ACID

- Atomicity

  All operations in a transaction do not end in the middle, either all or none are completed

- Consistency

  The integrity of the data is not compromised before and after the transaction is opened.

- Isolation

  The database allows multiple concurrent transactions to read, write and modify its data. Isolation is to prevent data inconsistency caused by cross execution when multiple transactions are executed concurrently.

- Durability

  After the transaction is completed, changes to the data are permanent and will not be lost even if the system fails.


### Transaction

- Show closed transactions

  - COMMIT
  - ROLLBACK

- Implicitly closing a transaction

  - DDL

```sql
START TRANSACTION ;

INSERT INTO `table_name`
    VALUES (value_1, value_2, value_3, value_n);;


SAVEPOINT a;

UPDATE `table_name`
    SET column_name = value 
WHERE condition;

SAVEPOINT b;

ROLLBACK to a;  
COMMIT ; 

SELECT *
FROM `table_name`;
```

## DQL(Data Query Language)

**Keywords**
 - SHOW
 - SELECT

### SHOW

1. Query all table names in the database

```sql
SHOW FULL TABLES FROM `database_name`
```

2. Query all columns in the table

```sql
SHOW FULL COLUMNS FROM `table_name`
```

### SELECT

1. Select all columns in the table

```sql
SELECT * FROM `table_name`;
```

2. Specify the columns in the table

```sql
SELECT 
    column_1, 
    column_2,
    column_3,
    column_n,
FROM `table_name`;
```