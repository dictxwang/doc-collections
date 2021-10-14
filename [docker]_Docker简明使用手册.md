# Docker简明使用手册

本文基于docker实践总结而来，仅记录常用的一些操作。更多的docker使用方法，参见docker官方说明文档：

https://docs.docker.com/engine/reference/run/

### 一、基本操作
#### 1、查看image、container

docker images	//查看所有的image

docker ps	//查看运行的container

docker container ls -a	//查看所有的container，包括已经停止的

docker info	//docker信息

docker version	//docker版本

docker history iamgename	//查看镜像历史信息

#### 2、对image或container的运行操作

docker rmi -f imageId	//-f是强制删除

docker rmi -f $(docker images -q)	//删除所有的images

docker stop containerId	//停止指定的container

docker stop $(docker ps -aq)	//停止所有的container

docker start containerId	//运行指定的container

docker restart containerId	//重启指定的container

docker inspect containerId/imageId	//查看容器信息

docker top containerId	//查看容器内运行的进程

### 二、文件操作

#### 1、文件复制（容器 -> 宿主机）

docker cp ${container}:/home/dictwang/ localdir/	// container可以是名称或者id

#### 2、文件复制（宿主机 -> 容器）

docker cp f.txt ${container}:/home/dictwang/

#### 3、查看日志

docker logs ${containername}	//查看容器日志

### 三、容器操作

#### 1、运行容器中的软件

*以操作redis为例：*

docker exec -it ${containerId} redis-cli

#### 2、在容器中执行shell（以默认用户）

docker exec -it ${containerId/name} bash

#### 3、在容器中执行shell（以指定用户）

docker exec -it --user root ${containerName} bash

### 四、镜像操作

#### 1、保存镜像

docker save savename imagename

#### 2、加载镜像

方式a、docker load --input savename

方式b、docker load < savename

#### 3、搜索镜像

docker search xxx

#### 4、构建镜像

```shell
docker build
docker build -f /p1/c2/Dockerfile
docker build -t wangqiang/hello-001 .   # 指定镜像名称
docker build -t wangqiang/hello:0.1.1 . # 指定镜像名称及版本（未指定版本时，默认为latest）
```

#### 5、发布镜像

```shell
docker tag dictwang/hello 10.143.55.43:9088/dictwang/hello:0.1.0 # 发布前需要先打tag，ip:port指定私服位置，:0.1.0是指定tag号
docker login 10.143.55.43:9088 # 登录私服
docker push 10.143.55.43:9088/dictwang/hello:0.1.0 # 推送发布到私服上
```

### 附：其他操作

#### 附1：安装docker-machine

下载路劲：https://github.com/docker/machine/releases/tag/v0.16.2

##### A、windows平台

```shell
if [[ ! -d "$HOME/bin" ]]; then mkdir -p "$HOME/bin"; fi && \curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-Windows-x86_64.exe > "$HOME/bin/docker-machine.exe" && \chmod +x "$HOME/bin/docker-machine.exe"
// 可以将安装后exe文件拷贝到C:\Program Files\Docker\Docker\resources\bin，便于和docker、docker-compose命令统一管理
```

##### B、MacOS平台

```shell
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/usr/local/bin/docker-machine && \ chmod +x /usr/local/bin/docker-machine
```

##### C、Linux平台

```shell
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine && chmod +x /tmp/docker-machine && sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

####  附2、安装docker-compose

docker-compose是docker的命令行工具，用于操作（打包、运行等）由多个容器组合而成的应用

##### A、命令安装

```shell
1、curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
2、chmod +x /usr/local/bin/docker-compose
```

##### B、基本操作 

```shell
创建 Dockerfile
定义 docker-compose.yml
构建 docker-compose build
运行 docker-compose up -d (-d是在后台运行)
```
