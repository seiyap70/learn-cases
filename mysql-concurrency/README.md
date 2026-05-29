# MySQL 并发数据处理学习指南

本指南按业务场景组织，覆盖 MySQL InnoDB 并发处理的常见问题、原理分析和最佳实践。

## 目录

- [A. 唯一性约束 / 防重复](A-唯一性约束-防重复.md)
- [B. 库存 / 余额扣减](B-库存余额扣减.md)
- [C. 状态机流转](C-状态机流转.md)
- [D. 计数器 / 累加](D-计数器累加.md)
- [E. 排队 / 抢占](E-排队抢占.md)
- [F. 关联数据一致性](F-关联数据一致性.md)
- [G. 读写分离下的并发](G-读写分离.md)
- [H. DDL 与并发 DML](H-DDL与并发DML.md)
- [I. 死锁](I-死锁.md)

## 前置知识

阅读本指南前，需了解以下 InnoDB 核心概念：

- **MVCC**：多版本并发控制，通过 undo log 版本链实现快照读
- **锁类型**：record lock、gap lock、next-key lock、insert intention lock、predicate lock
- **索引与锁**：InnoDB 行锁本质是索引锁，WHERE 条件走不同索引路径会锁定不同索引上的记录
- **隔离级别**：RC（读已提交）vs RR（可重复读），gap lock 行为差异显著

详细锁机制原理参考 MySQL 官方文档 [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)。
