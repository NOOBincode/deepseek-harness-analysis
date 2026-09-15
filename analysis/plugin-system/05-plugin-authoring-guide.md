# 05 · 插件编写指南(以真实插件为模板)

> 分析对象:[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) @ `dbbaa4a37`
> 模板来源:`packages/subagent/subagent-spawn-in-process/src/index.ts`(函数插件)、`packages/llm/llm-deepseek/src/index.ts`(带 Config 与注册句柄的函数插件)、`packages/sandbox/sandbox/src/index.ts`(服务插件的 Definition 侧)
> 事故来源:`docs/postmortem/0001-acp-default-export-drops-inject.md`(真实线上事故复盘)
> 前置:[03](./03-capability-seam-anatomy.md)(选哪条缝)、[04](./04-extension-points-catalog.md)(挂哪个点)、[01](./01-cordis-runtime-internals.md)(effect 与 inject 的机制)。

---

## 第〇节 五条硬规则(先记这个)

| # | 规则 | 出处 |
|---|---|---|
| 1 | **服务包默认导出服务类;函数插件具名导出 `name`/`inject`/`Config`/`apply`,并且不得有 default export。** 混用会让 Loader 丢掉函数插件的 namespace。 | `packages/AGENTS.md:5` + postmortem 0001 |
| 2 | **可选服务用 `ctx.get(name)`,不要用 `ctx.<name>`。** 属性代理是"祖先上溯"的,跨 shadow 读兄弟服务会抛错。 | `packages/AGENTS.md:6` + postmortem 0001 |
| 3 | **一切注册都是 effect**:`ctx.effect()` / `ctx.on()` / 各 registry 的 `register()` 返回 disposer。 | `AGENTS.md:106` |
| 4 | **部署相关的可变量必须是经过校验的 `Config` 字段**,不是 `DEFAULT_*` 常量、不是测试钩子。 | `AGENTS.md:116` |
| 5 | **产品可见插件必须有经真实 Loader + app/进程启动的 REAL-composition 测试**;手搓 `ctx.plugin(...)` 的用例不算。 | `packages/AGENTS.md:7` |

---

## 第一节 两种插件形态

### 1.1 函数插件(绝大多数)

一个完整、可照抄的最小形态:

```typescript
// packages/subagent/subagent-spawn-in-process/src/index.ts(全文 70 行,此处节选)
import type { Context } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'
import type { ContinuableCreateSpec, ResolvedSubagentStartRequest, SubagentCapabilities, SubagentProvider } from '@deepseek-ai/dsh-subagent'
import { startInProcessRun } from '@deepseek-ai/dsh-subagent-in-process-driver'

export const name = 'subagent-spawn-in-process'
// `tools` is deliberately not injected: the child factory already provides it during setup,
// and adding it here would unnecessarily change this provider's apply timing.
export const inject = ['subagents']

/** Config: the registry name to register the provider under. */
export interface Config {
  /** Provider name on `ctx.subagents` (default `spawn`). */
  providerName: string
}

export const Config: z<Config> = z.object({
  providerName: z.string().default('spawn'),
})

/** class SpawnInProcessProvider implements SubagentProvider(:41):capabilities 全开、
 *  inheritsParentContext = false、start() 委托 startInProcessRun、prepareContinuable() 返回 {} */
class SpawnInProcessProvider implements SubagentProvider { /* … */ }

export function apply(ctx: Context, config: Config): void {
  ctx.subagents.registerProvider(new SpawnInProcessProvider(config.providerName))
}
```

四个导出槽位的契约:

| 导出 | 类型 | 谁读它 | 缺省行为 |
|---|---|---|---|
| `name` | `string` | `RegistryService.plugin` → `Plugin.Runtime.name`(`vendor/cordis/src/registry.ts:326`);Loader 日志与 `fiber.name`(`fiber.ts:336-343`)用它 | 无名 fiber 在诊断里显示 `'root'`(向上找最近的有名祖先) |
| `inject` | `string[]` 或 `{ [name]: interceptConfig }` | `Inject.resolve`(`registry.ts:71-88`)→ 成为 `Fiber.inject`,决定 epoch 与激活时机 | 空 inject ⇒ 依赖齐备,fiber 立即 `_reload()` |
| `Config` | **Standard Schema 校验器**(不是普通对象) | `resolveConfig`(`fiber.ts:50-62`)在 `_reload` 里对原始 config 校验 | 无 `Config` ⇒ config 原样传入 |
| `apply` | `(ctx, config) => any` | `Fiber._runner.execute`(`fiber.ts:250-261`) | —— |

`name` 之外,**`apply` 返回的任何值都被忽略**(不同于类插件会 new + init,见 [01](./01-cordis-runtime-internals.md)第三节)。持久贡献只能通过 effect。

两个来自真实代码的写法要点:

1. **`inject` 要写"本插件真正需要的最小集",并且要注释为什么不多写**。上面 20-21 行的注释解释为何不 inject `tools`:"adding it here would unnecessarily change this provider's apply timing"——因为 `inject` 决定激活时机,多写一个服务就会让插件晚挂。
2. **`Config` 接口与同名 `Config` 常量并存**:接口给消费者类型,常量给 Cordis 校验器(`docs/cordis-tutorial/05-config.md:34`:"consumers get the type, Cordis gets the validator")。类型标注用 `z<Config>`(harness 惯例)或 `Schema<Config>`;二者都指 schemastery 的产物类型。

### 1.2 服务插件(提供新能力的包)

服务包的形态是**默认导出服务类**:

```typescript
// packages/sandbox/sandbox/src/index.ts:146-176(节选)
declare module '@deepseek-ai/cordis' {
  interface Context {
    sandbox: SandboxProvider
  }
}

export abstract class SandboxProvider extends Service {
  /* v8 ignore next -- abstract service construction is covered through concrete provider packages. */
  constructor(ctx: Context) {
    super(ctx, 'sandbox')
  }

  abstract confine(argv: readonly string[], policy: SandboxPolicy): ConfinedArgv
}

export default SandboxProvider
```

四件必做:

1. **`declare module '@deepseek-ai/cordis'` 合并 `Context`**,让所有插件获得 `ctx.sandbox` 的类型;这是"服务的 `ctx` 键"的唯一真源(见 `docs/AGENTS.md` 的 type-equiv 规则)。
2. **`super(ctx, '<key>')` 在构造函数里完成注册**(`service.ts:57`,`ctx.reflect.provide(name, self, this[Service.check])`)。若服务"注册了但还不能用",实现 `static [Service.check]()`(范例:`vendor/loader/src/index.ts:166-170`)。
3. **`export default <服务类>`**。
4. **词汇类型必须从本包导出**(`SandboxPolicy`、`ConfinedArgv` 等),因为 Consumer 不允许 import Provider 的包——这是缝的边界(`packages/AGENTS.md:10`)。

对比表:

| 维度 | 函数插件 | 服务插件 |
|---|---|---|
| 导出形式 | 具名 `name`/`inject`/`Config`/`apply` | `export default class X extends Service` |
| Loader 归一化后拿到什么 | 模块 namespace 本身 | 类(`unwrapExports` 取到 default) |
| 注册动作 | `apply` 内调用某个服务的 `register*` | 构造函数的 `super(ctx, key)` |
| 能否有 `Config` | 能(`apply` 第二参数) | 能(`static Config`,构造时由 Cordis 校验) |
| 真实样本 | `subagent-spawn-in-process`、`llm-deepseek`、`hooks-claude-code` | `sandbox`、`session-persistence`、`subagent`(注册表类)、`compaction` |

**混用即事故**——这就是第四节 postmortem 的 Bug #1。

---

## 第二节 `Config`:校验、默认值与"无硬编码可调项"

### 2.1 校验发生在 `apply` 之前,失败即 FAILED

```typescript
// vendor/cordis/src/fiber.ts:50-62
export function resolveConfig(runtime: Plugin.Runtime, config: any) {
  if (!runtime.Config) return config
  // TODO: async validation
  const result = runtime.Config['~standard'].validate(config)
  if ('then' in result) {
    throw new TypeError('Async config validation is not supported')
  }
  if (result.issues) {
    throw new ValidationError(result.issues)
  } else {
    return result.value
  }
}
```

`ValidationError`(`fiber.ts:19-36`)把 issues 拼成多行消息(带 `at <path>`)。调用链是 `_reload` → `_resolveConfig`(`fiber.ts:641-644`):

```typescript
private _resolveConfig(config: any) {
  config = this.context.waterfall(this, 'internal/config', config, () => config)
  return this.runtime ? resolveConfig(this.runtime, config) : config
}
```

注意顺序:**先过 `internal/config` waterfall(Loader 在这里做 `!!js` 插值),再做 Schema 校验**。所以 `!!js` 求值结果也受 schema 约束;而配置校验失败会落进 `_reload` 的 catch(`fiber.ts:659-663`):`_error` 被记录、epoch 归 INACTIVE、fiber 变 FAILED,`await()` 把它抛出给 `Entry._await()`,最终由 `assertEntriesActivated` 汇总成启动失败。

### 2.2 写 `Config` 的三条实践

1. **每个字段都有 JSDoc**,因为生成的配置目录(`docs/config-catalog.md`)从这里取描述;仓库对公开配置项的文档有门控。
2. **默认值写在 schema 里**(`z.string().default('spawn')`),而不是 `apply` 里的 `?? 'spawn'`——这样 `--dump-config` 与文档目录能看到默认值。
3. **可写回的配置要实现 `simplify`**:Loader 在条目更新时用它把运行时配置转回用户形式:

   ```typescript
   // vendor/loader/src/index.ts:103-109
   ctx.on('internal/update', async function (config, noSave, next) {
     if (!this.entry || noSave || this.parent.fiber?.entry === this.entry) return next()
     await next()
     const unparse = this.runtime?.Config?.['simplify']
     this.entry.options.config = unparse ? unparse(config) : config
     this.entry.parent.tree.write()
   }, { global: true, prepend: true })
   ```

   没有 `simplify` 时,运行时态的配置会被原样写进 `cordis.patch.yml`——如果 `apply` 过程中 config 被解析/展开过,写回的就是展开态。

### 2.3 "No hardcoded tunables"

`AGENTS.md:116` 的原文规则:

> **No hardcoded tunables in plugins**: deployment-varying choices are validated `Config` fields changeable from cordis.yml; a `DEFAULT_*` constant or test hook is not configurability. Protocol constants, external specs, and security invariants stay fixed.

判定"该不该做成配置"的两问:

| 问题 | 是 | 否 |
|---|---|---|
| 部署之间会不同吗? | 必须是 `Config` 字段 | 留常量 |
| 它是协议常量 / 外部规范 / 安全不变量吗? | 必须固定为常量 | 可以是配置 |

配套规则是 `AGENTS.md` 的 **"Misconfiguration fails loud"**:自包含的配置错误在**加载期**失败;不能自包含的(如引用了不存在的 provider 名)在**最早可解析的点**失败,绝不静默跳过(`docs/cordis-tutorial/05-config.md:68`)。

一个真实的"边界拆分"范例是 `LocalSandboxProvider` 的配置:

```typescript
// packages/sandbox/sandbox-local/src/index.ts:250-256
export class LocalSandboxProvider extends SandboxProvider {
  // Inline schema call: the config catalog walks `static Config` statically.
  static Config: z<Config> = z.object({
    runnerCommand: z.array(z.string()).default([]),
    runnerFailureSignatures: z.array(z.string()).default([]),
    probeTimeoutMs: z.natural().default(5_000),
  })
```

runner 命令与探测超时**是**配置(随宿主机不同);而"沙箱模式词汇只有三个值"、"策略逐调用携带"是契约,固定不动(`packages/sandbox/sandbox/src/index.ts:29`、`:61-68`)。

---

## 第三节 effect 处置与 HMR 安全

### 3.1 贡献必须挂在当前 fiber 上

```typescript
// packages/subagent/subagent/src/index.ts:502-525(节选)
registerProvider(provider: SubagentProvider): () => void {
  const name = provider.name
  return this.ctx.effect(function* (this: SubagentRuntime) {
    if (this.providers.has(name)) {
      throw new SubagentError(`a subagent provider named "${name}" is already registered`, 'DUPLICATE_PROVIDER')
    }
    this.providers.set(name, provider)
    yield () => { this.providers.delete(name); this.emitLifecycle('subagent/provider-removed', name) }
    this.ctx.emit('subagent/provider-added', provider)
  }.bind(this), 'subagents.registerProvider()')
}
```

三条可迁移的模式:

1. **注册项与 disposer 成对出现,注册体现在时执行、清理逻辑写在 `return`/`yield` 里**。`providers.set` 与 `providers.delete` 必须成对——这就是 [01](./01-cordis-runtime-internals.md)第四节"生命周期 = 注册的反向回放"在业务层的形态。
2. **effect 带 label**(`'subagents.registerProvider()'`),`getEffects()` 才能给出可读诊断。
3. **在 effect 内抛错是安全的**:fiber 会把已收集的 disposer 回滚(`fiber.ts:523-537`),错误经 `await()` 抛给加载方。

### 3.2 三种常见处置需求与其范式

| 需求 | 范式 | 真实样本 |
|---|---|---|
| 一个注册句柄要被后续代码主动调用(不只是 fiber 卸载时清理) | registry 返回一个**注册句柄对象**而非裸 disposer | `ctx.llm.registerAdapter` 返回 `AdapterRegistrationHandle`,可 `.replace([PROVIDER])`(`llm/src/index.ts:387`;使用见 `llm-deepseek/src/index.ts:492-503`) |
| 注册体需要在"原子区段"内整体换掉候选集 | 先 `prepare*` 校验、再 `commit*` 一次同步交换 | `llm/src/index.ts:423`、`:454-462` |
| 需要对外部资源(watcher、子进程)负责 | 返回的 disposer 写成 **async 且等到 quiescence** | `Hmr.registerConfig` 的 disposer(`vendor/hmr/src/index.ts:177-181`):`await watcher.close()` 之后 `await this.configRefreshes.get(registration)?.running` |

最后一条对应 `docs/defensive-patterns.md:19-21` 的规则:"A teardown that issues kills/aborts but returns before the work stops leaves orphans."

### 3.3 HMR 安全是被测试强制的

`packages/AGENTS.md:17`:

> **Registry contributions prove disposal** through the HMR-safety test required by [testing policy](https://github.com/deepseek-ai/deepseek-harness/blob/dbbaa4a37fb9098aba814c97d2956f7b2f105f46/docs/testing.md): dispose the fiber and observe removal.

写法(以注册表类插件为例):

```text
1. 挂载插件,注册一项
2. 断言注册表里有它
3. 处置承载它的 fiber(或在测试里触发 HMR 等价的 dispose)
4. 断言注册表里没有它、监听器不再触发、外部资源已关闭
5. 若它是"服务已注册但不可用"的 check,断言依赖者在服务消失后回到 PENDING
```

反例(会被判为无效测试):只断言 `dispose()` 返回了 Promise;只断言服务对象还存在(没断言注册表键被删);只测 effect 存在而不测回收。

### 3.4 可选服务:永远 `ctx.get(name)`

```typescript
// packages/llm/llm-deepseek/src/index.ts:450-452
const credentials = ctx.get('credentials')
if (credentials !== undefined) {
  const hit = await credentials.resolve(ref)
  ...
}
```

`ctx.get(name)` 走 `ReflectService.get` → `_getImpl`(`reflect.ts:233-243`),按 isolate 符号查全局 store,**默认 strict**:provider fiber 不在 ACTIVE 时返回 `undefined`,而不是把一个正在拆卸的服务交回来。`ctx.<name>` 则走祖先上溯,跨 shadow 时会抛 `cannot get property ... without inject`。

---

## 第四节 事故复盘:`export default apply` 丢掉了 `inject`

来源:`docs/postmortem/0001-acp-default-export-drops-inject.md`。这是本仓库唯一被记录为"178 个单测全绿、100% 覆盖率、真实编辑器一连就崩"的事故,值得完整走一遍。

### 4.1 现象

ACP 服务器(`dsh --profile acp`)在真实编辑器(Zed)连接瞬间崩溃:第一个 `session/new` 返回 `Internal error: cannot get property "agents" without inject`;修好后又发现 `session/load` 返回同类错误(但名字是 `sessionPersistence`)。

### 4.2 Bug #1:多写的一行 `export default apply`

```typescript
// docs/postmortem/0001-acp-default-export-drops-inject.md:31-37(事故代码)
export const name = 'acp'
export const inject = ['agents', 'sessions', 'sessionPersistence']
export function apply(ctx: Context, config: AcpConfig): void { /* … */ }
// …
export default apply   // ← the bug
```

机制在 Loader 的归一化函数里:

```typescript
// vendor/loader/src/index.ts:191-199
/** Normalize ESM/CJS/default export shapes before applying a plugin. */
unwrapExports(exports: any) {
  if (isNullable(exports)) return exports
  exports = exports.default ?? exports
  // https://github.com/evanw/esbuild/issues/2623
  // https://esbuild.github.io/content-types/#default-interop
  if (!exports.__esModule) return exports
  return exports.default ?? exports
}
```

`exports.default ?? exports` 优先取 `.default`,于是 Loader 拿到的是**裸 `apply` 函数**——`name`/`inject`/`Config` 作为 sibling 具名导出被整体丢弃(`Entry._init` 调 `this.loader.unwrapExports(await this.parent.tree.import(...))`,`loader/src/config/entry.ts:277-283`)。空 `inject` 的 fiber 不会等 `agents`,`apply` 第一行读 `ctx.agents` 就走到根 fiber 抛错。

**修复**:删掉 `export default apply`。

### 4.3 Bug #2:可选服务用了属性读,跨 shadow 上溯失败

`session/load` → `agents.resume(...)` → `AgentLoop.resume()`,后者读 `this.ctx.sessionPersistence`。但 `AgentLoop` 的 `static inject` **刻意不含** `sessionPersistence`(否则非持久化 demo 会永远 PENDING)。服务由兄弟分支提供,于是祖先上溯走到根 fiber 抛错。

```typescript
// docs/postmortem/0001-acp-default-export-drops-inject.md:64-75(引用 reflect.ts 的 get handler)
let fiber = (ctx[symbols.shadow] as Context ?? ctx).fiber   // ← starts at AgentLoop's fiber
while (true) {
  const impl = fiber.store?.[prop]
  if (impl) return getTraceable(ctx, impl.value)
  if (prop in fiber.inject) { /* inactive-context error */ }
  if (!fiber.runtime) throw error                            // ← reached root, throw
  if (fiber.parent[symbols.isolate][prop] !== key) throw error
  fiber = fiber.parent.fiber                                 // ← ancestor-only
}
```

**修复**:改用 `ctx.get('sessionPersistence')`。落点在今天的源码里仍可见:

```typescript
// packages/core/agent-loop/src/index.ts:429
const persistence = sessionId === undefined ? undefined : ctx.get('sessionPersistence')
// packages/core/agent-loop/src/index.ts:444-445
const fiber = ctx.inject(['sessionPersistence'], (childCtx: Context) => {
  void this.resumeWith(ctx, childCtx.sessionPersistence, {
```

注意第二种写法(`ctx.inject([...], cb)`)才是"我确实需要它、需要等它就绪"的正确表达;`ctx.get` 用于真正可选的读取。

### 4.4 为什么测试全绿

| 测试 | 为什么漏掉 |
|---|---|
| 内存 harness 手搓 `ctx.plugin({ name, inject, apply })` | `inject` 是手动给的,`unwrapExports` 根本不在路径上 ⇒ 永远复现不了 Bug #1 |
| 同上,全部平铺在一个 root context | `resume` 要么顶层调用(走 `!runtime` 旁路直接全局查),要么 shadow 起点仍在 root ⇒ 掩蔽 Bug #2 |
| 唯一无密钥 e2e 只发 `initialize` 并检查 stdout 纯净 | `initialize` 到不了 factory |
| 唯一驱动 `session/new`/`session/load` 的测试是 key-gated | CI 无 key 直接跳过 |

结论(postmortem 原话):"Coverage proves lines *ran*; it says nothing about whether the feature works *the way it ships*."

### 4.5 事故后加的四道护栏(可照抄)

1. 删除 `export default apply`;
2. `AgentLoop.resume` 改用 `ctx.get('sessionPersistence')` 并留下解释 shadow 陷阱的注释;
3. **无密钥 `session/new` e2e**,经真实 stdio 启动 profile,断言 `session/new` 成功——恢复 `export default apply` 时该测试必须失败;
4. e2e 子进程显式传 `TSX_TSCONFIG_PATH`,避免 tsx 从临时 cwd 找不到仓库 tsconfig 而静默落到 `lib/` 旧构建。

---

## 第五节 测试与门:插件作者要过哪些关

| 门 | 要求 | 出处 |
|---|---|---|
| per-file 100% 覆盖率 | `pnpm run test:coverage` 对 `packages/*/*/src` **逐文件** 100% | `docs/testing.md:10` |
| HMR-safety | 每个注册表必须有"处置 fiber 并观察移除"的测试 | `docs/testing.md:9`;`packages/AGENTS.md:17` |
| REAL-composition | 产品可见插件必须经真实 `cordis.yml` + Loader + app/进程启动;只 mock 昂贵/非确定边界 | `docs/testing.md:39`;`packages/AGENTS.md:7` |
| snapshot 回放 | 任何 model-/protocol-/human-visible 变更,在同一 PR 增改一个无密钥录制会话场景 | `docs/testing.md:53-55` |
| invariant | 发布 `./invariant` 时,`verify-package-invariants` 要求非空 installer、检查的是本包拥有的关系 | `packages/AGENTS.md:19` |
| 文档同步 | README + JSDoc 与行为同 PR 更新;`doc-sync` 30+ 门 | `packages/AGENTS.md:26`;`scripts/run-gates.ts:715-770` |
| Agent Note | 非平凡变更必须在同一 PR 携带 Agent Note | `AGENTS.md:126` |
| 双 SDK 投影 | 改 agent-loop / 会话生命周期 / `SessionEventMap` 时,TS 与 Python SDK 的期望输出同 PR 更新 | `AGENTS.md` 的 "Both SDKs project the loop" |

**REAL-composition 测试的最小形态**(postmortem 护栏 3 的形态):写一份 test-only 的 `cordis.yml`,用真实 Loader 启动,驱动"不调用模型也能验证"的那个操作(`session/new`、工具注册、配置解析),断言可观察输出。这样它可以无密钥跑在 CI 里。

---

## 第六节 常见错误清单

| 症状 | 根因 | 修法 |
|---|---|---|
| `cannot get property "<name>" without inject` | ① 插件有 `export default`,namespace 被丢;② 读的是没声明进 `inject` 的服务 | 删 default export;把该名字加进 `inject`,或改 `ctx.get(name)` |
| `cannot get required service "<name>" in inactive context` | 声明了 inject 但 provider 未 ACTIVE(或已下线) | 检查 provider 的条目是否存在/是否被 `disabled` 门控;用 `dsh --profile X --dump-config` 看条目表 |
| 插件挂上了但功能不生效 | fiber 停在 PENDING | `assertEntriesActivated` 的报错会点名缺失服务;或看 `internal/status` |
| 改 `cordis.patch.yml` 没反应 | profile 是 `patchReload: 'startup'`,或没挂 HMR | `docs/architecture.md:29`;`apps/cli/src/profile-boot.ts:355-371` |
| 用户覆盖删掉后行为回不到 bundle 默认 | 把同一个解析出的 patch 对象重复 apply(insert 行按引用入树) | 每代重新 `structuredClone`(`profile-boot.ts:323-333`) |
| HMR 后旧注册还在 | 注册没走 `ctx.effect()`/`ctx.on()`,或用了模块级变量 | 让 `register()` 返回 effect disposer,并补 HMR-safety 测试 |
| 配置校验失败信息看不懂 | 没写 `Config` schema 或字段无描述 | 补 `Config` + 字段 JSDoc;失败消息由 `ValidationError` 生成多行 issues |
| 测试全绿但真实启动崩 | 只用手搓 `ctx.plugin(...)` | 补 REAL-composition 无密钥用例(第五节) |
| `waterfall` 之后的逻辑不执行 | 某个监听器没调 `next()` | `AGENTS.md:110`;检查自己与依赖的监听器 |
| 服务重复注册导致加载失败 | 同一 scope 挂了两个同 key provider | `reflect.ts:289-291` 的报错会点名先占者;用 `isolate` realm 或删一行 |
| `INACTIVE_EFFECT` | 在 UNLOADING 的 fiber 上新建 effect,或在树已处置后注册 watcher | `fiber.ts:420-422`;启动器侧对 HMR 注册失败做了专门容忍(`app-boot/src/index.ts:274-280`) |

---

## 第七节 从零写一个插件:操作序列

```text
0. 判定角色:"新增能力"(Service Definition + Provider)还是"给既有流程加一段"(监听扩展点)?
   → 前者照 03 的三角色;后者照 04 选点。
1. 建包 packages/<group>/<pkg>/:private、version 与根一致、type: module、main lib/index.js、
   types lib/types/index.d.ts、@deepseek-ai/cordis 同时在 peerDependencies 与 devDependencies、
   每个 dsh peer 在 devDependencies 也有一份、@deepseek-ai/schemastery 放 dependencies
   (docs/cookbook/adding-a-package.md:25)
2. 写 src/index.ts:选形态并严格遵守 §1 的导出规则(不得混用);name 用包短名;inject 只写真正
   需要的并注释为什么;部署相关值 → Config(Schemastery + JSDoc + default);贡献全部走
   ctx.effect()/ctx.on();外部资源在 disposer 里等到 quiescence;契约词汇从 Definition 包 import 类型
3. 写 README:契约、配置、限制、Model Experience 段、Known Limitations(packages/AGENTS.md:26-28)
4. 写测试:行为分支单测;HMR-safety(dispose 后观察移除);REAL-composition(test-only cordis.yml
   经真实 Loader/app 启动);改动 model-visible 输出时更新无密钥 snapshot 场景
5. 若拥有"两个独立可观测事实"的关系 → 加 src/invariant.ts + ./invariant 导出(模板见 03 §6.7);
   否则在 README 写明为何不发布
6. 加组合行:在相应 bundle 的 cordis.patch.yml 里 insert 一行(id + name + config);行序无语义,
   但 id 必须唯一(重复 id 在挂载前被拒绝:group.ts:64);跑 pnpm run test:coverage / doc-sync
```

---

## 关键文件/符号索引表

| 文件 | 内容 |
|---|---|
| `packages/AGENTS.md:5-7` | 导出规则、`ctx.get` 规则、REAL-composition 要求 |
| `packages/AGENTS.md:10,17,19,23,26` | Service Definition 设计、HMR-safety、invariant 发布、tsconfig、README |
| `AGENTS.md:106-116,126` | 注册即 effect、invariant 规则、事件 JSDoc、waterfall、no hardcoded tunables、Agent Note |
| `vendor/loader/src/index.ts:191-199` | `unwrapExports`:default export 丢 namespace 的机制 |
| `vendor/cordis/src/registry.ts:71-88,300-336` | `Inject.resolve`、`plugin()` 的形态归一化 |
| `vendor/cordis/src/fiber.ts:19-62` | `ValidationError` 与 `resolveConfig`(配置校验落点) |
| `vendor/cordis/src/fiber.ts:250-261,415-561` | 构造函数插件分支、`effect()` 全程 |
| `vendor/cordis/src/fiber.ts:641-673` | `_resolveConfig` 与 `_reload` 的失败处理 |
| `vendor/cordis/src/reflect.ts:136-171,233-243` | 属性上溯 vs `ctx.get` 的全局查 |
| `vendor/loader/src/index.ts:103-109` | 配置写回与 `Config['simplify']` |
| `packages/subagent/subagent-spawn-in-process/src/index.ts:19-70` | 函数插件最小模板 |
| `packages/llm/llm-deepseek/src/index.ts:84-85,187,421,492-503` | 带 Config、注册句柄、`replace` 的完整样本 |
| `packages/sandbox/sandbox/src/index.ts:146-176` | 服务插件 Definition 模板 |
| `packages/sandbox/sandbox-local/src/index.ts:250-256` | `static Config` 与配置目录的关系 |
| `packages/subagent/subagent/src/index.ts:502-525` | effect 注册与回滚范式 |
| `vendor/hmr/src/index.ts:134-187` | 外部资源 disposer 的 quiescence 写法 |
| `packages/core/agent-loop/src/index.ts:429,444-445` | 可选服务两种读法的正确落点 |
| `docs/postmortem/0001-acp-default-export-drops-inject.md` | 事故全文:`:31-54` Bug #1、`:56-88` Bug #2、`:89-106` 为何漏测与护栏 |
| `docs/cordis-tutorial/05-config.md` | Config 教程与 `!!js` 边界(`:72-80`) |
| `docs/cordis-tutorial/02-lifecycle-and-effects.md` | effect 入门 |
| `docs/testing.md:9,10,39,53-55` | 覆盖率门、HMR-safety、REAL-composition、snapshot 纪律 |
| `docs/cookbook/adding-a-package.md:25,117` | package.json/tsconfig 约束与测试要求 |

---

> 回到模块首页:[plugin-system/README.md](./README.md)
