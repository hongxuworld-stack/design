# Networking Essentials

## 是什么

Networking 关注 clients 和 services 如何通信、request 如何在系统里流动，以及当网络变慢、过载或断开时系统会发生什么。

## 默认选择

- 大多数 public-facing request/response APIs 用 HTTP over TCP。
- 除非系统有特殊需求，否则用 REST 或简单 JSON APIs。
- 内部 service-to-service calls 如果很在意 latency 和 serialization efficiency，可以用 gRPC。
- 只需要 server 单向推送时，用 SSE。
- 需要真正 bidirectional real-time communication 时，用 WebSockets。

## 什么时候提

- 系统需要 real-time updates。
- services 要跨 regions 或 data centers 通信。
- 需要 load balancing。
- 设计里有 long-lived client connections。
- latency 是产品需求的一部分。

## Tradeoffs

- HTTP 简单、通用，但比 binary internal protocols 效率低。
- SSE 比 WebSockets 简单，但只支持 server-to-client push。
- WebSockets 支持双向通信，但要管理 stateful connections。
- Layer 7 load balancers 可以基于 HTTP 内容路由；Layer 4 load balancers 对 TCP connections 更简单、更快。
- Multi-region systems 能降低用户 latency，但会增加 replication 和 consistency 的复杂度。

## 面试表达

- "I would start with HTTP/REST externally because it is simple and standard."
- "For internal high-throughput service calls, gRPC could reduce serialization overhead."
- "If the client only needs server push, SSE is enough; I would use WebSockets only for bidirectional updates."
- "Persistent connections change the load-balancing story because the server now owns connection state."

