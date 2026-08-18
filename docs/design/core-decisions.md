# 核心设计决策

- 状态：Draft
- 日期：2026-08-19

相关文档：

- [RFC-0001：HTBP Core 0.1](../rfcs/RFC-0001-htbp-core.md)
- [接入设计：Provider 与 Agent](provider-agent-integration.md)

## 1. 协议是一棵树，不是 REST API 旁挂说明

HTBP 的协议对象是一个由 Tree Path 寻址的工具 namespace。上游能力可以来自 MCP、HTTP、Plugin、
进程内 Provider、设备或远端 HTBP，但 Agent 看到的是同一棵树和同一套 Help/Invoke/Error 语义。

这取代了最初“保留 Provider 原生 Method/Path，只旁挂 `~help`”的设计。原因是仅有说明层无法统一：

```txt
身份可见性
局部与全局发现
调用 body
错误恢复
联邦路径
多上游适配后的同一消费面
```

## 2. Core surface 保持小而闭合

HTBP 0.1 Core 只要求形成完整的发现—调用回路：

```txt
GET  /<path>/~help
GET  /<path>/~tree?depth=N
POST /<node>/<tool>
POST /<node>  {tool,arguments}
```

`~describe/~search/~feedback/~register/~authorize/~mcp/~skill` 都不是 Core 必选面。Search、
Feedback、Remote 等可以形成独立协议 profile；reverse registration、managed OAuth、MCP projection
则是 tool-bridge 产品/adapter profile。扩展名被保留是为了避免路径冲突，不表示 Provider 已实现。

## 3. 为什么保留 `~` Control Segment

以 `~` 开头的 segment 专属于 control plane：

```txt
/docs/context7/~help
/docs/~tree
/~search
```

这使 control request 不会和普通节点或工具混淆，也允许 Agent 从任意 Tree Path 做局部发现。
普通节点 segment 一律不得以 `~` 开头；未知 control segment 必须走 `404`，不能落到工具调用。

## 4. Help 是单一语义模型的多种表现

Help 同时提供：

```txt
application/json   规范性机器可读形状
text/plain         紧凑行式 DSL
text/markdown      默认的人读与 Agent 阅读形态
```

三种表现必须从同一个模型渲染，避免独立维护后字段漂移。JSON/DSL 是结构化互操作面；Markdown
排版可以改进，但客户端不应解析其版式。

缺失或未知 `Accept` 默认 Markdown。显式 JSON 优先于 Markdown，Markdown 优先于 DSL。

## 5. 两级披露，而不是一次发送全部 schema

工具节点的 `~help` 是索引：保留 name、path、scope、简短 description 和副作用提示，可以省略
input/output schema。需要调用某个工具时，再读取：

```txt
GET /<node>/<tool>/~help
```

这让拥有大量工具或大 schema 的节点仍可被低成本浏览。工具级 Help 是路径投影，不要求在 registry
里持久化一个伪节点。

## 6. 直接 URL 是首选，信封是通用兼容面

普通工具在 Help 中宣告独立路径：

```txt
POST /<node>/<tool>
body = arguments object
```

`cmd.path` 指向节点自身、或名字不适合单个 URL segment 的命令使用：

```txt
POST /<node>
body = {tool, arguments}
```

工具节点仍可接受信封，便于旧客户端。tool-bridge 的 builtin/context/skillhub 通常采用该形状，
但这是 reference taxonomy，不是 Core 路由规则；Agent 必须以 `cmd.path` 为准。

## 7. 发现和调用共享同一授权下界

Core 标准化 `read/call` 的可见性和调用语义；action token 可以扩展。tool-bridge IAM profile 另使用
`write/register/admin`，并以 segment-aware glob 表达 path scope：deny 优先、无匹配默认拒绝、
未知 action fail closed。其他实现可以使用不同 policy engine，但必须保持相同的对外可见性语义。

授权分两层：

```txt
read 不允许  → 404，不泄露节点存在性
read 允许但命令 action 不允许 → 403
```

根 Help/Tree 作为已认证入口，但返回内容必须裁剪。Tree、Help、Search、Feedback、MCP 投影和远端
聚合都复用该规则；派生索引和缓存不能变成授权真源。

## 8. 错误形状属于协议核心

不同上游错误必须归一为：

```json
{"code":"invalid_argument","message":"...","retryable":false}
```

稳定 code 是：

```txt
not_found  permission_denied  invalid_argument  conflict
unavailable  rate_limited  internal
```

客户端按 code + HTTP status 做控制流，不匹配 message。只有 rate-limited、unavailable、internal
可以重试；401 仍使用 permission_denied，501 仍使用 unavailable。

## 9. 节点 kind 是提示，Help 才是可调用真相

tool-bridge reference profile 当前有：

```txt
directory mcp http builtin context device remote tool skillhub
```

这不是 Core 的封闭枚举。客户端必须容忍未知 kind，并读取 `~help`。Provider 只展示实际可调用的
command；只读挂载必须隐藏写动词，Plugin 或设备没有实现的方法不能因为 profile 默认值而出现在 Help。

## 10. `~tree` 必须有界且诚实

Tree 默认深度 2、最大深度 8、默认最多展开 500 节点。达到深度、节点、环或不透明远端边界时，
使用 `truncated:true`，不能把未展开节点伪装成叶子。

非根 Tree 必须对应真实节点，并使用它的真实 kind/description；不能凭 URL 伪造 directory。

## 11. 可选扩展遵循显式 capability

`~describe` 只返回可选能力。没有能力就 `404`。全局 Search 只有根声明 `search` 后才存在；
`search:semantic` 另行声明。Context/SkillHub 的可选动词也必须和实际 handler 一致。

扩展的共同规则：

```txt
未声明 = 不可假设
派生状态 = 非授权真源
未知可选字段 = 消费端忽略
未知写入字段 = 服务端拒绝
```

## 12. tool-bridge 的 `~register` 是产品管理面

最初草案把 `~register` 宽泛描述为账号、token 或 OAuth client onboarding。tool-bridge product profile
把同名路径用于反向 NodeInput 注册：URL path 必须等于 body.path，并执行自己的 register scope、
registerPaths、保留根、credential binding 和配置校验。

NodeInput/IAM/SecretStore 没有进入 Core，其他 Provider 不承担兼容义务。OAuth client registration
应继续使用标准 OAuth metadata；这次同名语义碰撞必须在兼容章节明确标成破坏性变更。

## 13. `~skill` 暂不进入 Core

Skill 仍是有价值的渐进指导层，但当前本地实现没有稳定的 Skill wire contract。因此 `~skill` 只保留
路径；本地可以返回 501，remote 可以透传。等信任边界、格式、缓存和发现条件稳定后再单独标准化。

## 14. 联邦保持同形，但身份不透传

Remote mount 把本地挂载前缀剥掉后，同形转发 `~help`、`~tree`、`~skill` 和 POST。远端目标必须
在 allowlist 中；空 allowlist fail closed。

本地调用者的 Secret Key 不得传给远端。Remote mount 使用受保护的独立 credential 建立远端身份，
并通过 Via/hop limit、canonical path 和挂载边界检查阻止环与路径逃逸。该 credential 可以比本地
caller 权限更高，这是必须最小化并审计的服务账号 delegation，不是端到端 caller identity。

## 15. 当前不再开放的问题

以下问题已由 0.1 wire 定型，不再作为待讨论项：

```txt
是否有 JSON Help：有，且是规范性表现。
是否定义树发现：有，~tree 是 Core。
是否统一调用：有，直接 POST + 信封 POST。
是否统一错误：有，七码 ErrorBody。
~register 是什么：Core 只保留命名；tool-bridge product profile 用于反向节点注册。
~skill 是否必选：不是，仍是保留扩展。
```

仍可后续标准化的内容包括专用 media type、Skill RFC、origin-level discovery、正式 DSL grammar、
streaming/long-running hint 和跨实现 conformance suite。
