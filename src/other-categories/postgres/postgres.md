# Postgres

## 利用 Docker 快速在 **当前工作目录(`pwd`)** 启动一个本地 postgres 环境

使用 alpine based image，并且挂载了 unix socket。

```bash
$ mkdir conf
$ docker run -i --rm postgres:15-alpine cat /usr/local/share/postgresql/postgresql.conf.sample > conf/postgresql.conf
$ docker run -d \
-v `pwd`/conf/postgresql.conf:/etc/postgresql/postgresql.conf \
-v `pwd`/data:/var/lib/postgresql/data \
-v /var/run/postgresql:/var/run/postgresql \
-p 5432:5432 \
-e POSTGRES_USER=<YOUR USER NAME> \
-e LANG=zh_CN.utf8 \
-e POSTGRES_INITDB_ARGS="--locale-provider=icu --icu-locale=zh-CN" \
-e POSTGRES_PASSWORD=<YOUR PASSWORD> \
postgres:15-alpine -c 'config_file=/etc/postgresql/postgresql.conf'
```

## 使用 `pgcli` 来获得拥有更友好的客户端

```bash
$ sudo pacman -S pgcli
# -or-
$ sudo apt-get install pgcli
# -or-
$ brew install pgcli
# -or-
$ pip install -U pgcli
```

### 默认启动直接通过 unix socket 连接 postgres

```bash
$ pgcli
Server: PostgreSQL 15.1
Version: 3.5.0
Home: http://pgcli.com
m4n5ter> exit
Goodbye!
```

## Postgres 一般应该调整的参数

此处内容来自:
https://juejin.cn/post/6903313925727584264

官方文档:
https://www.postgresql.org/docs/current/admin.html
中文社区翻译:
http://www.postgres.cn/

#### max_connections

允许的最大客户端连接数。这个参数设置大小和`work_mem`有一些关系。配置的越高，可能会占用系统更多的内存。通常可以设置数百个连接，如果要使用上千个连接，建议配置连接池来减少开销。

#### shared_buffers

Postgres 使用自己的缓冲区，也使用Linux操作系统内核缓冲 OS Cache。这就说明数据两次存储在内存中，首先是 Postgres 缓冲区，然后是操作系统内核缓冲区。与其他数据库不同，Postgres 不提供直接IO，所以这又被称为双缓冲。Postgres 缓冲区称为`shared_buffer`，建议设置为物理内存的 1/4。而实际配置取决于硬件配置和工作负载，如果你的内存很大，而你又想多缓冲一些数据到内存中，可以继续调大`shared_buffer`。  

#### effective_cache_size

这个参数主要用于 Postgres 查询优化器。是单个查询可用的磁盘高速缓存的有效大小的一个假设，是一个估算值，它并不占据系统内存。由于优化器需要进行估算成本，较高的值更有可能使用索引扫描，较低的值则有可能使用顺序扫描。一般这个值设置为内存的 1/2 是正常保守的设置，设置为内存的 3/4 是比较推荐的值。通过free命令查看操作系统的统计信息，您可能会更好的估算该值。

```shell
[pg@e22 ~]$ free -g
              total        used        free      shared  buff/cache   available
Mem:             62           2           5          16          55          40
Swap:             7           0           7
```

#### work_mem

这个参数主要用于写入临时文件之前内部排序操作和散列表使用的内存量，增加`work_mem`参数将使Postgres 可以进行更大的内存排序。这个参数和`max_connections`有一些关系，假设你设置为 30MB，则 40 个用户同时执行查询排序，很快就会使用 1.2GB 的实际内存。同时对于复杂查询，可能会运行多个排序和散列操作，例如涉及到8张表进行合并排序，此时就需要 8 倍的`work_mem`。

如下面案例所示，该环境使用 4MB 的`work_mem`，在执行排序操作的时候，使用的`Sort Method`是`external merge Disk`。

```sql
kms=> explain (analyze,buffers) select * from KMS_BUSINESS_HALL_TOTAL  order by buss_query_info;
                                                                       QUERY PLAN                                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=262167.99..567195.15 rows=2614336 width=52) (actual time=2782.203..5184.442 rows=3137204 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=68 read=25939, temp read=28863 written=28947
   ->  Sort  (cost=261167.97..264435.89 rows=1307168 width=52) (actual time=2760.566..3453.783 rows=1045735 loops=3)
         Sort Key: buss_query_info
         Sort Method: external merge  Disk: 50568kB
         Worker 0:  Sort Method: external merge  Disk: 50840kB
         Worker 1:  Sort Method: external merge  Disk: 49944kB
         Buffers: shared hit=68 read=25939, temp read=28863 written=28947
         ->  Parallel Seq Scan on kms_business_hall_total  (cost=0.00..39010.68 rows=1307168 width=52) (actual time=0.547..259.524 rows=1045735 loops=3)
               Buffers: shared read=25939
 Planning Time: 0.540 ms
 Execution Time: 5461.516 ms
(14 rows)
复制代码
```

当我们把参数修改成 512MB 的时候，可以看到`Sort Method`变成了`quicksort Memory`，变成了内存排序。

```sql
kms=> set work_mem to "512MB";
SET
kms=> explain (analyze,buffers) select * from KMS_BUSINESS_HALL_TOTAL  order by buss_query_info;
                                                                QUERY PLAN                                                                
------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=395831.79..403674.80 rows=3137204 width=52) (actual time=7870.826..8204.794 rows=3137204 loops=1)
   Sort Key: buss_query_info
   Sort Method: quicksort  Memory: 359833kB
   Buffers: shared hit=25939
   ->  Seq Scan on kms_business_hall_total  (cost=0.00..57311.04 rows=3137204 width=52) (actual time=0.019..373.067 rows=3137204 loops=1)
         Buffers: shared hit=25939
 Planning Time: 0.081 ms
 Execution Time: 8419.994 ms
(8 rows)
复制代码
```

#### maintenance_work_mem

指定维护操作使用的最大内存量，例如（Vacuum、Create Index和Alter Table Add Foreign Key），默认值是 64MB。由于通常正常运行的数据库中不会有大量并发的此类操作，可以设置的较大一些，提高清理和创建索引外键的速度。

```sql
postgres=# set maintenance_work_mem to "64MB";
SET
Time: 1.971 ms
postgres=# create index idx1_test on test(id);
CREATE INDEX
Time: 7483.621 ms (00:07.484)
postgres=# set maintenance_work_mem to "2GB";
SET
Time: 0.543 ms
postgres=# drop index idx1_test;
DROP INDEX
Time: 133.984 ms
postgres=# create index idx1_test on test(id);
CREATE INDEX
Time: 5661.018 ms (00:05.661)
复制代码
```

可以看到在使用默认的 64MB 创建索引，速度为 7.4 秒，而设置为 2GB 后，创建速度是 5.6 秒

#### wal_sync_method

每次发生事务后，Postgres 会强制将提交写到 WAL 日志的方式。可以使用`pg_test_fsync`命令在你的操作系统上进行测试，`fdatasync`是 Linux 上的默认方法。如下所示，我的环境测试下来`fdatasync`还是速度可以的。不支持的方法像`fsync_writethrough`直接显示`n/a`。

```shell
postgres=# show wal_sync_method ;
 wal_sync_method 
-----------------
 fdatasync
(1 row)

[pg@e22 ~]$ pg_test_fsync -s 3
3 seconds per test
O_DIRECT supported on this platform for open_datasync and open_sync.

Compare file sync methods using one 8kB write:
(in wal_sync_method preference order, except fdatasync is Linux's default)
        open_datasync                      4782.871 ops/sec     209 usecs/op
        fdatasync                          4935.556 ops/sec     203 usecs/op
        fsync                              3781.254 ops/sec     264 usecs/op
        fsync_writethrough                              n/a
        open_sync                          3850.219 ops/sec     260 usecs/op

Compare file sync methods using two 8kB writes:
(in wal_sync_method preference order, except fdatasync is Linux's default)
        open_datasync                      2469.646 ops/sec     405 usecs/op
        fdatasync                          4412.266 ops/sec     227 usecs/op
        fsync                              3432.794 ops/sec     291 usecs/op
        fsync_writethrough                              n/a
        open_sync                          1929.221 ops/sec     518 usecs/op

Compare open_sync with different write sizes:
(This is designed to compare the cost of writing 16kB in different write
open_sync sizes.)
         1 * 16kB open_sync write          3159.780 ops/sec     316 usecs/op
         2 *  8kB open_sync writes         1944.723 ops/sec     514 usecs/op
         4 *  4kB open_sync writes          993.173 ops/sec    1007 usecs/op
         8 *  2kB open_sync writes          493.396 ops/sec    2027 usecs/op
        16 *  1kB open_sync writes          249.762 ops/sec    4004 usecs/op

Test if fsync on non-write file descriptor is honored:
(If the times are similar, fsync() can sync data written on a different
descriptor.)
        write, fsync, close                3719.973 ops/sec     269 usecs/op
        write, close, fsync                3651.820 ops/sec     274 usecs/op

Non-sync'ed 8kB writes:
        write                            400577.329 ops/sec       2 usecs/op
复制代码
```

#### wal_buffers

事务日志缓冲区的大小，Postgres 将 WAL 记录写入缓冲区，然后再将缓冲区刷新到磁盘。在PostgreSQL 12版中，默认值为 `-1`，也就是选择等于 `shared_buffers` 的 1/32 。如果自动的选择太大或太小可以手工设置该值。一般考虑设置为 16MB。

#### synchronous_commit

客户端执行提交，并且等待 WAL 写入磁盘之后，然后再将成功状态返回给客户端。可以设置为 `on`，`remote_apply`，`remote_write`，`local`，`off` 等值。默认设置为 `on`。如果设置为`off`，会关闭`sync_commit`，客户端提交之后就立马返回，不用等记录刷新到磁盘。此时如果PostgreSQL实例崩溃，则最后几个异步提交将会丢失。

#### default_statistics_target

PostgreSQL 使用统计信息来生成执行计划。统计信息可以通过手动`Analyze`命令或者是`autovacuum`进程启动的自动分析来收集，`default_statistics_target`参数指定在收集和记录这些统计信息时的详细程度。默认值为`100`对于大多数工作负载是比较合理的，对于非常简单的查询，较小的值可能会有用，而对于复杂的查询（尤其是针对大型表的查询），较大的值可能会更好。为了不要一刀切，可以使用`ALTER TABLE .. ALTER COLUMN .. SET STATISTICS`覆盖特定表列的默认收集统计信息的详细程度。

#### checkpoint_timeout、max_wal_size，min_wal_size、checkpoint_completion_target

了解这两个参数以前，首先我们来看一下，触发检查点的几个操作。

-   直接执行`checkpoint`命令
-   执行需要检查点的命令（例如`pg_start_backup`,`Create database`,`pg_ctl stop/start`等等）
-   自上一个检查点以来，达到了已经配置的时间量（`checkpoint_timeout` ）
-   自上一个检查点以来生成的`WAL`数量（`max_wal_size`）

使用默认值，检查点将在`checkpoint_timeout=5min`。也就是每 5 分钟触发一次。而[max_wal_size](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-MAX-WAL-SIZE)设置是自动检查点之间增长的最大预写日志记录（WAL）量。默认是 1GB，如果超过了 1GB，则会发生检查点。这是一个软限制。在一个特殊的情况下，比如系统遭遇到短时间的高负载，日志产生几秒种就可以达到 1GB，这个速度已经明显超过了`checkpoint_timeout` ，`pg_wal`目录的大小会急剧增加。此时我们可以从日志中看到相关类似的警告。

```sql
LOG:  checkpoints are occurring too frequently (9 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
LOG:  checkpoints are occurring too frequently (2 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
复制代码
```

所以要合理配置`max_wal_size`，以避免频繁的进行检查点。一般推荐设置为 16GB 以上，不过具体设置多大还需要和工作负荷相匹配。

`min_wal_size`参数是只要 WAL 磁盘使用量保持在这个设置之下，在做检查点时，旧的 WAL 文件总是被回收以便未来使用，而不是直接被删除。

而检查点的写入不是全部立马完成的，PostgreSQL 会将一次检查点的所有操作分散到一段时间内。这段时间由参数`checkpoint_completion_target`控制，它是一个分数，默认为 0.5。也就是在两次检查点之间的 0.5 比例完成写盘操作。如果设置的很小，则检查点进程就会更加迅速的写盘，设置的很大，则就会比较慢。一般推荐设置为 0.9，让检查点的写入分散一点。但是缺点就是出现故障的时候，影响恢复的时间。

### linux navicat reset

下面的方法不会丢失已经存在连接(Navicat 16 Premium)：

```bash
#!/bin/bash

# Backup
cp ~/.config/dconf/user ~/.config/dconf/user.bk
cp ~/.config/navicat/Premium/preferences.json ~/.config/navicat/Premium/preferences.json.bk

# Clear data in dconf
dconf reset -f /com/premiumsoft/navicat-premium/
# Remove data fields in config file
sed -i -E 's/,?"([A-Z0-9]+)":\{([^\}]+)},?//g' ~/.config/navicat/Premium/preferences.json## Links
```

* [Postgres Current Version Document](https://www.postgresql.org/docs/current)
