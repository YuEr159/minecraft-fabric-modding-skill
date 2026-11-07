# 90 NeoForge 对照

> Fabric 为主、NeoForge 作对照。本文件的价值在于：同一份逻辑搬到 NeoForge 端时，
> 哪些机制变了、哪些"看起来一样但运行时空间不同"。全部 1.20.1–26.3 实机实证。

## 1. FML 代际边界图（2026-10-15 全部实机实证）

排查"jar 被静默跳过/拒载"时先对表：

| MC | NeoForge | FML | mods 文件名 | loaderVersion | 依赖字段 | `${version}` 占位符 | `[[mixins]]` 块 | 运行时命名 | 网络 API |
|---|---|---|---|---|---|---|---|---|---|
| 1.20.1 | 20.1 = Forge 47 fork（**modId=forge**） | 47.2.2 | `mods.toml` | `[24,)` | `mandatory=true` | 不支持（字面量） | **不解析** → MANIFEST `MixinConfigs` | **SRG**（`f_`/`m_` 号） | SimpleChannel |
| 1.20.2 | 20.2.93 | 1.0.16 | `mods.toml` | `[1,)` | `mandatory=true` | 不支持 | 支持 | mojmap | SimpleChannel |
| 1.20.3 | 20.3.8-beta | 1.0.16 | `mods.toml` | `[1,)` | `mandatory=true` | 不支持 | 支持 | mojmap | SimpleChannel |
| 1.20.4 | 20.4.251 | 2.0.17 | `mods.toml` | `[1,)` | `type=required`（过渡期也认 mandatory） | 不支持 | 支持 | mojmap | RegisterPayloadHandlerEvent |
| 1.20.5 | 20.5.21-beta | 3.0.x | `neoforge.mods.toml` | `[3,)` | `type=required` | 支持 | 支持 | mojmap | 同上 |
| 1.20.6 | 20.6.135 | 3.0.45 | `neoforge.mods.toml` | `[3,)` | `type=required` | 支持 | 支持 | mojmap | 同上 |

**九条关键实证**：

1. **FML 2 代（1.0.16/2.0.17）不认 `neoforge.mods.toml` 文件名**——字节码 0 次出现，jar 被静默跳过零警告。
2. **`${version}` 占位符是 FML 3 代特性**——2 代按字面量传 Maven 解析即炸 `Illegal version number`。
3. **JAVA_21 的 mixins.json 在 Java 17 世代 = 扫描期无声死亡**——日志止于 ModLauncher 扫描、无 crash report、debug.log 不建。
4. **FML 1.0.16 只认 `mandatory=true`**，2.0.17 起认 `type=required`。
5. **1.20.1 是 Forge 47**：包名 `net.minecraftforge.fml.loading.FMLEnvironment`（不是 neoforged），modId 依赖写 `forge`。
6. **代差先验**：NeoForge 版本号 ≠ FML 代数（20.3.8-beta 实为 FML 1.0.16 + Java 17 世代）。新版本先查 FML 代、Java 世代、元数据文件名、mixins compatibilityLevel 四件套再动手。
7. 网络 API 三代：SimpleChannel（≤20.3，`MessageConsumer` SAM + `consumerMainThread` 是**字段非方法**）→ RegisterPayloadHandlerEvent（20.4–20.6）→ payload 注册表（1.21+）。
8. **1.20.5 moddev 本机不可构建的机制证据**：moddev 2.x 需要 `neoforge-moddev-bundle` capability，该 variant 打包在 NeoForge **主 jar** 里，而 20.5.21-beta 的 Maven 只有 installer/universal/sources/userdev 四个 jar，主 jar 从未发布。
9. 1.20.6 函数目录是**复数 `functions/`**（单数 `function/` 是 1.21 才引入）。

## 2. 自定义命令参数类型必须走 RegisterEvent（实证，最高价值）

**规则**：1.21.x 全系用 `RegisterEvent` + `Registry.register`（mod bus 事件窗口），**不要字段级直塞**。26.x（26.1–26.3）可维持直塞（其 `ClientboundCommandsPacket` 序列化只查 `ArgumentTypeInfos.BY_CLASS` 不查注册表 id，字节码实证）。

**因果链（1.21.4 七轮探针收口）**：

1. 字段直塞只动 `MappedRegistry` 的 `byId`/`toId` 两套结构。
2. NeoForge 对 `COMMAND_ARGUMENT_TYPE`（在 `VANILLA_SYNC_REGISTRIES` 中）拍 `RegistrySnapshot` 时从 `keySet()`（走 `byKey`/`byLocation`）派生 → **直塞条目被丢**。
3. 客户端进档时 `FrozenRegistrySync` → `applySnapshot` 会 `unfreeze→clear→registerIdMapping` 全量重建 registry。
4. 重建后读侧 `byId(62) = null` → serializer 数据未消费 → 字节流错位 → **`VarInt too big`**。

**分支矩阵**：1.21.1–1.21.4 必崩；1.21.5–1.21.11 **侥幸正确（潜伏风险）**；26.x 机制上安全但建议统一走 RegisterEvent。构造期直接 `Registry.register` 也不行（`Registry is already frozen`）——唯一正路是事件窗口：

```java
modBus.addListener(RegisterEvent.class, e -> {
    if (COMMAND_ARGUMENT_TYPE.key().equals(e.getRegistryKey())) {
        Registry.register(e.getRegistry(), id, info);
    }
});
```

**调试方法论**：统一长度的 Entry 列表 = 读侧按 id 索引的正常表现；读到 null stub = `byId` 返回 null → 把"字节流错位"定位到具体节点偏移。

## 3. 1.21.4 `VarInt too big` 的通用教训

这不是 NeoForge 独有——任何"注册表条目在客户端重建后丢失"都会表现成字节流错位。先查注册是否走了正确窗口，再查网络层。

## 4. 1.20.1（Forge 47）三特化（实证）

- **SRG 直写边界**：运行时成员名是 SRG（`m_107393_` / `f_107289_`）。mixin 类内**所有运行时按名匹配的元素**都要直写 SRG——`@Inject`/`@Redirect` 的 method + target 字符串、`@Accessor` 值、**`@Shadow` 的字段名和方法名**（曾误判为"编译期处理"，连错两轮）。Java 源码其余部分编译期 remap 即可。
- **SRG 号不能跨版本复用**：官方 mappings 双链推导（mojmap→obf→SRG）。
- **FML 47 不解析 `mods.toml` 的 `[[mixins]]` 块**（FML PR#31 起 1.20.2+ 才支持）→ 改用 MANIFEST `MixinConfigs` 属性。
- **注册**：vanilla `Registry.register` 撞 "Can not register to a locked registry" → 必须 `IForgeRegistry.register(RL, V)`。

## 5. FMLEnvironment 代际（实证）

| 版本线 | 写法 |
|---|---|
| 1.21.9+ / 26.x（FML 11.0） | `FMLEnvironment.getDist().isClient()`（静态字段 `dist` 已删） |
| 1.21 – 1.21.8 / 1.20.2–1.20.6 | `FMLEnvironment.dist.isClient()` |
| 1.20.1 | `net.minecraftforge.fml.loading.FMLEnvironment.dist.isClient()`（Forge 47，包名不是 neoforged） |

误用 → `cannot find symbol`。

## 6. NeoForge 侧 mixin 与网络要点（实证）

- **运行时是 mojmap 空间** → mixin 注解直写 mojmap 名（如 `tickParticles`；注意 mojmap 里静态队列方法可能是 `tickParticleList`，按版本实测）。
- MixinExtras 版本缺陷（26.1.1-beta 0.5.3 的 `@Redirect` CCE）→ 收窄 `versionRange` 给友好拒绝（见 30-mixin-remap §8）。
- 权限：`requires(Commands.hasPermission(Commands.LEVEL_GAMEMASTERS))`（1.21.9+/26.x）与 `requires(source -> source.hasPermission(2))`（更早）语义等价 = 权限等级 ≥2。
- 26.x `Holder$Reference.bindComponents` 新建世界 NPE（**推测**：26.3.0.64-beta 组件绑定 + 全局数据包组合问题，与 mod 无关，未复查）。

## 7. gradle 配置（neoforge 模块，moddev 插件）

```groovy
plugins {
    id 'java-library'
    id 'net.neoforged.moddev' version '2.0.144'
}
neoForge {
    version = project.neoforge_version
    runs {
        configureEach {
            systemProperty('neoforge.enabledGameTestNamespaces', project.mod_id)
        }
        client { client() }
        server { server(); programArgument '--nogui' }
        gameTestServer { type = 'gameTestServer' }
        data {
            data()
            programArguments.addAll '--mod', project.mod_id, '--all',
                '--output', file('src/generated/resources/').getAbsolutePath(),
                '--existing', file('src/main/resources/').getAbsolutePath()
        }
    }
    mods { "${project.mod_id}" { sourceSet sourceSets.main } }
}
```

- `neoFormVersion = project.neo_form_version` 在 common/build.gradle 中**必须保留**（见 20-version-matrix §3）。
- NeoForge 侧支持 GameTest（`enabledGameTestNamespaces`），Fabric 侧无开箱能力——这是 T1 测试在 NeoForge 端的额外抓手。
