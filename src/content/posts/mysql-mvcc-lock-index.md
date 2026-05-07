---
title: 从一次慢查询看 MySQL 索引、锁与 MVCC
date: 2025-10-09
excerpt: 这篇更像数据库学习笔记，用一个订单查询变慢的例子，把索引、回表、锁、MVCC 和排查思路串起来。
tags: [MySQL, MVCC, 索引, 锁, 数据库]
---

MySQL 这块知识点很多，刚学的时候很容易变成背八股：B+ 树、最左前缀、MVCC、undo log、redo log、间隙锁、Next-Key Lock。背概念当然有用，但我后来发现，如果没有放到一条具体 SQL 里，很容易讲得很散。

所以这篇我想用一个慢查询例子来串一下。不是说这个例子覆盖所有情况，而是用它练习怎么把索引、锁和 MVCC 放在一起理解。

## 一个看起来很普通的查询

假设订单表有几千万数据，业务里有一个接口查询用户最近订单：

```sql
select id, user_id, status, amount, created_at
from orders
where user_id = ?
  and status = ?
order by created_at desc
limit 20;
```

这条 SQL 很常见。用户打开订单列表时就会查。如果数据量小，怎么写都快；但数据量一上来，索引设计不合理就会变慢。

最开始我可能只会想：给 `user_id` 加索引就行。因为 where 条件里有 `user_id`。但后面看执行计划才发现，只加单列索引不一定够。

## 先看 EXPLAIN

排查慢 SQL，第一步应该看执行计划。重点不是把 EXPLAIN 每一列都背下来，而是先看几个关键字段：

- `type`：访问类型，至少希望是 ref 或 range。
- `key`：实际用到哪个索引。
- `rows`：预估扫描多少行。
- `Extra`：有没有 Using filesort、Using temporary。

如果只有 `idx_user_id(user_id)`，MySQL 可以根据用户 ID 找到这个用户的所有订单，但还要继续过滤 `status`，然后按 `created_at` 排序。

如果一个用户订单很多，比如几万条，这个扫描和排序就不便宜。

## 联合索引怎么设计

这条 SQL 更适合建联合索引：

```sql
create index idx_user_status_created
on orders(user_id, status, created_at);
```

这个索引顺序不是随便来的。`user_id` 是等值查询，`status` 也是等值查询，`created_at` 用于排序。这样 MySQL 可以先定位用户，再定位状态，最后利用索引里的时间顺序减少排序成本。

如果查询是：

```sql
where user_id = ?
order by created_at desc
```

那索引可能要调整成 `(user_id, created_at)`。所以索引不是看单个字段重要不重要，而是看业务查询模式。

## 回表问题

InnoDB 的二级索引叶子节点里存的是索引字段和主键值。如果查询字段不在索引里，就要根据主键回到聚簇索引查完整记录，这就是回表。

如果订单列表只展示 `id、user_id、status、amount、created_at`，可以考虑覆盖索引：

```sql
create index idx_user_status_created_cover
on orders(user_id, status, created_at, amount);
```

这样查询字段基本都在索引里，回表次数会减少。

但覆盖索引也不是越多越好。索引字段越多，索引体积越大，写入和更新成本也会增加。所以我觉得覆盖索引适合高频读、字段稳定、写入压力可接受的接口。

## MVCC 到底解决什么

MVCC 我以前背的时候会说：多版本并发控制，通过 undo log 和 Read View 实现一致性读。这个说法没错，但有点抽象。

我现在的理解是：MVCC 让普通查询不用和写操作互相阻塞。

比如事务 A 正在更新某一行，事务 B 做普通 select，不一定要等事务 A 提交。事务 B 可以根据自己的 Read View 读到一个可见版本。

在可重复读隔离级别下，同一个事务里多次普通查询看到的结果是一致的。这就是快照读。

但要注意，MVCC 不代表没有锁。下面这些是当前读，会加锁：

```sql
select ... for update;
update ...
delete ...
```

所以如果接口卡住，不一定是查询慢，也可能是锁等待。

## 锁和索引关系很大

InnoDB 的行锁是加在索引上的。如果查询条件没有命中索引，可能扫描很多行，也可能锁住更多范围。

比如：

```sql
update orders
set status = 'CANCELLED'
where order_no = ?;
```

如果 `order_no` 没有索引，这条 update 可能会扫很多数据。虽然最后只更新一行，但扫描过程中会带来很大的锁风险。

在可重复读隔离级别下，范围查询还可能涉及间隙锁和 Next-Key Lock，用来防止幻读。这个机制本身是为了正确性，但如果 SQL 写得不好，就可能影响并发。

所以我现在理解索引的作用，不只是加快查询，也会影响加锁范围。

## 我会怎么排查一条慢 SQL

如果真的遇到订单查询变慢，我会按这个顺序看：

1. 查慢 SQL 日志，确认是哪条 SQL 慢。
2. 用 EXPLAIN 看执行计划。
3. 看实际使用的索引和 rows。
4. 看 Extra 里有没有 filesort 或 temporary。
5. 根据 where 和 order by 调整联合索引。
6. 判断是否需要覆盖索引。
7. 看是否有锁等待。
8. 看事务是否太长。
9. 再看连接池、Buffer Pool、磁盘 IO 这些系统指标。

这个顺序不一定每次都完全一样，但至少不会一上来就盲目加索引。

## 一个学生项目里怎么体现数据库能力

如果简历里只写“熟悉 MySQL 索引、事务、MVCC”，其实很容易像背书。项目里最好能结合具体场景讲。

比如可以这样说：

```text
在消息推送系统中，针对消息状态查询和延时消息扫描设计联合索引；
分析了状态、时间字段组合查询的执行计划；
避免全表扫描导致扫描任务抖动。
```

这样比单纯列知识点更像真的做过。

## 小结

MySQL 优化不是看到慢就加索引。要先看执行计划，理解查询路径，再结合业务访问模式设计索引。

索引、锁和 MVCC 不是孤立知识点。索引影响扫描行数，也影响加锁范围；MVCC 提高读写并发，但当前读仍然会加锁；覆盖索引能减少回表，但会增加写入成本。

我觉得学 MySQL 最重要的是把概念放回 SQL 里理解。能解释一条 SQL 为什么慢、怎么改、改完有什么副作用，比单纯背概念更有价值。
