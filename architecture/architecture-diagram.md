# GGFramework 架构图

> 本文用可视化图呈现 GGFramework 的**分层架构**、**Phase 2 接线进度**与 **Core 单一源引用约束**。
> 现状 / 目标的**逐仓库依赖图**与版本矩阵见 [模块依赖图与版本矩阵](dependency-graph.md)；文字方案见 [整体技术方案](technical-solution.md)。

---

## 1. 分层架构总览（目标态）

依赖方向自上而下：**集成层 → 功能模块层 → 基础设施层**；第三方依赖被隔离在功能模块内部，不上抬到 Core。同层模块之间不直接依赖。

```mermaid
graph TD
    subgraph L4["集成 / 消费层"]
        Demo["Demo<br/>集成演示工程"]
        Game["具体游戏项目"]
    end

    subgraph L3["功能模块层（可插拔）"]
        GGResource["GGResource<br/>资源 / AssetBundle"]
        GGNetwork["GGNetwork<br/>HTTP / WS / Download"]
        GGUI["GGUI<br/>UI 框架"]
        GGUser["GGUser<br/>用户系统"]
    end

    subgraph L2["基础设施层（单一源）"]
        Core["GGFramework-Core<br/>Singleton / CoroutineUtil / TaskSystem"]
    end

    subgraph L1["第三方依赖（隔离在模块内）"]
        ILRuntime["ILRuntime"]
        SimpleJson["SimpleJson"]
        NGUI["NGUI / Loxodon"]
        CodeStage["CodeStage.FPSCounter"]
        SharpZipLib["SharpZipLib"]
        StateMachine["MonsterLove.StateMachine"]
    end

    Demo --> GGResource
    Demo --> GGNetwork
    Demo --> GGUI
    Demo --> GGUser
    Game --> GGResource
    Game --> GGNetwork
    Game --> GGUI
    Game --> GGUser

    GGResource --> Core
    GGNetwork --> Core
    GGUI --> Core
    GGUser --> Core

    GGNetwork --> ILRuntime
    GGNetwork --> SimpleJson
    GGUI --> NGUI
    GGUI --> CodeStage
    GGUI --> SharpZipLib
    GGResource --> SharpZipLib
    GGUser --> SimpleJson
    GGUser --> StateMachine
```

> 说明：`GGResource --> Core` 等四条边是**目标态**（经 UPM git URL 引用单一源）。现状中除 GGUser 外仍是源码拷贝，见 §2 与 [vendoring-drift](vendoring-drift.md)。

---

## 2. Phase 2 接线进度（现状快照）

粗实线 = 已接线（UPM git URL 引用 Core 并删除本地副本）；虚线 = 待接线（仍持有本地副本）。

```mermaid
graph TD
    Core["GGFramework-Core<br/>tag v0.1.0"]

    GGUser["GGUser<br/>已接线 · EditMode 68/68"]
    GGResource["GGResource<br/>待接（2021.3）"]
    GGNetwork["GGNetwork<br/>待接（2021.3）"]
    GGUI["GGUI<br/>待接（2021.3）"]
    Demo["Demo<br/>待接（2019.4 兼容性待评估）"]

    GGUser ==>|UPM git URL| Core
    GGResource -.->|待接线| Core
    GGNetwork -.->|待接线| Core
    GGUI -.->|待接线| Core
    Demo -.->|待接线| Core
```

> 推广前提：本机仅装 Unity 6000.3.13f1，只能真实验证 GGUser；2021.3/2019.4 仓库接线须在对应版本环境编译验证后再推送。详见 [整体技术方案](technical-solution.md) §6.1。

---

## 3. Core 单一源引用约束（CS0012 防线）

`Singleton<>` 派生类型的程序集**必须直接引用** `GGFramework.Core.Runtime`——asmdef 引用**不具传递性**，主程序集引用了 Core，测试程序集仍需各自引用，否则编译报 **CS0012**。GGUser 试点中两个测试 asmdef 漏加引用即触发此错。

```mermaid
graph LR
    subgraph CorePkg["GGFramework-Core（UPM 包，仓库根即包根）"]
        CoreRuntime["GGFramework.Core.Runtime<br/>Singleton / CoroutineUtil / TaskSystem"]
        CoreTests["GGFramework.Core.Tests"]
        CoreTests --> CoreRuntime
    end

    subgraph Consumer["消费方工程（以 GGUser 为例）"]
        MainAsm["GGFramework.GGUser（主程序集）"]
        EditTests["Tests.EditMode"]
        PlayTests["Tests.PlayMode"]
    end

    MainAsm -->|直接引用| CoreRuntime
    EditTests -->|必须直接引用<br/>否则 CS0012| CoreRuntime
    PlayTests -->|必须直接引用<br/>否则 CS0012| CoreRuntime
```

**另一条约束（类型二义性）**：`Singleton`/`CoroutineUtil` 在全局命名空间，"引用 Core"与"删除本地同名副本"必须**同批原子完成**，否则两份同名类型并存导致二义性编译错误。

---

## 4. 图例约定

| 记号 | 含义 |
|---|---|
| 实线箭头 `-->` | 工程级依赖 / 引用（目标态或已实现） |
| 粗实线箭头 `==>` | 已接线的 UPM git URL 引用 |
| 虚线箭头 `-.->` | 待接线 / 现状源码拷贝（无声明、会漂移） |
| `subgraph` 分层 | 集成层 / 功能模块层 / 基础设施层 / 第三方依赖 |

> 现状与目标的**逐仓库拷贝关系图**（哪份源码被拷进哪个仓库）见 [dependency-graph.md](dependency-graph.md) §1–§2，此处不重复。
