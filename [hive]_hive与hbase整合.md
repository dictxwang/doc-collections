# hive与hbase整合

### 零、内部表与外部表

1、创建表时未指定external关键字，则默认创建的是内部表（managed table）

2、内部表由Hive自身管理，外部表由hdfs管理

3、内部表数据存储位置是 hive.metastore.warehouse.dir指定的，默认是 /hive/warehouse

4、外部表数据存储位置由自己指定，如果未指定，将会在/hive/warehouse 新建一个同表名的路径，用于保存数据

5、删除内部表会同时删除元数据和存储数据，删除外部表仅删除元数据，hdfs上的文件不会动

6、对内部表的修改会直接同步给元数据，而对外部表的修改需要修复（msck repair table tbl_name）

7、hive与hbase关联时，hive中创建的表都是外部表（external table）。

### 一、从Hive到HBase

这种方式是指，初始状态HBase中没有数据表，通过在Hive中操作实现和HBase整合的过程。

#### 1、在Hive中操作新建表

``` shell
create external table wangqiang_test_01 (key int, value string) stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' with serdeproperties ("hbase.columns.mapping"=":key,cfl:val") tblproperties ("hbase.table.name"="default:wangqiang_test_01");
```

### 二、从HBase到Hive

这种方式是指，初始状态HBase中已经有数据表，通过操作Hive实现和HBase关联的过程。

#### 1、在HBase中创建表

```shell
create 'default:wangqiang_test_02', {NAME=>'basic_info'}, {NAME=>'other_info'}
```

#### 2、在HBase中插入数据

```shell
put 'wangqiang_test_02', '00017','basic_info:name','tom'
put 'wangqiang_test_02', '00017','basic_info:age','17'
put 'wangqiang_test_02', '00017','basic_info:sex','man'
put 'wangqiang_test_02', '00017','other_info:telPhone','176xxxxxxxx'
put 'wangqiang_test_02', '00017','other_info:country','China'
put 'wangqiang_test_02', '00023','basic_info:name','mary'
put 'wangqiang_test_02', '00023','basic_info:age',23
put 'wangqiang_test_02', '00023','basic_info:sex','woman'
put 'wangqiang_test_02', '00023','basic_info:edu','college'
put 'wangqiang_test_02', '00023','other_info:email','cdsvo@163.com'
put 'wangqiang_test_02', '00023','other_info:country','Japan'
```

#### 3、在Hive中创建一个外部表

```shell
create external table wangqiang_test_02(id int,
name string,
age string,
sex string,
edu string,
country string,
telPhone string,  
email string
)
stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
with serdeproperties ("hbase.columns.mapping" = "
:key,
basic_info:name,
basic_info:age,
basic_info:sex,
basic_info:edu,
other_info:country,
other_info:telPhone,
other_info:email
")
tblproperties("hbase.table.name" = "default:wangqiang_test_02");
```