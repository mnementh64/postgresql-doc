# Table design

Some explanations and best practices to design well tables, especially tables with special usage :

* hundred of millions / billions of rows
* or / and heavily updated, inserted or deleted

Using `pgstattuple` extension, it's easy to get an accurate row size (without having to perform any VACUUM FULL), which is very useful to verify / understand how PostgreSQL deal with data types, NULL values, ... 


##Row size

Some example of good explanations : [here](https://stackoverflow.com/questions/13524222/table-size-with-page-layout) and [here](https://stackoverflow.com/questions/13570613/making-sense-of-postgres-row-sizes).

Official doc is [here](https://www.postgresql.org/docs/9.6/static/storage-page-layout.html#HEAPTUPLEHEADERDATA-TABLE).

Row is made of a :

* header (23 bytes)
* 1 byte of data alignement (8 bytes per block) --> could be used for NULL bitmap : see [below](/postgresql/table_design/#null-and-disk-space) for details 
* depending on each column type (for example, [numerical types sizes](https://www.postgresql.org/docs/9.6/static/datatype-numeric.html#DATATYPE-INT) from official documentation) 
* some padding to respect data alignment (see `typealign` field description of this [official doc page](https://www.postgresql.org/docs/current/static/catalog-pg-type.html))

Padding doesn't apply for the last column.

####Best practices

Column order could change the row size because some types have a padding to respect data alignement.

Example :

    integer are 4 bytes and need a data alignement of 4 bytes
    
Let's create / fill a table with 2 integer columns.

```sql
DROP TABLE IF EXISTS my_table; 
CREATE TABLE my_table AS SELECT 
        generate_series(1,100000) as a1
        ,generate_series(1,100000) as a2;
```

The row size is : **32** as expected : 24 (header) + 2 * 4 (2 integers)

But with 3 integers and 1 timestamp (expected to be 8 bytes, see [official doc](https://www.postgresql.org/docs/9.6/static/datatype-datetime.html)), 

```sql
DROP TABLE IF EXISTS my_table; 
CREATE TABLE my_table AS SELECT 
        1 as a1
        ,2 as a2
        ,3 as a3
        ,generate_series('2018-01-01'::timestamp, '2018-12-31'::timestamp, '1 day'::interval) as a4;
```

The row size is : **48** that is : 24 (header) + 3 * 4 (12 integers) + 4 (padding of last integer) + 8 (timestamp)

The 2nd integer cost 8 bytes, and not 4 !


So imagine column orders including this padding constraint, then check a solution with this methodology. 


##NULL and disk space

Each row is composed of a header then other blocks for data.

At the end of the header (23 bytes), there is a byte free for a NULL bitmap that could be used to flag NULL in any of the first 8 columns. 

So, since a table has a 9th column (with a `NOT NULL` constraint or not), the null bitmap is extended of an additional 8 bytes (`MAXALIGN` on 64bits CPUs) to flag any of the next 64 (8*8) columns.

And so on.

Reference [here](https://stackoverflow.com/questions/5008753/does-not-using-null-in-postgresql-still-use-a-null-bitmap-in-the-header).
Also [here](https://stackoverflow.com/questions/12145772/do-nullable-columns-occupy-additional-space-in-postgresql/12147130#12147130).

Assuming :

* a table with all columns as integer (can be null),
* an integer is 4 bytes long,
* `select tuple_len / tuple_count from pgstattuple('my_table')` query gives the tuple size,

we get :

| number of columns | constraints  | nb columns full of null | tuple size | explanation |
|-------------------|--------------|-------------------------|------------|-------------|
| 8                 | none         | 8                       | 24         | standard header = 23+1
| 9                 | none         | 9                       | 32         | 24 (standard header) + 8 (new NULL bitmap)
| 9                 | none         | 8                       | 36         | 32 (previous) + 4 (int)
| 9                 | none         | 1                       | 64         | 24 (standard header)+ 8 * 4 (8 int) + 8 (new NULL bitmap)
| 9                 | 9th NOT NULL | 1                       | 64         | 24 (standard header)+ 8 * 4 (8 int) + 8 (new NULL bitmap)
| 9                 | 1th NOT NULL | 1                       | 64         | 24 (standard header)+ 8 * 4 (8 int) + 8 (new NULL bitmap)

####Methodology of tests

For the last row of the test, you could use the following steps :

Create / fill the table (100000 rows)

```sql
DROP TABLE IF EXISTS my_table; 
CREATE TABLE my_table AS SELECT 
		generate_series(1,100000) as a1
		,generate_series(1,100000) as a2
		,generate_series(1,100000) as a3
		,generate_series(1,100000) as a4
		,generate_series(1,100000) as a5
		,generate_series(1,100000) as a6
		,generate_series(1,100000) as a7
		,generate_series(1,100000) as a8
		,generate_series(1,100000) as a9;
```

then

```sql
-- add the NOT NULL constraint
ALTER TABLE my_table ALTER COLUMN a1 SET NOT NULL;

-- fill a column with null values
update my_table set a5=null;
```

and get the tuple size 

```sql
select tuple_len / tuple_count from pgstattuple('my_table');
```


####Best practices

Especially useful for tables with a lot of rows. 

* be aware of the data types : minimize the size (use integer for seconds instead of bigint for millis if possible)
* be aware of columns order because of alignment padding - it could help you save a lot of disk space and speed up writes, reads, ...
* NOT NULL : constraint must whenever it's possible (more for explicit coherence reasons than performance - see this [discussion](https://dba.stackexchange.com/questions/140410/what-are-the-consequences-of-not-specifying-not-null-in-postgresql-for-fields-wh)).
* No special rules on how NULL columns or NOT NULL columns must be ordered.

When modifying a table :

* it's very cheap to append a new column, even to a huge table, if it could be NULL (only set a bit in the null bitmap).
* think about it before adding a 9th column - it's very hard to understand what triggers the 8 bytes extension of the null bitmap for all rows containing at least one NULL. But when it occurs, it will have to rewrite all these rows (because the null bitmap is located between the header and the data). 

