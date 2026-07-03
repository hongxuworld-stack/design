# Numbers to Know

## 是什么

Back-of-the-envelope numbers 用来支撑设计决策。比如判断某个 component 是否需要 scaling、caching、sharding、queues 或 regional deployment。

## Latency Intuition

- Memory access: nanoseconds。
- SSD reads: microseconds。
- Same data center network calls: low milliseconds。
- Cross-region 或 cross-continent calls: tens to hundreds of milliseconds。
- Redis cache hits 通常在 low-millisecond range。
- Database queries 差异很大；uncached reads 通常明显慢于 cache hits。

## Capacity Intuition

- 一个调优好的 database 能力通常比很多候选人想象中强。
- 如果 memory 放得下，单个 Redis instance 可以处理很高的 operation volume。
- App servers 通常在 CPU、memory、connections 或 latency 接近上限时 horizontal scale。
- Message queues 适合 producers 和 consumers 需要 buffering、fanout 或 independent scaling 的场景。

## 什么时候用数字

- 估算 QPS。
- 估算 storage。
- 判断一个 database 是否足够。
- 判断 read replicas 是否足够。
- 判断 cache 是否值得引入复杂度。
- 估算 server count。

## 面试表达

- "Before sharding, I would check whether the write throughput and storage actually exceed a single database plus replicas."
- "If we expect 50k requests per second and each app server handles around 5k, we need roughly 10 servers plus headroom."
- "The global latency requirement matters because users far from the region will pay cross-region network cost."

