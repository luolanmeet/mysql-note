### InnoDB存储引擎文件

#### 表空间文件

`InnoDB`采用将存储的数据按表空间（`tablespace`）进行存放的设计。

默认配置下会有一个初始大小为`10MB`，名为`ibdata1`的文件。

该文件就是默认的表空间文件（`tablespace file`）。

可以通过多个文件组成一个表空间，同时制定文件的属性如

```sql
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata1:2000M:autoextend
```

若两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。

> 数据分开存，怎么查？

若设置了参数`innodb_file_per_table`，则用户可以将每个基于`InnoDB`存储引擎的表产生

一个独立表空间，名为：表名.frm。

需要注意的是，这些单独的表空间文件仅存储该表的数据、索引和插入缓冲`BITMAP`等，

其余信息还是存放在默认的表空间中

> 什么信息？



#### 重做日志文件

在默认情况下，在`InnoDB`存储引擎的数据目录下会有两个名为`ib_logfile0`和

`ib_logfile1`的文件。这些文件即为重做日志文件（`redo log file`）。

参数`innodb_log_files_in_group`指定了日志文件组中重做日志文件的数量，默认为2。

参数`innodb_mirrored_log_groups`指定了日志镜像文件组的数量，默认为1，

表示只有一个日志文件组，没有镜像。