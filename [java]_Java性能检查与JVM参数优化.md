# Java性能检查与JVM参数优化

### 一、排查进程占用cpu高的常用步骤

1、通过top命令查看cpu占用高的进程号，如：123

2、进一步找到占用cpu高的线程号： top -H -p 123，假设线程号27433

3、将上一步的线程号转换成16进制： prinf "%x\n” 27433，转换结果6b29n

4、通过jstack打印堆栈信息，jstack 123 > jstack.123

5、通过第3步转换得到的线程16进制，在jstack.123中找到线程状态

### 二、堆栈内存分析

1、通过jmap命令输出堆栈信息文件： jmap -dump:format=b,file=jmap_dump.file pid

2、通过jhat查看或者通过mat工具分析

  a、jhat查看： jhat -J-mx512m [file] file是dump出来的二进制文件，默认端口7000 ([http://localhost:7000](http://localhost:7000/)类似这样查看)

  b、mat工具： http://www.eclipse.org/mat/downloads.php 下载mat后使用

### 三、性能状态查看命令

#### 1、jstat

   jstat -gccapacity 62211 打印内存使用情况

   jstat -gcutil 62211 2000 10 打印gc情况

  jstat -gc 62211 查看gc情况

#### 2、jinfo

   jinfo 62211 打印jvm详细信息（包括安装路径、版本、配置等）

#### 3、jps

  jps 查看系统运行的java进程号信息

#### 4、jmap

   jmap -histo 62211 打印对象情况

   jmap -heap 62211 打印jvm内存配置和使用情况

   jmap -dump:format=b,file=jmap-dump.142.97.old 28030 dump出内存信息，二进制格式，需要通过jhat查看

### 四、jvm参数优化

#### 1、调优前置相关参数

-XX:+PrintGC gc概要信息

-XX:+PrintGCDetails 详细信息

-XX:+PrintGCTimestamps gc时间信息

-XX:+PrintGCApplicationStopedTime gc造成的应用暂停时间

-Xloggc:文件路径

#### 2、显示完整的堆栈信息

-XX:+OmitStackTraceInFastThrow  可以打出完整的堆栈信息

-XX:+HeapDumpOnOutOfMemeryError 当内存溢出时，dump出内存的快照，用来查找内存溢出的故障

#### 3、G1配置示例

-Xms256m -Xmx512m -Xss512k -XX:PermSize=64M -XX:MaxPermSize=128M -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+PrintGCDetails

-Xms2048m -Xmx2048m -Xss512k -XX:PermSize=128M -XX:MaxPermSize=256M -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+PrintGCDetails

-Xmx2048m -XX:MaxPermSize=256m -XX:+UseG1GC -XX:MaxGCPauseMillis=200

注意：在某些1.8的jdk版本中，持久代的相关配置（ -XX:PermSize，-XX:MaxPermSize）已不需要了。

### 附：其他

#### 1、jps查看不到java进程的问题

  现象：ps命令可以看到，但是jps看不到

  原因：jps实际上是通过查找 /tmp/hsperfdata_${uid} 路径下的文件来展示的，但是/tmp/路径会被系统自动清理

  方案：可以通过指定jvm参数，将此路径重定向到其他路径 -Djava.io.tmpdir=/data/jps/