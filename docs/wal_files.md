# Wal files

##What's is a WAL file ? 

[This is a good reading](http://eulerto.blogspot.fr/2011/11/understanding-wal-nomenclature.html).

WAL means "Write Ahead Log". Any transaction is first written out as a WAL file before applied to on-disk data. See [checkpoints page](/postgresql/checkpoints/) for details. 


##File nomenclature

Files are located into `$PG_DIR/pg_xlog` for version 9.x and in `$PG_DIR/pg_wal` since 10. 

Each file is sized 16MB. This value is computed by default from the `shared_buffers` parameters :

    #wal_buffers = -1                       # min 32kB, -1 sets based on shared_buffers
                                            # (change requires restart)

WAL files are named like : `000000020000070A0000008E` where :

* `00000002` : the timeline
* `0000070A` : the logical xlog file 
* `0000008E` : the physical xlog file

Let's have a look to a `pg_xlog` content by typing `ls -l /var/lib/postgresql/9.6/main/pg_xlog/`. It outputs :

    -rw------- 1 postgres postgres 16777216 Jan 23 04:56 0000000100001D710000001D
    -rw------- 1 postgres postgres 16777216 Jan 23 05:50 0000000100001D710000001E
    -rw------- 1 postgres postgres 16777216 Jan 23 05:58 0000000100001D710000001F
    -rw------- 1 postgres postgres 16777216 Jan 23 04:50 0000000100001D7100000020
    -rw------- 1 postgres postgres 16777216 Jan 23 05:36 0000000100001D7100000021
    -rw------- 1 postgres postgres 16777216 Jan 23 05:55 0000000100001D7100000022
    -rw------- 1 postgres postgres 16777216 Jan 23 04:35 0000000100001D7100000023
    -rw------- 1 postgres postgres 16777216 Jan 23 05:43 0000000100001D7100000024
    -rw------- 1 postgres postgres 16777216 Jan 23 04:55 0000000100001D7100000025
    -rw------- 1 postgres postgres 16777216 Jan 23 05:20 0000000100001D7100000026

To see a wal file content, use the `pg_xlogdump` tool by typing :

    /usr/lib/postgresql/9.6/bin/pg_xlogdump -n 20 -f /var/lib/postgresql/9.6/main/pg_xlog/0000000100001D7000000037
    
It outputs a human readable version of the binary file content :

    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/37000028, prev 1D70/36FFFFC0, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757724 blk 678
    rmgr: Heap        len (rec/tot):      3/   111, tx:  265963650, lsn: 1D70/37000068, prev 1D70/37000028, desc: INSERT off 51, blkref #0: rel 1663/20430/16757720 blk 9692
    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/370000D8, prev 1D70/37000068, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757723 blk 678
    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/37000118, prev 1D70/370000D8, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757724 blk 678
    rmgr: Heap        len (rec/tot):      3/   111, tx:  265963650, lsn: 1D70/37000158, prev 1D70/37000118, desc: INSERT off 52, blkref #0: rel 1663/20430/16757720 blk 9692
    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/370001C8, prev 1D70/37000158, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757723 blk 678
    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/37000208, prev 1D70/370001C8, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757724 blk 678
    rmgr: Heap        len (rec/tot):      3/   111, tx:  265963650, lsn: 1D70/37000248, prev 1D70/37000208, desc: INSERT off 53, blkref #0: rel 1663/20430/16757720 blk 9692
    rmgr: Btree       len (rec/tot):      2/    64, tx:  265963650, lsn: 1D70/370002B8, prev 1D70/37000248, desc: INSERT_LEAF off 243, blkref #0: rel 1663/20430/16757723 blk 678

The first column displays the resource manager : what kind of resources is impacted by the transaction ? Index btree, Heap operation like INSERT, ...


##Transaction location

Something like `77FA/5F11E520` is a transaction log location, it means the location of a transaction in a log file :

* `77FA` is the logical xlog file
* `5F11E520` is the offset inside the logical xlog.

To know in what log file a transaction is located, use 

    postgres=# select pg_xlogfile_name('68A/16E1DA8');
         pg_xlogfile_name  
    --------------------------
     000000020000068A00000001
    (1 row)

To know in which log file postgreSQL is currently writing, use 

    postgres=# select pg_xlogfile_name(pg_current_xlog_location());
         pg_xlogfile_name  
    --------------------------
     000000020000074B000000E4
    (1 row)
