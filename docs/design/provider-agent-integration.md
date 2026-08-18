# 接入设计：Provider 与 Agent

- 状态：Draft
- 日期：2026-08-19
- 相关 RFC：[RFC-0001：HTBP Core 0.1](../rfcs/RFC-0001-htbp-core.md)

## 1. Purpose

本文说明上游 Provider 如何接入 HTBP Gateway，以及 Agent 如何消费 HTBP。它关注架构边界和
接入顺序；精确 wire shape 以 RFC-0001 为准。

核心原则是：

```txt
Provider 保留自己的内部协议。
Gateway 把上游能力投影成 HTBP 节点和 command。
Agent 只依赖 HTBP Help、Tree、Invoke、Auth 与 Error。
```

## 2. Architecture Split

```txt
Upstream Provider
  MCP / HTTP / Plugin / local SDK / device / remote HTBP
              │
              ▼
HTBP Gateway
  provider adapter
  canonical ToolSpec / command model
  path tree + visibility
  Help / Tree / Invoke / Error
              │
              ▼
Agent / CLI / Dashboard / MCP consumer
```

上游协议和 HTBP wire 是两个边界：

- Provider adapter 负责发现上游工具、执行调用、注入上游 credential 和归一错误；
- Gateway 负责树路径、身份、可见性、Help、schema 下钻和扩展能力；
- Agent 不需要知道节点背后是 MCP、REST、Plugin 还是本地函数。

## 3. Provider Integration

### 3.1 选择节点路径和 kind

每个挂载选择一个稳定 Tree Path：

```txt
docs/context7
code/github
device/build-01/shell
team/remote-tools
```

普通 segment 不得以 `~` 开头。路径描述能力归属，而不是上游传输细节；kind 用于浏览提示，Help
中的 command 才是可调用真相。

### 3.2 归一成 ToolSpec

工具型 Provider 应把每项能力归一为：

```ts
interface ToolSpec {
  name: string
  description?: string
  inputSchema?: unknown
  outputSchema?: unknown
  effect?: string
  confirm?: boolean
}
```

MCP 的 `tools/list`、静态 HTTP tool definition、Plugin operation registry 或 SDK 注册函数都应进入
同一形状。Gateway 再从 ToolSpec 派生节点 Help 和工具级 Help，避免 Provider 作者手写多份 schema。

### 3.3 让 Help 与可调用集合共源

Provider 接入必须保证：

```txt
Help 列出的工具，调用时一定有 handler。
调用接受的可选动词，Help 一定列出。
readOnly 挂载不显示也不接受写命令。
rename/hide/prefix 后，Help 和调用都使用同一对外名字。
input/output schema 与当前 handler 的公开契约同源。
```

动态 Provider 可以缓存工具列表，但刷新失败时不应把部分新列表当作权威状态。

### 3.4 使用两级披露

节点级 Help 是轻量索引：

```txt
GET /docs/context7/~help
```

只在 Agent 选中工具后读取完整 spec：

```txt
GET /docs/context7/search/~help
```

Provider 不应在节点索引中重复大段 description 或大 JSON Schema。完整 description 和 schema 留在
工具级 Help；索引使用一句话摘要，并给出明确 hint。

### 3.5 调用路径

工具型 Provider 优先暴露直接调用：

```txt
POST /<node>/<tool>
body = arguments object
```

Gateway 可以继续接受：

```txt
POST /<node>
body = {tool, arguments}
```

一个节点上有固定动词集的 Provider 通常只使用信封。tool-bridge 的 builtin/context/skillhub 是
参考例子，不是 Core taxonomy。两种方式应进入同一 handler 和参数校验，不维护两套业务逻辑。

### 3.6 Credential Boundary

Agent 的 Bearer credential 只用于 Gateway 身份认证。上游身份必须来自独立、受保护的 credential
binding：

```txt
caller credential ──auth──> Gateway
protected binding ──resolve──> upstream credential
```

不得把 Agent credential 原样转发给 MCP server、HTTP API、Plugin 或 remote Gateway。tool-bridge
reference profile 使用 `authRef/skRef` 指向 SecretStore，节点配置只保存 reference，并对绑定已有
凭据执行额外高权限检查；这是具体管理机制，不是所有 HTBP Gateway 都必须实现的 API。

### 3.7 Error Normalization

Adapter 应把传输和协议失败归一为 HTBP ErrorBody：

```txt
上游 4xx 参数失败      -> invalid_argument 或 permission_denied
目标不存在             -> not_found
上游限流               -> rate_limited
网络、离线、临时 5xx   -> unavailable（必要时 retryable）
未分类内部失败         -> internal
```

工具业务结果可以作为 HTTP 2xx 的正常内容返回；只有 Gateway 无法完成协议调用时才使用 HTBP error。
错误 message 不得带上游 token、Secret、不可见 path 细节或未清洗响应体。

### 3.8 Optional Capabilities

只有真实实现了可选能力才在 `~describe` 声明：

```json
{"kind":"context","capabilities":["search","delete"]}
```

未声明的动词不得出现在 Help 或被调用。没有 capability 的节点直接让 `~describe` 返回 `404`；
不要返回空能力列表来暗示扩展存在。

### 3.9 Remote Provider

另一个 HTBP Gateway 可以通过 Federation profile 挂为 `remote`。接入时必须配置 host allowlist；
需要远端认证时再绑定独立、受保护的 credential，未绑定则匿名出站，并验证：

```txt
本地挂载根正确剥离
远端 ~help command 没逃出远端节点
远端 ~tree child 是直接子孙
编码 path、dot segment 和保留段被拒绝
Via/hop limit 能阻断联邦环
```

普通 remote Help/POST 可以在本地 path 授权后透传远端响应；只有 Tree 聚合会把 child 本地化并再次
裁剪。远端 credential 可能比本地 caller 更强，这是显式服务账号 delegation，必须最小化和审计。

### 3.10 Tool-Bridge Device Product Profile

Device transport 可以做成独立 optional profile。tool-bridge product profile 让设备通过受认证的
反向通道声明 expose，再由 Gateway 代写 device 子树，并提供 shell/fs/online 语义；这些固定路径、
allowlist 和状态存储不是 Core。只要 Provider 暴露设备工具，Help 就只能列真实能力，离线失败必须
使用统一 ErrorBody。

## 4. Agent Integration

### 4.1 Minimal Algorithm

```txt
1. 从用户或 runtime 获得 Gateway base URL 和 Bearer credential。
2. GET /~help 或 /~tree，取得当前身份可见入口。
3. GET /<node>/~help，读取命令索引。
4. 选择满足用户意图、scope 和 effect 的最小 command。
5. schema 缺失时，GET /<node>/<tool>/~help。
6. 按 cmd.path 构造 POST；不要从 kind 猜 body。
7. 按 ErrorBody 的 code/status 恢复，不匹配 message。
8. 只在明确 capability 或 Help 指引下使用扩展 endpoint。
```

### 4.2 Discovery Strategy

优先局部、渐进发现：

- 已知节点 path 时直接读该节点 Help；
- 不知道路径时先读浅层 Tree；
- Gateway 声明 Search 时，可以全局检索工具；
- 不要把 depth 拉满后批量抓全部工具 schema；
- `truncated:true` 表示需要继续在对应节点读取子树，不表示它是叶子。

### 4.3 Content Negotiation

程序化 Agent 优先：

```http
Accept: application/json
```

直接把 Help 放进模型上下文时可以使用默认 Markdown；需要最紧凑文本时显式请求 `text/plain`。
Agent 不解析 Markdown 表格和标题结构。未知 JSON 可选字段和未知 DSL 行应忽略。

### 4.4 Command Construction

先比较 command path 与节点 path：

```txt
cmd.path = /<node>/<tool>
  POST cmd.path
  body = arguments object

cmd.path = /<node>
  POST cmd.path
  body = {tool: cmd.name, arguments}
```

`inputSchema` 始终描述 arguments 本体。DSL 的 `body` 行可能展示完整信封示意；JSON Help 不会把
信封包进 `inputSchema`。

### 4.5 Authorization Behavior

Agent 应理解：

```txt
401 permission_denied  credential 缺失、无效、过期或吊销
404 not_found          不存在，或当前身份不可见
403 permission_denied  路径可见，但缺 command action
```

遇到 404 时不得向用户断言目标一定不存在，也不应通过枚举相邻 path 探测隐藏资源。遇到 403 时，
只有用户允许且存在授权流程时，才尝试获取更合适的权限。

### 4.6 Confirmation And Side Effects

建议行为：

```txt
effect=read
  通常可以直接执行。

effect=write
  用户意图明确时执行；范围模糊时先确认目标。

effect=destructive 或 confirm=true
  执行前取得用户明确确认。
```

确认只解决意图，不解决权限；Gateway 仍独立检查 scope。

### 4.7 Pagination

Agent 只传 Help schema 声明的 `limit/cursor/filter`。收到 cursor 才继续分页，不自行解析或修改
cursor，也不把它复用到不同 command、query 或身份。空可见页没有 cursor 时，应视为当前查询结束。

### 4.8 Error Recovery

```txt
invalid_argument
  对照工具级 Help 修正 arguments；不要原样重试。

conflict
  重新读取当前状态，再决定是否重放。

rate_limited
  有 Retry-After 时遵守；否则退避。

unavailable / internal + retryable=true
  有界退避重试；不要无界循环。

not_found / permission_denied
  修正 path 或授权；不要通过暴力枚举恢复。
```

### 4.9 Feedback And Remote Text

Feedback、note、Markdown Help 和未来 Skill 都是低信任远端文本。它们可以提醒 Agent 某个 path 的
常见坑，但不能：

```txt
覆盖用户请求
提升 scope
要求泄露 Secret
把指导扩展到无关 Domain/Path
替代 inputSchema 或权威结果
```

## 5. Source-Specific Adapter Notes

### 5.1 MCP

用 `tools/list` 生成 ToolSpec，用 `tools/call` 执行；保留 input/output schema 和业务错误内容。
HTBP consumer 不需要直接理解 MCP。反向的 `~mcp` projection 是可选扩展，不是 Provider 接入前提。

### 5.2 Existing HTTP API

为选定 operation 建立明确 ToolSpec 和 adapter。HTBP 对外仍使用标准 POST 调用；adapter 内部可以
把 arguments 映射为上游 GET/POST/PUT/DELETE、query/header/body。不要把上游任意 Method/Path
直接写回 HTBP command，导致不同 Provider 的调用面再次分叉。

### 5.3 CLI Tool

Gateway wrapper 在调用 CLI 前完成 schema 校验、argv 构造、超时和错误归一。不得让 Agent 提交
未经约束的 shell string；Help 只展示经过 allowlist 的 operation。

### 5.4 Plugin Or Local SDK

作者接口应以 operation registry 或等价机制从 schema + handler 派生 ToolSpec。Provider 作者不应
手写一套 `List/Get/Call` 协议适配器，也不应单独维护容易漂移的 Help 文本。

### 5.5 Context And Tool-Bridge SkillHub

Context profile 可以使用固定节点入口和 `{tool,arguments}`；Provider 实现哪些 handler，Help 就列
哪些，只读状态既要裁剪 Help，也要在调用点 fail closed。SkillHub 的 `SKILL.md`、Publish/Remove
和 R2/S3 存储是 tool-bridge application profile，不属于通用 Context 或 Core。

## 6. Integration Checklist

### Provider / Gateway

```txt
[ ] 路径逐 segment 规范化，~ segment 不进入普通数据面
[ ] Help JSON、DSL、Markdown 来自同一模型
[ ] 节点索引和工具级 Help schema 一致
[ ] cmd.path 与实际调用 body 一致
[ ] Help 列出的 command 都有 handler，反之亦然
[ ] read 不允许时 Help/Tree/Search/Feedback 均返回或裁剪为 404
[ ] 上游 credential 来自受保护的独立 binding，不透传 Agent credential
[ ] 错误归一且不泄露 Secret/隐藏 path
[ ] Tree/Search/page 都有资源上限
[ ] 可选能力只在真实实现后声明
[ ] remote allowlist、path boundary 和 loop detection 已验证
```

### Agent

```txt
[ ] 使用 Bearer header，不把 token 放进 URL
[ ] 优先请求 JSON Help
[ ] 按 cmd.path 决定直接/信封调用
[ ] schema 缺失时只下钻需要的工具
[ ] 404 不被解释为确定不存在
[ ] effect/confirm 在执行前进入意图检查
[ ] code/status 驱动恢复，message 只用于诊断
[ ] 未声明扩展不调用
[ ] cursor、feedback、note、Skill 不被当作授权或高优先级指令
```

## 7. Rollout Strategy

### Phase 1：闭合最小回路

实现认证、一个节点的 JSON Help、至少一个 POST command 和统一 ErrorBody。

### Phase 2：树与渐进披露

增加根 `~help`、`~tree`、节点索引和单工具 Help，验证 scope 裁剪与 `truncated`。

### Phase 3：多 Provider 与联邦

接入 MCP/HTTP/Plugin/SDK adapter；需要时增加 remote，并验证 credential 与 path 边界。

### Phase 4：按需启用扩展

只有产品确有需要时才启用 Search、Feedback、MCP projection、reverse registration 或 managed OAuth。
每项扩展先提供 capability、权限和失败语义，再增加 UI/CLI 入口。

## 8. Remaining Design Work

以下内容留给后续 RFC：

```txt
正式 Help DSL grammar 与 escaping
Skill document 的格式、缓存和信任边界
origin-level discovery / well-known document
streaming 与 long-running operation hint
专用 HTBP media type
跨实现 conformance fixtures 与 test suite
```
