# hadoop执行python并行任务

摘要：记录基于hadoop的stream(**hadoop-streaming-2.8.1.jar**)执行python编写的map-reduce任务。hadoop并行任务的大致流程可以描述为：

 infile -> stdin -> mapper jobs -> partition/sort -> stdin -> reducer jobs > outfile

*python代码中map和reduce均通过sys.stdin获取上一步的输入数据。*

*以word-count任务进一步说明如下：*

### 一、编写脚本

#### 1、mapper脚本（/data/jobs/mapper-01.py）

```python
# -*- coding: utf8 -*-
__author__ = 'dictxwang'

import sys

for line in sys.stdin:

line = line.strip()
parts = line.split(" ")
for part in parts:
	print("%s\t1" % part)
```

#### 2、reducer脚本（/data/jobs/reducer-01.py）

```python
# -*- coding: utf8 -*-
__author__ = 'dictxwang'

import sys

current_word = None
current_count = 0
word = None

for line in sys.stdin:

    line = line.strip()
    word, count = line.split("\t", 1)
    if not count.isdecimal():
    	continue

    count = int(count)

    if current_word == word:
    	current_count += count
    else:
    if current_word:
    print("%s\t%s" % (current_word, str(current_count)))
    current_word = word
    current_count = count

if current_word == word:
    print("%s\t%s" % (current_word, current_count))
```

### 二、脚本验证（通过管道方式）

假设有本地数据文件：/data/jobs/mr/1.log, /data/jobs/mr/2.log

验证命令： cat /data/jobs/mr/*.log | python3.6 mapper-01.py | sort -k1,1 | python3.6 reducer-01.py

### 三、推送本地数据文件到hdfs

```shell
bin/hadoop fs -put -f /data/jobs/mr/*.log /dictxwang/w01/
```

### 四、执行map-reduce任务

#### 1、方式一：指定单个python脚本文件

```shell
# 清理输出目录
bin/hadoop fs -rm -r /dictxwang/w01-out

# 任务执行
bin/hadoop jar ./share/hadoop/tools/lib/hadoop-streaming-2.8.1.jar -input /dictxwang/w01/*.log -output /dictxwang/w01-out/ -mapper "python3.6 mapper-01.py"  -reducer "python3.6 reducer-01.py" -file "/data/jobs/mapper-01.py" -file "/data/jobs/reducer-01.py" -jobconf mapred.reduce.tasks=2

# 验证输出（任务执行完成，会自动生成一个_SUCCESS 空文件）
bin/hadoop fs -ls /dictxwang/w01-out
```

#### 2、方式二：指定多个python脚本文件

```shell
# 清理输出目录
bin/hadoop fs -rm -r /dictxwang/w02-out

# 任务执行（其中 -file "/data/jobs/mr02/map-reduce-02" 指定的是一个python工程，由多个脚本文件组成）
bin/hadoop jar ./share/hadoop/tools/lib/hadoop-streaming-2.8.1.jar -input /dictxwang/w02/*.log -output /dictxwang/w02-out/ -mapper "python3.6 mapper-02.py"  -reducer "python3.6 reducer-02.py" -file "/data/jobs/mr02/map-reduce-02" -jobconf mapred.reduce.tasks=2

# 验证输出（任务执行完成，会自动生成一个_SUCCESS 空文件）
bin/hadoop fs -ls /dictxwang/w02-out
```

### 附、参数说明

#### 1、stream命令参数

```shell
-input 输入数据路径
-output 输出数据路径
-mapper mapper可执行程序或Java类
-reducer reducer可执行程序或Java类
-file  Optional 分发本地文件
-cacheFile  Optional 分发HDFS文件
-cacheArchive  Optional 分发HDFS压缩文件
-numReduceTasks  Optional reduce任务个数
-jobconf | -D NAME=VALUE Optional 作业配置参数
-combiner  Optional Combiner Java类
-partitioner  Optional Partitioner Java类
-inputformat Optional InputFormat Java类
-outputformat Optional OutputFormat Java类
-inputreader  Optional InputReader配置
-cmdenv = Optional 传给mapper和reducer的环境变量
-mapdebug  Optional mapper失败时运行的debug程序
-reducedebug  Optional reducer失败时运行的debug程序
-verbose Optional 详细输出模式
```

#### 2、mr作业配置参数

```shell
mapred.job.name 作业名
mapred.job.priority 作业优先级
mapred.job.map.capacity 最多同时运行map任务数
mapred.job.reduce.capacity 最多同时运行reduce任务数
hadoop.job.ugi 作业执行权限
mapred.map.tasks map任务个数
mapred.reduce.tasks reduce任务个数
mapred.job.groups 作业可运行的计算节点分组
mapred.task.timeout 任务没有响应（输入输出）的最大时间
mapred.compress.map.output map的输出是否压缩
mapred.map.output.compression.codec map的输出压缩方式
mapred.output.compress reduce的输出是否压缩
mapred.output.compression.codec reduce的输出压缩方式
stream.map.output.field.separator map输出分隔符
```