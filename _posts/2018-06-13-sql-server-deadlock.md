---
layout: post
title: SQL Server deadlock
date: 2018-06-13 16:44:31
categories: SQL
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## 遇到的问题

`Transaction (Process ID) was deadlocked on lock resources with another process and has been chosen as the deadlock victim.`

## 问题描述

根据Excel上传的签收数据信息，更新数据库里的发货单信息。   

首先根据Excel解析出来的数据流，逐条查询每个实体并进行更新操作，最后进行批量的更新。   

## 解决问题的思路

### 失败的尝试

1. 首先尝试降低事物隔离级别，改为`@Transactional(isolation = Isolation.READ_UNCOMMITTED)`
2. select语句中有join，对所有table都设为不加锁`with nolock`
3. select 查询语句上面加 更新锁 `with updlock`

### 成功的尝试
1. 如果把select和update的数据量控制为**1**，则不发生死锁问题。
2. 在update之前对数据进行排序，也不发生死锁问题。

### 分析问题
**batch update的并发操作导致的死锁，update操作默认持有排它锁。**比如事物A update的顺序是[1,2,3,4], 事物B update的顺序是[2,1,3,4],事物A 更新完1后尝试获取2的排它锁，但是2的排它锁被事物 B持有，并且事物B尝试获取1的排它锁，但是由于事物没有提交，都无法释放锁，出现循环等待，死锁也就产生了。

## 经验总结
最开始以为是select持有了共享锁，然后update的时候尝试获取排它锁导致的死锁，这是SQL Server常有的死锁情况。这里会有疑问，select并没有加任何的锁呀。。。这是SQL Server与MySQL的不同:

> MS-SQL Server 使用以下资源锁模式。 

>锁模式 描述    
> > 1. 共享 (S) 用于不更改或不更新数据的操作（只读操作），如 SELECT 语句。     
> > 2. 更新 (U) 用于可更新的资源中。防止当多个会话在读取、锁定以及随后可能进行的资源更新时发生常见形式的死锁。     
> > 3. 排它 (X) 用于数据修改操作，例如 INSERT、UPDATE 或 DELETE。确保不会同时同一资源进行多重更新。     
> > 4. 意向锁 用于建立锁的层次结构。意向锁的类型为：意向共享 (IS)、意向排它 (IX) 以及与意向排它共享 (SIX)。 
> > 5. 架构锁 在执行依赖于表架构的操作时使用。架构锁的类型为：架构修改 (Sch-M) 和架构稳定性 (Sch-S)。 
> > 6. 大容量更新 (BU) 向表中大容量复制数据并指定了 TABLOCK 提示时使用。 

>共享锁 
> > 共享 (S) 锁允许并发事务读取 (SELECT) 一个资源。资源上存在共享 (S) 锁时，任何其它事务都不能修改数据。一旦已经读取数据，便立即释放资源上的共享 (S) 锁，除非将事务隔离级别设置为可重复读或更高级别，或者在事务生存周期内用锁定提示保留共享 (S) 锁。 

>更新锁 
> > 更新 (U) 锁可以防止通常形式的死锁。一般更新模式由一个事务组成，此事务读取记录，获取资源（页或行）的共享 (S) 锁，然后修改行，此操作要求锁转换为排它 (X) 锁。如果两个事务获得了资源上的共享模式锁，然后试图同时更新数据，则一个事务尝试将锁转换为排它 (X) 锁。共享模式到排它锁的转换必须等待一段时间，因为一个事务的排它锁与其它事务的共享模式锁不兼容；发生锁等待。第二个事务试图获取排它 (X) 锁以进行更新。由于两个事务都要转换为排它 (X) 锁，并且每个事务都等待另一个事务释放共享模式锁，因此发生死锁。 

>若要避免这种潜在的死锁问题，请使用更新 (U) 锁。一次只有一个事务可以获得资源的更新 (U) 锁。如果事务修改资源，则更新 (U) 锁转换为排它 (X) 锁。否则，锁转换为共享锁。 

>排它锁 
> > 排它 (X) 锁可以防止并发事务对资源进行访问。其它事务不能读取或修改排它 (X) 锁锁定的数据。 

所以想要在select的时候就加排它锁，但是发现并不起作用。这是因为select语句是只对一条数据加锁成功，所以对于接下来的update语句没有起到加锁的作用。

**因此解决办法有一下3种：**

1. 逐条select 然后update，减小锁的范围

	优点：比较简单   
	缺点：IO较多，性能较慢  
  
2. 所有数据更新完毕后，插入前进行排序   

	优点：改动最小   
	缺点：代码不可读
	
3. 批量查询，更新，然后update

	优点：性能较高，IO最少
	缺点：内存消耗可能会增大。但是鉴于之前的操作，如果数据不合法才会返回为null，内存可以被gc，因此只有在不合法数据较多的情况下才考虑两种操作内存开销的比较。
	
## 多线程测试代码

```
	@Test
    public void test() throws ExecutionException {
        CompletableFuture[] results = IntStream.range(0, 20).mapToObj(i -> {
            return runFF();
        }).toArray(size -> new CompletableFuture[size]);

        CompletableFuture.allOf(results).get();

    }

    private CompletableFuture<Void> runFF() {
        return CompletableFuture.runAsync(() ->{
            ...
        });
    }
```








