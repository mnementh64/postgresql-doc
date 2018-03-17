# Cache system

A good reading could be found on this [blog](https://madusudanan.com/blog/understanding-postgres-caching-in-depth/).

The main cache system is `Shared buffers`, tuned in `postgres.conf` by the `shared_buffers = 128MB` parameter. It's used to cache :

* tables data
* tables indexes
* query plans

Each time a data becomes dirty, it has to be flushed to the disk and to WAL files.


The `wal_writer` process writes to WAL files and can be done asynchronously by changing a setting (see [details](/postgresql/pgsql_perf/#synchronous-commit)).

The `background writer` is responsible for the data writing : 

* triggered by checkpoint : it spreads the writing for almost all the configured checkpoint time.
* triggered by backend request : it reads from the cache and in case of dirty pages, writes to disk then loads to cache.
* triggered by clean process : it makes some place free in case of reallocation need.

The objective is to make the job mainly done by checkpointer process because :

* backend buffers introduce latency for users queries
* clean process means the cache is too small

 