# CentOS7下Docker的安装

参考：Docker安装官方说明文档 https://docs.docker.com/engine/install/centos/

*采用手工方式通过rpm包安装成功（版本18.09.9），Docker Engine现在被称作Docker-ce*

### 一、移除历史版本的docker
```shell
    sudo yum remove docker \
      docker-client \
      docker-client-latest \
      docker-common \
      docker-latest \
      docker-latest-logrotate \
      docker-logrotate \
      docker-engine
```

### 二、下载对应的rpm包（根据系统版本选择）
​    下载地址： https://download.docker.com/linux/centos/
​    需要下载4个包： 
​    docker-ce-18.09.9-3.el7.x86_64.rpm

​    docker-ce-cli-18.09.9-3.el7.x86_64.rpm

​	container-selinux-2.107-3.el7.noarch.rpm

​	containerd.io-1.2.2-3.el7.x86_64.rpm # 这个包是非必须的

### 三、安装rpm包

#### 1、更新系统的rpm包

```shell
#（yum --exclude=kernel* update  只升级依赖，不升级系统内核）
yum update
```

#### 2、安装container-selinux

```shell
sudo rpm -hiv container-selinux-2.107-3.el7.noarch.rpm --force --nodeps
```

#### 3、安装docker-ce-cli

```shell
sudo rpm -hiv docker-ce-cli-18.09.9-3.el7.x86_64.rpm
```

#### 4、安装docker-ce

```shell
sudo rpm -hiv docker-ce-18.09.9-3.el7.x86_64.rpm
```

中途如果遇到依赖包需要按照或者更新版本，按照提示执行相关rpm命令即可
例如：libcgroup系列包的安装：

```shell
 yum search libcgroup
 sudo yum install libcgroup-pam.i686
 sudo yum install libcgroup-pam.x86_64
 sudo yum install libcgroup-tools.x86_64
 sudo yum install libcgroup.x86_64
 sudo yum install libcgroup-devel.x86_64
```

*查看多个版本的包*

```shell
yum list libcgroup —showduplicates | sort -r
```

### 四、创建docker运行目录

docker运行时默认的目录是/var/lib/docker，此目录下会存放docker大部分数据，因此需要较大空间。

可以通过以下两种方式修改

#### 1、通过软链接的方式

```shell
mkdir -p /data/docker_base
ln -s /data/docker_base /var/lib/docker
```

 #### 2、通过修改配置文件方式

```shell
vim /etc/docker/daemon.json
{
“graph”: “/data/docker_base"
}
```

### 五、将普通用户添加到docker组
​    cat /etc/group | grep docker

​    如果没有docker组，先执行添加： sodu groupadd docker

​    添加普通用户到docker组： sudo gpasswd -a ocm docker

​    更新用户组： newgrp docker

### 六、启动docker

#### 1、设置docker镜像拉取地址（避免拉取慢的问题）

​        阿里云加速地址查看： https://cr.console.aliyun.com/undefined/instances/mirrors

​        vim /etc/docker/daemon.json

​        {"registry-mirrors":["https://xxxxxxx.mirror.aliyuncs.com"]}

#### 2、启动服务

​        a、sudo service docker start

​        b、sudo systemctl start docker.service

#### 3、重启服务

​		systemctl daemon-reload  && systemctl restart docker

### 七、验证docker运行

​    sudo docker run hello-world

### 其他

#### 1、查看docker服务配置项
```shell
vim /usr/lib/systemd/system/docker.service
```

#### 2、配置镜像私服
```shell
{
        "registry-mirrors":["http://192.168.0.1:9089"],
        "insecure-registries":["192.168.0.1:9089", "192.168.0.1:9088"]
}
```

*"http://192.168.0.1:9089" 是nexus搭建的docker group库，包含了proxy库和hosted库*

*"192.168.0.1:9088” 是nexus搭建的docker hosted库*

#### 3、开启dockerd监听端口

开启tcp监听端口，允许客户端远程连接

##### a、sudo vim /usr/lib/systemd/system/docker.service

​    修改[Service]下的配置项 ExecStart

​    ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375  -H fd:// --containerd=/run/containerd/containerd.sock

​    (监听地址也可以设置为真实IP加端口，如-H tcp://192.168.0.2:2375)

##### b、重新加载service配置

```shell
sudo systemctl daemon-reload
```

##### C、重启dockerd服务

```shell
sudo service docker restart
```

