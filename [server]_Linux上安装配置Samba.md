# Linux上安装配置Samba

Samba，是种用来让UNIX系列的操作系统与微软Windows操作系统的SMB/CIFS（Server Message Block/Common Internet File System）网络协议做链接的自由软件。可以利用Samba搭建团队内的共享磁盘。

本文记录在CentOS上安装配置Samba的过程。

### 一、安装与启动

#### 1、查看是否已安装

```shell
rpm -qa | grep samba
```

#### 2、安装Samba

```shell
yum install samba
```

#### 3、设置开机启动

```shell
chkconfig smb on
```

#### 4、启动Samba服务

```shell
service smb start
```

### 二、配置与管理

#### 1、配置Samba

Samba配置文件路径是 /etc/samba/smb.conf。

配置文件是分节点管理的，需要配置的节点包括[global]、[homes]、[printers]、[public]和[SambaServer]。

#### 2、管理Samba

##### a、用户管理

Samba用户管理命令是 smbpasswd，常用的操作包括：

```shell
smbpasswd -a username #添加用户
smbpasswd -d username #冻结用户
smbpasswd -e username #恢复用户
smbpasswd -n username # 密码置空 (需要在global中配置 null passwords -true)
smbpasswd -x username # 删除用户
```

添加Samba用户的完整流程Shell脚本示例：

```shell
#!/bin/bash

user_id=$1

echo "add samba user:$user_id"

# 在samba文件根路径创建用户特有的路径
sudo mkdir /data/share/$user_id
# 在Linux系统中添加用户
sudo useradd -d /data/share/$user_id -g share $user_id
# 变更路径拥有者和访问权限
sudo chown -R $user_id:share /data/share/$user_id
sudo chmod a+xr /data/share/$user_id

# 设置用户登录Linux系统的密码
sudo passwd $user_id
# 添加Samba用户，并设置密码
sudo smbpasswd -a $user_id

echo "finish add samba user:$user_id"
```

##### b、数据库管理

Samba有专用数据库用于保存用户信息，并且提供了用户数据管理命令pdbedit。

```shell
pdbedit -L #查看有哪些可以的samba用户
```

### 三、常见问题

#### 1、文件名乱码

在[global]节点添加 unix charset = gb2312。

#### 2、显示隐藏文件

在[global]节点添加 veto files = /.*/，即可屏蔽掉隐藏文件。

#### 3、windows连接失败

这个有可能是smbd绑定的端口问题，正常情况下smbd会同时监听139和445两个端口，在[global]添加 smb ports=139 配置项可以解决。

#### 4、windows认证失败

这种情况是由于windows下的远程访问加密与sbmd不兼容导致可以在[global]节点添加 server signing = auto 配置项可以解决。

### 四、配置文件示例

```shell
[global]
	
# ----------------------- Network Related Options -------------------------
#
# workgroup = NT-Domain-Name or Workgroup-Name, eg: MIDEARTH
#
# server string is the equivalent of the NT Description field
#
# netbios name can be used to specify a server name not tied to the hostname
#
# Interfaces lets you configure Samba to use multiple interfaces
# If you have multiple network interfaces then you can list the ones
# you want to listen on (never omit localhost)
#
# Hosts Allow/Hosts Deny lets you restrict who can connect, and you can
# specifiy it as a per share option as well
#
	# 如果是入域（WANG-INC.COM）的windows机器需要访问samba，需要添加域后缀（WANG-INC）
	workgroup = WORKGROUP, WANG-INC
	server string = YourSambaServerName
	
	netbios name = SambaServer
	smb ports = 139
	server signing = auto
	veto files = /.*/
;	interfaces = lo eth0 192.168.12.2/24 192.168.13.2/24 
;	hosts allow = 127. 192.168.12. 192.168.13.
	
# --------------------------- Logging Options -----------------------------
#
# Log File let you specify where to put logs and how to split them up.
#
# Max Log Size let you specify the max size log files should reach
	
	# logs split per machine
	log file = /var/log/samba/log.%m
	# max 50KB per log file, then rotate
	max log size = 50
	
# ----------------------- Standalone Server Options ------------------------
#
# Scurity can be set to user, share(deprecated) or server(deprecated)
#
# Backend to store user information in. New installations should 
# use either tdbsam or ldapsam. smbpasswd is available for backwards 
# compatibility. tdbsam requires no further configuration.

	security = user
	passdb backend = tdbsam
	# 避免中文文件名乱码
	unix charset = gb2312

# ----------------------- Domain Members Options ------------------------
#
# Security must be set to domain or ads
#
# Use the realm option only with security = ads
# Specifies the Active Directory realm the host is part of
#
# Backend to store user information in. New installations should 
# use either tdbsam or ldapsam. smbpasswd is available for backwards 
# compatibility. tdbsam requires no further configuration.
#
# Use password server option only with security = server or if you can't
# use the DNS to locate Domain Controllers
# The argument list may include:
#   password server = My_PDC_Name [My_BDC_Name] [My_Next_BDC_Name]
# or to auto-locate the domain controller/s
#   password server = *
	
	
;	security = domain
;	passdb backend = tdbsam
	realm = WANG-INC

;	password server = <NT-Server-Name>

# ----------------------- Domain Controller Options ------------------------
#
# Security must be set to user for domain controllers
#
# Backend to store user information in. New installations should 
# use either tdbsam or ldapsam. smbpasswd is available for backwards 
# compatibility. tdbsam requires no further configuration.
#
# Domain Master specifies Samba to be the Domain Master Browser. This
# allows Samba to collate browse lists between subnets. Don't use this
# if you already have a Windows NT domain controller doing this job
#
# Domain Logons let Samba be a domain logon server for Windows workstations. 
#
# Logon Scrpit let yuou specify a script to be run at login time on the client
# You need to provide it in a share called NETLOGON
#
# Logon Path let you specify where user profiles are stored (UNC path)
#
# Various scripts can be used on a domain controller or stand-alone
# machine to add or delete corresponding unix accounts
#
;	security = user
;	passdb backend = tdbsam
	
;	domain master = yes 
;	domain logons = yes
	
	# the login script name depends on the machine name
;	logon script = %m.bat
	# the login script name depends on the unix user used
;	logon script = %u.bat
;	logon path = \\%L\Profiles\%u
	# disables profiles support by specifing an empty path
;	logon path =          
	
;	add user script = /usr/sbin/useradd "%u" -n -g users
;	add group script = /usr/sbin/groupadd "%g"
;	add machine script = /usr/sbin/useradd -n -c "Workstation (%u)" -M -d /nohome -s /bin/false "%u"
;	delete user script = /usr/sbin/userdel "%u"
;	delete user from group script = /usr/sbin/userdel "%u" "%g"
;	delete group script = /usr/sbin/groupdel "%g"
	
	
# ----------------------- Browser Control Options ----------------------------
#
# set local master to no if you don't want Samba to become a master
# browser on your network. Otherwise the normal election rules apply
#
# OS Level determines the precedence of this server in master browser
# elections. The default value should be reasonable
#
# Preferred Master causes Samba to force a local browser election on startup
# and gives it a slightly higher chance of winning the election
;	local master = no
;	os level = 33
;	preferred master = yes
	
#----------------------------- Name Resolution -------------------------------
# Windows Internet Name Serving Support Section:
# Note: Samba can be either a WINS Server, or a WINS Client, but NOT both
#
# - WINS Support: Tells the NMBD component of Samba to enable it's WINS Server
#
# - WINS Server: Tells the NMBD components of Samba to be a WINS Client
#
# - WINS Proxy: Tells Samba to answer name resolution queries on
#   behalf of a non WINS capable client, for this to work there must be
#   at least one	WINS Server on the network. The default is NO.
#
# DNS Proxy - tells Samba whether or not to try to resolve NetBIOS names
# via DNS nslookups.
	
;	wins support = yes
;	wins server = w.x.y.z
;	wins proxy = yes
	
;	dns proxy = yes
	
# --------------------------- Printing Options -----------------------------
#
# Load Printers let you load automatically the list of printers rather
# than setting them up individually
#
# Cups Options let you pass the cups libs custom options, setting it to raw
# for example will let you use drivers on your Windows clients
#
# Printcap Name let you specify an alternative printcap file
#
# You can choose a non default printing system using the Printing option
	
	load printers = no
	cups options = raw

;	printcap name = /etc/printcap
	#obtain list of printers automatically on SystemV
;	printcap name = lpstat
;	printing = cups

# --------------------------- Filesystem Options ---------------------------
#
# The following options can be uncommented if the filesystem supports
# Extended Attributes and they are enabled (usually by the mount option
# user_xattr). Thess options will let the admin store the DOS attributes
# in an EA and make samba not mess with the permission bits.
#
# Note: these options can also be set just per share, setting them in global
# makes them the default for all shares

;	map archive = no
;	map hidden = no
;	map read only = no
;	map system = no
;	store dos attributes = yes


#============================ Share Definitions ==============================
	
[homes]
	comment = Home Directories
	browseable = no
	writable = yes
;	valid users = %S
;	valid users = MYDOMAIN\%S
	
[printers]
	comment = All Printers
	path = /var/spool/samba
	browseable = no
	guest ok = no
	writable = no
	printable = no
	
# Un-comment the following and create the netlogon directory for Domain Logons
;	[netlogon]
;	comment = Network Logon Service
;	path = /var/lib/samba/netlogon
;	guest ok = yes
;	writable = no
;	share modes = no
	
	
# Un-comment the following to provide a specific roving profile share
# the default is to use the user's home directory
;	[Profiles]
;	path = /var/lib/samba/profiles
;	browseable = no
;	guest ok = yes
	
	
# A publicly accessible directory, but read only, except for people in
# the "staff" group
	[public]
	comment = Public Stuff
	path = /data/share/FrontPublic
	browseable = no
	public = no
	writable = no
	printable = no
;	write list = +staff

	[SambaServer]
	comment = Default Position
	path = /data/share/share
	public = no
	writable = yes
	valid users = @share

```

