### InnoDB行记录格式

`InnoDB`存储引擎和大多数数据库一样，记录是以行的形式存储的。

这意味着页总保存着表中一行行的数据。

`InnoDB`存储引擎提供了`Compact`和`Redundant`两种格式来存放行记录数据。

`Redundant`格式是为兼容之前版本而保留的，`Compact`是`MySQL 5.1`版本默认的行格式。

可以通过命令 `SHOW TABLE STATUS LIKE "table_name"` 来查看那当前表使用的行格式。