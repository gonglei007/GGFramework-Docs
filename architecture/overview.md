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
| **Demo** | 集成演示 | **全局命名空间** | 扁平，无分层 |

## 4. 集成方式：现状 vs 目标

### 现状——源码拷贝（vendoring）

模块之间**没有任何工程级依赖声明**：

- `manifest.json` 无跨模块 UPM 引用（除 GGNetwork 依赖 `com.ourpalm.ilruntime` 外全是 `com.unity.*`）。
- 无跨模块 asmdef 引用；四个库模块此前都**没有 `package.json`**（Core 是第一个）。
- 实际"依赖"靠拷贝：每个模块在 `Scripts/Dependency/Utils` 各存一份基础类（`Singleton`/`CoroutineUtil`/`GameDebugger`/`TaskSystem`/`ZipUtil`）；GGResource 被整包拷进 GGUI 与 Demo；GGNetwork 与 GGUser 各自拷贝 `UIAdaptor`/`LogAdaptor`；Demo 在 `ggsource/Scripts` 下扁平拷贝了全部模块共 138 个文件。

详见 [源码拷贝集成与副本漂移](vendoring-drift.md)。

### 目标——UPM git URL 单一源

以 `GGFramework-Core` 为起点，把共享基础类收敛为**单一源包**，各消费方经 UPM git URL 引用、并**删除本地副本**，从根上消除漂移。路线详见 [ADR-0002](../adr/0002-vendoring-转-upm-迁移路线.md)。

## 5. 关键结构性问题（治理清单）

1. **副本漂移**：68 组同名文件中 41 组内容已不一致；源头修复不传播（曾出现 GGResource 的 bug 在 GGUI 副本已修、源头未修的反向漂移）。
2. **命名空间不统一**：GGResource 用 `Loxodon.Framework.Bundles`，GGNetwork 混入 `GGFramework.GGTask`，Demo 全在全局命名空间。
3. **asmdef 分层不一致**：GGNetwork/GGUser 有完整分层，GGResource/GGUI 自有核心落在 `Assembly-CSharp` 无边界。
4. **Unity 版本三分裂**：2019.4 / 2021.3 / 6000.3 并存，跨仓库统一编译/测试困难。见 [依赖图与版本矩阵](dependency-graph.md)。
5. **无 CI**：任一仓库都没有 GitHub Actions 等自动化；Core 抽出后是引入 CI 的天然起点。
6. **测试覆盖极不均**：GGNetwork、GGUser 有测试，GGResource/GGUI 此前零测试，Core 现有 10 个 EditMode 用例。

## 6. 工程卫生约定

- `.gitignore` 位于各**仓库根**（非 Unity 工程子目录），统一忽略：`*.csproj`、`*.sln`、`UserSettings/`、`.qoder/`。
- 由于这些生成物**早于忽略规则被提交**，补规则后必须再 `git rm --cached <精确路径>`（保留磁盘文件、可逆）才能真正解除跟踪。
