# Monitoring 

Explain how to deploy via Docker / configure a TIG stack : [Telegraf](https://github.com/influxdata/telegraf), [Influxdb](https://docs.influxdata.com/influxdb/v1.4/introduction/) and [Grafana](https://grafana.com/).

##Architecture


##Central Server

####Docker-compose.yml

    version: '2'
    
    services:
        ## InfluxDB container
        influxdb:
            image: influxdb:latest
            restart: always
            ports:
              - "80:8086"
            volumes:
              - ${PWD}/data/influxdb:/var/lib/influxdb
              - ${PWD}/data/influxdb/influxdb.conf:/etc/influxdb/influxdb.conf:ro
    
    ## Grafana container
        grafana:
            image: grafana/grafana
            restart: always
            ports:
              - "3000:3000"
            volumes:
              - ${PWD}/data/grafana/var/lib/grafana:/var/lib/grafana
              - ${PWD}/data/grafana/etc/grafana/grafana.ini:/etc/grafana/grafana.ini
              - ${PWD}/data/grafana/var/log/grafana:/var/log/grafana


####Installation

Pick up a monitoring server to host influxDB & Grafana

Pull images

    docker pull grafana/grafana
    docker pull influxdb

Create sub-dirs

    mkdir -p /your_home/monitoring/data/grafana/etc/grafana
    mkdir -p /your_home/monitoring/data/grafana/var/lib/grafana
    mkdir -p /your_home/monitoring/data/grafana/var/log/grafana
    mkdir -p /your_home/monitoring/data/influxdb/

Generate default InfluxDB config file

    cd /your_home/monitoring
    docker run --rm influxdb influxd config > data/influxdb/influxdb.conf

Get default grafana.ini file from docker container to host

    docker run -d grafana/grafana bash

Find it

    docker ps

Go in it

    docker exec -it CONTAINER_ID bash

Cat the /etc/grafana/grafana/ini file and copy its content to data/grafana/etc/grafana/grafana.ini file.

Copy the compose yaml (see below)

    /your_home/monitoring/docker-compose-central.yml
    mv /your_home/monitoring/docker-compose-central.yml /your_home/monitoring/docker-compose.yml


Start the stack

    docker-compose up -d


Here are some grafana dashboards :

* [PostgreSQL overview](/resources/grafana-dashboard-pgsql-overview.json)
* [PostgreSQL advanced view](/resources/grafana-dashboard-pgsql-advanced.json)
* [PostgreSQL single table monitoring](/resources/grafana-dashboard-single-table.json)
* [Central server (influxdb + grafana) overview](/resources/grafana-dashboard-influxdb.json)


####Control influxdb

Identify the influxDB container ID with :

    docker ps
    
Then

    docker exec -it INFLUX_CONTAINER_ID bash
    
And 
    
    influx
    
To get the influx cli. Connect to the telegraf db by :

    use telegraf
    
Display the tables (measurements) :

    show measurements
    
Display a measurement content :

    select * from the_measurement where time > now() - 1h    
    


##Collector

We have to deploy a telegraf per target server.

####Docker-compose.yml

    version: '2'
    
    services:
    ## Telegraf container
        telegraf:
            image: telegraf:latest
            volumes:
                # standard config
                - ${PWD}/data/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
                # custom config - can't make it workw
    #            - ${PWD}/data/telegraf/telegraf-ext/custom.config:/etc/telegraf/telegraf.d/custom.conf
                - /var/run/docker.sock:/var/run/docker.sock:ro
            restart: always

####Installation

Install Docker / docker-compose if necessary : 

    install docker
    install docker-compose

Configure Docker to access network through proxy for example :

    sudo mkdir -p /etc/systemd/system/docker.service.d/
    sudo nano /etc/systemd/system/docker.service.d/docker.conf
    
    #Copy in this file something like
    [Service]
    ExecStart=
    Environment="HTTP_PROXY=the_proxy"
    
    sudo systemctl daemon-reload
    sudo systemctl restart docker.service

Pull images

    docker pull telegraf

Create sub-dirs

    mkdir -p /your_home/monitoring/data/telegraf/

Generate default config files

    cd /your_home/monitoring
    docker run --rm telegraf -sample-config > data/telegraf/telegraf.conf

General configuration, edit `telegraf.conf`

In [global_tags] part, add : 

    customer = "the_customer"

In [agent] part, set :

    interval = "60s"
    debug = true                    # might be useful !
    hostname = "the_host_name"    # or whatever

In [[outputs.influxdb]] part, set

    urls = ["http://influx.mymonitoring.com:80"]
    # If necessary
    http_proxy = "http://corporate.proxy:3128"

Copy the compose yaml

    /your_home/monitoring/docker-compose-telegraf.yml
    mv /your_home/monitoring/docker-compose-telegraf.yml /your_home/monitoring/docker-compose.yml

Start the stack

    docker-compose up -d

Check the logs. You should see :

    telegraf_1  | 2018/02/05 10:38:37 I! Using config file: /etc/telegraf/telegraf.conf
    telegraf_1  | 2018-02-05T10:38:37Z D! Attempting connection to output: influxdb
    telegraf_1  | 2018-02-05T10:38:37Z D! Successfully connected to output: influxdb
    telegraf_1  | 2018-02-05T10:38:37Z I! Starting Telegraf v1.5.2
    telegraf_1  | 2018-02-05T10:38:37Z I! Loaded outputs: influxdb
    telegraf_1  | 2018-02-05T10:38:37Z I! Loaded inputs: inputs.postgresql_extensible inputs.postgresql_extensible inputs.postgresql_extensible inputs.disk inputs.diskio inputs.kernel inputs.mem inputs.postgresql inputs.processes inputs.swap inputs.system inputs.cpu
    telegraf_1  | 2018-02-05T10:38:37Z I! Tags enabled: customer=the_customer host=the_host_name
    telegraf_1  | 2018-02-05T10:38:37Z I! Agent Config: Interval:1m0s, Quiet:false, Hostname:"the_host_name", Flush Interval:10s 


Ajouter des inputs à telegraf (change the database IP in each `[[inputs.postgresql]]` block):

    [[inputs.postgresql]]
      interval = "60s"
      ## specify address via a url matching:
      ##   postgres://[pqgotest[:password]]@localhost[/dbname]\
      ##       ?sslmode=[disable|verify-ca|verify-full]
      ## or a simple string:
      ##   host=localhost user=pqotest password=... sslmode=... dbname=app_production
      ##
      ## All connection parameters are optional.
      ##
      ## Without the dbname parameter, the driver will default to a database
      ## with the same name as the user. This dbname is just for instantiating a
      ## connection with the server and doesn't restrict the databases we are trying
      ## to grab metrics for.
      ##
      #address = "host=10.2.1.188 user=my_user sslmode=disable dbname=postgres"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=postgres"
      ## A  list of databases to explicitly ignore.  If not specified, metrics for all
      ## databases are gathered.  Do NOT use with the 'databases' option.
      # ignored_databases = ["postgres", "template0", "template1"]
      ignored_databases = ["postgres", "template0", "template1"]
      ## A list of databases to pull metrics about. If not specified, metrics for all
      ## databases are gathered.  Do NOT use with the 'ignored_databases' option.
    
    ## #######################################################
    ## Checkpoints
    ##
    [[inputs.postgresql_extensible]]
      interval = "300s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=postgres"
    [[inputs.postgresql_extensible.query]]
      #Checkpoints stats
      # checkpoints_timeout_pct | checkpoints_req_pct | avg_checkpoints_minutes | avg_checkpoint_write | total_written | checkpoint_write_pct | backend_write_pct | total_checkpoints | seconds_since_start | setting_checkpoint_timeout | setting_max_wal_size 
      sqlquery="SELECT (100*checkpoints_timed) / total_checkpoints as checkpoints_timeout_pct, 100 - (100*checkpoints_timed) / total_checkpoints AS checkpoints_req_pct, seconds_since_start / total_checkpoints / 60 AS avg_checkpoints_minutes, buffers_checkpoint * block_size / (checkpoints_timed + checkpoints_req) AS avg_checkpoint_write, block_size * (buffers_checkpoint + buffers_clean + buffers_backend) AS total_written, 100 * buffers_checkpoint / (buffers_checkpoint + buffers_clean + buffers_backend) AS checkpoint_write_pct, 100 * buffers_backend / (buffers_checkpoint + buffers_clean + buffers_backend) AS backend_write_pct, total_checkpoints,seconds_since_start, (select setting from pg_settings where name = 'checkpoint_timeout') as setting_checkpoint_timeout, (select (setting::bigint) * 16*1024*1024 from pg_settings where name = 'max_wal_size') as setting_max_wal_size FROM (SELECT EXTRACT(EPOCH FROM (now() - stats_reset)) AS seconds_since_start, *, (checkpoints_timed+checkpoints_req) AS total_checkpoints FROM pg_stat_bgwriter WHERE (checkpoints_timed+checkpoints_req)>0) AS sub, (SELECT cast(current_setting('block_size') AS integer) AS block_size) bs"
      version=901
      withdbname=false
      tagvalue=""
      measurement="pg_stat_checkpoints"
    
    ## #######################################################
    ## Sizes
    ##
    ## Create one new block per database
    ## Change de "dbName" in the "address" line
    ##
    [[inputs.postgresql_extensible]]
      interval = "1800s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=the_db"
    [[inputs.postgresql_extensible.query]]
      #Size stats
      sqlquery="SELECT (SELECT current_database()) as dbname, sum(pg_table_size(table_name))::bigint AS table_size, sum(pg_indexes_size(table_name))::bigint AS index_size FROM (SELECT ('' || table_schema || '.' || table_name || '') AS table_name FROM information_schema.tables WHERE table_schema='public') AS all_tables1"
      version=901
      withdbname=false
      tagvalue="dbname"
      measurement="pg_stat_size"
    
    ## #######################################################
    ## Bloating
    ##
    ## Create one new block per database
    ## Change de "dbName" in the "address" line
    ## 
    ## CAREFUL : user indexes query could be harmful for the DB load
    ##
    [[inputs.postgresql_extensible]]
      interval = "900s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=the_db"
    [[inputs.postgresql_extensible.query]]
      #Bloating tables system
      sqlquery="SELECT 'tables' as relation, 'system' as owner, (SELECT current_database()) as dbname, bloat_size::bigint, real_size::bigint, (bloat_size / real_size)::float as bloat_ratio FROM (SELECT SUM(bloat_bytes) as bloat_size FROM (SELECT  CASE WHEN (tblpages-est_tblpages_ff)*bs > 0 THEN ((tblpages-est_tblpages_ff)*bs)::bigint ELSE 0 END AS bloat_bytes, CASE WHEN tblpages - est_tblpages_ff > 0 THEN round((100 * (tblpages - est_tblpages_ff)/tblpages::float)::numeric, 1) ELSE 0 END AS bloat_ratio FROM (SELECT ceil( reltuples / ( (bs-page_hdr)/tpl_size ) ) + ceil( toasttuples / 4 ) AS est_tblpages, ceil( reltuples / ( (bs-page_hdr)*fillfactor/(tpl_size*100) ) ) + ceil( toasttuples / 4 ) AS est_tblpages_ff, tblpages, bs FROM (SELECT ( 4 + tpl_hdr_size + tpl_data_size + (2*ma) - CASE WHEN tpl_hdr_size%ma = 0 THEN ma ELSE tpl_hdr_size%ma END - CASE WHEN ceil(tpl_data_size)::int%ma = 0 THEN ma ELSE ceil(tpl_data_size)::int%ma END) AS tpl_size, (heappages + toastpages) AS tblpages, reltuples, toasttuples, bs, page_hdr, fillfactor FROM (SELECT ns.nspname AS schemaname, tbl.relname AS tblname, tbl.reltuples, tbl.relpages AS heappages, coalesce(toast.relpages, 0) AS toastpages, coalesce(toast.reltuples, 0) AS toasttuples, coalesce(substring(array_to_string(tbl.reloptions, ' ') FROM '%fillfactor=#\"__#\"%' FOR '#')::smallint, 100) AS fillfactor, current_setting('block_size')::numeric AS bs, CASE WHEN version()~'mingw32' OR version()~'64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END AS ma, 24 AS page_hdr, 23 + CASE WHEN MAX(coalesce(null_frac,0)) > 0 THEN ( 7 + count(*) ) / 8 ELSE 0::int END + CASE WHEN tbl.relhasoids THEN 4 ELSE 0 END AS tpl_hdr_size, sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 1024) ) AS tpl_data_size FROM pg_attribute AS att JOIN pg_class AS tbl ON att.attrelid = tbl.oid JOIN pg_namespace AS ns ON ns.oid = tbl.relnamespace JOIN pg_stats AS s ON s.schemaname=ns.nspname AND s.tablename = tbl.relname AND s.inherited=false AND s.attname=att.attname LEFT JOIN pg_class AS toast ON tbl.reltoastrelid = toast.oid WHERE att.attnum > 0 AND NOT att.attisdropped AND tbl.relkind = 'r' AND ns.nspname = 'pg_catalog' GROUP BY 1,2,3,4,5,6,7,8,9,10, tbl.relhasoids) AS s) AS s2) AS s3) AS s4) as s5, (SELECT sum(pg_table_size(table_name)) as real_size FROM (SELECT ('' || table_schema || '.' || table_name || '') AS table_name FROM information_schema.tables WHERE table_schema='pg_catalog') AS all_tables1) as p1"
      version=901
      withdbname=false
      tagvalue="relation,owner,dbname"
      measurement="pg_stat_bloat"
    [[inputs.postgresql_extensible.query]]
      #Bloating index system
      sqlquery="SELECT 'index' as relation, 'system' as owner, (SELECT current_database()) as dbname, bloat_size::bigint, real_size::bigint, (bloat_size / real_size)::float as bloat_ratio FROM (SELECT SUM(bloat_bytes) as bloat_size FROM (WITH btree_index_atts AS (SELECT nspname, relname, reltuples, relpages, indrelid, relam, regexp_split_to_table(indkey::text, ' ')::smallint AS attnum, indexrelid as index_oid FROM pg_index JOIN pg_class ON pg_class.oid=pg_index.indexrelid JOIN pg_namespace ON pg_namespace.oid = pg_class.relnamespace JOIN pg_am ON pg_class.relam = pg_am.oid WHERE pg_am.amname = 'btree' AND pg_namespace.nspname='pg_catalog'), index_item_sizes AS (SELECT i.nspname, i.relname, i.reltuples, i.relpages, i.relam, (quote_ident(s.schemaname) || '.' || quote_ident(s.tablename))::regclass AS starelid, a.attrelid AS table_oid, index_oid, current_setting('block_size')::numeric AS bs, CASE WHEN version() ~ 'mingw32' OR version() ~ '64-bit' THEN 8 ELSE 4 END AS maxalign, 24 AS pagehdr, CASE WHEN max(coalesce(s.null_frac,0)) = 0 THEN 2 ELSE 6 END AS index_tuple_hdr, sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 2048) ) AS nulldatawidth FROM pg_attribute AS a JOIN pg_stats AS s ON (quote_ident(s.schemaname) || '.' || quote_ident(s.tablename))::regclass=a.attrelid AND s.attname = a.attname  JOIN btree_index_atts AS i ON i.indrelid = a.attrelid AND a.attnum = i.attnum WHERE a.attnum > 0 GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9), index_aligned AS (SELECT bs, reltuples, relpages, relam, table_oid, ( 2 + maxalign - CASE WHEN index_tuple_hdr%maxalign = 0 THEN maxalign ELSE index_tuple_hdr%maxalign END + nulldatawidth + maxalign - CASE WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign ELSE nulldatawidth::integer%maxalign END)::numeric AS nulldatahdrwidth, pagehdr FROM index_item_sizes AS s1), otta_calc AS (SELECT bs, table_oid, relpages, coalesce(ceil((reltuples*(4+nulldatahdrwidth))/(bs-pagehdr::float)) + CASE WHEN am.amname IN ('hash','btree') THEN 1 ELSE 0 END , 0) AS otta FROM index_aligned AS s2 LEFT JOIN pg_am am ON s2.relam = am.oid), raw_bloat AS (SELECT CASE WHEN sub.relpages <= otta THEN 0 ELSE bs*(sub.relpages-otta)::bigint END AS wastedbytes FROM otta_calc AS sub JOIN pg_class AS c ON c.oid=sub.table_oid) SELECT wastedbytes as bloat_bytes FROM raw_bloat) as s1) as s2, (SELECT sum(pg_indexes_size(table_name)) as real_size FROM (SELECT ('' || table_schema || '.' || table_name || '') AS table_name FROM information_schema.tables WHERE table_schema='pg_catalog') AS all_tables1) as p1"
      version=901
      withdbname=false
      tagvalue="relation,owner,dbname"
      measurement="pg_stat_bloat"
    [[inputs.postgresql_extensible.query]]
      #Bloating tables user
      sqlquery="SELECT 'tables' as relation, 'user' as owner, (SELECT current_database()) as dbname, bloat_size::bigint, real_size::bigint, (bloat_size / real_size)::float as bloat_ratio FROM (SELECT SUM(bloat_bytes) as bloat_size FROM (SELECT  CASE WHEN (tblpages-est_tblpages_ff)*bs > 0 THEN ((tblpages-est_tblpages_ff)*bs)::bigint ELSE 0 END AS bloat_bytes, CASE WHEN tblpages - est_tblpages_ff > 0 THEN round((100 * (tblpages - est_tblpages_ff)/tblpages::float)::numeric, 1) ELSE 0 END AS bloat_ratio FROM (SELECT ceil( reltuples / ( (bs-page_hdr)/tpl_size ) ) + ceil( toasttuples / 4 ) AS est_tblpages, ceil( reltuples / ( (bs-page_hdr)*fillfactor/(tpl_size*100) ) ) + ceil( toasttuples / 4 ) AS est_tblpages_ff, tblpages, bs FROM (SELECT ( 4 + tpl_hdr_size + tpl_data_size + (2*ma) - CASE WHEN tpl_hdr_size%ma = 0 THEN ma ELSE tpl_hdr_size%ma END - CASE WHEN ceil(tpl_data_size)::int%ma = 0 THEN ma ELSE ceil(tpl_data_size)::int%ma END) AS tpl_size, (heappages + toastpages) AS tblpages, reltuples, toasttuples, bs, page_hdr, fillfactor FROM (SELECT ns.nspname AS schemaname, tbl.relname AS tblname, tbl.reltuples, tbl.relpages AS heappages, coalesce(toast.relpages, 0) AS toastpages, coalesce(toast.reltuples, 0) AS toasttuples, coalesce(substring(array_to_string(tbl.reloptions, ' ') FROM '%fillfactor=#\"__#\"%' FOR '#')::smallint, 100) AS fillfactor, current_setting('block_size')::numeric AS bs, CASE WHEN version()~'mingw32' OR version()~'64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END AS ma, 24 AS page_hdr, 23 + CASE WHEN MAX(coalesce(null_frac,0)) > 0 THEN ( 7 + count(*) ) / 8 ELSE 0::int END + CASE WHEN tbl.relhasoids THEN 4 ELSE 0 END AS tpl_hdr_size, sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 1024) ) AS tpl_data_size FROM pg_attribute AS att JOIN pg_class AS tbl ON att.attrelid = tbl.oid JOIN pg_namespace AS ns ON ns.oid = tbl.relnamespace JOIN pg_stats AS s ON s.schemaname=ns.nspname AND s.tablename = tbl.relname AND s.inherited=false AND s.attname=att.attname LEFT JOIN pg_class AS toast ON tbl.reltoastrelid = toast.oid WHERE att.attnum > 0 AND NOT att.attisdropped AND tbl.relkind = 'r' AND ns.nspname = 'public' GROUP BY 1,2,3,4,5,6,7,8,9,10, tbl.relhasoids) AS s) AS s2) AS s3) AS s4) as s5, (SELECT sum(pg_table_size(table_name)) as real_size FROM (SELECT ('' || table_schema || '.' || table_name || '') AS table_name FROM information_schema.tables WHERE table_schema='public') AS all_tables1) as p1"
      version=901
      withdbname=false
      tagvalue="relation,owner,dbname"
      measurement="pg_stat_bloat"
    [[inputs.postgresql_extensible.query]]
      #Bloating index user
      sqlquery="SELECT 'index' as relation, 'user' as owner, (SELECT current_database()) as dbname, bloat_size::bigint, real_size::bigint, (bloat_size / real_size)::float as bloat_ratio FROM (SELECT SUM(bloat_bytes) as bloat_size FROM (WITH btree_index_atts AS (SELECT nspname, relname, reltuples, relpages, indrelid, relam, regexp_split_to_table(indkey::text, ' ')::smallint AS attnum, indexrelid as index_oid FROM pg_index JOIN pg_class ON pg_class.oid=pg_index.indexrelid JOIN pg_namespace ON pg_namespace.oid = pg_class.relnamespace JOIN pg_am ON pg_class.relam = pg_am.oid WHERE pg_am.amname = 'btree' AND pg_namespace.nspname='public'), index_item_sizes AS (SELECT i.nspname, i.relname, i.reltuples, i.relpages, i.relam, (quote_ident(s.schemaname) || '.' || quote_ident(s.tablename))::regclass AS starelid, a.attrelid AS table_oid, index_oid, current_setting('block_size')::numeric AS bs, CASE WHEN version() ~ 'mingw32' OR version() ~ '64-bit' THEN 8 ELSE 4 END AS maxalign, 24 AS pagehdr, CASE WHEN max(coalesce(s.null_frac,0)) = 0 THEN 2 ELSE 6 END AS index_tuple_hdr, sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 2048) ) AS nulldatawidth FROM pg_attribute AS a JOIN pg_stats AS s ON (quote_ident(s.schemaname) || '.' || quote_ident(s.tablename))::regclass=a.attrelid AND s.attname = a.attname  JOIN btree_index_atts AS i ON i.indrelid = a.attrelid AND a.attnum = i.attnum WHERE a.attnum > 0 GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9), index_aligned AS (SELECT bs, reltuples, relpages, relam, table_oid, ( 2 + maxalign - CASE WHEN index_tuple_hdr%maxalign = 0 THEN maxalign ELSE index_tuple_hdr%maxalign END + nulldatawidth + maxalign - CASE WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign ELSE nulldatawidth::integer%maxalign END)::numeric AS nulldatahdrwidth, pagehdr FROM index_item_sizes AS s1), otta_calc AS (SELECT bs, table_oid, relpages, coalesce(ceil((reltuples*(4+nulldatahdrwidth))/(bs-pagehdr::float)) + CASE WHEN am.amname IN ('hash','btree') THEN 1 ELSE 0 END , 0) AS otta FROM index_aligned AS s2 LEFT JOIN pg_am am ON s2.relam = am.oid), raw_bloat AS (SELECT CASE WHEN sub.relpages <= otta THEN 0 ELSE bs*(sub.relpages-otta)::bigint END AS wastedbytes FROM otta_calc AS sub JOIN pg_class AS c ON c.oid=sub.table_oid) SELECT wastedbytes as bloat_bytes FROM raw_bloat) as s1) as s2, (SELECT sum(pg_indexes_size(table_name)) as real_size FROM (SELECT ('' || table_schema || '.' || table_name || '') AS table_name FROM information_schema.tables WHERE table_schema='public') AS all_tables1) as p1"
      version=901
      withdbname=false
      tagvalue="relation,owner,dbname"
      measurement="pg_stat_bloat"
    
    ## #######################################################
    ## Cache hit
    ##
    ## Create one new block per database
    ## Change de "dbName" in the "address" line
    ##
    [[inputs.postgresql_extensible]]
      interval = "30s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=the_db"
    [[inputs.postgresql_extensible.query]]
      #Cache hits stats
      sqlquery="WITH stats_heap AS (SELECT (SELECT sum(heap_blks_read) FROM pg_statio_user_tables)::bigint as heap_read, (SELECT sum(heap_blks_hit) FROM pg_statio_user_tables)::bigint as heap_hit), stats_idx AS (SELECT (SELECT sum(idx_blks_read) FROM pg_statio_user_indexes)::bigint as idx_read, (SELECT sum(idx_blks_hit) FROM pg_statio_user_indexes)::bigint as idx_hit), bs AS (SELECT cast(current_setting('block_size') AS integer) AS block_size) SELECT current_database() AS dbname, (heap_read * block_size)::bigint AS heap_read, (heap_hit * block_size)::bigint AS heap_hit, ((heap_hit::float - heap_read::float) / heap_hit::float)::float as heap_ratio, (idx_read * block_size)::bigint AS idx_read, (idx_hit * block_size)::bigint AS idx_hit, ((idx_hit::float - idx_read::float) / idx_hit::float)::float as idx_ratio FROM bs, stats_heap, stats_idx"
      version=901
      withdbname=false
      tagvalue="dbname"
      measurement="pg_stat_cache"
    
    ## #######################################################
    ## PostgreSQL Server settings
    ##
    [[inputs.postgresql_extensible]]
      interval = "3600s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=postgres"
    [[inputs.postgresql_extensible.query]]
      #Settings
      sqlquery="select name, setting, unit, category from pg_settings"
      version=901
      withdbname=false
      tagvalue="name"
      measurement="pg_settings"
    
    ## #######################################################
    ## Single table statistics
    ##
    ## General stats
    ##
    [[inputs.postgresql_extensible]]
      interval = "60s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=the_db"
    [[inputs.postgresql_extensible.query]]
      sqlquery="select (SELECT current_database()) as db_name, 'probes_global_statistics' as table_name, seq_scan::bigint, seq_tup_read::bigint, idx_scan::bigint, idx_tup_fetch::bigint as idx_tup_read, n_tup_ins::bigint, n_tup_del::bigint, (n_tup_upd + n_tup_hot_upd)::bigint as n_tup_total_upd, (CASE WHEN (n_tup_hot_upd + n_tup_upd) > 0 THEN n_tup_hot_upd::float / (n_tup_hot_upd + n_tup_upd)::float ELSE 0 END)::float as hot_upd_ratio, n_live_tup::bigint, n_dead_tup::bigint, (n_dead_tup::float / (n_live_tup + n_dead_tup)::float) as bloating_ratio, (vacuum_count + autovacuum_count)::bigint as total_vacuum, (analyze_count + autoanalyze_count)::bigint as total_analyze, EXTRACT(EPOCH FROM (now() - CASE WHEN COALESCE(last_autovacuum, to_timestamp(0)) < COALESCE(last_vacuum, to_timestamp(0)) THEN COALESCE(last_vacuum, to_timestamp(0)) ELSE COALESCE(last_autovacuum, to_timestamp(0)) END)) as duration_since_last_vacuum, EXTRACT(EPOCH FROM (now() - CASE WHEN COALESCE(last_autoanalyze, to_timestamp(0)) < COALESCE(last_analyze, to_timestamp(0)) THEN COALESCE(last_analyze, to_timestamp(0)) ELSE COALESCE(last_autoanalyze, to_timestamp(0)) END)) as duration_since_last_analyze from pg_stat_user_tables where relname='probes_global_statistics'"
      version=901
      withdbname=false
      tagvalue="table_name,db_name"
      measurement="pg_stat_single_table"

    ## #######################################################
    ## Single table statistics
    ##
    ## Cache usage
    ##
    [[inputs.postgresql_extensible]]
      interval = "60s"
      address = "host=10.xx.yy.196 user=my_user password=my_password sslmode=disable dbname=the_db"
    [[inputs.postgresql_extensible.query]]
      # single table cache usage
      sqlquery="WITH stats_heap AS (SELECT (SELECT heap_blks_read FROM pg_statio_user_tables where relname='probes_global_statistics')::bigint as heap_read, (SELECT heap_blks_hit FROM pg_statio_user_tables where relname='probes_global_statistics')::bigint as heap_hit), stats_idx AS (SELECT (SELECT sum(idx_blks_read) FROM pg_statio_user_indexes where relname='probes_global_statistics')::bigint as idx_read, (SELECT sum(idx_blks_hit) FROM pg_statio_user_indexes where relname='probes_global_statistics')::bigint as idx_hit), bs AS (SELECT cast(current_setting('block_size') AS integer) AS block_size) SELECT (SELECT current_database()) as db_name, 'probes_global_statistics' as table_name, (heap_read * block_size)::bigint AS heap_read, (heap_hit * block_size)::bigint AS heap_hit, (heap_hit::float / (heap_read + heap_hit)::float)::float as heap_ratio, (idx_read * block_size)::bigint AS idx_read, (idx_hit * block_size)::bigint AS idx_hit, (idx_hit::float / (idx_read + idx_hit)::float)::float as idx_ratio FROM bs, stats_heap, stats_idx"
      version=901
      withdbname=false
      tagvalue="table_name,db_name"
      measurement="pg_stat_single_table_cache"


Example of telegraf configuration :

* for a single database server : [here](/resources/telegraf-single-database.conf) 



