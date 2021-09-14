# cassandra的相关配置与操作

介绍cassandra的配置方式与常用的操作。

本文基于cassandra的3.11.2版本的使用经验总结而成。

各版本的官方文档： https://cassandra.apache.org/doc/

### 一、服务端配置

#### 1、配置

###### 部署机器

192.168.1.1

192.168.1.2

以其中一个机器的配置举例。

###### 配置文件【cassandra-env.sh】

MAX_HEAP_SIZE="1G"

HEAP_NEWSIZE="200M"

######配置文件【cassandra.yaml】

storage_port: 7000

ssl_storage_port: 7001

native_transport_port: 9042

rpc_port: 9160

listen_address: 192.168.1.1

seeds: 192.168.1.1,192.168.1.2  # 采用的是Gossip协议

rpc_keepalive: false

###### 启动命令

nohup bin/cassandra &

### 二、cql操作指南

cql是cassandra专用的查询语言，和sql语言相似。

通过bin/cqlsh 登录cassandra服务并进入cql命令行交互界面；这个操作实际上是调用cqlsh.py。

使用上面命令之前，需要修改cqlsh.py脚本的DEFAULT_HOST配置项。

keyspace： 相当于关系型数据库的database。

table： 相当于关系型数据库的table。

#### 1、create keyspace

创建ks的命令

*开发环境通常不需要数据副本：*

CREATE KEYSPACE ks_dictxwang WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'}  AND durable_writes = false;

*生成环境最好设置数据副本，避免单点造成数据丢失：*

CREATE KEYSPACE ks_dictxwang WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '2'}  AND durable_writes = false;

*合并代码或者pull操作都可能产生冲突*

#### 2、create table

创建table（表、列簇）的命令

use ks_dictxwang;  # 首先进入ks空间

create table tab_001 (doc_id varchar, partner_key varchar, doc_key varchar, title varchar, doc_detail text, pv_count int, create_time timestamp, update_time timestamp, primary key(doc_id));

*查看cassandra支持的数据类型（Data Types）*

https://cassandra.apache.org/doc/latest/cassandra/cql/types.html#native-types

#### 3、describe

查看ks或者table的信息，主要是创建信息。

describe keyspaces;  # 查找所有的ks

describe ks_dictxwang; # 查看指定的ks

describe tab_001; # 查看指定的table

#### 4、创建索引

create index doc_partner_key_idx on ks_dictxwang.tab_001(partner_key);

#### 5、数据查询

*主键查询*

select * from tab_001 where doc_id = 'test001';

*非主键查询*

select * from tab_001 where create_time >= '2021-05-04' ALLOW FILTERING;

*（非主键查询时，当扫描数量超过 tombstone_failure_threshold 限制，会报错，此时可以适当调大 tombstone_failure_threshold 数值）*

#### 6、alter table

用于修改非primary_key字段

alter table tab_001 drop doc_detail;
alter table tab_001 add doc_details varchar;

#### 7、expand on

格式化输出

在控制台执行 expand on 后续命令执行结果就会用k-v形式格式输出；和mysql的\G功能类似。

### 三、编程访问cassandra

#### 1、java访问cassandra

##### （1）spring-data方式

java8 + spring5.x + spring-data-cassandra

http://www.cnblogs.com/zzd-zxj/p/6135481.html

##### （2）、cassandra-driver方式

http://www.baeldung.com/cassandra-with-java

##### （3）、thrift方式

*此方式已过时，官方不推荐使用。*

#### 2、python访问cassandra

安装cassandra-driver，pip install cassandra-driver

### 四、重启cassandra

重启cassandra服务之前，务必先执行 bin/nodetool flush 命令，确保memtable中俄数据都保存到了磁盘上，避免数据丢失。

*另外，bin/nodetool 还有一些其他的运维命令。*

### 五、启用G1垃圾回收

修改conf/jvm.options文件：放开注释 -XX:+UseG1GC ，同时注释掉 CMS Settings 下所有的配置项。

### 六、节点扩容与删除

操作步骤如下：

*i、部署新的节点，修改配置，其中seeds配置为任一存活的老节点*

*ii、关闭所有老节点的数据压缩 ./nodetool disableautocompaction ，提高数据转移速度*

*iii、启动新节点，同样关闭数据压缩 ./nodetool disableautocompaction*

*iv、迁移过程中，通过./nodetool status 命令查看新节点的状态，直至其状态为 UN*

*v、所有新节点迁移完成后，修改新节点的seeds配置，并重新启动*

*vi、下线老节点，在老节点上执行 ./nodetool decommission*

*注意： 新节点首次启动和老节点下线中的状态是UL*

