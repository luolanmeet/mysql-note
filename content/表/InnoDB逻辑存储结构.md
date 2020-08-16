### InnoDB逻辑存储结构

`InnoDB`存储引擎所有数据都被逻辑地存放在一个空间中，称之为表空间（`table space`）。

表空间又由段（`segment`）、区（`extent`）、页（`page`）组成。



#### 表空间

表空间可以看做是`InnoDB`存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。



#### 段

表空间是由各种段组成的，常见的段有

* 数据段（B+树的叶子节点）
* 索引段（B+树的非叶子节点）
* 回滚段



#### 区

区是由连续页组成的空间，在任何情况下每个区的大小都为`1MB`。为了保证区中

页的连续性，`InnoDB`存储引擎一次从磁盘申请4~5个区。在默认情况下，`InnoDB`

存储引擎页的大小为`16KB`，即一个区中一共有64个连续的页。

> 每个段开始时，先用32个页大小的碎片页（`fragment page`）来存放数据，
>
> 在使用完这些页之后才是64个连续页的申请。



#### 页

`InnoDB`中页也被称为块。页是`InnoDB`磁盘管理的最小单位。在`InnoDB`存储引擎中

默认每个页的大小为`16KB`，从`InnoDB 1.2.x`开始，可以通过`innodb_page_size`将

页的大小设置为`4KB`、`8KB`、`16KB`。若设置完成，则不可再次修改，除非通过

`mysqldump`导入和导出操作来产生新的库。

常见的页类型有

* 数据页（`B-tree Node`）
* `undo`页（`undo Log Page`）
* 系统页（`System Page`）
* 事务数据页（`Transaction system Page`）
* 插入缓冲位图页（`Insert Buffer Bitmap`）
* 插入缓冲空闲列表页（`Insert Buffer Free List`）
* 未压缩的二进制大对象页（`Uncompressed BLOB Page`）
* 压缩的二进制大对象页（`compressed BLOB Page`）



#### 行

`InnoDB`存储引擎是面向列的（`row-oriented`），也就是数据是按行进行存放的。

每个页存放的行记录也是有限制的，最多存放16KB / 2B - 200行的记录，即7992行记录。