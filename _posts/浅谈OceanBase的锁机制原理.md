# 浅谈OceanBase的锁机制原理

OceanBase锁定粒度为行锁，默认情况下的隔离级别为读取已提交（read committed)。另外，读操作总是读取某个版本的快照数据，不需要加锁。

- 只写事务（修改单行）：事务预提交时对待修改的数据行加写锁，事务提交时释放写锁。
- 只写事务（修改多行）：事务预提交时对待修改的多个数据行加写锁，事务提交时释放写锁。为了保证一致性，采用两阶段镇的方式实现，即需要在事务预提交阶段获取所有数据行的写锁，如果获取某行写锁失败，整个事务执行失败。
- 读写事务（read commited）：读写事务中的读操作读取某个版本的快照，写操作的加锁方式与只写事务相同。

为了保证系统并发性能，OceanBase暂时不支持更高的隔离级别。另外，为了支持对一致性要求很高的业务，OceanBase允许用户显式锁住某个数据行。例如，有一张账务表account（account_id，balance），其中account_id为主键。假设需要从A账户
(account_id=1)向B账户（account_id=2）转账100元，那么，A账户需要减少100元，B账户需要增加100元，整个转账操作是一个事务，执行过程中需要防止A账户和B账户被其他事务并发修改。

如以下代码所示，OceanBase提供了”select...for update”语句用于显示锁住A账户或者B账户，防止转账过程中被其他事务并发修改。
```sql
select balance as balance_a
    from account
    where account id=1
    for update; //锁住A账户
select balance as balance_b
    from account
    where account_id=2
    for update; //锁住B账户
```  
事务执行过程中可能会发生死锁，例如事务T1持有账户A的写锁并尝试获取账户B的写锁，事务T2持有账户B的写锁并尝试获取账户A的写锁，这两个事务因为循环等待而出现死锁。OceanBase目前处理死锁的方式很简单，事务执行过程中如果超过一定时间无法获取写锁，则自动回滚。
