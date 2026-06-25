# HTBP

HTBP（HTTP ToolBridge Protocol）是一个面向 Agent 的轻量 HTTP tool discovery 约定。它关注的是 Provider 如何在现有 HTTP API 旁边暴露 `~help` 和 `~skill`，让只有 HTTP 能力的 Agent 能理解并调用工具。

本仓库只保存 protocol RFC、设计决策和接入说明，不包含具体 server 实现。

## 文档

- [RFC-0001：HTBP 核心协议](docs/rfcs/RFC-0001-htbp-core.md)
- [核心设计决策](docs/design/core-decisions.md)
- [接入设计：Provider 与 Agent](docs/design/provider-agent-integration.md)

## 边界

HTBP 定义：

- `GET {path}/~help`
- `GET {path}/~skill`
- 可选的 register/auth discovery 约定
- Help DSL 和 Skill Markdown 的语义

HTBP 不定义：

- Provider 的业务 Resource 形状
- Provider 应该使用哪些 HTTP method
- pagination、idempotency、retry 等业务 API contract
- 具体 MCP、CLI 或 SaaS API adapter 的实现方式

## Tool Bridge

MCP Streamable HTTP 到普通 HTTP call 的 Cloudflare Worker 实现已经拆到独立项目：

```txt
https://github.com/TokenRollAI/tool-bridge
```

该项目是 HTBP 的一个实现方向，不是 HTBP protocol 本身。
