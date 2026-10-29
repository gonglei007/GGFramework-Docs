# GGFramework 整体技术方案

> 本文是 GGFramework 的**统领性技术方案（master technical solution）**：说明框架要解决什么问题、按什么原则设计、目标架构长什么样、各模块的技术定位、如何集成、如何从现状迁移到目标，以及配套的工程规范、测试/CI 策略与风险。
>
> 细节分散在下列文档，本文只做**整合与索引**，不重复展开：
> - 现状拓扑与结构问题：[整体架构概览](overview.md)
> - 依赖关系与版本矩阵：[模块依赖图与版本矩阵](dependency-graph.md)
> - 分层架构可视化：[GGFramework 架构图](architecture-diagram.md)
> - 拷贝集成与漂移实证：[源码拷贝集成与副本漂移](vendoring-drift.md)
> - 关键决策：[ADR-0001 Core 独立仓库](../adr/0001-core-独立仓库.md)、[ADR-0002 迁移路线](../adr/0002-vendoring-转-upm-迁移路线.md)

---

## 1. 目标与范围

### 1.1 框架定位

GGFramework 是面向 Unity 游戏客户端的一组**可插拔框架模块**：资源加载、网络、UI、用户系统各自独立成库，另有一个集成演示工程。设计意图是让具体游戏项目**按需引入其中若干模块**，而非全量绑定。

### 1.2 本方案要解决的核心问题

| # | 问题 | 后果 | 治理归属 |
|---|---|---|---|
| P1 | 模块间靠**源码拷贝（vendoring）**复用，无工程级依赖声明 | 68 组同名文件中 41 组已漂移；源头修复不传播（曾出现下游副本已修、源头反错的反向漂移） | §5 集成机制、§6 路线 |
| P2 | **命名空间不统一** | GGResource 在 `Loxodon.Framework.Bundles`、GGNetwork 混入 `GGFramework.GGTask`、Demo 全在全局命名空间 | §6 Phase 4 |
| P3 | **asmdef 分层不一致** | GGResource/GGUI 自有核心落在 `Assembly-CSharp`，无程序集边界，无法被稳定引用 | §6 Phase 4 |
| P4 | ~~**Unity 版本三分裂**~~（已解决） | 5 个工程已统一到 6000.3.13f1，单一环境可编译/测试全部仓库 | §7 规范 |
| P5 | **无 CI、测试覆盖不均** | 回归全靠人工；GGResource/Demo 仍无自动化测试（接线以编译 exit 0 为门槛） | §8 测试与 CI |

### 1.3 范围边界

- **在范围内**：跨仓库的整体架构、集成机制、迁移路线、工程规范、测试/CI 策略。
- **不在范围内**：各模块**自身**的 API/接入细节——保留在各仓库 README（如 GGNetwork 的 `documents/quickstart.md`、`manual.md`）。本方案只给技术定位与对外契约。

---

## 2. 设计原则

1. **单一源（Single Source of Truth）**：共享基础类只有一个权威版本，消费方引用而非拷贝。架构文档同理，统一沉淀在 `GGFramework-Docs`。
2. **可插拔 / 低耦合**：模块之间尽量只依赖基础设施层（Core），功能模块之间不直接强绑定；第三方耦合隔离在各自模块或可选子程序集内。
3. **增量、可回退**：迁移分阶段推进，每阶段要么纯加法、要么原子替换，任一阶段都能独立回退（见 §6）。
4. **Unity 原生机制优先**：用 UPM git URL 而非 git submodule 共享代码；用 asmdef 而非目录约定划分程序集边界。
5. **版本锁定**：跨仓库引用锁定 tag（如 `#v0.1.1`），避免 `#branch` 带来的不可复现构建。

---

## 3. 目标架构（分层）

> 可视化见 [GGFramework 架构图](architecture-diagram.md)。此处给分层职责与依赖方向。

```
┌───────────────────────────────────────────────┐
│  集成 / 消费层    Demo、各具体游戏项目            │  ← 组合所需模块
├───────────────────────────────────────────────┤
│  功能模块层       GGResource · GGNetwork ·        │  ← 各自独立、可插拔
│                   GGUI · GGUser                   │
├───────────────────────────────────────────────┤
│  基础设施层       GGFramework-Core                │  ← 共享基础类单一源
│                   Singleton / CoroutineUtil /     │
│                   TaskSystem                       │
├───────────────────────────────────────────────┤
│  第三方依赖       ILRuntime · NGUI · Loxodon ·     │  ← 隔离在模块内部
│                   SharpZipLib · SimpleJson ·        │
│                   CodeStage.AdvancedFPSCounter      │
└───────────────────────────────────────────────┘
```

**依赖方向**：上层依赖下层，同层之间不直接依赖。功能模块统一经 UPM git URL 依赖 Core；第三方依赖被隔离在各自模块内，不上抬到 Core 主体（含第三方耦合的共享类走 Core.Optional 子程序集，见 §6 Phase 3）。

---

## 4. 模块技术方案

各模块的对外定位、命名空间现状、程序集边界与测试现状汇总如下（详细职责见 [overview.md](overview.md) §3）。

| 模块 | 技术定位 | 命名空间现状 | 程序集边界现状 | 关键第三方耦合 | 测试现状 |
|---|---|---|---|---|---|
| **Core** | 共享基础设施单一源 | 全局（`Singleton`/`CoroutineUtil`）+ `GGFramework.GGTask`（`TaskSystem`） | UPM 包：`GGFramework.Core.Runtime` + `GGFramework.Core.Tests` | 无（Phase 1 只纳零依赖类） | 10 个 EditMode，Unity 6000.3.13f1 验证全绿 |
| **GGResource** | 资源加载 / AssetBundle 管理 / 下载调度 | `Loxodon.Framework.Bundles`（**非** `GGFramework.*`） | 自有核心在 `Assembly-CSharp`，无边界 | SharpZipLib（`ZipUtil`） | 无自动化测试（接线以编译 exit 0 为门槛） |
| **GGNetwork** | HTTP / WebSocket / Download / HTTPDNS / ServiceCenter | `GGFramework.GGNetwork`（使用 `GGFramework.GGTask`，现来自 Core） | 完整分层：`LocalPackages ← Dependency ← GGFramework.GGNetwork ← Tests` | ILRuntime（`com.ourpalm.ilruntime`）、SimpleJson（Adaptor） | EditMode 160/160 全绿 |
| **GGUI** | UI 框架（分层 UI 生命周期 + NGUI 扩展） | 自有命名空间 | 自有核心在 `Assembly-CSharp`，无边界 | NGUI、LoxodonFramework、CodeStage.AdvancedFPSCounter；内含 GGResource 整包拷贝 | EditMode 11/11 全绿 |
| **GGUser** | 用户系统 / UID 持久化 / 平台聚合客户端 | `GGFramework.GGUser` 系 | 完整分层 | SimpleJson、MonsterLove.StateMachine | 9 EditMode + 2 PlayMode（EditMode 68/68 通过） |
| **Demo** | 集成演示 | **全局命名空间**（`TaskSystem` 现经 `using GGFramework.GGTask` 引入 Core） | 扁平、无分层（`ggsource/Scripts` 拷贝全部模块） | 随拷贝带入 | 无自动化测试（接线以编译 exit 0 为门槛） |

> 说明：GGUI 全仓 539 个 `.cs` 中，绝大多数属 `Assets/LocalPackages` 下的第三方包，项目自身可维护源码仅约 45 个（`Assets/Scripts` + `Demo/Scripts/Demo.cs`）。审查/改动应只针对自有源码，排除 LocalPackages、`.meta`、`Library/Temp/obj`。

### 4.1 Core 内部结构（单一源）

Core 是 UPM 包，**仓库根即包根**（无需 `?path=`），包名 `com.gonglei.ggframework.core`：

- `Runtime/GGFramework.Core.Runtime.asmdef`：`Singleton.cs`（含非泛型 `SingletonLifetime.IsQuitting`，修复泛型静态字段幽灵对象问题）、`CoroutineUtil.cs`、`TaskSystem.cs`。
- `Tests/GGFramework.Core.Tests.asmdef`：结构 / Singleton 退出期 / GTask 线程三类 EditMode 用例。
- `.meta` 承载稳定 GUID，**必须入库**（UPM 包的一部分，切勿 .gitignore）。

---

## 5. 集成机制：现状 → 目标

### 5.1 现状——源码拷贝（vendoring）

模块间**没有任何工程级依赖声明**（无跨模块 UPM 引用、无跨模块 asmdef 引用），"复用"靠把源码文件直接拷进另一个仓库。拷贝范围与漂移清单见 [vendoring-drift.md](vendoring-drift.md)。

### 5.2 目标——UPM git URL 单一源

以 Core 为起点，把共享基础类收敛为单一源包，消费方经 UPM git URL 引用并**删除本地副本**：

```json
"com.gonglei.ggframework.core": "https://github.com/gonglei007/GGFramework-Core.git#v0.1.1"
```

### 5.3 两条硬性约束（务必遵守）

1. **类型二义性**：`Singleton`/`CoroutineUtil` 在**全局命名空间**。"引用 Core"与"删除本地同名副本"必须**同批原子完成**，否则工程同时存在两份同名类型 → 编译期二义性错误。
2. **asmdef 引用不具传递性**：任何用到 `Singleton<>` 派生类型的程序集（**包括测试程序集**）都必须**直接**引用 `GGFramework.Core.Runtime`，否则报 **CS0012**（`The type 'Singleton<>' is defined in an assembly that is not referenced`）。GGUser 试点中正是两个测试 asmdef 漏加引用导致 EditMode 首跑失败，补引用后 68/68 全绿。

---

## 6. 迁移路线（Phase 0–4）

> 决策依据与取舍见 [ADR-0002](../adr/0002-vendoring-转-upm-迁移路线.md)。此处给**当前进度快照**。

| 阶段 | 内容 | 状态 |
|---|---|---|
| **Phase 0** | 缺陷修复（GGResource `GetAssets<T>` 恒真、`loadingList` 空引用、`DestroyObject` KeyNotFound；GGUser `SaveUID/LoadUID` 句柄泄漏；GGNetwork `Encode` 超 16MB 静默截断）+ 工程卫生（`.gitignore` 统一 + `git rm --cached`） | 已完成 |
| **Phase 1** | 抽出 `GGFramework-Core`（零第三方依赖的 `Singleton`/`CoroutineUtil`/`TaskSystem` + EditMode 测试）；纯加法、不参与任何现有工程编译 | 已完成（tag `v0.1.0` commit `0d4f009`；后热修 `Singleton` 补 `Application.isPlaying` 守卫升 `v0.1.1` commit `a5395876`） |
| **Phase 2** | 接线消费方：`manifest` 引用 Core + asmdef 加引用 + 删本地副本 + 回归测试，逐仓库原子进行 | **已完成**：5 消费方全部接线到 Core `v0.1.1`（GGResource `ec7dfc9`、GGUI `1c0b631`、GGNetwork `c439bf2`、GGUser `c77b466`、Demo `107ffe3`），各仓 batchmode 编译/EditMode 测试全绿并已推送 |
| **Phase 3** | 纳入含第三方耦合的共享类（`GameDebugger`/`ZipUtil`/`UIAdaptor`/`LogAdaptor`），以 **Core.Optional 子程序集**形式评估，避免 Core 主体背第三方依赖 | 待评估 |
| **Phase 4** | 收敛命名空间到 `GGFramework.*`；为 GGResource/GGUI 自有核心补 asmdef 边界；引入 CI（Core 先行） | 远期 |

### 6.1 Phase 2 逐仓库接线的标准动作

对每个消费方，在**同一批改动内**完成：

1. `manifest.json` 增加 Core 依赖（锁定 tag）。
2. **删除**本地 `Scripts/Dependency/Utils` 下的 `Singleton.cs`/`CoroutineUtil.cs`/`TaskSystem.cs` 副本（含 `.meta`）。
3. 主 asmdef **及所有测试 asmdef** 增加对 `GGFramework.Core.Runtime` 的引用（见 §5.3 约束 2）。
4. 在**匹配 Unity 版本**的环境编译 + 跑该仓库测试验证无回归。

> **验证前提（已满足）**：5 个工程已统一到 Unity 6000.3.13f1，本机即可对全部仓库真实编译/测试。Phase 2 各仓接线均在 batchmode 编译（exit 0、无 error CS）或 EditMode 测试全绿后才推送，不存在"本机无法验证的破坏性改动"。

---

## 7. 工程规范

### 7.1 仓库拓扑

- 工作区根 `d:\Works\GGFramework` **不是 Git 仓库**、无 `.gitmodules`；每个模块是**彼此独立的 Git 仓库**，须分别执行 Git 命令（推荐 `git -C <仓库绝对路径> <命令>`）。
- 远程统一为 `git@github.com:gonglei007/GGFramework-<名称>.git`。
- **默认分支不统一**：GGUI 用 `master`，其余用 `main`。批量脚本不能假设分支名一致。

### 7.2 版本矩阵（详见 [dependency-graph.md](dependency-graph.md) §3）

| 仓库 | Unity 版本 | 默认分支 |
|---|---|---|
| Core | 2021.3+（包声明 `unity: 2021.3`） | main |
| GGResource / GGNetwork / GGUI | 6000.3.13f1 | main / main / **master** |
| GGUser | 6000.3.13f1 | main |
| Demo | 6000.3.13f1 | main |

### 7.3 生成物治理

- `.gitignore` 位于各**仓库根**（非 Unity 工程子目录），统一忽略 `*.csproj`、`*.sln`、`UserSettings/`、`.qoder/`。
- 这些生成物**早于忽略规则被提交**，补规则后必须再 `git rm --cached <精确路径>`（保留磁盘文件、可逆）才能真正解除跟踪。
- **例外**：Core 包仓库的 `.meta` 承载 GUID，是包的一部分，**必须入库**。

### 7.4 提交规范

提交信息遵循 **conventional commits 前缀 + 中文描述**（如 `refactor(deps): GGUser 改用 GGFramework.Core 单一源，移除本地副本`）。

---

## 8. 测试与 CI 策略

### 8.1 测试现状

见 [dependency-graph.md](dependency-graph.md) §4。要点：Core（10 EditMode）、GGNetwork（EditMode 160/160）、GGUser（EditMode 68/68）、GGUI（EditMode 11/11）均有测试且全绿；GGResource/Demo 无自动化测试，接线以 batchmode 编译 exit 0 为门槛。

### 8.2 UPM 包测试的 testables 约束

Core 包内 `Tests/` 属 UPM 包测试，**消费工程须在 `manifest.json` 的 `testables` 中声明本包**才会被编译/运行：

```json
"testables": ["com.gonglei.ggframework.core"]
```

否则 `-runTests` 会得到 `total=0` 但 `result="Passed"` 的**假通过**。

### 8.3 CI 策略（Phase 4）

- 当前**无任何 CI**。Core 抽出后是引入 CI 的天然起点：先在 Core 仓库跑 EditMode 测试（GitHub Actions + Unity Test Framework）。
- 版本三分裂是 CI 的主要障碍：需按仓库矩阵分别指定 Unity 版本（`unity-version` 参数），或先只对能被现有 license/环境覆盖的仓库开 CI。

---

## 9. 风险与未决事项

| 风险 / 未决项 | 说明 | 缓解 |
|---|---|---|
| ~~**本机无法验证多数仓库**~~（已消除） | 5 工程已统一到 6000.3.13f1，本机可真实编译/测试全部仓库 | Phase 2 各仓均经 batchmode 编译/测试验证后推送 |
| ~~**Demo 接 Core 兼容性**~~（已消除） | Demo 已升级到 6000.3.13f1（≥ Core `unity:2021.3` 下限）并完成接线 | Demo `TaskSystem` 从全局命名空间迁到 `using GGFramework.GGTask`，编译 exit 0 |
| **Phase 3 类合并成本** | `GameDebugger`×3 版本差异大，合并需先统一行为 | 以 Core.Optional 子程序集承接，先合并再纳入 |
| **命名空间收敛的破坏性** | Phase 4 改命名空间会影响所有引用点 | 放在最后、分模块推进，每次配回归测试 |
| **文档漂移** | 多仓库文档若各写一份会重演代码漂移 | 跨仓库内容只在 `GGFramework-Docs` 写一份，模块 README 链接回这里 |

---

## 10. 附：关键事实速查

- Core：包名 `com.gonglei.ggframework.core`，仓库根即包根；tag `v0.1.0`(commit `0d4f009`) → `v0.1.1`(commit `a5395876`，热修 `Singleton` 补 `Application.isPlaying` 守卫）。
- Phase 2 全量接线（均锁 Core `#v0.1.1`，已推送）：GGResource `ec7dfc9`、GGUI `1c0b631`（master）、GGNetwork `c439bf2`、GGUser `c77b466`、Demo `107ffe3`。
- 单一源引用写法：`"com.gonglei.ggframework.core": "https://github.com/gonglei007/GGFramework-Core.git#v0.1.1"`。
- 两条硬约束：全局命名空间类型的**二义性**（引用+删副本同批）、asmdef 引用**不具传递性**（测试程序集也要直接引用 Core.Runtime）。
- Demo 特例：其 `TaskSystem` 副本原在**全局命名空间**，Core 版在 `GGFramework.GGTask`，故删副本后 5 个调用文件须补 `using GGFramework.GGTask;`；Demo 零 asmdef，靠 Core.Runtime 的 `autoReferenced` 被 `Assembly-CSharp` 自动引用。
