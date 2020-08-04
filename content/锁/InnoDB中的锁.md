### InnoDB中的锁

#### 锁类型

InnoDB存储引擎中实现了如下两种标准的<span style="border-bottom:2px dashed yellow;">行级锁</span>：

* 共享锁（`S Lock`），允许事务读一行数据
* 排他锁（`X Lock`），允许事务删除或更新一行数据。

共享锁和排他锁的兼容性

|      | X      | S      |
| ---- | ------ | ------ |
| X    | 不兼容 | 不兼容 |
| S    | 不兼容 | 兼容   |

如果一个事务T1已经获得了行r的共享锁，那么另外的事务T2可以立即获得r的共享锁，

因为读取并没有改变行r的数据，这种情况为锁兼容（`Lock Compatible`）。

如果事务T3想获得行r的排他锁，则其必须等待事务T1、T2释放行r上的共享锁，

这种情况为锁不兼容。

##### 意向锁

`InnoDB`存储引擎支持多粒度（`granular`）锁定，这种锁允许事务在行级上的锁和表级

上的锁同时存在。为了支持在不同粒度上进行加锁操作，`InnoDB`存储引擎支持一种额外的

锁方式，称之为意向锁（`Intention Lock`）

意向锁是将锁定的对象分为多个层次。

`InnoDB`存储引擎支持两种意向锁：

* 意向共享锁（`IS Lock`），事务想获得一张表中某几行的共享锁
* 意向排他锁（`IX Lock`），事务想获得一张表中某几行的排他锁

> `InnoDB`存储引擎的意向锁即为表级别的锁，其设计目的主要是为了在
>
> 一个事务中揭示下一行将被请求的锁类型。

> 若将上锁对象看出一个棵树，那么对最下层的对象（最细粒度对象）上锁（`X`、`S`锁），
>
> 需要先对上层粗粒度的对象上锁（`IX`、`IS`意向锁）。按照数据库->表->页->记录。
>
> 若其中任何一部分导致等待，那么需要等待粗粒度锁的完成。

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | 兼容   | 兼容   | 兼容   | 不兼容 |
| IX   | 兼容   | 兼容   | 不兼容 | 不兼容 |
| S    | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |



#### 一致性非锁定读

一致性非锁定读（`consistent nonlocking read`）指`InnoDB`存储引擎通过多版本控制（`MVCC`）

的方式读取当前执行时间数据库中行的数据。

如果读取的行正在执行`DELETE`或`UPDATE`操作，这时读取操作不会因此去等待行上的锁的释放。

相反地，`InnoDB`存储引擎会去读取行的一个快照数据。

> 之所以称为非锁定读，因为不需要等待访问的行上`X`锁的释放。
>
> 快照数据是指该行的之前版本的数据，该实现是通过`undo`段来完成的。而`undo`用来
>
> 在事务中回滚数据，因此快照数据本身是没有额外的开销。

<span style="border-bottom:2px dashed yellow;">非锁定读机制极大地提高了数据库的并发性。在`InnoDB`存储引擎的默认设置下，</span>

<span style="border-bottom:2px dashed yellow;">这是默认的读取方式，即读取不会占用和等待表上的锁。</span>

<span style="border-bottom:2px dashed yellow;">但是在不同的事务隔离级别下，读取的方式不同。</span>

> 在事务隔离级别`READ COMMITTED`和`REPEATABLE READ`（默认），`InnoDB`存储引擎
>
> 使用非锁定的一致性读。（mvcc）
>
> 但在`READ COMMITTED`事务隔离级别下，总是读取行的最新版本，如果行被锁定了，
>
> 读取最新一份快照数据。
>
> 在`REPEATABLE READ`事务隔离级别下，读取的是事务开始时的行数据版本。



##### 一致性锁定读

某些时候需要显示地对数据库读取操作进行加锁以保证数据逻辑的一致性。

`InnoDB`存储引擎对于`SELECT`支持两种一致性的锁定读（`locking read`）

* SELECT ... FOR UPDATE：加`X`锁
* SELECT ... LOCK IN SHARE MODE：加`S`锁

> 无论是哪种，都需要在一个事务中，当事务提交了，锁也就释放了。
>
> 因此在使用时需要加上`BEGIN`、`START TRANSACTION`或者是`SET AUTOCOMMIT=0`



##### 自增长与锁

在`InnoDB`存储引擎的内存结构中，对每个含有自增长值的表都会一个自增长计数器

（`auto-increment counter`）。当对含有自增长的计数器的表进行插入操作时，这个

计数器会初始化，并执行如下语句获取计数器的值

```SQL
SELECT MAX(auto_inc_col) FROM t FRO UPDATE
```

插入操作会依据这个自增长的计数器的值加1赋予自增长列。这种方式称为

`AUTO-INC Locking`。这种锁采用一种特殊的<span style="border-bottom:2px dashed yellow;">表锁机制</span>，为了提高插入的性能，

<span style="border-bottom:2px dashed yellow;">锁不在一个事务完成后才释放，而是在完成对自增长值插入的`SQL`语句后立即释放。</span>

虽然`AUTO-INC-Locking`从一定程度上提高了并发插入的效率，但还是有性能还是较差。

`MySQL5.1.22`版本开始，`InnoDB`存储引擎提供了一种轻量级互斥量的自增长实现机制，

并提供了一个`innodb_autoinc_lock_mode`来控制自增长的模式，默认值是1。



`innodb_autoinc_lock_mode`值说明

| innodb_autoinc_lock_mode | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| 0                        | `MySQL5.1.22`版本之前自增长的实现模式，通过表锁`AUTO-INC Locking`方式 |
| 1                        | 默认值。对于`simple inserts`，使用互斥量（`mutex`）去对内存中的计数器<br />进行累加操作。对于`bulk inserts`还是使用传统表锁。 |
| 2                        | 对于所有`insert-like`自增长值的产生都通过互斥量，而不是`AUTO-INC Locking`方式<br />但是有一定问题，并发插入时，自增长id可能不是连续的。<br />基于`Statement-Base Replication`会出现问题。因此使用这个模式<br />任何时候都应该使用`Row-Base Replication`。这样才能保证主从数据一致。 |

