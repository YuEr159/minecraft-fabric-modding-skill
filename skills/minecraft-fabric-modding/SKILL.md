---
name: minecraft-fabric-modding
description: >
  Minecraft Java 版 Fabric 模组开发的实战经验库（多版本维护向），专门处理跨版本移植、
  mixin 注入崩溃（InvalidAccessorException / MixinApplyError / No refMap）、映射核对
  （intermediary / yarn / Mojang / SRG）、loom 构建配置、CI 启动冒烟与游戏内测试体系
  这类深水区问题。Use when maintaining or porting a Fabric mod across Minecraft
  versions (1.14.4–26.x), debugging mixin apply crashes, verifying mappings at the
  bytecode level, setting up loom/Gradle builds and GitHub Actions smoke tests, or
  designing in-game test protocols. NeoForge covered as contrast. Not for beginner
  block/item tutorials (use minecraft-modding), datapacks, resource packs, server
  plugins, or Bedrock addons.
---

# Minecraft Fabric 模组开发（实战蒸馏版）

本技能的每条知识都来自 AnotherColorBlock 项目 2026-10 的实战（37 个 Fabric 版本站 + 23 个 NeoForge 版本站实机测试、55/55 CI 全绿、16 分支 mixin 批量修复），并标注证据等级：

- **实证**：有 CI、字节码（javap）、产物比对或实机复现作为证据
- **推测**：助手推断或讨论结论，未闭环验证
- **未验证**：明确指出边界未知

## 触发

- 触发：Fabric 模组跨版本移植与多版本维护、mixin 崩溃排查（`InvalidAccessorException` / `MixinApplyError` / `No refMap loaded` / 启动期无声死亡）、产物注解半 remap、intermediary/yarn/Mojang/SRG 映射核对、loom 与 Gradle 配置、`fabric.mod.json`、CI 启动冒烟、游戏内实机测试、数据包测试函数、NeoForge 对照知识。
- 不触发：从零入门加方块/物品/机器（用 `minecraft-modding` 新手版）、数据包/资源包创作、服务端插件（Bukkit/Paper/Spigot）、基岩版附加包、光影、整合包、服务器运维。

## 阶段 0：问诊

读 `references/00-intake.md` 执行：探测项目类型（`fabric.mod.json` → Fabric；`neoforge.mods.toml`/`mods.toml` → NeoForge），从 `gradle.properties` 读版本常量；区分"新写功能"与"跨版本移植/修崩溃"两类任务；确认目标 MC 版本与加载器后才进入路由。探测得到的信息不要重复问。

## 阶段 1：路由

| 问题类型 | 读 |
|---|---|
| 多加载器工程结构、common 抽取、侵入面评估、跨版本成本 | `references/10-architecture.md` |
| 版本差异悬崖、版本常量获取、HMCL 版本矩阵、pack_format/函数目录 | `references/20-version-matrix.md` |
| mixin 注入崩溃、映射号、remap 链路、字节码核验 | `references/30-mixin-remap.md` |
| Gradle/loom/ASM/JDK、GitHub Actions build+smoke | `references/40-build-ci.md` |
| 实机测试协议、数据包测试、HMCL 部署、人工清单 | `references/50-testing.md` |
| 环境与工具陷阱（本机 Gradle、PowerShell、javap、gh、多分支纪律） | `references/60-traps.md` |
| NeoForge 对照：FML 代际、RegisterEvent、SRG、网络 API 三代 | `references/90-neoforge.md` |

## 阶段 2：执行纪律

1. **先核版本与加载器，再动手**。同一段代码在不同版本线可能编译都过不了（`Identifier` vs `ResourceLocation`）。
2. **版本差异不从分支注释或其他版本推断**，用 `javap`/`unzip -l` 对目标版本产物核验。曾有一条"1.21.9 起运行时切 mojmap"的推测被 1.21.8 字节码实证推翻。
3. **给结论标证据等级**。推测写成推测，不冒充实证。
4. **验证分三层**：`gradlew build` 通过 → 服务器启动到 `Done`（CI smoke）→ 游戏内目标行为可见（人工）。编译绿 ≠ 能跑，CI 绿 ≠ 功能对。
5. **做不到的验证显式标注**"待环境（原因）"，绝不假装通过。
6. **多分支仓库操作前** `git branch --show-current` + `git status --short`，提交后 `git log --oneline -1` 核对落点。

## 失败处理

- 目标版本超出 1.14.4–26.3 且无实证覆盖 → 标记"未验证"，说明依据与风险，不猜测。
- 映射号/API 找不到权威来源 → 按 30-mixin-remap.md 的字节码核验流程现场核对，不凭记忆。
- 场景属于新手入门（加方块/物品/配方）→ 转 `minecraft-modding`。
- 构建环境不可用（本机 Gradle 受限）→ 走 CI 或用户终端，见 60-traps.md。

## 语言规则

- 中文为主；术语首现双语（中文 + 英文原词）。
- 代码、API、JSON、命令、报错原文逐字复制，不转写、不翻译。
- modid、包名、路径一律 ASCII 小写。
