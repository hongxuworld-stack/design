# Data Modeling

## 是什么

Data Modeling 是选择 entities、relationships、constraints 和 access patterns。这个选择会影响后面的 performance、scalability 和 maintainability。

## 默认选择

- 数据结构清晰、关系重要时，优先从 relational storage 开始。
- 一开始用 normalized model，避免重复数据。
- 只针对明确的 hot read paths 做 denormalize。
- access patterns 简单、schema flexibility 重要，或 horizontal scaling 是核心需求时，考虑 NoSQL。

## 什么时候提

- 你在定义 core entities。
- reads 和 writes 的性能需求不同。
- 系统有 strong consistency 需求。
- query patterns 已经比较明确。

## Tradeoffs

- Normalization 能提高 consistency、减少 duplicate data，但 joins 可能变贵。
- Denormalization 能加速 reads，但 updates 和 consistency 更难维护。
- Relational databases 支持 joins、constraints 和 transactions。
- NoSQL systems 更容易 horizontal scale，但通常必须围绕 access patterns 设计。

## 面试表达

- "I would start normalized, then denormalize the read-heavy path if the access pattern justifies it."
- "For DynamoDB-style modeling, I need to pick partition and sort keys based on the queries."
- "This data needs strong consistency, so I would avoid making it eventually consistent unless there is a clear business reason."

