# DB 기초 - 페이징 최적화

* A pipelined order by is therefore a very powerful means of optimization for paritial result queries.
* Using a pipelined order by is not only about saving the effort to sort the result, it is more about delivering the first results without reading and sorting all rows. 
  * possible to abort the execution after fetching a few rows without discarding the efforts to prepare the final result.
* Although the syntax of these queries varies from database to database, they still execute the queries in a very similar way. 

### Pipelined/Indexed ORDER BY

* SQL queries with an order by clause do not need to sort the result explicitly if the relevant index already delivers the rows in the required order.
  * That means the same index that is used for the `where` clause must also cover the `order by` clause.
* An `INDEX RANGE SCAN` delivers the result in index order anyway. To take advantage of this fact, we just have to extend the index definition so it corresponds to the `order by` clause
  * The sort operation `SORT ORDER BY` will disappeared from the execution plan even though the query still has an `order by` clause. 
  * The database exploits the index order and skips the explicit sort operation.

> If the index order corresponds to the `order by` clause, the database can omit the explicit sort operation.

* If the database uses a sort operation even though you expected a pipelined execution, it can have two reasons
  * (1) the execution plan with the explicit sort operation has a better cost value
  * (2) the index order in the scanned index range does not correspond to the `order by` clause.
  * Use the full index definition in the `order by` clause to find the reason for an explicit sort operation.

## Querying Top-N Rows

* Inform the database whenever you don't need all rows.
* The following examples show the use of these well-known extensions by querying the ten most recent sales. 
  * The basis is always the same: fetching *all* sales, beginning with the most recent one. The respective top-N syntax just aborts the execution after fetching ten rows.

``` sql
// MYSQL
SELECT *
  FROM sales
 ORDER BY sale_date DESC
 LIMIT 10
 
 // ORACLE
 SELECT *
  FROM (
       SELECT *
         FROM sales
        ORDER BY sale_date DESC
       )
 WHERE rownum <= 10
```

* The database can only optimize a query for a partial result if it knows this from the beginning.
* If the optimizer is aware of the fact that we only need ten rows, it will prefer to use a pipelined `order by` if applicable:
* A pipelined top-N query doesn't need to read and sort the entire result set.
* Without using pipelined execution, the response time of this top-N query grows with the table size.
* The response time using a pipelined execution only grows with the number of selected rows.
* The linear response time growth for an execution without a pipelined `order by` is clearly visible. The response time for the pipelined execution remains constant.

![그림](https://use-the-index-luke.com/static/fig07_01_scalability_top_n.en.56HHwZVH.png)

* Although the response time of a pipelined top-N query does not depend on the table size, it still grows with the number of selected rows. 

## Paging Through Results

* The resulting challenge is that it has to skip the rows from the previous pages.
  * firstly the offset method, which numbers the rows from the beginning and uses a filter on this row number to discard the rows before the requested page. 
  * The second method, which I call the seek method, searches the last entry of the previous page and fetches only the following rows.
* The following examples show the more widely used offset method. Its main advantage is that it is very easy to handle—especially with databases that have a dedicated keyword for it (`offset`).

``` sql
// MYSQL
SELECT *
  FROM sales
 ORDER BY sale_date DESC
 LIMIT 10 OFFSET 10
 
 
 // ORACLE
 SELECT *
  FROM ( SELECT tmp.*, rownum rn
           FROM ( SELECT *
                    FROM sales
                   ORDER BY sale_date DESC
                ) tmp
          WHERE rownum <= 20
       )
 WHERE rn > 10
```

* The Oracle database supports `offset` since release 12c. Earlier releases provide the pseudo column `ROWNUM` that numbers the rows in the result set automatically. It is, however, not possible to apply a greater than or equal to (`>=`) filter on this pseudo-column. To make this work, you need to first “materialize” the row numbers by renaming the column with an alias.
* below image shows that the scanned index range becomes greater when fetching more pages.

![그림](https://use-the-index-luke.com/static/fig07_02_topn_paging.en.R96GPtBJ.png)

* This has two disadvantages
  * (1) : the pages drift when inserting new sales because the numbering is always done from scratch
  * (2) : the response time increases when browsing further back.
* The seek method avoids both problems because it uses the *values* of the previous page as a delimiter. That means it searches for the values that must *come behind* the last entry from the previous page. This can be expressed with a simple `where` clause. 

``` sql
SELECT *
  FROM sales
 WHERE sale_date < ?
 ORDER BY sale_date DESC
 FETCH FIRST 10 ROWS ONLY
```

* Instead of a row number, you use the last value of the previous page to specify the lower bound. This has a huge benefit in terms of performance because the database can use the `SALE_DATE < ?` condition for index access. 
  * That means that the database can truly skip the rows from the previous pages.
  * On top of that, you will also get stable results if new rows are inserted.
* Without a deterministic `order by` clause, the database by definition does not deliver a deterministic row sequence. 
* The only reason you *usually* get a consistent row sequence is that the database *usually* executes the query in the same way. Nevertheless, the database could in fact shuffle the rows having the same `SALE_DATE` and still fulfill the `order by` clause.
  * In recent releases it might indeed happen that you get the result in a different order every time you run the query, not because the database shuffles the result intentionally but because the database might utilize parallel query execution.
* Paging requires a deterministic sort order.
  * Even if the functional specifications only require sorting “by date, latest first”, we as the developers must make sure the `order by` clause yields a deterministic row sequence.
* extend the `order by` clause and the index with the primary key `SALE_ID` to get a deterministic row sequence. 

``` sql
SELECT *
  FROM sales
 WHERE (sale_date, sale_id) < (?, ?)
 ORDER BY sale_date DESC, sale_id DESC
 FETCH FIRST 10 ROWS ONLY
```

* compares the performance characteristics of the offset and the seek methods

![그림](https://use-the-index-luke.com/static/fig07_04_paging_scalability.en.Owe6Uvmd.png)

* Of course the seek method has drawbacks as well, the difficulty in handling it being the most important one. You not only have to phrase the `where` clause very carefully—you also cannot fetch arbitrary pages. 

## Using Window Functions for Efficient Pagination

* Window functions offer yet another way to implement pagination in SQL. 

``` sql
SELECT *
  FROM ( SELECT sales.*
              , ROW_NUMBER() OVER (ORDER BY sale_date DESC
                                          , sale_id   DESC) rn
           FROM sales
       ) tmp
 WHERE rn between 11 and 20
 ORDER BY sale_date DESC, sale_id DESC
```

* The `ROW_NUMBER` function enumerates the rows according to the sort order defined in the `over` clause.
* The Oracle database recognizes the abort condition and uses the index on `SALE_DATE` and `SALE_ID` to produce a pipelined top-N behavior
* This query is as efficient as the offset method explained 
* The strength of window functions is not pagination, however, but analytical calculations.