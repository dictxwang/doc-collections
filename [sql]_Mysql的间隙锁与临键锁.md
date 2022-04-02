# Mysql的间隙锁与临键锁

*mysql的间隙锁（gap locks）和临键锁（next-key locks）是在Repeatable Read事务隔离级别下产生的，目的是防止出现数据幻读（Phantom Read）。*
*临建锁实际上是记录锁（record lock）加间隙锁（gap locks）。*

### 一、创建表

```mysql
| wangqiang_test6 | CREATE TABLE `wangqiang_test6` (
  `tid` int(11) NOT NULL AUTO_INCREMENT,
  `cid` int(4) DEFAULT NULL,
  `fname` varchar(16) DEFAULT NULL,
  `sname` varchar(16) DEFAULT NULL,
  PRIMARY KEY (`tid`),
  UNIQUE KEY `cid_idx_u` (`cid`),
  KEY `fname` (`fname`)
) ENGINE=InnoDB AUTO_INCREMENT=32 DEFAULT CHARSET=utf8 |
```

其中包含了tid主键索引，cid构成的唯一二级索引，以及fname构成的普通二级索引。

在Repeatable Read隔离级别，主键索引和唯一二级索引实际上是相同的处理逻辑。

### 二、准备初始数据

```sql
+-----+------+-------+--------+
| tid | cid  | fname | sname  |
+-----+------+-------+--------+
|   5 | 1005 | wang  | 123xyz |
|  10 | 1010 | zhao  | 123xyz |
|  12 | 1012 | qian  | 123xyz |
|  20 | 1020 | li    | 123xyz |
+-----+------+-------+--------+
```

如此，针对主键tid，产生了5个间隙： (min_int, 5]、(5, 10]、(10, 12]、(12, 20]、(20, max_int]

### 三、演示只产生记录锁（Record Lock）

分别开启两个mysql链接会话（用两个客户端分别登录数据库即可）：S1和S2

```shell
step1: 在S1执行 mysql> begin;  # 开启S1的事务（也可以使用 start transaction; ）
step2: 在S1执行 mysql> select * from wangqiang_test6 where tid = 5 for update;
step3: 在S2执行 mysql> update wangqiang_test6 set sname = "xxx" where tid = 5;  # 此时会等待锁而阻塞
step4: 终止step3，重新在S2执行 mysql> update wangqiang_test6 set sname = "xxx" where tid = 10;  # 此时修改成功
step5: 在S2执行 mysql> insert wangqiang_test6(tid, cid, fname, sname) values (6, 1011, "zhou", "xxx");  # 数据插入成功
step6: 在S1执行 mysql> rollback;  # 借用回滚模式，终止S1的事务
```

>>结论: 当采用唯一索引查询单条记录并进行加锁时，如果记录存在，则给这条结果加记录锁；另外的线程对这条记录的修改操作将被阻塞。

### 四、演示主键（唯一索引）产生间隙锁（Gap Locks）

重置数据，并分别开启两个mysql链接会话（用两个客户端分别登录数据库即可）：S1和S2

```shell
step1: 在S1执行 mysql> begin;  # 开启S1的事务
step2: 在S1执行 mysql> select * from wangqiang_test6 where tid = 7 for update;
step3: 在S2执行 insert wangqiang_test6(tid, cid, fname, sname) values (8, 1011, "zhou", "xxx");  # 此时会等待锁而阻塞
step4: 终止step3，重新在S2执行 insert wangqiang_test6(tid, cid, fname, sname) values (6, 1011, "zhou", "xxx");  # 此时同样会等待锁而阻塞
step5: 终止step4，重新在S2执行 mysql> insert wangqiang_test6(tid, cid, fname, sname) values (4, 1011, "zhou", "xxx");  # 数据插入成功
step6: 在S1执行 mysql> rollback;  # 借用回滚模式，终止S1的事务
```

>>结论：当采用唯一索引查询单条记录并进行加锁时，如果记录不存在，则会在这条记录的前后产生间隙锁。
>>如果是范围查找，同样会产生间隙锁；此时除了条件范围直接的记录外，还包括条件范围两端的间隙。
>>查看间隙锁是否打开：执行 mysql> show variables like "innodb_locks_unsafe_for_binlog";  # 如果Value是OFF说明已经打开间隙锁

### 五、演示普通二级索引产生的临键锁（next-key locks）

重置数据，并分别开启两个mysql链接会话（用两个客户端分别登录数据库即可）：S1和S2

```shell
step1: 在S1执行 mysql> begin;  # 开启S1的事务
step2: 在S1执行 mysql> select * from wangqiang_test6 where fname = "qian" for update;
step3: 在S2执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1030, "jiang", "000x");  # 数据插入成功
step4: 在S2执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1030, "meng", "000x");  # 阻塞
step5: 终止step4，重新在S2执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1031, "ren", "000x");  # 阻塞
step6: 终止step5，重新在S2执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1032, "xu", "000x");  # 数据插入成功
step7: 在S1执行 mysql> rollback;  # 借用回滚模式，终止S1的事务
```

>>结论：当采用普通二级索引查询单条记录并加锁时，及时该条数据存储，已让会产生间隙锁，连同自身的记录锁构成了临键锁。
>>如果查询条件指定的数据不存在，已让会在条件前后产生间隙锁，从而构成临键锁。

### 六、演示死锁的发生

重置数据，并分别开启两个mysql链接会话（用两个客户端分别登录数据库即可）：S1和S2

```shell
step1: 在S1执行 mysql> begin;  # 开启S1的事务
step2: 在S1执行 mysql> select * from wangqiang_test6 where cid = 1001 for update;  # 此时在tid上产生了间隙锁 (1001, 1005]
step3: 在S2执行 mysql> begin;  # 开启S2的事务
step4: 在S2执行 mysql> select * from wangqiang_test6 where cid = 1002 for update;  # 此时同样在tid上产生了间隙锁 (1002, 1005]
step5: 在S2执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1003, "meng", "xyz");  # 阻塞，因为S1同时获取了互斥锁
step6: 在S1执行 mysql> insert into wangqiang_test6 (cid, fname, sname) values (1004, "tan", "xyz");  # 发生死锁，自动终止S1，S2会成功提交
           死锁信息： ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

>>结论：当两个会话相互占有了对方需要的互斥锁时，就会发生死锁。
>>如果事务隔离级别是ReadCommited，在这种情况下不会发生死锁，但是其中一个会话会阻塞等待另一个会话提交或者回滚。

### 七、一些锁相关的操作

#### 1、information_schema库中几个锁相关的表

```mysql
mysql> select * from information_schema.innodb_locks;  # 查询当前锁状态（lock_mode、lock_type、lock_table等）
mysql> select * from information_schema.innodb_trx;  # 查询当前事务线程
mysql> select * from information_schema.innodb_lock_waits;  # 查看锁等待状况
```

#### 2、查看锁阻塞线程

```mysql
mysql> show processlist;  # 查看当前session进程
mysql> show engine innodb status;  # 此命令可以查看到死锁日志
mysql> show open tables where in_use > 0;  # 查看是否有锁表
mysql> show status like "%lock%";  # 查看锁表状况
```

#### 3、主动锁表与释放

```mysql
mysql> lock table t1 read;
mysql> unlock table t1;
```

#### 4、锁的关键字

```shell
S： 共享锁
X： 排他锁
IS： 意向共享锁
IX： 意向排他锁
```

#### 5、查看事务隔离级别

```mysql
mysql> select @@global.tx_isolation,@@tx_isolation;  # 分别查看系统级别和会话级别
```

#### 6、修改事务隔离级别

```mysql
SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}
mysql> set session transaction isolation level read committed;  # 举例：修改当前会话的隔离级别
```