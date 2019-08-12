<!-- TOC -->

- [1. 引言](#1-%e5%bc%95%e8%a8%80)
- [2. 搭建环境](#2-%e6%90%ad%e5%bb%ba%e7%8e%af%e5%a2%83)
- [3. MySQL的配置文件](#3-mysql%e7%9a%84%e9%85%8d%e7%bd%ae%e6%96%87%e4%bb%b6)
- [4. 创建3307端口号步骤](#4-%e5%88%9b%e5%bb%ba3307%e7%ab%af%e5%8f%a3%e5%8f%b7%e6%ad%a5%e9%aa%a4)
- [5. MySQL的两种连接方式](#5-mysql%e7%9a%84%e4%b8%a4%e7%a7%8d%e8%bf%9e%e6%8e%a5%e6%96%b9%e5%bc%8f)
- [6. mysql、mysqld、mysqladmin](#6-mysqlmysqldmysqladmin)
  - [6.1. mysql](#61-mysql)
  - [6.2. mysqld](#62-mysqld)
  - [6.3. mysqladmin](#63-mysqladmin)

<!-- /TOC -->
# 1. 引言
最近在研究数据库时想搭建一个mysql集群，奈何自己只有一台电脑，所以借用阿里云ECS服务器为mysql创建多个实例端口，模拟搭建mysql集群。

第一步先搭建mysql多实例，网上的博客要不是抄过来抄过去就是没有指明搭建环境或者不是自己想要的，而且版本不一致，自己走了很多坑，通过翻阅无数遍大佬们的博客之后自己成功为mysql创建了3307端口号，这里记录一下自己搭建mysql多实例的过程。

# 2. 搭建环境
**代码未动，环境先行**

系统环境：阿里云ECS，镜像为Ubuntu 18.04.2 LTS

mysql版本：ubuntu自带的版本5.7.27-0ubuntu0.18.04.1

本地操作为ubuntu系统远程连接ECS

# 3. MySQL的配置文件

创建多实例的方式：<br>
一种是采用官方方式：利用mysqld_multi工具进行多实例的实现，但是自己采用多个配置文件的方式进行实现，为了后面搭建集群，也推荐使用这种方式，关于这两种方式的差异网上教程非常多。

创建配置文件之前先研究一下mysql的配置文件，只有好的过程才会有好的结果。


mysql5.7版本下配置文件主要在/etc/mysql/my.cnf中，在此文件中又加载了conf.d和mysql.conf.d这两个文件。

其实在my.cnf文件中引用了两个软连接，软连接其实类似于windows系统中的快捷方式。

配置文件的主要参数：
```conf
[mysqld_safe]
socket          = /var/run/mysqld/mysqld.sock   #socket通信设置
nice            = 0

[mysqld]
user            = mysql
pid-file        = /var/run/mysqld/mysqld.pid           # 进程pid文件路径
socket          = /var/run/mysqld/mysqld.sock       #socket通信设置
port            = 3306              #端口号
basedir         = /usr            #mysql的安装路径 
datadir         = /var/lib/mysql                   #数据文件的所在路径
tmpdir          = /tmp                                      #临时文件的保存路径
lc-messages-dir = /usr/share/mysql   
skip-external-locking

bind-address            = 127.0.0.1     # 设置此ip可以使其他主机访问数据
log_error = /var/log/mysql/error.log        # 错误日志文件的路径

```

# 4. 创建3307端口号步骤
* 什么是mysql多实例？

mysql多实例就是在一台机器上开启多个不同的端口号，运行多个mysql服务进程。通过不同的socket监听不同的服务端口号来提供各自的服务。
(这里关于socket的知识在后面回顾)


**使用不同的配置文件启动不同的进程来实现多实例**，在同一台机器上创建多个实例的条件：<br>（1）配置文件安装路径不能相同<br>
（2）数据库目录不能相同<br>
（3）启动脚本不能同名<br>
（4）端口不能相同<br>
（4）socket文件的生成路径不能相同

所以根据这些条件来配置多实例：

1. 创建mysql数据文件目录并修改权限

```
在/var/lib 目录下

mkdir 3307_mysql
chown -R mysql:mysql 3307_mysql
```

2. 创建配置文件

直接复制3306端口默认的配置文件
```
在/etc/mysql目录下：
cp -r conf.d  conf3307.d
cp my.cnf my3307.cnf
cp -r mysql.conf.d mysql3307.conf.d
```

3. 修改配置文件
```
vi my3307.cnf
修改软连接路径：
!includedir /etc/mysql/conf3307.d/
!includedir /etc/mysql/mysql3307.conf.d/
```

4. 进入mysql3307.conf.d目录修改mysqld.cnf文件
```
[mysqld_safe]
socket          = /var/lib/3307_mysql/mysqldsafe.sock
#nice           = 0

[mysqld]
user            = mysql
pid-file        = /var/lib/3307_mysql/mysqld.pid
socket          = /var/lib/3307_mysql/mysqld.sock
port            = 3307
basedir         = /usr
datadir         = /var/lib/3307_mysql
tmpdir          = /tmp
lc-messages-dir = /usr/share/mysql
skip-external-locking

# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 0.0.0.0

log_error = /var/lib/3307_mysql/error.log

```
注意socket的配置文件路径：socket    = /var/lib/3307_mysql/mysqldsafe.sock,这个路径在后面会用到自己在这里踩了很多坑

5. 添加读写权限
```
修改/etc/apparmor.d/usr.sbin.mysqld文件
添加权限

/var/lib/3308_mysql/ r,
/var/lib/3308_mysql/** rwk,

然后重新启动
service  apparmor reload
```

6. 初始化配置文件
```
初始化实例
mysqld --initialize-insecure --datadir=/var/lib/3307_mysql --user=mysql　　

其中--initialize-insecure 为创建时不带密码
```

7.实例的启动与关闭

```
启动实例
mysqld_safe --defaults-file=/etc/mysql/my3307.cnf

关闭实例
mysqladmin -uroot -S /var/lib/3308_mysql/mysqldsafe.sock shutdown

使用密码进行登陆
mysql -u root -p -S /var/lib/3307_mysql/mysqldsafe.sock -P 3307
``` 
完成以上步骤就可使用3307端口号了

在启动mysql服务后，会启动的默认的端口号3306，如果要使用其他端口号则需要手动进行启动

# 5. MySQL的两种连接方式
一种是TCP/IP连接方式，通常我们用的mysql -u root -p<br>
另一种是socket连接方式，指定端口连接mysql：mysql -u root -p -S /var/lib/3307_mysql/mysqldsafe.sock -P 3307



* 关于套接字
计算机网络中最常用的是5层协议：物理层、数据链路层、网络层、传输层、应用层
来自socket百度百科
传输层实现的是端到端的通信，端到端通信通过


# 6. mysql、mysqld、mysqladmin

关于MySQL数据库的几个命令自己一直是一知半解，但是在平时用的比较多，此出借官网来彻底弄懂MySQL程序中的这几个命令的关系。目前只了解这些，具体运用在实际中学习。以下部分很多是来自官网，比较可靠。


## 6.1. mysql
mysql是一个客户端命令行工具，是一个带有输入行编辑功能的简单SQL shell。这是最常用的命令行，他有很多参数和选项，具体要参考官网。

## 6.2. mysqld

mysqld也称为MySQL Server,是完成MySQL安装中大部分工作的主程序。要使用客户端程序，必须要运行mysqld，因为客户端程序要通过连接MySQL服务器进行数据库的操作。

## 6.3. mysqladmin
mysqladmin是一个管理MySQL服务器的客户端，可以使用他来检查服务器的配置和当前状态，以及创建和删除数据库等。
