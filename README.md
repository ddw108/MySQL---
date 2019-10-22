# MySQL-学习
MySQL学习笔记

## 1、事务隔离

#### 隔离级别

1. 读未提交：一个事务还未提交时，做的变更就可被别的事务看到
2. 读提交：一个事务提交后，才可以被其他事务看到
3. 可重复读：一个事务执行过程中看到的数据，总跟这个事务在启动时看到的是一致的
4. 串行化：对于同一行记录进行加锁，访问串行化

#### Innodb事务的视图理解

在实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准

配置方法：启动参数transaction-isolation

1. 串行化：直接加锁
2. 可重复读：事务启动时创建，整个事务存在期间都用这个视图
3. 读已提交：在sql执行时创建
4. 读未提交：无视图概念

#### 事务的实现方式

可重复读：

事实上，在事务中，每条记录在更新时会同时记录一条回滚操作；通过回滚操作，每条记录可以得到上一个状态的值。

同一条记录在系统中可以存在有多个版本，即数据库的多版本并发控制。

回滚日志会在没有事务需要用到的时候被删除，所以不建议使用长事务，会一直存在，导致回滚日志过多。

#### 避免长事务

客户端：

1. 在启动连接成功后，很多连接框架会默认先执行一个set autocommit=0的命令，这样就会把自动提交命令关掉，导致接下来的查询都在事务中，直到执行commit或rollback或断开连接。如果是长连接，就会导致意外的长事务。建议使用set autocommit=1的显示命令来启动事务，只有在需要时才启动事务。
2. 确认是否有只读事务，有的话尽量避免使用事务。
3. 通过set max_execution_time来控制每个语句的最大执行时间。

服务端：

监控长事务阈值，超过则报警或者kill。

## 2、索引

#### 主键索引与非主键索引的区别

1. 主键索引是表的根本B+树，主键索引的叶子节点存的是整行的数据，在innodb中，主键索引也被成为聚簇索引。
2. 非主键索引也对应一个B+树，叶子节点存的是主键的值，在innodb中也被称为二级索引。

eg：

1. select * from T where ID=500，即主键查询方式，只需要搜索主键索引ID这棵B+树。
2. select * from T where k=5，即非主键查询，需要先搜索K这棵B+树，得到ID值，再到ID这棵B+树搜索一次，这个过程又叫做回表。

#### 自增主键的使用

1. 主键索引的值应该尽量小，用**自增索引**比较合适。只有当只有一个索引且是唯一索引时，即k-v场景时，用业务字段做主键索引才比较合适。
2. 自增主键是插入式的数据模式，可以避免数据结构的调整，简化维护的过程

#### 联合索引的技巧

eg：select * from T where k between 3 and 5;

这条语句会先查一个k的索引树，拿到k=3对应的ID，再查ID的索引树拿结果；

...

直到查到k=5；

期间需要多次回表，我们可以通过设计联合索引来避免多次回表的过程。

1. 覆盖索引：如果直接是select ID T where k between 3 and 5;这时由于ID本来就在k的索引树上，所以此时的查询不需要回表；
2. 联合索引：可以将需要的联合查询字段建立所以，并且满足最左前缀原则。即考虑到业务要求，将查找频率较高的字段进行靠左建立索引；
3. 索引下推：like 'hello%' and age>10;MySQL5.6以前需要每条like 'hello%'的记录都回表去判断age>10；而MySQL5.6以后，会在查找前先进行一次age>10的过滤，再去查找like 'hello%'，可以减少回表率，提升检索的速度。

