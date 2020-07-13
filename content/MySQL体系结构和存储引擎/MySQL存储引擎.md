### MySQL存储引擎

> 存储引擎是基于表的，而不是数据库。

`MySQL`主要的存储引擎

* `InnoDB`存储引擎
* `MyISAM`存储引擎
* `NDB`存储引擎
* `Memory`存储引擎
* `Archive`存储引擎
* `Federated`存储引擎
* `Maria`存储引擎

> 可以通过 `SHOW ENGINES\G` 语句查看当前数据库支持的所有存储引擎。

#### 几个概念

* `OLAP`数据库、`OLTP`数据库

  > 数据处理大致可以分成两大类：
  >
  > * 在线事务处理`OLTP`（`Online Transaction Processing`）
  >
  > * 在线分析处理`OLAP`（`Online Analytical Processing`）
  >
  > `OLTP`是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。
  >
  > `OLAP`是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。

* `ETL`

  > `ETL`是英文`Extract-Transform-Load`的缩写，用来描述将数据从来源端经过
  >
  > 抽取（`extract`）、转换（`transform`）、加载（`load`）至目的端的过程。



#### InnoDB存储引擎

`InnoDB`存储引擎支持事务、行锁、外键，并支持类似于`Oracle`的非锁定读，

即默认读取操作不会产生锁。从`MySQL`数据库5.5.8版本开始，`InnoDB`存储引擎

是默认的存储引擎。<span style="border-bottom:2px dashed yellow;">其设计目标主要面向在线事务处理（`OLTP`）的应用。</span>

`InnoDB`通过多版本并发控制（`MVCC`）来获得高并发性，并且实现了`SQL`标准的

4种隔离级别，默认为`REPEATABLE`级别。同时使用一种被称为`next-key locking`的

策略来避免幻读（`phantom`）现象的产生。`InnoDB`存储引擎还提供插入缓冲（`insert buffer`）、

二次写（`double write`）、自适应哈希索引（`adaptive hash index`）、预读（`read ahead`）

等高性能和高可用的功能。

对于表中数据的存储，`InnoDB`存储引擎采用聚集（`clustered`）的方式，因此每张表的

存储都是按主键的顺序进行存放。如果没有显示地在表定义时指定主键，`InnoDB`存储引擎

会为每一行生成一个6字节的`ROWID`，并以此作为主键。



#### MyISAM存储引擎

`MyISAM`存储引擎不支持事务，表锁设计，支持全文索引，<span style="border-bottom:2px dashed yellow;">主要面向一些`OLAP`数据库应用。</span>

`MyISAM`存储引擎表由`MYD`和`MYI`组成，`MYD`用来存放数据文件，`MYI`用来存放索引文件。

<span style="border-bottom:2px dashed yellow;">`MyISAM`存储引擎只缓存（`cache`）索引文件，而不缓存数据文件。</span>



#### NDB存储引擎

`NDB`存储引擎是一个集群存储引擎，结果是`share nothing`的集群架构，因此能提供更高的

可用性。`NDB`的特点是全部数据放在内存中（`MySQL5.1`开始可以将非索引数据放在磁盘上），

因此主键查找速度极快，并且可以通过添加`NDB`数据存储节点线性提高数据库性能，是高可用、

高性能的集群系统。

NDB存储引擎的连接操作（`JOIN`）是在`MySQL`数据库层完成的，而不是在存储引擎层完成的。

这意味着复杂的连接操作需要巨大的网络开销，因此查询速度很慢。



#### Memory存储引擎

`Memory`存储引擎将表中的数据存放在内存中，如果数据库重启或发生崩溃，表中的数据都将消失。

非常适合用于存储临时数据的临时表。`Memory`存储引擎默认使用哈希索引。

`Memory`存储引擎速度非常快，但在使用上有一些限制。比如只支持表锁，并发性能较差，并且

不支持`TEXT`和`BLOB`列类型。存储变长字段（`varchar`）时是按照定长字段（`char`）的方式进行的，

因此会浪费内存。

> `MySQL`数据库使用`Memory`存储引擎作为临时表来存放查询的中间结果集（`intermediate result`）。
>
> 如果中间结果集大于`Memory`存储引擎表的容量设置，或者中间结果含有`TEXT`或`BLOB`列类型字段，
>
> 则`MySQL`会把其转换到`MyISAM`存储引擎表而存放到磁盘中，`MyISAM`不缓存数据文件，因此这时
>
> 产生的临时表的性能对于查询会有损失。



#### Maria存储引擎

`Maria`存储引擎是新开发的引擎，目标是取代原有的`MyISAM`存储引擎，可以看做是`MyISAM`的后续版本。

特点是：支持缓存数据和索引文件，行锁设计，提供`MVCC`功能，支持事务和非事务安全的选项，以及更好的

`BLOB`字符类型的处理性能。

