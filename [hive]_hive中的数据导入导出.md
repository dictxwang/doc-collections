# hive中的数据导入导出

*主要记录hive进行数据导入导出的常用操作。*

### 零、参数设置

#### 1、设置参数   

```shell
> set hive.enforce.bucketing=true;
```

#### 2、查看所有参数

```shell
> set;
```

#### 3、查看指定参数

```shell
> set hive.enforce.bucketing;
```

### 一、导入数据

#### 1、从本地系统导入数据

##### a、在hive中创建表

```shell
> use default;
> create table if not exists wq_load_data_local(name string, age int) row format delimited fields terminated by ' ' lines terminated by '\n';
```

##### b、查看新建的表（两种方式均可）

```shell
> desc wq_load_data_local;
> show create table wq_load_data_local;
```

##### c、准备导入数据

​    依据创建表时的字段分隔和行分隔
​    一行一条记录，不同字段之间用空格分隔
​    假设将数据文件保存至 /data/temp/hive_load_data.txt

##### d、执行导入

    ```shell
    > load data local inpath '/data/temp/hive_load_data.txt' into table wq_load_data_local;
    > load data local inpath '/data/temp/hive_load_data.txt' overwrite into table wq_load_data_local;  # 加上overwrite相当于先删除表中原有数据
    ```

##### e、检查导入结果

   ```shell
   > select * from wq_load_data_local limit 10;  # 直接查询hive表
   > ./hadoop dfs -ls /hive/warehouse/wq_load_data_local  # 通过hadoop命令查询
   ```

#### 2、从HDFS中导入数据

##### a、在hive中创建表

  ```shell
  > use default;
  > create table if not exists wq_load_data_hdfs(name string, age int) row format delimited fields terminated by ' ' lines terminated by '\n';
  ```

##### b、准备导入数据

​    依据创建表时的字段分隔和行分隔规则将记录录入到文件
​    并将文件推送至hdfs路径下

  ```shell
  > hadoop dfs -put /data/temp/hive_load_data2.txt /wangqiang/
  ```

##### c、执行导入

```shell
> load data inpath '/wangqiang/hive_load_data2.txt' into table wq_load_data_hdfs;
```

##### d、检查导入结果

  ```shell
  > select * from wq_load_data_hdfs limit 10;  # 直接查询hive表
  > ./hadoop dfs -ls /hive/warehouse/wq_load_data_hdfs  # 通过hadoop命令查询
  > ./hadoop dfs -ls /wangqiang/hive_load_data2.txt  # 导入前hdfs上的文件将自动删除
  ```

#### 3、从其他hive表导入数据

##### a、在hive中创建表

  ```shell
  > use default;
  > create table if not exists wq_load_data_local2(name string, age int) row format delimited fields terminated by ' ' lines terminated by '\n';
  ```

##### b、执行导入

 ```shell
 > insert into table wq_load_data_local2 select * from wq_load_data_local;
 ```

##### c、检查导入结果

 ```shell
 > select * from wq_load_data_local2 limit 10;
 ```

#### 4、导入到分区表

​    分区表通常是指定一个字段或多个字段进行分区，导入时的分区实现分为静态分区和动态分区两种方式。

##### a、在hive中创建分区表

 ```shell
 > use default;
 > create table if not exists wq_load_data_partition(name string) partitioned by(age int) row format delimited fields terminated by ' ' lines terminated by '\n';
 ```

##### b、从本地导入数据到分区表（需要手动指定分区）

 ```shell
 > load data local inpath '/data/temp/hive_load_data.txt' into table wq_load_data_partition patition(age=60);  # 这种方式会忽略原文件中age的值，统一用60覆盖
 ```

##### c、从hdfs导入数据到分区表（需要手动指定分区）

 ```shell
 > load data inpath '/wangqiang/hive_load_data2.txt' into table wq_load_data_partition partition(age=60);
 ```

##### d1、从其他hive表导入数据到分区表（手动指定分区）

 ```shell
 > insert into table wq_load_data_partition partition(age=60) select name from wq_load_data_local;  # 手动指定分区
 ```

##### d2、从其他hive表导入数据到分区表（动态指定分区）

 ```shell
 > set hive.exec.dynamic.partition=true;
 > set hive.exec.dynamic.partition.mode=nonstrict;  # 首先通过这两部设置开启动态分区
 > insert into table wq_load_data_local partition(age) select name, age from wq_load_data_local;  # 这里需要通过partition(age)指定分区字段，同时确保和select语句的字段顺序能对上
 > insert overwrite table wq_load_data_local partition(age) select name, age from wq_load_data_local;  # 使用overwrite代替into，实现对原数据的替换
 ```

##### e、查看hadoop上文件分布

 ```shell
 > ./hadoop dfs -ls /hive/warehouse/wq_load_data_partition
     /hive/warehouse/wq_load_data_partition/age=60
     /hive/warehouse/wq_load_data_partition/age=61  # 文件分两个路径存放
 ```

##### f、查看分区

 ```shell
 > show partitions wq_load_data_partition;
      OK
      age=60
      age=61
 ```

#### 5、导入到分桶表

##### a、设置开启分桶表支持

 ```shell
 > use default;
 > set hive.enforce.bucketing=true; 开启对分桶表的支持
 > set mapreduce.job.reduces=4; 设置与桶相同的reduce个数（默认只有一个reduce）
 ```

##### b、创建分桶表

 ```shell
 > create table wq_load_data_bucket(name string, age int) clustered by(age) into 4 buckets row format delimited fields terminated by ' ' lines terminated by '\n';
 ```

##### c、创建普通表

 ```shell
 > create table if not exists wq_load_data_bucket_n(name string, age int) row format delimited fields terminated by ' ' lines terminated by '\n';
 ```

##### d、从普通表导入到分桶表

 ```shell
 > insert into wq_load_data_bucket select * from wq_load_data_bucket_n;
 ```

##### e、查看hadoop上文件分布

```shell
 > ./hadoop dfs -ls /hive/warehouse/wq_load_data_bucket/
     /hive/warehouse/wq_load_data_bucket/000000_0.deflate
     /hive/warehouse/wq_load_data_bucket/000001_0.deflate
     /hive/warehouse/wq_load_data_bucket/000002_0.deflate
     /hive/warehouse/wq_load_data_bucket/000003_0.deflate  # 数据分多个文件保存
```

##### f、分桶抽样查询

​    tablesample语法解释： tablesample(bucket x out of y ) 

​        y是bucket总数(n)的因子或倍数，决定抽样比例，y/n或者 n/y，具体是y是否大于n

​        x表示从第几个bucket开始取数据，x, x+y, x+2y ...

​        x必须小于y

```shell
> select * from wq_load_data_bucket tablesample(bucket 1 out of 4 on age);
```

#### 6、创建表的同时导入数据

 ```shell
 > create table wq_load_data_local3 as select * from wq_load_data_local;
 ```

#### 7、同时向多个表导入数据

 ```shell
 > from wq_load_data_local insert overwrite table wq_load_data_partition partition(age) select name, age insert overwrite table wq_load_data_local3 select *;
 ```

### 二、导出数据

#### 1、导出到本地文件系统

 ```shell
 > insert overwrite local directory '/data/temp/wangqiang' row format delimited fields terminated by ' ' select * from wq_load_data_local;  # 注意本地路径原文件会清空
 ```

#### 2、导出到HDFS

 ```shell
 > insert overwrite directory '/wangqiang2' row format delimited fields terminated by ' ' lines terminated by '\n' select * from wq_load_data_local;
 ```

#### 3、其他方式导出

 ```shell
 > ./hive -e "select * from wq_load_data_local" >> /data/temp/wangqiang/1.txt
 > ./hive -e "select * from wq_load_data_bucket tablesample(bucket 1 out of 4 on age)" >> /data/temp/wangqiang/1.txt
 > ./hive -f "/data/temp/wangqiang/export.hql" >> /data/temp/wangqiang/1.txt  # -f 指定hql文件
 ```

### 三、删除数据

#### 1、按条件删除数据

 ```shell
 > insert overwrite table wq_load_data_local3 select * from wq_load_data_local3 where age = 61;  # 删除age=61以外的数据
 ```

#### 2、删除分区

 ```shell
 > alter table wq_load_data_partition drop partition (age=61);
 ```

#### 3、清空表

 ```shell
 > truncate table wq_load_data_local3;
 ```

#### 4、删除表

 ```shell
 > drop table if exists wq_load_data_local3;
 ```

#### 5、删除库

```shell
> drop database if exists wangqiang01;
> drop database if exists wangqiang01 cascade;
```