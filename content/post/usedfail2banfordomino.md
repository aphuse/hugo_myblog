+++
Categories = ["domino", "mail", "fail2ban"]
Description = "介绍LVM的基本用法"
Tags = ["domino", "mail", "fail2ban"]
date = "2016-06-15T20:17:54+08:00"
title = "使用fail2ban防止暴力攻击domino服务"

+++

&emsp;&emsp;当架设好domino邮件服务器并且将其放入公网之后，你就会发现日志里有很多如下的SMTP验证的错误。
```shell
[23671:00010-2690815744] 2016-06-14 15:54:23   SMTP Server: Authentication failed for user heshuai ; connecting host 118.184.13.95
[23671:00013-2181035776] 2016-06-14 15:54:23   SMTP Server: Authentication failed for user gaohuan ; connecting host 118.184.13.95
[23671:00012-2181035776] 2016-06-14 15:54:23   SMTP Server: Authentication failed for user hepin ; connecting host 118.184.13.95
[23671:00012-2181035776] 2016-06-14 15:54:24   SMTP Server: Authentication failed for user gaohuan ; connecting host 118.184.13.95
```

这种错误很明显是有人在暴力攻击你的domino服务器。那我们要想阻止这种暴力攻击的话，使用[fail2ban](www.fail2ban.org)工具是一个简单高效的方法。


在目录/etc/fail2ban/filter.d下新建domino-smtp.conf文件，内容如下：
```shell
# Fail2Ban filter configuration file
#
[Definition]
# 定义比配STMP验证失败正则表达式
# 如下比配前面日志里的记录
# [23671:00012-2181035776] 2016-06-14 15:54:24   SMTP Server: Authentication failed for user gaohuan ; connecting host 118.184.13.95
#
failregex = .* SMTP Server: Authentication failed for user .* connecting host <HOST>

ignoreregex =

```

然后编辑文件jail.local(注：如果没有这个文件就新建)，新加domino-smtp的jail配置
```shell
[domino-smtp]
enabled = true
port = smtp,465
protocol = tcp
filter = domino-smtp
logpath = /opt/ibm/notesdata/IBM_TECHNICAL_SUPPORT/console.log
```

最后重启fail2ban应用使配置生效，然后监控日志/var/log/fail2ban.log，你会看到如下记录，说明配置已经生效。
```shell
2016-06-15 19:36:04,215 fail2ban.actions        [1536]: NOTICE  [domino-smtp] Ban 118.184.13.95
```

或者也可以使用命令fail2ban-client status domino-smtp进行监控。更多fail2ban使用请查看官方网站。使用iptables -L -n 就会发现地址118.184.13.95已经被防火墙阻止访问25和465端口了。
