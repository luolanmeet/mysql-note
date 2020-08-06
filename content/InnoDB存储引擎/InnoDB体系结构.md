### InnoDB体系结构



#### 后台线程

InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理

不同的任务。

* `Master Thread`

  是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，

  保证数据的一致性，包括脏页的刷新、合并插入缓冲（`INSERT BUFFER`）、

  `UNDO`页的回收等。

  > 脏页刷新在 `InnoDB 1.2.x`就不在`Master Thread`线程做了

* `IO Thread`

  在`InnoDB`存储引擎中大量使用了`AIO`来处理写`IO`请求。这些`IO Thread`的

  工作主要是负责这些`IO`请求的回调处理。使用`innodb_read_io_threads`和

  `innodb_write_io_threads`参数进行设置。`InnoDB 1.0.x`开始默认都是配置为4。

  > SHOW VARIABLES LIKE "innodb_%io_threads";

* `Purge Thread`

  事务被提交后，其所使用的`undo log`可能不再需要，因此需要`Purge Thread`来

  回收已经使用并分配的`undo`页。可以通过`innodb_purge_threads`设置。

  > SHOW VARIABLES LIKE "innodb_purge_threads";

* `Purge Cleaner Thread`

  `InnoDB 1.2.x`引入的，作用是将之前版本中脏页刷新操作都放到独立的线程中完成。

  

#### 内存

##### 缓冲池

`InnoDB`存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。

因此可视为基于磁盘的数据库系统（`Disk-base Database`）。

基于磁盘的数据库系统通常使用缓冲池技术来提升数据库的整体性能。



缓冲池简单来说就是一块内存区域，在数据库中进行读取页的操作，首先将从磁盘

读取的页放在缓冲池中，这个过程称为将页`FIX`在缓冲池中。下一次读取相同的页时，

首先判断该页是否在缓冲池中。如在则读取该页，否则读取磁盘上的页。



对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新

到磁盘上。需要注意的时，页从缓冲池刷新到磁盘的操作并不是在每次页发生更新时

触发，而是通过一种称为`Checkpoint`的机制刷新会磁盘。

> 没读到内存的也，是直接在磁盘上修改吧？

对于`InnoDB`存储引擎而言，缓冲池的配置通过参数`innodb_buffer_pool_size`来设置。

> SHOW VARIABLES LIKE "innodb_buffer_pool_size";

从`InnoDB 1.0.x`版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到

不同的缓冲池实例中。这样可以减少数据库内部的资源竞争，增加数据库的并发处理能力。

可以通过`innodb_buffer_pool_instances`来进行配置。默认值是1。

> SHOW VARIABLES LIKE "innodb_buffer_pool_instances";

##### LRU List、Free List 和 Flush List

通常来说数据库中的缓冲池通过`LRU（Latest Recent Used）`最近最少使用算法进行管理的。

最频繁使用的也在`LRU`列表的前端，最少使用的页在`LRU`列表的尾端。当缓冲池不能存放

新读取到的页时，将首先释放`LRU`列表中尾端的页。

<span style="border-bottom:2px dashed yellow;">在`InnoDB`存储引擎中，缓冲池中页的大小默认为16KB，同样使用`LRU`算法进行管理。</span>

不同的是`InnoDB`存储引擎对传统的`LRU`算法进行优化。`LRU`列表中加入了`midpoint`位置。

新读取的页不直接放入`LRU`的首部，而是放入到`midpoint`的位置。在等待一段时间后才

加入到`LRU`的首部。`midpoint`之前的列表称为`new`列表，之后的称为`old`列表。

> 可以防止某些`SQL`将缓冲池中的页刷出，如全表扫描的`SQL`。
>
> 参数`innodb_old_pct`用于控制`midpoint`的位置，默认是37，
>
> 表示新读取的页插入到`LRU`列表尾端37%的位置。

> 从`old`加入到`new`的过程是？

`LRU`列表中的页被修改后，称该页为脏页（`dirty page`），即缓冲池中的页和磁盘

上的页的数据产生不一致。这是数据库会通过`Checkpoint`机制将脏页刷新回磁盘。

而`Flush`列表中的页即为脏页列表。脏页既存在于`LRU`列表，也存在与`Flush`列表中。