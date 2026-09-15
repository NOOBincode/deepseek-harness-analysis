# DeepSeek Harness 源码分析

> 对 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)(DSH)——一个"一切皆插件"的 Cordis agent harness——的源码分析。
> 分析基线:upstream commit [`dbbaa4a3`](https://github.com/deepseek-ai/deepseek-harness/commit/dbbaa4a37fb9098aba814c97d2956f7b2f105f46)

14 章正文 + 9 个模块的函数级深度展开。

---

## 这是什么

不是导读、也不是 API 手册,而是一份贴着源码写的分析:把 DSH 从"程序怎么启动"一路拆到"一次工具调用内部发生了什么",关键处直接给出真实代码、控制流与示意图。

它回答三类问题:

- **架构类**:为什么这个 harness 没有"内核"?插件系统怎么保证可逆与热替换?会话状态为什么只有一条日志?
- **机制类**:MCP 工具怎么被发现并进入模型请求?沙箱如何做到逐调用携带策略?压缩与 resume 各自改动了什么?
- **实现类**:某个函数的每个失败分支、每张状态表、每处竞态防护分别在哪里、为什么这么写?

## 先看结论

DSH 的四个贯穿性判据(全套文档反复用到):

| 不变式 | 含义 |
|---|---|
| **模型可见 ⟺ 已落日志** | 任何进入模型请求的内容必须能从会话日志重建;新增模型可见输入 = 新增 Session 事件 |
| **注册即 effect** | 每个贡献经 `ctx.effect()` 落地并返回精确 disposer,插件卸载(HMR)时自动回收 |
| **能力缝三角色** | 可替换能力恒为 Service Definition / Provider / Consumer,缺一即未完成 |
| **fail-closed / fail-loud** | 自包含错配在加载期抛错;策略不可判定时拒绝执行而非放行 |

一句话心智模型:**DSH 用"插件 + 能力缝 + 事件日志"替换了传统 agent 框架里所有"核心模块"**——没有上下文管理器、没有工具总线、没有 MCP 子系统、没有多智能体框架,它们分别是 `systemPrompt` 注册表、`ToolRuntime` 注册表、一个桥接插件、一条 subagent 能力缝。

## 一图看懂

从用户输入到工具结果回流的**端到端生命周期**(每一环都标注了深挖章节):

![DeepSeek Harness 端到端请求生命周期](./analysis/assets/diagrams/14-final-summary-27.svg)

<details><summary>这张图怎么读</summary>

- 上半段是"一次模型请求怎么被组装出来":profile 组装 → Session 打开/恢复 → 提示组装 → surface 折叠出消息历史 → 请求;
- 下半段是"一次工具调用怎么被执行与回流":调度分组 → 审批/守卫管道 → 沙箱或 MCP 或子 Agent → 结果落日志 → 下一步;
- 图中所有分叉都有对应章节,见[内容地图](#内容地图)。

</details>

## 内容地图

### 14 章正文(结论层)

| 部分 | 章节 |
|---|---|
| 一 · 总体架构 | 第一章 软件架构与程序入口 |
| 二 · 安全分析 | 第二章 安全分析 |
| 三 · 核心机制 | 第三章 Session 与 Memory · 第四章 Skills · 第五章 Tool Call · 第六章 MCP · 第七章 Sandbox · 第八章 Context · 第九章 Prompt · 第十章 Multi-Agent · 第十一章 持久化与 Resume |
| 四 · 架构亮点 | 第十二章 程序架构及亮点(能力缝、LLM 抽象、Typert 类型图、质量工程) |
| 五 · 扩展分析 | 第十三章 扩展生态(Hooks / ACP / Webhook / Web+Desktop / 双 SDK) |
| 总结 | 第十四章 总结结论(不变式、端到端生命周期、横向对比、设计代价) |

### 9 个模块深度展开(函数级)

| 模块 | 篇数 | 覆盖 |
|---|---|---|
| [`harness/`](./analysis/harness/README.md) | 8 | 主循环骨架(turn/step 状态机)、输入与步边界(inbox/wake)、一次模型请求全过程、助手流与落库、取消与处置、不变量与守卫、生命周期事件 |
| [`plugin-system/`](./analysis/plugin-system/README.md) | 6 | Cordis 运行时内部(代理/状态机/effect 回收/事件分发)、Loader 与组合(事务回滚、HMR)、能力缝解剖、扩展点目录、插件编写指南 |
| [`memory/`](./analysis/memory/README.md) | 8 | 事件日志提交路径、surface 与可见性、派生投影与水印、压缩选区与事务、落盘与恢复、状态型记忆、不变量与失败模式 |
| [`context/`](./analysis/context/README.md) | 7 | 六类来源通道、每步组装五段、运行时投影三态、token 计量与压力折算、压缩与溢写优先级、上下文类插件 |
| [`tool-call/`](./analysis/tool-call/README.md) | 7 | 注册表与可见性、执行管道逐段、调度与并发、取消与超时、PTC 模式、结果展示层 |
| [`mcp/`](./analysis/mcp/README.md) | 7 | 发现与两阶段同步、命名算法实测、执行与结果映射、连接监管、传输与安全、失败模式清单 |
| [`skills/`](./analysis/skills/README.md) | 6 | 格式与六档发现根、provider 注册表裁决、目录与按需加载、watcher 失效、作用域与出货 |
| [`sandbox/`](./analysis/sandbox/README.md) | 7 | 缝隙与策略、平台后端(bwrap/Landlock/Seatbelt/Windows ACL)、升级审批、消费方、E2B、边界清单 |
| [`multi-agent/`](./analysis/multi-agent/README.md) | 8 | Agent 生命周期、缝隙与 provider、子 Agent 组装、续存与管控、workflow 引擎、jobs、preset |

## 怎么读

| 你的目的 | 推荐路径 |
|---|---|
| **只想拿结论**(30 分钟) | [`analysis/14-final-summary.md`](./analysis/14-final-summary.md) → 需要细节时按阅读地图跳转 |
| **理解整体架构** | [`analysis/01-architecture-overview.md`](./analysis/01-architecture-overview.md) → [`analysis/12-architecture-highlights.md`](./analysis/12-architecture-highlights.md) → [`analysis/plugin-system/`](./analysis/plugin-system/README.md) |
| **看懂主循环怎么转** | [`analysis/harness/`](./analysis/harness/README.md) |
| **搞清会话状态与恢复** | [`analysis/03-session-memory.md`](./analysis/03-session-memory.md) / [`analysis/11-persistence.md`](./analysis/11-persistence.md) → [`analysis/memory/`](./analysis/memory/README.md) |
| **搞清上下文与压缩** | [`analysis/08-context.md`](./analysis/08-context.md) → [`analysis/context/`](./analysis/context/README.md) |
| **加一个能力**(工具/skill/MCP) | [`analysis/05-tool-call.md`](./analysis/05-tool-call.md) → [`analysis/tool-call/`](./analysis/tool-call/README.md) / [`analysis/skills/`](./analysis/skills/README.md) / [`analysis/mcp/`](./analysis/mcp/README.md) |
| **查具体实现** | 直接进对应模块目录,或读该章末尾的"关键文件索引表"按 `文件:行号` 回源码 |

完整导航(一图总览 + 分部分目录 + 阅读地图)见 **[`analysis/README.md`](./analysis/README.md)**。

## 怎么读图

- 图只画主干:节点是中文短语,不放函数名与行号——先顺着箭头看懂"发生了什么"。
- 要知道"具体调用了哪个函数",看每张图下面的 `阶段 | 做了什么 | 关键调用(文件:行)` 表。
- 要逐行核对,展开再下面的折叠块:原始调用树、时序轨迹与 Mermaid 源码都在那里。

## 仓库结构

```text
.
├── README.md                 # 本文件:介绍与引导
└── analysis/
    ├── README.md             # 文档集总索引(一图总览 + 分章目录 + 阅读地图)
    ├── 01-architecture-overview.md
    ├── ...
    ├── 14-final-summary.md
    ├── assets/diagrams/      # 渲染后的 SVG 图
    ├── harness/              # 8 篇:主循环、请求、流、处置、不变量、事件
    ├── plugin-system/        # 6 篇:插件设计函数级展开
    ├── memory/               # 8 篇:事件日志、surface、投影、压缩、落盘恢复、状态记忆
    ├── context/              # 7 篇:来源、组装、运行时投影、计量、压缩溢写、上下文插件
    ├── tool-call/            # 7 篇
    ├── mcp/                  # 7 篇
    ├── skills/               # 6 篇
    ├── sandbox/              # 7 篇
    └── multi-agent/          # 8 篇
```

## 一起完善

发现事实错误(行号漂移、符号名不符、结论与源码相反)欢迎提 Issue 或 PR,并附上 `文件:行号` 与 upstream commit;若 upstream 版本推进导致引用失效,请在 PR 中同步更新对应章节的基线说明。

## 声明

> 本仓库内容仅供技术学习与研究使用。DeepSeek Harness 的所有权利归其原权利人所有,本仓库与 upstream 项目无隶属关系。
