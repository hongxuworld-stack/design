# CAP Theorem

## 是什么

CAP Theorem 描述 distributed system 在 network partition 发生时的行为。因为 partition 可能发生，所以实际选择通常是在 failure 期间偏向 consistency 还是 availability。

## 术语

- Consistency: 每个 node 都返回同一个最新值。
- Availability: 每个 request 都能得到 response。
- Partition tolerance: network split 发生时，系统仍能继续运行。

## 默认思路

- feeds、recommendations、analytics、comments、social content 通常可以接受 eventual consistency。
- money、inventory、booking、permissions，以及 stale reads 会造成真实损失的场景，需要 strong consistency。
- 同一个系统的不同部分可以有不同 consistency requirements。

## Tradeoffs

- Strong consistency 能减少错误行为，但会增加 coordination 和 latency。
- High availability 让产品保持可用，但可能返回 stale data。
- CAP 主要讨论 partitions 期间；正常情况下，更常见的 tradeoff 是 consistency vs latency。

## 面试表达

- "For this feed, eventual consistency is acceptable because a short delay does not break correctness."
- "For seat booking, I need strong consistency to prevent double booking."
- "I would choose consistency per subsystem rather than applying one model to the whole product."

