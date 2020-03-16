---
layout: post
title: Session Cookie
date: 2019-08-26 20:51:31
categories: HTTP
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## session

就是一种让Request 变成 stateful 的机制。

比如网址：

假设有一个购物的网址是：market.tw, 当你把苹果加入购物车的时候，你其实是送一个request给服务器，然后服务器把你导到market.tw?item1=apple，接着你把火山硅肺病加入购物车，网址就会变成：market.tw?item1=apple&item2=pneumonoultramicroscopicsilicovolcanoconiosis

这个例子就是透过地址栏上的状态来作为session机制。

## Cookie

1. 浏览器发送一个request给server，server叫浏览器设置cookie，浏览器便把这些数据存在cookie里面。比如Set-Cookie:item=apple
2. 浏览器带着cookie一起发request给server，server根据cookie的内容决定状态。

## 关系

session机制可以只靠地址栏实现，跟cookie一点关系都么有，但实际中经常把它们绑定在一起，是因为靠cookie实现的session机制非常方便。

## 安全的考虑

假设要设计一个会员系统，如果登入以后直接把账号存在cookie里面，我可以篡改cookie为别的账号，有两个方法：

1. 把cookie里面的内容加密，cookie-based session，把所有的session状态都存在cookie中。缺点是cookie的大小是有限制的；而且加密方式也有可能被破解。
2. 透过ID来辨识身份。这个ID称为Session Identifier，简称Session ID。Server只在cookie里面存一个Session ID, 其余的状态都存在server那边，Session ID 通常会是一个随机数。