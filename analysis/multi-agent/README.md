# Multi-Agent 模块 · 深度展开文档集

> 分析对象:[innokria/deepseek-harness](https://github.com/innokria/deepseek-harness) @ `dbbaa4a37`

---

## 一、按卡点索引

| 卡点 | 去哪一篇 |
|---|---|
| `ctx.agents.create()` 到底在哪几行把 session / agent 变成"活"的?谁有权拆掉它? | [01](./01-agent-registry-and-lifecycle.md) |
| `withInitiator` 是授权吗?为什么到处都是显式 `parent` 字段? | [01](./01-agent-registry-and-lifecycle.md) |
| `ctx.subagents.start()` 在调 provider 之前做了几次校验?provider 抛错时调用者手上有什么? | [02](./02-subagent-seam-and-providers.md) |
| spawn 和 fork 只差一个 `seed`,那 `prepareContinuable` 为什么必须两个都实现? | [02](./02-subagent-seam-and-providers.md) |
| 子 agent 的"世界"(工具面、提示段、persona、沙箱策略)是按什么顺序装起来的? | [03](./03-child-agent-composition.md) |
| 为什么父 agent 自己注册的 MCP 工具,**不会**被它的子 agent 继承? | [03](./03-child-agent-composition.md) |
| `send_message` 给一个已经不在内存里的子 agent,会发生什么? | [04](./04-continuation-and-control.md) |
| `interrupt_agent` 到底停掉了什么、没停掉什么? | [04](./04-continuation-and-control.md) |
| workflow 的 `agent()` 失败为什么有时整脚本炸、有时只是那项变 `null`? | [05](./05-workflow-worker-thread.md) |
| worker 线程 + `vm` 是安全边界吗?能挡住什么? | [05](./05-workflow-worker-thread.md) |
| 后台 job 完成的通知为什么会"自己"开一个 turn?会不会无限自激? | [06](./06-jobs-and-notifications.md) |
| `job_kill` 返回 `requested` 之后,进程保证做了什么? | [06](./06-jobs-and-notifications.md) |
| 编辑 preset 的 `agent.cordis.yml`,对正在跑的会话和已经 spawn 的子 agent 各有什么影响? | [07](./07-preset-composition.md) |

读法建议:**先读第十章建立心智模型,再用本目录下钻到你要改/要调试的那个函数。**

---

## 二、模块索引

| 篇 | 主题 | 主要源码 |
|---|---|---|
| [01](./01-agent-registry-and-lifecycle.md) | Agent 注册表与生命周期 | `packages/core/agent/src/index.ts`、`runtime-types.ts`、`packages/core/agent-loop/src/index.ts` |
| [02](./02-subagent-seam-and-providers.md) | 能力缝与 provider | `packages/subagent/subagent/src/{index,types}.ts`、`subagent-in-process-driver/src/index.ts`、`subagent-spawn-in-process/src/index.ts`、`subagent-fork-in-process/src/index.ts` |
| [03](./03-child-agent-composition.md) | 子 Agent 的"世界"组装 | `packages/subagent/subagent/src/child-agent.ts`、`depth.ts`、`descriptor.ts` |
| [04](./04-continuation-and-control.md) | 续存与管控 | `packages/subagent/subagent/src/{continuation,continuation-activation,control,list-children,assistant-output}.ts`、`tool-subagent-control/src/{index,list-agents}.ts` |
| [05](./05-workflow-worker-thread.md) | workflow 引擎 | `packages/workflow/workflow-worker-thread/src/{index,runtime,host,realm,protocol}.ts`、`packages/workflow/workflow/src/index.ts` |
| [06](./06-jobs-and-notifications.md) | jobs 与完成通知 | `packages/jobs/jobs/src/index.ts`、`jobs-local/src/index.ts`、`tool-jobs/src/index.ts` |
| [07](./07-preset-composition.md) | preset 组合 | `packages/preset/agent-presets/src/{index,mount,preset}.ts`、`presets/standard/agent.cordis.yml` |

---

## 三、父 Agent → 派生子 Agent → 结果回流:函数级调用栈

本节把"父 Agent 派生子 Agent"这条链路按时间顺序摊开。父 Agent(正在跑模型循环的那个 agent)发一次 `subagent` 工具调用,工具层先确认调用者身份,再把请求交给注册表里按名字选出的 provider——也就是真正负责造 Agent 的那个后端。provider 通过 `ctx.agents.create()` 造出一个带独立 Session(会话日志)与独立 scope(作用域链,决定它能看见哪些工具)的子 Agent,送第一条 prompt,等它闲下来,再把它最后一条 assistant 输出与本轮停因包成一次委派的结果值。结果回流只有两条路:前台直接变成工具结果,后台变成一条作业完成通知;两条路线共享的,只有开头"造 Agent"这一步。

![流程图：README](../assets/diagrams/multi-agent__README-49.svg)

<details><summary>Mermaid 源码</summary>

```mermaid
flowchart LR
  A["父 Agent 发出委派"]
  B["工具层确认调用者身份"]
  C["挑一个 provider 后端"]
  D["校验它声明过的能力"]
  E["创建真子 Agent"]
  F["子 Agent 在自己的 Session 里跑"]
  G["读回输出与停因"]
  H["前台:直接当成工具结果"]
  I["后台:包成一条作业"]
  J["续存:子 Agent 常驻,之后还能再发消息"]
  K["作业完成发一条通知"]
  L["通知按状态注入或新开 turn"]
  M["停因不是 completed 就转成错误结果"]
  N["释放子 Agent"]

  A --> B --> C --> D --> E --> F --> G
  G --> H
  G --> I
  G --> J
  I --> K --> L
  G --> M
  J --> N
```

</details>

| 阶段 | 做了什么 | 关键调用(文件:行) |
|---|---|---|
| 1 确认调用者 | 工具层要求 `exec.agent` 存在,缺失就直接抛错,不去猜调用者是谁 | `tool-subagent/src/index.ts:472-476` |
| 2 选路线 | 按 `run_in_background` 与 continuable 配置解析出前台、one-shot 后台或续存三条路线 | `tool-subagent/src/index.ts:287-305` |
| 3 能力缝校验 | 取出 provider 后逐项核对能力位,再把请求参数与父 Agent 组装成 resolved request | `subagent/subagent/src/index.ts:556-586` |
| 4 深度与策略 | 子深度等于父深度加一,超过上限直接抛错;委派策略必须在第一个 await 之前抓到 | `subagent-in-process-driver/src/index.ts:111-120` |
| 5 未发布窗口 | setup 里依次完成策略落日志、join 父 preset、persona 与工具掩码、描述符挂载 | `subagent-in-process-driver/src/index.ts:122-132` |
| 6 创建 | 经工厂建会话与驱动,setup 结束后提交并发布 | `agent-loop/src/index.ts:826-835` |
| 7 发布五步 | 会话入表 → Agent 入表 → 广播会话 → 广播 Agent → 发 session-start | `agent-loop/src/index.ts:662-677` |
| 8 驱动一轮 | 给子 Agent 送第一条 prompt,然后等它闲下来 | `subagent-in-process-driver/src/index.ts:178-182` |
| 9 读结果 | 取最后一条非空 assistant 消息作为输出,并映射出本轮停因 | `subagent-in-process-driver/src/index.ts:212-238` |
| 10 前台回流 | 停因不是 completed 就抛错,变成 isError 工具结果,但保留部分输出 | `tool-subagent/src/index.ts:207-237` |
| 11 后台回流 | one-shot 后台委派被包成一条作业,启动异常折成 killed 或 failed | `tool-subagent/src/index.ts:544-560`、`:143-153` |
| 12 完成通知 | owner 空闲且唤醒预算没用完就新开一个 turn,否则注入上下文 | `tool-jobs/src/index.ts:278-299` |
| 13 续存路线 | 子 Agent 的 inbox(它收消息的队列)接受初始 prompt 就立即返回,只回一个 subagentId,不等它跑完 | `subagent/subagent/src/continuation.ts:102-190` |
| 14 释放 | 结果失败优先于释放失败,两者都失败才合成一个 AggregateError | `tool-subagent/src/index.ts:207-237` |

<details><summary>原图:函数级调用栈</summary>

![时序图：README](../assets/diagrams/multi-agent__README-47.svg)

<details><summary>Mermaid 源码</summary>

```mermaid
sequenceDiagram
    autonumber
    participant P as 父 Agent(ReactLoopAgent)
    participant T as tool-subagent.execute
    participant S as SubagentRuntime.start
    participant D as startInProcessRun
    participant R as AgentRegistry
    participant F as AgentLoopFactory
    participant C as Registry(子 Agent)
    participant U as SubagentRun.result

    P->>T: tool-call subagent{description,prompt,...}
    Note over T: exec.agent 缺失即抛<br/>tool-subagent/src/index.ts:472-476
    T->>T: resolveDelegationRun → runInBackgroun<br/>tool-subagent/src/index.ts:287-305
    T->>S: ctx.subagents.start(provider, request)<br/>tool-subagent/src/index.ts:563
    Note over S: expectProvider → assertCapabilities(641-657)<br/>assertSubagentMaxDepth → assertObjectJsonSchema<br/>snapshotSubagentDescriptor(561-565)
    S->>D: provider.start(resolved)<br/>subagent/src/index.ts:567
    Note over D: resolveChildDepth(111)<br/>captureDelegatedPolicyOverrides(119)<br/>⚠ 必须在第一个 await 之前
    D->>R: parent.ctx.agents.create({...})<br/>driver/src/index.ts:134-143
    R->>F: Reflect.apply(target.createAgent, receiver, [ownerCtx, options])<br/>agent/src/index.ts:397
    Note over F: createStoredSession(730-738)<br/>prepare → ReactLoopAgent(624)
    F->>F: await setup(prepared.agent.ctx, prepared.agent)<br/>agent-loop/src/index.ts:826
    Note over F: setup 内容见第 03 篇:<br/>策略落日志 → applyChildComposition → descriptor
    F->>F: setupCommit?.commit() → appendUnstoredSuffix(828)
    F->>R: detachSession = sessions.enter(session)<br/>agent-loop/src/index.ts:664
    F->>R: detachAgent = agents.enter(agent, parentAgent)<br/>agent-loop/src/index.ts:667
    F->>C: sessions.announce(session) → agents.announce(agent)<br/>agent-loop/src/index.ts:668-670
    Note over C: 未发布窗口结束:监听者才第一次看到它
    F->>P: emitAgentEvent('agent/session-start')(675)
    D->>C: child.followup(createUserMessage(prompt))<br/>driver/src/index.ts:181
    D->>C: await child.whenIdle()(182)
    D->>U: readResult(child, boundary, cancelled, structured)<br/>driver/src/index.ts:212-238
    U->>T: SubagentResult{output, structured?, diagnostic?, stopReason}
    Note over T: stopReasonError(156) → 非 completed 即 throw<br/>withDiagnosticAndPartialText → isError 工具结果
    T->>T: run.dispose()(225),结果失败优先于 dispose 失败
```

</details>

</details>

### 三条路线的分叉点

同一次委派,前台、one-shot 后台、续存三条路线在工具层就分开了:前台会一直等到子 Agent 跑完再拿结果;one-shot 后台把这次委派包成一条作业,先把 jobId 交还给模型;续存路线则让子 Agent 常驻下来,只回一个 subagentId,之后还能用 `send_message` 追加消息。三者的共同点是"造 Agent"这一步完全一样,差别只在结果怎么回去、以及要不要保留这个子 Agent。展开见 [04](./04-continuation-and-control.md) 与 [06](./06-jobs-and-notifications.md)。

<details><summary>原图:三条路线的分叉树</summary>

```text
tool-subagent.execute                                       (tool-subagent/src/index.ts:471)
 ├─ continuable → ctx.subagents.startContinuable       (:530)
 │                 └─ SubagentContinuationManager.startContinuable  (continuation.ts:102)
 │                      └─ ContinuableActivationRegistry.materialize (continuation-activation.ts:458)
 │                           └─ ctx.agents.create(...)  ← 与前台同一条创建事务
 ├─ one-shot 后台 → ctx.jobs.start({kind:'subagent', owner:parent, run})  (:544)
 │                 └─ LocalJobRegistry.start           (jobs-local/src/index.ts:131)
 │                      └─ run() 里仍调用 ctx.subagents.start(...)  (:550)
 └─ 前台       → ctx.subagents.start(provider, {...request, signal})      (:563)
```

</details>

结果回流只有两条路:前台把委派结果直接交回工具调用者;后台走作业完成通知,由通知自己决定是给 owner 注入一条消息,还是新开一个 turn(`tool-jobs/src/index.ts:278-299`)。这两条路的代码完全不共用,唯一的共同点是开头那一步——都得先造出一个真 Agent。

---

## 四、额外覆盖的机制

`enter`/`announce` 的 reentrancy 规则、`agent/disposed` 配对、`detachRequested`;provider 能力位逐项;`scope 父链 → preset standing key` 的后果推演;`coldResume` 的授权阶梯;worker 的 slot/waiter/terminate 三态;`jobs` 的 `servesOwner` 与 layer 化监听;preset 的两道硬门逐行。

---

## 五、全模块不变量速查

1. **创建即事务**:`setup` 期间 agent/session 都未发布;`setup` 抛错、commit 抛错、owner 销毁三者任一发生,都回滚且不发布任何 id(`packages/core/agent/src/index.ts:100-118`)。
2. **所有权是能力**:`AgentHandle.dispose` 只交给创建者;`ctx.agents.get(id)` 只返回裸 `Agent`(`packages/core/agent/src/index.ts:146-163`)。
3. **存在 ≠ 授权**:`withInitiator` 只做进程内因果归属,跨边界一律显式传主体(`packages/core/agent/src/index.ts:319`)。
4. **能力先声明后校验**:provider 的 `capabilities` 五元组在 `start()` 里逐项 fail-loud,绝不"接受后忽略"(`packages/subagent/subagent/src/index.ts:641-657`)。
5. **子 agent 继承的是 preset standing 层,不是父 agent 自己的 scope 层**(`packages/preset/agent-presets/src/mount.ts:243-251`)。
6. **委派策略在第一个 await 之前捕获**,并以 `source:'delegation'` 写进子会话日志(`child-agent.ts:242-268`)。
7. **致命错误上抛,普通失败降级**:workflow 的 `parallel`/`pipeline` 只把普通异常降为 `null`(`runtime.ts:414-425,444-458`)。
8. **结算 first-wins,通知最后发**:`settle()` 先提交记录、释放 waiter,再通知监听者(`jobs-local/src/index.ts:416-440`)。
9. **preset 挂载两道硬门**:有行未激活整树拒绝;有行把服务发布到 root realm 整树拒绝(`mount.ts:403-413`)。

---

## 六、关键文件/符号索引表

| 文件 | 符号 | 本目录中的角色 |
|---|---|---|
| `packages/core/agent/src/index.ts` | `AgentRegistry.create/resume` `enter` `announce` `withInitiator` | 创建入口与所有权 |
| `packages/core/agent/src/runtime-types.ts` | `Agent`(运行面)、`Inbox`、`agent/*` 事件 | Agent 公开契约 |
| `packages/core/agent-loop/src/index.ts` | `createAgent` `resumeWith` `setupAndPublish` `prepare` | 发布事务 |
| `packages/core/agent-loop/src/agent.ts` | `ReactLoopAgent`(scope、followup/steer/inject) | 驱动实现 |
| `packages/subagent/subagent/src/index.ts` | `SubagentRuntime.start/startContinuable/sendMessage/interrupt` | 能力缝 Service Definition |
| `packages/subagent/subagent/src/types.ts` | `SubagentProvider` `SubagentStartRequest` `SubagentResult` `SubagentRun` | 全字段契约 |
| `packages/subagent/subagent-in-process-driver/src/index.ts` | `startInProcessRun` `drivePublishedRun` `readResult` | 一次性驱动 |
| `packages/subagent/subagent/src/child-agent.ts` | `applyChildComposition` `captureDelegatedPolicyOverrides` `childSessionMeta` | 子世界组装 |
| `packages/subagent/subagent/src/continuation.ts` | `startContinuable` `sendMessage` `coldResume` | 续存编排 |
| `packages/subagent/subagent/src/continuation-activation.ts` | `materialize` `interrupt` `submitAdmitted` | Activation 图 |
| `packages/subagent/tool-subagent-control/src/list-agents.ts` | `statusOf` | 状态读取 |
| `packages/workflow/workflow-worker-thread/src/runtime.ts` | `agent` `parallel` `pipeline` `acquireSlot` | 脚本钩子 |
| `packages/workflow/workflow-worker-thread/src/host.ts` | `WorkerRun.startChild/cancel/dispose/onResult` | 宿主侧 RPC |
| `packages/jobs/jobs-local/src/index.ts` | `start` `kill` `wait` `settle` `servesOwner` | 本地实现 |
| `packages/jobs/tool-jobs/src/index.ts` | `job_output` `job_kill` `onJobDone` 唤醒预算 | 模型侧消费 |
| `packages/preset/agent-presets/src/index.ts` | `mount` `composeFrom` `ensureStanding` | preset 组合 |
| `packages/preset/agent-presets/src/mount.ts` | `mountPreset` `inactiveRows` `leakedServices` `standingMountFor` | 两道硬门 |
