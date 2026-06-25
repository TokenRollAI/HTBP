# HTBP 架构草案

本仓库当前包含 HTTP ToolBridge Protocol（HTBP）的架构设计草案，以及一版可部署到 Cloudflare Workers 的 MCP Streamable HTTP bridge 原型。

- [RFC-0001：HTBP 核心协议](docs/rfcs/RFC-0001-htbp-core.md)
- [核心设计决策](docs/design/core-decisions.md)
- [接入设计：Provider 与 Agent](docs/design/provider-agent-integration.md)

## MCP HTTP Bridge

本原型把远端 MCP Streamable HTTP server 的 `tools/list` 和 `tools/call` 暴露成普通 HTTP endpoint，并为 configured server 生成 `~help` / `~skill`。

```txt
GET  /api/servers
GET  /api/servers/{server}/tools
POST /api/servers/{server}/tools/{tool}/call
GET  /mcp/{server}/~help
GET  /mcp/{server}/~skill
POST /mcp/{server}/tools/{tool}
```

本阶段不支持 stdio，也不支持 SSE fallback；只面向 MCP Streamable HTTP。

### 本地开发

```bash
npm install
npm run dev
```

Worker API 本地运行：

```bash
npm run build
npm run worker:dev
```

### 配置 MCP server

通过 `MCP_SERVERS_JSON` 配置 server。非 secret 可以放在 `wrangler.jsonc` vars；secret 应通过 `.dev.vars` 或 `wrangler secret put` 注入。

```json
{
  "context7": {
    "name": "Context7",
    "endpoint": "https://mcp.context7.com/mcp",
    "description": "Documentation MCP server",
    "headers": {
      "Authorization": "Bearer ${CONTEXT7_TOKEN}"
    }
  }
}
```

### Bridge auth

如果不配置 `AUTH_BEARER_TOKEN` 或 `OAUTH_ISSUER`，bridge 处于 unauthenticated mode。

静态 bearer gate：

```bash
wrangler secret put AUTH_BEARER_TOKEN
```

OAuth JWT verification：

```txt
OAUTH_ISSUER=https://issuer.example.com
OAUTH_REQUIRED_AUDIENCE=htbp
OAUTH_JWKS_URI=https://issuer.example.com/.well-known/jwks.json
```

`OAUTH_JWKS_URI` 可省略；Worker 会读取 issuer 的 OpenID configuration 来发现 `jwks_uri`。
