# 10 架构：多加载器与多版本工程结构

> 提炼自 AnotherColorBlock 的 common/fabric/neoforge 三模块改造（2026-10 实证，CI 全绿 + 60 站实机）。
> 适用于"一个 mod 要同时支持多 MC 版本或多加载器"的工程决策。

## 1. 核心策略：每 MC 版本一个 git 分支（实证）

- **每个 MC 版本一条分支**（`1.20.1`、`1.21.8`、`26.3`…），分支内是 `common` / `fabric`（/ `neoforge`）三模块。
- **不做运行时多版本兼容**：MC 小版本之间二进制不兼容（方法签名、类结构都在变），一个 jar 通吃多个 MC 版本是伪需求。
- **版本差异全部收敛到 `gradle.properties`**：`minecraft_version` / `yarn_mappings` / `loader_version` / `fabric_version` / `neo_form_version` / `neoforge_version`。分支间 diff 应只有常量 + 真正的 API 适配。
- 裸版本（如 `1.21`、`1.20`）可由相邻分支的产物 + 依赖 range 覆盖，不必单独建分支——前提是 API 无断代（见 20-version-matrix）。

反例教训：上游单模块单 Fabric 项目、每版本复制一份全量代码，导致任何修复要手工重复 N 次。三模块结构下修一次 common 即全线生效。

## 2. 模块划分：「逻辑合、机制分」（实证）

| 模块 | 放什么 | 映射 |
|---|---|---|
| `common` | 命令树、业务数据、payload 数据结构与编解码、纯逻辑算法、语言文件 | Mojang 官方名 |
| `fabric` | 入口类、`fabric.mod.json`、mixin（客户端）、Fabric 网络/命令/粒子注册薄壳、平台抽象实现 | **官方名（不是 yarn）** |
| `neoforge` | `@Mod` 入口、`neoforge.mods.toml`、mixin、NeoForge 注册薄壳 | 官方名（NeoForm） |
| 产物 | `modid-fabric-<mc>-<ver>.jar` + `modid-neoforge-<mc>-<ver>.jar` 两个独立 jar | — |

- **逻辑合**：命令树/表达式/数据/算法只写一份，进 common。
- **机制分**：命令注册（Fabric `CommandRegistrationCallback` / NeoForge `RegisterCommandsEvent`）、网络（`PayloadTypeRegistry`+`PlayerLookup` / `RegisterPayloadHandlersEvent`+`PacketDistributor`）、客户端接收、mixin 各适配各的。

### 关键实证：fabric 模块也用官方映射

`common` 用 `loom.officialMojangMappings()`，**fabric 模块同样用 official 而非 yarn**。否则 common 的 Mojang 名源码在 fabric 模块里对不上 yarn named jar，无法编译。做法源自 ory-jaredlll08/multiloader-template，一次 CI 失败后改的。

26.x 线更进一步：无混淆插件路径（`net.fabricmc.fabric-loom` 1.17+），零映射配置，详见 30-mixin-remap.md。

## 3. common 与平台模块的桥接方式

两种可行方式（本项目用 A）：

- **A. 源码直接并入编译**：`common/build.gradle` 暴露 `commonJava`/`commonResources` 配置（`sourceDirectories`），fabric/neoforge 的 `dependencies` 里 `commonJava(project(path: ':common', configuration: 'commonJava'))`。规避跨映射 jar remap 问题。
- **B. 常规 jar 依赖 + loom remap**：loom 自动把 common 的 Mojang 名产物 remap 成 Yarn（fabric 侧）；neoforge 侧同为 Mojang 名无需 remap。风险是 remap 链路本身出问题（26.x 尤甚）。

## 4. 平台抽象接口（实证，只需 4–5 个）

| 接口 | 职责 | Fabric 实现 | NeoForge 实现 |
|---|---|---|---|
| `IPayloadSender` | 向世界内玩家发 payload | `PlayerLookup.world` + `ServerPlayNetworking.send` | `PacketDistributor.sendToPlayers` |
| `IClientPayloadContext` | 客户端上下文（client/player） | `ClientPlayNetworking.Context` | `ClientPayloadContext` |
| `IParticleSpawner` | 生成粒子 | `particleEngine.createParticle` | 同名（Mojang 名一致） |
| `IMessageSink` | 错误/消息上报 | 聊天框（注意延迟获取，见 60-traps） | 同 |
| `IPlatformPaths` | configDir / gameDir | `FabricLoader.getInstance().getConfigDir()` | `FMLPaths` |

抽样 268 个源文件评估：63% 文件 / 68% 行可直接进 common；抽象上述接口后实际可复用 **80–85%**（**推测**：该数字是统计口径下的评估，非严格实测）。

## 5. 侵入面评估法（实证，判断跨版本成本的核心）

列出模组对原版的**全部注入点**，并明确"完全不碰"的边界。AnotherColorBlock 的实例：

- 侵入面：4 个注入点 / 5 个客户端 Mixin / 约 340 行。
- 不碰：方块、实体、注册表、服务端世界逻辑、存档结构。
- 结论：跨版本成本低，因为 mixin 是仅有的耦合面，而 mixin 目标在版本间变化可枚举（见 30-mixin-remap）。

新项目做多版本决策时，先做这张表：侵入点越多、越深（触及世界/存档结构），跨版本成本越高。

## 6. 跨版本成本经验公式（实证）

≈ **70% 机械翻译 + 20% API 改名 + 10% 真迁移**

实测样本：
- `1.21.8 → 1.21.11`：约 280 行改动
- `1.21.11 → 26.2`：约 1470 行改动，其中八成是 Yarn→Mojang 机械替换（`Identifier`、`ParticleEngine.createParticle` 等改名）

用途：估工与判断是否值得开新版本分支。若真迁移比例超过 30%，考虑砍功能而非硬移植。

## 7. 新版本跟进的硬门槛（实证，26.4 案例）

派生新版本分支前，三个外部依赖必须齐：

1. MC 正式版已发布（snapshot 的 API 不稳定）
2. NeoForge 有对应版本产物（maven 有该版本线）
3. intermediary/yarn 有该版本映射（maven 有表项，fabric-loader 能装）

三者缺一就先挂起。26.4 挂起时的状态可作为参照：MC 26.4 仅 snapshot、neoforge maven 无 26.4 线、intermediary 26.4 空、仅 fabric-api 有 snapshot 构建。

## 8. 发布工程要点（实证）

- **产物命名冲突**：fabric/neoforge 同名 jar 互相覆盖 → `archivesName` 必须含加载器后缀：`"${mod_id}-fabric-${minecraft_version}"`。
- **元数据版本硬编码**：`mods.toml` 里写死 version 会让所有 jar 显示 1.0.0 → 用 `${version}` 模板 + `processResources` 的 `filesMatching` 展开；展开目标文件名要与源文件一致（`neoforge.mods.toml`）。
- **探针生命周期**：测试期留、发版前清。数据包探针随存档走不进仓库；分支上的探针代码（有行为副作用的死代码）根因关闭后必须 revert。
- **main 重建决策**：当 main 落后于最新版本分支且无独有内容 → 直接 fast-forward + 单独 README 提交，`--force-with-lease` 锚定旧 tip。
