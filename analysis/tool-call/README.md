# Tool Call 模块 · 函数级深度展开

> 分析对象:[innokria/deepseek-harness](https://github.com/innokria/deepseek-harness) @ `dbbaa4a37`
> 本目录是 [第五章 · Tool Call 机制实现细节](../05-tool-call.md) 的子模块文档集,把总览中每一个"一句话结论"落回到具体函数、具体分支、具体 `路径:行号`。
> 分析过程未修改仓库任何文件。

---

## 本模块与第五章的分工

第五章回答**"是什么、为什么这样设计"**;本目录回答**"这一行代码怎么走的、失败时走到哪一支"**。

| 维度 | 第五章(`../05-tool-call.md`) | 本目录 |
|---|---|---|
| 粒度 | 契约与机制总览,函数名 + 结论 | 函数级走查:进入条件、分支、产出对象形态 |
| 覆盖 | `ToolDefinition` 契约、分层注册表、四段式管道、并发调度、PTC、取消六节 | 同上六节各自展开成一篇,每篇 200–450 行 |
| 引用 | 关键结论标注行号 | 每段代码块、每个分支都标注行号;附"关键文件/符号索引表" |
| 图表 | 1 张总览流程图 | 每篇 ≥1 张 mermaid / ASCII 图(分层图、时序图、状态机、泳道图) |

**不重复的部分**:第五章第零节的总览结论、第一节 `ToolDefinition` 契约的四条语义、第三节 `schemaOf` 白名单投影的动机、第六节 `run_code` 的定位说明,本目录只在必要处引用,不重写。**新增的部分**:`ScopedLayers` 的回收时序、`view()` 一次遍历的完整数据流、`prepare/dispatch/finalize/finish` 每一段的失败分支矩阵、`fillPool`/`commitReady`/`drive` 的并发时序约束、`fuseToolSignals` 的监听器生命周期、`run_code` 的 lane 状态机与背压、Web 卡片的派生链。

## 篇目索引

| 文件 | 主题 | 核心源码 | 一句话 |
|---|---|---|---|
| [01-registry-and-visibility.md](./01-registry-and-visibility.md) | 注册表内部:分层、遮蔽、回收、可见性 | `packages/core/scope/src/store.ts`、`packages/core/tools/src/index.ts` | `ScopedLayers` 提供"全局层 + 每作用域 overlay"的插入序存储;`view()` 用一次遍历同时产出 `visible`/`knownNames`/`restrictableNames`,喂给 `get`/`schemas`/`executionMode`/`resolveExecution`,四者结构上不可能漂移 |
| [02-execution-pipeline.md](./02-execution-pipeline.md) | 四段式管道的函数级走查 | `packages/core/tools/src/index.ts:1354-1852` | `createExecution` 定型身份与三个 WeakMap;`prepareExecution` 跑有序前置门;`dispatchScheduledExecution` 包 around 并调 body;`finishScheduledExecution` 三层 try 保证任何失败都降级成结构化错误 |
| [03-scheduler-and-concurrency.md](./03-scheduler-and-concurrency.md) | agent-loop 侧调度器与并发时序 | `packages/core/agent-loop/src/tool-calls.ts`、`index.ts:1266` | 分组由首个未提交调用的 `executionMode` 决定;`fillPool` 滚动补池并在每次 `await` 后重读 abort;`commitReady` 只推进连续槽位,提交严格模型序 |
| [04-cancellation-and-timeout.md](./04-cancellation-and-timeout.md) | 取消、双码语义与协作式超时 | `index.ts:1500-1549,1879-1934`、`packages/guard/timeout-policy/src/index.ts`、`packages/util/timeout/src/index.ts` | 注册表把调用者信号 fuse 回 wrapper 的替换信号,取消从不放弃 promise;`ABORTED` 与 `ABORTED_BEFORE_DISPATCH` 只由 `bodyInvoked` 一个布尔区分 |
| [05-ptc-mode.md](./05-ptc-mode.md) | `run_code` 模式:契约、SDK、lane、背压 | `packages/core/tools/src/ptc.ts`、`ts-types.ts`、`py-types.ts` | 子派发走单条有序 lane + 有界并发池,复刻原生时序;`collapse` 谓词一处定义两处使用,"模型直呼被拒"返回带路线提示的可纠正错误 |
| [06-presentation-and-ui.md](./06-presentation-and-ui.md) | 展示层与 Web 卡片派生 | `packages/core/tools/src/presentation.ts`、`packages/client/ui-tool/`、`packages/client/ui-chat/`、`packages/web/tool-web/` | Host presenter 是纯函数且**不是** Web 卡片的来源;卡片由 Client 从 `tool/call` 参数、`tool/result` 内容与持久化 `meta` 自己派生 |

推荐阅读顺序:01 → 02 → 03 → 04(主线),05、06 为两条独立支线。

---

## 一次工具调用的完整函数级调用栈

以**原生模式**下一次模型工具调用为例(文件内行号均为 `dbbaa4a37`)。括号内是 `路径:行号`。

先看这张图的两个基点——栈的入口,以及它下面那一层调度器接口:

```typescript
// packages/core/agent-loop/src/tool-calls.ts:60-67
export async function executeToolCalls(
  ctx: Context,
  turn: number,
  step: number,
  toolCalls: ToolCallBlock[],
  signal: AbortSignal,
  acceptContext: (context: UserMessage) => void,
): Promise<{ concluded: boolean }> {
```

```typescript
// packages/core/tools/src/index.ts:444-453
export interface ToolRuntimeScheduler {
  /** Materialize input, run the ordered pre-execute/guard gate, and decide what stage follows. */
  prepare(exec: ToolExecutionInput): Promise<ScheduledToolPreparation>
  /** Run only the around-dispatch/body stage. */
  dispatch(exec: ToolRunContext): Promise<ScheduledToolDispatch>
  /** Run post-execute and definition-owned content finalization, then materialize and notify. */
  finalize(exec: ToolRunContext, result: ToolExecutionResult): Promise<ToolExecutionResult>
  /** Run definition-owned content finalization, then materialize and notify without post-execute. */
  finish(exec: ToolRunContext, result: ToolExecutionResult): ToolExecutionResult
}
```

```text
ReactLoopAgent.step()                                        agent.ts:307 → :486
├─ message.content.filter(block => block.type === 'tool-call')          agent.ts:486
└─ executeToolCalls(loopCtx, turn, step, toolCalls, signal, acceptor)   tool-calls.ts:60
   ├─ ctx.agents.requireInitiator()                                     tool-calls.ts:68
   ├─ toolCalls.map(...) → PlannedCall[]                                tool-calls.ts:72
   │  └─ parseArguments(raw)                                            tool-calls.ts:105
   │     ├─ raw === ''            → {}                                  tool-calls.ts:107
   │     ├─ JSON.parse 成功        → 解析值                              tool-calls.ts:107
   │     └─ JSON.parse 抛错        → 保留原文(交给工具 schema 报错)     tool-calls.ts:109
   └─ while (next < planned.length)                                     tool-calls.ts:85
      ├─ ctx.tools.executionMode(first.exec).kind                       index.ts:1266
      │  └─ resolveExecution(name, agent, parent !== undefined)         index.ts:1211
      │     ├─ get(name, scope)                                         index.ts:1194
      │     │  └─ view(scope)                                           index.ts:1142
      │     │     ├─ layers.chainLayers(scope)  远处祖先在前             store.ts:192
      │     │     ├─ layers.peek(scope)         本层,chain-blind        store.ts:180
      │     │     ├─ inherited = global.tools + 各祖先层(跳过 own)      index.ts:1151
      │     │     ├─ layers.every(l => l.admits(name)) → visible        index.ts:1164
      │     │     ├─ own.tools → visible(过滤之外)                      index.ts:1168
      │     │     └─ modeFor(scope) !== 'native' → visible.set(run_code) index.ts:1179
      │     └─ collapses(name, scope, nested)                           index.ts:1314
      ├─ group = parallel ? planned.slice(next) : [first]               tool-calls.ts:90
      └─ runGroup(ctx, turn, step, group, mode, signal, acceptContext)  tool-calls.ts:122
         ├─ fillPool()                                                  tool-calls.ts:199
         │  └─ startCall(i)                                             tool-calls.ts:165
         │     ├─ appendToolCall → session.append('tool/call', …)       tool-calls.ts:263
         │     └─ ctx.tools[TOOL_RUNTIME_SCHEDULER].prepare(exec)       index.ts:790
         │        └─ prepareScheduledExecution                          index.ts:1449
         │           └─ prepareExecution(input, next)                   index.ts:1453
         │              ├─ createExecution(input)                       index.ts:1354
         │              │  ├─ get(name, agent) + collapses(...)         index.ts:1370
         │              │  ├─ snapshotJsonValue(exec.arguments)         index.ts:1402
         │              │  ├─ deepFreeze(detached)                      index.ts:1406
         │              │  ├─ deferredContexts.set(execution, [])       index.ts:1407
         │              │  ├─ contentFinalizers.set(execution, f)       index.ts:1408
         │              │  └─ cancellationStates.set(execution, …)      index.ts:1409
         │              ├─ callerCancelled(exec)                        index.ts:1460
         │              ├─ ctx.waterfall(scopeTarget(this, agent),
         │              │     'tools/pre-execute', exec, → allow)       index.ts:1465
         │              ├─ gate.kind === 'ask' → serviceAsk(exec, gate) index.ts:1469 / :1679
         │              ├─ guardReason(exec)                            index.ts:1109
         │              └─ return { kind: 'dispatch', exec }            index.ts:1493
         │     └─ switch (prepared.kind)                                tool-calls.ts:172
         │        ├─ 'dispatch'    → dispatch(prepared.exec)            index.ts:791
         │        ├─ 'post-result' → slot{needsPost: true}              tool-calls.ts:188
         │        └─ 'final-result'→ slot{needsPost: false}             tool-calls.ts:191
         ├─ dispatch(prepared.exec)                                     index.ts:791
         │  └─ dispatchScheduledExecution(exec)                         index.ts:1559
         │     ├─ ctx.waterfall(…, 'tools/execute', mutableExec,
         │     │     () => dispatchToolBody(mutableExec))               index.ts:1563
         │     │  └─ dispatchToolBody(exec)                             index.ts:1522
         │     │     ├─ fuseToolSignals(callerSignal, exec.signal)      index.ts:1879
         │     │     ├─ isAborted(signal) → ABORTED_BEFORE_DISPATCH     index.ts:1530
         │     │     ├─ resolveExecution(...) 再解析一次                 index.ts:1536
         │     │     ├─ state.bodyInvoked = true                        index.ts:1538
         │     │     ├─ await tool.execute(exec.arguments, exec)        index.ts:1539 ← 工具体
         │     │     ├─ createSuccessResult(exec, tool, returned)       index.ts:1783
         │     │     │  ├─ snapshotToolValue                            index.ts:537
         │     │     │  ├─ validateJsonSchemaValue(output.schema, …)    index.ts:1785
         │     │     │  ├─ deepFreeze(value)                            index.ts:1787
         │     │     │  ├─ tool.output.render(args, value)              index.ts:1790
         │     │     │  ├─ snapshotProjection(…, 'render', …)           index.ts:1794
         │     │     │  └─ parent === undefined → presentationMeta      index.ts:1796
         │     │     └─ finally: fused.dispose(); exec.signal = wrapper index.ts:1547
         │     ├─ normalizeDispatchResult(exec, result)                 index.ts:1816
         │     └─ deferredContexts → additionalContexts(前置)           index.ts:1571
         ├─ while (inFlight.size > 0)                                   tool-calls.ts:221
         │  └─ commitReady()                                            tool-calls.ts:147
         │     ├─ slot.needsPost
         │     │  ├─ true  → finalize(exec, result)                     index.ts:792
         │     │  │  └─ finalizeScheduledExecution                      index.ts:1599
         │     │  │     ├─ postExecute(exec, result)                    index.ts:1732
         │     │  │     ├─ callerCancelled → cancellationResult         index.ts:1604 / :1508
         │     │  │     └─ finishScheduledExecution(exec, result)       index.ts:1621
         │     │  └─ false → finish(exec, result)                       index.ts:793
         │     │     └─ finishScheduledExecution                        index.ts:1621
         │     │        ├─ materializeFinalResult(result)               index.ts:1837
         │     │        ├─ applyFinalContent(exec, …) → finalizeContent index.ts:1639
         │     │        ├─ materializeFinalResult(…)  第二次             index.ts:1630
         │     │        └─ notifyResult: Object.freeze(exec)
         │     │              + emit 'tools/result'                     index.ts:1647
         │     ├─ appendToolResult → session.append('tool/result', …,
         │     │        { surfaceOp: 'append', sourceEventSeqs: [callSeq] })  tool-calls.ts:269
         │     └─ result.additionalContexts → acceptContext(每一条)     tool-calls.ts:157
         └─ 返回 { consumed, aborted, concluded }                       tool-calls.ts:246
└─ concluded ? { kind: 'completed' } : null                             agent.ts:492
```

`acceptContext` 就是 `agent.ts:490` 传入的闭包,把上下文 `splice` 进 `inbox.nextStep` 尾部;它在**下一个 step 边界**随 `preClaim` 一起投给模型(`agent.ts:244-255`)。工具结果本身则走 `tool/result` 会话事件,由 `deriveMessages()` 变成消息序列的权威副本——两条通道互不替代。

图中 `get(name, scope)` 取到的那个对象,其类型就是:

```typescript
// packages/core/tools/src/index.ts:214-280(节选)
export interface ToolDefinition extends ToolSchema {
  readonly output: ToolOutputDefinition
  // ...(略)
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>
  // ...(略)
  finalizeContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  // ...(略):presentCall 与各成员的长 JSDoc
  timeoutMs?: number
  isConcurrencySafe?(args: unknown): boolean
  presentResult?(args: unknown, result: ToolResult): ToolResultView | undefined
}
```

`execute` 返回的是**规范化 JSON 值**而不是内容块:`ContentBlock[]` 由 `output.render` 在 `createSuccessResult` 里投影出来(`index.ts:1790`),这正是 `finalizeContent` 能在最后一米重写内容的余地。

### PTC 模式下的分叉

`mode === 'ptc'` 时同一张图在 `dispatchToolBody` 处换成 `run_code` 工具体,再由它自己开一条有序 lane:

```text
tool.execute = run_code body                              ptc.ts:327
├─ runtime.run({ program: args.code, bindings: [tools] }) ptc.ts:619
│  └─ 程序内 await tools.grep(...) → binding(name)        ptc.ts:463
│     ├─ jsonNormalizeArgs(rawArgs) → {dispatched, logged} ptc.ts:150 / :467
│     ├─ subCallId = `<parent>:ptc:<n>`                    ptc.ts:469
│     ├─ input.parent = exec.token  ← 折叠的唯一豁免凭据    ptc.ts:476
│     └─ pendingQueue.push(…); wakeup(); void drive()      ptc.ts:524 / :585
│        └─ drive() 有序 lane                              ptc.ts:392
│           ├─ commitQueue[0].settled → await commit()     ptc.ts:401
│           ├─ pendingQueue[0] → classify() → capacity     ptc.ts:410-421
│           │  └─ await start(): append(start) + prepare   ptc.ts:533
│           └─ 全空 → quiescence 返回                       ptc.ts:437
└─ finally: runController.abort('run_code settled')
     → await drainDispatches()                            ptc.ts:632-633
```

详见 [05-ptc-mode.md](./05-ptc-mode.md)。

### 四段在 `ToolRuntime` 上的真实签名

栈图里 `prepare` / `dispatch` / `finalize` 三个节点的方法签名(均为节选):

```typescript
// packages/core/tools/src/index.ts:1453-1459
  private async prepareExecution<T>(
    input: ToolExecutionInput,
    next: (prepared: ScheduledToolPreparation) => T | PromiseLike<T>,
  ): Promise<T> {
    const created = this.createExecution(input)
    if (created.kind !== 'ready') return next(created)
    const exec = created.exec
```

```typescript
// packages/core/tools/src/index.ts:1559-1564
  private async dispatchScheduledExecution(exec: ToolRunContext): Promise<ScheduledToolDispatch> {
    try {
      const mutableExec = exec as MutableToolRunContext
      const carrier = scopeTarget(this, exec.agent)
      const result = await this.ctx.waterfall(
        carrier, 'tools/execute', mutableExec,
```

```typescript
// packages/core/tools/src/index.ts:1599-1607
  private async finalizeScheduledExecution(exec: ToolRunContext, result: ToolExecutionResult): Promise<ToolExecutionResult> {
    try {
      const postResult = await this.postExecute(exec, result)
      return this.finishScheduledExecution(
        exec,
        this.callerCancelled(exec) && !postResult.isError
          ? this.cancellationResult(exec, postResult)
          : postResult,
      )
```

三段都只有一条 `try`,失败一律交给 `finishScheduledExecution`(`:1621`,同步、自身还有两层 `try`)降级成结构化错误——这就是"任何失败都不会逃出管道"的落点。

---

## 关键文件总表

| 文件 | 行数 | 本模块用到的核心符号 |
|---|---|---|
| `packages/core/tools/src/index.ts` | 1936 | `ToolRuntime`(`:780`)、`view`(`:1142`)、`register`(`:1027`)、`restrict`(`:1061`)、`collapses`(`:1314`)、`prepareExecution`(`:1453`)、`dispatchToolBody`(`:1522`)、`postExecute`(`:1732`)、`createSuccessResult`(`:1783`)、`materializeFinalResult`(`:1837`)、`fuseToolSignals`(`:1879`) |
| `packages/core/tools/src/ptc.ts` | 678 | `createRunCodeTool`(`:293`)、`drive`(`:392`)、`binding`(`:463`)、`settle`(`:486`)、`drainDispatches`(`:448`)、`resolveFlavor`(`:112`) |
| `packages/core/tools/src/presentation.ts` | 389 | `ToolCallView`(`:46`)、`ToolResultView`(`:140`)、`ReadResultView`(`:281`)、`WebResultView`(`:347`) |
| `packages/core/tools/src/types.ts` | 58 | `tool/ptc-dispatch-start`(`:40`)、`tool/ptc-dispatch`(`:56`) |
| `packages/core/scope/src/store.ts` | 267 | `NamedEntries`(`:30`)、`AnonymousEntries`(`:114`)、`ScopedLayers`(`:159`)、`effect`(`:226`) |
| `packages/core/scope/src/index.ts` | 204 | `scopeOf`(`:154`)、`scopeChainOf`(`:98`)、`scopeTarget`(`:170`)、`bindScopeParent`(`:72`) |
| `packages/core/agent-loop/src/tool-calls.ts` | 290 | `executeToolCalls`(`:60`)、`runGroup`(`:122`)、`commitReady`(`:147`)、`startCall`(`:165`)、`fillPool`(`:199`)、`appendSkippedToolCall`(`:250`) |
| `packages/core/agent-loop/src/constants.ts` | 6 | `DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10`(`:5`) |
| `packages/guard/timeout-policy/src/index.ts` | 81 | `apply`(`:55`)、`TOOL_TIMEOUT`(`:25`)、`toolTimeoutResult`(`:41`) |
| `packages/util/timeout/src/index.ts` | 190 | `TimeoutReason`(`:12`)、`deadline`(`:91`)、`timeoutOf`(`:184`) |
| `packages/core/agent-tool-presentation/src/index.ts` | 72 | `apply`(`:59`)——agent 面 `presentAs` 选择器 |

---

## 声明

> 本文档集为对公开源码仓库的静态阅读分析,所有结论均来自对 `packages/` 的实际阅读并标注 `路径:行号` 供核对。DeepSeek Harness 的所有权利归其原权利人所有;分析中的任何错漏以仓库源码与官方文档为准。
