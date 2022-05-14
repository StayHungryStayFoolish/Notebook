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

*replace database_name、table_name、column*、column_definition、constraint*

```sql
USE `database_name`;

CREATE TABLE IF NOT EXISTS `table_name`
(
    column1 column_definition [constraint],
    column2 column_definition [constraint],
    column3 column_definition [constraint],
    ....
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;
```

### ALTER

*replace new_column_name、column_definition、existing_column*

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

*replace database_name、table_name、column*

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

