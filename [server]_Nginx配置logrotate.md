# Nginx日志轮转的配置

### 一、logrotate的配置

假设nginx的日志文件存放路径/data/nginx/logs/*.log，

假设logrotate的配置文件存放路径/data/oss/logrotate.conf。

logrotate.conf配置如下：

```shell
/data/nginx/logs/*.log {
    # 按天轮转
    daily
    # 保存7天的历史日志
    rotate 7
    # 开启日志压缩
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
    		# 重新加载nginx配置文件，打开新的日志文件
        /usr/local/nginx/sbin/nginx  -s reload
    endscript
}
```

重新加载nginx配置文件的另外两种实现方式：

```shell
kill -USR1 `cat /usr/local/nginx/sbin/nginx.pid` # 推荐使用这种方式
/usr/local/nginx/sbin/nginx  -s reopen
```

### 二、logrotate的执行

1、单次执行

```shell
/usr/sbin/logrotate -f /data/oss/logrotate.conf 
```

2、调试配置

```shell
/usr/sbin/logrotate -d -f /data/oss/logrotate.conf 
```

3、注意事项

logrotate.conf 配置文件的所有者必须和执行logrotate的用户一致。

### 三、logrotate的定时执行

配置crontab定时任务

```shell
# 每天00:00执行
0 0 * * * ( /usr/sbin/logrotate -f /data/oss/logrotate.conf )
```

