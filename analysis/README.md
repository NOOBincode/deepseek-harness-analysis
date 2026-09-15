# DeepSeek Harness 源码分析

> **先读这一页。** 本目录是 DeepSeek Harness(DSH)的源码分析:14 章正文 + 6 个模块的函数级深度展开。若只想拿结论,直接看 **[第十四章:总结结论](./14-final-summary.md)**;要钻实现细节,进[模块深度展开](#模块深度展开函数级);按关注点跳读见[阅读地图](#阅读地图)。

## 分析对象

| 项 | 值 |
|---|---|
| 仓库 | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) |
| commit | `dbbaa4a37`(文中 `文件:行号` 均以此版本为准) |
| 本地源码 | `../deepseek-harness/` |
| 形态 | pnpm monorepo:全插件 Cordis agent harness,`packages/` 约 50 个包,`apps/` 四端(cli/web/desktop/desktop-host),另有 Python SDK |

## 总述

DSH 的系统特征一句话:**"一切皆插件"**——从工具注册表、系统提示、LLM 提供方到 MCP 桥接,能力全部是 Cordis 插件,经 `ctx.effect()` 注册、由 `cordis.yml` 层叠组合、支持 HMR 热替换;agent 主循环只做"组装提示 → 流式请求 → 调度工具 → 落会话日志"这一件事,且状态真源只有一条 append-only 会话事件日志。

主要探讨的系统设计问题:

1. 软件架构设计与程序启动路径(monorepo 布局、Cordis 插件系统、profile 启动链)。
2. 安全分析:信息与凭据的使用边界、沙箱与审批防线、沙箱之外的风险点。
3. Session 事件溯源与 Memory 机制(日志为唯一权威、投影派生、压缩)。
4. 能力扩充机制(Skills、Tool Call、MCP)的技术实现与运行方式。
5. 隔离机制:Sandbox 策略模型、跨平台实现与升级审批。
6. 上下文与 Prompt 管理(每步组装、token 预算、增量重投)。
7. Multi-Agent 派生(subagent / workflow / jobs / preset)。
8. 持久化与恢复(物理格式、世代规则、版本迁移、resume 链)。
9. 程序架构亮点(能力缝三角色、Typert 类型图、质量工程体系)。
10. 扩展生态(Hooks、ACP、Webhook、Web/Desktop、双 SDK)。

> **图怎么读**:正文里的 `文件:行号` 都是可点击链接,直接跳到 upstream 对应行;图里只有中文短语,不放函数名与行号——先顺着箭头理解"发生了什么";要知道"具体调用了哪个函数",看每张图下面的 `阶段 | 做了什么 | 关键调用(文件:行)` 表;再往下的折叠块里是图的 Mermaid 源码与原始调用树。

## 一图总览

**一句话读法**:这张图从左到右、从上到下就是一次运行的完整分层——最上面是"用户怎么进来",中间是"启动时把哪些插件装进一棵树",再往下是"具备哪些核心服务",然后是"挂在核心服务上的各类能力插件",最后是"主循环每次干活时经过哪几步、状态落到哪里"。

**端到端生命周期**(每环标注深入章节):

![端到端请求生命周期](./assets/diagrams/14-final-summary-27.svg)

**分层速览**:入口 → 启动组装 → 核心服务 → 能力缝插件 → 主循环 → 持久化:

<details><summary>文字版分层图</summary>

```text
入口          dsh CLI / Web / Desktop / ACP / SDK
                    ↓
启动组装      profile 解析 → cordis.yml 层叠 → Loader 按依赖并发激活(失败回滚)
                    ↓
核心服务      tools / system-prompt / agent / agent-loop / session / scope
                    ↓
能力缝插件    llm  shell  skill  mcp  sandbox  fs  subagent  hooks  webhook …
              (每条能力 = Service Definition / Provider / Consumer 三角色)
                    ↓
主循环        ReactLoopAgent:组装提示 → 流式请求 → 调度工具 → 落会话日志
                    ↓
持久化        session-persistence:世代文件落盘 / resume 回读
```

</details>

## 分章目录

### 第一部分:总体架构

- [第一章:软件架构与程序入口](./01-architecture-overview.md)——monorepo 布局、Cordis 插件系统、`dsh --profile` 启动链、三端形态、模块依赖主干

### 第二部分:安全分析

- [第二章:安全分析](./02-security-analysis.md)——信息与凭据边界、沙箱之外的受信面、审批/守卫/清洗三层防线

### 第三部分:核心机制

- [第三章:Session 与 Memory 机制](./03-session-memory.md)——事件溯源、surface/projection、resume/repair、compaction、分层存储
- [第四章:Skills 的技术实现与运行方式](./04-skills.md)——SKILL.md 契约、provider 注册表、catalog 与按需加载
- [第五章:Tool Call 机制实现细节](./05-tool-call.md)——ToolDefinition 契约、分层注册表、四段式管道、并发调度、PTC 模式
- [第六章:MCP 技术架构与原理](./06-mcp.md)——发现(配置 + `tools/list` + `list_changed`)、命名契约、两阶段同步、主循环中的使用、重连监管
- [第七章:Sandbox 技术实现与运行机制](./07-sandbox.md)——能力缝三角色、策略逐调用携带、POSIX 与 Windows ACL、E2B、升级审批
- [第八章:Context 上下文管理实现细节](./08-context.md)——四层来源、preStep 组装链、token 计量、compaction 触发与落日志
- [第九章:Prompt 管理机制与实现细节](./09-prompt.md)——提示即注册表服务、assemble 瀑布、工具排序、重投时机
- [第十章:Multi-Agent 机制与实现细节](./10-multi-agent.md)——真 Agent 派生、subagent/workflow/jobs、preset 组合
- [第十一章:Session Storage / Transcript / Resume 持久化机制](./11-persistence.md)——JSONL 世代文件、checkpoint、格式迁移、冷读与恢复

### 第四部分:程序架构及亮点

- [第十二章:程序架构及亮点](./12-architecture-highlights.md)——effect 即注册、能力缝模式、LLM 抽象、Typert 类型图、质量工程

### 第五部分:扩展分析

- [第十三章:扩展生态——Hooks、ACP、Web/Desktop 与 SDK](./13-extensions-ecosystem.md)——五条进程边界的协议细节与内核落点

### 第六部分:总结

- [第十四章:总结结论](./14-final-summary.md)——四条不变式、端到端生命周期、横向对比、设计代价、阅读地图

## 模块深度展开(函数级)

九个模块在章节之外另有子目录,做函数级走查:调用栈逐段拆解、状态机与变量表、失败分支、边界用例与测试证据。章节给结论,模块给实现细节。

| 模块 | 目录 | 篇目 | 对应章节 |
|---|---|---|---|
| 运行时内核 | [`harness/`](./harness/README.md) | 8 篇:主循环骨架、输入与步边界、一次模型请求、助手流、取消与处置、不变量与守卫、生命周期事件 | 第一/十四章 |
| 插件设计 | [`plugin-system/`](./plugin-system/README.md) | 6 篇:Cordis 运行时内部、Loader 与组合、能力缝解剖、扩展点目录、插件编写指南 | 第一/十二章 |
| Memory | [`memory/`](./memory/README.md) | 8 篇:事件日志、surface 与可见性、派生投影、压缩、落盘与恢复、状态型记忆、不变量与失败模式 | 第三/十一章 |
| Context | [`context/`](./context/README.md) | 7 篇:来源分层、每步组装、运行时投影、token 计量、压缩与溢写、上下文插件 | 第八章 |
| Tool Call | [`tool-call/`](./tool-call/README.md) | 7 篇:注册表与可见性、执行管道、调度与并发、取消与超时、PTC 模式、展示层 | 第五章 |
| MCP | [`mcp/`](./mcp/README.md) | 7 篇:发现与同步、命名算法、执行与结果映射、连接监管、传输与安全、测试与失败模式 | 第六章 |
| Skills | [`skills/`](./skills/README.md) | 6 篇:格式与发现、provider 注册表、目录与加载、watcher 失效、作用域与组合 | 第四章 |
| Sandbox | [`sandbox/`](./sandbox/README.md) | 7 篇:缝隙与策略、平台后端、升级审批、消费方、E2B、边界与失败模式 | 第七/二章 |
| Multi-Agent | [`multi-agent/`](./multi-agent/README.md) | 8 篇:Agent 生命周期、缝隙与 provider、子 Agent 组装、续存管控、workflow、jobs、preset | 第十章 |

各模块目录内均有 [`README.md`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/README.md) 作为模块索引(含函数级调用栈图与篇目导读)。

## 阅读地图

| 你想了解 | 直接读 |
|---|---|
| 只看结论 / 全仓心智模型 | [第十四章](./14-final-summary.md) |
| 程序怎么启动、`dsh` 命令背后发生了什么 | [第一章](./01-architecture-overview.md)、[`plugin-system/02`](./plugin-system/02-loader-and-composition.md) |
| 插件怎么写、effect 与 HMR 怎么运作 | [`plugin-system/`](./plugin-system/README.md) |
| 安全边界在哪、哪些代码在沙箱之外 | [第二章](./02-security-analysis.md)、[`sandbox/03`](./sandbox/03-escalation-and-approval.md) |
| 会话状态怎么存、为什么"日志即真源" | [第三章](./03-session-memory.md)、[第十一章](./11-persistence.md) |
| 怎么加一个能力(工具 / skill / MCP) | [`tool-call/`](./tool-call/README.md)、[`skills/`](./skills/README.md)、[`mcp/`](./mcp/README.md) |
| 命令与文件操作的隔离怎么做的 | [第七章](./07-sandbox.md)、[`sandbox/`](./sandbox/README.md) |
| 上下文窗口爆了会怎样 | [第八章](./08-context.md) |
| 系统提示为什么不是一段字符串 | [第九章](./09-prompt.md) |
| 子 agent / workflow / 后台任务 | [第十章](./10-multi-agent.md)、[`multi-agent/`](./multi-agent/README.md) |
| 这个仓库的工程方法论 | [第十二章](./12-architecture-highlights.md) |
| 接 SDK、接 IDE、接外部事件 | [第十三章](./13-extensions-ecosystem.md) |

---

## 声明

> **本项目仅供学术研究与技术学习使用。**
>
> DeepSeek Harness 的所有权利归其原权利人所有;文中任何错漏以仓库源码与官方文档为准。
