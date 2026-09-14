# 插件设计(plugin-system)模块

> 分析对象:[innokria/deepseek-harness](https://github.com/innokria/deepseek-harness) @ `dbbaa4a37`
> 与第一章、第十二章的分工见文末[分工说明](#与第一十二章的分工)。

---

## 一、这个模块回答什么

四类问题在本模块得到闭式回答:

| 问题 | 落在哪一篇 |
|---|---|
| 读 `ctx.tools` 时到底发生了什么?fiber 状态机有哪几个真实状态?`ctx.effect()` 的 disposer 为什么能保证不泄漏? | [01](./01-cordis-runtime-internals.md) |
| `inject` 凭什么决定加载顺序?某个插件 `apply` 抛错后,树上其他条目会怎样?改一行 `cordis.patch.yml` 之后进程内走了哪条路径? | [02](./02-loader-and-composition.md) |
| 能力缝的三个角色在**文件级**如何切开?sandbox/llm/subagent/session-persistence 各自的契约方法、实现、调用点分别在哪一行?`./invariant` 什么时候必须发布、什么时候必须不发布? | [03](./03-capability-seam-anatomy.md) |
| 想在 turn/step/request/tool/compaction/session 各阶段插一脚,能挂钩子的确切位置有哪些?谁是注册方、谁是消费方? | [04](./04-extension-points-catalog.md) |
| 我现在就要写一个插件:导出什么、`Config` 怎么写、effect 怎么处置、测试要过哪些门?最经典的翻车姿势是什么? | [05](./05-plugin-authoring-guide.md) |

---

## 二、五篇导读

| # | 文档 | 一句话 | 主要源码面 |
|---|---|---|---|
| 01 | [Cordis 运行时内部实现](./01-cordis-runtime-internals.md) | Context 代理的属性读写如何落到 service store / reflect;`Service` 构造即注册;fiber 六状态状态机与转换函数;effect 树与逆序回收;registry 与 reflect 的分工;五种事件分发的实现级差异 | `vendor/cordis/src/{context,reflect,service,fiber,registry,events,utils}.ts` |
| 02 | [Loader、Include 与组合层](./02-loader-and-composition.md) | 条目树结构、依赖序并发激活的实现判定、事务化回滚、HMR 精确配置监听与热替换、`!!js` 求值边界、profile/bundle/patch 三层合成 | `vendor/{loader,include,hmr}/src/**`、`packages/boot/app-boot/src/*`、`apps/cli/src/profile-boot.ts` |
| 03 | [能力缝三角色的函数级解剖](./03-capability-seam-anatomy.md) | sandbox / llm / subagent / session-persistence 四条真实缝的三列对照(契约方法 → Provider 实现 → Consumer 调用点)+ invariant 伴随插件的判定与写法 | `packages/{sandbox,llm,subagent,session,core}/*/src/*` |
| 04 | [全仓扩展点目录](./04-extension-points-catalog.md) | 按内核阶段分类的事件、瀑布、hook 点;每项给注册方/消费方与 `路径:行号` | `packages/core/**/src/*`、`docs/event-producer-consumer.md` |
| 05 | [插件编写指南](./05-plugin-authoring-guide.md) | 函数插件 vs 服务插件的导出规则、`name`/`inject`/`Config`/`apply` 契约、Schemastery 校验、no hardcoded tunables、effect 处置与 HMR 安全、REAL-composition 测试与覆盖率门、default export 翻车 postmortem | `packages/**/src/index.ts`、`docs/postmortem/0001-*.md`、`docs/cordis-tutorial/*` |

---

## 三、Cordis 运行时对象关系图

下面这张图是本模块的总纲:每个框里的名字都在 `vendor/` 里有对应类或函数,括号内是定义位置。

![流程图：README](../assets/diagrams/plugin-system__README-49.svg)

<details><summary>Mermaid 源码</summary>

```mermaid
flowchart TB
    subgraph CTX["ctx:Context(被代理的对象) —— context.ts:42"]
        PROXY["Proxy(ctx, ReflectService.handler)<br/>context.ts:74"]
        ROOT["ctx.root = self(全应用唯一根)<br/>context.ts:75"]
    end

    subgraph BUILTIN["四个内建服务(根 fiber 安装)"]
        REFLECT["ReflectService<br/>reflect.ts:133<br/>store:Dict&lt;Impl,symbol&gt; / props"]
        REG["RegistryService<br/>registry.ts:195<br/>_internal:Map&lt;callback,Runtime&gt;"]
        EV["EventsService<br/>events.ts:131<br/>_hooks:Record&lt;name,Hook[]&gt;"]
        LOG["LoggerService<br/>logger.ts"]
    end

    subgraph FIBER["Fiber —— fiber.ts:184(每插件实例一棵)"]
        ST["state:PENDING|LOADING|ACTIVE<br/>|FAILED|UNLOADING|DISPOSED<br/>fiber.ts:147"]
        INJ["inject:Dict(服务名→intercept)<br/>构造自 registry.ts:330"]
        STORE["store:Dict&lt;Impl&gt; —— 已满足的依赖快照<br/>fiber.ts:198"]
        DISP["_disposables:DisposableList<br/>fiber.ts:203(卸载时 reverse 回收)"]
        EPOCH["_runner.epoch:string<br/>全依赖签名,缺一即 INACTIVE<br/>fiber.ts:611"]
    end

    subgraph SVC["Service 子类(服务插件)"]
        SCTOR["constructor:this.ctx.reflect.provide(name,self,check)<br/>service.ts:57 —— 构造即注册"]
    end

    PROXY -->|"属性读:reflect.handler.get<br/>reflect.ts:136"| WALK["从 ctx.fiber 沿 parent 向上找<br/>fiber.store[prop]<br/>reflect.ts:155-166"]
    WALK --> STORE
    REFLECT -->|provide| STORE
    REFLECT -->|notify(names)| FIBER
    REG -->|"plugin():new Fiber(...)"| FIBER
    EV -->|"register():ctx.fiber.effect(...)"| DISP
    SVC --> SCTOR --> REFLECT
    STORE -.->|"依赖齐备 → _refresh() → _reload()"| EPOCH
    EPOCH --> ST
    ST -->|"_unload()"| DISP
    ROOT --> CTX
```

</details>

三条读图要点,后续各篇反复用到:

1. **属性读不走普通原型链**。`ctx.tools` 这类读取被 `ReflectService.handler.get` 截获,按 `fiber.store` 沿 `parent.fiber` 逐级上溯(`reflect.ts:155-166`);走到根 fiber 仍未命中就抛 `cannot get property "tools" without inject`(`reflect.ts:144`)。这就是"必须诚实声明 `inject`"的机械保证。
2. **依赖满足与否被压成一个字符串 epoch**。`_refresh()` 把每个 inject 键解析成 `:<providerUid>` 拼起来,任一键缺席就置 `INACTIVE`(`fiber.ts:611-623`);`INACTIVE → 非 INACTIVE` 触发 `_reload()`,`非 INACTIVE → 其他` 触发 `_unload()`(`fiber.ts:625-639`)。HMR 重挂就是这条边被走了一遍。
3. **一切注册都挂在 fiber 的 effect 列表上**。`ctx.provide()`(`reflect.ts:278`)、`ctx.on()`(`events.ts:256`)、`ctx.mixin()`(`reflect.ts:366`)内部都是 `this.ctx.fiber.effect(...)`;卸载时 `_disposables.clear()` 返回**逆序**数组(`utils.ts:27-31`)逐个 await 处置(`fiber.ts:676-686`)。

---

## 四、与第一、十二章的分工

| 主题 | 第一章(已覆盖) | 第十二章(已覆盖) | 本模块新增 |
|---|---|---|---|
| 启动链 | `bin.ts` → `loadLayeredEnv` → `composeProfile` → `boot()` 全流程、profile 模板表、三个应用形态 | — | **函数级**:`mountRootInclude` 里 builtins 注入与固定 id 的取舍、`composeEntries` = 一次 `applyEntryPatches([], layers.flat())`、`watchUserPatches` 的闭包与 `INACTIVE_EFFECT` 容忍 |
| Cordis 概念 | Context 代理、`Service` 构造即注册、inject/effect/waterfall 各一小段 | `ctx.effect` 实现片段、事件声明合并、五种派发语义表 | **逐函数**:`ReflectService` 三个 trap、`Fiber.effect` 的 wrapper/inertia 全程、状态机的 `_getState`/`_updateState`/`_refresh`/`_setEpoch`、`notify` 的传播算法 |
| Loader/组合 | `applyEntryPatches` 算法、patch 层叠次序、`assertEntriesActivated` | Include 事务化对账一句带过 | **激活与回滚**:`EntryGroup.update` 的并发挂载 + 反向回滚、`Entry.update` 四种分支、`EntryTree.await` 的收敛循环、`Loader[Service.check]` 的 await 门 |
| HMR | 「更常用 `registerConfig()` 精确配置监听」 | accepted/declined 分类与 dispose→reload 概览 | **路径级**:`registerConfig` 的 `findWatchRoot`/depth 计算/返回的 effect disposer、`refreshConfig` 的 dirty+串行合并、`analyzeChanges` 的定点迭代、`partialReload` 的双缓存回滚 |
| 能力缝 | 三角色定义、shell/LLM/Web 三例 | 三角色为何是模式、LLM 抽象、Consumers 约束 | **四条缝的三列对照表** + `./invariant` 发布判定与两个真实伴随插件剖析 |
| 扩展点 | 一张"扩展点表"摘要 | 事件声明合并示例 | **按内核阶段分类的完整目录**,含 `@mode`、注册方、消费方、行号 |
| 写插件 | — | — | **可照抄的模板 + 检查清单 + 真实 postmortem 复盘** |
| 质量工程 | — | 覆盖率门、snapshot、REAL-composition、doc-sync 30+ 门 | 只引用不重述,用于说明"插件作者要过哪些门" |

**阅读依赖**:第一章第三节(启动链)是 02 的前置;第十二章第一节(effect 与事件)是 01 的前置;第十二章第二节(能力缝)是 03 的前置。

---

## 五、建议阅读路径

| 你是谁 | 建议顺序 |
|---|---|
| 想照葫芦画瓢写插件 | [05](./05-plugin-authoring-guide.md) → [03](./03-capability-seam-anatomy.md)(选一条缝仿写)→ [01](./01-cordis-runtime-internals.md)(搞懂 effect 与 inject 为何这样设计) |
| 想替换某能力(换 provider / 换沙箱后端) | [03](./03-capability-seam-anatomy.md) → [02](./02-loader-and-composition.md)(改哪层 patch)→ [05](./05-plugin-authoring-guide.md)(Provider 的测试门) |
| 想加拦截逻辑(审批、重试、遥测) | [04](./04-extension-points-catalog.md) → [01](./01-cordis-runtime-internals.md)(waterfall 语义)→ [03](./03-capability-seam-anatomy.md)(invariant 写法) |
| 在排查"启动失败 / 插件不激活 / 改了配置没生效" | [02](./02-loader-and-composition.md) → [01](./01-cordis-runtime-internals.md)(fiber 状态与依赖判定) |
| 只想拿到全貌 | 本页第三节的图 + [04](./04-extension-points-catalog.md) 的阶段表 |

---

## 六、本模块引用的关键文件(汇总索引)

| 文件 | 在本模块中的角色 |
|---|---|
| `vendor/cordis/src/context.ts` | `Context` 类、代理创建、四种内建服务安装、`extend`/`isolate`/`intercept` |
| `vendor/cordis/src/reflect.ts` | 三个 Proxy trap、服务解析上溯、`provide`/`notify`/`accessor`/`mixin` |
| `vendor/cordis/src/service.ts` | `Service` 基类:构造即注册、`resolveConfig` 的 intercept 合并 |
| `vendor/cordis/src/fiber.ts` | 六状态机、`effect()` 全程、`_checkImpl`/`_refresh`/`_setEpoch`/`_reload`/`_unload` |
| `vendor/cordis/src/registry.ts` | `plugin()`/`inject()`、`Plugin.Runtime`、`Inject.resolve` |
| `vendor/cordis/src/events.ts` | `dispatch` 与五种模式实现、内建 `internal/*` 事件契约 |
| `vendor/cordis/src/utils.ts` | `DisposableList`(逆序 `clear`)、`getTraceable`、`composeError` |
| `vendor/loader/src/index.ts` | `Loader` 服务:`!!js` 惰性插值、`unwrapExports`、`[Service.check]` 依赖门 |
| `vendor/loader/src/config/{entry,group,tree,utils,isolate}.ts` | 条目生命周期、事务化更新与回滚、树遍历与 `await`、`!!js` 求值、isolate realm |
| `vendor/include/src/index.ts` | `entryListSchema`、`applyEntryPatches`、文件型条目树与写回 |
| `vendor/hmr/src/index.ts` | `registerConfig` 精确监听、变更分类、热替换与回滚 |
| `packages/boot/app-boot/src/{index,profile}.ts` | `boot()`、`mountRootInclude`、`composeEntries`、`watchUserPatches`、profile/bundle 解析 |
| `apps/cli/src/profile-boot.ts` | `composeProfile` / `composeLive` / `runProfile` 的层叠与 live 重载 |
| `packages/bundle/*/cordis.patch.yml` | 三层合成里的 bundle 层真实内容 |
| `docs/event-producer-consumer.md` | 生成的事件生产者/消费者矩阵(04 的数据底座) |
| `docs/postmortem/0001-acp-default-export-drops-inject.md` | 插件导出规则的来源事故 |
