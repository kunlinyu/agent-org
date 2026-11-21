# DSH 插件体系术语备忘（T01 定稿）

> **性质**：参考备忘。记录 `03-提案评审清单.md` 阶段一 T01「插件体系与 Entry 组装」的术语定义，供后续评审与实现时对齐用。
> **不构成**设计决策，也不改动需求基线。
> **日期**：2026-11-21
> **依据版本**：本机安装版 DSH（`app.asar` 内 `docs/`）与 `profiles/desktop` 的实际配置

## 使用方式

- 每条定义只能依赖编号更小的定义，**不能**反向依赖更大的编号。
- 编号是稳定引用：讨论时说 S12，即指 `isolate(name, label)`。
- 最后附依赖链；改任一条定义前，先按依赖链检查会影响哪些定义。

## 术语使用约定（唯一叫法）

| 唯一术语 | 指什么 | 禁止使用的同义说法 |
|---|---|---|
| **Entry** | 配置中的一条记录：`id`、`config`、`disabled`，并引用一个 Plugin（S25） | 行、条目、插件行、row |
| **注册项** | 注册表里保存的一条记录（S15–S17） | 条目、注册条目 |
| **定义** | 本文件 S01–S35 中的一项 | 条目、第 N 条 |
| **plugin package** | 导出 Plugin 的 npm 包（S29） | 插件包（中文混用） |
| **bundle** | 自带一份 patch 的 npm 包（S30） | 插件组、插件集 |
| **profile** | 用户目录（S31） | 环境、配置集 |
| **fiber** | 一次加载的运行时实例（S18） | 实例、进程 |
| **composition** | 多层 patch 作用后的全部 Entry（S27） | 插件集合、插件列表、**"profile 组合"** |
| **组装** | Loader 把单个 profile 的 bundle 补丁与用户补丁叠加成 composition 的动作 | "profile 组合"、"多 profile 合并" |

---

## 一、术语定义（S01–S35）

**S01　Cordis**：DSH 所依赖的框架。

**S02　Plugin**：Cordis 所处理的单元，形态为类或函数。

**S03　加载（load）**：Cordis 使一个 Plugin 进入运行状态的动作。

**S04　Context**：Cordis 加载一个 Plugin 时传给它的对象，记作 `ctx`（Context 的记法）。

**S05　Context 树**：每个 Context 至多有一个父 Context；没有父的称为根；全部 Context 因此构成一棵树。子 Context 只能由父 Context 的方法创建。

**S06　Context 成员**：Cordis 直接定义在 Context 上的属性与方法（`plugin`、`inject`、`effect`、`on`、`emit`、`waterfall`、`extend`、`isolate`、`intercept`、`logger`、`root`、`fiber` 等）；不是 Service，不经键解析取得。

**S07　config**：加载一个 Plugin 时随同传入的配置值。

**S08　apply**：Plugin 的入口成员，签名 `apply(ctx, config)`；函数形态的 Plugin 本身即 `apply`，类形态以 `(ctx, config)` 实例化。

**S09　Service**：Context 上的具名能力；名字为字符串（即"键"），值为某个实现实例。键集合开放：本机文档中出现 166 个不同的 `ctx.<名字>`，扣除 S06 成员后 Service 键在百个量级，任何 Plugin 可继续新增。"稳定"指同名键表示同一类能力、实现可替换而名字不变。

**S10　键解析**：读取 `ctx.<名字>` 时，从当前 Context 沿 S05 的父链向上查找提供该名字的 Context，返回其实现实例；查不到即解析失败。

**S11　Context 方法 `extend(meta)`**：创建子 Context；子继承父的全部属性，`meta` 的自有属性遮蔽同名属性；父 Context 不被修改。`meta` 为键值对集合（含 symbol 键），其值成为子 Context 的自有属性，不是 Service。

**S12　Context 方法 `isolate(name, label)`**：创建子 Context；子树内对键 `name` 的读写按 `label` 匹配，不再按父的 `label` 匹配；只影响 `name` 一个键。`label` 为不透明标识；同一 `label` 传给两次调用，两棵子树在该键上匹配同一提供者。

**S13　Context 方法 `intercept(name, config)`**：创建子 Context；其子树内加载的 Plugin 看到键 `name` 的配置为"祖先的拦截配置在前、本次在后"的合并结果；只改配置，不换实现。

**S14　Context 作用域**：对某个键，由 `label` 划定的匹配范围，是 S10 查找的边界。它不是 Context，也不是树节点；未被 `isolate` 的键没有额外边界，解析可一直到根。

**S15　注册表**：由某个 Service 实例持有、按写入顺序保存注册项的集合。

**S16　移除函数（disposer）**：注册表在写入一个注册项时返回的函数；调用它使该注册项被移除。

**S17　注册**：向注册表写入注册项并取得移除函数的动作；一次加载可产生多次注册。

**S18　fiber**：一次加载产生的运行时实例，持有该次加载的 `ctx` 与 `config`；`ctx` 与 fiber 一一对应（`ctx.fiber` 指向它）；fiber 结束即卸载。

**S19　effect**：Context 成员方法 `ctx.effect(fn)`；`fn` 为调用方传入的无参函数，调用时立即执行一次，其返回的移除函数被收集到当前 fiber；fiber 卸载时按相反顺序调用这些移除函数。

**S20　inject**：类形态 Plugin 的静态字段，值为**依赖的** Service 名字清单（数组，或"名字 → config"映射）；清单内名字在当前 `ctx` 按 S10 全部可解析后，才调用其 `apply`；其中任一实现被替换时，该 Plugin 卸载并重跑。`ctx.inject(deps, cb)` 是等价的方法形式。

**S21　提供 Service**：`Service` 子类在其构造器内调用 `super(ctx, name)`；该实例因此同时是本次加载的 fiber 主体，并被登记为当前 Context 上名为 `name` 的 Service，随 fiber 卸载而移除。`name` 由插件作者选定，作为公开名字。

**S22　agent**：DSH 中承载一次运行的执行体对象；非 Cordis 概念。

**S23　注册作用域（scope key）**：注册表的分层可见性模型；`ScopeKey` 为不透明对象身份（本机 agent loop 以活跃 agent 对象作 key）；注册落在全局 layer 或某 key 的 layer，读取时按"全局 → 该 key"合并，后者遮蔽前者。

**S24　Config schema**：Plugin 自带的、校验 `config` 的 schema；不通过则该次加载被拒绝。

**S25　Entry**：配置中的一条记录，字段为 `id`、`config`、`disabled`，并引用一个 Plugin。

**S26　patch**：Entry 修改清单；按 `id` 命中后执行覆盖（`config` 整体替换，无深合并）、插入、禁用三者之一。

**S27　composition**：多层 patch 按固定顺序依次作用后得到的全部 Entry。

**S28　npm 包**：含 manifest 与代码的发行单元。

**S29　plugin package**：导出至少一个 Plugin 的 npm 包；Entry 以包名引用它。

**S30　bundle**：自带一份 patch（YAML 文件）的 npm 包。

**S31　profile**：一份用户目录，含根 Entry 列表文件（`cordis.yml`）、patch 文件（`cordis.patch.yml`）与 bundle 列表（`package.json` 的 `dsh.profile.bundles`）。

**S32　安装**：把一个 npm 包加入 profile 的依赖与 bundle 列表的动作；对 bundle 而言，等于把其 patch 纳入 composition 的叠加顺序。

**S33　Context 方法 `plugin(Plugin, config)`**：触发一次加载，产生并返回其 fiber。

**S34　Loader**：读取 Entry 列表，逐条解析包名指向的 Plugin、用 Config schema 校验 `config`、调用 `ctx.plugin`，产生 fiber。

**S35　HMR（Hot Module Replacement，热模块替换）**：patch 或已加载 npm 包的代码发生变化时，卸载受影响的 fiber 并以新 `config` 重新加载的机制；DSH 的实现同时覆盖模块替换与 profile 配置重载（包描述为 "Coordinated module and profile configuration hot reload"）。

---

## 二、术语依赖链

```
S01 Cordis
 └ S02 Plugin
    └ S03 加载
       └ S04 Context
          ├ S05 Context 树
          │   ├ S11 extend(meta)
          │   ├ S12 isolate(name,label)     ← S09 / S10
          │   └ S13 intercept(name,config)  ← S09 / S07
          ├ S06 Context 成员
          ├ S07 config
          ├ S08 apply                         ← S04 / S07
          ├ S09 Service                       ← S04 / S06
          │   ├ S10 键解析                    ← S05
          │   ├ S14 Context 作用域            ← S10 / S12
          │   ├ S15 注册表
          │   │   └ S16 移除函数              ← S15
          │   │       └ S17 注册              ← S15 / S16
          │   ├ S19 effect                    ← S15 / S18
          │   ├ S21 提供 Service              ← S18
          │   └ S23 注册作用域                ← S15 / S17 / S22
          ├ S18 fiber                         ← S03 / S04 / S07
          ├ S20 inject                        ← S08 / S09 / S10
          └ S33 ctx.plugin                    ← S08 / S18
S22 agent
S24 Config schema                             ← S02 / S07
S25 Entry                                     ← S02 / S07
 ├ S26 patch
 │   └ S27 composition
 │       └ S32 安装                            ← S28 / S30 / S31
 └ S29 plugin package                          ← S28
     └ S30 bundle                              ← S27 / S29
         └ S31 profile                         ← S25 / S26 / S30
S28 npm 包
S34 Loader                                     ← S24 / S25 / S29 / S33
S35 HMR                                        ← S18 / S26 / S07
```

## 三、易错点（评审中实际出现过的误解）

| 误解 | 正确表述 |
|---|---|
| Context 上只有 Service | 还有 Context 成员（S06）与 `extend(meta)` 带入的元数据 |
| `extend`／`isolate`／`intercept` 是 Service | 都是 Context 成员方法（S06） |
| `inject` 声明的是提供的 Service | 声明的是**依赖**（S20）；提供用 `super(ctx, name)`（S21） |
| 一次加载必然产出一个 Service | 只有 `Service` 子类如此；普通 Plugin 只有 fiber（S21） |
| fiber 是 Plugin 的属性 | fiber 是"一次加载"的运行时实例，一次加载一个 fiber（S18） |
| `ctx.effect(fn)` 会保存 `fn` 供以后重复调用 | `fn` 立即执行一次，被保存的是它返回的移除函数（S19） |
| 注册表返回的移除函数会被某处集中记录 | 不集中记录；只有经 `ctx.effect` 登记的才记在 fiber 上（S17、S19） |
| `isolate` 会切断整棵树的解析 | 只对指定的那一个键换匹配，其他键照常向上解析（S12、S14） |
| "稳定键"意味着数量固定且很少 | 数量开放（本机文档 166 个 `ctx.<名字>`，Service 键百个量级） |
| S33 的 `ctx.plugin` 与 S03 的"加载"是同一项 | S03 是动作，S33 是触发该动作的方法 |
| "行""条目"可以用来指 Entry | 只叫 **Entry**；"注册项"专指注册表里的记录，二者不是一回事 |

## 四、口径与未验证项

| 项 | 状态 |
|---|---|
| "166 个 `ctx.<名字>`" | 已按本机 `docs/**/*.md` 全文正则统计；含 Context 自身成员与少量误匹配，故结论取"Service 键在百个量级"而非精确值 |
| 同一作用域内重复提供同名 Service 的行为（抛错或后者生效） | **未实测**，文档未见明确条文；可实现中通过 `isolate` 分作用域来规避 |
| 解析失败时的具体错误类型／错误码 | **未实测** |
| `inject` 的生命周期语义（服务变更后卸载重跑） | 来自 `docs/cordis-api/registry.zh.md` 的 `ctx.inject` 条目 |
| HMR 覆盖范围（模块 + profile 配置） | 来自 `@deepseek-ai/dsh-hmr` 包描述 |

## 五、对应官方文档

| 主题 | 文档 |
|---|---|
| 五个核心概念、waterfall、Loader 配置 | `docs/cordis-primer.zh.md` |
| Context 与 `extend`／`isolate`／`intercept` | `docs/cordis-api/context.zh.md` |
| `ctx.plugin`、`ctx.inject`、Plugin 形态 | `docs/cordis-api/registry.zh.md` |
| fiber、`ctx.effect` | `docs/cordis-api/fiber.zh.md` |
| `Service` 基类与 `super(ctx, name)` | `docs/cordis-api/service.zh.md` |
| 注册作用域（scope key） | `docs/subsystems/scope.zh.md` |
