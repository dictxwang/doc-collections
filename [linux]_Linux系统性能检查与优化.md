# Linux系统性能检查与优化

### 一、性能检查常用命令

#### 1、系统概况

​	ulimit -n 查看当前用户允许打开的文件数

​	cat /proc/sys/fs/file-max  查看允许打开的文件硬限制

​	cat /proc/sys/net/ipv4/ip_local_port_range 查看可用端口号范围，用这种方式还可以查看其它网络配置参数

#### 2、cpu与负载

​	cat /proc/cpu 查看cpu信息（cpu核数、主频、指令集、缓存信息等）

​	top 显示进程信息

​	top -Hp pid  显示进程对应的线程信息

​	top -n1 打印一次

​	top -d 5 每5秒钟统计一次

​	top -c 显示完整的进程运行命令

​	top命令的交互参数

​		M 进程按内存占用倒序排列

​		P 进程按cpu占用倒序排列

​		c 显示完整的进程命令

​	

​	ps 显示进程的快照信息

​	ps aux 显示所有的进程

​	ps pid 显示指定进程的信息



​	dstat 系统资源统计的多功能工具（包括内存、cpu、磁盘io、网络io等）

​	

​	vmstat 查看虚拟内存统计信息

​	vmstat 2 10  每2秒统计一次，共统计10次

​	

​	sar 收集、报告并保存系统的动态信息

​	sar命令实际上是通过查看系统日志（/var/log/sa下的日志文件）

#### 3、内存

​	free 查看系统内存总量和使用量

​	free -h  将1024换算成1000

​	free -m 以MB单位显示

​	cat /proc/meminfo 查看系统内存信息（总量、使用量、内存型号信息等）

#### 4、网络

​	netstat 查看网络链接状态

​	ss -anti | grep -B 1 retrans  # 查看tcp重传数

​	cat /proc/net/snmp # 网络流量分析

​    netstat -s # 网络流量查看另一种方式（netstat -su只查看udp，netstat -st只查看tcp）

​    dstat -nf  实时流量监控

​    nload 实时流量监控（需要事先安装）

​    iftop 实时流量监控，能看到流量两端地址（需要事先安装）

​    tcpdump 网络抓包分析

​    iptables 防火墙设置

​    traceroute 路由跟踪工具

​    route 查看或管理本机路由表（route -n 用点分十进制ip代替主机名）

​    mtr 网络连通测试工具（结合了ping、traceroute、nslookup）

​    nslookup 域名查询

​    cat /etc/sysconfig/network 查看域名、网关、DNS服务器信息

​    ethtool 查看或配置网络硬件设备（网卡）

#### 5、磁盘IO

​    fdisk 磁盘分区工具

​	fdisk -l  显示磁盘信息列表



​	lsblk 显示块设备（磁盘）信息

​	lsblk -d -o name,rota 查看磁盘类型(SSD,HDD)，其中rota=1为HDD/rota=0为SSD

​	lsblk -d -f 查看磁盘格式化类型



​    df 显示磁盘空间使用情况

​	df -Th 查看磁盘格式化类型

​	df -hi 查看文件可挂载数量



​    du 估算文件空间使用情况

​	du -h 以方便人工识别的（K、M、G）单位展示

​	du --max-depth=1 只估算当前路径的统计结果，不估算子路径的统计结果



​    iostat 统计输出cpu和io设备（磁盘）的使用情况

​    

​	lsof 展示打开的文件

​	lsof -p pid 查看某一进程打开的文件

​	lsof -i 查看活动的链接

​	lsof -i:8090 查看监听8090端口的链接

​	lsof -i tcp:8090 查看监听tcp端口8090的进程信息

​	lsof -u root 查看用户root打开的文件

#### 6、日志

​    cat /var/log/sa/saxx: 通常会保存前30天的日志

​    cat /var/log/meesage.xx: 通常会保存前30天的消息

​    cat /proc/${pid}/oom_score: 进程的oom得分，系统资源不足时会优先kill oom得分最高（oom_score+oom_score_adj）的进程 oom_score_adj可以理解为自定义的优先级

### 二、常见性能问题解决方案

#### 1、解决tcp链接TIME_WAIT过多引起的问题

TIME_WAIT是tcp协议中，主动断开的一方进入的中间状态，默认等待两个MSL后，tcp链接终止。

通过修改tcp相关的系统内核参数解决：

##### a、编辑内核参数文件

​    sudo vim /etc/sysctl.conf

##### b、修改以下4个参数项

​	#修改系统默认的timeout时间，默认是63s

​    net.ipv4.tcp_fin_timeout=2

​	#开启tcp链接的重复使用（默认0）

​    net.ipv4.tcp_tw_reuse=1

​	#开启时间戳同步方式（主要影响RTT的计算）

​    net.ipv4.tcp_timestamps=1

​	#开启连接的快速回收（默认0）

​    net.ipv4.tcp_tw_recycle=1
##### c、重新加载参数

​	sudo sysctl -p

#### 2、实现空闲链接的快速回收

​	针对keepalive的场景下是用

​	修改系统参数 sudo vim /etc/sysctl.conf

​	#表示发送探测报文之前的链接空闲时间，默认为7200

​	net.ipv4.tcp_keepalive_time=60

​	#表示两次探测报文发送的时间间隔，默认为75

​	net.ipv4.tcp_keepalive_probes=3

​	#表示探测的次数

​	net.ipv4.tcp_keepalive_intvl=10

#### 3、tcp协议中sync包重传配置

​	修改系统参数 sudo vim /etc/sysctl.conf

​	#syn重传多少次后放弃（默认5）

​	net.ipv4.tcp_syn_retries=

​	#syn ack重传多少次后放弃（默认5）

​	net.ipv4.tcp_synack_retries

​	#syn 包队列

​	net.ipv4.tcp_max_syn_backlog

#### 4、查看tcp链接数的方法
##### a、方法1

```shell
ss -ant | awk 'NR>1 {a[$1]++} END {for (b in a) print b,a[b]}'
ESTAB 535
TIME-WAIT 80
LISTEN 13
```

##### b、方法2

```shell
netstat -an | awk '/^tcp/ {a[$NF]++} END {for (b in a) print b,a[b]}’
TIME_WAIT 91
SYN_SENT 7
ESTABLISHED 535
LISTEN 13
```

##### c、方法3

cat /proc/net/snmp

#### 5、tcp常见网络报错与分析
##### a、查看丢包情况

​	（1）网卡发现系统处理繁忙丢弃数据包        	

​		cat /proc/net/dev  #关注第4列出错数和第5列丢弃数

​    （2）传统非 NAPI 接口实现的网卡驱动，每个 CPU 有一个队列，当在队列中缓存的数据包数量超过 net.core.netdev_max_backlog 时，网卡驱动程序会丢掉数据包

​        cat /proc/net/softnet_stat    

（3）应用程序处理繁忙，系统丢弃数据包

​		#ListenOverflows：当 socket 的 listen queue 已满，当新增一个连接请求时，应用程序来不及处理；

​		#ListenDrops：包含上面的情况，除此之外，当内存不够无法为新的连接分配 socket 相关的数据结构时，也会加 1，当然还有别的异常情况下会增加 1

​        cat /proc/net/netstat | awk '/TcpExt/ { print $21,$22 }’

##### b、内存不足的情况

​	（1）内存分配不足
​     	   #查看分配情况

```shell
	cat /proc/sys/net/ipv4/tcp_mem
    183474  244633  366948
```

​        	简单来说，三个值分别表示进入 无压力、压力模式、内存上限的值，当到达最后一个值的时候就会报错。       

 			#查看使用情况     

```shell
	cat /proc/net/sockstat
	sockets: used 855
	TCP: inuse 17 orphan 1 tw 0 alloc 19 mem 3
	UDP: inuse 16 mem 10
	UDPLITE: inuse 0
	RAW: inuse 1
	FRAG: inuse 0 memory 0
```

​    （2）orphan sockets链接数量过多

​			#查看允许的最大orphans sockets数量

```
	cat /proc/sys/net/ipv4/tcp_max_orphans
	32768
```

​			#查看已有的orphans sockets数量

​			cat /proc/net/sockstat

 			*... ...*     *TCP: inuse 37 orphan 14 tw 8 alloc 39 mem 9*     *... …*

#### 6、内存整理
##### a、查看内存占用前20的进程：

​    ps aux | awk '{print $2, $4, $11}' | sort -k2rn | head -n 20

​    其中$4是指内存占用比例

##### b、内存清理

​    修改了jvm的内存参数后，会出现占用的内存不被释放的问题

​    echo 1 > /proc/sys/vm/drop_caches  # 这里可以通过改变信号量(1)，清理不同的内存(cache\buffer等)

##### c、内存信息

​    查看内存条数量

​    dmidecode | grep -A16 "Memory Device$"

=========================================================================================

### 附1、重要参数汇总

通过man dstat手册文档可以查看到

```shell
Performance tools
       dstat(1), ifstat(1), iftop(8), iostat(1), mpstat(1), netstat(1), nfsstat(1), nstat, vmstat(1), xosview(1), tcpdump(1)

   Debugging tools
       htop(1), lslk(1), lsof(8), top(1)

   Process tracing
       ltrace(1), pmap(1), ps(1), pstack(1), strace(1)

   Binary debugging
       ldd(1), file(1), nm(1), objdump(1), readelf(1)

   Memory usage tools
       free(1), memusage, memusagestat, slabtop(1)

   Accounting tools
       dump-acct, dump-utmp, sa(8)

   Hardware debugging tools
       dmidecode, ifinfo(1), lsdev(1), lshal(1), lshw(1), lsmod(8), lspci(8), lsusb(8), smartctl(8), x86info(1)

   Application debugging
       mailstats(8), qshape(1)

   Xorg related tools
       xdpyinfo(1), xrestop(1)

   Other useful info
       collectl(1), proc(5), procinfo(8), pidstat, sar, ifconfig
```

### 附2、常见配置示例

#### 1、sysctl.conf 优化配置示例文件

```shell
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_wmem = 8192 4336600 873200
net.ipv4.tcp_rmem = 32768 4336600 873200
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 786432 1048576 1572864
net.ipv4.tcp_fin_timeout = 30
#net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_time = 300
net.ipv4.ip_local_port_range = 1024 65000
```

