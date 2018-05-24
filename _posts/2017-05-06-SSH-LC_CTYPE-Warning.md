---
layout: post
title: MAC OS X – SSH LC_CTYPE Warningss
date: 2018-05-06 16:27:31
categories: linux
disqus: y
share: y
excerpt_separator: <!--more-->
---

在通过ssh登录到服务器上查看日志的时候，得到中文乱码，通过切换终端测试得到异常，通过以下的解决办法最终解决。

<!--more-->

通过终端ssh登录到服务器时，得到异常：
`warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory`
解决办法是：  
注释掉` /etc/ssh_config`文件中的一行信息：
`#   SendEnv LANG LC_*`  

