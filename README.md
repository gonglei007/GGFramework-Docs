# GGFramework 架构文档（伞形仓库）

本仓库是 **GGFramework 跨模块整体架构的唯一权威来源（single source of truth）**。

GGFramework 由多个**彼此独立的 Git 仓库**组成，父目录 `d:\Works\GGFramework` 本身**不是** Git 仓库、也没有 submodule。整体架构、模块依赖关系、跨仓库决策与迁移路线等**横切内容**统一沉淀在这里；各模块**自身**的 API/接入说明仍保留在各自仓库的 README 中。

> 维护原则：跨仓库内容只在本仓库写一份；模块 README 顶部链接回这里，避免文档也出现"副本漂移"。

## 模块仓库索引

| 仓库 | 职责 | Unity 版本 | 默认分支 | 链接 |
|---|---|---|---|---|
| GGFramework-Core | 共享基础设施单一源（`Singleton` / `CoroutineUtil` / `TaskSystem`），UPM 包 | 2021.3+（`package.json` 声明） | main | [repo](https://github.com/gonglei007/GGFramework-Core) |
| GGFramework-GGResource | 资源加载 / AssetBundle 管理（自身代码命名空间为 `Loxodon.Framework.Bundles`） | 6000.3.13f1 | main | [repo](https://github.com/gonglei007/GGFramework-GGResource) |
| GGFramework-GGNetwork | 网络框架：HTTP / WebSocket / Download / HTTPDNS（依赖 ILRuntime） | 6000.3.13f1 | main | [repo](https://github.com/gonglei007/GGFramework-GGNetwork) |
| GGFramework-GGUI | UI 框架（内含 GGResource 的整包源码拷贝） | 6000.3.13f1 | **master** | [repo](https://github.com/gonglei007/GGFramework-GGUI) |
| GGFramework-GGUser | 用户系统（`UserSystem`、UID 持久化） | 6000.3.13f1 | main | [repo](https://github.com/gonglei007/GGFramework-GGUser) |
| GGFramework-Demo | 集成演示（扁平拷贝了全部模块源码，使用全局命名空间） | 6000.3.13f1 | main | [repo](https://github.com/gonglei007/GGFramework-Demo) |

> ⚠️ 注意：默认分支不统一——**GGUI 用 `master`，其余用 `main`**。批量脚本不能假设分支名一致。

## 文档导航

- **可运行的集成体验**：[Demo 开发者体验指南](https://github.com/gonglei007/GGFramework-Demo/blob/main/documents/showcase-guide.md)。独立大厅覆盖 Core 0.3.0、本地资源与 AssetBundle、回环 HTTP、UI 页面栈、模拟 SDK 与 Lua 规则。当前 Demo 使用同级 Core 的本地 UPM 引用；这是开发联调布局，未修改其他模块的发布依赖。

- **架构**
  - [整体技术方案](architecture/technical-solution.md)：**统领性方案**——目标/原则、分层架构、各模块技术定位、集成机制、迁移路线、工程规范、测试/CI 与风险
  - [GGFramework 架构图](architecture/architecture-diagram.md)：分层架构总览、Phase 2 接线进度、Core 引用约束（mermaid 可视化）
  - [整体架构概览](architecture/overview.md)：模块职责、分层、集成方式与关键结构问题
  - [模块依赖图与版本矩阵](architecture/dependency-graph.md)：现状依赖图、目标依赖图、Unity 版本/分支/远程矩阵
  - [源码拷贝集成与副本漂移](architecture/vendoring-drift.md)：vendoring 现状、漂移清单、危害与治理
- **架构决策记录（ADR）**
  - [ADR 索引与模板](adr/README.md)
  - [ADR-0001 Core 抽为独立仓库](adr/0001-core-独立仓库.md)
  - [ADR-0002 vendoring → UPM 迁移路线](adr/0002-vendoring-转-upm-迁移路线.md)
  - [ADR-0003 Phase 3 改为副本对齐](adr/0003-phase3-改为副本对齐.md)
- **路线图 / 开发计划**
  - [贴纸游戏 Unity 移植驱动的能力扩展计划](roadmap/sticker-game-unity-porting.md)：**前瞻提案**——由《贴纸小日子》浏览器原型移植 Unity 驱动，盘点框架能力缺口，给出拟新增底层模块（`GGSave`/`GGAudio`/`GGLocalization`/`GGAds` 等）、现有模块改良清单与分阶段路线（重大取舍待另立 ADR）

## 现状速览

- 模块之间此前**没有任何工程级依赖声明**（无跨模块 UPM 引用、无跨模块 asmdef 引用），所谓"相互依赖"实际靠**源码拷贝（vendoring）**。
- 后果：68 组同名文件中 **41 组内容已漂移**；任一模块的修复不会传播到其它副本。
- 治理：抽出 `GGFramework-Core` 作为共享基础类的单一源（详见 ADR-0001 / ADR-0002）。
- **迁移进度：Phase 0/1/2/3 已全部完成。** 5 个消费方（GGResource / GGUI / GGNetwork / GGUser / Demo）均已通过 UPM git URL 接线到 **Core `v0.1.1`** 并删除本地 `Singleton`/`CoroutineUtil`/`TaskSystem` 副本，基础类副本漂移已彻底消除；5 个工程已统一到 Unity 6000.3.13f1，本机可真实编译/测试全部仓库。含第三方耦合的 `GameDebugger`/`ZipUtil`/`UIAdaptor`/`LogAdaptor` 因 UPM 包无法引用工程内第三方、不纳入 Core，改以**副本对齐**治理（Phase 3 已完成，详见 [ADR-0003](adr/0003-phase3-改为副本对齐.md)）：`UIAdaptor` P0 回填、`GameDebugger` 对齐 canonical（Demo 保留有意禁用），`ZipUtil`/`LogAdaptor` 已一致。整体方案见 [整体技术方案](architecture/technical-solution.md)。
