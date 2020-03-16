---
layout: post
title: Linux mount
date: 2019-04-15 09:42:31
categories: Linux
share: y
excerpt_separator: <!--more-->
---


<!--more-->

## Shared subtrees

一种控制子挂载点能否在其他地方被看到的技术，它只会在bind mount 和 mount namespace中用到，属于不怎么常用的功能。接下来将以 bind mount 为例对 shared subtrees做一个简单介绍。

如果bind在一起的两个目录下的子目录再挂载了设备的话，他们之间还能相互看到子目录里挂载的内容吗？比如在第一个目录下的子目录里再mount了一个设备，那么在另一个目录下面能看到这个mount的设备里面的东西吗？答案是依赖于bind mount 的 propagation type。

### peer group

它是一个或多个挂载点的集合，他们之间可以共享挂载信息。有以下两种情况会使挂载点属于同一个peer group（前提条件是挂载点的propagation type是shared）

1. 利用mount --bind命令，前提是源必须是一个挂载点
2. 当创建新的mount namespace时，新namespace会拷贝一份老namespace的挂载点信息，于是新老namespace里的相同挂载点就会属于同一个peer group。

### propagation type

每个**挂载点**都有一个propagation type标志，由它来决定当一个挂载点的下面创建和移除挂载点的时候，是否会传播到属于相同peer group的其他挂载点下去，也就是同一个peer group 里的其他的挂载点下面是不是也会创建和移除相应的挂载点。

1. MS_SHARED: 完全共享
2. MS_SLAVE: 信息的传播是单向的，在同一个peer group里面，master的挂载点下面发生变化的时候，slave的挂载点下面跟着变化，但反之则不然，slave下发生变化的时候不会通知master，master不会发生变化。
3. MS_PRIVATE: 不共享
4. MS_UNBINDABLE: 这种类型的挂载点不能作为bind mount的源，主要用来防止递归嵌套情况的出现。

**需要注意的是：**

1. 挂载点是有父子关系的，比如挂载点/和/mnt/cdrom，/mnt/cdrom都是‘/’的子挂载点，‘/’是/mnt/cdrom的父挂载点

2. 默认情况下，如果父挂载点是MS_SHARED，那么子挂载点也是MS_SHARED的，否则子挂载点将会是MS_PRIVATE，跟爷爷挂载点没有关系


















