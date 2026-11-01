# DeepSeek Harness 完整学习手册

> 本手册面向第一次接触 `deepseek-harness` 的开发者，把项目简介、总体架构、核心功能原理与开发实践步骤整理成一份可通读的文档。文中所有文件引用都使用相对路径链接，可直接点击跳转到仓库对应文件。

## 阅读导航

- [项目简介](#项目简介)
- [功能架构图](#功能架构图)
- [总体架构](#总体架构)
- [核心功能原理](#核心功能原理)
- [插件原理](#插件原理)
- [关键概念源码实现详解](#关键概念源码实现详解)
- [Agent Loop 详解](#agent-loop-详解)
- [ACP（Agent Client Protocol）](#acpagent-client-protocol)
- [Hooks 钩子桥接](#hooks-钩子桥接)
- [开发实践步骤](#开发实践步骤)
- [文件索引](#文件索引)
- [命令速查](#命令速查)

---

## 项目简介


### 一句话概括

DeepSeek Harness（CLI 名称 `dsh`）是 DeepSeek AI 开源的 **Agent Harness**：一个构建在 [Cordis](https://github.com/cordiverse/cordis) 之上的、**一切都是插件（everything-is-a-plugin）** 的智能体运行时骨架。模型适配器、工具注册表、会话日志、文件系统、Shell、Agent 循环本身，都是可替换、可组合的插件。

### 它解决什么问题

- **能力可替换**：换 LLM adapter、换沙箱 provider、换工具后端，不需要改 Agent 循环。
- **一切可扩展**：新能力通过挂载插件进入系统，而不是改核心代码。
- **可观测可回放**：任何到达模型请求的内容都能从会话日志重建（“模型可见即已记录”）。
- **多形态交付**：同一套插件树可装配为 Web UI、Headless CLI、SDK JSON-RPC、ACP 自动化服务。

### 技术底座

| 项 | 值 |
| --- | --- |
| 框架 | Cordis（vendored 到 [`vendor/`](../vendor/README.md)） |
| 范式 | everything-is-a-plugin |
| 语言 | TypeScript，ESM only，`strict: true` |
| 运行时 | Node.js `>= 24.9`，Corepack + pnpm `11.28.5` |
| 包管理 | pnpm workspaces，所有包为 `@deepseek-ai/dsh-*` |
| 当前版本 | `0.2.2-alpha.2`（developer preview，兼容性会快速破坏性变更） |

来源：[`README.md`](../README.md)、[`package.json`](../package.json)。

### 运行形态（Profile）

Profile 是“一组有序 bundle + 补丁”的命名组合，装配出不同产品形态：

- `web`：默认 Web UI，`npx @deepseek-ai/dsh web`。
- `headless`：一次性任务运行器，`pnpm dsh --profile headless "task"`。
- `sdk` / `sdk-minimal`：SDK JSON-RPC 服务。
- `acp`：仅自动化的 Agent Client Protocol 服务。

它们共享第一层 [`dsh-base`](../packages/bundle/base/README.md)（模型适配器、工具、持久化、沙箱与审批策略、设置、凭据、遥测）。

### 从源码运行

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

真实 API 演示需要 `DEEPSEEK_API_KEY`（可放在根目录 `.env`）；Cordis 教程本身无需 key。

---

## 功能架构图


把 profile 装配、Cordis 服务总线、能力接缝、Agent Loop、工具流水线、会话日志与多入口放在一张图里：

```mermaid
flowchart TB
  subgraph Entry[入口]
    WEB[Web UI]
    SDK[JSON-RPC SDK]
    ACP[ACP stdio]
    HEAD[Headless CLI]
  end

  subgraph Assembly[装配层]
    PROFILE[Profile / Bundles]
    PATCH[cordis.patch.yml 叠加]
  end

  subgraph Bus[Cordis Context 服务总线]
    LLM[ctx.llm]
    TOOLS[ctx.tools]
    SHELL[ctx.shell]
    FS[ctx.fs]
    SANDBOX[ctx.sandbox]
    EVENTS[ctx.events]
    SESSIONCTX[ctx.sessions]
  end

  subgraph Loop[Agent Loop]
    INBOX[Inbox 投影]
    PHASE[Phase 状态机]
    PROMPT[system-prompt/assemble]
    MODEL[model selection]
    LOOPEXEC[tool executor]
  end

  subgraph Persist[持久化]
    LOG[Session Event Log]
    PROJ[ctx.sessionProjections]
  end

  WEB --> PROFILE
  SDK --> PROFILE
  ACP --> PROFILE
  HEAD --> PROFILE
  PROFILE --> PATCH
  PATCH --> Bus

  LLM -->|agent/request → llm/stream| LOOPEXEC
  TOOLS --> LOOPEXEC
  SHELL --> LOOPEXEC
  FS --> LOOPEXEC
  SANDBOX --> LOOPEXEC
  EVENTS --> Loop

  INBOX --> PHASE
  PHASE --> PROMPT
  PROMPT --> MODEL
  MODEL --> LOOPEXEC
  LOOPEXEC --> LOG
  LOG --> PROJ
  PROJ --> Entry
```

- 入口层：四种 profile 都是同一套插件的不同装配。
- 服务总线：能力接缝以 `ctx.<key>` 形式存在，provider 可替换。
- Agent Loop：消费 inbox、驱动 phase、组装提示、选择模型、调用工具执行器。
- 持久化：loop 与工具流水线把事实写进日志，投影供 UI/SDK 读取。

---

## 总体架构


### 一切都是插件

`dsh` 没有需要打补丁的“特权核心”。每个功能都是一个挂在共享 `Context` 上的 Cordis 插件——包括模型适配器、工具注册表、会话日志和 Agent 循环本身。扩展系统的方式是“在旁边再挂一个插件”，而不是修改循环。

```mermaid
flowchart LR
  subgraph Host[运行中的 dsh 进程]
    Ctx[共享 Context]
    P1[模型适配器插件]
    P2[工具注册表插件]
    P3[会话日志插件]
    P4[Agent 循环插件]
    P5[沙箱/审批插件]
    P1 --> Ctx
    P2 --> Ctx
    P3 --> Ctx
    P4 --> Ctx
    P5 --> Ctx
  end
  User[用户 / SDK] --> Host
  LLM[LLM Provider] <--> P1
```

### 系统组成：Profile → Bundle → Plugin

运行中的 `dsh` 是一棵在启动时按“有序 layer”组合出来的插件树。

```mermaid
flowchart TD
  subgraph Layers[从下到上的装配层]
    B[Profile 的有序 bundles]
    P[Profile 的 cordis.patch.yml]
    H[用户 home 补丁]
    C[命令行 patch]
  end
  B --> P --> H --> C
  C --> Tree[最终插件树]
  B1[dsh-base 第一层] --> B
  App[web-app / headless / sdk-app / acp-app] --> B
```

- **Profile**：命名组合，列出叠加的 bundle、持有树外插件，并保存用户的 `cordis.patch.yml`。
- **Bundle**：Cordis 配置行 + 所挂载代码的分发格式；其插入内容仍可被上层补丁覆盖。
- **Layer** 从上到下依次应用：profile 的有序 bundle → profile 补丁 → 用户 home 补丁 → 命令行 patch。

来源：[`docs/architecture.md`](../docs/architecture.md)、[`packages/bundle/README.md`](../packages/bundle/README.md)。

### 能力接缝的总体视图

一个可替换能力（seam）由三个角色组成，这是 `dsh` 保持“换 provider 即换产品”的核心机制。

```mermaid
flowchart LR
  SD[Service Definition<br/>声明接口] --> SP[Service Provider<br/>实现接口]
  SP --> C[Consumer<br/>通常是模型工具]
  C --> Model[模型可见的工具调用]
```

文件系统与子进程 provider 共享同一执行世界，因此把它们指向远程沙箱，Bash、PTY、LSP 会一起跟随，无需 provider 分叉。

来源：[`docs/capability-seams.md`](../docs/capability-seams.md)。

### 目录结构

```text
deepseek-harness/
├── apps/                 # 可运行应用：CLI、Web/Desktop 入口
├── packages/             # 全部 npm 包，按能力分组
├── vendor/               # vendored Cordis 源码 + 同步说明
├── native/               # 原生 Node addon（Landlock 沙箱等）
├── python/               # Python SDK 与打包运行时
├── docs/                 # 架构、子系统、生成图、教程、指南
├── scripts/              # 门禁与代码生成器（gen-* / verify-*）
├── snapshots/            # keyless 快照测试期望输出
├── benchmarks/           # 性能基准
├── website/              # VitePress 文档站点（docs 的投影）
└── tmp/                  # 被 gitignore 的临时/学习目录
```

完整分组说明见 [`packages/README.md`](../packages/README.md) 与 [`docs/architecture.md`](../docs/architecture.md)。

---

## 核心功能原理

### 基础概念关系（大白话）

把这几个词按“从大到小、从装配到运行”串成一句话：

```text
profile（配方） 决定这次用哪些 bundle
  → bundle（配件包） 打包一组插件和配置行
    → plugin（功能模块） 被加载后执行 apply(ctx, config)
      → fiber（插件管家） 记录这个插件做过的所有登记
        → ctx（共享总线） 挂载和查找所有 service
          → service（能力） 例如 ctx.llm / ctx.tools / ctx.shell
          → effect（带撤销的登记） 返回 disposer，由 fiber 逆序撤销
          → event（通信管道） 插件之间互相通知、拦截
```

- **profile**：决定“这次 `dsh` 长什么样”。它列出要叠哪些 bundle 和补丁层；`web`、`headless`、`sdk`、`acp` 就是不同的 profile。
- **bundle**：把一组相关插件和它们的 `cordis.yml` 配置行打包，方便复用和叠加。
- **plugin**：最小功能单元。函数、对象、类三种写法，加载时执行 `apply(ctx, config)`。
- **fiber**：plugin 被加载后创建的运行时实例，像管家一样收集该插件做的所有 `effect`，卸载时按逆序全部撤销。
- **ctx（Context）**：所有插件共享的“服务仓库”。插件把能力挂到 `ctx.xxx`，也从 `ctx.xxx` 取别人的能力。
- **service**：挂在 `ctx` 上的一个可替换能力，如 `ctx.llm`、`ctx.tools`、`ctx.shell`。
- **effect**：一次“可撤销的登记动作”。`ctx.on()`、`ctx.effect()`、注册服务都会返回 disposer；fiber 收集后逆序执行。
- **inject**：插件启动前声明的“依赖清单”，等需要的 service 都就绪才加载。
- **event**：类型化消息通道；`emit` / `parallel` / `serial` / `bail` / `waterfall` 决定通知方式。

```mermaid
flowchart LR
  Profile[profile 配方] --> Bundle[bundle 配件包]
  Bundle --> Plugin[plugin 功能模块]
  Plugin --> Fiber[fiber 运行时管家]
  Fiber --> Ctx[ctx 共享总线]
  Ctx --> S1[service: ctx.llm]
  Ctx --> S2[service: ctx.tools]
  Ctx --> S3[service: ctx.shell]
  Plugin --> Inject[inject 依赖清单]
  Inject --> Ctx
  Fiber --> Effect[effect: 收集 disposer]
  Effect --> Undo[卸载时逆序撤销]
  Ctx --> Event[event 通信管道]
```


### Cordis 五大核心思想

1. **插件是实现 `Service` 的对象**：可以是一个带 `inject` / `apply(ctx)` 的函数，也可以是 `Service` 子类。
2. **Context 是服务仓库**：服务占据稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件按 key 查找服务，而非 import 具体实现。
3. **用 `inject` 声明依赖**：插件声明需要哪些服务后，会等这些服务存在再挂载；加载顺序由服务依赖表达。
4. **用类型化事件通信**：通过 TypeScript 声明合并定义事件名，再按 `emit` / `waterfall` / `parallel` / `serial` / `bail` 派发。
5. **注册是可逆副作用**：提示段、工具 schema、adapter、provider、监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，插件卸载时自动回滚。

来源：[`docs/cordis-primer.md`](../docs/cordis-primer.md)。

### 事件派发模式

| 模式 | 是否 await | 派发顺序 | 是否有返回值 |
| --- | --- | --- | --- |
| `emit` | 否 | 注册顺序观察 | 无 |
| `waterfall` | 否 | 注册顺序观察 | 有（`next()` 委托/短路） |
| `parallel` | 是 | 并行观察 | 无 |
| `serial` | 是 | 注册顺序观察 | 有 |
| `bail` | 否 | 注册顺序直到第一个 bail | 有 |

关键规则：**waterfall 监听器必须调用 `next()` 才能委托**；不调用就短路整条链。事件模式属于公开契约，用 `@mode` 标注。

```mermaid
flowchart LR
  A[emit: 观察] --> B[waterfall: next 委托/短路]
  B --> C[parallel: 并行]
  C --> D[serial: 顺序]
  D --> E[bail: 首个 bail 停止]
```

### 事件是怎么派发和运行的（大白话）

一句话总结：**先注册到 `_hooks`，派发时先 `_resolve` 过滤出可见监听器，再按 `emit / parallel / serial / bail / waterfall` 五种模式之一运行。**

- **注册**：插件调用 `ctx.on(name, listener)`，`EventsService` 把监听器按 `prepend` 决定头插/尾插，写进 `_hooks[name]`，并用 `ctx.fiber.effect()` 登记 disposer；插件卸载时自动 `unregister`。见 [`events.ts#L294`](../vendor/cordis/src/events.ts#L294)、[`events.ts#L263`](../vendor/cordis/src/events.ts#L263)。
- **派发前解析**：`_resolve()` 先取可选的 `thisArg` 和事件名，再按 context filter 过滤出可见监听器，得到本次要跑的 callback 列表。见 [`events.ts#L162`](../vendor/cordis/src/events.ts#L162)。
- **运行模式**：
  - `emit`：同步顺序跑，忽略返回值（[events.ts#L203](../vendor/cordis/src/events.ts#L203)）。
  - `parallel`：并发执行，`Promise.allSettled` 等全部结束，有错抛 `AggregateError`（[events.ts#L191](../vendor/cordis/src/events.ts#L191)）。
  - `serial`：顺序 await，遇到首个 bail 值返回（[events.ts#L214](../vendor/cordis/src/events.ts#L214)）。
  - `bail`：同步顺序跑，遇到首个 bail 值返回（[events.ts#L228](../vendor/cordis/src/events.ts#L228)）。
  - `waterfall`：洋葱链；每个监听器收到 `(...args, next)`，调 `next()` 进入下一层，最终调用内置 `inner`；不调 `next()` 就短路，且 `next()` 重复调用会抛错（[events.ts#L246](../vendor/cordis/src/events.ts#L246)）。
- **bail 判定**：返回值不是 `null` / `false` / `undefined` 就视为 bail（[events.ts#L12](../vendor/cordis/src/events.ts#L12)）。

```mermaid
flowchart TB
  subgraph Register[注册阶段]
    ON[ctx.on name, listener] --> PUSH[写入 _hooks name]
    PUSH --> EFF[ctx.fiber.effect 登记 disposer]
    EFF --> UNREG[卸载时 unregister]
  end
  subgraph Dispatch[派发阶段]
    CALL[ctx.emit / parallel / serial / bail / waterfall] --> RESOLVE[_resolve 过滤可见监听器]
    RESOLVE --> LIST[_hooks name 回调列表]
    LIST --> MODE[按模式执行]
  end
  MODE --> EMIT[emit: 同步顺序, 忽略返回值]
  MODE --> PARALLEL[parallel: Promise.allSettled 并发]
  MODE --> SERIAL[serial: 顺序 await 到 bail]
  MODE --> BAIL[bail: 同步到 bail]
  MODE --> WATERFALL[waterfall: next 洋葱链]
  WATERFALL --> NEXT[next 进入下一层, 不调即短路]
```



### 能力接缝（Seam）

一个接缝由 Service Definition、Service Provider、Consumer 三角色组成；一个包可以合并多个角色，但只实现一个角色不算接缝。

```mermaid
flowchart TD
  subgraph Shell[以 shell 能力为例]
    SD[ctx.shell Service Definition]
    SP[shell-local Provider]
    CS[shell Consumer / 模型工具]
    SD --> SP --> CS
  end
  Sub[ctx.subprocess Provider] --> SP
  Model[模型请求] --> CS
```

来源：[`docs/capability-seams.md`](../docs/capability-seams.md)、[`packages/shell/README.md`](../packages/shell/README.md)。

### Agent 生命周期：turn → step → tool call

一次模型调用被组织为 **turn（回合）→ step（步骤）→ tool call** 三层生命周期。持久回放事实写入 `session/event`，实时控制/状态走 `agent/*`。

```mermaid
sequenceDiagram
  participant U as User
  participant A as Agent
  participant D as Driver(循环)
  participant S as Session
  participant LLM as ctx.llm
  participant T as ctx.tools
  U->>A: followup(content)
  A->>D: 唤醒 driver
  D->>S: turn/start
  D->>S: step/start
  D->>LLM: agent/request → prepareCall → llm/stream
  LLM-->>D: StreamChunk*
  D->>S: assistant/message
  D->>T: classify pending tool calls
  T-->>S: tool/call → tool/result
  D->>S: step/end
  D->>S: turn/end
  A-->>U: agent/assistant-stream / UI 状态
```

要点：

- 持久事件：`turn/start`、`step/start`、`system/message`、`user/message`、`assistant/message`、`tool/call`、`tool/result`、`turn/end`。
- 实时事件：`agent/status`、`agent/inbox/*`、`agent/request`、`agent/assistant-stream`。
- `system-prompt/assemble` 是 waterfall，按 step 的工具集组装提示。
- `agent/pre-step` 可权威拒绝或进入一个 step；`agent/turn-stopping` 是回合终点串行检查点。

来源：[`docs/agent-lifecycle.md`](../docs/agent-lifecycle.md)。

### 工具执行流水线

工具执行不改变循环，而是挂在一组共享事件上。

```mermaid
flowchart TD
  A[assistant 消息含 tool-call block] --> B[Session: tool/call]
  B --> C[tools/pre-execute waterfall<br/>hooks / 权限 / 沙箱]
  C --> D[单调 guards 允许或拒绝]
  D -->|允许| E[ctx.approval 一次性询问]
  D -->|拒绝| F[跳过工具主体]
  E -->|允许一次| G[tools/execute waterfall<br/>超时 / 重试 / 指标]
  E -->|拒绝/取消/不可用| F
  G --> H[工具 execute 主体]
  H --> I[fs 写/编辑意图检查]
  I --> J[tools/post-execute waterfall]
  J --> K[finalizeContent]
  K --> L[tools/result 冻结最终结果]
  L --> M[Session: tool/result]
  F --> J
```

`tools/result` 观察的是不可变、可无损 JSON 化的最终结果，因此 hooks 能跨工具族工作，又不让工具耦合到某个策略服务。

来源：[`docs/tool-execution-pipeline.md`](../docs/tool-execution-pipeline.md)。

### 会话持久化：模型可见即已记录

```mermaid
flowchart LR
  ModelReq[模型请求] -->|必须可重建| SessionLog[Session Event Log]
  SessionLog --> Projection[ctx.sessionProjections 投影]
  Projection --> View[客户端快照 snapshot]
```

- 任何到达模型请求的内容都必须能从日志重建；新增模型可见输入必须新增会话事件。
- `dsh-session-projection` 提供 `ctx.sessionProjections`，读取方通过 `snapshot()` 获取客户端视图。
- 会话日志版本由 `SESSION_FORMAT_VERSION` 机制管理，只有结构性格式变更才升版本。

来源：[`packages/core/session/README.md`](../packages/core/session/README.md)、[`packages/session/session-projection/README.md`](../packages/session/session-projection/README.md)。

### 关键服务与扩展点速查

| 目标 | 机制 |
| --- | --- |
| 加模型 provider | 在 `ctx.llm` 注册 adapter |
| 加模型可见能力 | 注册到 `ctx.tools`，schema 加入提示组装 |
| 给某会话换能力集 | 组合 agent preset（服务行需 `isolate` realm） |
| 加 shell 执行 | 注册 `ctx.shell` 后端（本地实现经 `ctx.subprocess` 派生） |
| 加人类命令 | 注册到 `ctx.commands` |
| 加后台任务 | 注册到 `ctx.jobs` |
| 加文件访问/策略 | 注册 `ctx.fs` provider 或监听 `fs/*` 事件 |
| 隔离子进程 | 使用 `ctx.sandbox` 后端，在派生前包装 argv |
| 拦截请求/工具/回合 | 监听 `agent/*` 或 `tools/*` 事件 |
| 加模型可见上下文 | 调用 `agent.inject()` |
| 加持久会话状态 | 扩展 `SessionEventMap`，从日志渲染与回放 |

---

## 插件原理


Cordis 里“插件”是运行时组织单位：它声明依赖、贡献服务、监听事件、产生副作用，并随 fiber 卸载被撤销。源码在 vendored [`vendor/cordis/src/`](../vendor/cordis/src/)。

### 插件的三种形状

- 函数插件：`(ctx, config) => disposer?`。
- 对象插件：`{ apply(ctx, config), inject?, Config? }`；判定条件是 `apply` 是函数（[registry.ts#L9](../vendor/cordis/src/registry.ts#L9)）。
- 类插件：继承 `Service`，构造器里 `super(ctx, name)` 注册服务。

统一声明见 [`Plugin` 类型](../vendor/cordis/src/registry.ts#L91)，对象插件见 [registry.ts#L130-L132](../vendor/cordis/src/registry.ts#L130)。

### 加载与生命周期

```mermaid
flowchart TD
  A[cordis.yml 声明插件] --> B[Loader 解析配置行]
  B --> C[RegistryService.plugin 归一化形状]
  C --> D[resolveConfig 校验 Config schema]
  D -->|失败| E[ValidationError 响亮失败]
  D -->|通过| F[Inject.resolve 等待依赖服务]
  F --> G[创建 Fiber 并 apply / construct]
  G --> H[ctx.effect / ctx.on 收集 disposer]
  H --> I[运行中]
  I --> J[插件卸载：disposer 逆序执行]
```

- `RegistryService.plugin()` 创建 runtime 与 Fiber：见 [registry.ts#L307-L330](../vendor/cordis/src/registry.ts#L307)。
- 配置校验在 fiber 启动前，失败抛 [`ValidationError`](../vendor/cordis/src/fiber.ts#L19)（[fiber.ts#L50](../vendor/cordis/src/fiber.ts#L50)）。
- 依赖声明 [`Inject`](../vendor/cordis/src/registry.ts#L19) 由 [`Inject.resolve()`](../vendor/cordis/src/registry.ts#L71) 归一化；服务不满足时插件不会提前挂载。

### 服务注册：`ctx.<key>`

`Service` 构造器最终调用 `ctx.reflect.provide(name, self, check)`（[service.ts#L42-L58](../vendor/cordis/src/service.ts#L42)），服务随 owning fiber 卸载自动移除。`Context` 构造器内置 `reflect / registry / events / logger` 四个服务（[context.ts#L70-L84](../vendor/cordis/src/context.ts#L70)）。

### 副作用与事件绑定

- [`Fiber.effect()`](../vendor/cordis/src/fiber.ts#L424) 收集 disposer，卸载时逆序执行（[fiber.ts#L433-L448](../vendor/cordis/src/fiber.ts#L433)）。
- [`EventsService.on()`](../vendor/cordis/src/events.ts#L294) 通过 `fiber.effect()` 注册监听，卸载时 `unregister`（[events.ts#L263-L269](../vendor/cordis/src/events.ts#L263)）。

---

## 关键概念源码实现详解


本节把前文概念落到源码，标注了可直接跳转的文件与行号（路径相对本手册，位于 `learnings/` 下，故以 `../` 开头）。

### 1. 插件、Service 与 Context 的实现

Cordis 的核心类型都在 vendored 运行时 [`vendor/cordis/src/`](../vendor/cordis/src/)。

- [`Service`](../vendor/cordis/src/service.ts#L11) 是所有服务的抽象基类；子类在构造器里调用 `super(ctx, name)`，最终通过 `ctx.reflect.provide(name, self, check)` 完成注册（[service.ts#L42-L58](../vendor/cordis/src/service.ts#L42)）。
- [`Context`](../vendor/cordis/src/context.ts#L42) 的构造器安装四个内置服务：`reflect`（反射/提供）、`registry`（插件注册）、`events`（事件总线）、`logger`（[context.ts#L70-L84](../vendor/cordis/src/context.ts#L70)）。
- 插件入口有三种形状：函数、`{ apply(ctx, config) }` 对象、`Service` 子类，声明见 [`registry.ts#L91-L110`](../vendor/cordis/src/registry.ts#L91)；对象插件的判定是“有 `apply` 函数”（[registry.ts#L9](../vendor/cordis/src/registry.ts#L9)）。
- `RegistryService.plugin()` 归一化插件形状、创建 Fiber 并启动，[registry.ts#L307-L330](../vendor/cordis/src/registry.ts#L307)。

```mermaid
flowchart LR
  Plugin[插件函数 / apply 对象 / Service 子类] --> Registry[RegistryService.plugin]
  Registry --> Fiber[Fiber 运行时实例]
  Fiber --> Context[共享 Context]
  Context --> Reflect[ctx.reflect.provide]
  Context --> Events[ctx.events 事件总线]
  Context --> Logger[ctx.logger]
```

### 2. 依赖注入与配置校验

- 依赖声明类型 [`Inject`](../vendor/cordis/src/registry.ts#L19)：数组表示“需要这些服务”，对象表示“服务名 → intercept 配置”。
- [`Inject.resolve()`](../vendor/cordis/src/registry.ts#L71) 把数组/对象/类继承的 inject 元数据归一化为服务名映射；`@Inject()` 装饰器最终调用 `ctx.inject()`，[registry.ts#L37-L59](../vendor/cordis/src/registry.ts#L37)。
- 配置校验在 Fiber 启动前执行：[`resolveConfig()`](../vendor/cordis/src/fiber.ts#L50) 用 standard-schema 校验，失败抛 [`ValidationError`](../vendor/cordis/src/fiber.ts#L19)。这正是“坏配置响亮失败”的实现点。

### 3. 可逆副作用 `ctx.effect()` 的实现

- [`Effect`](../vendor/cordis/src/fiber.ts#L83) 接受一个 disposer、一个 Promise<disposer>，或产出多个 disposer 的 iterable。
- [`Fiber.effect()`](../vendor/cordis/src/fiber.ts#L424) 立即执行 effect body，把返回的 disposer 收集进 `disposables`；卸载时**按注册逆序**执行（[fiber.ts#L433-L448](../vendor/cordis/src/fiber.ts#L433)）。
- `ctx.on()` 注册监听器同样经由 `ctx.fiber.effect()`，卸载时自动 `unregister`（[events.ts#L263-L269](../vendor/cordis/src/events.ts#L263)、[events.ts#L294-L307](../vendor/cordis/src/events.ts#L294)）。

```mermaid
flowchart LR
  A[ctx.effect 执行 body] --> B[收集 disposer1]
  B --> C[收集 disposer2]
  C --> D[返回外部 disposer]
  D -->|卸载| E[逆序执行 disposer2 → disposer1]
```

### 4. 事件派发实现

五种派发模式集中在 [`events.ts`](../vendor/cordis/src/events.ts) 的 [`EventsService`](../vendor/cordis/src/events.ts#L130)：

| 模式 | 实现方法 | 关键行为 |
| --- | --- | --- |
| `parallel` | [events.ts#L191](../vendor/cordis/src/events.ts#L191) | `Promise.allSettled` 并发执行，有错抛 `AggregateError` |
| `emit` | [events.ts#L203](../vendor/cordis/src/events.ts#L203) | 同步顺序执行，忽略返回值 |
| `serial` | [events.ts#L214](../vendor/cordis/src/events.ts#L214) | 顺序 await，首个 bail 值返回 |
| `bail` | [events.ts#L228](../vendor/cordis/src/events.ts#L228) | 同步顺序执行，首个 bail 值返回 |
| `waterfall` | [events.ts#L246](../vendor/cordis/src/events.ts#L246) | 洋葱式组合：`next()` 委托，`next()` 调两次报错 |

waterfall 的核心是闭包链：每次 `next()` 弹出下一个 listener，最终调用内置 `inner`（[events.ts#L248-L260](../vendor/cordis/src/events.ts#L248)）。bail 的判定规则是“返回值不是 null/false/undefined 即停止”（[events.ts#L12-L14](../vendor/cordis/src/events.ts#L12)）。

### 5. 能力接缝源码：以 `ctx.shell` 为例

- Service Definition：`declare module '@deepseek-ai/cordis'` 向 `Context` 声明 `shell`，抽象类 [`ShellExecutor extends Service`](../packages/shell/shell/src/index.ts#L64) 即接口（[shell/index.ts#L30](../packages/shell/shell/src/index.ts#L30)）。
- 请求/规格分层：模型可见的 [`ShellExecRequest`](../packages/shell/shell/src/types.ts#L61) 由 `resolve()` 填默认值、封顶后得到实现用的 [`ShellExecSpec`](../packages/shell/shell/src/types.ts#L111)；这正是“显式 resolve，而非 `run()` 里藏 `??` 默认值”的接缝模板。
- Provider 实现如 `bash-local`、`pwsh-local`、`bash-sandbox` 等挂到同一 `ctx.shell`；Consumer 是 `tool-bash` / `tool-bash-persistent` 等模型工具。

```mermaid
flowchart LR
  SD[Service Definition<br/>ShellExecutor / ctx.shell] --> P1[bash-local Provider]
  SD --> P2[pwsh-local Provider]
  SD --> P3[bash-sandbox Provider]
  P1 --> C[tool-bash Consumer]
  P2 --> C
  P3 --> C
  C --> Model[模型可调工具]
```

### 6. 工具注册与执行流水线源码

- [`ToolsService extends Service`](../packages/core/tools/src/index.ts#L99) 通过 [`register()`](../packages/core/tools/src/index.ts#L128) 注册工具定义并返回 disposer；同名工具在同一 scope 重复注册会抛错（[index.ts#L69](../packages/core/tools/src/index.ts#L69)）。
- 工具事件契约在 [`tools/src/events.ts`](../packages/core/tools/src/events.ts) 声明：`tools/pre-execute`（waterfall）、`tools/execute`（waterfall）、`tools/post-execute`（waterfall）、`tools/result`（emit）、`tools/change`（emit），见 [events.ts#L18-L56](../packages/core/tools/src/events.ts#L18)。
- 具体流水线在 [`ToolExecutionPipeline`](../packages/core/tools/src/pipeline.ts#L62)：`tools/pre-execute` 分发点 [pipeline.ts#L200](../packages/core/tools/src/pipeline.ts#L200)、`tools/execute` 分发点 [pipeline.ts#L300](../packages/core/tools/src/pipeline.ts#L300)、`finalizeContent` [pipeline.ts#L393](../packages/core/tools/src/pipeline.ts#L393)、`tools/result` [pipeline.ts#L409](../packages/core/tools/src/pipeline.ts#L409)、审批 [pipeline.ts#L436](../packages/core/tools/src/pipeline.ts#L436)。
- 执行器契约 [`ToolExecutor`](../packages/core/tools/src/executor.ts#L64) 只负责“描述 schema + 执行整批 + 可选写日志”，不拥有流水线策略。

### 7. Agent 循环源码：状态机与事件落账

默认驱动是 [`ReactLoopAgent`](../packages/core/agent-loop/src/agent.ts#L112)。其 `Phase` 状态机只有三种状态：

```mermaid
stateDiagram-v2
  [*] --> idle
  idle --> maintenance: 后台维护
  maintenance --> idle
  idle --> running: 有排队输入被唤醒
  running --> running: 连续 step
  running --> idle: turn 结束
  maintenance --> running: 被唤醒
```

- 状态定义：[agent.ts#L53-L61](../packages/core/agent-loop/src/agent.ts#L53)；状态切换对外发 `agent/status`：[agent.ts#L160-L168](../packages/core/agent-loop/src/agent.ts#L160)。
- 持久事件直接 `session.append`：`turn/start` [agent.ts#L336](../packages/core/agent-loop/src/agent.ts#L336)、`step/start` [agent.ts#L361](../packages/core/agent-loop/src/agent.ts#L361)、`assistant/message` [agent.ts#L505](../packages/core/agent-loop/src/agent.ts#L505)、`turn/end` [agent.ts#L422](../packages/core/agent-loop/src/agent.ts#L422)。
- 请求前先走 `agent/request` waterfall 提出配置：[agent.ts#L631](../packages/core/agent-loop/src/agent.ts#L631)。

### 8. 会话日志与投影源码

- 日志格式版本：[`SESSION_FORMAT_VERSION = 4`](../packages/core/session/src/types.ts#L90)；事件类型表是 merge-extensible 的 [`SessionEventMap`](../packages/core/session/src/types.ts#L287)。
- 每个事件统一包装为 [`SessionEvent`](../packages/core/session/src/types.ts#L501)，带 `data`、序号与可选 `ignorable`（[types.ts#L519](../packages/core/session/src/types.ts#L519)）——这就是“不认识该事件类型的旧 reader 也可安全忽略”的机制。
- 投影接缝 [`ProjectionDefinition`](../packages/session/session-projection/src/index.ts#L81)：`init()` 给空日志初值（[index.ts#L92](../packages/session/session-projection/src/index.ts#L92)）、`apply()` 是纯 fold（[index.ts#L101](../packages/session/session-projection/src/index.ts#L101)）、`wire.view()` 生成客户端视图（[index.ts#L114](../packages/session/session-projection/src/index.ts#L114)）。框架负责订阅与水印缓存，域插件只写纯计算。

```mermaid
flowchart LR
  E1[committed event] --> F[ProjectionDefinition.apply]
  F --> S[state 全量值]
  S --> V[wire.view 客户端视图]
  E2[next event] --> F
```

---

## Agent Loop 详解


默认驱动是 [`ReactLoopAgent`](../packages/core/agent-loop/src/agent.ts#L112)：一个会话由 **turn → step** 两层边界驱动，输入来自可持久化的 inbox，每个请求都从会话日志重建。

### 状态机

```mermaid
stateDiagram-v2
  [*] --> idle
  idle --> maintenance: 后台维护（abort 可控）
  maintenance --> idle: 维护完成
  idle --> running: inbox 有输入被唤醒
  running --> running: 连续 step
  running --> idle: turn 自然结束 / 无 next-step
  running --> maintenance: 需要维护
```

状态定义与 `agent/status` 发布：[agent.ts#L53-L61](../packages/core/agent-loop/src/agent.ts#L53)、[agent.ts#L160-L168](../packages/core/agent-loop/src/agent.ts#L160)。

### 输入队列：inbox 投影

- 待处理输入分成 `next-turn` 与 `next-step` 两级，由 [`ReactLoopInbox`](../packages/core/agent-loop/src/inbox.ts#L73) 实现。
- 持久化靠投影 [`inboxProjectionDefinition`](../packages/core/agent-loop/src/inbox.ts#L26)：fold `agent/inbox/spliced` 事件重建队列，坏 splice 抛错（[inbox.ts#L30-L55](../packages/core/agent-loop/src/inbox.ts#L30)）。

### 一个 step 的完整事件流

```mermaid
sequenceDiagram
  participant I as Inbox
  participant L as ReactLoopAgent
  participant S as Session
  participant P as ctx.systemPrompt
  participant LLM as ctx.llm
  participant T as ToolsPipeline
  I->>L: 排队输入唤醒 driver
  L->>S: turn/start
  L->>S: step/start
  L->>P: system-prompt/assemble（waterfall）
  L->>LLM: agent/request → prepareCall → llm/stream
  LLM-->>L: StreamChunk*
  L->>S: assistant/message
  L->>T: executeToolBatch
  T-->>S: tool/call → tool/result
  L->>S: step/end
  L->>S: turn/end
```

持久事件落账点：`turn/start` [agent.ts#L336](../packages/core/agent-loop/src/agent.ts#L336)、`step/start` [agent.ts#L361](../packages/core/agent-loop/src/agent.ts#L361)、`assistant/message` [agent.ts#L505](../packages/core/agent-loop/src/agent.ts#L505)、`turn/end` [agent.ts#L422](../packages/core/agent-loop/src/agent.ts#L422)。模型配置先经 `agent/request` waterfall：[agent.ts#L631](../packages/core/agent-loop/src/agent.ts#L631)。

### 模型选择与系统提示

- 模型选择投影与系统提示投影封装在 [`runtime-context.ts`](../packages/core/agent-loop/src/runtime-context.ts)。
- [`SystemPromptProjection`](../packages/core/agent-loop/src/runtime-context.ts#L68) 决定渲染后的系统提示如何落到 surface node（append / 替换），而非每次重发。

---

## ACP（Agent Client Protocol）


ACP 是“仅自动化”的协议入口：程序通过 **JSON-RPC stdio** 驱动持久化 Harness agent，不承载任何面向人类的 UI。

### 定位与启动

- 包：[`packages/acp/acp/README.md`](../packages/acp/acp/README.md)；profile bundle：[`packages/bundle/acp-app/README.md`](../packages/bundle/acp-app/README.md)。
- 启动：`pnpm dsh --profile acp`。stdout 只用于 newline-delimited ACP JSON-RPC 帧。
- 典型客户端：`dsh-subagent-acp`（跨进程子代理）与测试运行器。

### 自动化工作流

```mermaid
sequenceDiagram
  participant C as ACP 客户端
  participant S as dsh --profile acp
  C->>S: initialize
  C->>S: session/new（绝对 cwd + 可选 MCP）
  C->>S: session/set_config_option（model / reasoning_effort）
  C->>S: session/prompt
  S-->>C: session/update（committed 语义更新流）
  C->>S: session/cancel / $/cancel_request
  C->>S: session/close
  C->>S: session/list / session/resume（跨进程恢复）
```

### 协议方法速览

| 调用 | 作用 |
| --- | --- |
| `initialize` | 声明 ACP v1 能力与 `session/*` 扩展 |
| `authenticate` | 立即成功，无需认证 |
| `session/new` | 创建持久 agent，校验 workspace 与 MCP |
| `session/list` | 按最新优先分页列出可恢复会话 |
| `session/resume` | 恢复会话，日志重建但不重放旧更新 |
| `session/close` | 安静取消、排空更新、销毁后代、刷持久化 |
| `session/set_config_option` | 更新 `model` / `reasoning_effort` |
| `session/prompt` | 一次一个 prompt，按序返回更新 |
| `session/cancel` | 取消在途 prompt 或自主工作 |

### 实现源码映射

| 文件 | 职责 |
| --- | --- |
| [`src/index.ts`](../packages/acp/acp/src/index.ts) | 插件入口：Config、连接装配、会话记录、准入/结算、拆除 |
| [`src/content.ts`](../packages/acp/acp/src/content.ts) | 图像准入与投影、路由复查、prompt 重建 |
| [`src/codec.ts`](../packages/acp/acp/src/codec.ts) | turn 结束原因到 ACP `stopReason` 的纯映射 |
| [`src/session.ts`](../packages/acp/acp/src/session.ts) | 会话模块：Agent 句柄、MCP 挂载、模型选择、更新链 |
| [`src/updates.ts`](../packages/acp/acp/src/updates.ts) | 语义更新序列化交付 |
| [`src/mcp.ts`](../packages/acp/acp/src/mcp.ts) | MCP server 挂载与信任模型 |

核心承诺：**只发标准语义更新**、**能力与配置状态真实可验证**、**结算前达到静默**（quiescence）。

---

---

## Hooks 钩子桥接

Hooks 让已有的 **Claude Code / Codex hooks** 在 `dsh` 中继续生效：把参考工具里的 hooks 规则映射到 Harness 的 `agent/*` / `tools/*` 事件上，不需要为同样逻辑写原生 Cordis 插件。

### 包与分工

- 共享规则：[`dsh-hook-protocol`](../packages/experimental/hook-protocol/README.md)：定义“hook 能做什么、运行时发生什么”，两个桥接共用。
- Claude Code 方言：[`dsh-hooks-claude-code`](../packages/experimental/hooks-claude-code/README.md)。
- Codex 方言：[`dsh-hooks-codex`](../packages/experimental/hooks-codex/README.md)。

用户不直接装 `hook-protocol`，而是挂其中一个桥接并指向已有的 `hooks.json`。

### 一个 hook 能做什么

- **拦截动作**：以 exit code 2 退出 = 阻止本次 prompt 或 tool call，stderr 作为原因展示给模型。
- **请求确认**：Claude Code hook 可在工具运行前请求确认；Codex 桥不暴露该选项。
- **附加上下文**：hook 返回的文本进入下一次请求，模型可见。
- **按时机触发**：配置用事件名或正则选择触发点；缺省/空/`'*'` 表示该类型全部事件。
- **失败不打断**：除 exit code 2 外的失败只记录日志，不阻止 agent。
- **请求停止**：`{"continue": false}` 会被记录，但当前没有 run 级效果（见各 README 的 Known Limitations）。

只有 `command` handler 会执行；`http` / `mcp_tool` / `prompt` / `agent` 会被跳过并告警。

### 处理流水线

```mermaid
flowchart LR
  CFG[hooks.json / config] --> SEL[selectHookGroups 选择匹配组]
  SEL --> RUN[经 dsh-shell 执行 command]
  RUN --> DEC[decode 解码 exit code / stdout]
  DEC --> MERGE[合并多个 hook 结果<br/>deny > ask > allow]
  MERGE --> REC[记录 hook/invoked 与 hook/result]
  MERGE --> EV[映射到 agent/* / tools/* 决策]
  EV --> MODEL[block 原因 / 附加上下文 进入模型]
```

每一步都降级为“包含性结果”，不会抛错打断当前 turn：非法正则是“不匹配”，执行器拒绝变成无 exit code 的 `HookOutput`，exit 2 用 stderr 作为 block 原因，其余失败保持非阻塞。

### Codex 支持面

Codex 当前 10 个 hook 事件中，本桥支持 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`Stop`；`PermissionRequest`、`PreCompact`、`PostCompact`、`SubagentStart`、`SubagentStop` 暂不支持，相关配置解析时被丢弃。详细限制见 [`hooks-codex` 的 Known Limitations](../packages/experimental/hooks-codex/README.md#known-limitations-and-deferred-work)。



## 开发实践步骤


### 阶段 0：把环境跑起来

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
corepack enable          # 启用 pnpm 11.28.5
pnpm install             # 安装依赖并配置 Lefthook hooks
pnpm run typecheck       # 首次克隆后跑一次
pnpm run build
pnpm dsh web             # 起 Web UI（默认 http://127.0.0.1:3080）
```

前提：Node.js `>= 24.9` 且支持 TypeScript type stripping；不要用 `NODE_OPTIONS=--no-experimental-strip-types`。详见 [`docs/development.md`](../docs/development.md)。

### 阶段 1：先懂 Cordis

- 概念速查：读 [`docs/cordis-primer.md`](../docs/cordis-primer.md)。
- 动手教程：从 [`docs/cordis-tutorial/index.md`](../docs/cordis-tutorial/index.md) 开始，逐章完成 7 个可运行示例：

```text
1. 第一个插件        —— 插件是一个函数，loader 挂载它
2. 生命周期与副作用  —— 注册随插件卸载被撤销
3. 服务             —— 在 ctx 上暴露能力，用 inject 依赖
4. 事件             —— 类型化事件、广播、waterfall 短路
5. 配置             —— 从 cordis.yml 校验配置，坏输入响亮失败
6. 组合与 HMR       —— 配置作为插件树、热重载、诊断未加载
7. 进入 harness     —— 对真实服务注册一个模型可调工具
```

教程目录就是本仓库的 `tmp/cordis-tutorial/`（gitignored），每个章节运行：

```sh
cd tmp/cordis-tutorial
node --import tsx ../../vendor/cordis/bin.js
```

### 阶段 2：读懂整体架构

- 必读 [`docs/architecture.md`](../docs/architecture.md)：profile/bundle、能力接缝、Agent 循环、会话日志、扩展点。
- 看图：`docs/agent-lifecycle.md`、`docs/tool-execution-pipeline.md`、`docs/capability-seams.md`。
- 看包图：`docs/module-graph.md`、`docs/graph-atlas.md`。

### 阶段 3：掌握包布局

- 总地图读 [`packages/README.md`](../packages/README.md)。
- 重点先看 `packages/core/`：`agent`、`agent-loop`、`session`、`tools`、`system-prompt`。
- 看一个接缝全家桶，推荐 `shell`：Service Definition → 本地 provider → 模型工具 Consumer。

### 阶段 4：真实运行与最小改动

```sh
pnpm dsh --profile headless "summarize this workspace"   # 需要 DEEPSEEK_API_KEY
pnpm run dev:web                                          # Web 热重载开发
```

最小改动练习（都带分步指南）：

- [`docs/cookbook/adding-a-tool.md`](../docs/cookbook/adding-a-tool.md)：加一个模型工具。
- [`docs/cookbook/adding-an-llm-adapter.md`](../docs/cookbook/adding-an-llm-adapter.md)：加 LLM adapter。
- [`docs/cookbook/adding-a-package.md`](../docs/cookbook/adding-a-package.md)：加 npm 包。
- [`docs/cookbook/adding-a-settings-card.md`](../docs/cookbook/adding-a-settings-card.md)：加设置页。

### 阶段 5：按仓库规范贡献

- 改 `packages/` 前必读 [`docs/architecture.md`](../docs/architecture.md)；文档改动遵循 [`docs/AGENTS.md`](../docs/AGENTS.md)。
- 每次提交前跑“最小覆盖集”，不要默认全量测试：
  - 行为改动 → 聚焦单测 + 快照。
  - 文档 → `pnpm run doc-sync`。
  - 发布路径/构建产物 → `pnpm run build` + `pnpm run hygiene`。
- 非平凡改动必须在同一 PR 写 Agent Note，见 [`.agents/notes/README.md`](../.agents/notes/README.md)。

### 学习路线图

```mermaid
flowchart TD
  A[0. 环境跑通<br/>install / typecheck / build] --> B[1. Cordis 概念 + 教程 7 章]
  B --> C[2. 总架构 architecture.md]
  C --> D[3. 包布局 packages/README.md]
  D --> E[4. 真实运行 headless / web]
  E --> F[5. 最小改动：加工具 / adapter / 包]
  F --> G[6. 测试与门禁 + Agent Note]
```

---

## 文件索引


常用入口文件与目录（相对本手册的路径，点击可跳转）：

- 项目总览：[`README.md`](../README.md)
- 架构：[`docs/architecture.md`](../docs/architecture.md)
- Cordis 概念：[`docs/cordis-primer.md`](../docs/cordis-primer.md)
- Cordis 教程：[`docs/cordis-tutorial/index.md`](../docs/cordis-tutorial/index.md)
- 开发指南：[`docs/development.md`](../docs/development.md)
- 测试策略：[`docs/testing.md`](../docs/testing.md)
- 能力接缝：[`docs/capability-seams.md`](../docs/capability-seams.md)
- Agent 生命周期：[`docs/agent-lifecycle.md`](../docs/agent-lifecycle.md)
- 工具流水线：[`docs/tool-execution-pipeline.md`](../docs/tool-execution-pipeline.md)
- 包间依赖图：[`docs/module-graph.md`](../docs/module-graph.md)
- 文档图索引：[`docs/graph-atlas.md`](../docs/graph-atlas.md)
- 包分组：[`packages/README.md`](../packages/README.md)
- 核心包：[`packages/core/README.md`](../packages/core/README.md)
- 扩展指南目录：[`docs/cookbook/`](../docs/cookbook/)
- 子系统参考：[`docs/subsystems/`](../docs/subsystems/)
- Agent Notes：[`.agents/notes/README.md`](../.agents/notes/README.md)
- 仓库门禁脚本：[`scripts/`](../scripts/)

---

## 命令速查


| 命令 | 用途 |
| --- | --- |
| `pnpm run typecheck` | TypeScript 全量类型检查 |
| `pnpm run lint` | oxlint |
| `pnpm run test` | vitest 单元测试 |
| `pnpm run test:coverage` | CI 覆盖率门禁 |
| `pnpm run test:snapshot` | keyless 快照测试 |
| `pnpm run build` | tsc + tsdown 产物构建 |
| `pnpm run hygiene` | knip + publint + workspace 约束 |
| `pnpm run doc-sync` | 文档全部门禁 |
| `pnpm run duplication` | 跨文件 TS 克隆检测 |
| `pnpm dsh web` | 从源码启动 Web UI |
| `pnpm dsh --profile headless "task"` | 一次性 Headless 任务 |
