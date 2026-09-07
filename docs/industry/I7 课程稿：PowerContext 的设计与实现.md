---
title: I7：PowerContext 的设计与实现
outline: deep
---

# I7：PowerContext 的设计与实现

> Easy Data x AI 课程 · 产业应用篇 · 第 7 节

::: tip 本节定位
本节以开源上下文管理系统 PowerContext 为案例，把 I6 的五个数据角色拆成可运行的领域对象与接口：Scope、Source、Memory、PreparedContext 与 Handoff。学习重点是架构边界与可替换接口：先理解每一层为什么这样设计，再学习如何调用。
:::

::: warning 版本提示
本文依据 2026 年 9 月初的 PowerContext 开源实现（oceanbase/powercontext master 分支）与配套文档整理，实验部分在本地最小模式（SQLite、未接入模型）下实测验证。项目仍在快速演进，安装方式、配置项、接口与内部约束应以目标版本文档和实测结果为准。
:::

## 学习目标

完成本节后，你将能够：

1. 说出 PowerContext 的定位，以及它与 PowerMem 的承接关系；
2. 画出 Scope、Source、Artifact、PreparedContext 的数据关系；
3. 解释不可变修订与精确引用为什么能支撑审计；
4. 区分直接写入与候选审核两类治理路径；
5. 说明 Work Contract、Handoff 与 Task Outcome 的交接闭环；
6. 描述 Core SDK、Client、Server、HTTP、MCP 与 CLI 的职责边界；
7. 比较 SQLite、seekDB、OceanBase 三个存储后端与四种检索模式的取舍；
8. 运行一个最小回路实验并解读关键响应字段。

## 1. PowerContext 是什么

PowerContext 是 OceanBase 社区开源的上下文管理系统，官方定位是让项目和 Agent 的上下文可以交接和继续。它是课程里 PowerMem 记忆系统的后续项目，在记忆之外扩展了证据、交接、经验与技能四类资产。

长期使用多个编码 Agent 的团队会遇到一类共性问题：这次会话完成了推理和决策，下次会话或换一个 Agent 接手时，决策依据、约束与进行到一半的任务都留在上一个会话里。PowerContext 把这类内容从聊天记录中拆出来，作为项目数据单独持久化，让人和多种 Agent 共享同一份工作上下文。

### 1.1 与 PowerMem 的承接关系

D4 用 PowerMem 演示了记忆的存储、检索与遗忘。PowerContext 是同一个团队对更大问题的回答：

| 维度 | PowerMem | PowerContext |
| --- | --- | --- |
| 关注对象 | 记忆条目的存取与衰减 | 证据、记忆、交接、经验、技能的完整链路 |
| 资产类型 | 记忆 | 记忆、交接、经验、技能 |
| 证据模型 | 依赖对话输入 | 独立来源层，记录可引用证据 |
| 交付形态 | SDK 接入为主 | 本地服务，HTTP、MCP、CLI 多入口 |
| 延续 | 课程 D4 的实现参考 | 本节的案例对象 |

PowerMem 解决的问题被 PowerContext 保留为记忆资产，同时上下文被当作一类需要版本、引用与生命周期的数据来管理。

### 1.2 四类资产与能力分级

PowerContext 管理四类资产：

| 资产 | 回答的问题 | 生命周期要点 |
| --- | --- | --- |
| Memory | 以后还要影响判断的知识 | 活跃与停用状态，跨会话复用 |
| Handoff | 进行中的工作交给谁、从哪开始 | 随任务推进，可提交为里程碑 |
| Experience | 审核过的判断经验 | 批准后的当前版本才能被引用 |
| Skill | 可重复执行的做法 | 批准后还要显式导出才可执行 |

对应能力按 Profile 分级，接入方可以按需选择：

| 级别 | 覆盖能力 |
| --- | --- |
| 最小 | 记忆读写、来源采集、上下文注入 |
| 推荐 | 任务契约、交接、确认与结果回写 |
| 完整 | 经验沉淀、技能管理、候选审核 |

分级贯穿整节：先跑通最小回路，再按任务需要扩展，避免一上来就承担全部功能。

## 2. 领域模型：归属、证据与不可变资产

I6 讨论的职责在 PowerContext 中落成四类持久对象与一组关系：Scope 划定归属，Source 保存证据，Artifact 保存不可变资产，PreparedContext 是临时装配结果。

![Scope 划定归属边界，Source 与 Artifact 落位，PreparedContext 按请求装配](/images/industry/I7/I7-01-domain-model.png)

### 2.1 Scope：一切数据的归属边界

Scope 是所有数据的隔离边界。来源、记忆、交接历史与统计都归属某个 Scope，检索和装配默认只在当前 Scope 内进行。

Scope 的两个设计要点值得注意：

- Scope 标识由系统生成，是不透明的字符串，应用中把它当普通键使用，不解析其中的任何含义；
- Scope 负责数据归属与隔离，负责不了授权。权限边界要由宿主与接入层的鉴权机制承担。

一个长期项目中，把仓库或工作流绑定到一个稳定的 Scope，所有参与会话共享这一份归属。会话标识、临时目录这类频繁变化的值不能当 Scope 用，否则每次会话都会换一个空的数据空间。

### 2.2 Source：可引用的证据

Source 记录发生过什么。PowerContext 把一次证据事件保存为不可变的观察，每个观察带精确的 SourceRef，引用不会因为内容被重新采集而漂移。

来源有两种物化方式：

| 物化方式 | 内容从哪里来 | 适用情况 |
| --- | --- | --- |
| captured | PowerContext 自己保存的内容快照 | 消息、任务结果等本地可持内容 |
| referenced | 外部不可变位置的引用 | 只能寻址外部对象的场景 |

Work Contract 与 Task Outcome 在系统里也以来源形式保存：委托基线是任务开始时写入的证据，任务结果回写后形成可审计的记录。

### 2.3 Artifact：不可变修订的资产层

PowerContext 用 Artifact 表达可复用的长期资产。一个 Artifact 有稳定的标识，内容按不可变 Revision 演进，精确引用格式为内容族、标识与修订号的组合，例如 `memory/project-notes@3`。

这样的设计带来三条约束：

- 写操作只产生新 Revision，已发布内容不修改；
- 旧 Revision 保持可读，引用旧版本不会悄悄变成新版本；
- Revision 是并发的比较基线，过期写入会被拒绝。

### 2.4 四类 Family 的数据形态

Artifact 按内容族划分，各自保存不同的字段：

| Family | 主要字段 | 本课程的对照 |
| --- | --- | --- |
| Memory | 条目清单、条目正文、活跃状态 | D4 的事实化记忆 |
| Handoff | 目标、状态、下一步、证据、已知缺口 | 跨会话的任务接力 |
| Experience | 情境、动作、结果、教训 | X1-3 的经验沉淀 |
| Skill | 名称、描述、指令、校验方式 | P4 的 Skill 资产 |

记忆条目本身是两层结构：逻辑条目保持不变，正文以不可变条目版本演进；内容哈希把引用与正文锚定在一起，读取时按需校验。课程 X1-1 讨论的冲突裁决与版本链，在这里是内置的存储契约。

### 2.5 数据关系小结

- Source 只做证据，进入不了注入面；
- Artifact 修订记录血缘，血缘来自实际传入的证据引用；
- 批准后的当前版本才能被检索与引用；
- PreparedContext 每次请求重新生成，不落库。

把这条关系链记牢，后面的接口与存储讨论就都有落点。

## 3. 修订、证据引用与生命周期管理

本节的设计围绕三个问题展开：内容变化后如何演进，引用如何保持可信，谁有权把变化发布为资产。

### 3.1 不可变修订

每次有效变更在同一 Artifact 下产生新的 Revision。修改记忆、提交交接、批准经验都会推进 Revision，旧 Revision 继续可读。

并发安全通过显式基线传递实现。客户端带着它认为的当前 Revision 写入，系统校验不匹配就拒绝：

| 场景 | 行为 |
| --- | --- |
| 无基线写入 | 按当前头追加，返回新 Revision |
| 显式基线等于当前头 | 正常提交 |
| 显式基线落后于当前头 | 返回 409 revision_conflict，不自动覆盖 |

实测中，把过期的 `expected_revision` 提交给已经前移的 Artifact，会得到 `revision_conflict` 错误。冲突可见、可重试，旧内容不会被静默覆盖。

### 3.2 精确引用与证据

引用落在三个粒度上：

| 引用对象 | 粒度 | 示例 |
| --- | --- | --- |
| Artifact Revision | 资产与修订号 | `experience/X@1` |
| Source 观察 | 来源与观察序号 | `content/session-7` |
| Memory 条目版本 | 条目与正文版本 | entry_id 与 entry_version_id 的组合 |

检索结果里的命中会带回完整引用，上下文装配把引用原样交给模型侧。引用证明内容可定位，内容是否仍然正确由调用方按版本与来源状态判断，两层职责分开。

### 3.3 直接写入与候选审核

不同内容族采用不同的治理路径，策略固定在内容族上：

| 内容族 | 治理路径 | 原因 |
| --- | --- | --- |
| Memory | 直接写入 Revision | 高频、低风险、需要即时可用 |
| Experience / Skill | 先生成候选，人工审核后发布 | 影响面大、需要人确认 |

候选不是 Artifact，不进入检索，也没有 Artifact 标识。审核动作本身是终态操作：批准在同一事务里写入新 Revision 并返回结果引用，拒绝只记录原因。给候选与目标分别携带版本号，防止审核过程中提案或目标被并发修改。

生成与确认分离是一条值得带走的经验：模型负责生成候选，人负责确认，确认结果成为唯一发布路径。

### 3.4 Handoff 与工作连续性闭环

任务交接在 PowerContext 里是一组连续动作，先把链路拆开看：

```text
Work Contract（委托基线）
  → Prepare → Draft → finalize
  → continue（接手方继续工作）
  → commit（显式提交为里程碑 Revision）
  → acknowledge（确认回执）
  → record Task Outcome（结果回写）
```

设计要点：

- 交接草稿先于提交存在，接手方可以预览，提交后才成为可引用的历史；
- 交接内容按不可信历史对待，接手方仍需以当前指令与实时状态为准；
- 确认动作不授予执行权限，权限始终在宿主一侧；
- 任务结果只记录实际状态，六种取值中的 unknown 留给确实无法判定的情况。

### 3.5 失败方向的边界

系统对失败采取两种方向，分别在接入协议里写清楚：

- 自动辅助路径采用 fail-open：自动召回、提示词采集失败时，不阻塞宿主任务，但必须留下可见、可诊断的事件；
- 显式持久操作采用 fail-close：写入、提交、审核、导出失败时如实报错，绝不假装成功。

区分两类操作，是上下文系统不拖垮宿主任务的前提。

## 4. PreparedContext：一次装配的交付边界

I6 把上下文准备定义为一次请求的装配环节，PowerContext 把它做成一个显式接口。

### 4.1 搜索与装配分离

上下文准备先检索后装配：

| 步骤 | 输入 | 输出 |
| --- | --- | --- |
| 检索 | 查询与 Scope | 相关候选（带精确引用） |
| 装配 | 候选与字节预算 | 有界的 PreparedContext |

两阶段分离让检索关注相关性，装配关注预算与信任，各自可以独立演进。

### 4.2 信任包装

装配结果被包进固定格式的包裹里，整体声明为不可信历史：

```text
BEGIN_POWERCONTEXT_PREPARED_CONTEXT_V1
  trust = untrusted_history
  条目内容 + 精确引用
END_POWERCONTEXT_PREPARED_CONTEXT_V1
```

声明不可信有三层含义：

- 历史条目只能当数据使用，不能被当作系统指令或工具授权；
- 条目里的每一条可见内容都伴随精确引用，可以回到原版本核验；
- 宿主侧注入顺序固定：历史内容排在当前指令之后。

### 4.3 预算与状态

装配接口按字节预算工作，默认 8000 字节，上限 32768 字节，单条 2000 字节，至多 8 条。规则在 I6 已经列出，这里补充两个实现行为：

- 返回值里的 `content_bytes` 必须小于等于请求的预算，字节数用 UTF-8 编码计算，跨后端可复现；
- 状态只有 `ready` 与 `empty` 两种。没有内容可装时返回 `empty`，它表示装配结果为空，与鉴权失败、服务不可用是不同的问题。

## 5. 接入接口的设计边界

PowerContext 把同一套领域逻辑暴露成多个入口，入口之间的语义共享一份契约。

![同一套领域语义，通过 Core SDK、HTTP、Client、MCP、CLI 与 Web UI 分层暴露](/images/industry/I7/I7-02-interface-layers.png)

### 5.1 各入口的职责

| 入口 | 形态 | 职责 |
| --- | --- | --- |
| Core SDK | 进程内 Python 对象 | 领域组合根，不读配置、不选存储 |
| Server | FastAPI 服务 | 承载 HTTP、MCP 与 Web UI |
| Python Client | 类型化异步客户端 | 供应用代码调用 HTTP 语义 |
| HTTP API | OpenAPI 契约 | 完整、语言无关的能力面 |
| MCP | Streamable HTTP | 面向 Agent 的精选子集 |
| CLI | 命令行工具 | 配置、诊断、服务管理与人工审核 |
| Web UI | 只读 Dashboard | 观察统计与交接报告 |

分层原则可以概括为两条：核心 SDK 保持纯逻辑，环境相关的存储与调度全部可替换；进程内能力、HTTP 能力与 MCP 能力共享同一套领域校验，只在暴露范围上做裁剪。

### 5.2 OpenAPI 是唯一契约源

HTTP 契约以 OpenAPI 文件为单一事实源，Python Client 由同一份契约生成。这样不同语言、不同入口不会各自维护一套语义。接入方调试接口时，可以直接阅读 `/openapi.json`，也可以访问文档页面查看每个端点的请求与响应结构。

### 5.3 MCP 与 HTTP 的裁剪

MCP 面向 Agent 运行时的自动调用，暴露的是精选子集：记忆的写入、检索、读取、修订与停用，来源采集，以及交接与任务结果相关操作都可以通过 MCP 完成。

上下文装配、技能导出与经验生成类操作保留在 HTTP 与 CLI 侧，不投影成 MCP 工具。裁剪的原因是职责不同：MCP 服务于运行中的 Agent，装配与人工管理发生在宿主之外。

### 5.4 错误与可观测契约

所有接口返回统一的错误包裹：

```text
HTTP 状态码
  + {"error": {"code": ..., "message": ..., "details": ...}}
  + X-PowerContext-Request-ID 请求头
```

常见语义包括 401 未认证、404 找不到对象、409 版本或状态冲突、422 请求契约不合法、503 能力暂不可用。请求标识贯穿日志与响应，排查问题时可以按标识串联一次请求的完整链路。

插件侧还定义了类型化错误分类与单行 JSON 事件，区分认证失败、版本不匹配、服务不可用与响应格式错误。先定可观测契约再写插件，是这类集成系统值得借鉴的工程顺序。

## 6. 存储后端与检索契约

上下文数据最终落在存储层，检索契约把存储差异挡在接口后面。

### 6.1 三个存储后端

| 后端 | 形态 | 检索能力 | 适用场景 |
| --- | --- | --- | --- |
| SQLite | 本地文件，默认 | FTS5 全文 + sqlite-vec 向量 | 个人与单机开发 |
| seekDB | 嵌入式向量数据库 | 向量与混合检索 | Linux / macOS 本地增强 |
| OceanBase | 分布式数据库 | FULLTEXT 与向量 HNSW | 团队共享与规模化部署 |

![三个存储后端共享同一份检索契约，能力差异通过能力探测暴露](/images/industry/I7/I7-03-storage-backends.png)

后端的差异通过配置切换，领域逻辑与接口层不感知具体实现。课程 I2 讨论过的向量、全文、混合检索概念，在这里变成后端各自实现的索引能力。

### 6.2 检索与事务的一致性

每个后端都要满足同一份存储契约：

- 内容 Revision、当前头投影与检索索引在同一个事务里提交或回滚；
- 头投影与索引可以重建，权威数据只保存一份；
- 检索只命中当前头且状态为活跃的内容。

实测中最能体现这份契约的是启动行为：未配置任何模型时，SQLite 后端依然初始化全文索引，记忆写入与检索立即可用。

### 6.3 向量能力是可选配置

向量与混合检索需要配置 Embedding 模型、向量维度等参数，属于显式能力：

| 情况 | 行为 |
| --- | --- |
| 未配置向量 | 只提供全文与自动模式 |
| 配置向量 | 提供向量与混合检索模式 |
| 显式请求缺失能力 | 返回明确的错误，不静默降级 |

自动模式可以在混合检索不可用时退回全文，但退回行为要如实报告。能力探测接口把这些信息暴露给调用方，客户端在调用前就知道服务端支持什么。

### 6.4 检索流程与可选重排

一次检索在实现上分阶段完成：

```text
召回（全文 / 向量 / 混合）
  → 融合与粗排（混合时按融合常数合并）
  → 可选重排（listwise 排序，默认关闭）
  → 返回带引用的命中
```

重排需要额外调用生成模型，默认关闭。开启重排时，模型只输出候选的排序位置，返回的引用与身份由系统映射，模型侧不能虚构内容标识。

### 6.5 工程取舍小结

| 取舍问题 | 结论 |
| --- | --- |
| 要不要向量检索 | 先看查询是否需要语义召回；不需要就不配置 |
| 用什么后端 | 单机先 SQLite，规模化再评估分布式 |
| 显式还是自动模式 | 显式请求缺失能力要报错，自动模式降级要如实 |
| 开不开重排 | 默认关，按质量需求评估额外延迟 |

课程 I6 强调过能力要按需引入，这里的结论一致：检索能力按任务质量需求逐级启用，每级都先验证收益。

## 7. 一个最小可运行实验

本节实验在本地最小模式下完成：SQLite 存储、未接入模型、未启用鉴权。官方安装说明以 macOS / Linux 与 Python 3.11 以上为主；课程验证环境为 2026 年 9 月 7 日的本地 Windows 与 Python 3.12，同一份 HTTP 契约下跑通。

### 7.1 安装与启动

按官方快速入门安装 CLI 与 Server：

```bash
uv tool install "powercontext[cli,server] @ git+https://github.com/oceanbase/powercontext.git@master"
```

启动服务：

```bash
powercontext server run
```

默认监听 `127.0.0.1:8000`。用下面的命令确认服务健康：

```bash
powercontext ready
powercontext capabilities
```

最小模式下，`capabilities` 会显示四类内容族与 `auto`、`fts` 两种检索模式，模型相关能力为关闭状态。

### 7.2 最小回路

下面用标准库脚本完成创建 Scope、写入记忆、检索与装配四个动作。响应内容较长，脚本只打印要点：

```python
import json
import urllib.request

BASE = "http://127.0.0.1:8000"

def api(path, method="GET", payload=None):
    req = urllib.request.Request(BASE + path, method=method)
    if payload is not None:
        req.add_header("Content-Type", "application/json")
        req.data = json.dumps(payload).encode("utf-8")
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode("utf-8"))

# 1. 创建 Scope，返回由服务端生成的不透明 scope_id
scope = api("/v1/scopes", "POST", {
    "title": "I7 demo",
    "summary": "PowerContext 课程实验",
    "idempotency_key": "demo-scope-1",
})
sid = scope["scope_id"]
print("scope:", sid)

# 2. 显式写入一条记忆
remember = api("/v1/memory/remember", "POST", {
    "scope_id": sid,
    "kind": "constraint",
    "text": "回答存储后端取舍问题时必须给出适用条件。",
})
print("revision:", remember["memory"]["revision"])
print("entry_id:", remember["entry"]["citation"]["entry_id"])

# 3. 检索记忆
search = api("/v1/memory/search", "POST", {
    "scope_id": sid,
    "query": "存储后端 取舍 约束",
    "mode": "auto",
    "limit": 5,
})
print("mode:", search["mode"])
for hit in search["hits"]:
    print("hit:", hit["citation"]["memory_ref"]["revision"], hit["matched_by"])

# 4. 装配本次请求的上下文
prepared = api("/v1/context/prepare", "POST", {
    "scope_id": sid,
    "query": "存储后端取舍",
    "max_bytes": 1024,
})
print("status:", prepared["status"])
print("content_bytes:", prepared["content_bytes"])
```

示例脚本的打印含义：

| 打印项 | 含义 |
| --- | --- |
| `scope:` 开头的字符串 | 服务端生成的不透明 Scope 标识 |
| `revision: 1` | 第一条记忆写入产生 Revision 1 |
| `entry_id:` 开头的字符串 | 该条记忆的逻辑条目标识 |
| `mode: fts` | 未配置向量模型时，自动模式落到全文检索 |
| `hit:` 行 | 命中条目携带 Revision 与命中通道 |
| `status: ready` | 装配成功，内容非空 |
| `content_bytes:` 数值 | 实际内容字节数，小于等于请求的 1024 |

### 7.3 冲突行为验证

先在第 7.2 步的 Scope 上再次写入一条记忆，把 Artifact 的当前 Revision 推进到 2，然后携带过期的 `expected_revision: 1` 写入：

```python
api("/v1/memory/remember", "POST", {
    "scope_id": sid,
    "kind": "constraint",
    "text": "第二次写入，Revision 前移到 2。",
})

resp = api("/v1/memory/remember", "POST", {
    "scope_id": sid,
    "kind": "constraint",
    "text": "基于旧基线的写入。",
    "expected_revision": 1,
})
```

当前 Revision 已经是 2 时，第二步得到 HTTP 409 响应。标准库在收到非 2xx 状态时会抛出 `HTTPError`，响应体里的错误码为 `revision_conflict`。把 `expected_revision` 换成服务端返回的当前 Revision 即可正常提交。

### 7.4 清理

停止服务进程，删除实验数据目录即可。数据目录位置由配置中的 `POWERCONTEXT_HOME` 决定，未设置时使用默认用户数据目录。

## 8. 能力边界

当前课程不把以下能力作为使用 PowerContext 的前提：

- 未配置模型时的记忆提炼、经验生成与交接生成；
- 把任何 Scope 当作安全边界；
- 让历史注入内容自动获得指令或授权地位；
- 通过 MCP 调用全部 HTTP 能力；
- 把候选审核省略成纯模型判断；
- 引用旧内容时假设内容仍然正确。

需要跨团队共享、分布式部署或更细粒度权限时，需要额外配置存储、鉴权与部署方案，这些内容超出本节的单机范围。

## 9. 一套最小验收流程

接入或自研类似系统时，可以按下面的顺序验收：

1. 用最小模式启动服务，确认无模型也能完成记忆写入与全文检索；
2. 创建专属 Scope，确认数据归属与隔离生效；
3. 写入一条记忆并跨会话检索，确认引用字段完整；
4. 携带过期基线写入，确认得到显式的版本冲突；
5. 装配一次上下文，确认字节预算与信任包装生效；
6. 提交一次交接，用新会话续接并回写任务结果；
7. 开启向量配置，确认能力探测反映新模式；
8. 记录每次请求的耗时与注入字节，对照任务质量回归。

第 5 步与第 7 步最能暴露实现与文档的漂移，建议在每次版本升级后重跑。

## 10. 常见误区

### 10.1 误区一：用会话标识当 Scope

会话结束就要换新的数据空间，长期记忆与交接全部落空。Scope 应绑定长期稳定的项目或工作流标识。

### 10.2 误区二：装了 MCP 就拥有全部能力

MCP 暴露的是面向 Agent 的精选子集，上下文装配与生成类操作保留在 HTTP 与 CLI 侧。接入前先对照能力表确认入口是否覆盖任务。

### 10.3 误区三：向量检索一定优于全文检索

向量能力需要模型配置与索引维护，全文在术语精确场景更稳定。检索方式的取舍按查询类型与数据规模决定，课程 D2 与 I2 的结论在上下文系统中同样成立。

### 10.4 误区四：批准内容等于可以执行

Experience 批准后可以进入检索与引用，Skill 批准后还要显式导出，导出目标与授权由宿主侧决定。批准只表示内容通过审核。

### 10.5 误区五：没有模型就用不了

记忆写入、全文检索与装配在无模型最小模式下完整可用。模型影响的是提炼、经验生成这类增强能力，属于可选项。

## 11. 与其他课程的关系

- P3《Agent 记忆系统设计》：记忆设计决策与信任边界；
- P4《Skill 与 Agent 知识管理》：Skill 资产与按需加载；
- D2《AI 应用的数据层》：向量、全文与混合检索基础；
- D4《Agent 开发与记忆系统》：记忆系统的最小实现对照；
- X1 系列《探究 AI Agent 记忆系统》：记忆生命周期与冲突处理；
- X2《多 Skill 给上下文工程带来的麻烦》：全量注入问题的实证；
- I2《向量数据库与 RAG》：检索后端与索引能力；
- I6《上下文工程概述》：本节的架构分析框架；
- I8《案例场景和测评构建》：本节的系统接入测评链路。

## 参考资料

- [PowerContext 开源仓库](https://github.com/oceanbase/powercontext)
- [PowerContext 官方文档](https://oceanbase.github.io/powercontext/)
- [PowerContext 记忆层设计 RFC](https://github.com/oceanbase/powercontext/blob/master/docs/zh/rfcs/0014_memory_layer_design.md)
- [PowerContext 端到端测评架构 RFC](https://github.com/oceanbase/powercontext/blob/master/docs/zh/rfcs/0081_end_to_end_evaluation_architecture.md)
- [PowerMem 开源仓库](https://github.com/oceanbase/powermem)
- [I7 课程共建 Issue #96](https://github.com/datawhalechina/easy-data-x-ai/issues/96)

::: info 共建说明
欢迎在课程共建 Issue [#96](https://github.com/datawhalechina/easy-data-x-ai/issues/96) 中补充案例、实验、图示和评测方法。
:::
