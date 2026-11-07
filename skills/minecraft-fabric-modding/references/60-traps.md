# 60 环境与工具陷阱

> 这些坑与 MC 版本无关，但每一个都真实卡过几小时。遇到"明明代码没问题"时先对表。

## 1. 本机 Gradle 不可用（实证，Windows）

- **JDK 的 AF_UNIX loopback bug**：`Selector.open()` 在 Microsoft 21.0.6 与 Temurin 25.0.4.1 上必失败（`PipeImpl$Initializer$LoopbackConnector` 走 AF_UNIX 抽象地址，Winsock 不支持，但 `UnixDomainSockets.isSupported()` 误判为可用）→ **本机 Gradle 无法 fork daemon**。
- 结论：**构建验证一律走 CI 或用户终端**（如 `mcp__terminal__run_in_terminal`）；自己判断"构建失败"前先区分是代码问题还是环境问题。
- 全局 `~/.gradle/gradle.properties` 可能钉了特定 JDK（class major 69）→ 需要时用独立 `GRADLE_USER_HOME` 绕开，不动全局配置。
- 依赖 maven 不通时：探测本机代理（如 `127.0.0.1:7897`）+ `GRADLE_OPTS`；`maven.neoforged.net`/`projects.neoforged.net` 在部分网络不可达但 CI 可达。

## 2. Windows 平台细节（实证）

- **PowerShell 5.1 读无 BOM 的 `.ps1` 按系统 ANSI（GBK）解码**，中文脚本里的引号会错位 → 解析失败。修法：脚本加 UTF-8 BOM。
- **`javap`/`java` 无法解析 MSYS 的 `/tmp` 路径**（`/tmp/x.class` 报找不到）→ 必须用 `$TEMP` 或 Windows 绝对路径。
- **Git 2.54 不支持 `git add -q`**。
- git-bash 调 `cmd` 必须绝对路径。

## 3. GitHub CLI / 账号（实证）

- **`gh` 同一时间只认一个活动账号，且会漂移**（实证：从 YuEr159 无操作变回旧账号）。任何外向操作（push、repo create、API 写）前先 `gh api user --jq .login` 确认；切身份 `gh auth switch`。
- **artifact 下载**：`gh api .../actions/runs/<id>/artifacts` 拿 id → `curl -H "Authorization: token $(gh auth token)"` 下载；artifact 会过期（artifacts 0 时改用指纹法核对）。
- **映射号/版本常量查阅替代路径**（maven 不可达时）：
  - intermediary 号 ↔ obf：`FabricMC/intermediary` 仓库 `mappings/<版本>.tiny`。raw 下载大文件易超时 → `gh api repos/FabricMC/intermediary/contents/mappings/<v>.tiny` 拿 sha → `gh api .../git/blobs/<sha> -H "Accept: application/vnd.github.raw+json"`。
  - obf ↔ yarn 名：`FabricMC/yarn` 分支 `mappings/net/minecraft/client/particle/*.mapping`（enigma 格式，METHOD 行 = `METHOD method_X 切 yarn名 (描述符)`）。contents API 按目录列（recursive tree 对大仓库会截断，不能信）。
  - 版本常量：`gh search code "neoform_version=1.21.11"` 抄别家已落地项目。

## 4. 多分支 / 多会话工作纪律（实证，曾两次踩坑）

- 多分支仓库 + 双会话共用同一 checkout：**checkout 前 `git branch --show-current` + `git status --short`，提交后 `git log --oneline -1` 核对落点**（曾两次把修复提交落到错误分支）。
- 跨分支批量修复用脚本 + 每次核对分支名；单分支误提交用 GitHub API 或 `--force-with-lease` 撤销。
- 用 `git show <branch>:<file>` 读其他分支文件；该命令在某些终端环境（MSYS）可能被路径转换破坏 → 先试，失败了用 `git --no-pager show` 或临时 worktree。
- 探针代码（调试用死代码）合并前一律 revert，不留在发布分支。

## 5. 客户端类初始化陷阱（实证）

- **客户端类绝不能进双端 `main` entrypoint**：`Minecraft.getInstance()` 写在 main 入口 → 专用服务器无 `net.minecraft.client` 类，启动必崩。客户端桥接（如消息 sink）必须移到 `client` entrypoint。
- **静态 final 客户端字段 = 异常掩埋器**：`private static final Minecraft CLIENT = Minecraft.getInstance();` 在客户端类初始化前被读 → NPE 掩盖真实异常。修法：删掉静态 final 字段，改**延迟获取 + null 兜底**（回退 `System.out`），让原始异常不再被 NPE 掩盖。

## 6. 表达式引擎/反射教训（实证，纯 Java 逻辑的通用坑）

- **`Method.invoke` 必须同时解包 `Error` 和 `RuntimeException`**：`InvocationTargetException` 的 cause 可能是 `Error`（如向 final 字段赋值时 JVM 抛 `IllegalAccessError`），只解 `RuntimeException` 会让"错误输入必抛"类测试全部类型不符。
- **黄金回归期望值可能被单引号包裹**（`"'java.lang.ArithmeticException: / by zero'"`），判定前先剥包裹引号，否则异常分支永不进入。
- **空串表达式的等价语义**：`parse("")` 返回 null，调用方未判空处 NPE——判定器需补 `null-exe` 等价规则。
- **黄金回归范式可复用**：上游仓库测试用例导出 `golden.json` → 生成一个 Java 断言程序 → 用 `javac`+`java` 直接跑（不依赖 Gradle），在 Gradle 不可用的机器上验证纯 Java 逻辑。
- **科学计数法不支持**（本项目 Lexer 只收数字 + 单个小数点，`1e-6` 的 `e` 被当变量 → "undefine var: e"）→ 容差写 `0.000001`。写表达式引擎时要么支持科学计数法，要么在错误信息里明确提示。

## 7. 测试与统计口径（实证）

- 分析会话记录时**排除 `type: "attachment"` 记录**（CLAUDE.md/memory 注入内容会让 grep 统计严重虚高，把"搬进来的知识"误算成"这里产生的知识"）。
- **上下文压缩会产生 100% 内容重合的 fork/分身文件**——按人类消息集合 + assistant 长文前缀去重，别按文件数统计会话数。
- **不要相信会议纪要里的数字**：凡能回源码/jar 复核的都要复核（曾发现纪要 17 vs 源码 18 的算术错误）。

## 8. 版本生态判断（实证）

- **shaderpack 与加载器无关**（放 `shaderpacks/` 即可）；Iris/Sodium/Lithium 是 CaffeineMC 系双端同源发布（版本号两边一致）；worldedit 是"双语 jar"。
- **NeoForge 生态缺口**：replaymod / litematica / voxy 无 NeoForge 版；1.20.1–1.20.6 NeoForge 无官方 iris/sodium（1.20.1 可用 Oculus 替代）。
- **第三方 mod 版本依赖语义**：`versionRange = "1.21.4"` 是裸版本号 = Maven 推荐版本语义 = 任意版本可接受；`[1.21.5,1.22)` 才是硬范围。曾据此误判"必须禁用某 mod"。
- **不可分发修复 vs 可分发修复**：改本机缓存/配置（如 jij 缓存替换）永不随 jar 走；改 mod jar 内代码则所有玩家生效。发布时正确做法是收窄 `versionRange` 给友好拒绝，而不是教用户改环境。
