# 关于MySQL XA事务的隔离级别

## 为什么XA事务建议用SERIALIZABLE隔离级别

在MySQL最新的官方文档中，关于[XA Transactions](https://dev.mysql.com/doc/refman/8.0/en/xa.html)的介绍有这么一段描述：

> As with nondistributed transactions, `SERIALIZABLE` may be preferred if your applications are sensitive to read phenomena. `REPEATABLE READ` may not be sufficient for distributed transactions.

这段话表达的意思是，对于分布式XA事务， `REPEATABLE READ` 隔离级别是不够的。

MySQL旧版本的[文档](https://docs.oracle.com/cd/E17952_01/mysql-5.0-en/xa.html)，对于这个问题表达的更加直接：

> However, for a distributed transaction, you must use the `SERIALIZABLE` isolation level to achieve ACID properties. It is enough to use REPEATABLE READ for a nondistributed transaction, but not for a distributed transaction.

怎么理解呢？举个简单的例子：假设MySQL使用的是`REPEATABLE READ` 隔离级别，XA事务 `T1 ` 修改的数据涉及两个节点 `A` 和 `B`，当事务 `T1` 在 `A` 上完成commit，而在 `B` 上还没commit之前，也就是说这时事务 `T1` 并没有真正结束，另一个XA事务 `T2` 已经可以访问到 `T1` 在 `A` 上提交后数据，这不是出现脏读了吗？

那么使用`SERIALIZABLE`就能保证吗？还是看例子：事务 `T1` 修改节点 `A` 上的数据 `a -> a'`，修改 `B` 上的数据 `b -> b'`，在提交阶段，可能被其他事务 `T2` 读取到了 `a'`， 因为使用了`SERIALIZABLE`隔离级别， MySQL 会对所有读加锁，那么 `T2` 在 `B` 上读取 `b` 时会被一直阻塞，直到 `T1` 在 `B` 上完成commit，这时 `T2` 在 `B` 读取到的就是 `b'`。 也就是说，`SERIALIZABLE`隔离级别保证了读到 `a'` 的事务，不会读到 `b` ，而是读到 `b'`，确保了事务ACID的要求。

更加详细的描述可以参考[鹅厂 TDSQL XA 事务隔离级别的奥秘](https://cloud.tencent.com/developer/article/1005380)，他们的结论是：

> 如果某个并发事务调度机制可以让具有依赖关系的事务构成一个有向无环图(DAG)，那么这个调度就是可串行化的调度。由于每个后端DB都在使用serializable隔离级别，所以每个后端DB上面并发执行的事务分支构成的依赖关系图一定是DAG。
> 
> 只要所有连接都是用serializable隔离级别，那么TDSQL XA执行的事务仍然可以达到可串行化隔离级别。

## SERIALIZABLE性能差，有更好的实现方式吗

如果分布式事务想实现`read-committed`以上的隔离级别，又不想使用`SERIALIZABLE`，有什么更好的方式吗？

当然有，想想看`TiDB`是怎么做的，底层`TiKV`是一个整体，有全局的MVCC，所以能够做到分布式事务的Snapshot隔离级别。

PostgreSQL社区中，有[Postgres-XC](http://postgresxc.wikia.com/wiki/Postgres-XC_Wiki)和[Postgres-XL](https://www.postgres-xl.org/)的方案，采用的并发机制是全局MVCC 和本地写锁。 `Postgres-XC` 维持了全局活跃事务列表，从而提供全局MVCC。

虽然MySQL也实现了MVCC，但它没有将底层K/V带有时间戳的版本信息暴露出来。也就是说，多个MySQL实例组成的集群没有全局的MVCC，无法得到全局一致性快照，自然就很难做到分布式的Snapshot隔离级别。腾讯的[这篇文章](https://cloud.tencent.com/developer/article/1005380)也分析了这么做比较困难：

> 由于MySQL innodb使用MVCC做select（除了serializable和for update/lock in share mode子句），还需要将这个全局事务id给予innodb做事务id，同时，还需要TDSQL XA集群的多个set的innodb 共享各自的本地事务状态给所有其他innodb（这也是PGXL 所做的），**任何一个innodb的本地事务的启动，prepare，commit，abort都需要通知给所有其他innodb实例。只有这样做，集群中的每个innodb实例才能够建立全局完全有一致的、当前集群中正在处理的所有事务的状态，以便做多版本并发控制。** 这本身都会造成极大的性能开销，并且导致set之间的严重依赖，降低系统可靠性。这些都是我们要极力避免的。

## 结论

根据上面的分析，如果使用MySQL 的 XA分布式事务，最安全的方式还是按照官方建议，使用`SERIALIZABLE`隔离级别。

如果想基于MySQL做改造，实现全局MVCC，从而实现分布式事务的Snapshot隔离级别，目前还没有看到MySQL社区有这类项目，相信实现难度比较大。

