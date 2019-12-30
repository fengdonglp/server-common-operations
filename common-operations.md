# centos 服务器常用操作

## systemctl

``` bash
# 启动服务
systemctl start xxx.service

# 停止服务
systemctl stop xxx.service

# 重新启动服务
systemctl restart xxx.service

# 停止开机自启动
systemctl disable xxxx.service

# 查看服务当前状态
systemctl status xxx.service

# 查看所有已启动的服务
systemctl list-units --type=service

# 查看已启动的服务列表
systemctl list-unit-files|grep enabled
```

## 阿里云防火墙与安全组

阿里云ECS默认入口规则会限制所有入口ip的限制，即便服务器防火墙开放了端口。

所以我这里将服务器防火墙关掉，统一走安全组策略。

### 防火墙相关操作

``` bash
# 启动防火墙
systemctl start firewalld

# 开机自启
systemctl enable firewalld.service 

# 重启
systemctl restart  firewalld
# 或者 这种无需断开连接
firewall-cmd --reload
# 或者 类似重启
firewall-cmd --complete-reload

# 停止
systemctl stop firewalld

# 永久开放80端口/指定端口， 返回success表明成功
# --zone #作用域
# --add-port=80/tcp #添加端口，格式为：端口/通讯协议
# --permanent #永久生效，没有此参数重启后失效
firewall-cmd --zone=public --add-port=80/tcp --permanent 

# 查询指定端口是否开启
firewall-cmd --query-port=80/tcp

# 查询有哪些端口是开启的
firewall-cmd --list-port

# 查看状态
firewall-cmd --state
```

## nginx

``` bash
# EPEL (Extra Packages for Enterprise Linux)是基于Fedora的一个项目，为“红帽系”的操作系统提供额外的软件包，适用于RHEL、CentOS和Scientific Linux.包含了很多未带的软件
yum install -y epel-release

yum -y update

yum install -y nginx

# 安装成功后，默认的网站目录为： /usr/share/nginx/html

# 默认的配置文件为：/etc/nginx/nginx.conf

# 自定义配置文件目录为: /etc/nginx/conf.d/

# 如果是阿里云ECS，则需要开启http和https的安全组规则，否则在防火墙开启：
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
firewall-cmd --reload
```

## mysql（^8.0）

### 安装

``` bash
# 删除默认的mariadb
yum remove mariadb.x86_64

# 在官网找到mysql源下载地址：https://dev.mysql.com/downloads/repo/yum/ ， 注意要找linux7的下载地址
# 例如：https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm
wget https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm

yum -y install mysql80-community-release-el7-3.noarch.rpm

# 查看是否安装成功
yum search mysql

# 安装
yum -y install mysql-community-server​

# 启动
systemctl start mysqld.service

# 开机启动
systemctl enable mysqld
```

### 重置密码

``` bash
# 关闭mysql服务
systemctl stop mysqld

# 开启免密登陆
# 打开配置文件
vi /etc/my.cnf
# 在文件内添加免密配置
skip-grant-tables
# 保存后退出重启mysql服务
systemctl start mysqld

# 进入mysql
mysql

# 刷新权限
flush privileges;

# 修改root账户密码
use mysql;
UPDATE user SET authentication_string=PASSWORD('password') where User='root';

# 刷新权限
flush privileges;

# 退出
exit

# 再从配置文件里将免密登陆注释掉保存重启后登陆
mysql -u root -p
```

### 配置外部访问（navicat连接）

``` bash
# 首先阿里云安全组/防火墙开启 mysql 端口
# 查看 mysql 端口，登陆进 mysql 后执行
show global variables like 'port';

# 创建外部访问新用户（由于root是针对localhost的，所以就算设置了也无法外部访问）
CREATE USER 'super'@'%' IDENTIFIED BY 'password';

# 刷新权限
flush privileges;

# 添加权限（权限操作可参考：https://blog.csdn.net/xiaolong_4_2/article/details/85281918）
grant all on *.* to 'super'@'%';

# 添加加密插件认证密码(默认是 mysql_native_password)
ALTER USER 'super'@'%' IDENTIFIED WITH mysql_native_password BY 'password';

# 刷新权限
flush privileges;
```

### 密码强度（待补充）

## redis

``` bash
# 安装
yum install redis

# 启动
systemctl start redis

# 修改配置支持远程连接
vi /etc/redis.conf
# 注释掉 bind 127.0.0.1
# 添加密码认证 取消注释，并添加密码
requirepass password
# 保存

# 重启
systemctl restart redis
```

## mongodb

``` bash
# 参考官网yum源 https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/, 并添加源文件
vim /etc/yum.repos.d/mongodb-org-4.2.repo

# 添加源信息
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc

# 安装
yum install mongodb-org
# 上面这种命令默认会安装 mongodb-org-server mongodb-org-mongos mongodb-org-shell mongodb-org-tools

# 修改配置文件开启访问控制
vi /etc/mongod.conf

# 开启 security
security:
  authorization: enabled

# 修改绑定ip开启外部访问
bindIp: 0.0.0.0

# 增加MongoDB的打开文件和进程限制
echo "mongod     soft    nofiles   64000" >> /etc/security/limits.conf
echo "mongod     soft    nproc     64000" >> /etc/security/limits.conf

# 启动
systemctl start mongod

# 创建数据库用户
mongo
use admin
db.createUser({user: "mongo-admin", pwd: "password", roles:[{role: "userAdminAnyDatabase", db: "admin"}]})
# 上面创建了mongo-admin用户，并授予权限

# 退出
quit()

# 检测创建的用户，注意：mongo-admin有创建用户的权限
mongo -u mongo-admin -p --authenticationDatabase admin
```

## nvm（node版本管理工具）

``` bash
# 官方安装方式(由于国内的原因，可能会出现拒绝连接的情况)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash

# 将以下脚本添加到.bashrc文件中（默认安装成功后会自动添加，如果没有请手动添加）
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm

# 确保开机可用可在 .bash_profile 中添加一下脚本
source .bashrc

# 用git安装
git clone https://github.com/creationix/nvm.git ~/.nvm && cd ~/.nvm && git checkout `git describe --abbrev=0 --tags`

# 安装
. nvm.sh

# 需要将第一种方法添加的脚本都添加进去

```