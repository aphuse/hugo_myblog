+++
Categories = ["数据库", "DB2"]
Description = "介绍安装DB2后，如何进行客户端配置连接DB2数据库服务"
Tags = ["数据库", "DB2"]
date = "2015-12-07T14:59:54+08:00"
title = "DB2数据库客户端配置连接"

+++

&emsp;&emsp;安装好DB2数据库之后，如果想要进行客户端的连接，首先要在服务端做一些配置，才能顺利连接得上数据库服务。下面我们来介绍一下如何进行配置。

#### 系统环境
- 操作系统：linux/unix
- DB2: v9.5
- DB2实例名：db2
- DB2服务器IP： 192.168.10.203

<!--more-->

#### 服务端配置

- 查看一下服务名
```shell
$ db2 get dbm cfg |grep SVC
 TCP/IP Service name                          (SVCENAME) = DB2_db2
```

- 设置指定实例的服务名
```shell
$ db2 update dbm cfg using SVCENAME DB2_db2
```

- 查看设置是否生效
```shell
$ db2 get dbm cfg |grep SVC
 TCP/IP Service name                          (SVCENAME) = DB2_db2
```

- 修改/etc/services文件
```shell
$ cat /etc/services|grep DB2_db2
DB2_db2         50000/tcp
DB2_db2_1       50001/tcp
DB2_db2_2       50002/tcp
DB2_db2_END     50003/tcp
```

- 设置通讯协议为tcpip
```shell
$ db2set DB2COMM=tcpip
$ db2set
DB2COMM=TCPIP
```

- 重启db2实例，就可以进行客户端设置连接了
```shell
$ db2stop
$ db2start
```


#### 客户端配置

- 建节点目录
```shell
$ db2 catalog TCPIP node n1 remote 192.168.10.203 server 50000
$ db2 terminate
```

- 建数据库编目
```shell
$ db2 catalog database demo as demo_as at node n1 authentication server
$ db2 terminate
```

- 连接数据库
```shell
$ db2 connect to demo_as user db2 using passwd
```

- 删除数据库编码和节点命令
```shell
$ db2 uncatalog db demo_as
$ db2 uncatalog node n1
```

- 查看节点信息
```shell
$ db2 list node directory
```
