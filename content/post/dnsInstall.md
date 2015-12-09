+++
Categories = ["linux", "DNS服务"]
Description = "介绍如何在centos6.7下架设DNS服务"
Tags = ["DNS服务", "linux"]
date = "2015-12-01T15:24:43-18:00"
title = "centos6.7下架设DNS服务"

+++

#### 系统环境
  - 系统发行版：CentOS release 6.7 (Final)
  - 内核版本：GNU/Linux 2.6.32-573.8.1.el6.x86_64

#### 安装bind
```shell
yum install bind bind-chroot bind-utils
```
#### 配置bind
&emsp;&emsp;相关配置文件可以参考/usr/share/doc/bind-*/sample/里面的示例配置文件
<!--more-->
- 编辑/etc/named.conf文件

```shell
//file:/etc/named.conf
acl corpnets {192.168.0.0/16;}; // 授权核心网络
options {
	listen-on port 53 { any; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { corpnets; };
	allow-query-cache { corpnets; };
	recursion yes;
	forwarders { 114.114.114.114; };
	dnssec-enable yes;
	dnssec-validation no;
	dnssec-lookaside auto;
	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";
	managed-keys-directory "/var/named/dynamic";
};
//日志配置
logging {
        channel default_debug {
                file "data/named.run";
                severity warning;
        };
};
// 使用view，所有的zone都要配置在view中
view nets192 {
	match-clients 	   { corpnets;};
	match-destinations { any; };
	recursion yes;

	zone "." IN {
        	type hint;
        	file "named.ca";
	};
	include "/etc/named.rfc1912.zones";
};

```

- 编辑/etc/named.rfc1912.zones文件

```shell
//file:/etc/named.rfc1912.zones
//配置本地域名解析，此配置在运行named服务后会自动生成，可以不用修改
zone "localhost.localdomain" IN {
	type master;
	file "named.localhost";
	allow-update { none; };
};
zone "localhost" IN {
	type master;
	file "named.localhost";
	allow-update { none; };
};
zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" IN {
	type master;
	file "named.loopback";
	allow-update { none; };
};
zone "1.0.0.127.in-addr.arpa" IN {
	type master;
	file "named.loopback";
	allow-update { none; };
};
zone "0.in-addr.arpa" IN {
	type master;
	file "named.empty";
	allow-update { none; };
};
//配置域名正向解析和反向解析，这里的配置需要自己手动添加
zone "test.cn" IN {
	type master;
	file "192.test.cn.zone";
	allow-update { none; };
};
zone "1.168.192.in-addr.arpa" IN {
        type master;
        file "192.test.cn.local";
        allow-update { none; };
};
```

- 编辑文件/var/named/192.test.cn.zone

```shell
;file:/var/named/192.test.cn.zone
;test.cn
$TTL	3600
@ IN  SOA localhost root (
					201512011402 ; serial (d. adams)
					3H           ; refresh
					15M          ; retry
					1W           ; expiry
					1D )         ; minimum
      IN NS   localhost.
@     IN A    192.168.1.224
dns   IN A    192.168.1.224
www   IN A    192.168.1.4
ftp   IN A    192.168.1.5

```

- 编辑文件/var/named/192.test.cn.local

```shell
;file:/var/named/192.test.cn.local
;test.cn
$TTL	3600
@ IN  SOA  localhost. root.localhost.  (
           201512011402 ; Serial
           1M           ; Refresh
           15M          ; Retry
           1W           ; Expire
           1h )         ; Minimum
        IN      NS      localhost.
224     IN      PTR     dns.test.cn.
4       IN      PTR     www.test.cn.
5       IN      PTR     ftp.test.cn.

```

#### 开放named服务端口
&emsp;&emsp;named服务使用的端口号是53，配置防火墙开放53端口。执行下面代码打开53端口并保存设定。
```shell
iptable -A INPUT -m state --state NEW -m tcp -p tcp --dport 53 -j ACCEPT
iptable -A INPUT -m state --state NEW -m udp -p udp --dport 53 -j ACCEPT
iptables-save
```

#### 启动named服务,测试
- 启动服务

```shell
service named start
```
- 测试

```shell
dig @192.168.1.224 www.test.cn
```

#### 配置named服务开机启动
```shell
chkconfig named on
```
--------

#### 问题及解决方案

---------
##### 问题一

&emsp;&emsp;error (network unreachable) resolving 'www.linuser.com.dlv.isc.org/DLV/IN': 2001:502:ad09::23#53
类似这样的报错是由于开启了IPv6，关闭dns IPv6即可。

###### 解决方法

1) 在文件/etc/sysconfig/named后面添加一行
```shell
  OPTIONS="-4"
```
2) 在/etc/named.conf中注释掉下面的内容
```shell
  options {
        listen-on port 53 { any; };
        // listen-on-v6 port 53 { ::1; };

        ......
  }
```
