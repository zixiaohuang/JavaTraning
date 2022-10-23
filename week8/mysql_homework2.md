本周作业<br>

#### 题目 01- 完成 ReadView 案例，解释为什么 RR 和 RC 隔离级别下看到查询结果不一致<br>

要求：<br>

##### 完成案例 01- 读已提交 RC 隔离级别下的可见性分析<br>

##### 完成案例 02- 可重复读 RR 隔离级别下的可见性分析<br>

用通俗易懂的方式记录整个案例过程，可以画图与截图<br>
做完案例给出结论，并对结论进行分析<br>

实验前准备：<br>
创建表

```sql
CREATE TABLE `tab_user` (
  `id` int(11) NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `age` int(11) NOT NULL,
  `address` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
Insert into tab_user(id,name,age,address) values (1,'刘备',18,'蜀国');
```

操作步骤

<img src="https://user-images.githubusercontent.com/32962270/197382347-0db42174-a056-4775-b728-3c96daf74516.png" alt="image-20221023155441664" style="zoom: 67%;" />

版本链：

<img src="https://user-images.githubusercontent.com/32962270/197382380-e8102500-50e1-47de-a642-c416be963919.png" alt="image-20221023160400073" style="zoom:67%;" />

事务01:

```sql
# 事务01
 -- 查询事务隔离级别:
 select @@tx_isolation;
 -- 设置数据库的隔离级别
 set session transaction isolation level read committed; SELECT * FROM tab_user; # 默认是刘备
 \# Transaction 100
 BEGIN;

UPDATE tab_user SET name = '关羽' WHERE id = 1; UPDATE tab_user SET name = '张飞' WHERE id = 1; COMMIT;
```



事务02:

~~~sql
`# 事务02`
 `-- 查询事务隔离级别:`
 `select @@tx_isolation;`
 `-- 设置数据库的隔离级别`
 `set session transaction isolation level read committed;`

```
# Transaction 200
```

`BEGIN;`
 `\# 更新了一些别的表的记录`
 `...`
 `UPDATE tab_user SET name = '赵云' WHERE id = 1;`

`UPDATE tab_user SET name = '诸葛亮' WHERE id = 1; COMMIT;
~~~



事务3

```sql
# 事务03
 -- 查询事务隔离级别:
 select @@tx_isolation;
 -- 设置数据库的隔离级别
 set session transaction isolation level read committed;

BEGIN;

# SELECT01:Transaction 100、200未提交
SELECT * FROM tab_user WHERE id = 1; # 得到的列c的值为'刘备'
# SELECT02:Transaction 100提交，Transaction 200未提交 SELECT * FROM tab_user WHERE id = 1; # 得到的列c的值为'张飞'
# SELECT03:Transaction 100、200提交
SELECT * FROM tab_user WHERE id = 1; # 得到的列c的值为'诸葛亮' COMMIT;
```

其他

```sql
  -- 开启事务:还有一种方式begin start transaction
-- 提交事务:
commit
-- 回滚事务:
rollback
-- 查询事务隔离级别:
select @@tx_isolation;
-- 设置数据库的隔离级别
set session transaction isolation level read committed
-- 级别字符串:`read uncommitted`、`read committed`、`repeatable read【默认】`、 `serializable`
-- 查看当前运行的事务 
SELECT
    a.trx_id,a.trx_state,a.trx_started,a.trx_query,
   b.ID,b.USER,b.DB,b.COMMAND,b.TIME,b.STATE,b.INFO,
   c.PROCESSLIST_USER,c.PROCESSLIST_HOST,c.PROCESSLIST_DB,   		 d.SQL_TEXT
FROM
   information_schema.INNODB_TRX a
LEFT JOIN information_schema.PROCESSLIST b ON a.trx_mysql_thread_id = b.id
AND b.COMMAND = 'Sleep'
LEFT JOIN PERFORMANCE_SCHEMA.threads c ON b.id = c.PROCESSLIST_ID
LEFT JOIN PERFORMANCE_SCHEMA.events_statements_current d ON d.THREAD_ID =
c.THREAD_ID;
```



回答范式：

1. 案例 01- 读已提交 RC 隔离级别下的可见性分析

   RC read committed：读已提交，一个事务读到另一个事务已经提交的数据<br>
   存在问题：不可重复读<br>

   RC

   **实践过程**

   T5时刻，事务02更新阻塞

   T6时刻，事务03查询为刘备

   T7时刻，事务01事务提交成功后，事务02获取到锁，更新成功

   T8时刻，事务03查询为张飞，即事务01已提交的值

   T9时刻，事务02提交成功

   T10时刻，事务03查询到事务02更新值，即诸葛亮

   **结论**
   使用RC隔离级别的事务每次查询开始都会生成一个独立的ReadView，更新ReadView的活跃事务id m_ids，已经提交的事务会被移除，变得可见

   每次查询会从版本链中挑选可见的记录，如果版本trx_id在m_ids列表中，则不可见。

   

2. 案例 02- 可重复读 RR 隔离级别下的可见性分析<br>
   RR repeatable read：可重复读，这个级别是MySQL的默认隔离级别，它解决了脏读的问题，同时也保证了同一个事务多次读取同样的记录是一致的，但这个级别还是会出现幻读的情况。幻读是指当一个事务A读取某一个范围的数据时，另一个事务B在这个范围插入行，A事务再次读取这个范围的数据时，会产生幻读<br>

   **实践过程**

   需要注意，在mac中navicat设置隔离级别后再设置有可能会存在，虽然显示隔离级别已经修改成功了，但实际还是之前的隔离级别，导致实验不成功，可新建立连接解决。

   T6、T8、T10时刻，事务3查询均为刘备

   **结论**
   RR隔离级别事务，只会在第一次执行查询语句时生成一个ReadView，之后的查询就不会重复生成了，所以即便事务01、事务02提交了事务，ReadView中的活跃id m_ids列表还是事务03开启时的值，确保可重复读。

   

#### 题目 02- 什么是索引？<br>

要点：<br>
优点是什么？<br>
缺点是什么？<br>
索引分类有哪些？特点是什么？<br>
索引创建的原则是什么？<br>
有哪些使用索引的注意事项？<br>
如何知道 SQL 是否用到了索引？<br>
请你解释一下索引的原理是什么？【重点】<br>

说清楚为什么要用 B+Tree<br>
<br>
<br>

#### 题目 03- 什么是 MVCC？<br>

要点：<br>

<img src="https://user-images.githubusercontent.com/32962270/197382280-529923fc-4e0a-4229-8c12-fbf26716a792.png" alt="image-20221023154538967" style="zoom:50%;" />

Redo 日志<br>
ReadView<br>
如何判断可见性<br>

理解MVCC实现原理<br>
核心思想：读不加锁，读写不冲突<br>
关键要素：Undo日志和ReadView<br>
实现原理：数据快照，不同的事务访问不同版本的数据快照，从而实现事务下对不同数据的隔离级别。InnoDB通过事务的undo日志实现了多版本的数据快照。<br>
<br>
Undo日志分类：

对于RU隔离级别事务：直接读取记录最新版本就行，不需要Undo日志

对于串行化隔离级别事务，使用加锁方式来访问记录，不需要Undo日志

对于RC和RR隔离级别事务，需要用到Undo的版本链

1.Insert Undo日志：Insert操作产生的日志<br>
insert操作记录只对本事务可见，对其他事务不可见，所以事务提交后直接删除Undo日志无需回收<br>
<br>
2.Update Undo日志：Update或Delete操作产生的日志<br>
修改事务后，不能立即删除Update Undo日志而是会存入UndoLog链表中，等待Purge线程回收<br>
<br>
ReadView<br>
m_ids:生成ReadView时，当前活跃的事务id列表<br>
m_low_limit_id:事务id下限，当前活跃事务中最小的事务id<br>
m_up_limit_id:事务id上限，生成ReadView时，应该分配给下一个事务的id值<br>
m_creator_trx_id:生成该ReadView的事务的事务id<br>
<br>
什么时候生成ReadView？<br>
RC和RR隔离级别的差异原因是因为ReadView的生成时机不同<br>
RC：开启事务后，每次select生成ReadView<br>
RR：开启事务后，第一次select生成ReadView<br>
<br>
对比LBCC（Lock Based Concurrency Control）：当事务只是加读锁，那么其他事务就不能有写锁，也就是不能修改数据；而假如当前事务要加写锁，那么其他事务就不能持有任何锁。<br>
<br>
可见性判断<br>TRX_ID小于m_low_limit_id，表示在当前事务前已经提交的事务，可以被当前事务访问

TRX_ID等于m_create_trx_id，表示是当前事务，可以被当前事务访问

TRX_ID大于m_up_limit_id，表示是在当前事务后才创建的事务，当前事务不可见

TRX_ID在low和up的limit之间，则需要判断是否在m_ids里面。如果RC级别，已提交，不在m_ids活跃事务id里面，则可见；如果RR级别，无论是否提交都不可见
