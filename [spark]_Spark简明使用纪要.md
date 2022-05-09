# Spark简明使用纪要

### 零、配置

#### 1、基于环境变量的配置

```shell
> cp conf/spark-env.sh.template conf/spark-env.sh
> vim spark-env.sh
	export JAVA_HOME=/usr/lib/java/jdk1.8.0_151
	export SPARK_MASTER_IP=x.x.x.x
	export SPARK_MASTER_PORT=7077
	export SPARK_WORKER_CORES=1
	export SPARK_WORKER_INSTANCES=1
	export SPARK_WORKER_MEMORY=900M
	export HADOOP_HOME=/data/hadoop-2.8.1/  # hadoop主目录路径（非必须）
	export HADOOP_CONF_DIR=/data/hadoop-2.8.1/etc/hadoop/  # hadoop配置文件路径（非必须）
```

### 一、独立集群模式

#### 1、只启动客户端模式

```shell
> bin/spark-shell  # 这种情况下不需要slave(worker节点)
> :quit  # 退出shell
```

#### 2、连接服务的模式

```shell
> sbin/start-master.sh  # 默认监听7077端口，webui监听8080端口
> sbin/start-slave.sh spark://x.x.x.x:7077  # 启动worker单节点
> bin/spark-shell --master spark://x.x.x.x:7077  # 启动shell终端，并连接到master独立集群
```

#### 3、spark-shell操作示例

```shell
> bin/spark-shell  # 启动shell终端
scala> var textFile = spark.read.textFile("/data/temp/spark/ming.txt")  # 读取文本文件
scala> textFile.count()  # 统计文件行数
scala> textFile.filter(line=>line.contains("ming")).count()  # 过滤包含关键字的行数
```

### 二、集群启动

#### 1、在Mesos上启动

#### 2、在YARN上启动

### 三、提交任务

#### 1、读取本地数据文件的任务

```shell
> vim /data/temp/spark/first_test.py
    # -*- coding: utf8 -*-
    if __name__=="__main__":
        from pyspark import SparkContext
        sc=SparkContext(appName='firt app')
        word=sc.textFile('file:///data/temp/spark/ming.txt')
        num_i=word.filter(lambda s:'ming' in s).count()
        print(num_i)
> bin/spark-submit /data/temp/spark/first_test.py  # 提交到spark本地
> bin/spark-submit --master spark://10.143.55.57:7077 /data/temp/spark/first_test.py  # 也可以提交到spark独立集群
```

#### 2、读取hdfs数据文件的任务

```shell
> vim /data/temp/spark/second_test.py
    # -*- coding: utf8 -*-
    if __name__=="__main__":
        from pyspark import SparkContext
        sc=SparkContext(appName='firt app')
        word=sc.textFile('hdfs://10.153.55.57:9000/wangqiang/spark/ming.txt')  # 如果配置了hadoop信息，这里可以省略hdfs://10.153.55.57:9000协议和路径前缀信息
        num_i=word.filter(lambda s:'ming' in s).count()
        print(num_i)
> bin/spark-submit /data/temp/spark/second_test.py  # 提交到spark本地
> bin/spark-submit --master spark://10.143.55.57:7077 /data/temp/spark/second.test.py  # 也可以提交到spark独立集群
```

### 四、spark访问hive

#### 1、修改配置

修改hive-site.xml的配置项：

```shell
     <property>
        <name>hive.metastore.uris</name>
        <value>thrift://10.143.55.57:9083</value>
     </property>
```

将hive-site.xml复制到$SPARK_HOME/conf路径下
复制$HIVE_HOME/lib/mysql-connector-java-5.1.34.jar 到 $SPARK_HOME/jars/ 路径下

#### 2、启动spark-shell

```shell
> bin/spark-shell --master spark://10.143.55.57:7077 --conf spark.sql.hive.metastore.version=2.1.1 --conf spark.sql.hive.metastore.jars=/data/hive-2.1.1/lib/*  # 如果spark内置版本和hive版本不一致，需要指定这两个配置；也可以将配置项写入spark-defaults.conf
```

#### 3、执行spark-sql命令

```shell
scala> spark.sql("use default");  # 切换数据库（如果不切换默认是default）
scala> spark.sql("show tables").show();  # 显示当前库的所有表
scala> spark.sql("select * from wq_load_data_bucket").show();  # 查询数据并显示
```

#### 4、从hive表读取数据保存到hdfs

```shell
scala> val tf = spark.sql("select * from wq_load_data_bucket");  # 读取hive数据，读出来的数据类型是DataFrame
scala> tf.rdd.saveAsTextFile("/wangqiang/spark/bucket01")  # 这里指定的是文件路径，最终会保存成多个文件
/wangqiang/spark/bucket01/_SUCCESS
/wangqiang/spark/bucket01/part-00000
/wangqiang/spark/bucket01/part-00001
/wangqiang/spark/bucket01/part-00002
/wangqiang/spark/bucket01/part-00003
scala> tf.rdd.partitions.length  # 查看rdd分区数量，这个数量和最终存储到hdfs文件数量一致
scala> tf.rdd.repartition(100).saveAsTextFile("/wangqiang/spark/dev_basic_info_2")  # 重新进行数据分区，控制文件落盘数量
scala> tf.toJSON.rdd.repartition(100).saveAsTextFile("/wangqiang/spark/dev_basic_info_3")  # 转成json字符串再落盘
```

### 五、提交scala应用

#### 1、配置编程环境

安装scala

安装sbt scala的打包工具

安装idea的scala插件

#### 2、编写代码

新建first工程

配置工程依赖 build.sbt

```scala
    libraryDependencies += "org.apache.spark" %% "spark-core" % "3.2.1"
```

编写first.tt.HelloWorld.scala 类


```scala
package first.tt
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.SparkConf
object HelloWorld {
  def main(args: Array[String]) {
    // hadoop上的文件
    val logFile = "/wangqiang/spark/ming.txt"
    // 也可以是一批文件
    // val logFile = "/wangqiang/spark/batch/*"
    val conf = new SparkConf().setAppName("First Application")
    val sc = new SparkContext(conf)
    val logData = sc.textFile(logFile, 2).cache()
    val numAs = logData.filter(line => line.contains("ming")).count()
    val numBs = logData.filter(line => line.contains("Ming")).count()
    println(s"Lines with ming: $numAs, Lines with Ming: $numBs")
    sc.stop()
  }
}
```
#### 3、打包jar

sbt package  # first_2.13-0.1.jar

并将jar包上传到spark所在的服务器

#### 4、提交spark应用

```shell
bin/spark-submit --master spark://10.143.55.57:7077 --class=first.tt.HelloWorld /data/temp/wangqiang/first_2.13-0.1.jar
```

### 六、测试自带示例

#### 1、与kafka结合的测试

```shell
bin/run-example streaming.JavaDirectKafkaWordCount 10.143.55.166:9092 spark_t spark_test_1  # 从kafka接收消息，并计算单词数
bin/run-example streaming.DirectKafkaWordCount 10.143.55.166:9092 spark_t spark_test_1 # scala版本的kafka消息流
```

#### 2、与tcp结合的测试

```shell
nc -lk 9999  # 使用netcat启动tcp服务
bin/run-example streaming.NetworkWordCount localhost 9999  # 从tcp服务获取数据流
```

