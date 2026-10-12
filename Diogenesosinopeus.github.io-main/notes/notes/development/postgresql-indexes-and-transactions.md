# PostgreSQL 索引与事务：从查询到提交

索引帮助 PostgreSQL 更快定位行，但会占用空间，也让插入、更新和删除多做维护工作。事务则把一组数据库操作组织成一个可提交或回滚的单元。

## 从查询形状设计索引

假设应用经常查询某个客户最近的订单：

```sql
SELECT id, created_at, total
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

可以评估下面的复合 B-tree 索引：

```sql
CREATE INDEX orders_customer_created_idx
ON orders (customer_id, created_at DESC);
```

把常用过滤列和排序列一起考虑，通常比“给每列都建索引”更有效。复合索引的列顺序会影响可用查询范围；设计后应使用真实数据和执行计划验证，而不是只凭直觉。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at, total
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

`ANALYZE` 会实际执行查询，因此对有副作用或成本很高的语句要谨慎。小表、低选择性的条件或查询返回大部分表行时，顺序扫描也可能更合适。

## 用事务保护一组操作

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

如果中途发生错误，应用可以回滚，使转账不至于只完成一半。事务应尽量短：不要在打开事务后等待用户输入或远程服务响应。

## 记住索引的成本

- 更新索引列时，索引也要维护。
- 重复或很少使用的索引会增加存储和写入成本。
- 用 `pg_stat_user_indexes`、慢查询日志和执行计划观察实际使用情况。
- 为唯一性和完整性建约束时，优先使用数据库约束表达规则；不要只靠应用代码检查。

## 延伸阅读

- [PostgreSQL 索引文档](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL 事务教程](https://www.postgresql.org/docs/current/tutorial-transactions.html)
