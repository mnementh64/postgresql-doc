# Query optimizations

##Fill factor

Fill factor [reading](https://www.dbrnd.com/2016/03/postgresql-the-awesome-table-fillfactor-to-speedup-update-and-select-statement/).

The fill factor (default value 100%) tells Postgres what to do with all pages of an updated tuple. In case of 100%, even if a tuple length is less than the minimal page size (8KB by default), the whole page is flagged as dirty and cannot be reused, even for new tuple insertion (it allocates a new page).

So for heavily updated tables with rows less than 8KB, it's worth decreasing the fillfactor value (no less than 10) : it saves many pages allocation, so may IOs and RAM, and speed up updates as well as select (table data is less fragmented).

To measure a table row size, use the following query :

```sql
SELECT l.what, l.nr AS "bytes/ct"
     , CASE WHEN is_size THEN pg_size_pretty(nr) END AS bytes_pretty
     , CASE WHEN is_size THEN nr / x.ct END          AS bytes_per_row
FROM  (
   SELECT min(tableoid)        AS tbl      -- same as 'public.tbl'::regclass::oid
        , count(*)             AS ct
        , sum(length(t::text)) AS txt_len  -- length in characters
   FROM   public.my_table t  -- provide table name *once*
   ) x
 , LATERAL (
   VALUES
      (true , 'core_relation_size'               , pg_relation_size(tbl))
    , (true , 'visibility_map'                   , pg_relation_size(tbl, 'vm'))
    , (true , 'free_space_map'                   , pg_relation_size(tbl, 'fsm'))
    , (true , 'table_size_incl_toast'            , pg_table_size(tbl))
    , (true , 'indexes_size'                     , pg_indexes_size(tbl))
    , (true , 'total_size_incl_toast_and_indexes', pg_total_relation_size(tbl))
    , (true , 'live_rows_in_text_representation' , txt_len)
    , (false, '------------------------------'   , NULL)
    , (false, 'row_count'                        , ct)
    , (false, 'live_tuples'                      , pg_stat_get_live_tuples(tbl))
    , (false, 'dead_tuples'                      , pg_stat_get_dead_tuples(tbl))
   ) l(is_size, what, nr);
```   

##HOT update

[Good reading](https://www.dbrnd.com/2016/12/postgresql-increase-the-speed-of-update-query-using-hot-update-heap-only-tuple-mvcc-fill-factor-vacuum-fragmentation/).

HOT (Heap Only Tuple) update is related to the understanding of the fillfactor parameter of each table.

HOT concept allows PostgreSQL to reuse a dead page from a previous transaction for an UPDATE or an INSERT. It limits the table fragmentation and speed up the process not to have to create a new tuple.


!!!Warning
    
    HOT updates are possible only when there is no index linked to the updated columns. 
    
!!!Tip
    
    If you have big updates, instead of changing large portions of the table at once, you might want to split them up in a couple of chunks.

!!!Tip

    You might not be blocking HOT update by adding indexes. You might get better overall performance without them. 
    

##Long queries

For example, to list all queries that last for more than 2 minutes, try the query :

```sql
SELECT EXTRACT(EPOCH FROM (now() - query_start))::bigint as "runtime", usename, application_name, datname, state, query
  FROM  pg_stat_activity
  WHERE now() - query_start > interval '2 minutes' and state = 'active'
ORDER BY runtime DESC;
```

