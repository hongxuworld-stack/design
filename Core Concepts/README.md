# Core Concepts

来源: https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts

这些是 System Design 面试里最常出现的基础概念。它们不是用来硬塞进答案里的 buzzwords，而是用来帮助你解释设计选择、scale trigger 和 tradeoff 的工具。

## 学习顺序

1. [Networking Essentials](./01-networking-essentials.md)
2. [API Design](./02-api-design.md)
3. [Data Modeling](./03-data-modeling.md)
4. [Database Indexing](./04-database-indexing.md)
5. [Caching](./05-caching.md)
6. [Sharding](./06-sharding.md)
7. [Consistent Hashing](./07-consistent-hashing.md)
8. [CAP Theorem](./08-cap-theorem.md)
9. [Numbers to Know](./09-numbers-to-know.md)

## 面试中怎么用

- 先从简单方案开始: single database、普通 API、HTTP、基本 indexes。
- 只有当 requirements 或 scale 逼出来时，才加复杂度。
- 每用一个技术，都说清楚 trigger 和 tradeoff。
- 需要做设计决策时，用 rough numbers 支撑，比如要不要 shard、cache、replicate 或加 queue。
- 表达要具体: "feed 可以接受 eventual consistency" 比 "NoSQL scales better" 更有说服力。

## 快速地图

| Concept | 什么时候用 | 主要风险 |
| --- | --- | --- |
| Networking | clients、services、regions 之间要通信 | latency、failure、persistent connections |
| API Design | 定义 client 和 system 的交互 | 在 API 细节上花太久 |
| Data Modeling | 决定存什么数据、怎么组织 | access pattern 或 consistency model 选错 |
| Database Indexing | 高频查询需要变快 | write overhead、external index staleness |
| Caching | read-heavy 热数据压垮 storage | invalidation、stampede、stale data |
| Sharding | 单个 database 扛不住 storage/write/load | cross-shard queries、resharding |
| Consistent Hashing | nodes 会动态增加或减少 | 没有 virtual nodes 时分布不均 |
| CAP Theorem | distributed data 要处理 network partitions | 盲目选择 consistency 或 availability |
| Numbers to Know | 需要用数字支撑容量决策 | 用过时的 scale 假设 |

