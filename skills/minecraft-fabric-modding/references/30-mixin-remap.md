# 30 映射与 Mixin：remap 链路与崩溃排查

> 本文件是本技能最核心的章节——两个月的真实崩溃几乎全部集中在这里。
> 所有结论标注证据等级；字节码级结论可用 §7 的流程现场复现。

## 1. 四套命名空间与运行时空间（先建立心智模型）

| 命名空间 | 是什么 | 何时出现 |
|---|---|---|
| 混淆名（obf） | Mojang 发布的原始 jar 名（`a`, `b`, `fjx`…） | 线上游戏 |
| intermediary（Yarn 号） | Fabric 的稳定中间名 `class_XXX` / `method_XXXXX` / `field_XXXXX` | 🔴 Fabric 运行时空间 |
| yarn | Fabric 的人类可读名 | 开发期（fabric 模块用 yarn 映射时） |
| Mojang 官方名（mojmap） | Mojang 的可读名（`ParticleEngine`…） | 开发期（common/neoforge）；NeoForge 运行时空间 |
| SRG | Forge 的稳定名 `m_XXXXX_` / `f_XXXXX_` | 1.20.1 Forge 47 运行时空间（见 90-neoforge） |

**第一性依据（实证）**：
- 🔴 Fabric 产物运行在 **intermediary 空间**（searge 式 `class_` 号）。无 refmap 时（`mixins.json` 不写 `refmap` 字段），**注解字符串必须已经是 intermediary 号**。
- 🔵 NeoForge 运行时是 **mojmap 空间**（1.20.1 除外，是 SRG）。
- 推论：同一份 mixin 不能假设一套注解字符串通吃两个加载器；Fabric 与 NeoForge 的 mixin 是独立路径，可分别写。

## 2. 🔴 核心事故：mixin 注解「半 remap」

**症状**：本地编译通过、产物正常，进游戏客户端崩溃——`@Shadow ... not located in target class`、`No refMap loaded`、`InvalidAccessorException`、或 PREPARE 阶段 MixinApplyError。

**根因链（实证，tiny-remapper CLI 复现）**：

1. loom 的 `remapJar` 内置 tiny-remapper 的 mixin 扩展：按 `@Mixin` target 类**解析成员**（以编译空间名 + 描述符），再按（成员名, 描述符）查映射树。
2. 解析失败时**只打一条 warn**（`Cannot remap X ... targets [类]`）然后**静默跳过**——不报错、不失败。
3. 于是产物里出现「半 remap」：`@Mixin` 类名转成 intermediary 了，但 `@Shadow`/`@Inject`/`@Redirect` 的成员名字符串残留编译空间名。
4. 运行时无 refmap → 注解字符串被当 intermediary 用 → 找不到 → `defaultRequire: 1` 使整个 mixin 配置变换失败。

**两个实际触发场景**：
- **1.20.1 – 1.21.8**：共享代码里 `@Inject(method="tickParticles")` 写的是 **yarn 名**，而编译空间是 mojmap，mojmap 里该方法叫 `tickParticleList` → 按 mojmap 语义解析失败。
- **1.21.9 / 1.21.10**：MC 把 `tickParticles`/`tickParticle` 整体迁入 `ParticleGroup`（`class_11938`），代码 target 仍是 `ParticleEngine` → 全部注解串解析失败。

**定位手段**：
- 构建日志搜 `Cannot remap`（warn 级，容易被淹没）。
- 小范围复现：tiny-remapper CLI 对产物 jar 跑一遍。
- 产物验证：`unzip -p <jar> <MixinClass>.class | strings | grep <应为remap后的名>`——应为 0 命中。

## 3. 🔴 修复模式：注解字符串直写 intermediary 号

**做法**：`@Inject` 的 `method`、`@Mixin`/`@At` 的 `target` 字符串**直写 intermediary 号**。产物内注解与运行时空间直配，完全绕开 loom remap。已用于 16 个分支（实证）。

**对照组（实证）**：1.21.11 的 mixin target 与 MC 新结构吻合（`@Mixin(ParticleGroup.class)` + 方法确实在 ParticleGroup 里），remap 全链路成功 → 该分支源码**直接写 mojmap 名**（`tickParticles`、`tickParticle`）即可正常工作，无需直写号。**判断依据**：若 target 类正确且成员名在编译空间可解析，走正常 remap；若任一环解析失败（错的 target 类 / 跨映射空间的成员名），才需要直写号。先试正常写法，崩溃再用直写修。

```java
// 反例（依赖 remap，链路上任一环失败就半 remap）
@Inject(method = "tickParticles", at = @At("HEAD"))

// 修复（直写 intermediary，产物与运行时空间直配）
@Inject(method = "method_3048", at = @At("HEAD"))
```

**intermediary 号恒定表（实证）**：

| 号 | 含义 | 恒定范围 | 备注 |
|---|---|---|---|
| `class_703` | `Particle` | 1.14.4 – 1.21.8 | 1.21.9+ 仍是 class_703（未重编） |
| `class_5878` | `ParticleGroup` | （ParticleGroup 存在期） | 1.21.9+ 换 `class_11938` |
| `class_11938` | `ParticleGroup` | 1.21.9+ | MC 结构重构后新号 |
| `method_3048` | `tickParticles` | 1.14.4 – 1.21.8 | ⚠️ **1.19.2 及以前签名带 `Collection` 参数**（`(Ljava/util/Collection;)V`），1.20 起无参 |
| `method_3059` | `tickParticle(Particle)` | 1.14.4 – 1.21.8 | — |
| `method_3070` | `Particle.tick()` | 1.14.4 – 1.21.8 | — |
| `method_3086` | `Particle.isAlive()` | 1.14.4 – 1.21.8 | — |
| `method_74287` | `tickParticles`（ParticleGroup 版） | 1.21.9 – 1.21.10 | 无参；1.21.11 起直写 mojmap 名可正常 remap，无需此号 |
| `method_74288` | `tickParticle(Particle)`（ParticleGroup 版） | 1.21.9 – 1.21.10 | 同上 |

**@Shadow 配套规则**：@Shadow 方法写不了号时，用「唯一前缀同名」的 target 名（比如在 mixin 里声明一个前缀方法把名字占住），确保解析能命中。

**NeoForge 侧**：运行时是 mojmap 空间，直写 mojmap 名即可（`tickParticles`、`tickParticleList` 按版本实测）。

## 4. 🔴 @Accessor / @Shadow 的 owner 精确匹配规则（实证，高频）

**规则**：fabric-loader 做字段/方法 remap 时按 **owner 精确匹配**，**不推导继承关系**。`@Accessor("field_3861")` 的 owner 必须是该字段**真正声明**的那个类，否则以错误 owner 查映射树查不到 → 注解值不 remap（保持字面）→ 运行时字段名是别的 → `InvalidAccessorException` 启动崩溃。

**具体分界线（实证，源码逐分支核对）**：

| 版本 | 颜色字段（rCol/gCol/bCol/alpha）声明在 | @Mixin target 必须是 |
|---|---|---|
| 1.17.1 – 1.21.8 | 父类 `Particle`（`class_703`） | `Particle` |
| ≥ 1.21.9 | `SingleQuadParticle` 自身 | `SingleQuadParticle` |

注：`SingleQuadParticle` 类 1.20.x 才引入，但低版本（1.17.1/1.18.2/1.19.2）的 accessor mixin **同样 target `Particle`**（字段 owner 一致），无需改成 `BillboardParticle`——早期会话中"低版本需改 BillboardParticle"的说法是讨论稿，实际落地代码从未使用。

**「碰巧正确」陷阱**：1.21.9+ 的修复与新结构恰好吻合；如果把 1.21.11 也盲目纳入"旧版批量修复"（target 改回 Particle）**反而会崩**（曾误改一次，用 GitHub API 撤销还原）。批量修复前先判对该分支处于分界线哪一侧。

## 5. 🔴 mixins.json 悬空类引用 = 全量 mixin 瘫痪（实证）

`mixins.json` 里列了已删除的类（代码删了 `ParticleManagerAccessor` 但 json 未删）→ PREPARE 阶段失败 → **整个 mixin 子系统进入错误态**，后续任何类变换都抛 MixinApplyError，最先经过 preLaunch 的某个第三方 mod 会成为"背锅者"。

**排查口诀**：崩溃指认第三方 mod 时，先检查自己的 mixins.json 是否引用了不存在的类。修法：json 与源码同步增删。

## 6. 🔴 26.x 映射生态（实证，与 1.21.x 及以下完全不同）

- **无官方 mojmap**：MC 26.x 起 Mojang 停止发布 ProGuard 映射（`client_mappings`），loom 报 `Failed to find official mojang mappings`（piston-meta manifest 无此键）。
- **yarn 26.x**：迁移为 `org.relativitymc:modern-yarn:<ver>:v2`（发布在 GitHub Packages，需 GPR 认证；maven.fabricmc.net 无 26.3 yarn）——C2ME 同款引用。modern-yarn tiny 是**三列**：official（`net/minecraft/resources/Identifier`）→ intermediary（`class_2960`）→ named（yarn）。
- **intermediary 26.x**：由 RelativityMC 维护（CodeMC 镜像 `repo.codemc.io/repository/relativitymc/`）；官方渠道止于 1.21.11。
- **loom 版本**：1.18-SNAPSHOT 才有 26.x 官方映射获取路径；稳定版 1.17.21 不支持 26.3 官方映射，但 **1.17.20 的无混淆插件路径**（`net.fabricmc.fabric-loom` 新插件 id）在 26.x 上工作良好——26.x = non-obfuscated MC（Mojang 直发可读名 jar），零映射配置，mixin AP 在 javac 阶段直出 official 名（fabric-api 0.162.0+26.3 线上同款）。
- **旧插件 id `fabric-loom` 在 26.x 上断链**：loom 1.18 的 `remapJar` 把字节码 remap 到 official，但 mixin 注解 remap 到 intermediary；loader 无 intermediary→official 映射 → 注解停留中间名 → 客户端崩溃 `No refMap loaded`。走无混淆插件路径即可绕开。
- **loom 1.18 在 26.x 的已知副作用**：`:fabric:tasks --all` 会崩；minecraft jar 缓存落在**项目级** `.gradle/loom-cache/` 而非用户级。
- **`intermediaryUrl` + `useIntermediateMappings=false` 只许本机用**：CI 上它会报 `Mappings src namespace must match remap src namespace, expected official but got intermediary` 及上百个编译错误。
- **fabric-api 26.x 聚合 jar = jar-in-jar**（`META-INF/jars/*.jar`），内嵌模块以官方名发布（如 fabric-command-api-v2 3.1.0 的 `registerArgumentType(net.minecraft.resources.Identifier,...)`）；2.x 旧模块才是 intermediary。loom 会拆成 remapped_mods + 主 jar 进 dev classpath。

## 7. 字节码核验流程（可复现，随时重建事实）

**原则**：映射表改名 ≠ 全部。类/方法级改名用映射表能查；**签名级变革（方法移除、注入点实现变化、队列实现类）只能靠字节码**。曾有一条"1.21.9 起运行时切 mojmap"的推测，被 1.21.8 的 javap 实证推翻。

**本机流程（loom named jar）**：

```bash
# 1. 找 loom 缓存里的 named jar
JAR=$(ls ~/.gradle/caches/fabric-loom/minecraftMaven/net/minecraft/minecraft-clientonly/<ver>-loom.mappings.*/minecraft-clientonly-*.jar | head -1)
# 2. 解出目标类
unzip -p "$JAR" "<类路径>.class" > "$TEMP/x.class"     # ⚠️ Windows 上 javap/java 无法解析 MSYS /tmp，必须 $TEMP 或绝对路径
# 3. 查签名 / 字节码
javap -p "$TEMP/x.class"     # 方法与字段签名
javap -c "$TEMP/x.class"     # 字节码（<init> 内容、字段初始化）
```

**注意（实证）**：loom 的 named jar 是 **NeoForge 分支（NeoForm）反编译产物**，不是纯 vanilla mojmap：
- `ResourceLocation` → `Identifier`（非官方改名，编译期按此写，运行时无影响）
- 批量校验：提取源码全部 `import net.minecraft.*`，逐一在 named jar 中检查类文件存在性。

**本机 maven 不可达时查映射（🔴 Fabric）**：见 60-traps.md §「映射号查阅替代路径」。

**每个新版本分支的例行核验**：对所有 mixin 的注入点与 @Accessor/@Invoker/@Shadow 目标跑一遍本流程。待核验项示例（来自真实待办）：新版本 `ParticleGroup.<init>` 是否仍用 `EvictingQueue`；`Particle.remove()` 是否保留；`SingletonArgumentInfo` 是否仍是常量参数注册路径。

## 8. 其他高频 mixin 陷阱（实证）

- **compatibilityLevel 必须 ≤ 运行时 JVM**：`mixins.json` 写 `JAVA_21` 但用户 JVM 是 17 → **扫描期无声死亡**（日志止于 ModLauncher 扫描、无 crash report、debug.log 不建）。sponge-mixin 源码中声明级别高于运行时直接抛 `IllegalArgumentException`。
- **1.20.1 Forge 47 SRG 直写**：见 90-neoforge.md §4。
- **loom 1.13.6 自身要求 JVM 21 运行**（"Dependency requires at least JVM runtime version 21"），即使 MC 是 Java 17 时代——Gradle 运行层用 JDK 21，MC 本体由 toolchain 17 启动。
- **`@Mixin` target 必须包含被注入的方法**：方法在哪个类，target 就是哪个类。1.21.9 起 `tickParticle*` 在 `ParticleGroup` 不在 `ParticleEngine`；26.x 同理（对照 api-verification 表）。
- **MixinExtras 版本 bug**：NeoForge 26.1.1-beta 内置 mixinextras 0.5.3 的 `@Redirect` 有兼容 CCE；26.1.2（0.5.5）正常——是版本线自身缺陷，不是你的 mod bug。无补丁时收窄 `versionRange` 给友好拒绝，不要教用户改 jij 缓存（不可分发）。
