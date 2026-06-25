# 接入设计：Provider 与 Agent

状态：Draft
日期：2026-06-25
相关 RFC：[RFC-0001：HTBP Core](../rfcs/RFC-0001-htbp-core.md)

## 1. Purpose

本文描述 Provider 和 Agent 如何在架构层面接入 HTBP。本文不定义任何具体 server、gateway、SDK、MCP adapter 或 CLI wrapper 的实现细节。

核心原则是：

```txt
保持 Provider 已有 API 不变。
在 Resource 旁边增加一个很小的 control plane，用来教 Agent 如何调用它。
```

## 2. Architecture Split

HTBP 接入包含三个关注点：

```txt
Provider Data Plane
  已有 HTTP API。HTBP 不定义它。

Provider Control Plane
  ~help、~skill、可选 ~register。

Authorization Plane
  Bearer token 和可选 OAuth metadata。复用既有标准。
```

最重要的边界是：`~help` 和 `~skill` 描述 Provider 的 API；它们本身不是业务 API。

## 3. Provider Integration

### 3.1 选择 adopted path

Provider 自行选择在哪些 Path 暴露 HTBP。

示例：

```txt
/~help
/api/~help
/api/repos/~help
/api/repos/{owner}/{repo}/~help
/mcp/github/~help
```

Provider 不需要在每个 Path 都暴露 HTBP。可以先从一个 coarse path 开始，后续再逐步增加更具体的 Help。

### 3.2 保留 Control Segment

对每个 adopted path namespace，保留以下末尾 segment：

```txt
~help
~skill
~register
```

如果真实业务 Resource 已经使用精确末尾 segment `~help`，就不要在该 namespace 挂载 HTBP。可以挂载到上一级、下一级或另一个 host/base path。

### 3.3 把 Help 写成 command index

Help 应该足够短，使 Agent 能在行动前读完。

Help 应该包含：

```txt
Resource Path 或 template
短 title 和 summary
Skill link
auth model
需要时给出 OAuth protected resource metadata pointer
需要时给出 Register pointer
以 METHOD + path template 表达的 command
query parameter
必要 header
JSON Body shape
scope
response media type
必要时给出 effect 和 confirmation hint
```

Help 应该避免：

```txt
长教程
完整复制业务文档
OpenAPI 级别的完整 schema
Provider 营销文案
secret 或 sample token
```

### 3.4 把 Skill 写成 operational guidance

Skill 应解释 Help 难以紧凑表达的 judgment。

好的 Skill 内容包括：

```txt
什么时候使用这个 Resource
常见 workflow
必要 call order
dangerous operation
confirmation rule
pagination strategy
long-running operation strategy
error recovery
不包含真实 secret 的 example
```

Skill 不应该替代 Help。如果 Agent 只需要一次显然的 call，Help 应该足够。

### 3.5 暴露 auth requirement

Help 至少应该说明 call 是否无需 auth、需要 bearer token，或需要 OAuth：

```txt
auth none
auth bearer
auth oauth2
```

对每个受保护 command，Help 应列出相关 scope：

```txt
cmd update PATCH /api/repos/{owner}/{repo}
  auth oauth2 scope=repo.write
```

Provider runtime 仍然是 authorization decision 的最终权威。Help 是指南，不是 access control decision。

### 3.6 OAuth Integration

如果使用 OAuth，Provider 应避免发明 HTBP 专用 OAuth。

推荐形态：

```txt
~help 声明：
  auth oauth2
  oauth_resource <protected-resource-metadata-url>

401 response 包含：
  WWW-Authenticate: Bearer resource_metadata="<protected-resource-metadata-url>"

Protected resource metadata 指向：
  authorization server metadata

Authorization server metadata 指向：
  authorization endpoint
  token endpoint
  可选 registration endpoint
```

如果需要 self-registration，`~register` 可以是：

```txt
OAuth dynamic client registration 的薄封装
指向真实 registration endpoint 的 redirect 或 pointer
Provider 自定义 onboarding flow
人工 approval request 入口
```

### 3.7 Credential-Specific Help

Provider 可以按 token 返回不同 Help。

示例：

```txt
匿名用户：
  公开只读 command

只读 token：
  read command 和 read scope

管理员 token：
  read、write、delete command
```

这样可以减少 Agent context，也可以避免展示当前 credential 无法执行的 operation。

### 3.8 Caching

control document 应该支持普通 HTTP caching validator：

```http
ETag: "help-repos-2026-06-25"
Cache-Control: private, max-age=300
```

当 Help 依赖 credential 时，使用 `private`。只有当 document 对所有 caller 完全一致时，才使用 public caching。

### 3.9 适配不同 tool source

#### 已有 REST API

保持 REST API 不变，在选定 Resource Path 旁边增加 `~help` 和 `~skill`。

#### CLI Tool

把每个受支持的 CLI operation 包装成 Provider 自定义 HTTP endpoint。HTBP 只描述这些 HTTP endpoint。Wrapper 在调用 CLI 前应校验参数。

#### MCP Server

把选定 MCP capability 暴露成 Provider 自定义 HTTP endpoint。HTBP 描述这些 endpoint，并用 Skill 说明 workflow。Agent 不需要直接讲 MCP。

#### SaaS API Aggregator

用 aggregator 已有 API shape 暴露下游 SaaS operation。通过 Help scope 和 Skill workflow 说明会影响哪个下游 account、workspace 或 tenant。

## 4. Agent Integration

### 4.1 Minimal Agent Algorithm

```txt
1. 从 user、runtime 或已有 context 接收 Domain 或 Resource Path。
2. 通过向 Path 追加 /~help 构造 Help URL。
3. 使用已有 Bearer token 和可选 X-Agent-Id 获取 Help。
4. 选择满足 user request 的最小 command。
5. 构造 Provider 的普通 HTTP request：
     METHOD + Path + Query + Header + JSON Body。
6. 当 Help 不足以指导 workflow 时，获取 Skill。
7. 执行 Provider request。
8. 按 Provider 的普通 response 和 error format 解释结果。
```

### 4.2 Auth Flow

当 request 返回 `401` 时，Agent 应该：

```txt
1. 读取 WWW-Authenticate challenge。
2. 如果存在 resource_metadata，获取 protected resource metadata。
3. 通过 authorization server metadata 找到 auth endpoint 和 token endpoint。
4. 如果 runtime 支持 OAuth，使用对应 OAuth flow。
5. 如果 runtime 不支持 OAuth，把 authorization URL 或 onboarding instruction 呈现给 user。
```

当 Help 声明 `register` 时，Agent 只有在 user 或 runtime 允许 registration 时，才应调用 `~register`。

### 4.3 Skill Fetch Policy

Agent 不应该盲目获取所有 Skill。

以下情况适合获取 Skill：

```txt
Help command 存在歧义
operation 具有 write、delete 或 external effect
workflow 需要多次 call
command 有 Provider 特有 caveat
user 询问 tool 如何工作
```

以下情况不需要获取 Skill：

```txt
Help 已清楚描述一个 read-only single call
task 可以通过当前缓存 Help 完成
Skill path 属于无关或不可信 Domain
```

### 4.4 Command Construction

Agent 应直接把 Help field 映射到 HTTP call：

```txt
cmd     -> HTTP Method 和 Path template
q       -> query string parameter
h       -> request header
body    -> JSON Body 和 Content-Type
auth    -> Authorization 行为
scope   -> token 或 OAuth scope requirement
returns -> expected success response
```

示例 Help：

```txt
cmd list GET /api/repos
  q owner string required
  q limit integer optional default=20 max=100
  auth oauth2 scope=repo.read
```

Agent call：

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/api/repos?owner=octo&limit=20"
```

### 4.5 Confirmation Behavior

`effect` 和 `confirm` 是 hint，不是业务语义本身。

推荐 Agent behavior：

```txt
effect read:
  除非 user 或 runtime policy 要求，否则不额外确认。

effect write:
  当 user intent 清楚时可以执行。

effect delete:
  除非 user 已经明确确认，否则要求 explicit confirmation。

confirm user:
  执行前询问 user。

confirm provider:
  按 Provider 的 confirmation flow 处理。
```

### 4.6 Error Handling

Agent 应把这些结果视为正常情况：

```txt
~help 返回 404：
  该 Path 未暴露 HTBP。只有在合适时才尝试 parent path。

~skill 返回 404：
  Help 存在但没有更深 Skill。继续按 Help 操作。

401：
  authenticate 或 request authorization。

403：
  token 有效但权限不足。只有在 user 或 runtime 允许时，才请求更合适的 authorization。

429：
  如果存在 Retry-After，则遵守它。
```

## 5. Agent-Facing Metadata Inventory

HTBP 应暴露足够信息让 Agent 正确行动，但不能让每次 discovery 都变成完整 API specification。

### 5.1 Core Metadata

```txt
protocol marker
Resource Path 或 template
title
summary
command list
Method/Path/Query/Body/Header mapping
response media type
Skill link
```

### 5.2 Auth Metadata

```txt
auth scheme
required scopes
OAuth protected resource metadata URL
optional registration URL
optional agent identity header support
token placement rule: 只能放在 Authorization header
```

### 5.3 Operational Metadata

```txt
effect class
confirmation hint
rate-limit note
pagination hint
long-running operation hint
streaming hint
retry hint
```

这些 field 是 Provider guidance。HTBP 不强迫 Provider 的业务 API 采用统一语义。

### 5.4 Navigation Metadata

```txt
link parent <url>
link child <url>
link item <url-template>
link docs <url>
```

Navigation metadata 让 Agent 可以渐进式进入 Resource tree，而不是一次加载整个 Domain。

## 6. Recommended Rollout Strategy

### Phase 1：一个 coarse Help

在主 API namespace 暴露一个 Help：

```txt
GET /api/~help
```

先列出少量高价值 read-only operation。

### Phase 2：为 multi-step workflow 增加 Skill

暴露：

```txt
GET /api/~skill
```

记录 Agent 容易做错的少数 workflow。

### Phase 3：增加 resource-local Help

在 local context 重要的位置暴露更具体的 Help：

```txt
GET /api/repos/~help
GET /api/repos/{owner}/{repo}/~help
```

### Phase 4：增加 OAuth 与 Register pointer

增加：

```txt
oauth_resource <url>
register <url>
POST /api/~register
```

只有当 Provider 准备好支持 Agent onboarding 时，才需要做这一步。

## 7. Design Checklist

发布 HTBP 接入前，检查：

```txt
Help 是否准确描述了真实 HTTP call？
Agent 是否能只靠 Help 完成一次 read-only request？
Skill 是否提供了 workflow value，而不是重复 Help？
Token 是否只通过 Authorization header 发送？
write 和 delete command 是否标记 effect 与 confirmation hint？
需要 OAuth 时，401 是否包含有用的 WWW-Authenticate challenge？
Help 是否足够小，能放进 Agent context？
Provider-specific extension 是否能被旧 Agent 忽略？
```

## 8. Open Design Questions

以下问题留给后续 RFC：

```txt
是否注册 HTBP 专用 media type？
是否定义 origin-level discovery document？
是否为 Help DSL 提供 formal grammar？
是否把 JSON Help 标准化为 parallel representation？
是否为 streaming 和 long-running operation 定义标准 hint？
是否提供标准 conformance test suite？
```

这些问题不影响当前 core architecture。

