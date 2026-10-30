# GGFramework 能力扩展开发计划（贴纸游戏 Unity 移植驱动）

> **文档状态**：提案 / 草案（Proposed）。本文是**前瞻性能力扩展计划**，不是已拍板的架构决策。
> **驱动来源**：浏览器发行原型《贴纸小日子 / Cozy Sticker Day》（`d:\Works\oi-games\sticker-game\prototype`，含 Node 服务端 `server/`）。其技术文档明确「本原型用于发行展示，**后续正式客户端使用 Unity**」。
> **更新日期**：2026-10-30。
> **与现有文档的关系**：[整体技术方案](../architecture/technical-solution.md) 是「现状治理（消除副本漂移、Phase 0–4）」的统领方案；本文是「面向新游戏（贴纸游戏）落地时，GGFramework 还应补齐/改良哪些能力」的**扩展计划**，不改动既有 Phase 0–4 结论。凡涉及**新建独立仓库、存档后端选型、UGUI 落地**等重大取舍，应在推进时**另立 ADR**（见 §7、§8），本文不代为拍板。

---

## 1. 背景与目的

贴纸游戏原型虽名为「原型」，实际已长成一个**小型商业休闲游戏**的完整骨架：它在纯前端（HTML/CSS/JS + PixiJS）里「手搓」了大量本应属于框架的通用能力——存档（含版本迁移/事务/冲突检测）、音频、经济、本地化、遥测、加载管线、引导等。

移植到 Unity 时，这些正是 GGFramework 应当接管的部分。本文的目的：

1. 把原型暴露出的**真实通用需求**，盘点为 GGFramework 的能力缺口；
2. 给出**拟新增底层模块**清单（按优先级）与**现有模块改良**清单；
3. 给出**分阶段推进路线**与**待决策的开放问题**。

> 方法论：本文每条建议都绑定原型中的具体实现作为需求样本（文件路径相对 `prototype/`），避免凭空设计。

---

## 2. 原型技术画像（需求样本）

| 领域 | 原型实现 | 关键机制（值得框架化的点） |
|---|---|---|
| 启动 / 加载编排 | `js/startup.js`、`js/content-loader.js`、`js/page-loader.js` | 声明式依赖顺序、按需加载、并发合并、失败重试、进度上报 |
| 资源缓存 | `js/asset-cache.js` | **96 MiB 预算 LRU 淘汰**、Blob 复用、`pin/release` 引用计数、`AbortController` 延迟 250 ms 取消、WebP 无损交付变体映射 |
| 存档 / 持久化 | `js/save-repository.js`、`js/store.js` | 版本迁移(v1→v3)、**事务提交 + revision 冲突检测**、多窗口冲突、快照副本隔离、损坏档不覆盖、导入/导出 |
| 经济 / 元系统 | `js/economy.js`、`js/lion-meta.js` | 货币 earn/spend、奖励表驱动、道具库存、每日生命、权益(entitlements)、领取去重 |
| 商业化 | `js/store.js`、`js/product-rules.js` | 模拟 IAP、**待处理订单 token 幂等**、恢复购买、去广告权益(30天/永久)、奖励广告 |
| 账户 / 鉴权 | `js/auth-client.js` + `server/auth.cjs` | session 校验、**过期自动刷新**、401→重登、跨标签页登出同步 |
| 音频 | `js/audio.js` | SFX/BGM 双通道 + 独立音量 + 开关持久化、首次手势解锁、失焦暂停、voice 池上限、BGM ducking、加载失败合成音兜底 |
| 语音陪伴(AI) | `js/companion-voice*.js` + `server/companion-voice-provider.cjs` | 录音→上传→ASR→AI→TTS、能力探测(demo/live)、push-to-talk、权限降级 |
| 每日 / 签到 / 成就 | `js/daily.js`、`js/achievement.js` | UTC+8 日历排期、连续签到、成就条件 + 领取去重 |
| 玩法会话 | `js/session.js`、`js/play-controller.js` | 关内状态机、暂停/撤销、命中检测、**generation 代次失效**、结算 |
| 渲染 / 相机 / 输入 | `js/pixi-board.js`、`js/board-camera.js`、`js/board-input.js` | GPU 渲染 + DOM 回退、缩放/边界/惯性、拖放/双指/键盘 |
| 本地化 | `js/ui-locale.js`、`js/i18n-v3.js` | zh/en 双语、运行时切换 |
| 遥测 / 埋点 | telemetry events + `server/player-monitor.cjs` | 事件环形缓冲、隐私开关、上报 |
| 内容注册 / 热更 | `data/content-registry.json`、`runtime-content` | 统一注册表、按需数据、版本、`npm run catalog` 生成 |
| UI 路由 / 弹窗 | `js/hash-router.js`、`js/modal.js`、`js/ui-motion.js` | 页面路由/深链、模态、Toast、动效 |
| 作品导出 | `js/artwork-composer.js`、`js/lamination.js` | 合成截图、下载/保存、覆膜滤镜 |

---

## 3. 能力映射盘点（现有框架覆盖度）

| 原型子系统 | 对应模块 | 覆盖度 | 说明 |
|---|---|---|---|
| HTTP / 长连接 / 下载 / HTTPDNS | **GGNetwork** | ✅ 覆盖 | UnityWebRequest + WebSocket/Pomelo + DownloadSystem + HTTPDNS，能力强 |
| 登录 / 支付 / SDK / 聚合平台 | **GGUser** | ✅ 大部分 | `ILoginChannel`/`IPayChannel` + `GGPlatform` + 掉单补单骨架已有 |
| UI 页面/窗口/弹窗/加载 | **GGUI** | 🟡 部分 | 五层 UI 生命周期 + 接口层抽象；**UGUI 仍占位、缺路由/Toast/动效** |
| 资源加载双路径 | **GGResource** | 🟡 部分 | Simulation/AB 双路径、优先级、缓存已有；**缺预算淘汰、请求取消/去重、引用计数** |
| 启动依赖编排 | Core + GGResource | 🟡 部分 | 有单例/协程/TaskSystem，缺**启动流程编排器** |
| 对象池 / 状态机 / 事件总线 | Core | 🟡 部分 | GGUser 借第三方 MonsterLove.StateMachine；Core 无 FSM/事件总线/对象池 |
| **存档 / 持久化** | — | ❌ 缺口 | 框架完全没有，原型却很重 |
| **音频** | (Demo `SoundSystem`) | ❌ 缺口 | 非框架级模块 |
| **本地化** | (GGUI NGUI 专属) | ❌ 缺口 | 无框架级 i18n |
| **经济 / 道具 / 元系统** | — | ❌ 缺口 | |
| **每日 / 签到 / 成就** | — | ❌ 缺口 | |
| **广告(Ads)** | — | ❌ 缺口 | GGUser 只有 IAP，无广告 |
| **遥测 / 埋点** | — | ❌ 缺口 | |
| **引导 / 教学** | — | ❌ 缺口 | |
| 玩法渲染 / 相机 / 手势 | (Unity 引擎层) | ⚪ 非框架职责 | Unity 原生 Sprite/相机/Input System 承担，框架至多提供手势/相机辅助 |

---

## 4. 拟新增底层模块

> 命名沿用 `GG*` 前缀；仓库形态（独立仓库 / Core 子包 / 功能模块）待定，见 §7。

### 4.1 P0 — 几乎每个 Unity 游戏都要，且原型已证明需求真实

#### 4.1.1 `GGSave`（存档 / 持久化）—— 最该补的一块

对标 `save-repository.js` / `store.js`，需求清单：

- **后端可插拔**：`PlayerPrefs` / 文件 / 云存档三档（原型是 localStorage；Unity 侧须先决定单机还是云，见 §7）。
- **版本迁移框架**：带 `version` 字段 + 可注册迁移步骤；原型有 v1→v3 迁移、`artRevision` 的 `count-prefix` / `explicit-id-map` 两种策略、旧档只读归档。
- **事务 + 冲突检测**：`Commit(fn)` 语义——写前比对原始串检测并发/多端冲突、失败回滚、损坏档不被默认值覆盖。
- **快照隔离**：`Snapshot()` 返回深拷贝，禁止业务直接改存档对象。
- 导入 / 导出 / 脱敏、写盘节流与 `OnApplicationPause/Quit` 落盘。

#### 4.1.2 `GGAudio`（音频）—— 对标 `audio.js`

- SFX / BGM 双通道 + 独立音量 + 开关持久化（依赖 `GGSave`）。
- 按需加载 + 预热、应用失焦/后台自动暂停恢复、voice 上限与回收。
- BGM ducking（语音/剧情播放时压低背景乐）、加载失败兜底。
- 与 Unity `AudioMixer` 对接（分组/闪避用 Mixer 的 Ducking）。

#### 4.1.3 Core 增补：对象池 / 状态机 / 事件总线 / 代次令牌

- **对象池**：贴纸海量实例化；Demo 的 Lua 层已有 ObjectPool，应下沉为 C# 通用池进 Core。
- **轻量状态机**：玩法会话（`session.js`）与登录支付（GGUser 现借第三方 MonsterLove）都需要；Core 提供零依赖 FSM 可去掉第三方绑定。
- **全局事件总线 / 消息**：原型大量用 `window.dispatchEvent`（`sticker:ready`、`sticker:auth-ending`）解耦，Unity 侧需要中央事件系统。
- **generation / 代次令牌**：原型「离开关卡即失效旧代次，旧加载/回调不影响新关卡」是异步生命周期管理的通用痛点，值得进 Core 或协程工具。
- （可选）**Result / 错误码类型**、**JSON 抽象**：各模块现各自绑 SimpleJson，Core 统一后可减少副本漂移，与既有「副本对齐」治理方向一致。

### 4.2 P1 — 商业休闲游戏标配

| 模块 | 职责 | 对标原型 | 要点 |
|---|---|---|---|
| `GGLocalization` | 框架级 i18n | `ui-locale.js`/`i18n-v3.js` | key-value + 文本/资源本地化 + 运行时切换 + 变更广播；把 GGUI 里 NGUI 专属的 `UILabelLocalization` 提升为后端无关能力 |
| `GGAds` | 广告聚合 | `store.js`/`modes-and-rewards` | `IAdChannel` 抽象 + AdMob/MAX/Unity Ads 适配；激励视频、去广告权益(30天/永久)、插屏；**奖励回调幂等**（对标「待处理 token 匹配」防重复发货）。可作 GGUser 姊妹模块 |
| `GGAnalytics` | 遥测 / 埋点 | telemetry + `player-monitor.cjs` | 事件埋点抽象 + 批量上报（复用 GGNetwork）+ 隐私开关 + 本地环形缓冲 |
| `GGMeta` | 经济/道具/进度元系统 | `economy.js`/`lion-meta.js`/`daily.js`/`achievement.js` | 货币(earn/spend/去重)、道具库存、**奖励表驱动**、每日生命与恢复、权益、成就/签到/每日的**幂等领取 + UTC+8 排期**骨架 |

### 4.3 P2 — 锦上添花

| 模块 | 职责 | 对标原型 |
|---|---|---|
| `GGGuide` | 引导 / 教学 | `tutorial.js`：分步引导、首次说明弹窗、强/弱引导、可跳过 |
| `GGMedia` | 作品导出 / 分享 | `artwork-composer.js`/`lamination.js`：RenderTexture 截图合成、保存相册、分享、覆膜/滤镜 Shader |
| `GGConfig` | 远程配置 / 内容清单 | `content-registry.json`：远程配置拉取 + 内容清单 + 版本 + 热更（Demo 已有 Lua Hotfix，可结合） |

---

## 5. 现有模块改良清单

### 5.1 GGResource（改良优先级最高，对标 `asset-cache.js`）

- **内存预算 + LRU 淘汰**：按估算字节数（`blob.size + w*h*4`）做预算淘汰（原型 96 MiB），正在使用的可临时超预算。
- **请求去重 + 取消 + 引用计数**：并发合并（同 URL 复用一个请求）、`pin/release`、离场后延迟 250 ms 取消未完成请求（避免误杀立即重进）；对应 `UnityWebRequest.Cancel()`。
- **交付变体映射**：对标 WebP 无损副本 + 回退原图（`delivery` map + 哈希校验）；Unity 侧对应 ASTC/ETC2 变体或按需画质档。
- **进度事件化**：把加载进度做成事件，直接喂给 GGUI 的 `UILoading`（原型启动页/关卡加载页实时显示「已载入 N/M」）。

### 5.2 GGNetwork（对标 `auth-client.js` / 语音 transport）

- **Session/Token 生命周期**：token 过期自动刷新、401→触发重登、请求前 session 校验（正好接上 GGUser 的 Login TODO）。
- **请求级超时 + 取消**（原型统一 15s/30s `AbortController`）、**并发合并**。
- **大文件 / 流式上传**：语音陪伴需上传录音（multipart）；当前 DownloadSystem 只管下载，建议补上传通道。
- **REST/JSON 约定层**：原型是标准 REST + JSON，可加一层轻量 RESTful 约定与错误码映射（现有偏「签名 + JsonObject」）。
- 语音 ASR/TTS/AI 的 **provider 抽象**放业务层，transport 复用 GGNetwork。

### 5.3 GGUI

- **落地 UGUI 实现**：当前 `Implementation/UGUI/` 仅占位骨架，而 GGUser 的 Demo 已用 uGUI；移植休闲游戏大概率走 UGUI，这块必须补实（是否弃 NGUI 见 §7）。
- **补 UI 通用件**：Toast、模态/弹窗队列、**页面路由/深链**（当前只有页面栈无路由）、UI 动效系统、骨架屏/占位流光。
- **安全区 / 自适应**：对标原型对 `env(safe-area-inset)`、`visualViewport`、popover 定位（语音面板）的处理——移动端刚需。
- **加载页接资源进度**：`UILoading` 与 GGResource 进度事件打通。

### 5.4 GGUser

- **补 Login 的「游戏服认证/登录」对接**（README 明确的 TODO）——与 GGNetwork 组合，覆盖原型的 session 校验/刷新/登出。
- **多端登出同步**：对标原型 `storage` 事件跨标签页登出；Unity 侧对应「服务端踢下线/异地登录」推送（走长连接）。
- **真实渠道适配器**：`AppleStoreChannel`/`GooglePlayChannel` 目前为骨架。
- 恢复购买/掉单补单已有骨架，可直接对接原型的「待处理订单 token 幂等」语义。

### 5.5 Core

- 见 §4.1.3（对象池 / 状态机 / 事件总线 / 代次令牌）。

---

## 6. 分阶段推进路线

| 阶段 | 内容 | 产出 |
|---|---|---|
| **阶段 0 · 基础设施** | Core 增补（对象池/状态机/事件总线）+ `GGSave` + `GGAudio` + `GGLocalization` | 所有上层的地基；原型已给出完整需求样本 |
| **阶段 1 · 资源管线** | GGResource 补预算淘汰/取消/引用计数/进度事件 + `GGConfig` 内容清单 | 支撑海量贴纸的按需加载与内存可控 |
| **阶段 2 · UI / 导航** | GGUI 落地 UGUI + Toast/路由/动效/安全区，接资源进度 | 完整 UI 外壳 |
| **阶段 3 · 网络 / 账户 / 云存档** | GGNetwork session + GGUser 游戏服认证 + GGSave 云后端 | 单机→联网的关键节点（见 §7） |
| **阶段 4 · 商业化 / 元系统** | `GGAds` + `GGMeta`（经济/成就/每日）+ `GGAnalytics` | 变现与运营闭环 |
| **阶段 5 · 玩法层** | 贴纸场景渲染/相机/拖放（主要靠 Unity 原生 + Input System）+ `GGGuide` + 语音陪伴 + `GGMedia` | 具体游戏玩法 |

> 阶段 5 的「玩法渲染/相机/手势」主要是 Unity 引擎层与游戏工程本身的职责，框架层只提供手势/相机辅助，不作为通用模块强行抽象。

---

## 7. 开放问题 / 待决策（推进时应另立 ADR）

| # | 待决策项 | 影响 | 备注 |
|---|---|---|---|
| Q1 | **单机本地存档 vs 云存档 + 服务端权威结算** | 决定 `GGSave` 后端与阶段 3 是否启动 | 原型技术文档称云存档/权威结算为「后续联网产品化工作」，当前是本地 localStorage |
| Q2 | **UGUI 落地 vs 继续 NGUI** | 决定 GGUI `Implementation/UGUI` 投入与美术管线 | GGUser Demo 已用 uGUI；休闲游戏主流为 UGUI |
| Q3 | **新模块的仓库形态** | 独立 Git 仓库 / Core 子包 / 归入现有功能模块 | 涉及是否复用 Core 单一源模型、是否新增 UPM 包名 |
| Q4 | **`GGAds` 独立 vs 并入 GGUser** | GGUser 已有 IAP 聚合，广告与之同属商业化 | 二者常由同一聚合平台 SDK 提供 |
| Q5 | **命名空间 / asmdef 规范** | 新模块应直接采用 `GGFramework.*` + asmdef 边界 | 与 [technical-solution.md](../architecture/technical-solution.md) Phase 4 收敛方向一致，新模块不再走 vendoring |

> 上述任一项一旦拍板，应新增对应 ADR（遵循 [adr/README](../adr/README.md) 的模板与「一旦接受不再修改正文」约定），本计划文档随之滚动更新引用，而不在正文里代替决策。

---

## 8. 落地约定

新增/改良模块时沿用本仓库既有工程规范（详见 [technical-solution.md](../architecture/technical-solution.md) §5、§7、§8）：

1. **单一源**：共享基础类只从 Core 引用，不再 vendoring；跨仓库文档只在 `GGFramework-Docs` 写一份。
2. **程序集边界**：新模块直接用 `GGFramework.*` 命名空间 + asmdef 分层，测试程序集须**直接**引用 `GGFramework.Core.Runtime`（asmdef 引用不具传递性，否则 CS0012）。
3. **版本锁定**：跨仓库 UPM 引用锁 tag（如 `#v0.1.x`），不用 `#branch`。
4. **测试门槛**：EditMode 用例须核对 `testcasecount > 0`（UPM 包测试须在消费工程 `manifest.json` 的 `testables` 声明，避免零用例假通过）。
5. **提交与门控**：conventional commits 前缀 + 中文描述；每仓独立 commit → L3 深度安全审查（0 findings）→ push；注意 GGUI 用 `master`、其余用 `main`。

---

## 9. 一句话结论

现有 **网络(GGNetwork)、UI 骨架(GGUI)、登录支付(GGUser)** 能较好承接原型的联网与商业化外壳；但原型里最扎实的三块通用能力——**存档（版本迁移/事务/冲突）、音频、本地化**——框架目前是空白，加上 **Core 缺对象池/状态机/事件总线**，应作为移植前的 **P0** 补齐；`GGResource` 的缓存淘汰/请求取消、`GGUI` 的 UGUI 落地与路由/Toast，是现有模块最该完善的点。
