# Nginx配置SSL自签名证书

*记录通过openssl创建SSL自签名证书，以及nginx通过自签名证书实现https转发的配置。*

### 一、创建SSL自签名证书
#### 1、创建CA证书

##### a、创建私钥

```shell
openssl genrsa -out ca-key.pem 1024
```

##### b、创建csr证书请求

```shell
openssl req -new -key ca-key.pem -out ca-req.csr -subj "/C=CN/ST=BJ/L=BJ/O=sg_ocm_1/OU=sg_ocm_1a/CN=OCM_CD"
    #C 国家
    #ST 省份
    #L 城市
    #O 证书组织
    #OU 组织单元
    #CN 通用名称（Common Name）
```

##### c、生成crt证书

```shell
openssl x509 -req -in ca-req.csr -out ca-cert.pem -signkey ca-key.pem -days 3650
```

#### 2、创建服务端证书

##### a、创建服务器端私钥

```shell
openssl genrsa -out server-key.pem 1024
```

##### b、创建csr证书（两种方式）

```shell
    openssl req -new -out server-req.csr -key server-key.pem -subj "/C=CN/ST=BJ/L=BJ/O=sg_ocm_2/OU=sg_ocm_2a/CN=test.wangqiang.cc"
    #../conf/server.csr.cnf 内容见下文的“其他事项”
    openssl req -new -sha256 -out server-req.csr -key server-key.pem -config ../conf/server.csr.cnf
```

##### c、生成crt证书（两种方式）

```shell
openssl x509 -req -in server-req.csr -out server-cert.pem -signkey server-key.pem -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -days 3650
    #../conf/v3.ext 内容见下文的“其他事项”
    openssl x509 -req -in server-req.csr -out server-cert.pem -signkey server-key.pem -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -days 3650 -sha256 -extfile ../conf/v3.ext
```

*完成上面两步操作后，会得到以下7个文件*

```shell
    ca-cert.pem
    ca-cert.srl
    ca-key.pem
    ca-req.csr
    server-cert.pem
    server-key.pem
    server-req.csr
```

#### 3、验证证书

```shell
openssl verify -CAfile ca-cert.pem  server-cert.pem
```

#### 4、手动安装客户端证书

*ssl证书在用户访问站点时，会自动安装；当然也可以手动安装。*

##### a、导出客户端证书

```
openssl pkcs12 -export -out ca.p12 -inkey ca-key.pem -in ca-cert.pem
```

##### b、安装证书

​    双击证书->输入密码->选择"受信任的根证书颁发机构" -> 确定完成安装

### 二、配置Nginx的https转发

#### 1、拷贝ssl文件

​    将上面生成的7个文件放置到 nginx的配置路径 conf/ssl/下

#### 2、配置转发

```nginx
server {
		listen       80;    # 支持http访问
		listen      443 ssl;    # 支持https访问
		server_name  test.wangqiang.cc;
		#charset utf-8;

		ssl_certificate     /usr/local/nginx/conf/ssl/server-cert.pem;
    	ssl_certificate_key /usr/local/nginx/conf/ssl/server-key.pem;
    
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
    
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    
        #location配置和http时的配置一致
    	#同样支持websocket通过wss访问
}
```

### 三、其他事项

#### 1、两个SSL扩展配置文件

 ##### a、../conf/server.csr.cnf

```shell
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=CN
ST=Si Chuan
L=Cheng Du
O=Sogou.inc
OU=OCM
emailAddress=wangqiang0208@gmail.com
CN = *.wangqiang.cc
```

##### b、../conf/v3.ext

```shell
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = test.wangqiang.cc
DNS.2 = test.wangqiang03.cc
DNS.2 = test.wangqiang.cn
```

#### 2、不同主域名使用同一个SSL证书

​    通过SAN(Subject Alternative Names)可以实现不同主域名的配置

​    在v3.ext 文件中设置

```shell
[alt_names]
DNS.1 = test.wangqiang.cc
DNS.2 = test.wangqiang03.cc
DNS.2 = test.wangqiang.cn
```

#### 3、Haproxy配置https的透传转发

​    这里的透传转发是指SSL证书不安装在haproxy上，haproxy仅做请求转发

```shell
#第一组前后端配置
frontend    wq-servers
bind    192.168.1.1:443
mode    tcp
log     global
use_backend wq-server-https

backend     wq-server-https
mode    tcp
balance roundrobin
server s1   192.168.1.3:443   weight 1 maxconn 10000 check inter 10s

#第二组前后端配置
frontend    wq-portals
bind    192.168.1.1:443
mode    tcp
log     global
use_backend wq-portal-https

backend     wq-portal-https
mode    tcp
balance roundrobin
server s1   192.168.1.3:443   weight 1 maxconn 10000 check inter 10s
```

