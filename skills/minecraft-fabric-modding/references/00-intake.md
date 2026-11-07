# 00 问诊（Intake）

> 动手前的探测与确认清单。目标：用最少的问题锁定「加载器 + MC 版本 + 任务类型」。

## 1. 探测项目（不问已知信息）

| 探测到 | 结论 |
|---|---|
| `fabric.mod.json` | Fabric 项目 |
| `neoforge.mods.toml` | NeoForge，FML 3 代（1.20.5+） |
| `mods.toml` | NeoForge/Forge 旧代（1.20.1–1.20.4），或 1.12.2 Forge |
| `gradle.properties` 含 `minecraft_version=` | 主版本常量（多分支仓库每个分支不同） |
| `gradle/libs.versions.toml` | 版本目录写法，常量在这里 |
| `settings.gradle` 的 `include` | 单模块还是 common/fabric/neoforge 三模块 |

版本常量读取（多分支仓库）：`git show <branch>:gradle.properties`，或统一用脚本遍历全部分支：

```bash
for b in $(git branch --format='%(refname:short)'); do
  echo "== $b =="; git show "$b:gradle.properties" 2>/dev/null | grep -E '^(minecraft_version|loader_version|fabric_version|yarn_mappings|neo_form_version|neoforge_version)='
done
```

## 2. 任务类型确认（决定路由）

| 用户说法 | 任务类型 | 首要参考 |
|---|---|---|
| "移植到 1.21.x / 新开一个版本分支" | 跨版本移植 | 20-version-matrix + 30-mixin-remap |
| "进游戏崩了 / mixin 没生效 / InvalidAccessor" | 崩溃排查 | 30-mixin-remap |
| "产物注解是半 remap / Cannot remap" | remap 链路 | 30-mixin-remap |
| "CI 红了 / 需要冒烟 / 构建配不起来" | 构建与 CI | 40-build-ci |
| "怎么做实机测试 / 数据包命令没反应" | 测试体系 | 50-testing |
| "加个 NeoForge 端 / FML 拒载" | NeoForge 对照 | 90-neoforge |
| "想加个方块/物品/机器" | 入门开发 | 非本技能，转 `minecraft-modding` |

## 3. 版本线确认（跨版本任务必问/必查）

1. 目标 MC 版本（精确到 patch，如 `1.21.8` 而非 `1.21`）。
2. 目标加载器（Fabric / NeoForge / 两者）。
3. 仓库是"每版本一分支"还是"单模块多版本"——前者按分支名直达，后者需确认 `minecraft_version_range`。
4. 相邻分支是否存在可作模板（找最近的已测通分支复制，而不是从零）。

## 4. 环境探测（影响验证路径）

| 探测 | 影响 |
|---|---|
| 本机 gradlew 能否跑通（`./gradlew --version`） | 受限则构建验证走 CI/终端（60-traps） |
| `maven.fabricmc.net` / `maven.neoforged.net` 可达性 | 不可达则版本常量查 GitHub（20-version-matrix） |
| 是否有 HMCL / 启动器实例目录 | 决定实机测试路径（50-testing） |
| `gh api user --jq .login` | 确认 GitHub 活动账号（60-traps 账号漂移） |

## 5. 输出路由结论

问诊结束时明确四件事，写进工作记录：

1. 加载器 + MC 版本 + 目标分支/模块
2. 任务类型（移植 / 修崩溃 / 构建 / 测试 / 对照）
3. 模板分支（如有）
4. 验证路径（本机 build / CI / 实机，以及无法覆盖的部分）
