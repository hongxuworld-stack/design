# Caching

## 是什么

Caching 把经常读取的数据放到更快的层里，让系统避免反复做 expensive work 或 database reads。

## 默认选择

- application data 通常用 Redis 做 cache-aside。
- static assets 和 large media 用 CDN。
- 稳定的 config 或 feature flags 可以用小型 in-process cache。
- TTLs 和 invalidation rules 要有意识地设计。

## 什么时候提

- read traffic 明显高于 write traffic。
- 同一批数据被反复请求。
- database read latency 或 load 成为瓶颈。
- 系统可以接受一定 staleness。

## Tradeoffs

- 数据变化时，cache invalidation 很难。
- 大量 request 同时 miss 时，会出现 cache stampede。
- cache outage 可能把所有流量打回 database。
- 对频繁变化的数据做 caching，可能只增加复杂度，没有收益。

## 常见防御

- writes 之后 delete 或 update cache entries。
- 能接受轻微 staleness 时，用较短 TTLs。
- 用 request coalescing 或 locks 防止 stampedes。
- stagger TTLs，避免很多 keys 同时过期。
- cache down 时，用 circuit breakers 或 graceful degradation。

## 面试表达

- "I would cache only hot read paths, not every table."
- "On a miss, the service reads from the database, writes to Redis with a TTL, and returns the result."
- "For this object, stale data for a few seconds is acceptable, so TTL-based caching is reasonable."

