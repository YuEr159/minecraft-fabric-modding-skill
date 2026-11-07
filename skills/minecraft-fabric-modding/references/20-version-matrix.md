# 20 版本矩阵：悬崖表与常量

> 全部条目来自 AnotherColorBlock 37 个 Fabric 版本站 + 23 个 NeoForge 版本站的实测（2026-10），
> 每条标注证据等级。适用线：**1.14.4 – 26.3**。表外版本一律标"未验证"。

## 1. 版本悬崖表（跨版本移植必查）

移植到某版本前，先查本表看该版本踩在哪些断代点上。🔴=Fabric 特有，🔵=加载器通用。

### 1.1 低版本段（1.14 – 1.19）

| 断代点 | 结论 | 证据 |
|---|---|---|
| **1.15** | `player.getX/getY/getZ` 是 1.15 新增（`method_23317/23318/23321`）；1.14.4 用 `position()` | 实证 |
| 1.16 / 1.16.1 vs 1.16.5 | 数据包 pack_format 5→6（1.16.2 分界）；mcfunction 不带斜杠 | 实证 |
| **1.17.1 / 1.18.2** | 旧 `ArgumentSerializer` 体系（Serializer 实现 3 方法，无 Template/`type()`/`unpack`；注册走 vanilla `ArgumentTypes.register(String, Class, EmptyArgumentSerializer(Supplier))`；源码实证 1.18.2）；`CommandRegistrationCallback` 是**两参 v1 签名**（1.19.2 起三参带 `registryAccess`）；全链无 `CommandBuildContext`；网络用 `writeResourceLocation(getKey)`/`get(readResourceLocation)`（无 writeId/readById）；`Component` 还是 `new TextComponent`/`TranslatableComponent` | 实证 |
| 1.17.1 / 1.18.2 / 1.19.2 | 🔴 `SingleQuadParticle` 类**不存在**（1.20.x 才引入）；颜色字段 @Accessor 的 mixin target 用父类 `Particle`（1.21.8 及更早一致） | 实证（源码） |
| 1.17 / 1.18 | joml 需手动 `implementation` + `include`（1.10.5）；loom 的 `include` 只进运行时类路径，编译仍需 `implementation` | 实证 |
| **1.19.2** | `ArgumentTypeInfo` 新体系**已存在**（源码实证：`SingletonArgumentInfo.contextFree` + `ArgumentTypeRegistry.registerArgumentType` + `ArgumentTypeInfo.Template`；与 1.19.3+ 同构）→ 从 1.18.2 的 `ArgumentSerializer` 旧体到此完成断代；`Component.literal/translatable` 工厂可用；`BuiltInRegistries` 尚未命名 → `core.Registry.PARTICLE_TYPE`；`particle()` 无参；`sendSuccess` 旧签名；`CommandRegistrationCallback` 三参带 `registryAccess` | 实证（源码） |
| **1.19.3** | `BuiltInRegistries` 命名 + `core.registries` 拆包；`Vec3Argument` 移入 `coordinates` 子包（1.19.2 在 `arguments` 根包）；**`ParticleArgument.particle()` 增加 `CommandBuildContext` 参数**（1.19.2 的 jar 在 1.19.4 实机 `NoSuchMethodError` 实证）→ **1.19.x 不能一份 jar 全覆盖**：1.19.2 分支覆盖 1.19/1.19.1，1.19.3/1.19.4 需独立分支。⚠️ `sendSuccess` 的 Supplier 重载边界不是 1.19.3——1.19.4 编译错误实证 supplier 版 1.20 才引入（提交 a331198） | 实证（源码 + 实机） |

📌 Java 8 目标线（1.14.4/1.15.2/1.16.5）的降级配方见 40-build-ci.md §6。

### 1.2 1.20 段

| 断代点 | 结论 | 证据 |
|---|---|---|
| **1.20** | joml 进入 MC libraries（不再手动加） | 实证 |
| 1.20.x | ⚠️ `markDead` 是 **yarn 名**；Mojang 名下一直是 `remove()`——本项目 1.20.1 起全部用 `remove()`（源码实证，mixin 注释里的"1.20.5+ 改名"说法是映射空间混淆）。看到"某版本改名"先查是哪套命名空间的改名 | 实证（源码 + 提交 6df1e81） |
| **1.20.2** | NeoForge 网络 `MessageConsumer` 直接收 Context（非 Supplier）；🔴 `--enable-preview` 地雷见 40-build-ci.md | 实证 |
| **1.20.4** | `workHandler().execute`（无 `enqueueWork`）；NeoForge `mods.toml` + FML 2 代（见 90-neoforge.md） | 实证 |
| **1.20.5** | `ArgumentTypeInfo` 体系引入但类名仍是 `ArgumentTypes`（复数旧名）——⚠️ 本条来自早期调研笔记，本项目源码未直接使用该区别（见 1.20.6 行说明） | 推测 |
| **1.20.6** | `fromNamespaceAndPath` 尚不存在，只有 `ResourceLocation.of`（源码实证）。⚠️ 网上流传的"1.20.6 起 `ArgumentTypes`→`ArgumentTypeInfos` 改名"在本项目源码中不成立：1.20.1 起 `ArgumentTypeInfos.class` 已作为 @Mixin target 使用（mixin accessor 存在于 1.20.1–26.3 全部分支）。参数类型体系的真正断代在 **1.18.2 → 1.19.2** 之间（见 1.1 节） | 实证（源码逐分支） |

### 1.3 1.21 段（断代最密集）

| 断代点 | 结论 | 证据 |
|---|---|---|
| **1.21 / 1.21.1** | 无 `ParticleRenderer`/`ParticleGroup` 类，粒子 tick 还在 `ParticleEngine` 内 → 需要整套旧版 Mixin；`Commands.hasPermission(int)` 尚不存在，用 `source.hasPermission(2)` | 实证 |
| **1.21.2** | 🔴 粒子队列结构（ParticleGroup）**未变**——仍是旧引擎 ParticleEngine 直管，ParticleGroup 是 **1.21.9** 引入 | 实证（源码） |
| **1.21.6** | `Commands.hasPermission(int)` 才有（此前 `source.hasPermission(2)`） | 实证（源码逐分支） |
| **1.21.9** | ① **粒子 tick 逻辑整体迁入 `ParticleGroup`**（1.21.8 及更早由 `ParticleEngine` 直管 `Map<RenderType, Queue<Particle>>`；ParticleGroup/ParticleLimit 均为 1.21.9 spec 引入，target 大改见 30-mixin-remap）② FML 11.0：`FMLEnvironment.getDist().isClient()`（静态字段 `dist` 已删）③ 🔴 mixin 颜色字段（rCol/gCol/bCol/alpha）从父类 `Particle` 搬到 `SingleQuadParticle` 自身 → @Accessor target 反转 | 实证（源码 + 字节码） |
| **1.21.11** | ① **`ResourceLocation` → `Identifier` 改名在此版本**（1.21.10 及更早全部是 ResourceLocation，源码逐分支核对；不要信"1.21.2 改名"的旧说法）② `ParticleGroup` 队列初始化是 Guava `EvictingQueue.create(16384)`（1.21.9/1.21.10 尚是 `new ArrayDeque<>(N)`）→ 容量 ModifyArg 的 target 必须是 `com/google/common/collect/EvictingQueue.create:(I)...`（remap=false）③ 常量参数注册用 `SingletonArgumentInfo.contextFree(...)`（旧版 `ConstantArgumentTypes` 已无）④ `ParticleEngine.setLevel(ClientLevel)`（不是 setWorld）；`Minecraft.level` 字段（不是 world）⑤ ParticleLimit 重新出现（1.21.9/1.21.10 无该类） | 实证（源码逐分支 + 字节码） |

### 1.4 26.x 段

| 断代点 | 结论 | 证据 |
|---|---|---|
| **26.x 全线** | 无官方 mojmap（ProGuard mappings 停发）→ Fabric 映射生态特殊，见 30-mixin-remap.md；`Minecraft.level`；`ParticleEngine.setLevel(ClientLevel)`；源码类是 `net.minecraft.resources.Identifier`（1.21.10 及以下是 `ResourceLocation`，1.21.11 起改名）——跨分支照抄代码必错，先 `unzip -l` 验证 | 实证 |
| **26.1 → 26.2** | 26.1.2 client jar **无 `Hud` 类**，`gui.getChat()` 直接用；26.2 起拆出 `gui.hud.getChat()` | 实证（26.1 dump + javap） |
| **26.2** | `IParticle` 已合并颜色接口（`setRenderColor`）；1.21.11 是 `setColor` + `setAlpha` 两步；`addParticle`→`createParticle`、`setMaxAge`→`setLifetime` | 实证 |
| **26.x 运行时要求** | Java 25（1.20–1.21 线 Java 21）；Gradle ≥ 9.x | 实证 |

### 1.5 加载器线（🔴Fabric 高价值）

| 断代点 | 结论 | 证据 |
|---|---|---|
| **fabric-api depends id 三态（实证，源码逐分支核对）** | ① 1.17.1（fabric-api 0.46.x）：聚合 mod **id 是 `fabric`** → depends 写 `"fabric"`；② 1.18.2/1.20.1+（≥0.77.0）：id 是 `fabric-api` → 写 `"fabric-api"`；③ **1.19.2 分支写 `"fabric"`**：因为要覆盖 1.19/1.19.1，而那版 fabric-api 0.58.5 的 id 是旧聚合 `fabric` 且新版 fabric-api 带 `provides [fabric]` 别名——写 `fabric` 两端通吃（commit 实证） | 实证 |
| **1.20.x 全线** | 函数目录**复数 `functions/`**（1.20.1/1.20.2/1.20.4/1.20.6 测试包实测一致） | 实证 |
| **1.21.4 – 1.21.11** | 函数目录**单数 `function/`**（1.21.4/1.21.5/1.21.6/1.21.7/1.21.8/1.21.9/1.21.10/1.21.11 测试包实测均为单数；1.21.8 另有字节码实证：`ServerFunctionLibrary` 只有 `"function"` 一个路径常量） | 实证 |
| 1.21 – 1.21.3 | 未验证；历史文档曾记"1.21~1.21.8 复数"，与 1.21.8 字节码及测试包实测矛盾（以实测为准）。存疑版本用**双目录并放定位法**（`function/` + `functions/` 同时放，看哪边被加载） | 未验证 |

## 2. pack_format 全表（实证，数据包测试必查）

| MC 版本 | pack_format |
|---|---|
| 1.13 – 1.14.4 | 4 |
| 1.15 – 1.16.1 | 5 |
| 1.16.2 – 1.16.5 | 6 |
| 1.17 / 1.18 | 9 |
| 1.19.x | 15 |
| 1.20 – 1.20.3 | 15 |
| 1.20.4 | 18 |
| 1.20.5 / 1.20.6 | 41 |
| 1.21 – 1.21.8 | 34 |
| 1.21.9 – 1.21.11 | 121 |
| 26.x | 34 |

- pack_format **高于目标版本会被判为未来版本直接禁用**（121 放到 1.21.8 上不生效）——降版本测试时必须同步改 pack。
- `pack.mcmeta` 可写 `min_format`/`max_format` 放宽。
- `/datapack enable "file/<name>"` **必须带引号**（参数字面量是 `StringArgumentType.string()`，不加引号 `/` 和 `:` 是非法字符）；裸名能解析但包真实 id 是 `file/<name>`。

## 3. 版本常量获取（权威渠道）

| 常量 | 来源 | 说明 |
|---|---|---|
| `minecraft_version` | 版本目标本身 | 如 `1.21.11`、`26.3` |
| `loader_version` | maven.fabricmc.net | fabric-loader 最新稳定 |
| `fabric_version` | maven.fabricmc.net | fabric-api，形如 `0.141.6+1.21.11` |
| `yarn_mappings` | maven.fabricmc.net | 形如 `1.21.11+build.6`；26.x 特殊（见 30-mixin-remap） |
| `neo_form_version` | 格式 **`<MC>-<时间戳>`**（如 `1.21.11-20251209.172050`） | 上游 neoforged/NeoForge 主仓库 gradle.properties 的 `neoform_version` **缺 `<MC>-` 前缀**（快照用），发布版必须补。最可靠取法：`gh search code "neoform_version=1.21.11"` 抄别家已落地项目 |
| `neoforge_version` | projects.neoforged.net / 上游分支 | 如 `21.11.42`、`26.3.0.43-beta` |

- `neo_form_version` **确实被引用**（`common/build.gradle` 的 `neoFormVersion = project.neo_form_version`）——曾误判"未被引用"删行导致配置失败。
- 本机 maven 不可达时的轻量验证：GitHub 代码搜索（`gh search code`）找同版本落地项目抄版本号。

## 4. 版本常量实测矩阵（AnotherColorBlock 各分支 gradle.properties，2026-10）

可作为同版本线的取数参考（版本常量会随时间前进，新分支以官方渠道为准）：

| 分支 | mc | yarn | loader | fabric-api | neoforge |
|---|---|---|---|---|---|
| 1.14.4 | 1.14.4 | — | 0.20.0 | 0.28.5+1.14 | — |
| 1.15.2 | 1.15.2 | — | 0.19.5 | 0.28.5+1.15 | — |
| 1.16.5 | 1.16.5 | — | 0.19.5 | 0.42.0+1.16 | — |
| 1.17.1 | 1.17.1 | — | 0.19.5 | 0.46.1+1.17 | — |
| 1.18.2 | 1.18.2 | — | 0.19.5 | 0.77.0+1.18.2 | — |
| 1.19.2 | 1.19.2 | — | 0.19.5 | 0.77.0+1.19.2 | — |
| 1.19.4 | 1.19.4 | — | 0.19.5 | 0.87.2+1.19.4 | — |
| 1.20.1 | 1.20.1 | 1.20.1+build.10 | 0.19.5 | 0.92.12+1.20.1 | 20.1（Forge 47） |
| 1.20.2 | 1.20.2 | 1.20.2+build.1 | 0.19.5 | 0.91.6+1.20.2 | 20.2.93 |
| 1.20.4 | 1.20.4 | 1.20.4+build.1 | 0.19.5 | 0.97.3+1.20.4 | 20.4.251 |
| 1.20.5 | 1.20.5 | 1.20.5+build.1 | 0.19.5 | 0.97.8+1.20.5 | 20.5.21-beta |
| 1.20.6 | 1.20.6 | 1.20.6+build.3 | 0.19.5 | 0.100.8+1.20.6 | 20.6.135 |
| 1.21.1 | 1.21.1 | 1.21.1+build.3 | 0.19.5 | 0.116.17+1.21.1 | 21.1.252 |
| 1.21.4 | 1.21.4 | 1.21.4+build.8 | 0.19.5 | 0.119.4+1.21.4 | 21.4.136 |
| 1.21.8 | 1.21.8 | 1.21.8+build.1 | 0.19.5 | 0.136.1+1.21.8 | 21.8.50 |
| 1.21.11 | 1.21.11 | 1.21.11+build.6 | 0.18.4 | 0.141.6+1.21.11 | 21.11.42 |
| 26.1 | 26.1.2 | 26.1.2+build.4 | 0.19.3 | 0.145.1+26.1 | 26.1.2.115 |
| 26.2 | 26.2 | 26.2+build.3 | 0.19.3 | 0.156.0+26.2 | 26.2.0.57 |
| 26.3 | 26.3 | 26.3+build.1 | 0.19.5 | 0.162.0+26.3 | 26.3.0.43-beta |

JDK 规则：1.20–1.21 线 = Java 21；26.x = **Java 25**；1.14.4–1.16.5 目标字节码 = Java 8（用 JDK 21 编译，见 40-build-ci.md）。

## 5. 版本覆盖策略：按 range 边界而非版本列表盘点

未覆盖版本盘点法（实证）：按分支 `minecraft_version_range` 的**边界**推导覆盖，不按版本号列表猜：

- `>=1.14.4 <=1.15` 覆盖到 1.15；1.15.1 需放宽 range。
- **1.16.2 是 Registry 断代点**：1.16/1.16.1 更接近 1.15.2 jar 系，不能由 1.16.5 分支覆盖。
- 裸版本（1.21/1.20/1.19/1.18/1.17、1.19.3、1.19.1、1.18.1）可由相邻分支产物 + range 覆盖，前提是 API 无断代（对表 §1）。

合成版本（如 `1.21.11`）与裸版本（`1.21`）的区别在于范围语义：`versionRange = "1.21.4"` 是**裸版本号 = Maven 推荐版本语义 = 任意版本可接受**，不是硬拒绝；`[1.21.5,1.22)` 才是硬范围。第三方 mod 版本依赖判断常踩此坑。

## 6. 已定案的边界（不要再试探）

- **1.13.2 生态真空，放弃**：官方 Fabric 工具链无 1.13.2（intermediary/yarn/loader-meta/fabric-api 全线缺失）；legacy-fabric-api 无 1.13.x 发行；Modrinth 1211 个版本无一条支持 1.13.x。
- **1.12.2 放弃**：连 Brigadier 与数据包 function 都不存在。
- Fabric 线全历史覆盖定格 **37 站**（1.14.4 – 26.3）；NeoForge 线 **23 站**（1.20.1 – 26.3）。
