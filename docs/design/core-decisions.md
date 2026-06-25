# 核心设计决策

状态：Draft
日期：2026-06-25
相关文档：

- [RFC-0001：HTBP Core](../rfcs/RFC-0001-htbp-core.md)
- [接入设计：Provider 与 Agent](provider-agent-integration.md)

## 1. 协议边界

HTBP 不是一套新的业务 API 规范。它只定义 Provider 如何在已有 HTTP API 旁边暴露 Agent 可读的控制面。

HTBP 只标准化三类 endpoint：

```txt
GET  {path}/~help
GET  {path}/~skill
POST {path}/~register
```

其中 `~help` 是核心必选能力，`~skill` 是推荐能力，`~register` 是可选能力。

HTBP 不定义：

```txt
业务 Resource 的命名
业务 Path 的层级
业务 HTTP Method 的语义
业务 response shape
业务 error shape
业务 pagination / idempotency / retry / streaming 规则
```

这些内容全部由 Provider 原有 API 决定。HTBP 只描述它们。

## 2. 为什么使用 `~` Control Segment

HTBP 采用末尾 `~` segment 作为 control plane 标记：

```txt
/api/repos/~help
/api/repos/~skill
/api/repos/{owner}/{repo}/~help
```

这个设计有三个目的：

```txt
1. 避免占用常见业务 Path，例如 /help、/skill。
2. 允许在任意 Resource Path 下进行 local discovery。
3. 让 Agent 通过简单后缀规则构造 control endpoint。
```

Provider 一旦在某个 Path namespace 采用 HTBP，就应该把精确末尾段 `~help`、`~skill`、`~register` 视为保留段。

## 3. 为什么不要求 `/.well-known/htbp`

`/.well-known/` 适合 origin-level metadata，但 HTBP 的核心是 resource-local metadata。

Agent 可能被用户直接带到某个具体 Path：

```txt
https://api.example.com/api/repos/octo/demo
```

此时最有价值的问题不是“这个 origin 有哪些所有能力”，而是：

```txt
这个 Resource Path 附近能做什么？
我应该怎样调用这里的 API？
有没有更深的 Skill 指南？
```

所以 HTBP 的核心 discovery 是：

```txt
GET /api/repos/octo/demo/~help
```

未来可以补充 origin-level discovery，但它不属于当前核心。

## 4. Help 与 Skill 的职责分工

Help 是 command index，目标是短、准、可直接进 context。

Help 应回答：

```txt
有哪些 command？
每个 command 用哪个 Method 和 Path？
需要哪些 Query / Header / JSON Body？
需要什么 auth / scope？
成功 response 大概是什么？
是否有 write/delete/external effect？
是否需要 confirmation？
更深的 Skill 在哪里？
```

Skill 是 operational guidance，目标是教 Agent 做对复杂流程。

Skill 应回答：

```txt
什么时候使用这些 endpoint？
多步 workflow 的顺序是什么？
哪些操作危险？
哪些错误常见？
如何处理 pagination / long-running operation？
什么时候需要用户确认？
```

一个简单只读调用应该只靠 Help 完成。只有当 task 需要 judgment 或 workflow 时，Agent 才应该读取 Skill。

## 5. Auth 与 OAuth 设计

HTBP 不发明新的 auth protocol。

基础调用使用标准 header：

```http
Authorization: Bearer <token>
```

Agent 身份可以用可选 header 表达：

```http
X-Agent-Id: <agent-id>
```

`X-Agent-Id` 不是 credential，只用于 audit、attribution 或 policy hint。

如果 Provider 使用 OAuth，HTBP 复用既有 OAuth metadata：

```txt
Help:
  auth oauth2
  oauth_resource <protected-resource-metadata-url>

401:
  WWW-Authenticate: Bearer resource_metadata="<protected-resource-metadata-url>"
```

`~register` 是可选 onboarding endpoint。它可以封装或指向 OAuth Dynamic Client Registration，也可以指向 Provider 自定义接入流程。

## 6. Agent 消费流程

最小 Agent flow：

```txt
1. 拿到 Domain 或 Resource Path。
2. 请求 {path}/~help。
3. 选择满足用户意图的最小 command。
4. 按 Help 构造 Provider 原生 HTTP request。
5. 必要时读取 {path}/~skill。
6. 执行业务 API。
7. 按 Provider 原有 response / error format 解释结果。
```

Agent 不应该一次性抓完整 Domain 的所有 Skill。HTBP 默认是 progressive disclosure。

## 7. 当前已定决策

| 主题 | 决策 |
| --- | --- |
| 控制面寻址 | 使用末尾 `~help`、`~skill`、`~register` |
| Resource 形态 | 不限制，由 Provider 自己定义 |
| 业务 Method | 不限制，沿用 Provider 原有 API |
| Help 格式 | `text/plain` 紧凑 DSL |
| Skill 格式 | Markdown，推荐 `text/markdown` |
| Auth | 标准 `Authorization: Bearer` |
| Agent 标识 | 可选 `X-Agent-Id` |
| OAuth | 复用 OAuth metadata，不自创 |
| Register | 可选 endpoint，只定义宽泛用途 |
| Discovery 方式 | resource-local，progressive disclosure |
| `OPTIONS` / `HEAD` | 不作为 discovery 依赖 |
| `.well-known` | 当前不要求，留给后续 RFC |

## 8. 当前待讨论问题

以下问题不阻塞核心协议，但需要后续拍板：

```txt
Help DSL 是否需要 formal grammar？
是否提供 JSON Help 作为并列表达？
是否注册 HTBP 专用 media type？
是否定义 origin-level discovery？
是否为 streaming / long-running operation 定义标准 hint？
是否需要 conformance test suite？
```

