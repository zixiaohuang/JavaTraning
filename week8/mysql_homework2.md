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

##### 索引分类有哪些？特点是什么？

按照数量分类：

1.单列索引

- 主键索引：值必须是唯一的，不允许有空值

`ALTER TABLE table_name ADD PRIMARY KEY (column_name);`

- 普通索引：基本索引类型，没有什么限制，允许在定义索引的列中插入重复值和空值

`ALTER TABLE table_name ADD INDEX index_name(column_name);`

- 唯一索引：索引列中的值必须是唯一的，但是允许为空值

`CREATE UNIQUE INDEX index_name ON table(column_name);`

- 全文索引：只能在文本类型CHAR、VARCHAR、TEXT类型字段上创建全文索引。

```sql
 #创建表时，创建全文索引
 CREATE TABLE `t_fulltext` ( 
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `content` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  FULLTEXT KEY `idx_content` (`content`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

\#创建全文索引
 ALTER TABLE `t_fulltext` ADD FULLTEXT INDEX `idx_content`(`content`);
```

- 空间索引
- 前缀索引

2.组合索引

`ALTER TABLE table_name ADD INDEX index_name(column1,column2);`

MyIsam索引

1.主键索引<br>
<img src="https://user-images.githubusercontent.com/32962270/197398259-2524f43b-7813-4764-ac82-5553a436e8c1.png" alt="image-20221023221009032" style="zoom:67%;" />


2.辅助索引
MyISAM中辅助索引和主键索引结构一样，没有区别，叶子节点存储的都是行记录的**磁盘地址**。只是主键索引的键值是唯一的，而辅助索引的键值可以重复。



InnoDB索引
**1.聚簇索引**

<img src="https://user-images.githubusercontent.com/32962270/197398311-27671c35-b679-44c4-a6ac-4ae5221d65e6.png" alt="image-20221023175525168" style="zoom:67%;" />


一般情况下，聚簇索引等同于主键索引。主键索引的叶子节点会存储数据行。

**InnoDB要求表必须有一个主键索引。**当一个表没有创建主键索引时，InnoDB会自动创建一个ROWID字段来构建聚簇索引。

InnoDB创建索引的具体规则如下：

1.在表上定义主键PRIMARY KEY，InnoDB将主键索引作用聚簇索引

2.如果表上没有定义主键，InnoDB会选择第一个不为NULL的唯一索引列用作聚簇索引

3.如果以上两个都没有，InnoDB会使用一个6字节长整形的隐式字段ROWID字段构建聚簇索引。该ROWID字段会在插入新行时自动递增。

**磁盘IO次数：**2次+检索叶子节点数量

**2.辅助索引**

<img src="https://user-images.githubusercontent.com/32962270/197398348-afa2443b-8e23-4b81-9df4-6b20b530d1d8.png" alt="image-20221023180600328" style="zoom:67%;" />


除聚簇索引之外的所有索引都称为辅助索引。辅助索引只会存储主键值而非磁盘地址。

底层叶子节点按照(age, id)的顺序排序，先按照age列从小到大排序，age列相同时按照id列从小到大排序

使用辅助索引需要检索两遍索引：

- 首先检索辅助索引获得主键
- 然后使用主键到主索引中检索获得记录

**磁盘IO次数：**2次+检索叶子节点数量 + 记录数*3 （因为需要回表查询，回表查询大概需要3次磁盘IO）



**3.组合索引**

表t_multiple_indx，id为主键列，创建了一个联合索引id x_abc(a,b,c)，构建的B+树索引结构如图所示。索引树中节点中的索引项按照(a,b,c)的顺序从大到小排列，先按照a排序，a列相同时按照b列排序，b列相同按照c列排序。如果索引项都相同，按照主键id排序

<img src="https://user-images.githubusercontent.com/32962270/197398399-619a8cbe-250d-4360-8506-98fd0c7a68fc.png" alt="image-20221023192746579" style="zoom:67%;" />

最左匹配原则：组合索引查询是，会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配。如果查询条件不包括a列，比如筛选条件只有(b,c)或者c列是无法使用组合索引的。所以创建(a,b,c)相当于创建了(a)、(a,b)、(a,b,c)三个索引

组合索引创建原则：

1. 频繁出现在where条件中的列，建议创建组合索引
2. 频繁出现order by 和 group by语句的列，建议按照顺序去创建组合索引
3. 常出现select语句中的列，也建议创建组合索引



**4.覆盖索引**

select中列数据，如果**可以直接在辅助索引树上全部获取**，也就是索引树已经“覆盖”了我们的查询要求，MySQL就不会白费力气的回表查询

##### 有哪些使用索引的注意事项？

注意事项

- 不要在列上使用函数和进行运算，会导致索引失效
- 尽量避免使用!=或not in 或<>等否定操作符，会导致索引失效
- 组合索引最左匹配原则
- 如果mysql估计使用全表扫描要比使用索引快,则不使用索引



不需要使用索引

- 数据唯一性差的字段不要使用索引。因为无法准确的找到想要的数据,所以查完索引后依然还需要过一遍数据,这样反而增加了查询量
- 频繁更新的字段不要使用索引,频繁更新会导致索引也会频繁更新,降低写的效率
- 字段不在where语句出现时不要添加索引:只有在where语句出现，mysql才会去使用索引
- 数据量少的表不要使用索引,使用了改善也不大



##### 如何知道 SQL 是否用到了索引？

`explain select a,b from  t_multiple_index where  b=16;`
<img width="832" alt="image" src="https://user-images.githubusercontent.com/32962270/197398452-5a8ab3fb-4a65-46f4-a080-5994e94ca2d1.png">


`explain select * from t_multiple_index where b=16 and c=4 and a=13;`
<img width="867" alt="image" src="https://user-images.githubusercontent.com/32962270/197398501-e929e531-b798-4a42-af74-798534b8806d.png">


通过explain查看

possible keys:可能使用哪些索引来查找。key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。 

key：实际采用哪个索引来优化对该表的访问。



##### 请你解释一下索引的原理是什么？【重点】

帮助MySQL高效获取数据的数据结构，加快数据库查询速度

优势：

- 提高数据检索效率、降低数据库的IO成本
- 索引列对数据进行排序，降低数据排序的成本，降低CPU消耗

劣势：

- 占据磁盘空间
- 降低更新表的效率，每次对表增删改操作，不仅要保存数据，还要维护索引文件



##### 说清楚为什么要用 B+Tree

**Hash表**

优点：等值查询

缺点：

1.不支持范围快速查找，范围查找时还是只能通过扫描全表方式

2.数据结构比较稀疏，不适合做聚合，不适合范围查找

**二叉查找树**

理想情况可以达到O(logn)

极端情况下，会退化为单向链表=查找全表扫描

**红黑树**

近似平衡二叉树，通过染色和平衡因子，通过左旋/右旋控制二叉树平衡（层级最多相差1）

缺点：

1.时间复杂度和树高相关：树有多高就需要检索多少次，每个节点的读取，都对应一次磁盘IO操作

2.平衡二叉树不支持范围查询快速查找，范围查询时需要从根节点多次遍历，查询效率极差

3.数据量大的情况下，索引存储空间占用极大

**B树**

想减少IO操作，就要尽量降低树的高度。每个节点存储多个元素，就将二叉树改造成了多叉树，通过增加树的叉树，将树从高瘦变成矮胖。
<img src="https://user-images.githubusercontent.com/32962270/197398549-ee5e8cc3-266a-4afa-bd8a-19111594b7a9.png" alt="image-20221023154538967" style="zoom:67%;" /><br>
优点：

1.磁盘IO次数大大减少。

2.比较是在内存进行的，比较的耗时可以忽略不计

3.B树高度一般2至3层就能满足大部分的应用场景，所以使用B树构建索引可以很好的提升查询的效率

缺点：

1.B树不支持范围查询的快速查找，需要回到根节点重新遍历查找

2.空间占用较大：如果data存储的是行记录，行的大小随着列数的增多，所占空间会变大。一个页中可存储的数据量就会变少，树相应就会变高，磁盘IO次数就会变大。

**B+树：改进的B树，非叶子节点不存储数据**

B树：非叶子节点和叶子节点都会存储数据

B+树：只有叶子节点才会存储数据，非叶子节点只存储键值。叶子节点之间使用双向指针连接，最底层的叶子节点形成一个双向有序链表
<img src="https://user-images.githubusercontent.com/32962270/197398530-41f8141a-d8e0-41dc-ba6e-244a46ffa255.png" alt="image-20221023154538967" style="zoom:67%;" />

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



参考：

1.Java高级工程师训练营
