# ADR-0002: vendoring → UPM 单一源迁移路线

- 状态：已接受（进行中）
- 日期：2026-10-27
- 相关：ADR-0001；仓库 `GGFramework-Core` 及全部消费方仓库

## 背景

现状是源码拷贝（vendoring）集成，导致 68 组同名文件中 41 组漂移，源头修复不传播（见 [vendoring-drift](../architecture/vendoring-drift.md)）。Core 独立仓库已建立（ADR-0001），需要一条**低风险、可增量、可回退**的路线，把消费方从"持有副本"迁移到"UPM 引用单一源"，且不破坏任何现有工程的编译。

## 决策

采用**分阶段迁移**，每阶段都是纯加法或原子替换，任何阶段都可独立回退：

### Phase 0（已完成）——缺陷修复与工程卫生

- 修复审查发现的 P0/P1/P2 缺陷（GGResource `GetAssets<T>` 恒真、`loadingList` 空引用、`DestroyObject` KeyNotFound；GGUser `SaveUID/LoadUID` 句柄泄漏；GGNetwork `Encode` body 超 16MB 静默截断）。
- 统一各仓库 `.gitignore`（`*.csproj`/`*.sln`/`UserSettings/`/`.qoder/`）并对已跟踪的生成物 `git rm --cached`。

### Phase 1（已完成）——建立 Core 单一源

- 抽出 `GGFramework-Core`，仅纳入**零第三方依赖**的 `Singleton`/`CoroutineUtil`/`TaskSystem`，配 EditMode 测试。
- Core **不参与任何现有工程编译**，是纯加法、可逆的"安全基础层"。

### Phase 2（进行中）——接线消费方

> **进度快照**：**GGUser 试点已完成**（commit `2b70348`，`manifest` 引用 Core `#v0.1.0` + 主/测试 asmdef 加 `GGFramework.Core.Runtime` 引用 + 删除本地 `Singleton`/`CoroutineUtil` 副本，EditMode 68/68 全绿，已推送）。GGResource/GGNetwork/GGUI/Demo 待接。
>
> **试点暴露的关键约束**：asmdef 引用**不具传递性**——用到 `Singleton<>` 派生类型的**测试程序集**也必须直接引用 `GGFramework.Core.Runtime`，否则编译报 **CS0012**。GGUser 首跑即因两个测试 asmdef 漏加引用而失败，补齐后全绿。

对每个消费方，**在同一批改动内**完成：

1. `manifest.json` 增加 `"com.gonglei.ggframework.core": "https://github.com/gonglei007/GGFramework-Core.git#v0.1.0"`。
2. **删除**本地 `Scripts/Dependency/Utils` 下的 `Singleton.cs`/`CoroutineUtil.cs`/`TaskSystem.cs` 副本。
3. 相关 asmdef（**含所有测试 asmdef**，见上方 CS0012 约束）增加对 `GGFramework.Core.Runtime` 的引用。
4. 编译 + 跑该仓库测试验证无回归。

> **关键约束（二义性）**：`Singleton`/`CoroutineUtil` 在**全局命名空间**。若"引用 Core"与"删除本地副本"不同批完成，工程会同时存在两份同名类型 → **类型二义性编译错误**。因此第 1、2 步必须原子提交，不可分步。

### Phase 3（待评估）——纳入含第三方耦合的类

- `GameDebugger`（耦合 CodeStage.AdvancedFPSCounter）、`ZipUtil`（耦合 SharpZipLib）、`UIAdaptor`/`LogAdaptor`（SimpleJson）。
- 以 **Core.Optional 子程序集**形式评估纳入，避免让 Core 主体背上第三方依赖；三个版本差异大的类（如 `GameDebugger`）需先做合并。

### Phase 4（远期）——收敛命名空间与统一分层

- 逐步把 `Loxodon.Framework.Bundles`（GGResource）、混入的 `GGFramework.GGTask`（GGNetwork）等统一到 `GGFramework.*`。
- 为 GGResource/GGUI 自有核心补 asmdef 边界（脱离 `Assembly-CSharp`）。
- 引入 CI（Core 仓库先行，跑 EditMode 测试）。

## 理由与取舍

- **分阶段 + 纯加法优先**：Phase 0/1 不触碰任何现有工程的编译产物，风险为零；真正有破坏性的"删副本"集中在 Phase 2 且逐仓库原子进行，爆炸半径可控。
- **从最小依赖面起步**：Phase 1 只搬零第三方依赖的类，使 Core 可独立编译验证；把有耦合的类推迟，避免 Core 一上来就要求消费方预装第三方库。
- **取舍**：Phase 2 的原子替换需要一次性改 manifest + 删副本 + 改 asmdef，比"先引用后删"更费神，但这是二义性约束下的唯一安全做法。

## 被拒方案

- **一次性大重构（统一命名空间 + 全量 UPM 化 + 删所有副本）**：爆炸半径过大、无法增量验证、回退困难。
- **保留副本 + 用脚本定期同步**：治标不治本，仍需人工解冲突，漂移会再现。
- **改用 git submodule 共享基础类**：submodule 在多独立仓库 + Unity 工程下管理复杂，且 UPM git URL 已是 Unity 原生机制，无需引入 submodule。

## 影响

- 正面：迁移完成后共享基础类只有一个权威版本，源头修复自动传播到所有消费方；为统一命名空间、CI、依赖治理打下地基。
- 负面/成本：Phase 2 每接入一个仓库都要原子改三处并回归测试；Demo(2019.4) 接入需先评估与 Core `unity:2021.3` 下限的兼容性。
- 失效条件：若将来改用 monorepo 或引入私有 UPM registry，引用方式需相应调整，但"单一源 + 删副本同批"的核心原则不变。
