# Nginx编译安装与基础配置

### 一、用于7层转发服务器的安装

#### 1、安装依赖包

```shell
yum install pcre-devel.x86_64  # perl命令包，支持rewrite功能
yum install openssl-devel.x86_64  # 支持https配置
yum install zlib-devel.x86_64  # 支持压缩和解压功能
```

#### 2、编译nginx

```shell
   ./configure --prefix=/usr/local/nginx --with-pcre-jit --with-pcre --with-http_stub_status_module --with-openssl --with-http_ssl_module --with-openssl-opt=' enable-weak-ssl-ciphers enable-ssl3 enable-ssl3-method' --with-http_secure_link_module --with-debug --with-http_realip_module --with-http_v2_module --add-module=xxx --add-module=yyy
```

#### 3、安装nginx

```shell
 make && mak install
```

#### 4、配置log等目录

```shell
mkdir -p /data/nginx/logs
mkdir -p /data/nginx/core
chown -R dictxwang:users /data/nginx
ln -s /data/nginx/logs /usr/local/nginx/logs
```

### 二、用于4层转发服务器的安装

#### 1、安装依赖包

```shell
yum install pcre-devel.x86_64
yum install openssl-devel.x86_64
```

#### 2、编译nginx

```shell
   ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_stub_status_module --with-http_auth_request_module --with-threads --with-stream --with-stream_ssl_module --with-mail --with-mail_ssl_module --with-file-aio --with-http_v2_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic'
```

#### 3、安装nginx

```shell
make && make install
```

#### 4、配置logs等目录

```shell
mkdir -p /data/nginx/logs
mkdir -p /data/nginx/core
chown -R dictxwang:users /data/nginx
ln -s /data/nginx/logs /usr/local/nginx/logs
```

### 三、常用的基础配置

#### 1、核心配置

```nginx
user root root;
worker_processes auto; # auto表示和cpu核数一致，也可以配置具体数字
working_directory /data/nginx;
worker_rlimit_core 2G;

error_log logs/error.log error; # 全局错误日志配置
pid   sbin/nginx.pid;
worker_rlimit_nofile 51200; # 最大允许打开的文件数
```

#### 2、events配置

```nginx
events
{
  use epoll;  # 采用epoll模式
  worker_connections 51200;  # 允许最大的链接数
}
```

#### 3、http中客户端相关的配置

```nginx
http
{
  include  mime.types;
  default_type application/octet-stream;
  #realip相关配置
  set_real_ip_from  10.0.0.0/8;
  set_real_ip_from  192.168.0.0/16;
  real_ip_header   X-Real-IP;
  real_ip_recursive on;  # 开启时，将获取非受信任的ip
  
  include xnet.conf;
  server_names_hash_bucket_size 128;
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;
  client_max_body_size 20m;
}
```

#### 4、http中日志相关的配置

```nginx
http
{
  #日志格式
  log_format main '$remote_addr - $remote_user [$time_local] $request '
                   '"$status" $body_bytes_sent $request_time "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for" '
                   '"$http_x_real_ip"';
  log_format mini '$time_local $status $body_bytes_sent $request_time $upstream_cache_status $server_name';
  #全局日志
  access_log logs/access_log main;
  access_log logs/status_log mini;
}
```



#### 5、http中转发性能相关的配置

```nginx
http
{
  sendfile    on;  # 开启零拷贝
  tcp_nopush   on;
  tcp_nodelay on;
  
  #链接超时配置
  keepalive_timeout 30;
  send_timeout    300;
  #代理转发超时配置
  proxy_next_upstream error timeout;
  proxy_connect_timeout 5;
  proxy_read_timeout 10;

  #开启gzip
  gzip on;
  gzip_min_length 1000;
  gzip_comp_level 6;
  gzip_types    text/plain text/xml application/x-javascript text/css application/xml;
}
```



#### 6、http中fastcgi相关的配置

```nginx
http
{
  fastcgi_connect_timeout 300;
  fastcgi_send_timeout 300;
  fastcgi_read_timeout 300;
  fastcgi_buffer_size 64k;
  fastcgi_buffers 4 64k;
  fastcgi_busy_buffers_size 128k;
  fastcgi_temp_file_write_size 128k;
}
```



#### 7、http中缓存相关的配置

```nginx
http
{
  #转发缓存配置
  proxy_cache_path /dev/shm/nginx/cache2/ levels=2:2 keys_zone=cache1:154m max_size=1g inactive=24h ;
  proxy_cache_path /dev/shm/nginx/cache/ levels=2:2 keys_zone=cache2:308m max_size=3g inactive=72h ;
  proxy_temp_path /dev/shm/nginx/temp/;
  
  # fastcgi缓存配置
  fastcgi_cache_path /data/nginx/fastcgi_cache/ levels=2:2 keys_zone=fccache:128m max_size=4g inactive=72h ;
  fastcgi_temp_path /data/nginx/fastcgi_temp/;
  # 缓存配置也可以放在server节点下
}
```

#### 8、http中的server配置

```nginx
http
{
  #server通常分转发域名放在单独的文件中，并通过include指令引入
  include vhosts/*.conf;
}
```

### 四、常用的运维命令

#### 1、查看nginx版本和编译信息

nginx -v 简要显示版本信息

nginx -V 显示版本信息和安装时的参数

#### 2、常用启动退出命令

nginx -s stop 快速退出

nginx -s quit 优雅退出

nginx -s reload 更换配置，重启工作进程（这也是平滑重启nginx的一种方式，也可以使用kill -HUP `cat /var/run/nginx.pid`的方式）

nginx -s reopen 重新打开日志文件（或者执行 kill -USR1 `cat /usr/local/nginx/sbin/nginx.pid`）

nginx 直接启动nginx

#### 3、windows下的命令

start nginx 启动

tasklist /fi "imagename eq nginx.exe" 查看当前的nginx进程

### 五、文档资料

nginx说明文档

http://tengine.taobao.org/book/chapter_02.html

windows版本说明

http://nginx.org/cn/docs/windows.html

nginx内置变量说明

http://nginx.org/en/docs/http/ngx_http_core_module.html#variables