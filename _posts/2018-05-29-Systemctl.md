---
layout: post
title: Systemctl
date: 2018-05-29 14:48:31
categories: linux
share: y
excerpt_separator: <!--more-->
---



<!--more-->

### 简介

`systemctl`是系统服务管理器指令，它实际上将service和chkconfig两个命令组合到一起。

任务            | 旧指令                         | 新指令
-------------  | -------------                 | -----
使某服务自动启动  | `chkconfig --level 3 httpd on` | `systemctl enable httpd.service`
使某服务不自动启动 | `chkconfig --level 3 httpd off` | `systemctl disable httpd.service`
检查服务状态 | `service httpd status` | `systemctl status httpd.service （服务详细信息）`
显示所有已启动的服务 | `chkconfig --list` | `systemctl list-units --type=service`
启动某服务 | `service httpd start` | `systemctl start httpd.service`
停止某服务 | `service httpd stop` | `systemctl stop httpd.service`
重启某服务 | `service httpd restart` | `systemctl restart httpd.service`

###systemctl 命令有两大类功能：

* 控制 systemd 系统
* 管理系统上运行的服务

检查 systemd 的版本:
`systemctl --version`   
确认1号进程，systemd 进程作为系统中的 1 号进程

`Systemd`是初始化系统和系统管理器。
