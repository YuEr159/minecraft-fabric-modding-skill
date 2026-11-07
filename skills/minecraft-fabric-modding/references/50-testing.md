# 50 实机测试体系

> 60 站（37 Fabric + 23 NeoForge）逐版本实机回归沉淀的测试体系。本文件回答：
> 怎么把"进游戏里验一遍"变成可重复、可判定、不造假的过程。

## 1. 分层测试体系（实证）

| 层 | 载体 | 能验什么 | 不能验什么 |
|---|---|---|---|
| **T0 单元测试** | `:common:test`（JUnit） | 纯 Java 逻辑：表达式、参数解析、payload 编解码 round-trip | 一切 MC 运行时行为 |
| **T1 服务端自动化** | CI smoke（runServer 到 `Done`）+ 命令树断言 | 模组加载、命令注册、服务端执行路径不炸 | 客户端渲染 |
| **T2 客户端人工瘦测** | 实机（HMCL 实例或启动器） | 粒子可见性、mixin 注入生效、图片/视频显示、双端联机 | 不能自动化，只能规程化 |

**铁律**：服务端负责「注册链 + 不崩」，客户端负责「看得见 + 渲染出来」。CI 绿 ≠ 功能对；真值永远来自实机。

## 2. T2 人工瘦测五判定（实证，每站必跑）

1. **进档 entity id** —— 世界加载成功，玩家实体正常。
2. **命令执行** —— 主命令可执行（有反馈的命令断言内容）。
3. **errprobe 预期失败** —— 故意写错的命令必须报错且错误直出（验证错误不被吞）。
4. **粒子目视** —— 目标视觉效果正确（形状/颜色/寿命）。
5. **日志零异常** —— `logs/latest.log` 扫描红线关键字无命中。

五判定全过才算一站通过。任一项"看不到但应该没事"= 不通过。

## 3. 单存档双身份（实证，10-11 修订版）

- 单存档内 `/op self` ↔ `/deop self` 切换身份测两种视角（早期用双存档，后简化为单档双身份）。
- 管理员身份测命令全量；`/deop self` 后测无权限视角（命令不可见/被拒）。
- **每身份都跑一条原版对照命令**（如 `/gamemode creative`）：若原版命令也报未知，说明是版本全局问题（权限/剪枝），不是 mod 专属。
- 世界/存档**向下不兼容**（高版本建的档低版本打不开）→ 每个版本必须新建存档。

## 4. 数据包函数测试法（实证，绕开聊天栏限制）

- **多行命令必须走数据包函数**：聊天栏单行输入框拒收多行粘贴（26.x 尤其；`/a\n/b` 被吞换行合体成"未知或不完整的命令"）。
- 规程：存档 datapacks 放测试包 → `/datapack enable "file/<name>"`（**引号必须**）→ `/function <ns>:baseline` → `/function <ns>:deep`。
- **函数内 `sendSuccess` 静默**：26.x 起 `/function` 内部所有命令输出被静默（**含原版命令**），`say` 正常；1.21.x 及以下不静默。→ **批量链路的主判据是粒子目视，不是聊天回显。**
- **函数目录单复数**：1.20.x 是复数 `functions/`；1.21.4+ 实测均为单数 `function/`（1.21–1.21.3 未验证）。未知边界用**双目录并放定位法**：`function/` + `functions/` 同时放，看哪边被加载。
- **pack_format 随版本**（20-version-matrix §2）；高 pack 会被判未来版本直接禁用。
- 1.19.x 的 mcfunction **要带斜杠**（1.16/1.17 时代不带）。

**症状三联征**（= 目录名问题）：enable 成功 + 函数全报「未知的函数」+ advancement 探针不加载。

## 5. 实机部署 SOP（实证）

1. **拉产物**：`gh run download` 从该分支最新绿 CI 取 jar；**sha 必须与分支 HEAD 核对**。
2. **实例矩阵**：按 20-version-matrix §4 的版本常量建 HMCL 实例；命名 `<mc版本>-fabric` / `<mc版本>-neoforge`。
3. **mods/ 纪律**：只放对应版本的 jar，**禁止混装**；Fabric 与 NeoForge jar 不能混装（元数据 + 启动钩子不同）；唯一例外是双语 jar（同时含两套元数据）。
4. **被替换的旧 jar 备份为 `.old-*`**；第三方 mod 冲突用 `.disabled` 后缀禁用而非删除（iris/sodium 有强依赖 → 必须成对禁用）。
5. **离线直启**（无 Mojang 认证）：HMCL `launch.bat` 是完整命令行模板；离线化 = 删 authlib-injector javaagent + `--accessToken 0 --clientId HMCL-Offline --xuid 0`，main class 保持 `net.neoforged.fml.startup.Client`。离线直启**不写 latest.log**，stdout 才是全量日志。

## 6. 发布前人工回归清单（骨架）

发布闸门前必须人工确认（CI 不可覆盖，见 40-build-ci §9）：

1. 粒子渲染可见性与图案形状；
2. 颜色/透明度、寿命（age/-1 永久）、销毁观感；
3. 速度表达式逐 tick 表现；
4. 客户端 Mixin 注入生效（表现为 1–3 全部异常时先查 mixin）；
5. 图片/视频像素级显示、旋转/翻转/矩阵变换、缓存重载；
6. 分组增删改的可见效果；
7. **双端联机全流程**（独立服务端发包 → 客户端不断线不崩溃，必须有真实网络往返）；
8. 性能：大批量粒子帧率、上限、长时间挂机内存。

## 7. 日志判定标准（实证）

- 红线关键字：`ERROR` / `Exception` / `Caused by` / `FAILED` / `mixin apply failed` / `Packet type already registered` / `ClassCastException`——命中即失败。
- **噪音豁免清单**（已知无关，判定不影响）：Realms 401、UBO resize、xaero 超时、在线数据过期。每项目自建豁免清单并写明依据。
- 日志编码：Windows 需 `-Dfile.encoding=UTF-8`，否则中文断言误判。

## 8. 测试场纪律（实证）

- 测试用数据包探针**随存档走**，不进仓库不进产物。
- 分支上的探针代码（有行为副作用的死代码）根因关闭后必须 revert；**测试期留、发版前清**。
- CI 产物与实例 jar 一致性三层核对：md5 直比 → artifact 过期时用指纹法（类级字节码比对 / manifest 字段核对）→ 结论可溯源。
- 逐站明细登记（版本｜jar sha｜pack｜函数目录｜平台）回报表化，收官时汇总成总报告（本技能即由 60 站总报告蒸馏）。

## 9. HMCL 实例矩阵

完整版本常量矩阵（19 版本 × 双平台：Fabric Loader / fabric-api 文件 / NeoForge 版本 / Java 版本）见 20-version-matrix §4。要点复述：

- Java：1.20–1.21 线 Java 21；26.x 必须 Java 25。
- 26.x fabric 实例的 fabric-api 必须 ≥ 对应版本号（26.2 实例不能用 26.3 的 fabric-api，loader 会拒载）。
- 1.20.1 NeoForge = Forge 47（HMCL 的 NeoForge 列表里选 Forge 47）。
