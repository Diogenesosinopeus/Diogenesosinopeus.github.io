
Redis 不只是一个字符串缓存。选对数据结构，可以让常见操作更直接，也能少做重复的序列化和读改写。

## 先按访问方式选结构

| 需求 | 常见结构 | 例子 |
| --- | --- | --- |
| 缓存一段完整结果、计数器 | String | 页面缓存、访问次数 |
| 一个对象的多个字段 | Hash | 用户偏好、设备状态 |
| 去重与成员判断 | Set | 已处理的任务 ID |
| 按分数排序 | Sorted Set | 排行榜、延迟队列索引 |
| 先进先出工作队列 | List 或 Streams | 简单任务队列、可追踪事件流 |

结构并非只能用于表格里的场景。关键是让需要的读写操作贴合 Redis 提供的原子命令。

## 缓存示例

```text
SET product:42 '{"name":"键盘","stock":8}' EX 300
GET product:42
TTL product:42
```

`EX 300` 让整个键在 300 秒后过期。缓存未命中时从数据库读取，再写入缓存，是常见的 cache-aside 流程：

1. 先读缓存。
2. 未命中时读数据库。
3. 将结果写入缓存并设置合理 TTL。
4. 更新数据库后，删除对应缓存键或按一致性策略更新缓存。

TTL 能限制陈旧数据的最长存活时间，但不能单独解决缓存与数据库的一致性问题。TTL 也不应该被设得长到让失效数据长期可见。

## 哈希与键级过期

```text
HSET device:7 status online firmware 1.4.2
HGET device:7 status
EXPIRE device:7 600
```

`EXPIRE` 设置的是整个 `device:7` 键的生命周期。设计键名时可以加入业务对象和字段语义，例如 `user:42:profile`，并避免把访问令牌、密码等敏感值当作普通缓存内容长期保存。

## 选择时检查三件事

- **原子操作**：计数优先考虑 `INCR`，不要在应用中先读再加一再写。
- **生命周期**：缓存、验证码、临时锁等通常需要明确 TTL；永久数据则要有清理和容量计划。
- **并发语义**：锁、队列和任务处理要考虑重复执行、过期、消费者故障等情况。

## 延伸阅读

- [Redis 数据类型总览](https://redis.io/docs/latest/develop/data-types/)
- [Redis 数据类型选择指南](https://redis.io/docs/latest/develop/data-types/compare-data-types/)
- [Redis 键过期说明](https://redis.io/docs/latest/develop/using-commands/keyspace/)
