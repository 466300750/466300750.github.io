---
layout: post  
title: MAC OS X – SSH LC_CTYPE Warningss  
date: 2018-05-24 13:55:00    
categories: linux   
disqus: y
---      

通过终端ssh登录到服务器时，得到异常：
`warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory`
解决办法是：  
注释掉` /etc/ssh_config`文件中的一行信息：
`#   SendEnv LANG LC_*`
