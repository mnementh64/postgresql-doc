#Tuning Linux for PostgreSQL 

A reference conf could be found [here](https://www.youtube.com/watch?v=Lbx-JVcGIFo) and its slides [here](https://www.percona.com/live/data-performance-conference-2016/sites/default/files/slides/pl_2016_kosmodemiansky.pdf). 
Good advices [there](https://www.postgresql-sessions.org/_media/en/5/performances_systeme_pg.pdf).

##Kernel cache tuning

Kernel parameters `dirty*` manage the cache behavior. A good understanding of how cache works could be found [there](https://lonesysadmin.net/2013/12/22/better-linux-disk-caching-performance-vm-dirty_ratio/).  

By default :

```bash
scaillet@Sylvain ~ $ sudo sysctl -a | grep dirty

vm.dirty_background_bytes = 0       # ratio limit applies
vm.dirty_background_ratio = 10      # 10% of RAM is available for cache --> huge on modern servers ! 
vm.dirty_bytes = 0                  # ratio limit applies
vm.dirty_expire_centisecs = 3000    # cache has to flush at least every 30s
vm.dirty_ratio = 20                 # hard limit of 20% of RAM
vm.dirty_writeback_centisecs = 500  
vm.dirtytime_expire_seconds = 43200
```

They are set for basic OS usage. In case of Database server with high workload, it should be tuned not to reduce the impact of [the checkpoint tuning](/postgresql/statistics/#checkpoint-and-linux-cache), for example.

One of the main drawback of bad tuning is heavy I/O pikes. They could occured because :

* the cache write asynchroneously most of the time
* lot of I/O overwhelm the cache
* the cache decided to flush everything by turning writings synchroneously  

If we force the cache to flush its data more frequently, it might reduce the heavy pike cases.

You can also see statistics on the page cache in /proc/vmstat:

    $ cat /proc/vmstat | egrep "dirty|writeback"
     nr_dirty 878
     nr_writeback 0
     nr_writeback_temp 0

In my case I have 878 dirty pages waiting to be written to disk.

Typically for our usage (heavy workload), a good way is to reduce the cache size by lowering ratios :

    vm.dirty_background_ratio = 5
    vm.dirty_ratio = 10

Another approch is given by Ilya Kosmodemiansky in [the slide 39](https://www.percona.com/live/data-performance-conference-2016/sites/default/files/slides/pl_2016_kosmodemiansky.pdf).

    vm.dirty_background_bytes = 67108864
    vm.dirty_bytes = 536870912 

These values looks more reasonable for RAID with 512MB cache on board.

In fact, it's good to configure the cache to have max some hundreds of MB to synchronize, no more : less than 1GB.


**What on VM ??**
In the following [slides](https://www.postgresql-sessions.org/_media/en/5/performances_systeme_pg.pdf), the 13th gives a definitive advice : no VM for Postgres !
 

####Kernel other parameters

On dedicated servers, we could tune some other kernel parameters.

* Swap is bad for SGBD. Set : 

    vm.swappiness = 5 # or 10
    
* Overcommit is also bad. Set : 

    vm.overcommit_memory = 2 # to deactivate
    vm.overcommit_ratio = (RAM – SWAP) * 100 / RAM

To know how much swap is configured on a server, type `swapon -s` to display the size in Kbytes.

To learn more about overcommit values, see the [linux documentation](https://www.kernel.org/doc/Documentation/vm/overcommit-accounting).
Possible values are : 

    0: heuristic overcommit (this is the default)
    1: always overcommit, never check
    2: always check, never overcommit

    
* NUMA architecture is also bad for PostgreSQL : 

    vm.zone_reclaim_mode = 0 # to deactivate
    
    
    