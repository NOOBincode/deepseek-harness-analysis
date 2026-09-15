# 第三章:Session 与 Memory 机制(DeepSeek Harness 源码分析)

> 分析对象:[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) @ `dbbaa4a37`(pnpm monorepo)
> **深入阅读(函数级)**:[`memory/`](./memory/README.md) —— 事件日志提交路径、surface 可见性与 provenance、派生投影与水印、压缩选区与事务、落盘与恢复、状态型记忆
> 版本权威:`docs/session-format-status.md`(当前写者格式 v3,已随 `dsh-v0.1.5-alpha.1` 发布)

---

## 第〇节 一句话结论与总览

DSH 的 Session 是**一个纯事件溯源(event-sourced)系统**:`packages/core/session/src/types.ts:269` 的 `SessionEventMap` 声明了一张可合并扩展(declaration merging)的追加式(append-only)事件日志,它是会话的**唯一权威**;模型可见历史、请求头、UI 投影、统计、标题、压缩全部是从日志**派生**出来的折叠(fold)。仓库的根级不变式写死在 `AGENTS.md`:**"模型可见 ⟺ 已落日志"(Model-visible ⟺ logged)**——任何进入模型请求的内容必须能从 Session 日志重建;要让模型看到新东西,只能新增一个 Session 事件。

围绕这条不变式,机制分四层:

1. **日志层(core/session)**:`Session.append()` 同步写入内存日志并广播 `session/event`;seq 严格连续(`seq = log.length`),事件深冻结、强制 JSON 无损序列化。
2. **派生层(surface + projection)**:`SurfaceManager` 在日志之上维护一张"模型可见节点有序表"(surface);`Session.deriveMessages()` 把 surface 折叠成 LLM `Message[]`;`ctx.sessionProjections` 注册表把任意领域状态(todo 清单、标题、统计)折叠成带水印的增量单元。
3. **持久层(session-persistence)**:`ctx.sessionPersistence` 是 Service Definition;`session-persistence-jsonl` 用"每个会话一个目录、每个格式世代一个 JSONL(可 zstd 压缩)文件"落地,追加是 best-effort,`flush` 才是崩溃存活保证;格式世代(v0→v3)由 `session-format*` 的相邻迁移链负责升级。
4. **改写层(compaction)**:`compaction/*` 事件是 log-only 的锁与摘要记录;真正的"压缩"是一条携带 `surfaceOp: { op: 'replace', startSeq, endSeq }` 的 `user/message`,把 surface 上的一段节点遮蔽(shadow)成一个摘要节点——**日志从不删除,只有 surface 被重写**。

```text
                agent-loop (ReactLoopAgent)
                |  每步: append(事件)          ^  每步: deriveMessages() 重建请求
                v                              |
+---------------------- 内存层 -----------------------------------------+
| Session.log : SessionEvent[]  (追加式, seq 连续, 深冻结)               |
|   |- SurfaceManager --> surface.nodes (模型可见节点有序表, replace 重写) |
|   |- deriveMessages() --> Message[] (缓存投影, replaceGeneration 失效)   |
|   |- requestHeader()/requestContext() --> 增量折叠的 EpochHeader/路由    |
|   '- sessionProjections --> 领域单元 (todos / sessionStats / title ...) |
+-----------+--------------------------------------------+--------------+
            | session/event (firehose)                    | session/flush (并行屏障)
            v                                             v
+---------------------- 持久层 -----------------------------------------+
| ctx.sessionPersistence (Service Definition)                           |
|   '- session-persistence-jsonl: <root>/<project>/<sid>/log.v3.jsonl   |
|      首行 header + 每事件一行; torn-tail 截断修复; 跨进程写锁 (lease)    |
|      读路径: scanLog -> sessionFormatCatalog (v0..v3 codec + 迁移链)    |
+-----------+--------------------------------------------+--------------+
            | 派生/查询读模型(全部可丢弃重建)                              |
            v
+-----------------------------------------------------------------------+
| session-projection-cache (storage-domain KV, per-record JSON/SQLite)  |
| session-query-sqlite (FTS5 全文索引, live-preferred 双源合并)           |
| session-telemetry / session-log-deepseek (出站遥测与日志投递)           |
+-----------------------------------------------------------------------+
```

---

## 第一节 事件溯源架构:Session 日志为唯一权威

### 1.1 事件信封与类型词汇

每个事件是一个以 `type` 为判别式的真联合(`packages/core/session/src/types.ts:465`):

```typescript
export type SessionEvent<T extends SessionEventType = SessionEventType> = {
  [K in SessionEventType]: {
    type: K
    /** Monotonic sequence number within the session. */
    seq: SessionSeq
    /** Unix epoch milliseconds. */
    time: number
    data: SessionEventMap[K]
    /** 读者不认识 type 且无此标记时必须拒绝重建,而不是静默丢弃 */
    ignorable?: true
  } & (K extends SurfaceEventType ? SurfaceIntent<K> : {
    surfaceOp?: never          // 非 surface 事件编译期禁止携带 surface 元数据
    sourceEventSeqs?: never
  })
}[T]
```

上面是文档化的简写;真实源码里 `ignorable` 的完整读者契约(`types.ts:473-482`)是这样钉住的:

```typescript
// packages/core/session/src/types.ts:466-488
[K in SessionEventType]: {
  type: K
  seq: SessionSeq
  time: number
  data: SessionEventMap[K]
  // ...(略): 468-482 行是 seq / time / data 的字段注释与 `ignorable` 的完整读者契约
  ignorable?: true
} & (K extends SurfaceEventType ? SurfaceIntent<K> : {
  surfaceOp?: never
  sourceEventSeqs?: never
})
```

核心词汇(`SessionEventMap`,`types.ts:269-401`)分四类:

| 类别 | 事件 | 语义 |
|---|---|---|
| 边界 | `turn/start` `turn/end` `step/start` `step/end` | turn = 一轮用户输入到完成;step = 一次模型调用加其工具执行;`turn/end.reason` 是 `completed/aborted/blocked/error/max-tokens/interrupted` 的封闭联合(`types.ts:200`) |
| 消息(surface 事件) | `system/message` `user/message` `assistant/message` `tool/result` | 仅有的四个 `SurfaceEventType`(`types.ts:412`),必须携带 `surfaceOp`;`assistant/message` 内嵌精确的压缩后原始流 `stream` 与 `usage`,并支持 `interrupted: true` 标记中断前缀 |
| 请求锚 | `request/header` `request/context` | 下一次请求的完整头快照(config/adapterDefaults/tools)与路由元数据;log-only,最新快照即重建值(`request-header.ts:63` 的 `foldRequestHeader`) |
| 生命周期 | `session/end-seed` `assistant/attempt` `tool/call` | `end-seed` 是构造 seed 边界的持久投影;`attempt` 记录未产出 surface 消息的失败/取消尝试;`tool/call` 记录模型原始 arguments 字符串 |

词汇扩展走 **declaration merging**:如 compaction 在 `packages/compaction/compaction/src/types.ts:17` 向 `SessionEventMap` 合并 `compaction/start|summary|end|prune`;session-title 在 `packages/session/session-title/src/index.ts:71` 合并 `session/title`;todo 合并 `todo/write`。**不 bump 版本号**:`SESSION_FORMAT_VERSION`(`types.ts:88`,当前 `3`)只在结构性变化(header 形状、事件信封、核心事件语义、surface 机制)时递增;新增普通事件类型由信封上的 `ignorable: true` 守卫兜底——读者遇到未知且未标记的事件必须**拒绝重建**(`packages/session/session-persistence/src/storage-contract.ts:69` 的 `validateStoredEvents`)。本构建认识的全部词汇由生成文件 `packages/core/session/src/known-event-types.ts:22` 枚举(56 种)。"拒绝重建"落在后端共享校验里,失败信息就是给用户看的:

```typescript
// packages/session/session-persistence/src/storage-contract.ts:74-80
for (const event of events) {
  if (!KNOWN_SESSION_EVENT_TYPES.has(event.type) && event.ignorable !== true) {
    throw unsupported(
      `session "${meta.id}" contains event type "${event.type}" (seq ${event.seq}) unknown to this harness and not marked ignorable; refusing to interpret the log — it was likely written by a newer harness`,
      location,
    )
  }
}
```

### 1.2 追加路径:验证先于落日志

`Session.append()`(`packages/core/session/src/index.ts:710`)是唯一的写入入口,顺序是:**快照 → 验证 → 提交 → 通知**:

```typescript
append<T extends SessionEventType>(type: T, data: SessionEventMap[T], ...opts) {
  const dataSnapshot = snapshotJsonValue(data)          // 一次遍历完成"读取+校验+拷贝"
  if (dataSnapshot === undefined) throw new Error(`session event "${type}" carries non-JSON-serializable data`)
  ...
  const event = deepFreeze({ type, seq: SessionSeq(this.log.length), time: Date.now(), data: dataSnapshot, ... })
  validateSessionEventData(event, ...)                  // 事件级规则(request/header 禁带空 tools 等)
  this.surfaceManager.validateNext(event)               // surface 元数据与 seq 连续性,不改动已提交状态
  // 收集监听器快照在 log.push 之前,回调在之后
  this.log.push(event)                                  // 提交点
  invokeContainedSessionObservers(...)                  // 逐监听器 containment,失败不回滚已提交追加
  return event
}
```

要点(`index.ts:674-761` 的 JSDoc 明确约定):

- **热路径不阻塞 I/O**:持久化插件异步缓冲;事件一旦进入 log 即已提交,观察者失败只被记录。
- **防重入**:同一 store 条目上 `appending` 为真时再 append 直接抛错(`index.ts:729`)。
- **返回的是落入日志的快照**,不是调用方仍可变的输入。

上面伪代码对应的真实提交段(`index.ts:728-754`),"提交点"就是那一次 `log.push`:

```typescript
// packages/core/session/src/index.ts:728-754
// ...(略): 728-731 行是 `appending` 防重入守卫(见上文要点)
const event = deepFreeze({
  type,
  seq: SessionSeq(this.log.length),
  time: Date.now(),
  data: dataSnapshot,
  ...(surfaceMetadataSnapshot as { surfaceOp?: unknown; sourceEventSeqs?: unknown }),
} as unknown as SessionEvent<T>)
validateSessionEventData(event, `session event "${type}" at seq ${event.seq}`)
this.surfaceManager.validateNext(event as SessionEvent)
// ...(略): 742-748 行收集监听器快照,746-753 行只在 push 之后回调
this.log.push(event as SessionEvent)
```

- 构造 seed(resume/fork/replay)走**与 append 完全相同的验证**(`index.ts:562-582`):`snapshotJsonValue` 脱拷贝、信封白名单校验(`assertSessionEventEnvelope`,`index.ts:199`)、seq 必须从 0 连续、`surfaceManager.validateNext` 逐个预检——"不能构造出任何持久化后端无法存储的活日志"。

### 1.3 关系不变式:invariant 伴随插件

包级不变式(`packages/core/session/src/invariant.ts`)以 Cordis 伴随插件形式安装,在 `internal/dispatch` 阶段对候选事件做**预提交纯校验**(结果暂存 WeakMap),`session/event` 到达时才把转移应用到已提交 trace(`invariant.ts:227-245`)。校验项包括:seq 严格递增、turn/step 编号恰好递增且嵌套合法、`tool/result` 必须有本步内的 `tool/call` 配对(合成 `TOOL_NOT_STARTED` 修复事件除外,`invariant.ts:138`)、`request/header` 与 `request/context` 必须在打开的 turn 内(`invariant.ts:154-159`)等。

"预提交纯校验 → 提交后才推进"的两半写在同一张 `stagedTransitions` WeakMap 上(`invariant.ts:237-245`):

```typescript
// packages/core/session/src/invariant.ts:237-245
ctx.on('internal/dispatch', (_mode, eventName, args) => {
  if (eventName !== 'session/event') return
  const [session, event] = args as [Session, SessionEvent]
  const trace = traceFor(session)
  const transition = validateEvent(trace, event, fail)
  // A later dispatch listener may veto. Validation is pure, so abandoning
  // this weakly keyed transition does not advance or retain the session.
  stagedTransitions.set(event, { session, trace, transition })
}, { global: true })
```

已提交事件的另一侧消费同一份暂存转移(`invariant.ts:227-235`):取不到匹配的暂存记录就直接 `fail('session/event reached publication without matching pre-commit validation')`;否则 `stagedTransitions.delete(event)` 之后才 `applyTransition(staged.trace, staged.transition)`。

### 1.4 存储元数据不入日志

`SessionHeader`(`types.ts:93-130`:version/id/createdAt/cwd/parentSession/isSeeded/origin/delegationDepth/agentPreset)是**日志之外的不可变存储元数据**——它是存储关注点,不是可重放的会话状态。fork 继承前缀的精确长度 `inheritedEventCount` 同样是 Session 状态而非 header 字段(`types.ts:108-111`);它的持久投影是日志里最后一条带 `{ inherited: true }` 的 `session/end-seed` 事件(`types.ts:379-400`)。`firstLiveSeq`(`index.ts:497`)区分"本进程追加的第一条 seq"与"构造时进入的 seed 前缀"。

---

## 第二节 Surface 与 Projection:派生机制

### 2.1 Surface:模型可见节点的有序表

Surface 是日志之上的派生层(`packages/core/session/src/surface.ts`),日志仍是唯一权威。只有四个 `SurfaceEventType` 能进入 surface,每个 surface 事件必须声明自己如何加入:

```typescript
export type SurfaceOp =
  | 'append'                                                    // 追加到尾部(正常路径)
  | { op: 'replace'; startSeq: SessionSeq; endSeq: SessionSeq } // 遮蔽 [startSeq..endSeq] 区间
```

折叠规则由 `SurfaceManager`(`surface.ts:504`)增量执行,离线的 `foldSurface()`(`surface.ts:487`)对完整日志重放同一规则。每次替换在**验证阶段**就接受一整套结构性检查(`planSurfaceEvent`,`surface.ts:421`):

- seq 连续性:`event.seq !== expectedSeq` 即拒(`surface.ts:428`);
- 来源完整:replace 节点的 `sourceEventSeqs` 必须包含**每一个**被遮蔽的 surface 节点(`assertProvenance`,`surface.ts:300`),且只能引用更早的 seq;
- `tool/result` 替换只允许改 `content`(其余字段深度相等,`assertToolResultRewrite`,`surface.ts:365`);
- **系统提示头部保护**:覆盖 surface 节点 0 的替换,若节点 0 是 `system/message`,替换者必须是恰好覆盖该节点的 `system/message`(`assertSystemHeadRewrite`,`surface.ts:404-418`)——压缩区间拿不到系统提示。

这四条检查不是散落的防御,而是 `planSurfaceEvent()` 里固定顺序的一段(`surface.ts:428-439`):先验 seq 连续性,再按 `append`/`replace` 分派,replace 才走三重断言:

```typescript
// packages/core/session/src/surface.ts:428-439
if (event.seq !== expectedSeq) {
  throw new Error(`session event seq ${event.seq} is not contiguous; expected ${expectedSeq}`)
}
const surfaceOp = validateSurfaceMetadata(event)
if (surfaceOp === undefined) return
if (surfaceOp === 'append') {
  return { kind: 'append', seq: event.seq }
}
const range = replacementRange(state, surfaceOp)
assertProvenance(event, range.shadowedSeqs)
assertToolResultRewrite(event, range.shadowedSeqs, events, baseSeq)
assertSystemHeadRewrite(event, state, range.startIdx, range.shadowedSeqs, events, baseSeq)
```

来源完整性断言的落点(`surface.ts:300-303`)——`shadowedSeqs` 的每一个都必须出现在 `sources` 集合里:

```typescript
// packages/core/session/src/surface.ts:300-417
const missing = shadowedSeqs.filter(seq => !sources.has(seq))
if (missing.length > 0) {
  throw new Error(`surface replace: sourceEventSeqs must include every shadowed surface node; missing ${missing.join(', ')}`)
}
// ...(略): 同文件 415-417 行的系统提示节点 0 保护
if (event.type !== 'system/message' || shadowedSeqs.length !== 1) {
  throw new Error('surface replace: node 0 holds the system prompt and may be rewritten only by a system/message over exactly that node')
}
```

`replaceGeneration` 计数器(`surface.ts:194`)是缓存失效信号:每次替换递增。

### 2.2 deriveMessages:从 surface 折叠模型输入

逐节点投影规则是纯函数 `deriveEventMessage()`(`surface.ts:92`):

```typescript
export function deriveEventMessage(event: SessionEvent): Message | null {
  switch (event.type) {
    case 'user/message': return event.data                       // 逐字投影(人类 prompt / agent.inject / goal 轮次)
    case 'system/message':
    case 'assistant/message':
      if (event.data.message.content.length === 0) return null  // 空内容节点不上线(max-tokens 占位/清空系统提示)
      return event.data.message
    case 'tool/result': return event.data.message
    default: return null                                        // 边界、attempt、log-only 记录不产消息
  }
}
```

`Session.deriveMessages()`(`index.ts:832`)把它折叠到活 surface 上,带三级缓存:逐节点只投影一次(O(新增节点));`replaceGeneration` 变化时整体重建;返回数组是每次新建的快照,但其中的 `Message` 对象是**共享且深冻结**的——派发、持久历史、模型请求引用同一份冻结消息。这就是"模型可见 ⟺ 已落日志"的执行机构:请求的 `messages` 字段除了 surface 没有第二个来源。

```typescript
// packages/core/session/src/index.ts:832-853
deriveMessages(): Message[] {
  const surface = this.surface
  const nodes = surface.nodes
  const generation = surface.replaceGeneration
  if (generation !== this.derivedGeneration) {
    this.derived = []
    this.derivedNodes = 0
    this.derivedGeneration = generation
  }
  for (const seq of nodes.slice(this.derivedNodes)) {
    const msg = this.deriveEventMessage(this.log[seq]!)
    // A surface node is one of the five message-producing types, but an
    // empty-content assistant/message (a max-tokens step that hosts only
    // usage) derives to null and must not enter the transcript.
    if (msg) this.derived.push(msg)
  }
  this.derivedNodes = nodes.length
  return [...this.derived]
}
```

### 2.3 请求头折叠与 reason

`request/header` 事件记录下一次请求的完整 `EpochHeader`(config + adapterDefaults + tools);`foldRequestHeader`(`request-header.ts:63`)取最新快照即重建值——纯离线路径就是一次 `if (event.type === 'request/header') state = canonicalHeader(event.data.header)` 的循环(`request-header.ts:63-69`),活 Session 用 `requestHeader()`(`index.ts:776`)增量维护同一折叠。

`reason` 四值(`types.ts:261`):`initial`(新会话首条)/ `resume`(进程重启或 fork seed 后的首个请求)/ `change`(头变化,可带 `startsSeries`)/ `series`(头未变但开新消息系列)。`request/context` 只在路由、容量或系统提示更新模式变化时记录,不参与请求重建。JSONL 存储层对 `sourceEventSeqs` 做区间压缩编码(`seq-ranges.ts:18` 的 `encodeSeqRanges`)。

### 2.4 领域投影:`ctx.sessionProjections`

surface 只回答"模型看到什么";UI/遥测/工具需要"会话现在处于什么状态"。`packages/session/session-projection/src/index.ts` 定义了投影能力缝(capability seam):

- 每个领域注册一个 `ProjectionDefinition`(`index.ts:48` 起):`key`、`stateSchema`(zod)、`init(header, inheritedEventCount)`、纯同步 `apply(state, event)`(不感兴趣的事件必须返回同一引用,`Object.is` 门控下游零工作)、可选 `wire.view`、`stateVersion`。
- 注册表 `SessionProjectionRegistry`(`index.ts:199`)对 `session/event` 只订阅一次,把每条已提交事件**急切地**驱过所有已注册单元(`drive`,`index.ts:220`);cell 按 `(unit, session)` 惰性建立,落后时从内存日志补折叠——门控就是一句 `const changed = !Object.is(next, previousState)`,引用不变则下游零工作(`index.ts:680-684`)。
- 读面:`stateOf`(host 内部状态)、`snapshot`(所有 wire 单元在同一 `asOfSeq` 水位线的一致切面,`index.ts:338`)、`onChanged` 变更推送。
- **承载规则**(模块头注释):携带状态的日志事件必须携带**变更后的完整状态**,绝不只带 delta——`todo/write` 写整份清单、`session/title` 是最新标题快照、`goal/change` 同理(`packages/goal/goal/src/index.ts:613`)。

实例:`todos` 单元(`packages/todo/tool-todo/src/index.ts`,`todo/write` 整表替换,turn/start 清空、turn/end 保留完成清单);`sessionStats` 单元(`packages/session/session-stats/src/projection.ts:113`,以 `step/end` 为步计数权威,配对 `tool/call→tool/result` 计时);`turnOutline` 单元(`packages/session/session-turn-outline/src/projection.ts:85`,turn 预览供聊天轨);标题服务把 `session/title` 事件折叠成 latest-wins 快照(`session-title/src/index.ts:71-78`)。

### 2.5 遥测与出站

`session-telemetry` 的协调器(`packages/session/session-telemetry/src/coordinator.ts:75`)订阅 `session/created|event|disposed|flush` 火 hose,把每条事件过 `session-telemetry/record` 瀑布(部署方可挂脱敏规则)后交给后端 sink;`session-telemetry-otel` 是 OpenTelemetry 后端。`session-log-deepseek`(`src/index.ts`)把会话日志增量贡献给官方 DeepSeek API 请求,已确认水位线写在规范日志里,重启后可保守重发不确定尾部。

---

## 第三节 Resume 与 Repair:从持久日志重建活会话

### 3.1 正常 resume 链路

`AgentLoop.resume()`(`packages/core/agent-loop/src/index.ts:844`)的主链路(伪代码改写):

```text
handle = sessionPersistence.open(id, 'write')          # 先抢写所有权,排除并发 resume
events = handle.read(0)                                # 物理有效日志(已完成格式迁移与 torn-tail 处理)
closers = interruptedTurnClosers(events)               # 语义级崩溃修复(见 3.2)
if closers: handle.append(closers)                     # 修复事件经同一 handle 落盘
session = sessions.prepare(id, { seed: events+closers, meta: handle.header,
                               inheritedEventCount, eventState })   # 构造即重放(见 1.2)
publish: sessions.enter(session); agents.enter(agent); sessions.announce(session)
```

之后 `ReactLoopAgent` 的第一次请求会追加 `request/header { reason: 'resume' }`(`agent.ts:571`)。整个构造 seed 经过与 append 相同的逐事件验证(见第一节),所以**resume 就是一次受控重放**;`Session.fromRestore`(`index.ts:530`)在事件已独立所有或已深冻结时跳过拷贝。

### 3.2 崩溃修复:interruptedTurnClosers

`packages/core/session/src/repair.ts:29` 生成确定性的合成收尾事件,关闭"崩溃尾巴":扫描持久日志,若末尾存在未关闭的 turn,则按序补:

1. 每个悬挂工具调用一条合成 `tool/result`(`isError: true`):调用曾记录开始的给 `TOOL_OUTCOME_UNKNOWN`("结果未持久化,是否重试由工具语义决定"),从未记录开始的给 `TOOL_NOT_STARTED`("可安全重试")——文本见 `repair.ts:106-107`;
2. 一条 `step/end`(若 step 打开);
3. 一条 `turn/end { reason: { kind: 'interrupted' } }`。

合成事件复用最后一条真实事件的时间戳,seq 接续日志;均衡日志返回空数组。

```typescript
// packages/core/session/src/repair.ts:81-133
const last = events.at(-1)
if (openTurn === null || last === undefined) return []
// ...(略): 84-86 行注释说明 seq 基线与时间戳复用最后一条真实事件
let seq = last.seq + 1
const time = last.time
// ...(略): 91-126 行按 Map 插入序为每个未配对 tool-call 补 isError 结果
// Close an open step next — a turn/end while a step is open is an invariant
// violation, so the step's boundary must be synthesized before the turn's.
if (openStep !== null) {
  closers.push({ type: 'step/end', seq: SessionSeq(seq++), time, data: { turn: openTurn, step: openStep } })
}
closers.push({ type: 'turn/end', seq: SessionSeq(seq++), time, data: { turn: openTurn, reason: { kind: 'interrupted' } } })
```

同一函数也被 `session-query` 的冷读复用:`readColdSessionLog`(`packages/session-query/session-query/src/cold-read.ts:31`)只读路径在内存中追加同样的事件、**不回写**,使崩溃中段的日志也能折叠成均衡 transcript。

### 3.3 物理层修复:torn tail 与世代文件

JSONL 后端把"物理有效前缀"与"语义均衡"分开处理:

- 读路径 `SessionLogScanner`(`packages/session/session-persistence-jsonl/src/format.ts:385`)按 `\n` 切完整记录;`finish()` 把**无换行结尾的最后一条**当作 torn tail 忽略,返回 `committedBytes` 安全截断点;`recoverable` 模式下一条无法解码的已提交行会抑制后续行,遇到 `turn/end` 才抛错(`format.ts:497-515`)——宁缺毋滥,绝不向读者返回撕裂尾部。

切分与残片拼装是同一段循环(`format.ts:426-441`),只有走完 `\n` 的记录才会进入 `consumeEventLine`,尾部残片被拷进 `fragments` 等下一批:

```typescript
// packages/session/session-persistence-jsonl/src/format.ts:426-436
const fragment = chunk.subarray(lineStart, newline)
let line = fragment
if (this.fragments.length > 0) {
  if (fragment.length > 0) this.fragments.push(fragment)
  line = Buffer.concat(this.fragments, this.fragmentBytes + fragment.length)
  this.fragments = []
  this.fragmentBytes = 0
}
this.consumeEventLine(line, chunkStart + newline + 1)
lineStart = newline + 1
}
```

每解出一行就推进水位:`this.committedBytes = endByte`(`format.ts:518`)——`committedBytes` 只认完整行的字节边界,`recoverable` 模式下遇到 `turn/end` 的坏行才抛错(`format.ts:511-515`)。
- 写路径在首次追加前执行截断修复:`persistContiguous`(`session-persistence-jsonl/src/storage.ts:319-343`)先 `truncateTornTail`,再把从撕裂帧里抢救出的完整事件 `recoveredTail` 持久重写,然后才写新批次。
- 每个会话一个目录:`<root>/<projectKey(cwd)>/<encodeSegment(id)>/`(`format.ts:253/266`),每个格式世代一个不可变文件(`log.v3.jsonl` 或 `.jsonl.zstd`,`format.ts:57`);历史世代只读不删。

### 3.4 格式迁移链:构建期静态相邻链

`docs/session-format-status.md` 是版本权威:写者版本只由 `SESSION_FORMAT_VERSION`(=3)持有,已发布格式 v3 的证据是 `dsh-v0.1.5-alpha.1`。迁移机制:

- `session-format` 定义纯接口族(`packages/session/session-format/src/types.ts`):`SessionFormatCodec`(物理行编解码,随一个已发布格式冻结)、`SessionFormatMigration`(**相邻** vN→vN+1 转换,`chain.ts:30` 强制 `toVersion === fromVersion + 1`)、`SessionFormatCatalog`(分类 + 流式恢复 + 当前格式编码器)。
- `createSessionFormatChain`(`chain.ts:41`)在构建时校验链**完备且唯一**(v0 到 current 每相邻边恰好一条,缺边/重名即抛),`createStream` 把各边 stage 串成单遍流(`chain.ts:89-127`),header 迁移与 body 迁移分步、inherited cut 逐级传递。相邻性与完备性是两段独立的循环:前者 `if (to !== from + 1) throw new SessionFormatError(\`${migration.name} must declare adjacent v${from}->v${from + 1}\`)`(`chain.ts:30-32`),后者逐版本查表 `Session migration v${version}->v${version + 1} is missing`(`chain.ts:65-71`)。
- 目录是生成代码(`packages/session/session-format-catalog/src/generated.ts:14`):codecs = [v0, v1, v2, v3],migrations = [v0→v1, v1→v2, v2→v3],当前编码器是 v3 codec;`restoreCurrent` 先按 v3 已发布校验恢复,再过**已安装 Session 包**的完整校验(`current.ts:37` 直接 `Session.fromRestore` 空跑一遍)。
- JSONL 后端打开历史文件时按版本分发:低于当前版本走 `requireStoredLog` 里的迁移准备(`session-persistence-jsonl/src/index.ts:495-519`,按 `(sourcePath, revision)` 记忆化,一次解码/迁移操作可 join),写打开时迁移结果**发布为新的当前世代文件**后再接管;高于当前版本或未知必需词汇一律 fail-closed 拒绝(`storage-contract.ts:46-53/69-104`)。

---

## 第四节 Compaction:压缩的触发与执行

### 4.1 总览:日志不动,重写 surface

压缩由能力缝 `ctx.compaction`(`packages/compaction/compaction/src/index.ts:96` 的 `CompactionEngine`)定义,三个入口:

- `compactIfNeeded(agent, trigger, signal)` — 自动策略,`trigger` 为 `'pressure'` 或 `'context-overflow'`;
- `compactNow(agent, signal, sourceCommandId?)` — 人工 `/compact`,经 `agent.runMaintenance` 在空闲 agent 上串行化;
- `compactRegion(start, end, agent, signal?)` — 强制压缩一个 surface 位置区间(两端必须 tool-pairing 平衡)。

一次成功压缩的**日志足迹**是四个事件(`compaction/src/types.ts:17-90` 合并进 `SessionEventMap`):

```text
compaction/start  { compactionId, turn }        # log-only; 分布式锁,持有到 compaction/end
compaction/summary{ summary, shadowedRange, shadowedSeqs, shadowedTokenCount, provider, model, ... }
user/message      surfaceOp: { op:'replace', startSeq, endSeq }   # 真正的 surface 替换
                  sourceEventSeqs: [start.seq, summary.seq, ...shadowedSeqs]
compaction/end    { compactionId, turn, error? }                  # 释放锁; error 记录失败尝试
```

真实源码里的声明(`types.ts:19-35` 与 `:68-89`),`compaction/summary` 的注释直接把"相邻的 `user/message` 才是替换者"写成契约:

```typescript
// packages/compaction/compaction/src/types.ts:19-33
/**
 * Marks the start of a compaction — log-only, holds the lock until
 * `compaction/end`. A numbered owner is strictly enclosed by that open turn;
 * `null` identifies a standalone manual transaction between turns.
 */
'compaction/start': { compactionId: CompactionId; sourceCommandId?: CommandId; turn: number | null }
/**
 * Completed summary, its inputs, and its model call facts — log-only, no surfaceOp.
 * The summary content is in `data.summary`; the actual surface replacement
 * is performed by the immediately following `user/message` event that
 * shadows the compacted range. That adjacency is contractual — the
 * shadowed pricing fields are the replacement's shadow price, so a
 * consumer may pair a replacement with the metering event directly
 * before it (`compaction/prune` documents the shared protocol).
 */
```

`compaction/end` 释放同名锁、`error` 记录失败尝试;`compaction/prune` 则是纯裁剪路径的影子价格事件,注释把"替换事件必须紧跟其后同步追加"写成硬契约(`types.ts:68-89`)。那份"紧随其后的 `user/message`"在源码里就是下面这一句(`compaction-basic/src/region.ts:491-494`,逐字):`session.append('user/message', checkpointMessage, { surfaceOp: { op: 'replace', startSeq: start, endSeq: end }, sourceEventSeqs: [startEvent.seq, summaryEvent.seq, ...shadowedSeqs] })`——`sourceEventSeqs` 的前两项是锁与摘要事件,其后是被遮蔽的全部 surface 节点。
被遮蔽的历史**仍在日志里**(transcript 用 append-origin 事件还原,`surface.ts:60` 的 `isAppendSurfaceEvent`),只是不再投影给模型。摘要经 `frameSummary` 包成 `<compacted-summary>` 检查点(`compaction-basic/src/summarizer.ts:186`),其消息来源用 `compactCheckpointSource(compactionId)`(`compaction/src/checkpoint.ts:33`)标记,消费端凭 `{ kind:'plugin', plugin:'compact' }` 识别。

### 4.2 触发:压力与溢出

`BasicCompactionEngine`(`packages/compaction/compaction-basic/src/index.ts:104`)默认 `auto: true`(`config.ts:95`),注册两条自动路径(`_registerAutomaticCompaction`,`index.ts:138`):

1. **步间压力**:`ctx.on('agent/pre-step')` 里以 `'pressure'` 调 `compactIfNeeded`。失败只警告并放行回合(`index.ts:162`),只有 `TargetPressureConfigError`(路由模型未配 contextWindow)按目标去重警告。
2. **上下文溢出恢复**:`ctx.on('agent/request-error')` 拦截 `CONTEXT_WINDOW_EXCEEDED`,压缩成功(surface `replaceGeneration` 前进)则返回 `{ kind: 'retry' }` 让循环重试同一请求(`index.ts:180-224`);每 agent 最多 `maxOverflowRetries`(默认 1,`config.ts:93`)次,一次成功响应即重置。

压力判定(`compactIfNeeded`,`index.ts:259-333`):用 `ctx.tokenMeter.measure(session)` 对**最新一次已持久化路由请求**的信封估价;阈值 = `contextWindow × thresholdRatio`(默认 0.8,`config.ts:20`);超阈值后先做**无模型裁剪**(若挂载 `toolResultPruner`),重新估价仍超阈值才选区摘要;摘要后仍超阈值按 `compactionRetries`(默认 1)重试,耗尽即抛错。溢出路径绕过阈值与保留尾策略,直接强制一次有用的平衡缩减。配置支持按精确 `provider/model` 的 `modelPolicies` 覆盖(`config.ts:105` 的 `resolveTargetPolicy`)。

### 4.3 选区:保留近期尾部 + 工具配对平衡

`selectCompactableRange`(`compaction-basic/src/region.ts:117`)的规则:

- surface 节点 0 的 `system/message` 永远不在区间内(`region.ts:131`);
- 从尾部向前累计 token,直到达到 `retainTokens`(默认 `contextWindow × 0.16`,`config.ts:23/145`)——这段**逐字保留**;
- 切割点必须 tool-pairing 平衡:`toolPairingBalancedBefore/After`(`compaction/src/tool-pairing.ts:112/124`)按当前 surface 顺序折叠 `assistant/message` 的 tool-call 块(+1)与 `tool/result`(−1),缓存按 `replaceGeneration` 失效——保证不切散"助手调用 ↔ 工具结果"对。

### 4.4 执行:一段同步包围异步摘要的事务

`compactSurfaceRegion`(`region.ts:173`)是共享事务主体:

```text
validateSurfaceRegion(start,end)            # 只读校验:位置存在、顺序、两端平衡
inspectCompactionEntryState(session)        # 倒扫日志: 打开turn/未配对compaction-start/最新end-seed
assertCompactionInactive(...)               # 已有活跃锁 → ManualCompactionError('busy')
                                            #   除非更新的 session/end-seed 证明锁属于已结束生命周期
session.append('compaction/start', ...)     # ← 锁在此落日志; 此后才让出异步
prepared = prepareCompaction(...)           # 计价快照 + 摘要输入(系统头+工具+区间消息)
summarized = await summarize(...)           # 唯一异步段
assertStable(...)                           # whole-surface 或 selected-span 稳定检查,变则 SurfaceChangedError
commitCompactionBody(...)                   # 同步连写 summary + replace user/message + end
options.flush?.()                           # 人工路径的持久化检查点
```

关键细节:

- **摘要输入复用 KV cache**:`buildSummarizationInput`(`region.ts:529`)重建最后一次路由请求的可缓存前缀——surface 节点 0 的系统提示 + header 工具 + 区间派生消息;摘要指令作为**最后一条 user 消息**追加(`summarizer.ts:31-66` 的 `COMPACTION_INSTRUCTION`),因此这次辅助调用是会话的真实前缀。
- **摘要必须更小**:`framedSummaryTokenCount >= shadowedRouteTokenCount` 即拒(`region.ts:403`)——"替代是否降低下一次请求压力"用路由价格回答。
- **锁的崩溃语义**:`compaction/start` 与状态检查同步相邻,摘要期间的任何失败恰好尝试一次带 `error` 的 `compaction/end`;连关闭都失败则故意留下未配对 start——但重启后,位于其后更新的 `session/end-seed` 证明它属于已结束生命周期,锁自动失效(`region.ts:307-319`)。这正是 `session/end-seed` 文档里"standalone bracket 读者"的用途(`types.ts:396-398`)。
- 无模型裁剪器 `ToolResultPruner.pruneSession`(`compaction-tool-result-pruner/src/index.ts:136`)对超长 `tool/result` 做头/尾保留+中间替换(Unicode code point 级),每个替换前同步相邻写一条 `compaction/prune` 影子价格事件,使纯消费者无需保留逐节点价格即可扣减。

### 4.5 人工入口

`/compact`(`packages/compaction/command-compact/src/index.ts:85` 注册)调 `compactNow`,把六类预期失败译成人类可读结果:`busy / cancelled / changed / summary / commit / persistence`(`index.ts:24-56`)。人工压缩用 `owner: null` 的独立 bracket(turn 外),并要求完成后经 `ctx.sessions.flush(agent.session)` 过持久化检查点(`compaction-basic/src/index.ts:396-399`)再放行排队输入。

---

## 第五节 多层级存储:内存投影 → 持久化 → 派生读模型

### 5.1 第 0 层:内存即真相的工作集

活会话的全部工作状态都在 `Session` 实例上:`log`(数组)、`SurfaceManager`(增量 surface)、`deriveMessages` 缓存、`requestHeader/requestContext` 增量折叠、以及 projection registry 的 per-session cell。**没有任何一层内存状态是不可从日志重建的**——这是分层成立的前提。

### 5.2 第 1 层:持久化能力缝 `ctx.sessionPersistence`

`SessionPersistence`(`packages/session/session-persistence/src/index.ts:135`)是抽象 Service:`create(header)` / `open(id, 'read'|'write')` / `flush()` / `stat` / `list`。统一语义(`index.ts:115-134`):事件从 seq 0 连续、从不改写;`append` best-effort,**`flush` 才是承诺崩溃存活的唯一操作**;`write` 打开原子抢占单写者所有权。

句柄契约 `SessionHandle`(`handle.ts:59`):`read` 句柄永不回退到已见前缀之前;`write` 句柄读自己的已成功追加;`close()` 是唯一拆除(幂等、不可取消)。

**JSONL 后端**(`session-persistence-jsonl`)的运行要点:

- **惰性物化**:`create` 只登记 `pending`(`storage.ts:414` 的 `registerCreated`),本进程立即可见;首个 append/flush 才落盘(`index.ts:308-327`)。崩溃在物化前 = 会话从未存在。
- **活事件路由**:后端 `install`(`storage.ts:534-566`)订阅 `session/event`,按 session id 路由进该会话的活跃写句柄 `enqueueLive`(200ms 有界批窗口,`storage.ts:36/274`);`session/flush` 触发 `drainLive + flush`;`session/disposed` 触发最终 drain 后 close;插件卸载 effect 关闭所有句柄。
- **单写者**:进程内 `JsonlBackendTracker.writers`(`storage.ts:395`)加跨进程内核锁(`SessionWriteLease`,写打开时获取,create 句柄在首次物化写前获取,`storage.ts:352`)。
- **每句柄串行化**:所有变更排队在 per-handle promise chain 上(`storage.ts:86` 的 `chain`),追加批次先 `materializeAppendBatch` 深快照再 `assertContiguous`(`storage-contract.ts:145`)。

### 5.3 持久化时机:checkpoint 策略插件

追加是 best-effort,**何时 flush 由策略插件决定**。`session-checkpoint-policy`(`packages/session/session-checkpoint-policy/src/index.ts:63`)挂三个语义检查点,全部 fail-closed(检查点失败则模型适配器/工具体不执行):

1. `llm/stream` 瀑布:构造下游流之前 `await ctx.sessions.flush(session)`——**已记录的请求前缀先持久化,再发请求**(`index.ts:29-38`);
2. `tools/execute`:顶层工具调用在派发前 flush 其已记录调用(`index.ts:70-75`);
3. `agent/pre-step`:每次请求边界 flush 上一步的响应/结果批次(`index.ts:79-82`)。

goal 轮驱动在自己的空转检查点也调 `sessions.flush`(`packages/goal/goal-round-driver/src/index.ts:145`)。`SessionStore.flush`(`packages/core/session/src/index.ts:1144`)是统一入口:并行广播 `session/flush`,全部监听器结算后才返回,首个失败在全部结算后抛出。

### 5.4 第 2 层:通用 KV 存储(storage hub → domain)

`packages/storage/` 是与会话正交的底层 KV 栈:

- `storage` 定义 `StorageBackend`/`KvFacet`/`KvUnit`(`storage/src/backend.ts:17/85`):后端拥有一个介质,unit 是"整快照 + 逐记录持久写",单次调用在介质上原子、resolve 即持久;`layout: 'single' | 'per-record'`,带 `version` 戳与 `compatibleVersions`。
- 后端实现:`storage-json`(每 unit 一个 JSON 文档或 per-record 目录树,原子重写,`storage-json/src/index.ts:39`)与 `storage-sqlite`(一个库文件 document-per-row,`units` 表校验版本戳,`storage-sqlite/src/index.ts:55`)。
- `storage-domain`(`storage-domain/src/index.ts:69` 的 `DomainFacility`)在其上提供 zod 校验的领域:`open(spec)` 按配置路由后端、加载全量并逐记录过 schema;`invalidRecords: 'backup-and-skip'` 的坏记录被挪为 `.bak` 并跳过(派生数据可丢,不许卡启动)。

### 5.5 第 3 层:派生读模型(全部可丢弃重建)

- **投影持久缓存** `session-projection-cache`:把 `sessionProjections.checkpoint(session)` 的 `(key → {ver, seq, val})` 行按会话写入 `session_projcache` 域(`session-projection-cache/src/spec.ts:98`,per-record 布局,version 7 + compatibleVersions [3,4,5,6])。写入纪律(`index.ts:246-262`):先取切面、**再 flush 会话日志**、最后落缓存——崩溃只可能让缓存落后于日志(更长尾部重放),绝不可能超前(幽灵值)。读取阶梯:`viewCheckpoint`(零 I/O)→ `restoreFloor` 定位尾部读 → `restore`(缓存状态 + 前向重放 + view;行 `ver` 不匹配或水位越界即丢弃重折叠,`session-projection/src/index.ts:495`)。写触发:turn/end、计数/间隔节流、create、dispose 四个强制点(`session-projection-cache/src/index.ts:301-339`)。
- **全文查询** `session-query-sqlite`:SQLite FTS5 库(schema v8,`session-query-sqlite/src/schema.ts:8`),`persisted_sessions` + `persisted_docs`(持久)与 `temp.live_sessions` + `temp.live_docs`(连接期临时)**双源**。`_reconcile`(`session-query-sqlite/src/index.ts:407`)对比 `sessionPersistence.list()` 的 revision 与 `ctx.sessions` 的活会话指纹,把变化整会话替换进索引(BEGIN IMMEDIATE 事务);冷读走 `readColdSessionLog`(含内存版崩溃修复,不回写),活会话优先。它是**派生索引**:版本不符即整库重置(`schema.ts:96-101` 的 `resetDerivedSchema`)。
- `session-query`(`session-query/src/index.ts:97` 的 `SessionQueryEngine`)提供后端无关的精确读/过滤/追溯,SQLite 后端补全文检索;`tool-session-query` 把它暴露为模型工具。

---

## 第六节 与 agent-loop 的衔接:每步 deriveMessages 重建模型输入

### 6.1 每步重建的完整序列

`ReactLoopAgent`(`packages/core/agent-loop/src/agent.ts:72`)的一个 step(`agent.ts:352`)按序做:

```text
preStep: 认领 inbox 输入 → systemPrompt.assemble → agent/pre-step 瀑布(compaction/checkpoint 在此挂)
prepareRequest: 以 session.requestHeader() 折叠值为种子 → agent/request 瀑布 → llm.prepareCall 绑定适配器
buildRequest(agent.ts:553):
  1. canonicalHeader(config, adapterDefaults, tools)
  2. 与 session.requestHeader() 基线 headerEquals → 按需 append('request/header', reason)
  3. 路由/容量/系统提示模式变化 → append('request/context')
  4. boundaryMessages = session.deriveMessages()        # ← 模型输入唯一来源
  5. 深冻结新出现的 Message 并 Object.freeze 请求体
systemPrompt.project + append('system/message'/'user/message')  # 先于 buildRequest 的第 4 步
preparedCall.stream(request)                          # checkpoint-policy 在适配器派发前 flush
stream 完结 → append('assistant/message' { message, usage, stream })  # 或失败路径 append('assistant/attempt')
工具块非空 → executeToolCalls → 每个结果 append('tool/result')(tool-calls.ts:282)
finally: append('step/end');turn 收尾 append('turn/end', { reason })
```

关键不变式的落点:`buildRequest` 的 `messages` 就是 `session.deriveMessages()` 的返回值(`agent.ts:603`),同一份深冻结消息对象被请求、日志、UI 共享;中途取消的流把已交付前缀落成 `assistant/message { interrupted: true }`(`agent.ts:405-419`);未产出可见内容的尝试落成 `assistant/attempt`(`agent.ts:421-430`)——**失败也落日志,但不污染 surface**。

### 6.2 surface 重写与请求系列

循环跟踪 `requestSurfaceGeneration`(`agent.ts:87/561-582`):surface 发生 replace(compaction 落地)后,下一个请求即使用**重写后的**派生历史,并在头未变时仍记一条 `request/header { reason: 'series' }` 显式开新消息系列——KV cache 边界在日志里有据可查。`session-checkpoint-policy` 的 `llm/stream` 拦截(见 5.3)保证:**任何一次模型请求的完整输入在该请求发出前已持久化**——崩溃后 resume 重放到的一定是模型真实见过的历史。

### 6.3 生命周期衔接

创建与 resume 共享一套事务(`agent-loop/src/index.ts`):`sessions.prepare` 构造(带 seed 即重放)→ `createStoredSession` 抢占持久写句柄 → setup → `appendUnstoredSuffix`(`index.ts:749`)把发布前窗口的事件(构造 seed 标记、delegation 策略记录等不经 `session/event` 的追加)先 flush 进句柄 → `publish` 依次 `sessions.enter → agents.enter → sessions.announce → agents.announce`(`index.ts:662-678`)。从此持久化后端的活事件路由接管该会话的每一条新事件(见 5.2)。fork 则是 `SessionStore.fork`(`index.ts:1203`):选一个不在打开 turn 内的边界 seq,拷贝前缀为子会话 seed,header 记 `parentSession` 与 `isSeeded: true`,构造器在切口处追加 `{ inherited: true }` 的 `session/end-seed`。

---

## 第七节 关键文件索引表

| 文件 | 角色 | 关键符号(行号) |
|---|---|---|
| `packages/core/session/src/types.ts` | 事件词汇、信封、header、版本常量 | `SESSION_FORMAT_VERSION`(88)、`SessionHeader`(93)、`TurnEndReasonMap`(200)、`EpochHeader`(232)、`SessionEventMap`(269)、`SurfaceOp`(434)、`SessionEvent`(465)、`ignorable`(483) |
| `packages/core/session/src/index.ts` | Session 类与内存 store | `Session`(446)、`append`(710)、`deriveMessages`(832)、`requestHeader`(776)、`fork`(1203)、`SessionStore.flush`(1144) |
| `packages/core/session/src/surface.ts` | surface 折叠与逐节点投影 | `deriveEventMessage`(92)、`SurfaceManager`(504)、`foldSurface`(487)、`assertProvenance`(269)、`assertSystemHeadRewrite`(404) |
| `packages/core/session/src/invariant.ts` | 关系不变式伴随插件 | `validateEvent`(55)、安装器(193) |
| `packages/core/session/src/repair.ts` | 崩溃尾部语义修复 | `interruptedTurnClosers`(29)、`TOOL_NOT_STARTED`(15)、`TOOL_OUTCOME_UNKNOWN`(18) |
| `packages/core/session/src/request-header.ts` | 请求头折叠/比较 | `foldRequestHeader`(63)、`headerEquals`(43)、`canonicalHeader`(21) |
| `packages/core/session/src/seq-ranges.ts` | sourceEventSeqs 区间编码 | `encodeSeqRanges`(18)、`decodeSeqRanges`(37) |
| `packages/core/session/src/known-event-types.ts` | 生成的事件词汇表 | `KNOWN_SESSION_EVENT_TYPES`(22) |
| `packages/core/session/src/preparation.ts` | 未发布 Session 的所有权包装 | `SessionPreparation`(20) |
| `packages/core/agent-loop/src/agent.ts` | 循环驱动,每步重建请求 | `turn`(269)、`step`(352)、`buildRequest`(553)、`deriveMessages` 调用(603) |
| `packages/core/agent-loop/src/index.ts` | 创建/resume/fork 事务 | `create`(699)、`resume`(844)、`resumeWith` 崩溃修复(892)、`appendUnstoredSuffix`(749) |
| `packages/session/session-persistence/src/index.ts` | 持久化 Service Definition | `SessionPersistence`(135) |
| `packages/session/session-persistence/src/handle.ts` | 句柄契约 | `SessionHandle`(59) |
| `packages/session/session-persistence/src/storage-contract.ts` | 后端共享校验 | `validateStoredEvents`(69)、`assertVersion`(46)、`assertContiguous`(145) |
| `packages/session/session-persistence-jsonl/src/format.ts` | JSONL 物理格式 | `SessionLogScanner`(385)、`scanLog`(530)、`encodeSegment`(198)、`projectDir`(253) |
| `packages/session/session-persistence-jsonl/src/index.ts` | JSONL 后端服务 | `JsonlSessionPersistence`(235)、`create`(308)、`open`(336)、迁移准备(495) |
| `packages/session/session-persistence-jsonl/src/storage.ts` | 句柄/路由/写后缓冲 | `JsonlSessionHandle`(85)、`enqueueLive`(274)、`persistContiguous`(319)、`JsonlBackendTracker.install`(534) |
| `packages/session/session-checkpoint-policy/src/index.ts` | 语义持久化检查点 | `apply`(63) |
| `packages/session/session-format/src/types.ts` | 格式/迁移接口族 | `SessionFormatMigration`(45)、`SessionFormatChain`(78)、`SessionFormatCatalog`(210) |
| `packages/session/session-format/src/chain.ts` | 相邻链编译 | `createSessionFormatChain`(41) |
| `packages/session/session-format-catalog/src/generated.ts` | 生成的 codec/迁移目录 | `sessionFormatCatalog`(14) |
| `packages/session/session-format-v0-to-v1` / `-v1-to-v2` / `-v2-to-v3` | 三个相邻迁移包(含冻结 codec) | 各 `codec.ts` / `migration.ts` |
| `packages/session/session-projection/src/index.ts` | 领域投影注册表 | `ProjectionDefinition`(48)、`SessionProjectionRegistry`(199)、`restore`(495)、`checkpoint`(396) |
| `packages/session/session-projection-cache/src/spec.ts` / `index.ts` | 投影持久缓存域与写纪律 | `projectionCacheDomainSpec`(98)、`write`(246)、`coldSnapshot`(277) |
| `packages/session/session-title/src/index.ts` | 标题服务与 `session/title` 事件 | 事件合并(71)、`SessionTitleService` |
| `packages/session/session-stats/src/projection.ts` | `sessionStats` 单元 | `sessionStatsProjectionDefinition`(113) |
| `packages/session/session-turn-outline/src/projection.ts` | `turnOutline` 单元 | `turnOutlineProjectionDefinition`(85) |
| `packages/session/session-telemetry/src/coordinator.ts` | 遥测采集协调器 | `SessionTelemetryCoordinator`(75) |
| `packages/session/session-log-deepseek/src/index.ts` | 日志增量投递 DeepSeek API | `acceptedThrough`(117) |
| `packages/session-query/session-query/src/index.ts` / `cold-read.ts` | 查询 Service Definition 与冷读 | `SessionQueryEngine`(97)、`readColdSessionLog`(31) |
| `packages/session-query/session-query-sqlite/src/schema.ts` / `index.ts` | FTS5 派生索引 | `openSearchDatabase`(46)、`_reconcile`(407) |
| `packages/compaction/compaction/src/index.ts` / `types.ts` / `tool-pairing.ts` / `checkpoint.ts` | 压缩能力缝 | `CompactionEngine`(96)、`compaction/*` 事件合并(types.ts:17)、`toolPairingBalancedBefore`(112)、`compactCheckpointSource`(33) |
| `packages/compaction/compaction-basic/src/index.ts` / `region.ts` / `summarizer.ts` / `config.ts` | 基本压缩后端 | `_registerAutomaticCompaction`(138)、`compactIfNeeded`(259)、`compactSurfaceRegion`(region.ts:173)、`selectCompactableRange`(region.ts:117)、`COMPACTION_INSTRUCTION`(summarizer.ts:31) |
| `packages/compaction/compaction-tool-result-pruner/src/index.ts` | 无模型工具结果裁剪 | `ToolResultPruner.pruneSession`(136) |
| `packages/compaction/command-compact/src/index.ts` | `/compact` 命令 | `executeCompact`(59) |
| `packages/storage/storage/src/backend.ts` | KV 后端契约 | `StorageBackend`(17)、`KvUnit`(85) |
| `packages/storage/storage-domain/src/index.ts` | 领域数据层 | `DomainFacility.open`(103) |
| `packages/storage/storage-json/src/index.ts` / `storage-sqlite/src/index.ts` | json / sqlite 后端 | `JsonStorageBackend`(39)、`SqliteStorageBackend`(55) |
| `docs/session-format-status.md` | 版本/发布权威 | release record(`latestReleasedVersion: 3`) |
| `packages/goal/goal/src/index.ts` / `packages/todo/tool-todo/src/index.ts` | 带持久状态的包参照 | `goal/change` 追加(613)、`todo/write` 追加(210) |