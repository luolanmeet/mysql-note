### MySQL体系结构和存储引擎

#### MySQL的配置

当启动实例时，`MySQL`数据库会去读取配置文件，根据配置文件的参数来启动

数据库实例，如果没有配置文件则按编译时的默认参数设置启动实例。

> `Oracle`如果没有配置文件会提示错误，启动失败

配置的读取路径可用  `mysql --help | grep my.cnf`查看，`MySQL`按照顺序读取

配置文件，以读取到的最后一个配置中的参数为准

```html
/ect/my.cnf -> /ect/mysql/my.cnf -> /usr/local/mysql/etc/my.cnf -> ~/.my.cnf
```



