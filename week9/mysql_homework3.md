### **题目 01- 请你说一说 MySQL 的锁机制**

补充：

DML数据控制语言（Data Mainpulation Language）：对数据库中的***数据***进行一些简单操作，insert、update、delete&truncate

DDL数据定义语言（Data Definition Language）：对数据库中的某些***对象（database、table）***创建、查看、删除、修改



要求：

- #### 按照锁的粒度，锁的功能来分析

按锁功能划分：

1. **共享锁（shared lock， S锁、读锁）**：读锁是共享的，读锁之间互相不阻塞
2. **排他锁（exclusive lock，X锁、写锁）**：写锁是排他的，写锁阻塞其他的读和写锁

按粒度划分：

1. **全局锁**：锁DB，由SQL layer层实现

   阻塞DML、DDL及已经更新但未提交的语句

   ```sql
   # 加锁
   flush tables with read lock;
   
   # 释放锁
   unlock tables;
   ```

   典型应用：<font color=blue>全库逻辑备份</font>

   **对mysqldump的思考：**

   ```sql
   # 提交请求锁定所有数据库中的所有表，以保证数据的一致性，全局读锁
   mysqldump -uroot -p --host=localhost --all-databases --lock-all-tables>/root/db.sql
   
   # 一致性视图
   mysqldump -uroot -p --host=localhost --all-databases --single-transaction>/root/db.sql
   ```

   <font color=green>--lock-all-tables</font>：使用全局锁备份，<font color=red>很危险</font>。整个数据库将不能写入，备份期间会影响业务运行，如果在从库上加全局锁，则会导致不能执行主库同步过来的操作，造成主从延迟。

   <font color=green> --single-transaction </font>:innodb：支持事务的引擎，利用mvcc提供一致性视图，而不使用全局锁，不会影响业务的正常运行。

   myisam：不支持事务，只能通过全局锁获得一致性视图。

   

2. **表级锁**：锁Table，由SQL layer层实现

   - 表读锁（Table Read Lock），阻塞对当前表的写，但不阻塞读
   - 表写锁（Table Write Lock），阻塞对当前表的读和写
   - 元数据锁（Meta Data Lock， MDL），不需要显式指定，在访问表时会自动加上，作用保证读写的正确性
     - 当对表做***增删改查***操作时***加元数据读锁***
     - 当对表做***结构变更***操作时加元数据写锁

   ```sql
   # 查看所有表锁定状态
   show status like 'table_locks';
   
   # 查看当前表锁定状态
   show open tables;
   
   # 添加表读锁
   lock table t read;
   
   # 添加表写锁
   lock table t write;
   
   # 删除表锁
   unlock tables;
   ```

   

3. **行级锁**：锁Row的索引，由<font color=red>存储引擎</font>实现

   - 记录锁（Record Locks）：锁定索引中一条记录
   
   <img src="https://user-images.githubusercontent.com/32962270/200098172-f73f0dd8-67a0-491c-95ad-3d9d40b1e91e.png" alt="image-20221023221009032" style="zoom:67%;" />

   
   ```sql
   # 加记录读锁
   select * from t where id = 1 lock in share mode;
   
   # 加记录写锁
   select * from t where id = 1 for update;
   
   # 新增、修改、删除加记录写锁
   insert into t values (2, 22);
   update t set pubtime = 33 where id=2;
   delete from t where id = 2;
   ```
   
   
   
   - 间隙锁（Gap Locks）：仅仅锁住一个索引区间，开区间，不包括双端端点和索引记录
   
   <img src="https://user-images.githubusercontent.com/32962270/200098201-a5c06454-f587-4f78-8516-415e7c89f1e6.png" alt="image-20221023221009032" style="zoom:67%;" />

   
   - 临键锁（Next-Key Locks）：记录锁和间隙锁的组合，左开右闭区间，解决幻读问题
   
     <img src="https://user-images.githubusercontent.com/32962270/200098230-54907aaf-60fa-4b37-99d9-4d0f884d26a2.png" alt="image-20221023221009032" style="zoom:67%;" />

   
     - 默认情况下，InnoDB使用临键锁来锁定记录，但会在不同场景中退化
     - 场景1:唯一性字段等值且记录存在，退化为记录锁
     - 场景2:唯一性字段等值且记录不存在，退化为间隙锁
     - 场景3:唯一性字段范围，还是临键锁
     - 场景4:非唯一性字段，默认临键锁
   
   - 插入意向锁（Insert Intention Locks）：做insert时添加的对记录id的
   
     间隙锁不是意向锁
   
   - 意向锁：存储引擎级别的“表级”锁
   
   如何加行锁？
   
   	- Update、Delete、Insert语句，innoDB自动给涉及数据集加写锁
   	- 对于普通Select语句，InnoDB不会加任何锁
   	- 事务手动给Select记录集加读锁或写锁



加锁规则

主键索引

- 等值条件，命中，加记录锁
- 等值条件，未命中，加间隙锁
- 范围条件，命中，包含where条件的临键区间，加临键锁
- 范围条件，没有命中，加间隙锁

辅助索引

- 等值条件，命中，命中记录的辅助索引项+主键索引加记录锁，辅助索引项两侧加间隙锁
- 等值条件，未命中，加间隙锁
- 范围条件，命中，包含where条件的临键区间加临键锁。命中记录的id索引项加记录锁
- 范围条件，没有命中，加间隙锁

​	4.**意向锁**

意向锁是MySQL内部使用，不需要用户干预。意向锁和行锁可以共存，意向锁主要作用是为了**全表更新数据时的提升性能**

<img src="https://user-images.githubusercontent.com/32962270/200098254-31ebdb1c-d4ee-41ee-be4f-64d61b50816b.png" alt="image-20221023221009032" style="zoom:67%;" />


- #### 什么是死锁，为什么会发生，如何排查？

死锁：两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环的现象。

死锁的原因：

1. 互斥条件：一个资源每次只有被一个进程使用
2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放
3. 不剥夺条件：进程已获得的资源，在没有使用完之前，不能强行剥夺
4. 循环等待条件：多个进程间形成的一种互相循环等待的资源关系

````sql
SHOW ENGINE INNODB STATUS;
````

<img src="https://user-images.githubusercontent.com/32962270/200098323-7230322c-f9a3-4024-bdba-2d09b3ccbf2b.png" alt="image-20221023221009032" style="zoom:67%;" />


<img src="https://user-images.githubusercontent.com/32962270/200098297-8e9f451d-66a1-43ea-ac0f-9abd354aaeed.png" alt="image-20221023221009032" style="zoom:67%;" />

- #### 行锁是通过加在什么上完成的锁定？

通过给索引上的索引项加锁来实现。innodb聚簇索引不能没有主键，没有会用默认的rowid。

- #### 详细说说这条 SQL 的锁定情况： delete from tt where uid = 666 ;

  - **考虑点1**：uid是不是主键？
  - **考虑点2**：隔离级别是什么？
  - **考虑点3**：如果id不是主键，那id有索引吗？
  - **考虑点4**：如果id有索引，那是唯一索引吗？
  - **考虑点5**：SQL的具体执行计划是什么？走索引还是全表扫描（Mysql可能会自主优化，判断走索引还是走全表扫描）

RC隔离级别允许幻读，而RR隔离级别不允许幻读

如何保证两次当前读返回一致的记录？需要在第一次当前读与第二次当前读之间，其他的事务不会插入新的满足条件的记录并提交 -> <font color=red>间隙锁</font>

|                     | 隔离级别RC                                                   | 隔离级别RR                                                   |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| uid是主键           | uid=666记录上加写锁                                          | 同左                                                         |
| uid非主键是唯一索引 | 在辅助索引uid=666中找到符合条件的记录，加写锁；然后拿着主键去主键索引找到记录，加写锁 | 同左                                                         |
| uid非主键是普通索引 | 满足uid=666查询条件的记录均加锁，同时记录对应的主键索引上的记录也都加上了锁 | uid索引定位到第一条满足查询条件的记录，加记录上的写锁，加 GAP 上的间隙锁，然后加主键聚簇索引上的记录写锁，然后返回；然后读取下一条，重复进行。直至进行到第一条不满足条件的记录[667,f]，此时，不需要加记录写锁，但是仍旧需要加间隙锁，最后返回结束。 |
| uid无索引           | 没有索引，只能走聚簇索引，进行全表扫描，聚簇索引所有记录都被加上写锁 | 进行全表扫描的当前读，会锁上表中的所有记录，同时会锁上聚簇索引内的所有间隙，杜绝所有的并发更新/删除/插入操作 |

RR隔离级别下，针对一个复杂SQL，首先需要提取其where条件

- Index Key确定范围，需要加上间隙锁
- Index Filter过滤条件，视MySQL版本是否支持ICP，若支持ICP，则不满足Index FIlter的记录，不加写锁，否则需要加写锁
- Table FIlter过滤条件，无论是否满足，都需要加写锁



### **题目 02- 请你说一说 MySQL 的 SQL 优化**



最左前缀匹配原则：<font color=red>使用组合索引查询时，mysql会一直向右匹配直到遇到范围查询(> 、 < 、 between、like)就停止匹配</font>

**索引下推 ICP**

```sql
show VARIABLES like 'optimizer_switch';
# 打开ICP
SET optimizer_switch = 'index_condition_pushdown=on';

explain select * from t_multiple_index where a=13 and b>15 and c='5' and d='pdf';
```



使用索引下推

索引下推，即使中断了，也会继续用剩下的索引进行过滤
   
<img src="https://user-images.githubusercontent.com/32962270/200098448-73405e5e-1bf2-4516-b9b5-ac5d9a7d6661.png" alt="image-20221023221009032" style="zoom:67%;" />
   
<img src="https://user-images.githubusercontent.com/32962270/200098398-313e1a44-413a-4dd4-8e23-7955ffb2f593.png" alt="image-20221023221009032" style="zoom:67%;" />


筛选条件 a=13 and b >= 15 and c = 5

1.(13, 16, 4, id = 1) 存储引擎下推条件 c=5判断，不满足条件，直接丢弃

2.(13, 16, 5, id = 3)满足筛选条件，使用id=3回表获得id=3的行记录，返回给MySQL服务层，服务层使用剩余条件 d = ‘pdf’过滤，符合要求，缓存到结果集

3.(13, 16, 5, id = 6)满足筛选条件，使用id=6回表获得id=6的行记录，返回给MySQL服务层，服务层使用剩余条件 d = ‘pdf’过滤，不符合要求，直接丢弃

4.(14, 14, 14, id = 8)不满足筛选条件，执行器终止查询

5.最终获得一条记录，返回客户端



<font style=background:yellow>回表查询了2次，然后服务层筛选2次，最终返回客户端</font>



不使用索引下推

<img src="https://user-images.githubusercontent.com/32962270/200098532-62578ccf-5c4b-4648-9dfc-a0d2fef9c2e9.png" alt="image-20221023221009032" style="zoom:67%;" />
 
<img src="https://user-images.githubusercontent.com/32962270/200098484-59b586b8-dba8-4d61-856e-51ae080959b7.png" alt="image-20221030082510621" style="zoom:50%;" />

1.(13, 16, 4, id = 1) 根据最左匹配前缀原则，联合索引检索定位到索引项(13, 16, 4, id = 1), id = 1回表查询，获得id = 1行记录。返回给MySQL服务层，服务层使用剩余条件c=5 and d=‘pdf’过滤，不符合要求，直接丢弃。

2.(13, 16, 5, id = 3)满足筛选条件，使用id=3回表获得id=3的行记录，返回给MySQL服务层，服务层使用剩余条件 d = ‘pdf’过滤，符合要求，缓存到结果集

3.(13, 16, 5, id = 6)满足筛选条件，使用id=6回表获得id=6的行记录，返回给MySQL服务层，服务层使用剩余条件 d = ‘pdf’过滤，不符合要求，直接丢弃

4.(14, 14, 14, id = 8)不满足筛选条件，执行器终止查询

5.最终获得一条记录，返回客户端

<font style=background:yellow>回表查询了3次，然后在服务层筛选后（筛选3次），最后返回客户端</font>



小结：

<font color=red>不使用ICP，不满足最左前缀的索引条件的比较是在Server层进行的，非索引条件的比较是在Server层进行的</font>

<font color=red>使用ICP，所有的索引条件比较是在存储引擎层进行的，非索引条件的比较是在Server层进行的</font>



