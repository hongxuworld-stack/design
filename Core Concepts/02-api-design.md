# API Design

## 是什么

API Design 定义 clients 如何和 system 交互。面试里的目标不是设计完美 API，而是展示清晰的 resource boundary，然后快速进入更重要的 architecture 部分。

## 默认选择

- 大多数面试系统用 REST。
- 围绕 resources 设计 APIs。
- 自然使用 HTTP methods: `GET`、`POST`、`PUT/PATCH`、`DELETE`。
- 只列核心 flow 需要的关键 endpoints。

## 什么时候提

- requirements 涉及 user actions、reads、writes、search、booking、upload 或 feed retrieval。
- 面试官要求 API shape。
- 你需要澄清 system boundaries。

## 常见 API 点

- 大结果集需要 pagination。
- 持续变化的 feed 或 timeline 更适合 cursor pagination。
- 简单静态列表可以用 offset pagination。
- 用户登录态可以用 user session tokens。
- service-to-service calls 可以用 API keys 或 service credentials。
- 容易被 abuse、bots、scraping 或 expensive endpoints 需要 rate limiting。

## Tradeoffs

- API 细节讲太久会吃掉面试时间。
- Cursor pagination 更适合 live data，但复杂度略高。
- REST 兼容性强；GraphQL 对 flexible clients 有帮助，但会让 caching 和 authorization 更复杂。

## 面试表达

- "I will define the 4 or 5 APIs needed for the main flows and keep details light."
- "This list endpoint should use cursor pagination because new items can appear while the user is paging."
- "For abuse-sensitive endpoints, I would put rate limiting at the API gateway."

