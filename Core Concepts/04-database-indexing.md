# Database Indexing

## 是什么

Database Indexing 通过避免 full table scan 来加速查询。当 query 变慢时，index 通常是最先考虑的工具之一。

## 默认选择

- 对经常用于 filters、joins、ordering 的字段加 indexes。
- 普通 relational database lookups 和 range queries 用 B-tree indexes。
- 多字段一起过滤时，考虑 compound indexes。
- text search 或 geospatial needs 用 specialized indexes。

## 什么时候提

- 通过 email 或 username 查 user。
- 通过 parent ID 查 child records。
- 按 time range、location、status 或 category 查询。
- 需要 search text。
- 大结果集需要 sorting。

## Tradeoffs

- Indexes 加速 reads，但会拖慢 writes，因为每次 write 也要更新 index。
- Indexes 会占用 storage。
- External search indexes 可能落后于 source database。
- 选错 index 可能对真实 query pattern 没帮助。

## 面试表达

- "I would index the columns that appear in the high-frequency query path."
- "For city plus date lookup, I would consider a compound index on `(city, date)`."
- "For full-text search, I would not overload the primary database; I would sync into a search index."

