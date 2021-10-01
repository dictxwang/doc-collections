# Mysql主备模式安装与配置

本文记录在Linux CentOS下，使用rpm包安装mysql的流程，以及mysql主备模式的配置。

以Mysql的5.5.9-1版本为例。

### 一、移除历史版本

在安装mysql前，首先需要确认当前机器上是否存在历史版本的mysql，如果有先移除历史版本。

```shell
#查找是否有旧版本的mysql
># rmp -qa | grep mysql
># mysql-libs-5.1.61-4.el6.i686
#通过这样的命令彻底删除旧版本的mysql
># rpm -e --allmatches --nodeps mysql-libs-5.1.61-4.el6.i686
#也可以通过yum命令操作，如yum search和yum remove
```

### 二、安装Mysql

#### 1、下载rpm包

MySQL-server-5.5.9-1.linux2.6.x86_64.rpm

MySQL-client-5.5.9-1.linux2.6.x86_64.rpm

*rpm包搜索地址*

https://www.rpmfind.net/

*安装mysql的中途，如果缺少相关的依赖包，也可以通过上面的地址搜索下载*

#### 2、安装mysql rpm包

```shell
#安装server包
rpm -ivh MySQL-server-5.5.9-1.linux2.6.x86_64.rpm
rpm -ivh MySQL-client-5.5.9-1.linux2.6.x86_64.rpm

## 也可以先只检查依赖关系，不执行安装，例如：
rpm -ivh --test MySQL-server-5.5.9-1.linux2.6.x86_64.rpm
## 同样也可以使用yum命令安装，例如：
yum install MySQL-server-5.5.9-1.linux2.6.x86_64.rpm

## mysql路径说明
/usr/share/mysql # mysql的安装路径
/var/lib/mysql # mysql的数据库路径，此路径最好在配置中修改，避免根路径磁盘空间太小
```

#### 3、设置root密码

```shell
/usr/bin/mysqladmin -u root password 'your-new-password'
```

### 三、完成基础配置

#### 1、复制配置文件

从安装路径 */usr/share/mysql/* 下复制*.cnf配置文件到 */etc/my.cnf*

#### 2、修改配置项

```shell
# 设置编码
[mysql] # 客户端
default-character-set=utf8
[mysqld] # 服务端
character-set-server=utf8
# 修改绑定地址
bind-address=192.168.3.1
## 这里只列出了主从模式需要的配置
## 配置完成后，通过service mysql start 即可启动mysql服务
```

### 四、主从模式配置

#### 1、准备工作

a、需要先准备两台服务器，假设服务器ip和角色分别为：

master： 192.168.3.1

slave： 192.168.3.2

b、在master和slave上分别安装mysql服务，安装方式即上面提到的几点。

c、在master和slave上分别创建同名数据库sample_repl

#### 2、修改master的数据库配置

```shell
# 在mysqld节点下增加
binlog-do-db=sample_repl #需要同步的库
binlog-ignore-db=mysql  #忽略主备的库
```

#### 3、在master上创建备份账号

```shell
# 创建账号
create user u_slave;
# 设置权限
grant replication slave on *.* to u_slave@'%' identified by 'u_slave_password';
# 刷新权限
flush privileges;
```

#### 4、重启master数据库

```shell
service mysql restart;
# 查看master状态
show master status;
```

#### 5、配置slave

a、历史版本（5.1）

```shell
# 设置mysql服务编号，和master不同
server-id       = 2
master-host     =  192.168.3.1
master-user     =   u_slave
master-password =   u_slave_password
master-port     =  3306
master-connect-retry=60
replicate-do-db=sample_repl
replicate-ignore-db=mysql
```

b、当前版本（5.5开始）

```shell
change master to master_host='192.168.3.1', master_user='u_slave', master_password='u_slave_password', master_port=3306, replicate_do_db='sample_repl', replicate_ignore_db='mysql';
```

#### 6、启动slave

```shell
# 登录mysql后执行
start slave；
```

#### 7、查看slave状态

```shell
show slave status; # 或者 show slave status \G;
# 确保关键配置无错
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Replicate_Do_DB: sample_repl
Replicate_Ignore_DB: mysql
```

*注意：如果要实现双机互备，就是将主备机器互换，把以上流程再执行一次*

### 五、一些问题的处理

#### 1、系统mysql用户缺失问题的处理

```shell
# 添加用户组
groupadd mysql
# 添加用户目录
mkdir /home/mysql
# 添加用户
useradd -g mysql -d /home/mysql mysql
# 修改mysql安装目录权限
cd /usr/share/mysql
chown -R mysql:mysql .
# 修改mysql安装用户
mysql_install_db --user=mysql
# 修改root密码
mysqladmin -uroot -p 'your-root-password'
```

#### 2、查看数据库关键信息

```shell
# 查看编译参数
cat /usr/local/mysql/bin/mysqlbug | grep CONFIGURE_LINE

# 查看数据库的引擎
show engines;
# 查看数据库版本信息
\s;
# 查看数据的编码情况，或其他参数新
show variables like '%character%';
```

#### 3、主备模式主键冲突解决

```shell
# 在主数据库配置中添加
auto_increment_offset=1
auto_increment_increment=2

# 在从数据库配置中添加
auto_increment_offset=2
auto_increment_increment=2
```

#### 4、hostname变更出现问题的处理

如果修改了server的hostname，需要删除relay log，重置master-slave同步

Failed to open the relay log './localhost-relay-bin.000031' Could not find target log during relay log initialization

删除relay log，包括localhost-relay-bin.000031 localhost-relay-bin.index  relay-log.info

#### 5、名字映射问题的处理

错误信息

```shell
130330 23:17:19 [ERROR] Slave I/O: error connecting to master 'slaver@192.168.3.1:3306' - retry-time: 60 retries: 86400, Error_code: 1042
130330 23:18:21 [Warning] IP address '192.168.3.1' could not be resolved: no reverse address mapping.
```

解决办法

```shell
# 在master数据库增加配置项
[mysqld]
skip-name-resolve
```