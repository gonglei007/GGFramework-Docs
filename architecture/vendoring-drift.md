# 源码拷贝集成与副本漂移

## 1. 什么是 vendoring（源码拷贝集成）

GGFramework 各模块之间没有工程级依赖声明，"复用"靠**把源码文件直接拷进另一个仓库**。拷贝进来的副本与被拷贝的源头**各自独立演进**，久而久之内容不再一致——这就是**副本漂移（drift）**。

## 2. 拷贝范围

| 被拷贝内容 | 拷入位置 |
|---|---|
| 基础类 `Singleton` / `CoroutineUtil` / `GameDebugger` / `TaskSystem` / `ZipUtil` | 每个模块各存一份于 `Scripts/Dependency/Utils` |
| GGResource 整包 | `GGUI/Scripts/Dependency/Resource`、Demo |
| `UIAdaptor` / `LogAdaptor` | GGNetwork、GGUser 各一份 |
| 全部 4 个模块共 138 个文件 | Demo 的 `ggsource/Scripts`（扁平、全局命名空间） |

## 3. 漂移清单

- **68 组同名文件中，41 组内容已漂移。**
- 典型多版本：
  - `Singleton` ×5，存在 2 个版本（GGUser 版含非泛型 `SingletonLifetime.IsQuitting` 修复；其余为旧版）。
  - `ResourceSystem` ×3，存在 3 个版本。
  - `GameDebugger` ×3，存在 3 个版本（行数/功能均不同）。

## 4. 漂移的危害（真实案例）

审查中发现的 **P0 缺陷**正是漂移的直接证据：

- canonical 模块 **GGResource** 的 `ResourceSystem.GetAssets<T>` 写着 `resourceObject.assetObject.GetType() is T`——`GetType()` 返回 `Type`，`Type is T` **恒为真**，泛型过滤形同虚设。
- 而 **GGUI 里的拷贝副本**早已被修成正确的 `assetObject is T`。
- 即：**下游副本修好了，源头反而是错的**，且修复没有回流。这类"反向漂移"在纯拷贝模式下无法被任何机制发现。

其它同源问题：任一模块修 bug / 重构 / 加特性，都不会传播到其它副本；要同步只能人工逐个 diff，成本极高且易漏。

## 5. 治理方向

1. **抽单一源**：把零第三方依赖的共享基础类（`Singleton`/`CoroutineUtil`/`TaskSystem`）收敛进 `GGFramework-Core`（已落地，见 [ADR-0001](../adr/0001-core-独立仓库.md)）。
2. **UPM 引用替代拷贝**：消费方经 UPM git URL 引用 Core，锁定 tag 版本（见 [ADR-0002](../adr/0002-vendoring-转-upm-迁移路线.md)）。
3. **引用与删副本同批**：因 `Singleton`/`CoroutineUtil` 在全局命名空间，接线时必须"引用 Core + 删除本地同名副本"一次完成，否则类型二义性编译错误。
4. **含第三方耦合者延后**：`GameDebugger`（耦合 CodeStage.AdvancedFPSCounter）、`ZipUtil`（耦合 SharpZipLib）、`UIAdaptor`/`LogAdaptor`（SimpleJson）留待 Phase 2，以可选子程序集形式评估纳入。

## 6. 如何自检漂移

对任意同名文件跨仓库比对（示例）：

```powershell
# 比较两个仓库里同名基础类是否一致
$a = "d:\Works\GGFramework\GGFramework-GGResource\...\Singleton.cs"
$b = "d:\Works\GGFramework\GGFramework-GGUser\...\Singleton.cs"
if ((Get-FileHash $a).Hash -ne (Get-FileHash $b).Hash) { "DRIFT: $a vs $b" }
```

批量普查思路：按文件名分组统计出现次数 >1 的文件，再对每组做哈希/diff，哈希不一致即为漂移。
