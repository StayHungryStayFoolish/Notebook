# Select Advance

## Keywords syntax

| Syntax                 | Description                                                                                                                       |
|:-----------------------|:----------------------------------------------------------------------------------------------------------------------------------|
| AS                     | Aliases are used to give a table, or a column in a table, a temporary name.                                                       |
| DISTINCT               | Used to return only different values                                                                                              |
| WHERE                  | Used to extract only those records that fulfill a specified condition.                                                            |
| GROUP BY               | Groups rows that have the same values into summary rows.                                                                          |
| ORDER BY               | Used to sort the result-set in ascending or descending order.                                                                     |
| LIMIT `n_1` [OFFSET `n_2`] | Specify the number of records(offset the numbers) to return.                                                                      |
| HAVING                 | The HAVING clause was added to SQL because the WHERE keyword cannot be used with `COUNT(), SUM(), AVG(), MAX(), MIN()` functions. |
| INNER JOIN ... ON ...  | Selects records that have matching values in `both` tables.                                                                         |
| LEFT JOIN ... ON ...   | Returns all records from the `left table`, and the matching records (if any) from the right table.                                |
| RIGHT JOIN ... ON ...  | Returns all records from the `right table`, and the matching records (if any) from the left table.                                |

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


![MySQL JOIN](https://raw.githubusercontent.com/StayHungryStayFoolish/notebook-img/master/img/MySQL/mysql_join.png?raw=true)