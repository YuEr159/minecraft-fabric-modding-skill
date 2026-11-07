# 40 构建与 CI

> Gradle/loom 配置、依赖陷阱、GitHub Actions 双工作流范式。全部来自 AnotherColorBlock 实证。

## 1. loom 版本与插件 id 矩阵（🔴Fabric）

| MC 线 | loom 版本 | 插件 id | 映射配置 |
|---|---|---|---|
| 1.14.4 – 1.21.11 | 1.13.6（1.17 线及以下）/ 更高稳定版 | `fabric-loom` | 显式 `mappings`（yarn 或 `loom.officialMojangMappings()`） |
| 26.x | 1.17.20 | **`net.fabricmc.fabric-loom`**（新插件 id） | **零映射配置**（无混淆路径，见 30-mixin-remap §6） |

- loom 1.13.6 复用（1.20.1 模板）：loom `include` 仅运行时类路径，编译需 `implementation`。
- 26.x 用旧 id `fabric-loom` 会走混淆 remap 路径，强制要求 mappings，在无官方映射下断链（客户端崩溃 + CI 报 `Configuration 'mappings' has no dependencies`）。

## 2. 依赖陷阱（实证）

- **ASM 版本冲突**：NeoForm 传递 `strictly 9.3/9.6`，与 common 显式声明的 9.7.1 冲突（影响 1.21.2/1.21.4/1.21.6/1.21.8/1.21.10）。**修法：删掉 common 的显式 ASM 依赖，改走传递**。
- **26.x loader 需要外部 ASM**（`org/objectweb/asm/ClassReader` 不在 loader jar）：启动 `java -cp "fabric-loader-<v>.jar;server.jar;asm-9.10.1.jar" net.fabricmc.loader.impl.launch.knot.KnotServer --nogui`。
- **26.x fabric 模块用普通依赖**（无混淆插件没有 `modImplementation` 扩展）：`api "net.fabricmc:fabric-loader:..."`、`implementation "net.fabricmc.fabric-api:fabric-api:..."`。
- **SNAPSHOT 依赖不缓存**：`configurations.all { resolutionStrategy.cacheChangingModulesFor(0, 'seconds') }`。
- **fabric-api depends id 三态**：`fabric.mod.json` 的 `depends` key 在历史上是三种写法——1.17.1 写 `"fabric"`（0.46.x 聚合 id）；1.18.2/1.20.1+ 写 `"fabric-api"`；1.19.2 分支为覆盖 1.19.1（旧 id `fabric` + 新版带 `provides [fabric]` 别名）写 `"fabric"` 两端通吃。写错 → loader 找不到候选，运行必崩（见 20-version-matrix §1.5）。

## 3. JDK 规则与 toolchain（实证）

| MC 线 | 编译/运行 JDK | 字节码目标 |
|---|---|---|
| 1.20 – 1.21 | Java 21 | 21 |
| 26.x | **Java 25**（Gradle ≥ 9.x） | 25 |
| 1.14.4 – 1.16.5 | 用 JDK 21 编译 | **Java 8（major 52）** |

根 build.gradle 动态取 JDK：

```groovy
def requiredJava = providers.gradleProperty('minecraft_version').get().startsWith('26') ? 25 : 21
subprojects {
    apply plugin: 'java'
    java { toolchain.languageVersion = JavaLanguageVersion.of(requiredJava) }
    tasks.withType(JavaCompile).configureEach {
        it.options.encoding = 'UTF-8'
        it.options.release.set(requiredJava)
    }
    tasks.withType(ProcessResources).configureEach { it.setFilteringCharset('UTF-8') }
}
```

## 4. Java 8 降级全套配方（1.16.5 模板，实测首轮 CI 双绿）

- `MethodHandles.Lookup.defineHiddenClass`（Java 15+）→ `ClassLoader.defineClass` 反射；测试 JVM 加 `--add-opens java.base/java.lang=ALL-UNNAMED`。
- `record` → class + accessor；switch 表达式 → 语句；`instanceof` pattern → 强转。
- `mixins.json` 用 `compatibilityLevel: JAVA_8`。
- 编译：`sourceCompatibility/targetCompatibility = '1.8'`（**不用 `--release 8`**——与 loom 的 JDK 21 运行时 javac 冲突报 invalid flag）。
- 产物判据：字节码 major **52**。jij 打包 slf4j 1.7.36。
- 1.14.4 追加断代：`player.getX/getY/getZ` 是 1.15 新增 → 用 `position()`。

## 5. `--enable-preview` 隐形地雷（实证，影响 1.17.1–1.20.2 六分支）

switch **type pattern** 在 Java 17 是 preview；编译产物带 minor `65535` → **用户 JVM 不开 `--enable-preview` 就 `UnsupportedClassVersionError`**。治标=实例 JVM 加参数；**治本=源码改写成 if-instanceof 链**（本项目只有一处 codeGenExp，11 个 case）。

## 6. `fabric.mod.json` 三坑（实证）

- **depends id**：`fabric-api` vs `fabric`（见 §2）。
- **loader 版本下限推理**：`fabricloader >=0.17.0` 的推导 = max(所用 fabric-api 自己声明的 loader 要求, 你的 schema 用到的新特性最低版本)。
- **`minecraft_version_range` 语法**：loader 对 MC 版本号按**字符串语义**比较，`[1.21,1.21.2)` 和 `[1.21,1.21.1]` 都会**误拒 1.21.1**；可用写法 `>=1.21 <=1.21.1`。模板必须用 `${minecraft_version_range}` 展开（写死 `${minecraft_version}` 退化成精确等号）。

## 7. GitHub Actions：build.yml（实测骨架，可直接抄）

```yaml
name: Build & Test
on:
  push: { branches: ['**'] }
  pull_request:
  workflow_dispatch:
permissions: { contents: read }
jobs:
  build:
    strategy: { fail-fast: false, matrix: { module: [fabric, neoforge] } }
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 解析 MC 版本（决定 JDK）
        id: mcver
        run: echo "version=$(grep '^minecraft_version=' gradle.properties | cut -d= -f2)" >> $GITHUB_OUTPUT
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ startsWith(steps.mcver.outputs.version, '26') && '25' || '21' }}
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew :common:test --stacktrace
      - run: ./gradlew :${{ matrix.module }}:build --stacktrace
      - uses: actions/upload-artifact@v4
        with:
          name: modid-${{ matrix.module }}-${{ github.sha }}
          path: |
            **/build/libs/*.jar
            !**/build/libs/*-sources.jar
            !**/build/libs/*-javadoc.jar
          if-no-files-found: error
```

## 8. GitHub Actions：smoke.yml（实测骨架，抓编译期看不见的 bug）

> 首跑就抓出 3 个编译期完全看不见的 bug（eula 缺失、compatibilityLevel、客户端类进服务端入口）。

```yaml
name: Smoke Test（服务器启动冒烟）
on:
  workflow_dispatch:
  push: { branches: ['**'] }
permissions: { contents: read }
concurrency: { group: smoke-${{ github.ref }}, cancel-in-progress: true }
jobs:
  smoke:
    strategy: { fail-fast: false, matrix: { module: [fabric, neoforge] } }
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: mcver
        run: echo "version=$(grep '^minecraft_version=' gradle.properties | cut -d= -f2)" >> $GITHUB_OUTPUT
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: "${{ startsWith(steps.mcver.outputs.version, '26') && '25' || '21' }}" }
      - uses: gradle/actions/setup-gradle@v4
      - timeout-minutes: 15
        run: |
          set -euo pipefail
          rm -rf "${{ matrix.module }}/runs/server" || true
          mkdir -p "${{ matrix.module }}/runs/server"
          echo "eula=true" > "${{ matrix.module }}/runs/server/eula.txt"
          ( ./gradlew :${{ matrix.module }}:runServer --stacktrace > /tmp/runserver.log 2>&1 ) &
          GRADLE_PID=$!
          LOG="${{ matrix.module }}/runs/server/logs/latest.log"
          DONE=""
          for i in $(seq 1 80); do
            if [ -f "$LOG" ] && grep -qE "Done \([0-9.]+s\)|For help, type" "$LOG"; then DONE=yes; break; fi
            if ! kill -0 $GRADLE_PID 2>/dev/null; then cat /tmp/runserver.log; exit 1; fi
            sleep 10
          done
          [ -z "$DONE" ] && { echo "超时：服务器未出现 Done 日志"; cat "$LOG" || true; exit 1; }
          grep -E "Done|Mod loaded|Loading .* mods|Fabric|NeoForge" "$LOG" | tail -10
          if grep -iE "fatal|exception in (the )?(main|server )?thread" "$LOG" | grep -v "nogui" | grep -viE "crash-report-info" | head -5; then
            echo "日志包含致命错误！"; exit 1
          fi
          echo "冒烟通过"
```

**要点**：
- 就绪信号 = `Done ({秒}s)` 或 `For help, type`；轮询 80×10s；超时判失败。
- 致命错误 grep 排除 `nogui` 与 `crash-report-info` 的误伤。
- **构建过 ≠ 运行时加载过**：smoke 补的就是这一环。

## 9. CI 实测经验（实证）

- **编译绿 ≠ 能跑**：CI 只能验服务端半边（启动、命令树注册、执行路径不炸）；客户端渲染、mixin 注入生效、视觉永远人工（见 50-testing）。
- **CI 红线关键字**：`ERROR` / `Exception` / `Caused by` / `FAILED` / `mixin apply failed` / `Packet type already registered` / `ClassCastException`。`WARN` 不算失败。
- **Windows runner 编码**：`JAVA_TOOL_OPTIONS=-Dfile.encoding=UTF-8`，否则中文日志断言误判。
- **Gradle 冷启动**：`org.gradle.daemon=false` 时每次 1–3 分钟；loom 下载缓存建议 cache `~/.gradle/caches` + `~/.gradle/loom-cache`。
- **超时与僵尸进程**：`pkill -P` 或 `timeout-minutes` 兜底。
- **artifact 会过期**：发版/实机测试要尽早 `gh run download`；过期后用指纹法（md5/类级比对）核对实例 jar 与分支 HEAD。
- **dev runServer 的 fabric-api 坑**（🔴 实证）：loader 在 devlaunch 下扫描到聚合 jar 但 `HARD_DEP_NO_CANDIDATE <mod> {depends fabric @ [*]}` → 启动失败。真实服务器（mods/ 目录）才是可靠验证路径。
- **`systemProp.fabric.loom.ci=true`** 可绕开 loom 部分本地检查；`-D` 命令行参数不进 daemon，环境变量每次连接都传最可靠。

## 10. 多层版本矩阵的 CI 策略（实证）

- 每版本分支一套 workflow 会显著增加 CI 时间 → "最新版全量冒烟 + 其余版本仅启动检查"的分级策略。
- `fail-fast: false` 保证一个模块红不影响另一个模块出结果。
- **诚实标注 CI 覆盖率上限**：文档与 PR 描述里如实说明"CI 绿只代表服务端半边"，避免把 CI 绿当成全部测试通过。
