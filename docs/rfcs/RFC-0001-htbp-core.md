# RFC-0001：HTBP Core 0.1

- 状态：Draft
- 日期：2026-08-19
- 类别：Core Protocol
- 实现基线：[TokenRollAI/tool-bridge](https://github.com/TokenRollAI/tool-bridge) 的当前 wire contract
- 目标读者：Agent runtime、tool gateway implementer、SDK/CLI author、tool provider author、MCP adapter author

## Abstract

HTTP ToolBridge Protocol（HTBP）定义一套面向 Agent 的 HTTP 工具发现与调用协议。Provider
把工具、上下文与远端 HTBP 服务挂载到一棵由路径寻址、按身份裁剪的节点树；Agent 只需要标准
HTTP 能力，就可以逐层发现节点、读取命令 schema，并调用工具。

HTBP 0.1 的核心 wire surface 是：

```txt
GET  /<path>/~help              读取节点或单个工具的使用说明
GET  /<path>/~tree?depth=N      读取深度受限、按身份裁剪的树
POST /<node>/<tool>             直接调用；body 是 arguments 对象
POST /<node>                    信封调用；body 是 {tool, arguments}
```

所有核心数据面请求都经过 Bearer 身份认证。节点是否可见与命令是否可调用是两层不同的授权判断：
不可见路径以 `404` 表达，避免泄露其存在；可见但无权调用的路径以 `403` 表达。协议错误统一为
`{code,message,retryable}`。

`~describe`、`~search`、`~feedback`、`~register`、`~authorize`、`~mcp` 与 `~skill`
属于扩展面。本 RFC 规定它们的发现和兼容边界，并记录 tool-bridge 0.1 profile 已实现的形状；
实现不得仅因为路径被保留就假装支持某项扩展。

## 1. Introduction

Agent 接入工具时通常需要解决四个问题：

```txt
1. 能力位于哪里？
2. 当前身份能看到什么？
3. 请求 body 和返回值是什么形状？
4. 怎样在不预加载全部 schema 的情况下继续发现？
```

只在既有 REST API 旁边增加一份说明文档，无法给出统一的树遍历、调用、权限可见性、错误或
联邦语义。HTBP 因此把协议边界收敛为一个可组合的 HTTP namespace：

```txt
Tree       组织节点与局部命令
Help       描述节点、命令、schema 与下一步
Tree View  提供有界的结构发现
Invoke     提供直接调用和信封调用
Auth       按路径和 action 裁剪发现与调用
Errors     提供跨 Provider 一致的失败形状
Extensions 在明确声明后增加 Search、Feedback、MCP 等能力
```

核心原则是：

```txt
如果 Agent 能发送带 Bearer credential 的 HTTP 请求，它就能发现并调用当前身份获准使用的工具，
而无需安装本地 CLI、占用预注册 tool slot，或理解上游 Provider 的原生传输。
```

## 2. Requirements Language

本文中的“必须”“不得”“应该”“不应该”“可以”“可选”等词用于表达协议要求，对应含义参考
RFC 2119 与 RFC 8174。本文仍是 Draft；`htbp 0.1` 和 JSON 中的 `"htbp":"0.1"` 是当前
wire version marker。

## 3. Goals

HTBP 0.1 的目标是：

```txt
G1. 仅依赖 HTTP 即可发现和调用工具。
G2. 以路径树承载局部发现、权限裁剪与联邦挂载。
G3. 让文本阅读者、程序化客户端和人类分别获得 DSL、JSON、Markdown 表现。
G4. 通过节点索引 → 单工具详情实现 progressive disclosure。
G5. 为调用、错误、分页和授权 action 提供稳定的跨 Provider 语义。
G6. 允许未知节点 kind、未知 Help 行和未声明的可选扩展向前演进。
G7. 让 MCP、HTTP API、进程内工具、设备或远端 HTBP 树共享同一个消费面。
```

## 4. Non-Goals

HTBP 0.1 不定义：

```txt
N1. 上游 MCP、REST、CLI、SaaS 或设备协议的内部实现。
N2. Provider 如何存储节点、Secret、索引、反馈或调用结果。
N3. Dashboard、CLI 或 SDK 的产品界面。
N4. Plugin manifest、catalog 或部署生命周期。
N5. OAuth 本身；~authorize 只可以装配既有 OAuth 流程。
N6. 所有工具都必须共享的业务 Resource model。
N7. 用 Help 取代完整 OpenAPI、JSON Schema 或 Provider 文档。
N8. 要求实现支持所有扩展 endpoint 或所有节点 kind。
```

## 5. Terminology And Data Model

### 5.1 Gateway

Gateway 是暴露 HTBP HTTP surface 的服务。它可以直接执行工具，也可以代理 MCP、HTTP、Plugin、
Device 或另一个 HTBP Gateway。

### 5.2 Tree Path

Tree Path 是节点在 HTBP 树中的规范路径。JSON 和 DSL 中的 Tree Path：

- 使用 `/` 分隔 segment；
- 不带前导或尾随 `/`；
- 根路径表示为空字符串；DSL 展示根时使用 `/`；
- segment 在 HTTP URL 中逐段进行 percent-encoding；
- 不得包含空 segment；
- 普通节点 segment 不得以 `~` 开头。

示例：

```txt
JSON/DSL Tree Path       HTTP Path
""                       /
docs                     /docs
docs/context7            /docs/context7
docs/hello world         /docs/hello%20world
```

路径按 segment 比较，不得把字符串前缀当作祖先关系。例如 `docs/a` 是 `docs/a/x` 的祖先，
但不是 `docs/ab` 的祖先。

### 5.3 Node

Node 是树上有稳定 Tree Path 的发现单元。节点至少有：

```ts
interface NodeRef {
  path: string
  kind: string
  description: string
}
```

Core 只标准化 `directory` 的组织语义；其他 kind 是可扩展 token。tool-bridge reference profile
当前会产生以下 kind：

```txt
directory  只组织子节点
mcp        由 MCP server 提供工具
http       由 HTTP operation 定义提供工具
builtin    Gateway 内建命令节点
context    上下文对象读写节点
device     由已连接设备提供能力
remote     挂载另一个 HTBP 树
tool       由通用 tool provider 提供工具
skillhub   Skill 内容存储与检索节点
```

这些名称记录参考实现的 taxonomy，不要求其他 HTBP Provider 全部实现，也不把 `builtin`、
`device` 或 `skillhub` 的产品内部模型提升为 Core。Agent 必须把未知 kind 当作不透明节点，并以
`~help` 中的命令为准；不得因为 kind 未知就拒绝整个树或猜测调用语义。根节点使用
`directory`。

### 5.4 Command

Command 是节点公开的一项可调用能力。HTBP 0.1 command 至少声明：

```ts
interface Command {
  name: string
  method: "POST"
  path: string
  scope: string
}
```

Command 还可以声明 description、arguments JSON Schema、result JSON Schema、人读返回说明、
effect 和 confirmation hint。`~help` 是命令集合的权威发现入口；节点 kind 不能替代命令表。

### 5.5 Control Segment

HTTP Path 中以 `~` 开头的 segment 属于 HTBP control plane。普通节点不得占用此类 segment。
HTBP 0.1 使用或保留下列名称：

```txt
~help  ~tree  ~describe  ~search  ~feedback
~register  ~authorize  ~mcp  ~skill
```

未来版本可以增加其他 `~` segment。Agent 对未知 control segment 应按普通 `404` 处理；Provider
不得把未知 control request 误投递到工具数据面。

## 6. Protocol Overview

典型的渐进发现和调用过程是：

```txt
1. GET /~help 或 GET /~tree，取得当前身份可见的入口。
2. GET /<node>/~help，取得命令索引。
3. 若索引省略 schema，GET /<node>/<tool>/~help 取得单工具详情。
4. POST /<node>/<tool>，body 直接发送 arguments 对象。
5. 兼容旧客户端时，也可以 POST /<node>，发送 {tool,arguments} 信封。
6. 只在 ~describe 明确声明或 ~help 明确链接时使用可选扩展。
```

所有核心请求都使用 HTTPS 部署。除实现明确标记的公开诊断或带签名引用外，Gateway 必须先完成
认证，再进入 HTBP 树路由。

## 7. Authentication, Authorization And Visibility

### 7.1 Bearer Authentication

客户端使用标准 Bearer header：

```http
Authorization: Bearer <secret-key>
```

credential 不得放在 query string、Tree Path、Help、日志或错误消息中。缺失、无效、过期或已吊销
的 credential 返回：

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8

{"code":"permission_denied","message":"...","retryable":false}
```

tool-bridge 当前还接受不带 `Bearer` 前缀的裸 token 作为旧客户端兼容行为；它不是 HTBP 0.1
conformance 要求，新客户端不得依赖。

### 7.2 Actions

Core 规定每个 command 必须声明一个 action token，并标准化 `read` 与 `call` 的可见性/调用语义。
tool-bridge IAM profile 当前使用：

| action | 分类 | 含义 |
|---|---|---|
| `read` | Core | 发现节点、读取数据或只读命令 |
| `call` | Core | 调用普通工具或产生一般副作用 |
| `write` | Resource profile | 修改节点所管理的数据 |
| `register` | tool-bridge management profile | 注册、更新、授权或反向挂载节点 |
| `admin` | tool-bridge management profile | 安全管理、强制清理等高权限操作 |

实现不需要支持所有 profile action。未知 action 必须 fail closed；Agent 不得把未知 action 当作
`read` 或 `call`。

### 7.3 Tool-Bridge Path Scope Profile

Core 只要求 Gateway 能对 `(identity,path,action)` 作 fail-closed 决策，不规定 credential 内部格式。
tool-bridge IAM profile 让一个身份携带若干 path scope：

```ts
interface Scope {
  pattern: string
  actions: string[]
  effect?: "allow" | "deny" // 默认 allow
}
```

`*` 匹配一个 segment，`**` 匹配零个或多个 segment。判定顺序必须是：

```txt
1. 任意匹配的 deny → deny
2. 否则任意匹配的 allow → allow
3. 否则默认 deny
```

在该 profile 中，空 scope 集合表示没有任何权限。其他实现可以使用不同 policy engine，但对外仍须
遵守第 7.4 节的 404/403 可见性语义和 Help 中的 action token。

### 7.4 Visibility

读取非根节点的 `~help`、`~tree`，或任何会返回现有 path-specific 状态的局部 control document
前，Gateway 必须先判断目标路径的 `read` 权限。管理 profile 可以定义针对尚不存在节点的独立
action；一个不解析目标路径、对所有本地 path 都返回相同 `501` 的保留占位 endpoint 也不会泄露
节点状态。

- 目标对当前身份不可见时，必须返回 `404 not_found`，不得返回 `403` 泄露节点存在性。
- 目标可见、但 command 所需 action 不允许时，返回 `403 permission_denied`。
- 根 `~help` 和根 `~tree` 可以作为认证后的入口，但它们的子项必须按当前身份裁剪。
- Search、列表和联邦聚合也必须在返回前重新做可见性裁剪；派生索引不得成为授权依据。

## 8. Help

### 8.1 Endpoint

```http
GET /~help
GET /<node>/~help
GET /<node>/<tool>/~help
```

根 Help 描述虚拟根节点和可见的顶层子节点。节点 Help 描述节点命令或直接子节点。工具级 Help
用于第二级披露，返回单个工具的完整 description 和 schema；工具级路径不要求对应一个持久化节点。

### 8.2 Content Negotiation

`~help` 支持三种表现：

| Request `Accept` | Response `Content-Type` | 用途 |
|---|---|---|
| `application/json` | `application/json; charset=utf-8` | 规范性机器可读形状 |
| `text/plain` | `text/plain; charset=utf-8` | 紧凑 Help DSL |
| `text/markdown`、缺失或其他值 | `text/markdown; charset=utf-8` | 默认的人读/Agent 可读表现 |

当一个 `Accept` 同时包含多个已知类型时，选择优先级是 JSON、Markdown、DSL。0.1 negotiation
按大小写不敏感的 media-type 出现与否选择，不解释 `q` 权重；例如 `application/json;q=0` 仍会
命中 JSON。Markdown 与 DSL/JSON 来自同一个语义模型，但客户端不得把 Markdown 排版当作稳定
parser contract。未来若改为完整 HTTP preference parsing，需要提升协议版本或提供兼容说明。

### 8.3 Help JSON

JSON 表现的规范形状是：

```ts
interface HelpJson {
  htbp: "0.1"
  node: {
    path: string
    kind: string
    description: string
  }
  cmds: Array<{
    name: string
    method: "POST"
    path: string
    scope: string
    h?: string
    inputSchema?: unknown
    outputSchema?: unknown
    returns?: string
    effect?: string
    confirm?: true
  }>
  children?: Array<{
    path: string
    kind: string
    description: string
  }>
  hint?: string
  note?: string
  feedback?: Array<{
    id: string
    title: string
    score: number
  }>
}
```

约束：

- `cmds` 必须存在，可以为空。
- `inputSchema` 是 arguments 本体的 JSON Schema，不包含 `{tool,arguments}` 信封。
- `outputSchema` 是成功结果的 JSON Schema 声明；Provider 可以省略，Gateway 不因此校验返回值。
- `returns` 是人读说明，不能替代 `outputSchema`。
- `confirm` 只在值为真时出现；缺失表示协议未要求额外确认。
- `effect` 是语义提示；`destructive` command 未显式关闭确认时，Provider 应令 `confirm` 为真。
- `hint`、`note` 和 `feedback` 是可选增强信息，不得改变授权结果。
- 客户端必须忽略未知可选字段，但不得静默忽略自己发送请求中的未知参数。

示例：

```json
{
  "htbp": "0.1",
  "node": {
    "path": "docs/context7/search",
    "kind": "mcp",
    "description": "Search documentation"
  },
  "cmds": [
    {
      "name": "search",
      "method": "POST",
      "path": "/docs/context7/search",
      "scope": "call",
      "h": "Search the selected documentation source.",
      "inputSchema": {
        "type": "object",
        "properties": { "query": { "type": "string" } },
        "required": ["query"]
      },
      "effect": "read"
    }
  ]
}
```

### 8.4 Help DSL

DSL 是 UTF-8、line-oriented 文本。基本行是：

```txt
htbp 0.1
node <tree-path-or-/> <kind> "<description>"
hint <one-line-text>
note "<one-line-text>"
cmd <name> POST <absolute-http-path>
  h <description>
  body <compact-json-schema-or-envelope-schema>
  result <compact-json-schema>
  returns <text>
  scope <action>
  effect <text>
  confirm
node <child-path> <child-kind> "<child-description>"
feedback <count> GET /<path>/~feedback
  <id> <score> "<title>"
  use <one-line-instructions>
```

第一条 `node` 行表示当前节点；后续 `node` 行表示直接子节点。每条 `cmd` 的 `scope` 行必须存在。
属性属于最近一条 `cmd`，直到出现下一条 `cmd` 或顶层行。

`h`、`returns` 等长文本可以使用四个空格的续行；最小 parser 只需识别 `htbp`、`node`、
`cmd` 和紧随 command 的 `scope`。客户端必须忽略未知顶层行、未知 command 属性与续行，使
新增可选元数据保持向前兼容。

`body` 行描述请求 schema，而不是一个可以原样提交的实例：

- 直接调用 command 的 `body` 是 arguments JSON Schema；
- 信封调用 command 的 `body` 是 `{tool:<name>,arguments:<arguments-schema>}` 的紧凑示意。

### 8.5 Progressive Disclosure

工具节点的节点级 Help 可以只返回索引：每个 command 保留 `name`、`path`、`scope`、简短 `h`
及 effect/confirm，但省略较大的 input/output schema。此时 Help 应通过 `hint` 告诉客户端：

```txt
GET /<node>/<tool>/~help
```

可以取得该工具的完整 spec。Agent 不应在只需要少量工具时预取整个树的全部工具级 Help。

### 8.6 Markdown Representation

Markdown 表现应完整解释：

- 当前节点的 path、kind 和 description；
- 每个 command 的实际 POST 路径与 body 形状；
- required scope、effect 与 confirmation；
- schema 下钻路径和直接子节点；
- 存在时的 note 与 feedback。

Markdown 是默认阅读表现，不是独立语义源。若 Markdown 与 JSON/DSL 不一致，Provider 必须修正
渲染器；程序化客户端应请求 JSON。

## 9. Tree Discovery

### 9.1 Endpoint

```http
GET /~tree?depth=N
GET /<path>/~tree?depth=N
```

根请求返回当前身份可见的树。非根请求要求该路径真实存在且可见；实现不得为不存在的 path
伪造 directory 根。

### 9.2 Depth And Bounds

HTBP 0.1 profile：

- `depth` 默认 `2`；
- 非整数或小于 `1` 的值按默认值处理；
- 大于 `8` 的值钳制为 `8`；
- 一次响应最多展开 `500` 个节点。

达到深度、节点、环检测或不透明远端边界时，Provider 必须在对应节点设置
`"truncated":true`，而不是静默伪装成叶子。

### 9.3 Tree JSON

当 `Accept: application/json` 时返回：

```ts
interface TreeJson {
  path: string
  kind: string
  description: string
  online?: boolean
  truncated?: boolean
  children?: TreeJson[]
}
```

`online` 只在节点存在有意义的连接状态时出现。`children` 只包含当前身份可见的直接子节点。
列表面至少要求 `read`；对于 `mcp`、`http`、`remote`、`device`、`tool` 这类可调用节点，当前
profile 还要求 `call`，否则节点不出现在父 Help 或 Tree 中。直接请求该可见节点时仍按
`read -> 404`、`call -> 403` 的顺序判定。

### 9.4 Text Representations

`Accept: text/plain` 返回缩进树；默认 Markdown 把同一缩进树放在 `text` code fence 中。
文本行的参考形状是：

```txt
/ [directory] tool root
  docs [directory] Documentation
    docs/context7 [mcp] Context7 tools …
```

尾部 `…` 表示 `truncated:true`。文本表现供阅读，不作为机器解析的稳定形状。

## 10. Invocation

### 10.1 Direct Tool Invocation

Help 中 command path 指向 `/<node>/<tool>` 时，客户端应该直接调用：

```http
POST /<node>/<tool> HTTP/1.1
Authorization: Bearer <secret-key>
Content-Type: application/json
Accept: application/json

{...arguments}
```

body 必须是 JSON object。无参数时可以发送 `{}` 或空 JSON body；Provider 把它解释为 `{}`。
直接调用只识别节点后恰好一个工具 segment；隐藏工具、虚拟化前的旧名字或多余路径返回 `404`。
当工具名无法安全编码成单个 URL segment 时，客户端应改用信封调用。

直接工具 URL 是 owning node 的调用投影，授权目标应是 owning node 的 `read` 与 command `scope`，
不应额外要求身份对伪 tool child path 持有一份 `read`。当前 tool-bridge reference build 仍先检查原始
`<node>/<tool>` path 的 `read`，再检查 owning node，因此只精确授权 node path 时可能出现 envelope
成功而 direct 返回 `404`。这是已知 conformance gap；在修复并补齐测试前，客户端不能假设两种
入口对精确 scope 完全等价。

### 10.2 Envelope Invocation

当 `cmd.path` 指向节点自身，或 command name 不适合放进单个 URL segment 时，使用节点入口：

```http
POST /<node> HTTP/1.1
Authorization: Bearer <secret-key>
Content-Type: application/json
Accept: application/json

{
  "tool": "<command-name>",
  "arguments": {...}
}
```

`tool` 必须是字符串；缺失 `arguments` 等价于 `{}`。Provider 必须拒绝未知 command，而不是把
它转发给不确定的上游。

工具节点可以同时接受 envelope 作为兼容入口，但 `~help` 应优先宣告直接工具 URL。客户端必须
按 `cmd.path` 构造请求，不应仅从节点 kind 猜测调用形状。tool-bridge 的 builtin、context、
skillhub profile 通常使用这种信封，但这只是参考 taxonomy，不是 Core 路由规则。

### 10.3 Result Representation

成功结果是 Provider command 的 JSON 值：

- `Accept: application/json` 返回原始 JSON；
- 缺失、Markdown 或 text Accept 返回包含该 JSON 的 Markdown code fence；
- 协议/传输失败始终使用第 11 节的 JSON error body。

HTTP `2xx` 只表示 HTBP 调用成功完成。工具自身是否把某个结果视为业务失败，可以在其
`outputSchema`、`returns` 或结果内容中定义。

## 11. Errors

### 11.1 Error Body

所有 HTBP 协议错误必须使用：

```ts
interface ErrorBody {
  code:
    | "not_found"
    | "permission_denied"
    | "invalid_argument"
    | "conflict"
    | "unavailable"
    | "rate_limited"
    | "internal"
  message: string
  retryable: boolean
}
```

`message` 用于诊断，不得包含 credential、上游 Secret 或不可见资源细节。客户端必须以 `code`
和 HTTP status 做控制流，不应匹配 message 文案。

### 11.2 Status Mapping

| code | HTTP status | retryable |
|---|---:|---|
| `not_found` | 404 | false |
| `permission_denied` | 403 | false |
| `invalid_argument` | 400 | false |
| `conflict` | 409 | false |
| `rate_limited` | 429 | 可以为 true |
| `unavailable` | 503 | 可以为 true |
| `internal` | 500 | 可以为 true |

只有两个特例可以偏离该表：

- 未认证使用 `401`，code 仍为 `permission_denied`；
- 已知但未实现的能力使用 `501`，code 为 `unavailable`，且不可重试。

只有 `rate_limited`、`unavailable`、`internal` 可以声明 `retryable:true`。

### 11.3 Visibility-Safe Errors

下面情况必须返回 `404 not_found`：

- 路径不存在；
- 路径存在但当前身份没有 `read`；
- control extension 未知或未声明；已保留且被实现明确识别、但尚未实现的扩展可以按第 11.2 节返回
  `501 unavailable`；
- 请求试图把保留 segment 当作普通工具数据面。

当路径已可见，但身份缺少 command 所需 action 时，返回 `403 permission_denied`。

## 12. Pagination

分页 command 应使用统一 envelope：

```ts
interface Page<T> {
  items: T[]
  cursor?: string
}

interface ListOptions {
  cursor?: string
  filter?: Record<string, string>
  limit?: number
}
```

HTBP 0.1 profile 的默认 `limit` 是 `50`，上限是 `200`；超过上限可以钳制到 `200`。`cursor`
只表示继续位置，不是 credential 或授权证明。Provider 必须在翻页结果返回前重新执行可见性判断。

具体 command 只可以接受其 `~help` schema 声明的 filter key；未知写入参数必须返回
`invalid_argument`，不得静默忽略。

## 13. Optional Profiles And Reference Implementation Notes

本节不是 Core conformance 的必选 surface。13.1–13.3 与 13.8 描述可独立实现的协议 profile；
13.4–13.6 记录 tool-bridge 产品/adapter 行为，避免把当前实现误当成所有 HTBP Provider 的管理面。

### 13.1 Capability Discovery：`~describe`

可选能力通过：

```http
GET /<path>/~describe
Accept: application/json
```

返回：

```json
{"kind":"context","capabilities":["search","delete"]}
```

没有可选能力的节点返回 `404`。根 `GET /~describe` 可以声明全局 capability，例如
`search` 或 `search:semantic`。客户端不得只因某个保留 endpoint 名存在就推断能力已实现。

### 13.2 Global Tool Search：`~search`

当根 `~describe` 声明 `search` 时，可以调用：

```http
POST /~search
Content-Type: application/json
Accept: application/json

{
  "query": "documentation search",
  "opts": {
    "mode": "keyword",
    "limit": 20,
    "cursor": "..."
  }
}
```

`query` 必须是非空字符串。`mode` 是 `keyword` 或 `semantic`；后者要求
`search:semantic` capability。响应为：

```ts
interface SearchPage {
  items: Array<{
    path: string
    tool: {
      name: string
      description?: string
      inputSchema?: unknown
      outputSchema?: unknown
      effect?: string
      confirm?: boolean
    }
  }>
  cursor?: string
}
```

Search index 是派生数据。Provider 必须对候选重新读取规范工具定义并检查 `read` 与 `call`；
空的可见结果页不得通过 cursor 泄露隐藏命中量。

### 13.3 Per-Path Agent Feedback：`~feedback`

实现声明 Feedback 时，节点或工具子路径可以提供：

```txt
GET    /<path>/~feedback
GET    /<path>/~feedback/<id>
POST   /<path>/~feedback              {"title":"...","detail":"..."}
POST   /<path>/~feedback/<id>         {"vote":"up|down|clear"}
DELETE /<path>/~feedback/<id>
```

列表与详情需要目标 path 的 `read`；提交和投票还需要 `call`；删除需要 `admin`。`~help`
可以内联排序后的少量 `{id,title,score}`，详情仍通过 feedback endpoint 下钻。Feedback 是提示，
不得改变命令 schema、授权或权威业务状态。根路径不接受 Feedback：`GET|POST /~feedback`
必须返回 `404 not_found`。

### 13.4 Tool-Bridge Product Profile：`~register`

本小节是 tool-bridge 管理面记录，不属于 HTBP Core 或跨实现 conformance。tool-bridge 把
`POST /<path>/~register` 定义为反向节点注册，而不是公开账号注册或 OAuth Dynamic Client
Registration：

- URL path 必须与 body 的 `path` 完全一致；
- body 是实现支持的 NodeInput；
- 调用者必须具有目标 path 的 `register`；
- Gateway 执行自身的保留根、register path、credential binding 与节点配置校验后才写入；
- 失败不得留下部分注册状态。

`NodeInput`、register path 和 SecretStore 形状尚未在本 RFC 标准化，其他实现不因本小节承担兼容
义务。如需账号或 OAuth client 注册，应优先使用标准 OAuth metadata，避免无关 onboarding
语义碰撞同名保留 segment。

### 13.5 Tool-Bridge Product Profile：`~authorize`

tool-bridge 的 `POST /<path>/~authorize` 可以启动节点所需的托管 OAuth 流程。它要求节点可见且
调用者具有 management `register` action，返回 authorization URL/state 等实现定义的 JSON。
Plugin descriptor、credential binding、token storage 和 callback 是产品实现，不是 Core；OAuth
endpoint、PKCE 和 token exchange 仍由相应 OAuth 标准定义。

### 13.6 MCP Adapter：`~mcp`

`ALL /~mcp` 可以把当前 Bearer 身份可见且可调用的 HTBP command 投影为 MCP server。投影必须：

- 复用当前请求身份，不信任进程内旧会话身份；
- 只暴露当前可见且可调用的 command；
- 保持 Help、Tree、Search 等 control tool 与实际 HTBP endpoint 同源；
- 不扩大上游凭证或本地 path scope。

MCP transport 与 tool result block 的具体形状由 MCP 规范定义。

### 13.7 Skill：`~skill`

`~skill` 是保留扩展，不是 HTBP 0.1 Core 的必选 endpoint。本地没有 Skill 文档时，Provider 可以
返回 `501 unavailable`；远端联邦可以透传远端实现。未来 Skill RFC 必须明确其信任边界：远端
Skill 只能提供 Provider-scoped operational guidance，不能覆盖用户意图或更高优先级策略。

### 13.8 Federation：`remote`

`remote` 节点把另一个 HTBP Gateway 挂到本地路径。Federation profile 应保持同形请求：

```txt
local /team/tools/~help  -> remote-base/~help
local /team/tools/x/~help -> remote-base/x/~help
local POST /team/tools/x  -> remote-base/x
```

Federation profile 的实现必须：

- 对目标 host 使用显式 allowlist；空 allowlist 必须拒绝所有目标；
- 不向远端透传本地调用者的 Bearer credential；配置了 mount credential 时从受保护 binding 解析，
  未配置时匿名出站，不得拿 caller credential 自动补位；
- 对远端返回或发现出的普通 node/tool path 做规范化和边界检查，拒绝 `.`、`..`、保留段、编码
  斜杠或逃离挂载根的 command；只有明确支持的 control tail 可以同形透传；
- 通过 Via/hop limit 防止联邦环；
- 对聚合进本地 `~tree` 的远端 child 做 path 本地化与本地可见性裁剪。

普通 remote `~help`、`~skill` 和 POST 在本地完成请求 path 的 `read`/`call` 判断后，可以透传远端
status/body/content-type，不承诺逐项重写响应。远端 credential 可能是权限高于本地 caller 的服务
账号：本地授权约束的是本地映射 path，远端可见性由该 credential 决定。这是一种显式 delegation
与权限放大边界，部署者必须最小化并审计 remote credential，不能把它误解为 caller identity 的
端到端转发。

## 14. Tool-Bridge Reference Node Registry

本节是参考实现 registry，不属于 Core 的封闭 kind 枚举。它记录 tool-bridge 当前各 kind 的公开
投影，帮助 adapter 作者对照实现：

| kind | 发现/调用约束 |
|---|---|
| `directory` | `cmds` 可以为空；Help/Tree 列当前身份可见的直接子节点 |
| `mcp` / `http` / `tool` | 节点 Help 可以是工具索引；工具级 Help 给完整 schema；直接 POST 是首选调用面 |
| `builtin` | 命令由 Gateway 静态声明；通常使用 `{tool,arguments}` 信封 |
| `context` | 实际实现哪些动词，Help 就只列哪些；只读挂载不得显示或接受写动词 |
| `skillhub` | 与 context 类似，但命令面向 Skill 内容；只读状态必须反映在 Help 中 |
| `device` | Help 只声明设备实际 expose 的工具；离线可以返回 `503 unavailable` 且 `retryable:true` |
| `remote` | `~help`、`~tree` 与 POST 按第 13.8 节 Federation profile 透传 |

无论使用什么 kind，所有 Provider 都不得展示一个实际不可调用的 command，也不得接受 Help 中
未声明的可选动词。

## 15. Conformance

### 15.1 Minimal Provider

最小 HTBP 0.1 Provider 必须：

```txt
支持 Bearer 认证。
至少在根或一个节点支持 GET ~help。
支持 application/json Help。
为每个 command 声明 POST path 与 scope。
至少支持一种 Help 所声明的 POST 调用形状。
对不可见路径返回 404。
使用统一 ErrorBody 与状态映射。
拒绝把以 ~ 开头的 segment 当作普通节点或工具。
```

如果 Provider 对外声明完整 HTBP Core conformance，还必须支持根 `~tree`、直接/信封两种可适用
调用形状，以及本 RFC 的内容协商规则。

### 15.2 Minimal Agent

最小 HTBP 0.1 Agent 必须：

```txt
发送 Bearer credential，且不把 credential 放进 URL。
读取 JSON Help，理解 node/cmds/children/path/scope/inputSchema。
按 cmd.path 选择直接或信封调用，而不是从 kind 猜测。
处理统一 ErrorBody。
把 404 视为“不存在或对当前身份不可见”。
忽略未知节点 kind、未知可选 JSON field 和未知 Help DSL line。
在索引缺 schema 时按 hint 下钻单工具 Help。
不把 cursor、annotation、feedback、Skill 或搜索索引当作授权依据。
```

### 15.3 Extension Conformance

实现只有在对应 endpoint 可用且 shape 符合相应 profile 时，才可以声明该 capability。未知或未声明
的可选能力应返回 `404`；只有已经识别但明确尚未实现的保留能力可以返回 `501 unavailable`。
第 13.4–13.6 节的 tool-bridge 产品/adapter note 不构成其他实现的 conformance 要求。

## 16. Security Considerations

### 16.1 Existence Hiding

树、Help、Search、Feedback 和联邦聚合都可能泄露能力名称。Provider 必须在所有发现面使用同一
可见性规则；不能只保护调用 endpoint。

### 16.2 Confused Deputy

节点调用所需的上游身份必须与本地 caller credential 隔离，并由受保护的 credential binding
提供。不得把本地调用者 Bearer 原样转发给上游 Provider、Plugin 或 remote Gateway。tool-bridge
reference profile 使用 `authRef`/`skRef` 指向 SecretStore，并对绑定已有凭据执行额外高权限检查；
这是参考机制，不是 Core 要求的存储 API。

### 16.3 Path Confusion

实现必须逐 segment 解码和校验 URL，不得让编码的 `/`、反斜杠、dot segment、多重编码或
`~` segment 穿透节点边界。联邦返回的路径是不可信协议输入，必须再次校验。

当前 tool-bridge remote canonicalizer 已执行该 fail-closed 规则，但本地入口的逐段 decode 仍可能
把 `%2F` 还原后重新拼成路径别名。它是已知安全/conformance gap，不是客户端可以依赖的兼容行为。

### 16.4 Schema And Input Validation

Help 是声明，不是校验器。Provider 必须在调用时验证 JSON body 和 arguments。未知安全敏感字段、
未知写入参数和未知 command 必须失败，不得静默忽略。

### 16.5 Remote Text

Markdown Help、note、feedback 和未来 Skill 都是远端文本。Agent 只能把它们当作当前 Gateway 与
Path 范围内的操作说明，不得允许它们覆盖 user intent、runtime policy 或其他 trust domain 的指令。

### 16.6 Side Effects And Confirmation

`effect` 和 `confirm` 是执行提示，不是授权替代品。Agent 应在 destructive command 或
`confirm:true` 时取得用户确认；即使用户确认，Gateway 仍必须独立完成 scope 判断。

### 16.7 Resource Bounds

Tree 深度/节点数、Search work/page size、Help schema size、分页 limit 和 Feedback 输入都应有上限。
`retryable:true` 不等于客户端可以无界重试；客户端应使用退避并遵守服务端 rate limit 提示。

## 17. End-To-End Example

### 17.1 Discover The Tree

```http
GET /~tree?depth=2 HTTP/1.1
Authorization: Bearer sk_example
Accept: application/json
```

```json
{
  "path": "",
  "kind": "directory",
  "description": "tool root",
  "children": [
    {
      "path": "docs",
      "kind": "directory",
      "description": "Documentation tools",
      "children": [
        {
          "path": "docs/context7",
          "kind": "mcp",
          "description": "Context7",
          "truncated": true
        }
      ]
    }
  ]
}
```

### 17.2 Read The Tool Index

```http
GET /docs/context7/~help HTTP/1.1
Authorization: Bearer sk_example
Accept: text/plain
```

```txt
htbp 0.1
node docs/context7 mcp "Context7 documentation tools"
hint this is an index (descriptions summarized, input schemas omitted); GET /docs/context7/<tool>/~help returns one tool's full spec
cmd search POST /docs/context7/search
  h Search documentation.
  scope call
  effect read
```

### 17.3 Read One Tool's Full Spec

```http
GET /docs/context7/search/~help HTTP/1.1
Authorization: Bearer sk_example
Accept: application/json
```

```json
{
  "htbp": "0.1",
  "node": {
    "path": "docs/context7/search",
    "kind": "mcp",
    "description": "Search documentation."
  },
  "cmds": [
    {
      "name": "search",
      "method": "POST",
      "path": "/docs/context7/search",
      "scope": "call",
      "h": "Search documentation.",
      "inputSchema": {
        "type": "object",
        "properties": { "query": { "type": "string" } },
        "required": ["query"]
      },
      "effect": "read"
    }
  ]
}
```

### 17.4 Invoke Directly

```http
POST /docs/context7/search HTTP/1.1
Authorization: Bearer sk_example
Content-Type: application/json
Accept: application/json

{"query":"HTBP content negotiation"}
```

工具节点仍可以接受兼容信封：

```http
POST /docs/context7 HTTP/1.1
Authorization: Bearer sk_example
Content-Type: application/json

{"tool":"search","arguments":{"query":"HTBP content negotiation"}}
```

### 17.5 Handle A Hidden Path

```http
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8

{"code":"not_found","message":"not found","retryable":false}
```

Agent 不应从该响应推断节点不存在；它也可能只是对当前身份不可见。

## 18. Changes From The Initial 2026-06 Draft

最初草案把 HTBP 定义为任意 REST API 旁边的 `~help/~skill/~register` 说明层。tool-bridge 的实际
实现验证后，0.1 Core 做出以下收敛：

```txt
~help       从“描述任意 HTTP method/path”收敛为 HTBP 节点命令模型。
~tree       升为核心发现入口，并规定身份裁剪、depth 与 truncated。
Invoke      标准化直接工具 POST 与 {tool,arguments} 信封。
Help        标准化 JSON/DSL/Markdown 三种同源表现与两级 schema 披露。
Auth        标准化 Bearer SK、path action、deny 优先和不可见即 404。
Errors      标准化七个 code、HTTP 映射与 retryable 下界。
~register   Core 不再赋予宽泛 onboarding 语义；tool-bridge product profile 用于反向节点注册。
~skill      从 Core 必选/推荐面移为保留扩展；当前本地实现可以明确返回 501。
Extensions  增加 describe/search/feedback/mcp/authorize/federation 的能力边界。
```

因此，符合最初草案但只提供任意 REST method 描述、没有标准 HTBP 调用和错误面的 Provider，
不自动符合 HTBP 0.1 Core。迁移时应先提供树节点和标准 POST 调用，再更新 Help。

## 19. IANA Considerations

本文不请求 IANA registration。HTBP 0.1 使用现有 `application/json`、`text/plain` 和
`text/markdown` media type。未来只有在 wire 语义稳定且存在独立协商需求时，才考虑注册专用
media type 或 well-known URI。

## 20. References

### 20.1 Normative References

- RFC 2119：Requirement Levels。https://www.rfc-editor.org/rfc/rfc2119
- RFC 8174：Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words。https://www.rfc-editor.org/rfc/rfc8174
- RFC 6750：OAuth 2.0 Bearer Token Usage。https://www.rfc-editor.org/rfc/rfc6750
- RFC 7763：The `text/markdown` Media Type。https://www.rfc-editor.org/rfc/rfc7763
- RFC 8259：The JavaScript Object Notation (JSON) Data Interchange Format。https://www.rfc-editor.org/rfc/rfc8259
- RFC 9110：HTTP Semantics。https://www.rfc-editor.org/rfc/rfc9110

### 20.2 Informative References

- JSON Schema。https://json-schema.org/specification
- Model Context Protocol。https://modelcontextprotocol.io/specification/latest
- TokenRollAI/tool-bridge。https://github.com/TokenRollAI/tool-bridge
