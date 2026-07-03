# Consistent Hashing

## 是什么

Consistent Hashing 用来把 keys 分布到多个 nodes 上，并且在增加或删除 node 时，只移动一小部分 keys。

## 为什么重要

如果用简单的 modulo hashing，nodes 数量一变，大部分 keys 都会被重新映射。Consistent Hashing 可以避免这种大规模 reshuffling，所以特别适合 distributed caches 和 partitioned storage。

## 什么时候提

- Distributed cache clusters。
- 需要 elastic scaling 的 sharded databases。
- nodes 经常增加或减少的系统。
- 需要稳定地把 requests route 到 backend nodes。

## Tradeoffs

- 如果没有 virtual nodes，hash ring 可能不均匀。
- 比完整 remapping 更容易运维，但仍然不是零成本。
- node loss 仍然需要 replication 和 failure handling。

## 面试表达

- "For a distributed cache, I would use consistent hashing so adding cache nodes does not remap most keys."
- "Virtual nodes help smooth out uneven distribution across physical machines."
- "This is mainly needed because the cluster size can change over time."

