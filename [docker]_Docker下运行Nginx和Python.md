# Docker下运行Nginx和Python

本文介绍nginx和python在docker模式下的运行。其中python工程的运行分为直接运行和构建成镜像运行两种方式。

### 一、运行Nginx

#### 1、拉取镜像

docker pull nginx

#### 2、运行容器

##### a、默认配置运行

```shell
docker run --name nginx_qiang -d -p8080:80 nginx
# 参数说明
# --name 给容器一个唯一名字（在容器相关操作中可以代替containerId）
# -d 让容器在后台运行
# -p8080:80 将容器内的80端口桥接到宿主机的8080
```

##### b、映射目录、配置后运行

```shell
docker run --name nginx_qiang -d -p8080:80 -v /data/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf -v /data/docker/nginx/logs/:/var/log/nginx -v /data/docker/nginx/conf/conf.d/:/etc/nginx/conf.d -v /data/docker/nginx/html/:/usr/share/nginx/html nginx
# 通过将nginx配置目录和日志目录映射到宿主机，便于nginx的运维
```

#### 3、重启nginx

通常是在修改配置后执行，分为两边执行：

##### a、进入nginx容器

docker exec -it nginx_qiang(或者containerId) bash

##### b、执行reload

nginx -s reload



### 二、运行python

首先假设宿主机上有如下路径： 

*/data/docker/python*

#### 1、编写python脚本

##### a、main.py

```python
#!/usr/bin/python

print("Hello,World!");
```

##### b、requirements.txt

```shell
# 没有遗漏置空即可
```

#### 2、直接通过docker执行

```shell
docker run -v /data/docker/python:/usr/src/myapp -w /usr/src/myapp python python main.py
# 参数说明
# -v 挂载宿主机路径到docker的路径
# -w 指定脚本执行路径
# 第一个python 指使用python镜像
# 第二个python 指执行脚本的命令
```

```shell
docker run -v /data/docker/python/logs:/usr/src/myapp/logs -v /data/docker/python/:/usr/src/myapp -w /usr/src/myapp python python main.py
# 路径可以挂载多个，比如将容器内的日志重定向输出到宿主机，便于日志查看
```

#### 3、通过docker镜像执行

##### a、编写Dockerfile

```dockerfile
# basic image
FROM python

# set workfolder
WORKDIR /usr/src/pythonapp

# copy project files to workfolder
ADD . /usr/src/pythonapp

# install require modules
RUN pip install -r requirements.txt

# launch the app
CMD ["python", "main.py"]
```

##### b、构建镜像

```shell
docker build -t qiangwang/hello-python .
```

##### c、运行容器

```shell
docker run qiangwang/hello-python
```