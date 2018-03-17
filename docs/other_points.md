# Other points

Many points could be considered to improve Postgres performance for each well defined use case. Some of them are listed below with external references, and a summary of what is the expected gain.

##Autovacuum

Let it be more aggressive on big tables. At least, down these parameters :

    autovacuum_vacuum_scale_factor = 0.01       # or even less
    autovacuum_analyze_scale_factor = 0.01      # or even less
    
It's a good idea to also make it more aggressive (with the same parameter values ?) on main catalog tables :

|                 table_name                 | table_size | index_size | total_size 
|--------------------------------------------|------------|------------|------------
| pg_catalog.pg_statistic                    | 351 MB     | 6856 kB    | 358 MB
| pg_catalog.pg_attribute                    | 170 MB     | 81 MB      | 252 MB
| pg_catalog.pg_class                        | 55 MB      | 120 MB     | 174 MB
| pg_catalog.pg_depend                       | 26 MB      | 54 MB      | 80 MB
| pg_catalog.pg_type                         | 26 MB      | 27 MB      | 53 MB
| pg_catalog.pg_index                        | 27 MB      | 9760 kB    | 37 MB

A good reading : [https://wiki.postgresql.org/images/5/5e/Autovacuum_pgconfeu_gorthx.pdf](https://wiki.postgresql.org/images/5/5e/Autovacuum_pgconfeu_gorthx.pdf).


##Autoanalyze

To be tweacked at least like autovacuum.

Some day, when other parameters will be well mastered, you could have a look to analyze tuning by column. It could be interesting to consider two main cases, both related to huge tables :

* the default `default_statistics_target = 100` could be too small to compute good stats from a so small sample (100 rows),
* if the data distribution of a column do not change in the time, maybe we could turn autoanlyze off for this column. 


##Index

To design indexes for a table, one gold rule to follow is : 
    
    Index first for equality, then for ranges.

Good reading (in french) to understand multi-column indexes, partial indexes, ... : [http://use-the-index-luke.com/fr/sql/la-clause-where/rechercher-un-intervalle](http://use-the-index-luke.com/fr/sql/la-clause-where/rechercher-un-intervalle). 


##BRIN indexes

BRIN (`Block Range INdexes`) indexes were introduced in PostgreSQL 9.5 and were designed as range indexes. Link to [official doc](https://docs.postgresqlfr.org/9.6/brin.html).

They took much less place than B-tree indexes and are quite slower. Their overhead on INSERT is very tiny, unlike for B-tree.

To resume their interest.`BRIN indexes can speed things up a lot, but only if your data has a strong natural ordering to begin with. In our case, using a BRIN index was 500 times more space efficient than a B-tree index, and ran a bit faster too. We managed to improve our best case time by a factor of 2.6 and our worst case time by a factor of 30 with some simple optimizations`. Citation from [this post](http://dev.sortable.com/brin-indexes-in-postgres-9.5/).

!!!Note

    They could be of huge interest for time series data : all queries contains range clause for dates, never equality.
    In case of aggregation queries, it let the opportunity to bench the interest to add B-tree indexes on columns involved in GROUP BY clauses.
    
  
##Detailed statistics

####Log analyzer

[PgBadger](https://github.com/dalibo/pgbadger) is a powerful log analyzer that produce nice & useful reports. Just have to activate some logging flags. The most responsible for possible overhead is `log_min_duration_statement = 0`. On busy server, possibly increase this value not to log all queries.  


####Statements

The official extension [https://www.postgresql.org/docs/current/static/pgstatstatements.html](https://www.postgresql.org/docs/current/static/pgstatstatements.html) provides tracking of execution statistics on the server.

To reset the stats gathered by this extension : `SELECT pg_stat_statements_reset();`.

!!!Note
    
    This extension seems not to could cause a significant overhead.
    

####Vacuum

The official extension [https://www.postgresql.org/docs/current/static/pgstattuple.html](https://www.postgresql.org/docs/current/static/pgstattuple.html) provides various functions to obtain tuple-level / page-level statistics. 


!!!Warning
    
    No idea if this extension could cause or not a significant overhead.

####Cache

The official extension [https://www.postgresql.org/docs/current/static/pgbuffercache.html](https://www.postgresql.org/docs/current/static/pgbuffercache.html) provides a means for examining what's happening in the shared buffer cache in real time.

!!!Warning
    
    No idea if this extension could cause or not a significant overhead.


