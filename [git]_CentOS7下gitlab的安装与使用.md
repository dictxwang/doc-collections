# CentOS7下gitlab的安装与使用

### 一、安装

##### 1、下载安装包

https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-14.1.1-ce.0.el7.x86_64.rpm

##### 2、执行安装

rpm -hiv gitlab-ce-14.1.1-ce.0.el7.x86_64.rpm

##### 3、调整运行路径

mkdir /data/gitlab

ln -s /data/gitlab/ /var/opt/gitlab

### 二、配置与启动

##### 0、修改配置

​    sudo vim /etc/gitlab/gitlab.rb

##### 1、检查配置及依赖

​    gitlab-rake gitlab:check

##### 2、更新配置

​    gitlab-ctl reconfigure

##### 3、重启服务

​    gitlab-ctl restart

### 三、后台管理

##### 1、进入控制台

​    gitlab-rails console

​	a、检查邮件服务配置是否成功

Notify.test_email('wangqiang0208@gmail.com', 'title is gitlabTest', 'content:is just test').deliver_now

​	b、修改root用户密码

​		 *user=User.where(id: 1).first*

​		*user.password="12345678"*

​		*user.password_confirmation="12345678"*

​		*user.save!*

​		*quit*

##### 2、进入数据库控制台

​    gitlab-rails dbconsole

##### 3、查看日志

​    gitlab-ctl tail # 全局日志

​	gitlab-ctl tail nginx/gitlab_acces.log # nginx日志

​	gitlab-ctl tail postgresql # postgresql日志

##### 4、状态查询

​    gitlab-ctl status

###### A、gitlab命令所在位置

​    cd /opt/gitlab/embedded/bin

###### B、gitlab部署路径

​    cd /var/opt/gitlab/

###### C、gitlab rails运行路径

​    cd /var/opt/gitlab/gitlab-rails/etc

### 附：常用配置

##### i、配置邮件服务

​    *gitlab_rails['smtp_enable'] = true*

​	*gitlab_rails['smtp_address'] = "mail.163.com"*

​	*gitlab_rails['smtp_port'] = 465*

​	*gitlab_rails['smtp_user_name'] = "dictwang"*

​	*gitlab_rails['smtp_password'] = "xxxxyyyy"*

​	*gitlab_rails['smtp_domain'] = "163.com"*

​	*gitlab_rails['smtp_authentication'] = "login"*

​	*gitlab_rails['smtp_enable_starttls_auto'] = true*

​	*gitlab_rails['smtp_tls'] = false*

​	*gitlab_rails['smtp_pool'] = false*

​	*gitlab_rails['smtp_openssl_verify_mode'] = 'none'*

​	*gitlab_rails['gitlab_email_enabled'] = true*

​	*gitlab_rails['gitlab_email_from'] = 'dictwang@163.com'*

​	*gitlab_rails['gitlab_email_display_name'] = 'dictx4gitlab'*

​	*gitlab_rails['gitlab_email_reply_to'] = 'dict@s163.com'*

##### ii、配置mysql

​    *从GitLab 12.1开始，gitlab只支持postgresql，12.1之前的版本如下配置：

​	*postgresql['enable'] = false*

​	*gitlab_rails['db_adapter'] = 'mysql2'*

​	*gitlab_rails['db_encoding'] = 'utf8'*

​	*gitlab_rails['db_host'] = '127.0.0.1'*

​	*gitlab_rails['db_port'] = '3306'*

​	*gitlab_rails['db_username'] = 'gitlab'*

​	*gitlab_rails['db_password'] = 'gitlab'*

​	*gitlab_rails['db_host'] = "127.0.0.1"*
