# Performance with PostgreSQL 

Tuning the server is worth as the expected activity is not standard : heavy workload, bulk reads, high number of clients, ...

Standard advices and understanding to tune PostgreSQL for performance could be found [here](https://www.2ndquadrant.com/media/pdfs/talks/MonitoringBufferCache.pdf), written by the great Greg Smith itself.

A friend of him also answer [this post](https://stackoverflow.com/questions/9407442/optimise-postgresql-for-fast-testing) with great details !

Good advices also [there](http://pydanny-event-notes.readthedocs.io/en/latest/DjangoConEurope2012/10-steps-to-better-postgresql-performance.html).

Level of optimizations are mainly four :

* OS, hardware
* PostgreSQL configuration
* database design
* software, model and queries


##Tuning PostgreSQL 

####Use separate disks

It's important to use separate disks for :

* data : `data_directory = '/var/lib/postgresql/9.6/main'` in `postgresql.conf`
* wal : only way by symbolic link ?
* wal archives if activated
* logs : only way by symbolic link ?


####Synchronous commit

`postgresql.conf`'s parameter `synchronous_commit=on` define the commit method used. If ON, it waits for WAL files to be written to disk before acknowledge client query. Depending on HDD type, it could introduce some latency.

Setting this parameter do not harm data coherence but in case of server crash before last commit were written to disk, they could be lost. It doesn't matter in most of cases !

See this [blog post](https://blog.2ndquadrant.com/evolution-fault-tolerance-postgresql-synchronous-commit/) for details.

Other citation : `This is one of the most known Postgres performance tweaks`.   


####WAL write method

The tool `pg_test_fsync` packaged with PostgreSQL could be useful to verify if the default value for the `postgres.conf` parameter `wal_sync_method` is the more efficient (`fdatasync` by default on Linux OS).

The tool will execute some checks to output write stats with all avalaible methods (see [details](/postgresql/useful-tools/#pg_test_fsync)) :

    ./pg_test_fsync
    
    5 seconds per test
    O_DIRECT supported on this platform for open_datasync and open_sync.
    
    Compare file sync methods using one 8kB write:
    (in wal_sync_method preference order, except fdatasync is Linux's default)
            open_datasync                      1002.075 ops/sec     998 usecs/op
            fdatasync                          1062.147 ops/sec     941 usecs/op
            fsync                               597.879 ops/sec    1673 usecs/op
            fsync_writethrough                            n/a
            open_sync                            34.400 ops/sec   29070 usecs/op
    
    Compare file sync methods using two 8kB writes:
    (in wal_sync_method preference order, except fdatasync is Linux's default)
            open_datasync                       688.196 ops/sec    1453 usecs/op
            fdatasync                          1045.286 ops/sec     957 usecs/op
            fsync                               588.164 ops/sec    1700 usecs/op
            fsync_writethrough                            n/a
            open_sync                            25.560 ops/sec   39123 usecs/op
    
    Compare open_sync with different write sizes:
    (This is designed to compare the cost of writing 16kB in different write
    open_sync sizes.)
             1 * 16kB open_sync write            46.153 ops/sec   21667 usecs/op
             2 *  8kB open_sync writes           16.677 ops/sec   59964 usecs/op
             4 *  4kB open_sync writes            2.985 ops/sec  334965 usecs/op
             8 *  2kB open_sync writes            1.169 ops/sec  855245 usecs/op
            16 *  1kB open_sync writes            0.350 ops/sec  2860294 usecs/op
    
    Test if fsync on non-write file descriptor is honored:
    (If the times are similar, fsync() can sync data written on a different
    descriptor.)
            write, fsync, close                 626.905 ops/sec    1595 usecs/op
            write, close, fsync                 681.045 ops/sec    1468 usecs/op
    
    Non-sync'ed 8kB writes:
            write                            357014.644 ops/sec       3 usecs/op


####Mastering Checkpoints

!!!Note

    What is it ? Some [details](/postgresql/checkpoints/).

Default value of 5mn for checkpoints and 0.5 for the available fraction of this time to do the writing is not good for heavy workload usage.

We must set the checkpoint time to at least 30mn and maybe a lot of more, 90mn can be good. 

Anyway, once this time is decided, we have to know the amount of produced WAL files. 

Imagine we would like to set the value as :

    checkpoint_timeout = 30min


Let's execute the following queries to get basic knowledge about the server activity :

    postgres=# SELECT pg_current_xlog_insert_location();
     pg_current_xlog_insert_location 
    ---------------------------------
     3D/B4020A58
    (1 row)
    
    ... after 5 minutes ...
    
    postgres=# SELECT pg_current_xlog_insert_location();
     pg_current_xlog_insert_location 
    ---------------------------------
     3E/2203E0F8
    (1 row)
    
    postgres=# SELECT pg_xlog_location_diff('3E/2203E0F8', '3D/B4020A58');
     pg_xlog_location_diff 
    -----------------------
                1845614240
    (1 row)
      
So, every 5mn, ~1.8GB of WAL files is produced, that is 10GB for 30mn. But the `max_wal_size` must be set for 3 checkpoints, so tune it to **30GB**.

It's a good habit to let the checkpoint a good amount of time to write its stuff to disk. Evaluate the `checkpoint_completion_target` value as :

    checkpoint_completion_target = (checkpoint_timeout - 2min) / checkpoint_timeout
    And no more then 0.9

For 30mn checkpoints, it's 0.9, that means checkpoint process have 0.9*30=27mn to write data befroe complete the checkpoint.

Finally, activate the log to monitor the checkpoint activity :

    log_checkpoint = on
    

By the way, there are many writing methods that could be used for WAL files. See [here](/postgresql/pgsql_perf/#best-write-method) to find the best. The `wal_sync_method` parameter could be verified or changed by this way.

!!!Note

    Monitor checkpoints [running statistics](/postgresql/statistics/#checkpoints) to ensure the behavior is under control.
 

####Autovacuum

!!!Note

    What is it ? Some [details](/postgresql/autovacuum/).

[Good article](https://blog.2ndquadrant.com/autovacuum-tuning-basics/) about tuning autovacuum, again in 2nd quadrant blog !

Remember standard configuration of autovacuum stands for standard usages. It needs to be tuned for specific purposes.

Many risks with autovacuum :

* too many writing in DB so autovacuum could not free space enough quick,
* some tables have specific size and design that make the autovacuum job hard on them (like for timeseries). We could then observe [table bloating](/postgresql/autovacuum/#bloating),
* bloating could be observed even on catalog tables (especially if there are a huge number of tables in schema) !
* and many others I don't know !

Autovacuum parameters are :

* `autovacuum_max_workers` : (3 default) Could be increase for aggressive vacuum
* `autovacuum_naptime` : (1min default) minimum delay between autovacuum runs on any given database. Could be decrease for aggressive vacuum
* `autovacuum_vacuum_threshold` / `autovacuum_vacuum_scale_factor` : implied in autovacuum [trigger condition](/postgresql/autovacuum/#what-trigger-autovacuum)
* `autovacuum_analyze_threshold` / `autovacuum_analyze_scale_factor` : implied in autoanalyze [trigger condition](/postgresql/autovacuum/#analyze)
* `autovacuum_vacuum_cost_delay` : (20ms default) delay the process will sleep when the cost limit has been exceeded. Could be decrease for aggressive vacuum
* `autovacuum_vacuum_cost_limit` : (-1 means use vacuum_cost_limit value which is 200 default) arbitrary cost vacuums should not exceed. Coud be increase for aggressive vacuum



On big tables, default behavior should be customized table by table : 

```sql
-- reduce the scale factor (default is 0.2 , that is 20%)
ALTER TABLE <tablename> SET autovacuum_vacuum_scale_factor = 0.01;
```

For a million-row table this means autovacuum would start after ten thousand rows are invalidated rather than two hundred thousand. It helps stopping bloat from getting out of control.

Even more aggresive for some of the largest tables :

    ALTER TABLE table_name SET (autovacuum_vacuum_scale_factor = 0.0);
    ALTER TABLE table_name SET (autovacuum_vacuum_threshold = 5000);
    ALTER TABLE table_name SET (autovacuum_analyze_scale_factor = 0.0);
    ALTER TABLE table_name SET (autovacuum_analyze_threshold = 5000);  
 
Autovacuum will be triggered every 5000 inserts, updates, or deletes.

!!!Advice
    
    It's better autovacuum work more often and does less work than less often and does more work.

!!!Note

    Monitor [vacuum efficiency](/postgresql/statistics/#autovacuum) to ensure the behavior is under control.


####Analyze

!!!Note

    What is it ? Some [details](/postgresql/autovacuum/#analyze).


The default behavior of automatic process rely only on how many row have been inserted or updated, so depending of chat kind of data each column keep, it could be interesting to launch manual ANALYZE in some specific cases.

ANALYZE behavior could be tuned by changing the `default_statistics_target = 100` parameter which tells the analyze process how many rows of each table it must sample to compute stats. The greater this number is, the better stats are but it costs more in computation ! 


!!!Note

    Monitor [analyze efficiency](/postgresql/statistics/#autovacuum)


####Shared buffers

An excellent example of analyzing stats from background writer / checkpoint and shared buffer could be found in [this post](https://forums.postgresql.fr/viewtopic.php?id=2783).

Shared buffers should be :

* large enough to ensure the checkpointer process is mainly responsible for the buffer writing to disk (monitor who does the write)
* not too large so checkpoint could handle the writing in the time allowed

The basic tuning is 25% of the RAM on a dedicated server.

But how compute the more efficient RAM size it needs ?

 

####Query planner

The parameter `random_page_cost` sets the planner's estimate of the cost of a non-sequentially-fetched disk page , like fetching blocks using the index.

Its default value of 4.0 is quite high for modern storage system.

See [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server) for more details. The official [PostgreSQL documentation](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-RANDOM-PAGE-COST) page tells to also tune the other parameter `seq_page_cost` mostly to the same value.  






##Tuning Linux

See details on [the dedicated page](/postgresql/linux_tuning/).

