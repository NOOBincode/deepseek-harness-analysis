# 06 · jobs:后台执行、完成通知与回收(函数级走查)

> 源码:[`packages/jobs/jobs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/index.ts)(179 行,Service Definition)、[`types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts)(160 行)、[`brand.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/brand.ts)
> 实现:[`packages/jobs/jobs-local/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts)(534 行,进程内注册表)
> 消费者:[`packages/jobs/tool-jobs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts)(401 行,`job_output`/`job_list`/`job_kill`)
> 后台 subagent 的产生方:[`packages/subagent/tool-subagent/src/index.ts:544-560`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/subagent/tool-subagent/src/index.ts#L544-L560)
> 对应[第十章第 6 节](../10-multi-agent.md);子 agent 本身见 [02](./02-subagent-seam-and-providers.md)/[04](./04-continuation-and-control.md)。

---

## 第〇节 一句话结论

jobs 是**一个外壳,不是一种执行方式**。它只拥有**身份、生命周期状态与通知**;执行资源完全由生产者掌握,生产者只交出两个钩子:`cancel(reason?)`(同步、幂等)与 `done: Promise<JobOutcome>`(资源释放后才 resolve)。因此"后台 subagent"并不是新机制——它就是**一条 `kind:'subagent'` 的 job,其 `run()` 里照常调用 `ctx.subagents.start(...)`**。

```text
调用者(模型)                JobRegistry(seam)              LocalJobRegistry(实现)        生产者 hooks
subagent{run_in_background}  start(spec)  ─────────────▶  servesOwner? → 限额? → spec.run()
                              ◀─ JobId 'subagent-N' ─────  store.set + done.then(settle)
job_output(job_id, wait)     wait(id, ms) ─────────────▶  deadline(signal, ms)
                             read(id)     ─────────────▶  readOutput() 或终态 output;终态读 → reported
job_kill(job_id)             kill(id)     ─────────────▶  cancel(reason) → status='stopping' → reported
完成通知                       onJobDone ────────────────  settle():记录 → 释放 waiter → 通知
```

---

## 第一节 seam 的六条语义(抽象类即是契约)

`JobRegistry` 是抽象类([`jobs/src/index.ts:62-177`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/index.ts#L62-L177)),构造时就拒绝被直接加载:

```typescript
// packages/jobs/jobs/src/index.ts:63-71
constructor(ctx: Context) {
  // `abstract` erases at runtime, so a composition row naming this package
  // would register a ctx.jobs with no method implementations and fail far
  // from the misconfiguration. Fail loud at load instead.
  if (new.target === JobRegistry) {
    throw new Error('@deepseek-ai/dsh-jobs is the abstract job registry seam; load an implementation such as @deepseek-ai/dsh-jobs-local instead')
  }
  super(ctx, 'jobs')
}
```

类 JSDoc(`:41-60`)逐条列出实现必须遵守的语义:

| # | 语义 | 关键点 |
|---|---|---|
| 1 | **注册记录的生命周期长于 producer / controller fiber** | teardown 里 `cancel` 抛错只强制失败**那一条记录**;teardown 取消还会把记录标成 `reported`——一个它的 owner 正被销毁的记录没有读者了 |
| 2 | **归属访问按 owner 会话 id 围栏** | *Ids are predictable, so authorization — not secrecy — is the boundary* |
| 3 | **结算 first-wins** | 一份终态记录、释放 waiter、**一轮**包含式监听通知 |
| 4 | **完成通知最后发** | 在记录提交、且其他所有结算观察者都已看到之后才通知,因为 reporter 可能**同步**开一个模型 turn |
| 5 | **`start` 在无人服务该 owner 时拒绝** | 生产者不得启动 owner 收集不到、也停不掉的工作 |
| 6 | **owner 相对而非进程相对** | 一个注册表服务进程内**每个** composition(见第四节) |

第 6 条是理解 `jobs-local` 全部复杂度的钥匙(见第四节)。

六个抽象方法(`:82-176`):

| 方法 | 语义摘要 |
|---|---|
| `start(spec): JobId` | 预检访问 / 校验 / owner 清理 / 实现侧准入,然后启动并原子注册。**预检拒绝不留 id 也不留资源;`run()` 抛错什么都不注册;`run()` 返回后注册不可能失败** |
| `list(caller?): JobSnapshot[]` | 按注册序列出**调用者自己的与无主的**作业,不泄露别人的 label |
| `get(id, caller?)` | 非消费快照,**不改读游标也不改通知状态** |
| `read(id, caller?): JobRead` | 流式作业返回**自上次读以来的增量**;终态读是幂等的最终输出。**终态读标记 `reported`** |
| `kill(id, caller?, reason?)` | **先请求取消,再标记 stopping 与 reported**。生产者抛错则**不改作业状态**。返回 `'requested' \| 'already-finished'` |
| `wait(id, timeoutMs, caller?, signal?)` | 等结算或超时,**不取消作业**。caller abort 只在作业存活时 reject;结算之后终态快照胜出——**这样"为这个 waiter 抑制掉的通知"仍会被送达** |
| `onJobDone(listener)` / `onJobsChanged(listener)` / `attachController(name)` | 三者都是 effect-scoped 注册,返回 disposer |

`wait` 那条契约(`:122-131`)值得单独记住:*after settlement the terminal snapshot wins so a notice suppressed for this waiter is still delivered*。原因是 `settle()` 里有一行 `if (job.waiters > 0) job.reported = true`([`jobs-local/src/index.ts:422`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L422))——**有 waiter 在场时,终态通知会被抑制**(因为 waiter 自己会看到结果)。若这个 wait 后来因为 caller abort 而 reject,通知就再也没人发了;所以"已结算则返回终态快照"必须优先于 abort。

---

## 第二节 本地实现:`start` 的三道闸 + 四步注册

```typescript
// packages/jobs/jobs-local/src/index.ts:131-153(节选)
start(spec: JobStart): JobId {
  if (!this.servesOwner(spec.owner)) {
    throw new Error('background jobs unavailable: no job controller serves this agent (load @deepseek-ai/dsh-tool-jobs in its composition)')
  }
  if (spec.kind.length === 0) throw new Error('invalid job kind: expected a non-empty string')
  if (spec.label.length === 0) throw new Error('invalid job label: expected a non-empty string')
  if (spec.outputLimitBytes !== undefined
    && (!Number.isSafeInteger(spec.outputLimitBytes) || spec.outputLimitBytes <= 0)) {
    throw new Error(`invalid outputLimitBytes: expected a positive safe integer, got ${JSON.stringify(spec.outputLimitBytes)}`)
  }
  if (spec.owner !== undefined) this.ensureOwnerCleanup(spec.owner)
  const active = this.activeTaskCount(spec.owner)
  if (active >= this.maxConcurrentJobsPerOwner) {
    throw new Error(`background job limit reached for this owner (limit: ${this.maxConcurrentJobsPerOwner}); use job_kill to stop an unneeded job, wait for it to finish, then retry`)
  }
  const hooks = spec.run()          // ← 分界线:之前的 throw 不留痕迹;之后只有 done 能改状态
  const count = (this.counters.get(spec.kind) ?? 0) + 1
  this.counters.set(spec.kind, count)
  const id = JobId(`${spec.kind}-${count}`)
```

顺序上的三处讲究:

1. **`servesOwner` 在最前**(见第四节),因为它是"这个部署根本不该有后台作业"的配置错误,比参数校验更根本。
2. **`ensureOwnerCleanup` 在校验之后、限额之前**(`:141`):它给 owner 的 scope 挂一个 awaited cleanup effect。它自己也会失败——`agents.get(ownerId) !== owner` 时抛 *background job owner must be live*(`:454-456`),防止一个已被替换的实例持有作业。
3. **`spec.run()` 是唯一的分界线**。它之前的每个 `throw` 都不留痕迹;它之后只有 `hooks.done` 能改状态。注释:`Registration is complete and cannot fail from here, so the visible set has genuinely changed`(`:186-188`),然后才 `notifyChanged`。

`counters` 是按 **kind** 计数的(`:151-153`),所以 id 形如 `subagent-1`、`subagent-2`——同一 kind 的编号在进程内单调,进程重启后从 1 开始。这个 id **可预测**正是第一条语义成立的前提:围栏必须是授权而不是保密。

本地实现是 `LocalJobRegistry`([`jobs-local/src/index.ts:91-533`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L91-L533)),其 `Config` 只有一项:`maxConcurrentJobsPerOwner`(默认 `DEFAULT_MAX_CONCURRENT_TASKS_PER_OWNER = 10`,[`:28`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L28),[`:92-98`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L92-L98))。`activeTaskCount`([`:322-328`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L322-L328))按**精确 owner 对象**统计(`job.owner === owner`)且只算 `running`/`stopping`。

### 2.1 `done` 的两条来路

```typescript
// packages/jobs/jobs-local/src/index.ts:178-185
void hooks.done.then(
  (outcome) => { this.settle(job, outcome) },
  (error: unknown) => {
    // Contain a producer contract violation (`done` rejected) so cleanup and waiters cannot hang.
    this.selfCtx.logger.warn(`jobs: job ${job.id} producer done promise rejected (producer contract violation): ${String(error)}`)
    this.settle(job, { status: 'failed', detail: String(error) })
  },
)
```

生产者契约要求 `done` **不得 reject**([`types.ts:78-84`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L78-L84)),但实现仍然兜底成 `failed`——否则 cleanup 与 waiter 会永久挂住。

---

## 第三节 `job_output`:wait 与 timeout 的语义

```typescript
// packages/jobs/tool-jobs/src/index.ts:329-337(节选)
async execute(args, exec) {
  const id = validateJobId(args.job_id)
  if (args.wait === true) {
    const timeout = Math.min(args.timeout_ms ?? waitDefault, waitCap)
    await ctx.jobs.wait(id, timeout, exec.agent, exec.signal)
  }
  const read = ctx.jobs.read(id, exec.agent)
  return { text: read.text, job: publicJob(read.snapshot) }
}
```

四个可配置量(`:205-208`,默认值在 `Config`,`:47-52`):

| 配置 | 默认 | 作用 |
|---|---|---|
| `waitTimeoutMs` | 30 000 | 只给 `wait: true` 不给 `timeout_ms` 时的时长 |
| `maxWaitTimeoutMs` | 600 000 | 单次 wait 的硬上限;模型给的更大值**被夹到它** |
| `completionDelivery` | `'wakeup'` | 完成是否给 idle owner 开一个 turn |
| `maxConsecutiveWakes` | 3 | 连续唤醒预算(见第五节) |

工具描述把失败语义写进了模型可见文案(`:303-311`):*A timed-out wait returns `[status: running]` and leaves the job alive*。这也是这里**不用 `ToolDefinition.timeoutMs`** 的原因(注释 `:306-307`):一个超时的 wait 应当返回作业状态,而不是 `TOOL_TIMEOUT` 工具错误。**工具自己拥有它的截止时间。**

### 3.1 `wait` 的实现:distinguish timeout from abort

```typescript
// packages/jobs/jobs-local/src/index.ts:236-268(节选)
if (!isTerminal(job.status)) {
  if (signal?.aborted) throw new Error('wait aborted')
  // Abort removes the waiter synchronously so same-tick settlement cannot
  // suppress a notice for a wait that will reject.
  job.waiters += 1
  let counted = true
  const uncount = (): void => { if (!counted) return; counted = false; job.waiters -= 1 }
  try {
    // The scoped deadline distinguishes a successful wait timeout from
    // caller cancellation and clears its timer on every exit.
    using d = deadline(signal, timeoutMs, TASK_WAIT_TIMEOUT)
    await new Promise<void>((resolve, reject) => {
      const onSettled = (): void => { job.waitResolvers.delete(onSettled); d.signal.removeEventListener('abort', onAbort); resolve() }
      const onAbort = (): void => {
        job.waitResolvers.delete(onSettled)
        if (timeoutOf(d.signal, TASK_WAIT_TIMEOUT) !== undefined) resolve()
        else { uncount(); reject(new Error('wait aborted')) }
      }
      job.waitResolvers.add(onSettled)
```

三个细节:

1. **同一根 `AbortSignal` 承载两种含义**:超时(`TASK_WAIT_TIMEOUT`,`:25`)与调用者取消。靠 `timeoutOf(d.signal, TASK_WAIT_TIMEOUT)` 区分——超时就 resolve(返回当前快照),取消就 reject。
2. **`waiters` 计数同步增减**(`:238-246`):注释说明理由是"同一 tick 内的结算不得为一个即将 reject 的 wait 抑制掉通知"。
3. `using d = deadline(...)` 保证定时器在**每条出口**都被清理。

### 3.1 `read` 的实现与输出游标

`read` 只有四行逻辑([`jobs-local/src/index.ts:205-213`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L205-L213)):**有 `readOutput()` = 流式作业**,每次 `read` 是**消费式增量**,由一个游标拥有([`types.ts:85-90`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L85-L90):*each job has one consuming cursor*);**无 `readOutput()` = 仅最终输出**,存活时返回空串、结算后返回 `outcome.output`,**幂等、永不消费**;**终态读标记 `reported`**(`:211`)——这就是完成通知不会重复发的机制。

### 3.2 输出体积上限

`outputLimitBytes` 由 `JobStart` 指定、进入快照([`types.ts:51-55`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L51-L55)、[`:101-105`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L101-L105)),消费侧在 `tool-jobs` 里落成两条:`visibleOutputLimit`(`:184-189`)从 `exec.arguments.job_id` 反查该作业的上限(只对 `job_output`/`job_kill` 生效),配合 `tools/pre-execute` 的 `prepend` 监听(`:232-236`)与 `finalizeContent`(`:237-256`),在**完整结果已知处**执行字节上限——符合 *Apply bounds to the complete result* 规则。`job_output` 的渲染还保留了 `output`/`status` 的切分(注释 `:242-243`)。

---

## 第四节 `servesOwner`:owner 相对的可见性

```typescript
// packages/jobs/jobs-local/src/index.ts:315-319
private servesOwner(owner?: Agent): boolean {
  if (!this.layers.global.controllers.isEmpty()) return true
  return this.layers.chainLayers(owner === undefined ? undefined : scopeOf(owner.ctx))
    .some(layer => !layer.controllers.isEmpty())
}
```

`layers` 是 `ScopedLayers<JobLayer>`(`:116`),注释(`:104-115`)解释了为什么需要分层而不是一张平表:*The registry is one process-wide instance serving every composition, so a flat table would answer a per-owner question process-wide: one preset's job controls would hold `start()` open for an agent whose own composition loads none, and one settlement would reach every preset's notice listener. Layers make both reads owner-relative.*

三类读取各自解析 listener/controller 集合:**`servesOwner`**(`:315-319`)——global 层有 controller 则**服务所有 owner**,否则沿 owner 的 scope 链找;**`listenersFor`**(`:338-342`)与 **`changedFor`**(`:388-392`)——global 层的先,再沿 owner 的 scope 链逐层,两者解析规则完全相同。三者都通过 `this.layers.effect(this.ctx, ...)` 注册(`:281-305`),即**贡献落到注册者的 scope 层**——与工具注册表同一形状。

`attachController(name)`(`:297-305`)每次调用生成一个新的 `Symbol(name)` 作 token,注释:*One token per call keeps duplicate labels independently disposable*。全仓只有 `tool-jobs` 调用它([`tool-jobs/src/index.ts:259`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L259)),所以拒绝文案直接点名:`load @deepseek-ai/dsh-tool-jobs in its composition`。

---

## 第五节 完成通知回流:`busy → inject,idle → followup`(带预算)

```typescript
// packages/jobs/tool-jobs/src/index.ts:278-299(节选)
ctx.jobs.onJobDone((snapshot, owner) => {
  if (snapshot.reported || owner === undefined) return
  const message = createUserMessage({
    content: [{ type: 'text', text: fitCompletionNotice(snapshot) }],
    source: { kind: 'plugin', plugin: 'tool-jobs', form: 'notice', summary: completionSummary(snapshot) },
  })
  const spent = spentWakes.get(owner) ?? 0
  if (delivery === 'wakeup' && owner.status === 'idle' && spent < wakeBudget) {
    spentWakes.set(owner, spent + 1)
    owner.followup(message)
    return
  }
  owner.inject(message)
})
```

决策表:

| 条件 | 结果 |
|---|---|
| `snapshot.reported`(已被 read/kill/teardown 标过) | 丢弃,不发 |
| `owner === undefined`(无主作业) | 丢弃 |
| `delivery === 'wakeup'` 且 owner `idle` 且预算未耗尽 | `owner.followup(message)` —— **开一个新 turn**,预算 +1 |
| 其余(quiet / busy / 预算耗尽) | `owner.inject(message)` —— 排到**下一个 step 边界**,不唤醒 |

为什么 busy 用 `inject` 而不是排队:**它占着 turn 的下一个 step inbox,而 turn 无法越过它关闭**——所以多个作业同时结算只花一个 step(`:268-273` 的注释)。为什么 idle 用 `followup`:**未认领的通知等于模型永远不知道这个完成**。

### 5.1 唤醒预算:切断自激链

```typescript
// packages/jobs/tool-jobs/src/index.ts:210-229(节选)
const spentWakes = new WeakMap<Agent, number>()          // keyed by the exact Agent
if (waitDefault > waitCap) {
  throw new Error(`tool-jobs: waitTimeoutMs (${waitDefault}) exceeds maxWaitTimeoutMs (${waitCap})`)
}
// A budget is a count of turns. `Infinity` would leave the runaway chain this
// field exists to bound unbounded, and a fraction never names a turn at all.
if (!Number.isSafeInteger(wakeBudget)) {
  throw new Error(`tool-jobs: maxConsecutiveWakes (${wakeBudget}) must be a whole number of turns`)
}
// Nothing spends the budget under quiet delivery, so nothing needs to refill it.
if (delivery === 'wakeup') {
  ctx.on('agent/inbox/claimed', ({ agent, message }) => {
    // Claiming is the point the human's input actually enters a step; a notice
    // this plugin itself queued must not refill the budget it just spent.
    if (message.source.kind === 'user') spentWakes.delete(agent)
  })
}
```

- **预算只由 `kind === 'user'` 的输入认领重置**(`:224-228`)。这条过滤是必须的:否则本插件自己 queued 的通知被认领时会**把刚花掉的预算还回来**,自激链就封不住了。
- `WeakMap<Agent, number>` 键是**精确 Agent 实例**,所以同 id 的替换实例从满预算开始(`:211-213`)。
- 配置期就 fail-loud:默认 wait > 上限、预算不是非负整数,都在 `apply()` 里直接抛(`:214-221`)。

### 5.2 `settle()` 的顺序:提交 → 释放 waiter → 通知

```typescript
// packages/jobs/jobs-local/src/index.ts:416-440(节选)
private settle(job: TrackedTask, outcome: JobOutcome): void {
  if (isTerminal(job.status)) return
  job.status = outcome.status
  job.detail = outcome.detail
  job.output = outcome.output
  job.finishedAt = Date.now()
  if (job.waiters > 0) job.reported = true
  const snapshot = this.snapshot(job)
  const waitResolvers = [...job.waitResolvers]
  job.waitResolvers.clear()
  for (const resolveWait of waitResolvers) resolveWait()
  job.markSettled()
  this.notifyChanged(job.owner)
  if (this.listenersClosed) return
  for (const listener of this.listenersFor(job.owner)) { /* try/catch + promise 观察 */ }
}
```

顺序正是 seam 的第 3、4 条语义:**first-wins**(`isTerminal` 守卫)→ 提交记录 → 释放 waiter → `markSettled()` → 变更通知 → **最后**才发完成通知。监听者的同步 throw 与 promise reject 都被包含(前者 warn,后者观察但不等)。

---

## 第六节 `job_kill` 的终止保证

```typescript
// packages/jobs/jobs-local/src/index.ts:215-228
kill(id: JobId, caller?: Agent, reason?: string): 'requested' | 'already-finished' {
  const job = this.expect(id)
  this.assertAccess(job, caller)
  if (isTerminal(job.status)) {
    job.reported = true
    return 'already-finished'
  }
  // Cancel first so a throw leaves both lifecycle and notice state unchanged.
  job.cancel(reason)
  job.status = 'stopping'
  job.reported = true
  this.notifyChanged(job.owner)
  return 'requested'
}
```

**终止保证的准确表述**:

| 保证 | 依据 |
|---|---|
| `kill` 返回 `'requested'` 时,生产者的 `cancel(reason)` **已被同步调用** | `:223` 在改状态之前 |
| `cancel` 抛错时,**作业状态与通知状态都不变** | 同上,注释 *Cancel first so a throw leaves both lifecycle and notice state unchanged* |
| 状态立即变为 `stopping`,且**标记 `reported`** | `:224-225` —— 所以不会再有完成通知 |
| 作业**尚未**停止 | 工具描述原文:*Returns immediately; the job settles as killed once its work actually stops*([`tool-jobs/src/index.ts:363`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L363)) |
| 真正的终态是 `done` resolve 出的 `JobOutcome.status`(`killed`/`failed`/`completed`) | `settle()`(`:416-419`) |
| service / owner 销毁时会**等** `job.settled` | `disposeOwned`/`disposeAll`(`:467-475`/`481-...`) |

`cancel` 的契约([`types.ts:73-77`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L73-L77)):*Must be synchronous, idempotent, and eventually settle `done`; throws propagate*。所以实现刻意让 throw 在**改状态之前**发生。

`job_kill` 工具侧的返回([`tool-jobs/src/index.ts:389-398`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L389-L398))是 `outcome: 'cancellation-requested' | 'already-finished'` 加一份 `ctx.jobs.get(...)` 的当前快照——`get` 是**非消费**的,不会动 `reported`。

### 6.1 teardown 的强制失败

`disposeAll`(`jobs-local/src/index.ts:481-...)` 的顺序是:**先 `listenersClosed = true`**(注释:*each layer entry's undo belongs to the fiber that registered it, so this service may not drop them on its own way out*)→ `cancelForTeardown(all, 'jobs service disposed')` → `await Promise.all(all.map(job => job.settled))`。`cancelForTeardown`(`:507`)对抛错的 `cancel` **强制失败那一条记录**,避免 teardown 死锁;`listenersClosed` 让销毁后的结算不再通知任何监听者。

owner 侧的对称实现是 `disposeOwned`(`:467-475`):取消 → 等全部 settled → 从 store 删除 → **`notifyChanged(owner)`**(注释:*Removal is the one visible-set change no per-job record carries*)。

---

## 第七节 与后台 subagent 的关系

```typescript
// packages/subagent/tool-subagent/src/index.ts:544-556(节选)
const id = jobs.start({ kind: 'subagent', label: args.description, owner: parent, run: () => {
  const controller = new AbortController()
  const start = runtimeCtx.subagents.start(config.provider, { ...request, signal: controller.signal })
  return {
    cancel: (reason) => { controller.abort(reason ?? 'background subagent task killed') },
    done: settleStart(start, controller.signal),
  }
} })
```

三个观察:

1. **`kind: 'subagent'` 是 `JobKindMap` 的成员之一**(与 `bash` 并列,[`jobs/src/types.ts:23-26`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L23-L26)),插件可用 declaration merging 扩展。
2. **`cancel` 只是 `controller.abort(...)`**:因为 `cancel` 必须**同步**、而 `subagents.start()` 是异步的,所以这里用"先造 controller、再拿它的 signal 去 start"的写法——把异步启动与同步取消解耦。
3. **`done` 由 `settleStart` 提供**,它把两种失败分开:

```typescript
// packages/subagent/tool-subagent/src/index.ts:142-153
async function settleStart(start: Promise<SubagentRun>, signal: AbortSignal): Promise<JobOutcome> {
  try {
    return await settleRun(await start)
  } catch (error: unknown) {
    // Product providers aggregate startup and rollback failures. Cancellation
    // must not turn a failed cleanup into a cleanly killed Job.
    return signal.aborted && !(error instanceof AggregateError)
      ? { status: 'killed' }
      : { status: 'failed', detail: String(error) }
  }
}
```

`AggregateError` 是"多个失败聚合"的标志(provider 用它表示启动与回滚都失败),**此时即使已取消也报 `failed`**——**清理失败不得伪装成干净被杀**。

### 7.1 三条后台路线的分工

| 路线 | 触发 | 谁持有子 | 回收方式 |
|---|---|---|---|
| one-shot 后台 | `run_in_background: true` + `backgroundMode: one-shot` | job 的 `run()` 闭包持有 `SubagentRun` | `job_kill` → `controller.abort` → provider 取消 → `done` |
| 续存后台 | `backgroundMode: continuable`(默认调度为后台) | `SubagentContinuationManager` 持有 `AgentHandle` | `send_message` 继续对话;`ctx.subagents.drainContinuableChildren` 释放 |
| 前台 | 默认(`one-shot` 且未指定后台) | 工具调用的 `await` 持有 | 工具返回即结束;`exec.signal` 取消 |

调度判据是 `request.run_in_background ?? options.continuable`([`tool-subagent/src/index.ts:303`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/subagent/tool-subagent/src/index.ts#L303))——**one-shot 默认前台,continuable 默认后台**。续存路线**不**经过 jobs:它返回一个 `subagentId`,由 [04](./04-continuation-and-control.md) 的续存机制接管。

![时序图：06-jobs-and-notifications](../assets/diagrams/multi-agent__06-jobs-and-notifications-386.svg)

<details><summary>Mermaid 源码</summary>

```mermaid
sequenceDiagram
    autonumber
    participant M as 模型
    participant T as tool-jobs / tool-subagent
    participant J as LocalJobRegistry
    participant P as 生产者(subagent / bash)
    participant O as owner Agent

    M->>T: subagent{run_in_background: true}
    T->>J: start({kind:'subagent', owner, run})
    J->>J: servesOwner → 参数校验 → ensureOwnerCleanup → 限额
    J->>P: spec.run() 同步返回 hooks
    J-->>T: JobId 'subagent-1'
    T-->>M: {kind:'background', jobId}
    Note over P: 执行中(资源由生产者拥有)
    M->>T: job_output(job_id, wait:true, timeout_ms)
    T->>J: wait(id, min(timeout, cap))
    alt 超时
        J-->>T: 快照(status: running) → 作业仍存活
    else 结算
        J->>J: settle():first-wins → 释放 waiter → markSettled
        J->>J: notifyChanged → listenersFor(owner) → onJobDone
        J-->>T: 终态快照
    end
    T->>J: read(id) → 增量或最终输出;终态读标记 reported
    Note over J,O: onJobDone:busy→inject / idle 且预算内→followup(预算重置只在 kind==='user' 的输入被认领时)
    M->>T: job_kill(job_id)
    T->>J: kill(id)
    J->>P: cancel(reason)  ← 同步;抛错则状态不变
    J-->>T: 'requested'(status 已为 stopping,reported=true)
    P-->>J: done.resolve({status:'killed'})  ← 真正的终态
```

</details>

---

## 第八节 关键文件/符号索引表

| 符号 | 位置 | 职责 |
|---|---|---|
| `JobRegistry`(抽象) | [`packages/jobs/jobs/src/index.ts:62-177`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/index.ts#L62-L177) | seam;`new.target` 拒绝直接加载;六条实现语义 |
| `start` / `list` / `get` / `read` / `kill` / `wait` | [`jobs/src/index.ts:82`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/index.ts#L82) / `90` / `99` / `109` / `120` / `133` | 抽象方法契约 |
| `onJobDone` / `onJobsChanged` / `attachController` | [`jobs/src/index.ts:143`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/index.ts#L143) / `167` / `176` | effect-scoped 注册;owner 相对投递 |
| `JobStatus` / `JobKindMap` | [`jobs/src/types.ts:17`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L17) / `23-29` | 生命周期状态;`bash` / `subagent`,可声明合并扩展 |
| `JobStart` / `JobHooks` | [`jobs/src/types.ts:46-69`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L46-L69) / `72-91` | `kind`/`label`/`outputLimitBytes?`/`owner?`/`run()`;`cancel`(同步幂等)/ `done`(不 reject)/ `readOutput?` |
| `JobOutcome` / `JobSnapshot` / `JobRead` | [`jobs/src/types.ts:32-39`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L32-L39) / `97-128` / `131-140` | 终态、只读投影、读取返回 |
| `JobDoneListener` / `JobsChangedListener` | [`jobs/src/types.ts:146-160`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs/src/types.ts#L146-L160) | 完成通知(带精确 owner)/ 可见集变更(owner 粒度) |
| `LocalJobRegistry.start` | [`packages/jobs/jobs-local/src/index.ts:131-190`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L131-L190) | 三道闸 + `run()` 分界 + 四步注册 |
| `wait` / `read` / `kill` | [`jobs-local/src/index.ts:230-280`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L230-L280) / `205-213` / `215-228` | `deadline` 区分超时与取消;消费游标;先 cancel 后改状态 |
| `settle` | [`jobs-local/src/index.ts:416-440`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L416-L440) | first-wins → 释放 waiter → 变更通知 → **最后**完成通知 |
| `servesOwner` / `listenersFor` / `changedFor` | [`jobs-local/src/index.ts:315-319`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L315-L319) / `338-342` / `388-392` | owner 相对的三处解析 |
| `attachController` / `onJobDone` / `onJobsChanged` | [`jobs-local/src/index.ts:297-305`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L297-L305) / `281-287` / `289-295` | 分层注册;Symbol token 保证独立可销毁 |
| `ensureOwnerCleanup` / `disposeOwned` / `disposeAll` / `cancelForTeardown` | [`jobs-local/src/index.ts:448-464`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/jobs-local/src/index.ts#L448-L464) / `467-475` / `481-...` / `507` | owner scope 上的 awaited cleanup;抛错的 cancel 强制失败 |
| `Config`(tool-jobs) | [`packages/jobs/tool-jobs/src/index.ts:31-52`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L31-L52) | wait 默认/上限、`completionDelivery`、`maxConsecutiveWakes` |
| `job_output` / `job_list` / `job_kill` | [`tool-jobs/src/index.ts:301-339`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L301-L339) / `341-359` / `361-399` | 三个模型侧控制 |
| `onJobDone` 通知 + 唤醒预算 | [`tool-jobs/src/index.ts:278-299`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L278-L299) / `210-229` | busy→inject / idle→followup;预算只在 `kind==='user'` 认领时重置 |
| `visibleOutputLimit` / `finalizeTaskContent` | [`tool-jobs/src/index.ts:184-189`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/jobs/tool-jobs/src/index.ts#L184-L189) / `237-256` | 按作业的 `outputLimitBytes` 截断完整结果 |
| 后台 subagent 产生方 / `settleStart` / `resolveDelegationRun` | [`packages/subagent/tool-subagent/src/index.ts:544-560`](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/packages/subagent/tool-subagent/src/index.ts#L544-L560) / `142-153` / `287-305` | `kind:'subagent'` 的 job;`AggregateError` → `failed`;`run_in_background ?? continuable` |
