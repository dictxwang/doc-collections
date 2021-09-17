# Nginx常用转发配置

### 零、location配置方法说明
=  表示精确匹配

^~ 以某个常规字符串开头

~ 区分大小写的正则匹配

~* 不区分大小写的正则匹配

!~ 区分大小写不匹配

!~* 不区分大小写不匹配

/ 通用匹配（匹配特殊配置外的任何请求）

### 一、upstream的配置
#### 1、权重、备份与下线
```nginx
upstream upst_1{
	server 192.168.3.102:80 weight=1;
	server 192.168.3.103:80 weight=1 backup;
	server 192.168.3.104:80 weight=1 down;
}
```

#### 2、通过客户端ip做hash
```nginx
upstream upst_1{
	server 127.0.0.1:8080 ;
	server 127.0.0.1:9090 ;
	ip_hash;
}
```

#### 3、通过客户端ip做一致性hash
```nginx
upstream upst_1{
	hash $remote_addr consistent;
	server 192.168.1.1:8819;
	server 192.168.1.2:8819;
	server 192.168.1.3:8819;
}
```

#### 4、超时与错误
```nginx
upstream upst_1{
	server 192.168.2.1:9916 weight=5 max_fails=1 fail_timeout=10s;
	server 192.168.2.2:9916 weight=5 max_fails=1 fail_timeout=10s;
	server 192.168.2.3:9916 weight=5 max_fails=1 fail_timeout=10s;
}
```

### 二、四层代理配置实例
```nginx
stream {
    upstream upst_1 {
		##hash $remote_addr consistent;
		server 192.168.4.1:9916 weight=5 max_fails=1 fail_timeout=10s;
		server 192.168.4.2:9916 weight=5 max_fails=1 fail_timeout=10s;
    }

    server {
        listen 19504 so_keepalive=on;
        proxy_connect_timeout 30s;
        proxy_timeout 30s;
        proxy_pass upst_1;
        tcp_nodelay off;
    }
}
```

### 三、http代理转发与多路径配置
```nginx
server
{
    listen *:80;
    server_name     dictxwang.cc;
    error_log logs/cc.xxx.com_error.log error;
    charset  utf-8;

    location ~* ^/ccfile/(.*)$ {
        proxy_set_header Host file.cc.xxx.com;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        rewrite ^/cefile/(.*) /$1 break;  # rewrite 配置
        proxy_pass     http://file.cc.xxx.com;  # 直接转发到域名
    }

    location ~* ^/cc-upload/(.*)$ {
        #proxy_set_header Host file.cc.xxx.com;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass     http://192.168.0.1:8808/$1$is_args$query_string;  # 转发到java后端
    }

    location ~ ^/luckgw/(.*)$ {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        #设置跨域访问头（也可以在后端代码中设置）
        proxy_set_header 'Access-Control-Allow-Origin' '*';
        proxy_set_header 'Access-Control-Allow-Credentials' 'true';
        proxy_set_header 'Access-Control-Allow-Methods' 'POST, GET, OPTIONS';
        proxy_pass     http://lucky_gw/$1$is_args$args;  # 转发到upstream
    }

    ##websocket转发配置
    location /luckyws/edit-sync-server {
        proxy_pass     http://lucky_gw/ws/edit-sync-server$is_args$query_string;
        proxy_http_version      1.1;
        proxy_connect_timeout   4s;
        proxy_read_timeout      120s;
        proxy_send_timeout      12s;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass     http://web_cc_portal;
    }
}
```

### 四、几个基于特殊条件的rewrite
#### 0、rewrite几个参数说明

last 相当于apache里面的[L]标记，默认last，表示rewrite

break 本条规则匹配完成后，终止匹配，不再匹配后面的规则

redirect 返回302临时重定向，浏览器地址会显示跳转后的URL地址

permanent 返回301永久重定向，浏览器地址会显示跳转后的URL地址

#### 1、通过host判断

这种方式可以限制访问的域名，避免攻击者恶意将转发地址指向我的服务器

```nginx
if ($host != "www.allow.com") {
     rewrite ^/(.*)$ http://baidu.com
}
```

#### 2、通过userAgent判断

这种方式可以实现不同终端类型，跳转到不同的域名

```nginx
if ($http_user_agent ~* 'firefox') {
     rewrite ^/(.*)$ http://dictxwang.cc
}
```

#### 3、基于参数拆分的判断

特例：url多个参数名同名，通过if匹配先拆解参数

（如果是不同名的参数，可以使用arg_name 这样的方式获取参数name是参数的名称）

（$args 是所有参数 $is_args 相当于问号）

（rewrite最后的一个问号的作用是屏蔽源url自身的参数串）

```nginx
location ~ ^/z/Ask(New)?\.e(.*)$     {
     if ($args ~ ^sp=S(.*)&sp=(\d+)&sp=S([0-9a-z]*)(.*)$) {
          set $pq $1;
          set $psourceId $2;
          set $puid $3;
          set $pmore $4;
          rewrite ^(.*)$  http://xx.yy.com/cate/ask?q=$pq&sourceId=$psourceId&uid=$puid$pmore?;
     }
}
```

#### 4、基于负向零宽断言的判断

```nginx
rewrite ^/z/(?!Activity|expert|LoginState|LLog|Home|Ask)([a-zA-Z]*)(\.htm)(.*)$ http://dictxwang.cc/z/$1$2$3;
```

### 五、简单的文件服务转发

如果涉及到跨域，需要通过add_header设置跨域的header

add_header 'key' 'value' always;  # 如果不加always，当服务器返回状态码是4xx和5xx时将不会带这个header

add_header Real-Backend-Upstream $upstream_addr always;  # 如此配置将始终返回真实的后端地址

```nginx
server {
        listen  *:80;
        server_name img.oc.xxx-inc.com;

        access_log  logs/img.oc.xxx-inc.access.log main;
    
        location / {
            root   /data/backend_sf/;
            expires 1m;
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'POST, GET';
        }
    
        location /cdn {
            alias   /data/cdn;  # 使用alias不会将location追加到文件路径上
            expires 1m;
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'POST, GET';
        }
    
        location /file {
            root   /data/outer/;  # 使用root会将location追加到文件路径上
            expires 30d;
        }

}
```

### 六、按条件返回特定的header

```nginx
location / {
    client_max_body_size 500m;
    # 设置Content-Disposition，自动启用浏览器端的下载
    if ($arg_fname ~* \.(doc|docx|xlsx|xls|pptx|ppt|txt|pdf|zip|rar|txt|png|jpg|jpeg|gif|tiff|bmp)$) {
            add_header Content-Disposition "attachment;filename=$arg_fname";
    }
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://nginx_fdfs_file;
}
```

### 七、在location节点配置缓存

```nginx
proxy_cache_path /data/nginx/temp levels=1:2 keys_zone=cache:1m max_size=1g inactive=1m use_temp_path=off;
server {
    # 省略若干
    location ~* ^/word/(.*){
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache cache;
        proxy_cache_valid   any 1m;
        add_header  Nginx-Cache "$upstream_cache_status";
        proxy_pass http://file_fgcd_word/word/$1$is_args$args;
    }
}
```

### 八、访问权限控制
#### 1、deny和allow指令

```nginx
# 配置在server或者location节点
deny 10.128.0.0/16;  # 配置ip段
deny 10.129.0.0/16;
deny 10.161.0.0/16;

allow 10.129.181.138;  # 配置特定的单ip
allow 10.129.181.38;
allow all;  # 不在规则内的
```
#### 2、autoindex指令

```nginx
# 配置在server或location节点
autoindex on; # 默认为off
autoindex_exact_size off;  # 是否显示精确的文件大小（on时不按照xM xK展示）
autoindex_localtime on;  # 是否按24小时制展示时间
charset utf-8,gbk;  # 支持多种编码
expires 1m;  # 设置客户端文件过期时间
```

### 3、防盗链配置

根据referer来源判断

通过curl可以模拟  curl -e "x.oc.xxx-inc.com" https://kf.xxx.com/file/private/123.txt

```nginx
location ^~ /file/private/ {
    valid_referers none blocked *.oc.xxx.com;  # none允许不带referer请求，blocked允许referer不带协议(http:// or https://)开头
    if ($invalid_referer) {
        return 404;
    }

    alias    /data/file/private/;
    expires 1s;

}
```

### 九、map的使用

map的匹配同样支持正则表达式

#### 1、通过参数返回指定的header

map是在http内全局范围的，多个server可以复用

```nginx
map $arg_k1 $message {  # 根据参数k1指定message的值，其中default是默认值
        hello       123;
        world       456;
        default    789;
}
location / {
     add_header 'X-Real-Message' '${message}' always;
}
# 当参数k1=world时，将返回header值456
```

#### 2、通过header值指定对应的转发后端

```nginx
upstream  upst001{
    server 10.143.49.137:18871 weight=1;
}
map $http_ocmsgwapikey $backend_path {
     basicToolsetPressureTestApi.nodeTest     upst001/pressure/nod-test;
     basicFileAccessApi.makeWatermarkSafetyImage     upst001/file/access/watermark/safety-image;
     basicExternalToolApi.decryptWechatUrls     upst001/external/tool/decrypt/wechat-urls/payload;
}
server
{
    # 省略本例无关的若干配置
    location /api {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5;
        proxy_read_timeout 30;
        proxy_send_timeout 5;
        proxy_pass http://${backend_path}$is_args$args;
        add_header ocmSgwBackend $upstream_addr always;
    }
}
```

#### 3、通过路径指定转发后端映射

例如通过path配置转发映射

```nginx
map $api_name $message {
    hello	http://192.168.0.1:8080;
    world	http://192.168.0.2:8080;
}
location /api/(.*) {
    set $api_name $1;
    pass_proxy    $message;
}
```

