#### Master Thread 工作方式

#### InnoDB 1.0.x 之前的Master Thread

`Master Thread`具有最高的线程优先级。其内部由多个循环（`loop`）组成：

* 主循环（`loop`）
* 后台循环（`backgroup loop`）
* 刷新循环（`flush loop`）
* 暂停循环（`suspend loop`）

#### Loop

`Loop`被称为主循环，因为大多数的操作是在这个循环中，其中有两大部分的操作

* 每秒钟的操作
* 每10秒钟的操作

```c++
void master_thread() {
    loop:
    for (int i = 0; i < 10; i++) {
        do thing once per second
        sleep 1 second if necessary
    }
    do thing once per ten second
    goto loop;
}
```



##### 每秒一次的操作

包含

* 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）

  > 即使事务还没提交，`InnoDB`存储引擎仍然每秒会将重做日志缓冲中的内容
  >
  > 刷新到重做日志文件。

* 合并插入缓冲（可能）

  > `InnoDB`存储引擎会判断当前一秒内发生的IO次数，如果小于5，
  >
  > 会认为当前IO压力小，可以执行合并插入缓冲的操作。

* 至多刷新100个`InnoDB`的缓冲池的脏页（可能）

  > `InnoDB`存储引擎通过判断当前缓冲池中脏页的比例是否超过了
  >
  > 配置文件中的`innodb_max_dirty_pages_pct`这个参数，如果超过
  >
  > 则认为需要做磁盘同步动作，将100个脏页写入磁盘。

* 如果当前没有用户活动，则切换到backgroud loop（可能）



##### 每10秒的操作

包含

* 刷新100个脏页到磁盘（可能）

  > `InnoDB`引擎会判断过去10秒内磁盘的IO次数是否小于了200次，
  >
  > 如果则认为有足够的磁盘IO能力，因此将100个脏页刷新到磁盘。

* 合并至多5个插入缓冲（总是）

* 将日志缓冲刷新到磁盘（总是）

* 删除无用的`Undo`页（总是）

* 刷新100个或10个脏页到磁盘（总是）

  > 判断缓冲池中脏页的比例，如果超过70%的脏页，则刷新100个脏页到磁盘，
  >
  > 否则刷新10个的脏页到磁盘



#### backgroup loop

若当前没有用户活动（数据库空闲时）或者数据库关闭，就会切换到这个循环。

会执行一下操作

* 删除无用的`Undo`页（总是）
* 合并20个插入缓冲（总是）
* 跳回主循环（总是）
* 不断刷新100个页直到符合条件（可能，跳转到`flush loop`中完成）

若`flush loop`中也没有什么事情可做，则会切换到`suspend loop`。

将`master thread`挂起。



#### InnoDB 1.2.x 之前的Master Thread

`InnoDB 1.0.x`之前的版本对`IO`做了一定的硬编码。

从`InnoDB 1.0.x`开始提供了参数`innodb_io_capacity`，用来表示磁盘的

`IO`的吞吐量，默认值是200。对于刷新到磁盘页的数量，会按照此参数的

百分比进行控制。规则如下

* 在合并插入缓冲时，合并插入缓冲的数量为`innodb_io_capacity`值的5%
* 在从缓冲区刷新脏页时，刷新脏页的数量为`innodb_io_capacity`。

> 若用户使用了`SSD`类的磁盘，或者将几块磁盘做成了`RAID`，当存储设备
>
> 拥有更高的`IO`速度时，完全可以将`innodb_io_capacity`值调得再高些。

`InnoDB 1.0.x`带来的另一个参数是`innodb_adaptive_flushing`（自适应刷新），

该值影响每秒刷新脏页的数量。之前脏页比例超过`innodb_max_dirty_pages_pct`

才会触发刷新，现在小于也会刷新一定量的脏页。

还有一个参数是`innodb_purge_batch_size`，可以控制每次`full purge`回收的

undo页的数量。默认值是20。



#### InnoDB 1.2.x 的Master Thread

刷新脏页的操作被分离到 `Page Cleaner Thread`。