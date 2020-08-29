### Cardinality值

如何查看索引是否具有高选择性？

可以通过`SHOW INDEX`结果中的列`Cardinality`来观察。

`Cardinality`表示索引中不重复记录数量的预估值。注意，是一个预估值。



#### InnoDB存储引擎的Cardinality统计

`Cardinality`的统计是放在存储引擎层进行的。

数据库对于`Cardinality`的统计是通过采样（`Sample`）的方法来完成的。

采样过程如下

* 取得B+树索引中叶子节点的数量，即为A
* 随机取得B+树索引中的8个叶子节点。统计每个页不同记录的个数
* `Cardinality` = 不同记录个数和 * A / 8

`InnoDB`存储引擎内部对`Cardinality`信息的更新策略为

* 表中 1/16 的数据已发生过变化
* stat_modified_counter > 2000000000

手动触发`Cardinality`更新

* `ANALYZE TABLE t_xxx`
* `SHOW TABLE STATUS`
* `SHOW INDEX FROM t_xxx`
* 访问`INFOMATION_SCHEMA`下的`TABLES`和`STATISTICS`

> 如果表的数据量很大，并且存在多个辅助索引时，上述操作的执行会很慢。