### 2.2节
2.2节介绍了InnoDB的版本，但是从我搜索得到的结果来看，至少冲5.7开始，InnoDB的版本号跟MySQL版本号是一致的。使用命令`SHOW VARIABLES LIKE "innodb_version"`在MySQL5.7.30中可以看到InnoDB的版本是5.7.30。
### 2.3节
![figure2.1.png "InnoDB存储引擎体系结构"](figure2.1.png "InnoDB存储引擎体系结构")
InnoDB存储引擎有多个内存块，这些组成了一个内存池，主要作用有
* 维护所有进程/线程需要访问的多个内部数据结构
* 缓存磁盘数据，同时在真正修改磁盘文件的数据前会在这里缓存
* 重做日志(redo log)缓冲。
* ...
后台线程工作有：
* 刷新内存池数据，确保缓冲池中内存数据最新
* 将修改的数据文件刷新到磁盘文件
* 当数据库异常时，保证InnoDB能恢复到正常运行状态
InnoDB是多线程模型，包括：
1. Master Thread
非常核心的后台线程，主要负责：将缓冲池脏页数据落盘，保证数据一致性包括页刷新、合并插入缓冲(INSERT BUFFER)、UNDO页回收等。
2. IO Thread
InnoDB大量使用AIO(Async I/O)处理写I/O请求(注：5.7文档中说的是在Linux上使用异步I/O子系统(native AIO)执行read-ahead以及对数据文件页的写请求。对于其它类Unix系统，InnoDB使用只使用同步I/O，历史的，在Windows上只使用异步I/O。在Linux上使用异步I/O子系统需要`libaio`库)，IO Thread工作主要负责这些IO请求的回调(call back)处理。InnoDB包含write，read，insert buffer和log IO thread。前两个默认是4个线程，最多64个，最少一个，通过`	innodb_read_io_threads`和`	innodb_write_io_threads`参数设置。命令`show engine innodb status`可以查看InnoDB的IO threads。
```
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
```
注意，读线程的id总小于写线程。
3. purge thread
事务提交后可能不再需要的undo log，Purge Thread用来回收已经使用并分配的undo页。早期purge操作只能在Master thread中完成。后面把purge操作抽离到单独线程中进行减轻Master Thread工作，提高CPU使用率和存储引擎性能。在MySQL配置文件中加入
```
[mysqld]
innodb_purge_threads=1
```
启动独立purge thread。并且，从InnoDB 1.2(注意，InnoDB现在可能已经没有单独的版本号，这个其实也是较老版本的InnoDB版本了)开始，可以设置多个Purge Thread，进一步加快undo页回收，同时Purge Thread会随机读取undo页，多个purge thread可以进一步利用磁盘随机读取性能。使用命令`show variables like 'innodb_purge_threads'`查看信息。
4. Page Cleaner Thread
这个线程的作用是将之前在Master Thread中脏页的刷新操作移动到这个线程中执行。`show variables like 'innodb_page_cleaners'`可以查看相关信息。

InnoDB本身是基于磁盘存储记录按照页的方式管理，但是会使用内存缓冲池提高效率。在数据库中，读取页操作首先会把读到的页放到缓冲池，称之为"FIX"到缓冲池。下次在读取相同的页，先去缓冲池中找，有称之为命中，直接读取，没有命中，再去硬盘读。修改数据库页内容时，也是修改缓冲池缓存的页，再以一定频率落盘。落盘操作并不是在每次页更新时触发，而是通过Checkpoint机制落盘。InnoDB使用`innodb_buffer_pool_size`设置缓存大小.命令`show variables like 'innodb_buffer_pool_size'`可以查看具体值(以字节为单位)。也可以通过`information_schema`架构下的表`innodb_buffer_pool_stats`查看缓冲池状态。

注：使用表查询，结果是以页为单位，这里，页的大小不是linux系统的页大小(使用`getconf PAGE_SIZE`可以得到，一般是4KB)，是MySQL自己的页大小，默认是16KB，可以修改。

缓冲池包含的数据页有：索引页，数据页，undo页，插入缓冲(insert buffer)，自适应哈希索引(adaptive hash index)，InnoDB存储的锁信息(lock info)和数据字典信息(data dictionary)。

注：官方文档中，内存结构包含buffer pool，change buffer，adaptive hash index，log buffer。其中buffer pool包含表和索引数据。对没有缓存在buffer pool中的二级索引页的修改会被缓存到change buffer，这些buffer会被DML操作修改，在这些页由于其它读操作被加载到buffer pool会被合并。自适应哈希索引是的MySQL更像一个内存数据库且不丢失数据库事务特性和可靠性。log buffer是一块存放要写入到硬盘中的日志文件的内存缓存。参见[此处](https://dev.mysql.com/doc/refman/5.7/en/innodb-in-memory-structures.html)了解各个分节。

![figure2.2.png "inndb in-memory structure(左)"](figure2.2.png "inndb in-memory structure(左)")

![figure2.3.png "书中总结的内存结构"](figure2.3.png "书中总结的内存结构")

数据库缓冲池使用LRU算法管理。缓冲池页大小默认是16KB。InnoDB的存储引擎对LRU做了修改，增加了midpoint位置。通常的LRU，最频繁使用的在前端，最少使用的页在尾端，如果新的页无法放到缓冲池中，首先释放LRU列表尾端的页。InnoDB对于新加入的页不放到首端而是放到midpoint处，这叫做midpoint insert strategy。默认是LRU列表长度5/8处。`innodb_old_blocks_pct`可以设置该值。InnoDB对与midpoint之后的列表成为old列表，midpoint之前的列表成为new列表。new列表基本就是最活跃的热点数据。

之所以使用这种方式，是防止在特定情况下SQL会导致缓冲池的页都被刷出。常见于索引或者数据扫描操作。这些操作会访问表的很多页(包括索引页和数据页)，这些数据仅在此次查询使用，并不是真的活跃数据，如果都放到首部，就会导致热点数据被移出LRU中。InnoDB通过参数`innodb_old_blocks_time`进一步管理LRU。这个值表示页读取到mid位置要等多久才会加入到LRU中的new列表。

LRU管理的是已经读取的页，初始时没有数据，页在Free列表中。当需要从缓冲池分配页，先从Free中查找可用空闲页，存在就从Free删除，放入到LRU中。如果没有空闲页，就从LRU中淘汰一个页将这个页的空间分配给新的页。当页从LRU的old移动到new，这一操作称为`page made young`，而由于没有满足`innodb_old_blocks_time`而导致页留在old列表的操作叫做`page not made young`。命令`show engine innodb status`可以查看LRU和Free列表情况。其中Buffer pool size表示有多少页，Free buffer表示Free列表页大小，Database pages表示LRU列表页数量。Buffer pool hit rate是一个需要关注的值，它表示缓存池命中率，不要小于95%。`information_schema`架构下的`innodb_buffer_page_lru`可以查看LRU每个页的信息。

InnoDB也支持压缩页，支持1KB，2KB，4KB和8KB的页大小，这些页是同unzip_LRU列表管理的。LRU包含unzip_LRU的页。假定我们请求4KB的页，unzip_LRU分配内存的策略是：
1. 检查4KB的unzip_LRU是否有空闲页；
2. 有，使用；
3. 没有，检查8KB的unzip_LRU；
4. 如果有空闲页，分成2个4KB的页，加入到4KB的unzip_LRU列表中；
5. 如没有，从LRU中申请一个16KB的页，分成一个8KB，2个4KB的页放到对应unzip_LRU中。
在`information_schema.innodb_buffer_page_lru`中`where`条件`where compressed_size<>0`就可以查询unzip_LRU。

被修改的LRU页成为脏页(dirty page)，此时缓存的数据和硬盘的数据不一致，数据库通过checkpoint机制刷回硬盘，Flush列表就是脏页也列表。脏页会同时存在LRU和Flush中，二者不影响。`information_schema.innodb_buffer_page_lru`中`where`条件`where oldest_modification>0`可以查询脏页数据。在这个表里，如果字段`TABLE NAME`为`NULL`，表示为系统表空间。

注：这里有一个问题，Flush列表和LRU列表都有页数据，那么就会存在同步问题，如果Flush的页要时刻同步LRU页的数据？

InnoDB的内存池还包括了redo log buffer。重做日志会先缓存在这里，然后按照一定频率刷新到重做日志文件中。由于一般重做日志缓冲会每秒都刷新到日志文件中，所以只要保证每秒事务量在这个缓冲中即可。参数`innodb_log_buffer_size`控制了它的大小。重做日志缓冲会在：1)Master Thread线程每秒刷新重做日志缓冲到重做日志文件；2)事务提交时会重做日志缓冲刷新到重做日志文件；3)重做日志缓冲池剩余空间不足50%，刷新重做日志缓冲到重做日志文件。

InnoDB是通过一种叫做内存堆(heap)的方式管理的。对一些数据结构本身的内存进行分配时，需要从额外的内存池中申请，当这个额外的内存池不够，会从缓冲池中进行申请，例如，分配了缓冲池(innodb_buffer_pool)，但是缓冲池中的帧缓冲(frame buffer)还有对应的缓冲控制对象(buffer control block)，它们记录了LRU，锁，等待等信息，这些对象的内存需要从额外内存池申请。因此，如果设计了很大的InnoDB缓冲池，要相应增加这个值(注innodb_additional_mem_pool_size，这个已经在5.7被废弃了，参见[此处](http://mysql.taobao.org/monthly/2016/04/01/))。

### 总结
注意：修改`my.cnf`文件针对不同的内容需要加不同的preceding group。比如上面说的`innodb_buffer_pool_size`，需要加在`[mysqld]`下面才不会出错(不然，会报eror: Found option without preceding group in config file)。同时，官方文档说了，如果`innodb_buffer_pool_size`小于1G，不会划分多个缓冲池。只有大于1G，根据池大小会划分多个缓冲池。有多个缓冲池，才会在`show engine innodb status`中有每个池的信息。

`show engine innodb status`显示的是过去某个时间范围内InnoDB的状态。