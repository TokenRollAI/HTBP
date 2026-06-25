# RFC-0001：HTBP Core

状态：Draft
日期：2026-06-25
类别：Core Protocol
目标读者：Agent runtime、tool gateway implementer、CLI adapter author、REST API Provider、MCP adapter author

## Abstract

HTTP ToolBridge Protocol（HTBP）定义一套很小的 HTTP 原生约定，使 Agent 可以在不安装本地 CLI、不接入 MCP client、不依赖特定 SDK 的情况下，发现并使用以 HTTP 形式暴露的工具资源。

HTBP 不定义业务 Resource 应该长什么样。HTBP 只在 Provider 选择采用的 Path 下保留一个极小的 control plane：

```txt
GET  {path}/~help      面向 Agent 的紧凑 Help DSL
GET  {path}/~skill     面向 Agent 的 Markdown Skill
POST {path}/~register  可选的 Register 入口
```

真正执行工具能力的仍然是 Provider 已有的 HTTP API：

```txt
Domain + Path + Query + JSON Body + HTTP Method
```

HTBP 标准化的是：Agent 如何知道有哪些调用、如何组织参数、如何处理 auth，以及什么时候读取更详细的 Skill。HTBP 不标准化业务 API 本身。

## 1. Introduction

现代 Agent 的运行环境差异很大。有些环境有完整 shell、本地包管理器、MCP 支持和持久化凭据；另一些环境只有最小的 HTTP 能力，例如 `curl` 或 Web Fetch。

现有工具接入方式各有成本：

```txt
CLI tool 依赖本地 binary、runtime、env、working directory 和 filesystem 假设。
MCP server 依赖 MCP runtime 支持，并占用结构化 tool surface。
Native integration 绑定到特定 Agent platform，可移植性弱。
```

HTBP 提供第四种方式：

```txt
把已有 tool 和 API 暴露成 HTTP Resource。
把使用说明放在 Resource 旁边。
让 Agent 用普通 HTTP call 完成操作。
```

核心原则是：

```txt
如果 Agent 可以 fetch 一个 URL，它就应该可以学习如何使用这个 URL 对应的 tool Resource。
```

## 2. Requirements Language

本文中的“必须”“不得”“应该”“不应该”“可以”“可选”等词用于表达协议要求。本文是中文草案，不使用大小写区分规范等级；对应含义参考 RFC 2119 与 RFC 8174。

## 3. Goals

HTBP 的目标如下：

```txt
G1. 让 Agent 仅通过 HTTP 就能发现如何使用 tool Resource。
G2. 让 Provider 的最小实现足够小。
G3. 保留 Provider 已有 API 的 Path、Method、Query、Request Body、Response Body 和 error format。
G4. 支持 progressive disclosure：先读局部 Help，必要时再读更深的 Skill。
G5. Help 使用紧凑文本；Skill 使用 Markdown。
G6. 复用既有 HTTP 与 OAuth 机制，不重新发明 auth protocol。
G7. 不依赖 OPTIONS、HEAD、本地 CLI、package manager 或 MCP tool slot。
```

## 4. Non-Goals

HTBP 明确不定义以下内容：

```txt
N1. 通用 Resource model。
N2. OpenAPI 的替代品。
N3. OAuth 的替代品。
N4. MCP 的替代品。
N5. 所有 operation 都必须使用的 JSON Schema language。
N6. 业务 API 的 pagination、idempotency、side effect、retry、streaming 或 error semantics。
N7. 业务 Resource 必须遵循的 Path shape。
```

Provider 可以在 `~help` 或 `~skill` 中描述 pagination、idempotency、安全性、retry、streaming 和 error convention，但这些是 Provider 自己的 API contract，不是 HTBP 的 protocol requirement。

## 5. Terminology

### 5.1 Domain

Domain 是 Provider 采用 HTBP 的 HTTP origin 和可选 base path。

示例：

```txt
https://api.example.com
https://api.example.com/api
https://tools.example.com/acme
```

HTBP 不要求 Domain 下所有 Path 都实现 HTBP。Provider 可以只在选定 Path namespace 下采用 HTBP。

### 5.2 Resource Path

Resource Path 是任何由 Provider 拥有的 HTTP Path。HTBP 不定义 Resource Path 的形态或含义。

示例：

```txt
/api/repos
/api/repos/{owner}/{repo}
/api/search
/mcp/github
```

### 5.3 Control Segment

Control Segment 是以 `~` 开头、位于 Path 末尾的保留 segment。它用于 HTBP control plane。

本文定义以下 Control Segment：

```txt
~help
~skill
~register
```

本文只定义上述精确的末尾 segment。其他 `~` segment 保留给未来 HTBP extension 或 Provider 自定义 control document。

### 5.4 Help

Help 是面向 Agent 的紧凑 command metadata。Help 回答以下问题：

```txt
这里有哪些可调用能力？
应该使用哪个 Method、Path、Query、Header 和 JSON Body？
需要什么 auth 和 scope？
更深的 Skill 在哪里？
```

### 5.5 Skill

Skill 是 Markdown 格式的使用指南。它用于改进 Agent 在多步任务、风险操作或非显然调用中的行为。Skill 回答以下问题：

```txt
Agent 应该如何正确使用这些 call？
调用顺序是什么？
哪些行为应该避免？
常见错误如何恢复？
```

### 5.6 Register

Register 是可选的 onboarding 入口。它可以用于获取 API token、注册 OAuth client、启动人工 approval flow，或返回外部 registration mechanism 的指针。

HTBP 只定义 `~register` 的位置和宽泛用途，不要求 Provider 实现开放注册。

## 6. Protocol Overview

HTBP 把 Provider 现有 API 分成两个 plane：

```txt
Control Plane:
  GET  {path}/~help
  GET  {path}/~skill
  POST {path}/~register

Data Plane:
  Provider 已有 HTTP API
```

Control Plane 告诉 Agent 如何使用 Data Plane；Data Plane 执行真正的业务操作。

示例：

```txt
GET /api/repos/~help

GET /api/repos?owner=octo&limit=20
POST /api/repos
GET /api/repos/octo/demo/~help
PATCH /api/repos/octo/demo
```

`/api/repos` 与 `/api/repos/octo/demo` 是 Provider 定义的业务 API。HTBP 只定义 `/~help`、`/~skill` 和可选 `/~register` 的含义。

## 7. Control Segment URI Convention

### 7.1 Basic Form

对任意被 Provider 采用的 Resource Path `P`，HTBP control endpoint 为：

```txt
GET  P + "/~help"
GET  P + "/~skill"
POST P + "/~register"
```

示例：

```txt
GET  /~help
GET  /~skill
POST /~register

GET  /api/~help
GET  /api/repos/~help
GET  /api/repos/~skill
GET  /api/repos/octo/demo/~help
```

### 7.2 Control Document Scope

Control document 作用于 control segment 前面的那个 Resource Path。

示例：

```txt
/api/~help
  描述 /api Path namespace。

/api/repos/~help
  描述 /api/repos Resource 或 Resource namespace。

/api/repos/octo/demo/~help
  描述 /api/repos/octo/demo Resource context。
```

Provider 可以自行选择粒度。它可以只在 coarse namespace 暴露 Help，也可以只在具体 Resource 暴露 Help，或者两者都提供。

### 7.3 Query Handling

HTBP control endpoint 附着在 Path 上，而不是附着在已经带 Query 的完整 URL 上。

Agent 不应该把 `~help` 或 `~skill` 追加在 query string 后面。Query 参数属于 Provider data-plane operation，应在 Help 中描述。

正确：

```txt
GET /api/search/~help
```

错误：

```txt
GET /api/search?q=abc/~help
```

### 7.4 Existing Path Conflicts

当 Provider 在某个 Path namespace 采用 HTBP 时，末尾 segment `~help`、`~skill` 和 `~register` 在该 namespace 内保留给 HTBP control document。

如果 Provider 已经把其中某个精确末尾 segment 用于业务数据，应该在另一个相邻 Path、父 Path、子 Path 或独立 host 上采用 HTBP。

这个约定避免限制普通业务 Resource，同时给 Agent 一个稳定的 suffix pattern。

### 7.5 Relation To Well-Known URI

HTBP 不要求 `/.well-known/htbp`。

well-known URI 适合 origin-level metadata；HTBP 需要的是 resource-local metadata，因为 Agent 可能从任意 Provider Path 进入，并询问该 Path 该如何使用。因此 HTBP 使用末尾 `~` control segment，而不是只使用 root-level discovery prefix。

Provider 未来可以额外暴露类似 `/.well-known/htbp` 的 origin-level entry point，但本文不要求它。

## 8. Help Endpoint

### 8.1 Request

Agent 通过以下方式发现 Help：

```http
GET {path}/~help HTTP/1.1
Accept: text/plain
Authorization: Bearer <token>
X-Agent-Id: <optional-agent-id>
```

当 Provider 有意公开 Help 时，auth 可以省略。

### 8.2 Response Media Type

成功的 Help response 应该使用：

```http
Content-Type: text/plain; charset=utf-8
```

未来可以注册 HTBP 专用 media type，但 Agent 必须能够消费 `text/plain`。

### 8.3 Help DSL

Help DSL 是紧凑的 line-oriented text format。它优先服务于 LLM 直接读取，而不是完整 machine validation。

Help 应该以简短 header 开始：

```txt
htbp draft
resource /api/repos
title Repository tools
skill ./~skill
auth bearer
```

随后列出 command。command 是 Provider 已有 HTTP operation，由 Method、name 和 Path template 标识：

```txt
cmd list GET /api/repos
  q owner string required
  q limit integer optional default=20 max=100
  auth oauth2 scope=repo.read
  returns 200 application/json

cmd create POST /api/repos
  body application/json {"name":string,"private":boolean?}
  auth oauth2 scope=repo.write
  effect write
  confirm false
  returns 201 application/json
```

### 8.4 DSL Line Types

本文定义以下 line type：

```txt
htbp <revision>
resource <path-or-template>
title <short-title>
summary <short-summary>
skill <relative-or-absolute-url>
auth <none|bearer|oauth2>
oauth_resource <url>
register <relative-or-absolute-url>
cmd <name> <METHOD> <path-template>
q <name> <type> <required|optional> [attributes...]
h <name> <type> <required|optional> [attributes...]
body <media-type> <shape-or-reference>
returns <status> <media-type> [summary...]
scope <scope-token>
effect <read|write|delete|external|unknown>
confirm <true|false|provider|user>
rate <description>
note <short-note>
link <rel> <relative-or-absolute-url>
```

Agent 必须忽略未知 line type，除非未来 HTBP extension 通过显式 capability marker 把某类行定义为强制语义。

### 8.5 Types

DSL 的 type field 保持很小：

```txt
string
integer
number
boolean
object
array
file
json
```

Provider 可以在 attribute 中添加细化约束：

```txt
q since string optional format=date-time
q limit integer optional min=1 max=100
q status string optional enum=open,closed,all
```

### 8.6 JSON Body Shape

`body` 行用于紧凑描述 JSON Body。它不是完整 schema language。

示例：

```txt
body application/json {"name":string,"description":string?}
body application/json ref=./schemas/create-repo.json
body application/json see=./~skill#create
```

当 Body 复杂时，Provider 应该把完整说明放到 `~skill`，Help 保持紧凑。

### 8.7 Auth In Help

Help 应该在 document level 和 command level 声明所需 auth：

```txt
auth bearer

cmd delete DELETE /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.delete
```

command-level auth 覆盖 document-level auth。

`auth` 是给 Agent 的提示。Provider runtime 的实际 authorization decision 仍然是最终权威。

### 8.8 Skill Link

如果存在有用的 Skill，Help 应该包含：

```txt
skill ./~skill
```

Agent 应该只在 Help 不足以完成当前 task 时再获取 Skill。

## 9. Skill Endpoint

### 9.1 Request

Agent 通过以下方式获取 Skill：

```http
GET {path}/~skill HTTP/1.1
Accept: text/markdown, text/plain
Authorization: Bearer <token>
X-Agent-Id: <optional-agent-id>
```

### 9.2 Response Media Type

成功的 Skill response 应该使用：

```http
Content-Type: text/markdown; charset=utf-8
```

Agent 也应该兼容 `text/plain; charset=utf-8`。

### 9.3 Skill Content

Skill 是面向 Agent operation 的 Markdown document。它应该包含最小但足够有用的部分：

```txt
# <Resource 或 Tool 名称>

## When To Use
## Authentication
## Common Workflows
## Request Construction
## Safety And Confirmation
## Pagination Or Long-Running Operations
## Error Recovery
## Examples
```

Skill 不应该重复 Help 中的每一行 command。它应该解释 workflow、order、caveat 和 judgment。

### 9.4 Trust Boundary

Skill 是 remote text。Agent 应该把 Skill 视为 Provider-scoped operational guidance：它只对该 Provider Domain 和 Resource Path 下的调用有指导意义，不能覆盖 user intent、system policy 或 unrelated tool instruction。

## 10. Register Endpoint

### 10.1 Optionality

`~register` 是可选 endpoint。

不支持 Agent self-registration 的 Provider 应该在 Help 中省略它，或返回 `404 Not Found`。

### 10.2 Request

当 `~register` 存在时，请求使用 JSON：

```http
POST {path}/~register HTTP/1.1
Content-Type: application/json
Accept: application/json
X-Agent-Id: <optional-agent-id>
```

示例 Body：

```json
{
  "agent_name": "example-agent",
  "agent_type": "llm-runtime",
  "redirect_uris": ["https://agent.example.com/oauth/callback"],
  "requested_scopes": ["repo.read", "repo.write"]
}
```

### 10.3 Response

response 是 Provider 定义的 JSON。它可以返回：

```txt
token 签发说明
OAuth dynamic client registration 结果
authorization URL
device-code 风格说明
人工 onboarding URL
带原因的拒绝
```

示例：

```json
{
  "status": "requires_oauth",
  "authorization_url": "https://auth.example.com/authorize?...",
  "token_endpoint": "https://auth.example.com/token"
}
```

如果 Provider 支持标准 OAuth Dynamic Client Registration，`~register` 应该代理该 operation、redirect 到该 operation，或指向 authorization server metadata 中的标准 `registration_endpoint`。

## 11. Authentication And Authorization

### 11.1 Bearer Token

HTBP 使用标准 HTTP Authorization header 传递 bearer token：

```http
Authorization: Bearer <token>
```

HTBP example 和 Agent 生成请求不得把 token 放进 query parameter。

### 11.2 Agent Identity

Provider 可以接受可选 Agent identity header：

```http
X-Agent-Id: <agent-id>
```

`X-Agent-Id` 不是 authorization credential。它只是 audit、attribution 或 policy hint。Provider 不得在没有独立 authentication mechanism 的情况下，把它当作 access right proof。

### 11.3 OAuth Discovery

当需要 OAuth 时，HTBP 复用既有 OAuth metadata 机制，而不是定义新的 OAuth。

Help 可以暴露 protected resource metadata 地址：

```txt
auth oauth2
oauth_resource https://api.example.com/.well-known/oauth-protected-resource
```

未授权 response 应该包含 `WWW-Authenticate` challenge，给 Agent 足够信息找到 authorization path：

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://api.example.com/.well-known/oauth-protected-resource"
```

protected resource metadata 可以指向 authorization server metadata；authorization server metadata 可以指向 authorization endpoint、token endpoint 和可选 registration endpoint。

### 11.4 Scope

Help 应该在 command 附近声明 scope：

```txt
cmd read GET /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.read

cmd update PATCH /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.write
```

scope string 由 Provider 定义。HTBP 不分配全局 scope registry。

### 11.5 Public Help

Provider 可以公开 `~help` 和 `~skill`，但应该避免暴露敏感 operational detail，除非这是有意公开的内容。

Provider 也可以返回 credential-specific Help。例如，只读 token 收到的 Help 可以省略 write operation。

## 12. Caching

Provider 应该在 `~help` 和 `~skill` 上支持普通 HTTP caching header：

```http
ETag: "abc123"
Cache-Control: private, max-age=300
```

Agent 在可行时应该使用 conditional request：

```http
If-None-Match: "abc123"
```

HTBP 不要求 `HEAD`。conditional `GET` 已足够。

## 13. Errors

HTBP 不替换业务 API 的 error format。

对 control endpoint，Provider 应该使用普通 HTTP status code：

```txt
200 OK              返回 control document
304 Not Modified    cached control document 仍然有效
401 Unauthorized    缺少 credential 或 credential 无效
403 Forbidden       credential 有效但权限不足
404 Not Found       该 Path 未暴露 HTBP
406 Not Acceptable  请求 format 不支持
429 Too Many Requests
500 Server Error
```

control endpoint 的 error body 默认应该是简短文本：

```txt
error unauthorized
detail bearer token required
help /api/repos/~help
```

当 client 请求 JSON 时，Provider 可以返回 JSON。

## 14. Progressive Discovery

HTBP 面向 progressive context loading。

Agent 应该遵循以下顺序：

```txt
1. 从用户、runtime 或已有 context 提供的 Domain 或 Resource Path 开始。
2. 获取 {path}/~help。
3. 使用 Help 构造完成 task 所需的最小业务 API call。
4. 只有当 Help 不足或 operation 有风险时，才获取 {path}/~skill。
5. 只有在需要时，才沿 Help 中的 link 进入更具体 Path。
```

Provider 应该保持 Help 紧凑，把深层 operational guidance 放到 Skill 中。

## 15. Conformance

### 15.1 Minimal Provider

最小 HTBP Provider 必须：

```txt
至少在一个 adopted path 支持 GET {path}/~help
以 UTF-8 text 返回 Help
在 adopted path 保留精确末尾 segment ~help
描述真实的 Provider HTTP call，并不改变其语义
```

### 15.2 Useful Provider

实用 HTBP Provider 应该：

```txt
在 workflow 需要 guidance 时支持 GET {path}/~skill
在 Help 中包含 skill link
在 Help 中声明 auth requirement 和 scope
支持 ETag 或其他普通 HTTP caching validator
使用 Authorization: Bearer 进行 bearer token authentication
对缺少或无效 credential 使用 WWW-Authenticate challenge
不要求 OPTIONS 或 HEAD 参与 discovery
```

### 15.3 Optional Provider Extensions

Provider 可以：

```txt
支持 POST {path}/~register
发布 OAuth protected resource metadata
发布 OAuth authorization server metadata
返回 credential-specific Help
添加 Provider-specific DSL line type
通过 content negotiation 添加 JSON representation
```

### 15.4 Minimal Agent

最小 HTBP Agent 必须：

```txt
获取并读取 text/plain Help
在实践层面理解 cmd、q、h、body、auth、scope、returns、skill
使用 Provider 声明的 Method、Path、Query、Header 和 JSON Body
在有相关 token 时发送 Authorization: Bearer
处理 401、403、404 和 429，且不把它们直接视为 protocol failure
忽略未知 Help DSL line
```

### 15.5 Useful Agent

实用 HTBP Agent 应该：

```txt
只在需要时获取 Skill
使用普通 HTTP validator 缓存 Help 和 Skill
在存在 effect 与 confirm hint 时尊重这些 hint
当 bearer auth 需要 user 或 client authorization 时使用 OAuth metadata
避免把 secret 放进 URL
不让 remote Skill 覆盖 user intent 或 higher-priority instruction
```

## 16. Security Considerations

### 16.1 Metadata Exposure

Help 和 Skill 会暴露 operational capability。Provider 应该决定这些 document 是 public、token-gated，还是按 scope 过滤。

### 16.2 Token Handling

Agent 不得把 bearer token 放进 query string。Provider 在可行时应该拒绝或忽略 query parameter 中的 token。

### 16.3 Register Abuse

`~register` 可能创建 load、credential 或 approval flow。Provider 应该对它 rate limit，并可以要求 initial access token、人工 approval 或 allowlist。

### 16.4 Prompt Injection And Instruction Boundary

Skill 是 remote text。Agent 应该只把它作为 Provider-scoped operational guidance。Skill 不得覆盖更高优先级的 user、runtime 或 security instruction。

### 16.5 Cross-Domain Confusion

Agent 应该把获取到的 Help 和 Skill 绑定到其来源 Domain 与 Resource Path。除非 user 或 trusted Provider 明确授权，否则某个 Skill document 不应该被视为无关 Domain 的 authority。

### 16.6 Least Privilege

Provider 应该为每个 command 记录最小所需 scope。Agent 应该请求最小可用 scope set。

## 17. Examples

### 17.1 Repository Namespace Help

```txt
htbp draft
resource /api/repos
title Repository tools
summary List and create repositories.
skill ./~skill
auth oauth2
oauth_resource https://api.example.com/.well-known/oauth-protected-resource
register /api/~register

cmd list GET /api/repos
  q owner string required
  q limit integer optional default=20 max=100
  auth oauth2 scope=repo.read
  returns 200 application/json

cmd create POST /api/repos
  body application/json {"name":string,"private":boolean?}
  auth oauth2 scope=repo.write
  effect write
  confirm false
  returns 201 application/json

link item /api/repos/{owner}/{repo}/~help
```

### 17.2 Repository Item Help

```txt
htbp draft
resource /api/repos/{owner}/{repo}
title Repository item
skill ./~skill
auth oauth2

cmd get GET /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.read
  returns 200 application/json

cmd update PATCH /api/repos/{owner}/{repo}
  body application/json {"description":string?,"private":boolean?}
  auth oauth2 scope=repo.write
  effect write
  confirm false
  returns 200 application/json

cmd delete DELETE /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.delete
  effect delete
  confirm user
  returns 204 application/json
```

### 17.3 Skill Skeleton

```md
# Repository Tools

## When To Use

当用户需要查看或管理 repository 时使用这些 endpoint。

## Common Workflows

创建 repository 时，先调用 `POST /api/repos` 并传入 `name`。随后调用
`GET /api/repos/{owner}/{repo}` 验证结果状态。

## Safety And Confirmation

delete operation 需要用户明确确认。不要从含糊的 cleanup request 中推断 delete intent。
```

## 18. IANA Considerations

本文不请求 IANA registration。

未来 RFC 可以注册 HTBP 专用 media type 或 well-known URI，前提是协议后续定义 origin-level discovery document。

## 19. References

### 19.1 Normative References

- RFC 2119：用于表示 requirement level 的关键词。https://www.rfc-editor.org/rfc/rfc2119
- RFC 8174：RFC 2119 keyword 大小写歧义。https://www.rfc-editor.org/rfc/rfc8174
- RFC 6750：OAuth 2.0 Bearer Token Usage。https://www.rfc-editor.org/rfc/rfc6750

### 19.2 Informative References

- RFC 7591：OAuth 2.0 Dynamic Client Registration Protocol。https://www.rfc-editor.org/rfc/rfc7591
- RFC 8414：OAuth 2.0 Authorization Server Metadata。https://www.rfc-editor.org/rfc/rfc8414
- RFC 8615：Well-Known Uniform Resource Identifiers。https://www.rfc-editor.org/rfc/rfc8615
- RFC 9728：OAuth 2.0 Protected Resource Metadata。https://www.rfc-editor.org/rfc/rfc9728

