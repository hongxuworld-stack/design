# Sharding

## 是什么

Sharding 把数据拆到多个独立 database nodes 上，让单台机器不再负责存储或服务全部数据。

## 什么时候提

- 单个 database 扛不住 write throughput。
- data volume 超过单节点舒适范围。
- read replicas 已经不够。
- queries 天然按 tenant、user、region 或某个 partition key 聚合。

## Shard Key

Shard key 决定数据放在哪里。好的 shard key 应该:

- 均匀分布 load。
- 匹配常见 query patterns。
- 避免 hot partitions。
- 尽量让 transactions 和 joins 留在同一个 shard 内。

## 常见方式

- Hash-based sharding: 分布均匀，但 range querying 不自然。
- Range-based sharding: 适合天然分区的数据，但容易产生 hot spots。
- Directory-based sharding: routing table 灵活，但增加 lookup dependency 和 operational complexity。

## Tradeoffs

- Cross-shard queries 很贵。
- Cross-shard transactions 很难。
- Resharding 运维成本高。
- 某个 key 流量过高时，可能出现 hot shards。

## 面试表达

- "I would not shard until the capacity numbers show a single database plus replicas is not enough."
- "I would shard by `user_id` because most reads are user-scoped."
- "The tradeoff is that global queries now need fanout across shards and aggregation."

