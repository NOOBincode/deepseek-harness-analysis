# 03 · 能力缝三角色的函数级解剖

> 分析对象:[innokria/deepseek-harness](https://github.com/innokria/deepseek-harness) @ `dbbaa4a37`
> 样本:`ctx.sandbox`、`ctx.llm`、`ctx.subagents`、`ctx.sessionPersistence` 四条真实能力缝
> 前置:[第十二章第二节](../12-architecture-highlights.md)给三角色的定义与模式论证。

---

## 第〇节 一句话结论

一条 capability seam 由三角色组成:

- **Service Definition**:一个 `extends Service` 的类(抽象类或具体注册表),构造时 `super(ctx, '<key>')` 完成"构造即注册";它同时是**词汇类型的唯一出口**(`SandboxPolicy`、`GenerateOptions`、`SubagentProvider` 都必须从这里 import)。
- **Service Provider**:**另一个包**里实现该契约的类,由**函数插件**(`name`/`inject`/`Config`/`apply`)在 `apply` 里注册,注册调用返回 disposer(即 `ctx.effect`)。
- **Consumer**:`inject` 了该服务的第三个包,只 import Definition 的类型并调用契约方法。

第四件事是多数缝会缺的那一件:`./invariant` 伴随插件。判定标准不是"这个包重要吗",而是**"是否存在两个由不同角色独立观测、且可以互相偏离的事实"**。

```text
   Service Definition(包 A):class X extends Service
     super(ctx,'x') ← 构造即注册 | abstract doThing() ← 契约方法 | declare module cordis ← 词汇
              │ import 类型(不 import 实现)
   ┌──────────┴───────────┐
   ▼                      ▼
 Provider(包 B)          Consumer(包 C)
 class Y implements X     inject:['x'];ctx.x.doThing(...)
 apply(ctx){ ctx.x.register(new Y) }
   └──── §6 invariant 伴随插件旁听两侧的独立观测 ────┘
```

---

## 第一节 判定框架:什么算一条缝

| 条件 | 检查方式 | 反例 |
|---|---|---|
| Definition 必须是 **Cordis `Service`**,不能是 TypeScript `interface` | 文件里有 `extends Service` + `super(ctx, '<key>')` | 只有 `.d.ts` 的接口包没有 `ctx.<key>` 的注册与生命周期 |
| Provider 必须能被**独立替换**,Consumer 不感知 | Consumer 侧 `grep` 不到 provider 包名 | Consumer import provider 的专有类型即违规 |
| 三角色必须**同时存在** | 缺 Provider = 死契约;缺 Consumer = 无人使用的抽象(仓库气味:"a public service method with one internal caller",`packages/AGENTS.md:10`) | —— |

仓库允许把两个角色放进同一个包,但必须是有理由的**折叠**:`ctx.llm` 把 Definition 与 Consumer 合并,因为这里的 Consumer 是主循环本身,不存在可替换的 schema 面;`ctx.sessionPersistence` 的 Definition 附带共享校验脚手架(`storage-contract.ts`、`handle.ts`),但 Provider 依然独立成包。

定位一条缝的三条命令:

```text
grep -rn "super(ctx, '<key>')" packages/    # 1. Definition
grep -rn "extends <DefinitionClass>" packages/   # 2. Provider
grep -rn "ctx\.<key>\.\|ctx\.get('<key>')" packages/   # 3. Consumer
```

---

## 第二节 缝 1:`ctx.sandbox`(同世界进程隔离)

### 2.1 三列对照

| 角色 | 包 | 关键符号与位置 |
|---|---|---|
| **Definition** | `packages/sandbox/sandbox` | `SandboxProvider extends Service`(`src/index.ts:158`)、`super(ctx,'sandbox')`(`:161`)、`abstract confine(argv, policy)`(`:175`)、词汇 `SandboxMode`(`:29`)/`SandboxExecutionPolicy`(`:39`)/`SandboxPolicy`(`:69`)/`ConfinedArgv`(`:95`)/`RunnerFailureRule`(`:81`)、`SandboxUnavailableError`(`:131`)、`Context.sandbox` 声明(`:146-150`) |
| **Provider** | `packages/sandbox/sandbox-local` | `LocalSandboxProvider extends SandboxProvider`(`src/index.ts:250`),内部按平台选 bwrap / Landlock / Seatbelt / ACL 受限令牌 |
| **Consumer** | `packages/shell/bash-sandbox` | `this.ctx.sandbox.confine(['bash','-c',command], policy)`(`src/index.ts:180`);策略来自 `this.ctx.sandboxPolicy.resolve()`(`:86`) |
| **Consumer** | `packages/shell/pwsh-sandbox` | `this.ctx.sandbox.confine(this.argv(spec), policy)`(`src/index.ts:185`) |
| **Consumer** | `packages/terminal/terminal-bash` | `const sandbox = ctx.get('sandbox')`(`src/index.ts:103`)→ 缺失即抛(`:105`)→ `sandbox.confine(...).argv`(`:108`) |
| **旁路角色** | `packages/sandbox/sandbox-policy` | 提供 `ctx.sandboxPolicy`(部署默认值 + 逐会话覆盖),**不是** `ctx.sandbox` 的实现 |

### 2.2 Definition 的契约密度

```typescript
// packages/sandbox/sandbox/src/index.ts:152-176(节选)
/**
 * Abstract process-sandbox service. {@link confine} must return enforcing argv
 * or fail closed at wrap or runner-execution time; silent unconfined passthrough
 * is forbidden. ...
 */
export abstract class SandboxProvider extends Service {
  /* v8 ignore next -- abstract service construction is covered through concrete provider packages. */
  constructor(ctx: Context) { super(ctx, 'sandbox') }

  /**
   * Wrap `argv` so it executes confined under `policy` on this host; the
   * caller spawns the returned argv in place of its own.
   * @param argv - the exact argv the caller is about to spawn (program plus
   *   arguments), NOT a shell string — a shell-shaped consumer passes `['bash','-c',command]`.
   * @param policy - the file-effect policy this execution runs under, carried per call.
   * @returns the argv to spawn instead, plus the enforcement completeness ...
   */
  abstract confine(argv: readonly string[], policy: SandboxPolicy): ConfinedArgv
}
```

三个值得抄的设计:

1. **"逐调用携带"写进类型注释**(`SandboxPolicy` 的 JSDoc,`:61-68`):"carried PER CALL, not fixed on the provider: two consumers may confine under different policies at the same instant … Defaulting/resolution is an explicit step at the consumer boundary; the provider treats the policy as fully specified."——根 `AGENTS.md:116` 那条 "Explicit > implicit at package boundaries" 的落地形态。
2. **返回值携带"证据"而非布尔**(`ConfinedArgv`,`:95-116`):`enforcement: 'full' | 'partial'`、`denialSignatures`(该 backend 自己的拒绝方言,而非跨后端并集)、`runnerFailureRules`(区分"runner 启动失败"与"命令被拒绝")。**Provider 把判定所需事实全部交给 Consumer**,Consumer 无需知道 backend 是谁。
3. **失败关闭是类型化错误而非 `undefined`**:`SandboxUnavailableError`(`:131-144`)携带 `code = 'SANDBOX_UNAVAILABLE'`(`:124`)穿过结构化错误通道。

### 2.3 Provider 与 Consumer 的接缝

Provider 侧:`LocalSandboxProvider`(`sandbox-local/src/index.ts:250`)的 JSDoc 说明它"Registers as `ctx.sandbox`",并缓存链判定与 Windows ACL 写授权(`:242-249`);`static Config` 只有 `runnerCommand`、`runnerFailureSignatures`、`probeTimeoutMs` 三个字段(`:252-256`)——**能变的都在配置里,不能变的(模式词汇、逐调用策略)留在 Definition**。

Consumer 侧的调用点极薄,正是缝的目的:

```typescript
// packages/shell/bash-sandbox/src/index.ts:171-181
/**
 * Wrap one shell command via the `ctx.sandbox` provider. Provider errors
 * propagate unchanged; the returned argv is handed directly to the local
 * executor's subprocess path.
 * @param command - shell source for the confined inner `bash -c`.
 * @param policy - resolved confined execution policy.
 * @returns the provider's exact argv and settlement-classification facts.
 */
private confine(command: string, policy: SandboxPolicy): ConfinedArgv {
  return this.ctx.sandbox.confine(['bash', '-c', command], policy)
}
```

`terminal-bash` 展示"可选 provider"的正确写法:

```typescript
// packages/terminal/terminal-bash/src/index.ts:103-108
const sandbox = ctx.get('sandbox')
if (sandbox === undefined) {
  throw new Error(`terminal-bash: sandbox mode "${policy.mode}" requires a ctx.sandbox provider in the execution world`)
}
return sandbox.confine(argv, { ...policy, mode: policy.mode }).argv
```

**用 `ctx.get` 而不是 `ctx.sandbox`**:`terminal-bash` 的 `inject` 里不含 `sandbox`(非受限模式不需要),直接属性读会走祖先上溯并在根 fiber 抛 `without inject`(机制见 [01](./01-cordis-runtime-internals.md)第一节)。这条规则写在 `packages/AGENTS.md:6`。

---

## 第三节 缝 2:`ctx.llm`(provider 中立模型调用)

### 3.1 三列对照

| 角色 | 包 | 关键符号与位置 |
|---|---|---|
| **Definition(含 Consumer 折叠)** | `packages/llm/llm` | `LlmAdapter` 抽象类(`src/index.ts:200`)、唯一必实现的 `abstract stream(options)`(`:281`)、`prepareCall`(`:269`)、`LlmRuntime extends TypertRemoteService`(`:333`)、`registerAdapter(providers, adapter)`(`:387`)、`llm/stream` 声明(`:72`)、`@Remote listProviders()`(`:468-469`) |
| **Provider** | `packages/llm/llm-deepseek` | `name`(`src/index.ts:84`)、`inject = ['llm']`(`:85`)、`Config`(`:187`)、`apply`(`:421`)、`ctx.llm.registerAdapter([PROVIDER], adapter)`(`:492`) |
| **Provider** | `packages/llm/llm-pi-ai` | 同形态;出厂挂在 base 层(`packages/bundle/base/cordis.patch.yml:107-108`) |
| **Consumer** | `packages/core/agent-loop` | `agent.ts:541` `preparedCall = await this.loopCtx.llm.prepareCall(proposedConfig, signal)`;`agent.ts:390` `this.loopCtx.llm.stream(request)`;`index.ts:360` 的 `static inject` 含 `'llm'` |
| **Consumer** | 重试/回放插件 | `llm-retry`、`llm-replay` 监听 `llm/stream`(`docs/event-producer-consumer.md:49`) |

### 3.2 `registerAdapter` 的注册三件套与 Provider 接线

`registerAdapter`(`llm/src/index.ts:387`)的模式:**候选路由集先整体验证(`prepareRoutes`,`:423`)再在一个同步区段内原子交换(`commitRoutes`,`:454-462`)**;拒绝的候选"leaves the registry exactly as it was";effect 体注册、disposer 里逐个 `adapters.delete(provider)` 并 `emitAdaptersUpdated()`;重复 provider 整批拒绝(`DUPLICATE_ADAPTER`),all-or-nothing。

Provider 侧只有两行接线(`llm-deepseek/src/index.ts:487-492`),但 `adapter` 的构造方式才是关键(`:471-486`):它是 **transport-only** 的实现,所有会变的事实都以 **thunk** 注入(`options`、`resolveApiKey`、`resolveUserId`、`resolveAttachments`、`resolveImageAccess`、`prepareExtensions`)。其中 `options()`(`:425-443`)每次调用重新 resolve,所以设置变更无需重新注册就能到达下一个请求;唯一例外是 retry policy——注册时被捕获、无法按请求刷新,由 `registration.replace([PROVIDER])`(`:502`)在一个同步 registry 区段内换掉。注释解释了为什么不 dispose+register:"would publish an empty route set between the two, and an observer that reacted to it would see this provider disappear and come back"(`:498-501`)。

### 3.3 缝的折叠点与四类可替换物

`ctx.llm` 把 Definition 与 Consumer 合在一个包,但接缝仍在,表现为四种独立可替换:

| 可替换物 | 机制 | 位置 |
|---|---|---|
| 具体模型商 | `LlmAdapter` 子类 + `registerAdapter` | `llm-deepseek`、`llm-pi-ai` |
| 请求拦截 | `llm/stream` waterfall | `llm-retry`、`llm-replay`、`session-title` |
| 模型清单与路由 | `registerConfigurableProviders` / `replace` | Provider 的 `apply` |
| 请求装配与代际绑定 | `prepareCall` | `llm/src/index.ts:269-281` |

---

## 第四节 缝 3:`ctx.subagents`(子 agent 派生)

### 4.1 三列对照

| 角色 | 包 | 关键符号与位置 |
|---|---|---|
| **Definition** | `packages/subagent/subagent` | `SubagentRuntime extends TypertRemoteService`(`src/index.ts:188`)、`super(ctx,'subagents')`(`:199`)、`registerProvider`(`:509`)、`getProvider`(`:532`)、`list`(`:540`)、`start`(`:556`)、`startContinuable`(`:228`)、`sendMessage`(`:246`)、`@Remote` 三方法(`:385`、`:412`、`:479`) |
| **Provider** | `subagent-spawn-in-process` | `name`(`src/index.ts:19`)、`inject = ['subagents']`(`:22`)、`Config`(`:30`)、`class SpawnInProcessProvider implements SubagentProvider`(`:41`)、`apply` 里 `ctx.subagents.registerProvider(new SpawnInProcessProvider(config.providerName))`(`:68-69`) |
| **Provider** | `subagent-fork-in-process` | `ctx.subagents.registerProvider(new ForkInProcessProvider(config.providerName))`(`src/index.ts:95`) |
| **Provider** | `subagent-acp` / `subagent-dsh-sdk` / `subagent-codex` / `subagent-claude-code` | 注册点:`src/index.ts:205`、`:199`、`:135`、`:151`——**同一接口下的四种截然不同的实现** |
| **Consumer** | `packages/subagent/tool-subagent` | `ctx.subagents.getProvider(config.provider)`(`src/index.ts:357`)、`startContinuable({...})`(`:530`)、`start(config.provider, {...})`(`:550`、`:563`) |

### 4.2 Provider 注册必须返回 effect

```typescript
// packages/subagent/subagent/src/index.ts:509-525
registerProvider(provider: SubagentProvider): () => void {
  const name = provider.name
  return this.ctx.effect(function* (this: SubagentRuntime) {
    if (this.providers.has(name)) {
      throw new SubagentError(`a subagent provider named "${name}" is already registered`, 'DUPLICATE_PROVIDER')
    }
    this.providers.set(name, provider)
    yield () => { this.providers.delete(name); this.emitLifecycle('subagent/provider-removed', name) }
    // A throwing added-listener unwinds the yielded rollback, matching the
    // repository's fail-loud registration semantics.
    this.ctx.emit('subagent/provider-added', provider)
  }.bind(this), 'subagents.registerProvider()')
}
```

**"生成器 effect + 先登记回滚再广播"** 是关键形态:`yield () => {...}` 在 `provider-added` 广播**之前**登记回滚,因此若某个 added 监听器抛错,外层 effect 会展开已 yield 的回滚(`fiber.ts:375-382` 同步迭代分支),注册不会留半成品。`emitLifecycle` 是**含 scope 载体的派发**(构造于 `:200`,`createLifecycleEmitter(this.ctx, parent => scopeTarget(this, parent))`),使 `subagent/start` 的监听器只收到自己那份委派(`@dshScopeScan unsupported`,`:158`)。

### 4.3 Consumer 侧与 `start` 的发布边界

Definition 的模块注释(`:14-22`)划清三个公开操作:`start` 返回一次性已发布 run;`startContinuable` 建立持久化可续子 agent;`sendMessage` 在相邻 Agent 间转发消息,不暴露子进程是否常驻。Consumer 侧各有一处(`tool-subagent/src/index.ts:357`、`:530`、`:550`、`:563`)。

`start`(`:556-586`)把"能力校验"和"记录"排在任何委派之前:`expectProvider` → `assertCapabilities` → `assertSubagentMaxDepth` → `assertObjectJsonSchema` → 生成 descriptor → `await provider.start(resolved)`;发布后若 `establishCatalogChild` 失败,则 `void run.result.catch(() => undefined)` + `await run.dispose()` 后抛目录错误(注释:"No caller receives this run; the catalog error owns the failed start"),处置错误只 `logger.warn`。**provider promise fulfill 是唯一的发布边界**(JSDoc `:544-551`):provider 抛错 ⇒ 调用者拿不到 run,也就不发任何 run 生命周期事件。

---

## 第五节 缝 4:`ctx.sessionPersistence`(会话持久化后端)

### 5.1 三列对照

| 角色 | 包 | 关键符号与位置 |
|---|---|---|
| **Definition** | `packages/session/session-persistence` | `SessionPersistence extends Service`(`src/index.ts:135`)、`super(ctx,'sessionPersistence')`(`:136-138`)、五个抽象方法 `create`(`:147`)、`open`(`:162`)、`flush`(`:175`)、`stat`(`:191`)、`list`(`:198`);从 `handle.ts` / `storage-contract.ts` 再导出词汇与共享校验(`:17-44`) |
| **Provider** | `packages/session/session-persistence-jsonl` | `class JsonlSessionPersistence extends SessionPersistence`(`src/index.ts:235`)——拥有物理分帧、压缩、世代选择、独占发布 |
| **Provider(检索侧)** | `packages/session-query/session-query-sqlite` | 独立后端,消费同一契约 |
| **Consumer** | `packages/core/agent-loop` | `ctx.get('sessionPersistence')`(`src/index.ts:429`)、`ctx.inject(['sessionPersistence'], childCtx => this.resumeWith(...))`(`:444-445`)、另两处 opportunistic 读取(`:731`、`:845`) |
| **Consumer** | `packages/acp/acp` | `const persistence = ctx.sessionPersistence`(`src/index.ts:100`) |
| **Consumer** | `packages/feedback/message-feedback` | `stat`(`src/index.ts:227`)、`open(id,'read')`(`:246`)、`open(id, write ? 'write' : 'read')`(`:261`) |
| **Consumer** | `packages/experimental/agent-team` | `readPersistedSession(this.ctx.sessionPersistence, childId, signal)`(`src/roster.ts:349`、`:402`;`src/mailbox.ts:323`) |

### 5.2 契约里最重的两条语义

Definition 的类级 JSDoc(`:115-134`)规定了两件跨 backend 的行为:

- **"append 尽力而为,flush 是持久化屏障"**:`append` 只保证尽力持久化;`flush`(per handle 或 service-wide)才是耐久性屏障。这把耐久性从每个 backend 的自由裁量变成契约条款——Consumer(agent-loop)据此把 `session/flush` 当作下一轮之前的顺序与错误观测检查点(见 `packages/session/session-persistence/README.md` 的 write-path 段)。
- **可见性与新鲜度**:`create` 在当前进程立刻可通过 `stat`/`list`/`open` 观察到(即使 backend 延迟物化),"other processes see the session only once it materializes, and a session that never materialized before a crash never existed";`append`/`flush` 解析后新发起的读至少能看到该前缀。

其余共享语义(events 从 seq 0 连续且永不重写、撕裂的物理尾部永不返回给读者且由写路径在首次 append 前截断、只校验当前格式记录并对未知词汇 fail-closed)写在同段 JSDoc 里,由 `storage-contract.ts` 的 `assertContiguous`/`validateStoredEvents`/`materializeAppendBatch` 提供共享实现(`:37-44`)。

### 5.3 为什么这条缝没有 `./invariant`

`packages/session/session-persistence/package.json` 里**没有** `./invariant` 导出,README 给出理由(`README.md:99`):

> No runtime invariant companion is published; persistence correctness requires backend round-trip and crash-tail tests; this package exposes no continuously observable in-process relation.

---

## 第六节 invariant 伴随插件:判定与写法

### 6.1 发布判定:三问

| 问题 | 是 | 否 |
|---|---|---|
| 有没有**两个由不同角色独立观测**的事实(如"事件广播的载荷"与"注册表里的状态"、"请求内容"与"日志推导")? | 需要 invariant | 只检查单点状态 ⇒ 不发布 |
| 这两个事实**能否在正常运行中偏离**? | 需要 invariant | 只能由非法输入触发 ⇒ 交给类型/解析层 |
| 这套关系是否**归属于本包**? | 需要 invariant | 归别人 ⇒ 别人的伴随插件负责 |

`packages/AGENTS.md:19` 明列无效形态:**空的 installer、检查服务是否存在、检查插件元数据、检查 effect、检查固定示例**。两个真实的不发布判例:整条 `ctx.sandbox` 缝的 Definition 包(`packages/sandbox/sandbox/package.json`)没有 `./invariant`;`ctx.sessionPersistence` 的理由见 5.3。

### 6.2 装载机制:`register` 把 installer 变成子 fiber 插件

```typescript
// packages/runtime-diagnostics/invariants/src/index.ts:136-188(节选)
register(packageName: string, installer: InvariantInstaller): () => void {
  if (packageName.length === 0 || packageName.trim() !== packageName || /\s/.test(packageName)) {
    throw new Error('invariants: packageName must be non-blank and contain no whitespace')
  }
  if (this.registrations.has(packageName)) throw new Error(`invariants: package "${packageName}" is already registered`)
  ...
  registration = ctx.effect(async () => {
    if (!this.selected(packageName)) { return () => { registrations.delete(packageName) } }
    const installInvariant = (childCtx: Context) => (
      installer(childCtx, (message): never => { throw new InvariantError(packageName, message) })
    )
    try {
      const child = ctx.plugin(installer.inject === undefined
        ? installInvariant
        : Object.assign(installInvariant, { inject: installer.inject }))    // ← 成为子 fiber 的 inject
      try { await child } catch (error) { await child.dispose(); throw error }
      return async () => { try { await child.dispose() } finally { registrations.delete(packageName) } }
    } catch (error) { registrations.delete(packageName); throw error }
  }, `invariants.register(${JSON.stringify(packageName)})`)
  ...
}
```

四点:**installer 的 `inject`**(`InvariantInstaller` 接口的 `readonly inject?: Inject`,`:32-42`)**会成为安装它的子 fiber 的 `inject`**,所以伴随插件可以声明依赖而不必自己等待;**`fail` 是抛 `InvariantError` 的函数**(`:161-163`),消息自动带 `invariant violated by "<packageName>":` 前缀(`:62`);**失败会 dispose 子 fiber 并释放包名占用**(`:170-175`、`:184-187`);**过滤器**(`enabled`/`package_allowlist`/`package_blocklist`,`:15-22`、`selected()` `:121-126`)即使命中也不释放包名占用(`:149`)。

伴随插件的挂载位置是**组合层的一行**,不出厂默认挂载——只有 `dsh-sdk-minimal` 挂了四个(`packages/bundle/sdk-minimal/cordis.patch.yml:103-116`:`@deepseek-ai/dsh-invariants` 与 `dsh-session`/`dsh-agent`/`dsh-scope`/`dsh-agent-loop` 的 `.../invariant`)。

### 6.3 样本 A:`dsh-subagent/invariant`(注册表与生命周期配对)

```typescript
// packages/subagent/subagent/src/invariant.ts:9-22,56-62(节选)
export const name = 'subagent-invariant'
export const inject = ['invariants']

function validateRunEnd(start: SubagentRunInfo, end: SubagentRunEndInfo, fail: InvariantFailure): void {
  if (start.provider !== end.provider || start.id !== end.id || start.local !== end.local) {
    fail(`subagent/end identity diverges from subagent/start for run ${JSON.stringify(end.runId)}`)
  }
}

const install: InvariantInstaller = Object.assign((ctx: Context, fail: InvariantFailure) => {
  const providers = new Set(ctx.subagents.list())
  const runs = new Map<string, SubagentRunInfo>()
  ctx.on('internal/dispatch', (_mode, eventName, args) => { ... }, { global: true })
  ctx.on('subagent/provider-added', (provider) => { if (stagedProviders.delete(provider)) providers.add(provider.name) }, { global: true })
  ...
}, { inject: ['subagents'] })

export const apply = (ctx: Context): Promise<() => void> =>
  Promise.resolve(ctx.invariants.register(PACKAGE_NAME, install))
```

它断言的**两条独立可观测关系**:

1. **注册表(服务状态)与事件(广播载荷)一致**:采用**两阶段**写法——先在 `internal/dispatch`(`:30-62`)用 `staged*` 弱集合暂存并校验,再在真正的公开监听器里提交状态(`:64-83`)。这解决了 `ctx.emit` 同步派发时"无法区分事件载荷非法与状态机非法"的问题。
2. **生命周期配对**:`subagent/end` 必须有匹配的 `subagent/start` 且 `provider`/`id`/`local` 不变(`:56-61`);重复的 `provider-added` 或重复 `runId` 立即失败(`:34`、`:52`)。

它**刻意不检查**的东西同样重要(`:46-48` 注释):"Provider availability is an admission-time relationship. A published one-shot run may outlive provider removal"——即"run 存在 ⇒ provider 仍在注册表里"**不是**不变量。

### 6.4 样本 B:`dsh-tools/invariant`(管道阶段单调性 + 快照冻结)

```typescript
// packages/core/tools/src/invariant.ts:17-30,95-120(节选)
type ToolStage = 'pre' | 'execute' | 'post'
function validateResult(exec, result, fail): void {
  if (!Object.isFrozen(exec)) fail('tools/result execution must be frozen before publication')
  if (!Object.isFrozen(result) || !Object.isFrozen(result.content)) {
    fail('tools/result outcome and content must be frozen before publication')
  }
  if (exec.name.length === 0 || String(exec.callId).length === 0) fail('tools/result execution must carry non-empty name and callId')
}
...
  ctx.on('internal/dispatch', (_mode, eventName, args) => {
    if (eventName === 'tools/pre-execute') {
      const exec = args[0] as ToolExecution
      if (stages.has(exec)) fail('tools/pre-execute repeated for one execution')
      stages.set(exec, 'pre'); return
    }
    if (eventName === 'tools/execute') {
      const exec = args[0] as ToolExecution
      if (stages.get(exec) !== 'pre') fail('tools/execute must follow tools/pre-execute')
      stages.set(exec, 'execute'); return
    }
    if (eventName === 'tools/post-execute') {
      const exec = args[0] as ToolExecution
      const previous = stages.get(exec)
      if (previous !== 'pre' && previous !== 'execute') fail('tools/post-execute must follow tools/pre-execute or tools/execute')
      stages.set(exec, 'post'); return
    }
    if (eventName !== 'tools/result') return
    const [exec, result] = args as [Readonly<ToolExecution>, Readonly<ToolExecutionResult>]
    validateResult(exec, result, fail)
    stages.delete(exec)
  }, { global: true })
}, { inject: ['sessions'] })
```

三个手法:**旁听 `internal/dispatch` 而不是注册公开监听器**——公开监听器可被别的插件 `prepend`/短路,而 `internal/dispatch` 是 `dispatch()` 在派发前必然发出的事件(`events.ts:169`),并配 `{ global: true }` 跳过 scope 过滤(否则 agent-scoped 的 `tools/*` 会漏掉大半);**用 `WeakMap<object, ToolStage>` 以 `exec` 对象身份为键**(`:34`),身份精确且不泄漏;**同一条 `install` 还负责 PTC 子调度封闭性**(`:37-57`):`tool/ptc-dispatch*` 的 `rootCallId`/`parentCallId`/`subCallId` 必须自洽,`seed()`(`:58-74`)在会话创建时**重放整段历史**建立基线(`:77-78` 对已有会话播种),所以这套检查对"中途挂载 invariant"也成立。

### 6.5 样本 C:`dsh-agent-loop/invariant`(模型可见 ⟺ 已落日志)

```typescript
// packages/core/agent-loop/src/invariant.ts:19-57(节选)
const install: InvariantInstaller = Object.assign((ctx: Context, fail: InvariantFailure) => {
  // Prepend prevents a short-circuiting replay listener from silencing the check.
  ctx.on('llm/stream', (options: GenerateOptions, next) => {
    if (!isAgentLoopRequest(options)) return next()
    if (!Object.isFrozen(options)) fail('a loop-built request must be frozen')
    if (options.sessionId === undefined) fail('a loop-built request must carry a session id')
    const session = ctx.sessions.get(options.sessionId)
    if (!session) fail(`a loop-built request must carry a live session id, got "${String(options.sessionId)}"`)
    if (!Object.isFrozen(options.messages)) fail('a loop-built request must carry a frozen messages array')
    const events = session.snapshotEvents()
    if (!events.some(event => event.type === 'step/start')) return fail('a loop-built request with no step/start in its session log')
    const header = foldRequestHeader(events)
    if (header === undefined) return fail('a loop-built request with no request/header event in its session log')
    const expected = session.deriveMessages()
    if (JSON.stringify(options.messages) !== JSON.stringify(expected)) {
      fail(`llm request for session "${String(session.id)}" diverges from the dispatch-time durable derivation (log-reconstruction desync)`)
    }
    ...
    return next()
  }, { global: true, prepend: true })
}, { inject: ['sessions'] })
```

它是"**模型可见 ⟺ 已落日志**"的运行期执法者:被 AgentLoop 构造的请求进入 `llm/stream` 时,messages 必须逐字节等于从会话日志推导出的消息(`:40-43`),model/temperature/maxTokens/stop/tools 必须等于日志中折叠出的 `request/header`(`:46-54`)。三个手法:**`prepend: true` 且注释说明原因**(`:20`,不变量必须跑在可能短路的业务监听器之前);**用 `isAgentLoopRequest(options)` 收窄适用范围**(`:22`,别的调用方直接 `next()`);**失败用 `return fail(...)`**(`:34`、`:38`),配合 `InvariantFailure` 的 `never` 返回类型(`invariants/src/index.ts:29`)让控制流对 TypeScript 成立。

### 6.6 样本 D:`dsh-sandbox-policy/invariant`(日志侧枚举校验)

`packages/sandbox/sandbox-policy/src/invariant.ts` 断言"策略服务写进持久日志事件里的 mode 必须是已知模式"(`:17-19` 的 `fail(\`sandbox/mode carries unknown mode ${JSON.stringify(event.data.mode)}\`)`),即"服务内部枚举"与"落到日志里的字符串"这两个独立观测点一致。它示范了 6.1 第三问的分工:**关系归谁,谁发布**——缝的 Definition 不发布,策略包发布。

### 6.7 写法模板与配套三件

```typescript
// src/invariant.ts
/** Package-owned <关系名> invariants. @module <pkg>/invariant */
import type { Context } from '@deepseek-ai/cordis'
import type { InvariantFailure, InvariantInstaller } from '@deepseek-ai/dsh-invariants'

const PACKAGE_NAME = '@deepseek-ai/dsh-<pkg>'
/** Cordis companion plugin name. */
export const name = '<pkg>-invariant'
/** Service required before the companion can reserve package ownership. */
export const inject = ['invariants']

const install: InvariantInstaller = Object.assign((ctx: Context, fail: InvariantFailure) => {
  // 只写"本包契约允许偏离、且偏离可被独立观测"的关系
}, { inject: ['<本包需要的服务名>'] })

export const apply = (ctx: Context): Promise<() => void> =>
  Promise.resolve(ctx.invariants.register(PACKAGE_NAME, install))
```

1. `package.json` 增加 `"./invariant": { "types": "./lib/types/invariant.d.ts", "default": "./lib/invariant.js" }`(`packages/subagent/subagent/package.json:25`),`files` 增加 `lib/invariant.js`(`docs/cookbook/adding-a-package.md:25`);
2. `tsconfig.json` 仅在发布 `./invariant` 时引用 `runtime-diagnostics/invariants`(`packages/AGENTS.md:23`);
3. 在**某个组合的一行**里挂载它,否则永不执行。

---

## 第七节 四条缝的横向对照

| 维度 | `ctx.sandbox` | `ctx.llm` | `ctx.subagents` | `ctx.sessionPersistence` |
|---|---|---|---|---|
| Definition 包 | `sandbox/sandbox` | `llm/llm`(折叠 Consumer) | `subagent/subagent` | `session/session-persistence` |
| 契约形态 | 抽象类 | 抽象类 + 注册表运行时 | 具体注册表类(`TypertRemoteService`) | 抽象类 |
| 必实现方法数 | 1(`confine`) | 1(`stream`) | 0(Provider 是数据对象,注册进 Map) | 5(`create`/`open`/`flush`/`stat`/`list`) |
| Provider 数(出厂) | 1(`sandbox-local`,内部多 runner) | 2 | 6(`spawn`/`fork`/`acp`/`dsh-sdk`/`codex`/`claude-code`) | 2(`jsonl`;`sqlite` 检索侧) |
| 注册方式 | 服务子类构造即注册 | `registerAdapter(...)` | `registerProvider(...)` | 服务子类构造即注册 |
| 逐调用参数 | `SandboxPolicy` | `GenerateOptions` | `SubagentStartRequest` | `SessionHandle` 承载会话身份 |
| 主要错误形态 | `SandboxUnavailableError`(`SANDBOX_UNAVAILABLE`) | `LlmError` | `SubagentError` | 6 个语义化错误类(`errors.ts`) |
| 可替换性证明 | 换 backend 不动 `dsh-tool-bash` 的 schema | 换 adapter 不动主循环 | 换 provider 不动 `tool-subagent` 的 schema | 换 backend 不动 agent-loop 的 resume |
| 发布 `./invariant` | 否(Definition 不发布;`sandbox-policy` 发布) | 是(`llm/package.json:21`) | 是(`subagent/package.json:25`) | 否,README 写明理由(`README.md:99`) |
| 可选服务读取 | `ctx.get('sandbox')`(`terminal-bash:103`) | —— | —— | `ctx.get('sessionPersistence')`(`agent-loop:429`) |

---

## 关键文件/符号索引表

| 符号 | 位置 | 角色 |
|---|---|---|
| `SandboxProvider` / `confine` / `SandboxPolicy` / `ConfinedArgv` / `SANDBOX_UNAVAILABLE` | `packages/sandbox/sandbox/src/index.ts:158`、`:175`、`:69`、`:95`、`:124`、`:131` | Definition 类、唯一契约方法、逐调用策略、证据词汇、失败关闭错误通道 |
| `LocalSandboxProvider` | `packages/sandbox/sandbox-local/src/index.ts:250` | Provider(`static Config` 在 `:252`) |
| `bash-sandbox` / `pwsh-sandbox` 包装点 | `packages/shell/bash-sandbox/src/index.ts:180`、`packages/shell/pwsh-sandbox/src/index.ts:185` | Consumer |
| `terminal-bash` 可选读取 | `packages/terminal/terminal-bash/src/index.ts:103-108` | Consumer(`ctx.get`) |
| `LlmAdapter` / `abstract stream` / `LlmRuntime` / `registerAdapter` | `packages/llm/llm/src/index.ts:200`、`:281`、`:333`、`:387` | Definition 与注册点(`prepareRoutes` `:423`、`commitRoutes` `:454-462`) |
| `llm-deepseek` 插件头 / `apply` / 注册 / `replace` / thunk 注入 | `packages/llm/llm-deepseek/src/index.ts:84-85`、`:421`、`:492`、`:494-504`、`:471-486` | Provider |
| agent-loop 消费 `ctx.llm` | `packages/core/agent-loop/src/agent.ts:390`、`:541`;`src/index.ts:360` | Consumer |
| `SubagentRuntime` / `registerProvider` / `start`;六个 Provider 注册点;`tool-subagent` 消费点 | `subagent/subagent/src/index.ts:188`、`:509`、`:556`;`spawn-in-process:69`、`fork-in-process:95`、`acp:205`、`dsh-sdk:199`、`codex:135`、`claude-code:151`;`tool-subagent/src/index.ts:357`、`:530`、`:550`、`:563` | Definition / Provider / Consumer 全景 |
| `SessionPersistence` / 五个抽象方法 / 类级 JSDoc / `JsonlSessionPersistence` | `packages/session/session-persistence/src/index.ts:135`、`:147`/`:162`/`:175`/`:191`/`:198`、`:115-134`;`session-persistence-jsonl/src/index.ts:235` | Definition 契约条款与 Provider |
| 不发布 invariant 的理由 / agent-loop resume 可选读取 | `packages/session/session-persistence/README.md:99`;`packages/core/agent-loop/src/index.ts:429`、`:444-445` | 判定反例;Consumer |
| `InvariantFailure` / `InvariantInstaller` / `InvariantRegistry.register` | `packages/runtime-diagnostics/invariants/src/index.ts:29`、`:32-42`、`:136-188` | 伴随插件契约与装载机制 |
| `subagent` / `tools` / `agent-loop` / `sandbox-policy` 伴随插件 | `subagent/src/invariant.ts:22`;`core/tools/src/invariant.ts:33`;`core/agent-loop/src/invariant.ts:19`;`sandbox/sandbox-policy/src/invariant.ts:11` | 四个 invariant 样本 |
| invariant 挂载行 / `./invariant` 发布格式 | `packages/bundle/sdk-minimal/cordis.patch.yml:103-116`;`packages/subagent/subagent/package.json:25` | 组合层落点与打包约定 |
| 发布判定规则 | `packages/AGENTS.md:19`、`AGENTS.md:107` | 规则出处 |

---

> 下一篇:[04 · 全仓扩展点目录](./04-extension-points-catalog.md)——本篇的四条缝之外,内核各阶段还留了哪些挂钩子。
