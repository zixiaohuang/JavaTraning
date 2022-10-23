本周作业
题目 01- 完成 ReadView 案例，解释为什么 RR 和 RC 隔离级别下看到查询结果不一致
要求：
完成案例 01- 读已提交 RC 隔离级别下的可见性分析
完成案例 02- 可重复读 RR 隔离级别下的可见性分析
用通俗易懂的方式记录整个案例过程，可以画图与截图
做完案例给出结论，并对结论进行分析

理解MVCC实现原理
核心思想：读不加锁，读写不冲突
关键要素：Undo日志和ReadView
实现原理：数据快照，不同的事务访问不同版本的数据快照，从而实现事务下对不同数据的隔离级别。InnoDB通过事务的undo日志实现了多版本的数据快照。

Undo日志分类：
1.Insert Undo日志：Insert操作产生的日志
insert操作记录只对本事务可见，对其他事务不可见，所以事务提交后直接删除Undo日志无需回收

2.Update Undo日志：Update或Delete操作产生的日志
修改事务后，不能立即删除Update Undo日志而是会存入UndoLog链表中，等待Purge线程回收

ReadView
m_ids:生成ReadView时，当前活跃的事务id列表
m_low_limit_id:事务id下限，当前活跃事务中最小的事务id
m_up_limit_id:事务id上限，生成ReadView时，应该分配给下一个事务的id值
m_creator_trx_id:生成该ReadView的事务的事务id

什么时候生成ReadView？
RC和RR隔离级别的差异原因是因为ReadView的生成时机不同
RC：开启事务后，每次select生成ReadView
RR：开启事务后，第一次select生成ReadView

对比LBCC（Lock Based Concurrency Control）：当事务只是加读锁，那么其他事务就不能有写锁，也就是不能修改数据；而假如当前事务要加写锁，那么其他事务就不能持有任何锁。

实验前准备：
创建表
CREATE TABLE t (
    id INT PRIMARY KEY,
    c VARCHAR(100)
) Engine=InnoDB;

插入数据
INSERTINTOtVALUES(1,'刘备');

回答范式：
1. 案例 01- 读已提交 RC 隔离级别下的可见性分析
目标
RC read committed：读已提交，一个事务读到另一个事务已经提交的数据
存在问题：不可重复读
操作步骤
实践过程
结论

2. 案例 02- 可重复读 RR 隔离级别下的可见性分析
目标
RR repeatable read：可重复读，这个级别是MySQL的默认隔离级别，它解决了脏读的问题，同时也保证了同一个事务多次读取同样的记录是一致的，但这个级别还是会出现幻读的情况。幻读是指当一个事务A读取某一个范围的数据时，另一个事务B在这个范围插入行，A事务再次读取这个范围的数据时，会产生幻读
操作步骤
实践过程
结论
结论分析

题目 02- 什么是索引？
要点：
优点是什么？
缺点是什么？
索引分类有哪些？特点是什么？
索引创建的原则是什么？
有哪些使用索引的注意事项？
如何知道 SQL 是否用到了索引？
请你解释一下索引的原理是什么？【重点】
- 说清楚为什么要用 B+Tree


题目 03- 什么是 MVCC？
要点：

Redo 日志
ReadView
如何判断可见性