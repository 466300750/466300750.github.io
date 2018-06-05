---
layout: post
title: spring transactional and lock
date: 2018-06-05 11:48:31
categories: spring
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## Transaction Management   

#### Spring Framework对于事物管理提供了一致的抽象API：   

* Java Transaction API(JTA)
* JDBC
* Hibernate
* Java Persistence API(JPA)
* Java Data Objects(JDO)

#### Spring Framework 事物支持模型的优势   
通常Java EE 对于事物管理有两种方式：**global**和**local**

#### 事物隔离级别
- __Read uncommited__ -- 事物之间没有隔离。
- __Read commited__ -- 事物等待其它事物的行写锁解锁，不会读到脏数据。这种事物级别在读取数据行时会持有一个读锁，在更新或者删除数据行时会持有一个写锁。读锁会在它move off该行时释放该行的读锁，写锁要持续到commit或者rollback。
- __Repeatable read__ -- 事物等待其它事物的行写锁解锁，不会读到脏数据。读锁会锁住此次查询到的所有行，写锁会锁住此次插入、更新、删除的所有行。比如**` SELECT * FROM Orders`**读锁会锁住读到的所有行，但是还是可以往里面插入新数据的；**`DELETE FROM Orders WHERE Status = 'CLOSED'`**也只会锁住这次要删除的所有条件查询出来的数据。事物提交或回滚时释放锁。
- __Serializable__ -- 事物等待其它事物的行写锁解锁，不会读到脏数据。读锁会锁住此次查询所影响到的所有行，写锁会锁住此次插入、更新、删除所影响到的所有行。比如**` SELECT * FROM Orders`**读锁会锁住整个表格，避免了不可重复读；**`DELETE FROM Orders WHERE Status = 'CLOSED'`**锁的范围是所有的status=COLSED的行，并且不允许插入或者更新status=COLSED的行数据，因此也不存在幻读。

*注意:* 事物隔离级别并不会影响事物本身的更新，事物可以看到自己本身所做的任何修改。

### 数据库中读取然后自增   
多进程同时读取数据库order table的ID，然后进行自增，如果采用默认的**`@Transactional`**,这种隔离级别是数据库默认的__Read commited__，这样可能会出现两个进程读出来的ID相同，自增后写入数据库会出现主键冲突。因此，可以考虑提高事物隔离级别，__Repeatable read__。*但是*，如果生产环境是双机，那么就不是同一个session，这时候应该如何处理呢？
   
- DB2，Oracle和MySQL提供了 [Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)  **`select ... for update`**
- Sql Server提供了 [WITH Hints](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-2017)  **`with (UPDLOCK)`**

#### InnoDB提供了两种 Locking Read

- **`select ... for share`**   
  在读取的rows上面设置一个共享锁，其它session可以读，但是不能更改，直到你的事物提交。如果有的rows被其它事物改变了，那么你的请求只能等到事物结束，然后获取到最新的值。   
  但是这个语句不适合用于读取然后自增的场景，容易发生死锁。
- **`select ... for update`**   
  对于index records，锁rows和任何相关的index entries，就如你在这些rows上issure了一个update语句。其它任何update，select ... for share都会被阻塞。   
  这个语句适合于读取然后自增的场景。   
 
#### Locking Read Concurrency with NOWAIT and SKIP LOCKED     
从上面我们知道如果一行被`select ... for share`或者`select ... for update`锁住了，请求同样数据的事物必须等待前一事物释放锁，但是如果你想立马返回被锁住的请求行，或者直接跳过被锁住的rows，避免等待，那么你需要使用**NO WAIT** 和 **SKIP LOCKED**选项。

```
# Session 1:

mysql> CREATE TABLE t (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;

mysql> INSERT INTO t (i) VALUES(1),(2),(3);

mysql> START TRANSACTION;

mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE;
+---+
| i |
+---+
| 2 |
+---+

# Session 2:

mysql> START TRANSACTION;

mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE NOWAIT;
ERROR 3572 (HY000): Do not wait for lock.

# Session 3:

mysql> START TRANSACTION;

mysql> SELECT * FROM t FOR UPDATE SKIP LOCKED;
+---+
| i |
+---+
| 1 |
| 3 |
+---+
```
#### SQL Server 中的 select ... for update    
**WITH ( <table_hint> ) [ [, ]...n ]**   
`with (UPDLOCK)`的用法：	`SELECT * FROM t WITH (UPDLOCK ) WHERE i = 2;`
如果i上面没有索引那么会锁住整个page或者table，因此需要在i列上面创建索引，这样才会只锁住这行。
  