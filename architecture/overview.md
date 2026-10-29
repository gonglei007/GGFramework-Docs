# 整体架构概览

> 本文是 GGFramework 跨模块架构的顶层视图。模块自身的 API/接入细节见各仓库 README。

## 1. 定位

GGFramework 是面向 Unity 游戏客户端的一组**可插拔框架模块**：资源、网络、UI、用户系统各自独立成库，另有一个集成演示工程。目标是让项目按需引入其中若干模块。

## 2. 仓库拓扑（关键前提）

- 工作区根 `d:\Works\GGFramework` **不是 Git 仓库**，也**没有 `.gitmodules`**。
- 其下每个模块是**彼此完全独立的 Git 仓库**，需分别执行 Git 命令（推荐 `git -C <仓库绝对路径> <命令>`）。
- 因此不存在"一次 clone 拿到全部"的机制；模块组合同时依赖**人工源码拷贝**（现状）与**UPM git URL 引用**（目标）。

## 3. 模块职责

| 模块 | 职责 | 命名空间现状 | asmdef 分层现状 |
|---|---|---|---|
| **Core** | 共享基础设施单一源：`Singleton<T>` / `SingletonLifetime`、`CoroutineUtil`、`TaskSystem` / `GTask` | 全局（`Singleton`/`CoroutineUtil`）+ `GGFramework.GGTask`（`TaskSystem`） | 独立 UPM 包：`GGFramework.Core.Runtime` + `GGFramework.Core.Tests` |
| **GGResource** | 资源加载、AssetBundle 管理、下载列表调度 | 自身代码在 `Loxodon.Framework.Bundles`（**非** `GGFramework.GGResource`） | 自有核心落在 `Assembly-CSharp`，无程序集边界 |
| **GGNetwork** | HTTP / WebSocket / Download / HTTPDNS / ServiceCenter | `GGFramework.GGNetwork`，但混入 `GGFramework.GGTask` | 完整分层：`LocalPackages` ← `Dependency` ← `GGFramework.GGNetwork` ← `Tests` |
| **GGUI** | UI 框架 | 自有命名空间 | 自有核心落在 `Assembly-CSharp` |
| **GGUser** | 用户系统、UID 持久化 | `GGFramework.GGUser` 系 | 完整分层 |
| **Demo** | 集成演示 | **全局命名空间**（`TaskSystem` 经 `using GGFramework.GGTask` 引入 Core） | 扁平，无分层 |

## 4. 集成方式：接线前 vs 当前

### 接线前——源码拷贝（vendoring，历史）

接线前模块之间**没有任何工程级依赖声明**：

- `manifest.json` 无跨模块 UPM 引用（除 GGNetwork 依赖 `com.ourpalm.ilruntime` 外全是 `com.unity.*`）。
- 无跨模块 asmdef 引用；四个库模块此前都**没有 `package.json`**（Core 是第一个）。
- 实际"依赖"靠拷贝：每个模块在 `Scripts/Dependency/Utils` 各存一份基础类（`Singleton`/`CoroutineUtil`/`GameDebugger`/`TaskSystem`/`ZipUtil`）；GGResource 被整包拷进 GGUI 与 Demo；GGNetwork 与 GGUser 各自拷贝 `UIAdaptor`/`LogAdaptor`；Demo 在 `ggsource/Scripts` 下扁平拷贝了全部模块。

详见 [源码拷贝集成与副本漂移](vendoring-drift.md)。

### 当前——UPM git URL 单一源（Phase 2 已完成）

以 `GGFramework-Core` 为单一源包，**5 个消费方（GGResource/GGUI/GGNetwork/GGUser/Demo）均已**经 UPM git URL 引用 Core `#v0.1.1` 并**删除本地 `Singleton`/`CoroutineUtil`/`TaskSystem` 副本**，基础三类漂移已根除。含第三方耦合的 `GameDebugger`/`ZipUtil`/`UIAdaptor`/`LogAdaptor` 因 **UPM 包无法引用工程内第三方**、不纳入 Core，改以**副本对齐**治理（Phase 3 已完成，见 [ADR-0003](../adr/0003-phase3-改为副本对齐.md)）。路线详见 [ADR-0002](../adr/0002-vendoring-转-upm-迁移路线.md)。

## 5. 关键结构性问题（治理清单）

1. **副本漂移**：接线前 68 组同名文件中 41 组内容不一致；源头修复不传播（曾出现 GGResource 的 bug 在 GGUI 副本已修、源头未修的反向漂移）。**Phase 2 后 `Singleton`/`CoroutineUtil`/`TaskSystem` 已单一源化、漂移根除**；**Phase 3 对含第三方耦合的 `GameDebugger`/`UIAdaptor`/`ZipUtil`/`LogAdaptor` 做了副本对齐**（不纳入 Core，因 UPM 包无法引用工程内第三方）：`UIAdaptor` P0 回填、`GameDebugger` 对齐 canonical（Demo 保留有意禁用），`ZipUtil`/`LogAdaptor` 已一致，无意义漂移清零（见 [ADR-0003](../adr/0003-phase3-改为副本对齐.md)）。
2. **命名空间不统一**：GGResource 用 `Loxodon.Framework.Bundles`，GGNetwork 混入 `GGFramework.GGTask`，Demo 全在全局命名空间。
3. **asmdef 分层不一致**：GGNetwork/GGUser 有完整分层，GGResource/GGUI 自有核心落在 `Assembly-CSharp` 无边界。
4. ~~**Unity 版本三分裂**~~（已解决）：5 工程已统一到 6000.3.13f1，跨仓库可单一环境编译/测试。见 [依赖图与版本矩阵](dependency-graph.md)。
5. **无 CI**：任一仓库都没有 GitHub Actions 等自动化；Core 抽出后是引入 CI 的天然起点。
6. **测试覆盖极不均**：Core(10)、GGNetwork(160)、GGUser(68)、GGUI(11) 有 EditMode 测试且全绿；GGResource/Demo 仍无自动化测试（接线以编译 exit 0 为门槛）。

## 6. 工程卫生约定

- `.gitignore` 位于各**仓库根**（非 Unity 工程子目录），统一忽略：`*.csproj`、`*.sln`、`UserSettings/`、`.qoder/`。
- 由于这些生成物**早于忽略规则被提交**，补规则后必须再 `git rm --cached <精确路径>`（保留磁盘文件、可逆）才能真正解除跟踪。
