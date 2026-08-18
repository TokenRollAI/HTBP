# HTBP

HTBP（HTTP ToolBridge Protocol）是一套面向 Agent 的 HTTP 工具发现与调用协议。它把工具、
上下文和远端服务组织成一棵按身份裁剪的路径树；Agent 通过普通 HTTP 就能逐层发现节点、读取
schema 并调用工具。

当前规范版本是 `0.1`，wire contract 以
[RFC-0001：HTBP Core 0.1](docs/rfcs/RFC-0001-htbp-core.md) 为准。

## Core surface

```txt
GET  /<path>/~help              节点或单工具 Help
GET  /<path>/~tree?depth=N      深度受限的可见树
POST /<node>/<tool>             直接调用，body = arguments
POST /<node>                    信封调用，body = {tool, arguments}
```

Core 同时规定：

- `application/json`、`text/plain` 和默认 Markdown 三种 Help 表现；
- 节点索引到单工具完整 schema 的两级披露；
- Bearer 身份、path action、deny 优先和不可见路径返回 `404`；
- `{code,message,retryable}` 统一错误；
- 深度/节点有界的 `~tree` 与通用分页 envelope。

`~describe`、`~search`、`~feedback` 和 remote federation 可以形成可选协议 profile；
`~register`、`~authorize` 与 `~mcp` 记录 tool-bridge 的产品/adapter surface；`~skill` 目前只是
保留扩展。保留 endpoint 不等于实现支持，客户端只应在 capability 或 Help 明确声明后使用。

## 文档

- [RFC-0001：HTBP Core 0.1](docs/rfcs/RFC-0001-htbp-core.md)
- [核心设计决策](docs/design/core-decisions.md)
- [接入设计：Provider 与 Agent](docs/design/provider-agent-integration.md)

## 边界

HTBP 定义 Agent 看到的 HTTP 协议，不定义：

- 上游 MCP、REST、CLI、SaaS、Plugin 或设备的内部传输；
- Provider 的存储、部署、Dashboard 或管理实现；
- OAuth、MCP 或 JSON Schema 本身；
- 所有实现都必须支持的节点 kind 或可选扩展。

## Reference implementation

[TokenRollAI/tool-bridge](https://github.com/TokenRollAI/tool-bridge) 是 HTBP 0.1 的参考实现和当前
wire 行为基线。规范与实现冲突时应先核对当前代码与测试，再在同一轮修正规范或实现，不能让两者
长期漂移。
