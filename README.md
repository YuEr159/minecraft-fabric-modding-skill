# minecraft-fabric-modding-skill

Minecraft Java 版 **Fabric 模组开发**的实战经验技能包（Claude Code Skill），面向多版本维护场景。

蒸馏自 [AnotherColorBlock](https://github.com/YuEr159/AnotherColorBlock) 项目的真实工程历史：37 个 Fabric 版本站 + 23 个 NeoForge 版本站实机测试、28 分支 CI 全绿、16 分支 mixin 批量修复、60 站实机回归总报告。

## 解决什么问题

- **跨版本移植**：1.14.4 – 26.3 的版本悬崖表（18+ 断代点，逐条实证）
- **mixin 崩溃排查**：注解半 remap 根因链、intermediary 号恒定表、@Accessor owner 精确匹配规则
- **映射核对**：intermediary / yarn / Mojang / SRG 四套命名空间与字节码核验流程（javap 可复现）
- **构建与 CI**：loom 版本矩阵、26.x 无官方映射生态、GitHub Actions build + smoke 双工作流骨架
- **实机测试体系**：T0/T1/T2 分层、单存档双身份、数据包函数测试法、五判定标准
- **NeoForge 对照**：FML 代际图（1.20.1–26.3 全部实机）、RegisterEvent 注册根因、SRG 直写边界

**不是**新手入门教程（加方块/物品/配方请用 `minecraft-modding`），也不覆盖数据包/资源包/服务端插件/基岩版。

## 安装

### 方式一：Claude Code 插件市场

```
/plugin marketplace add YuEr159/minecraft-fabric-modding-skill
/plugin install minecraft-fabric-modding@minecraft-fabric-modding-skill
```

### 方式二：手动复制

```bash
git clone https://github.com/YuEr159/minecraft-fabric-modding-skill.git
cp -r minecraft-fabric-modding-skill/skills/minecraft-fabric-modding ~/.claude/skills/
```

## 结构

```
skills/minecraft-fabric-modding/
├── SKILL.md                    # 触发条件 + 问诊路由 + 执行纪律
└── references/
    ├── 00-intake.md            # 问诊：探测项目、任务分类、环境探测
    ├── 10-architecture.md      # 多加载器架构、common 抽取、跨版本成本公式
    ├── 20-version-matrix.md    # 版本悬崖表、pack_format 全表、版本常量矩阵
    ├── 30-mixin-remap.md       # 【核心】remap 链路、intermediary 号表、字节码核验
    ├── 40-build-ci.md          # Gradle/loom/JDK、build+smoke 工作流骨架
    ├── 50-testing.md           # 分层测试体系、实机部署 SOP、五判定
    ├── 60-traps.md             # 环境与工具陷阱、多分支纪律
    └── 90-neoforge.md          # NeoForge 对照：FML 代际、RegisterEvent、SRG
```

## 证据等级说明

每一条知识都标注了：

- **实证**：有 CI、字节码（javap）、产物比对或实机复现作为证据
- **推测**：推断或讨论结论，未闭环验证
- **未验证**：明确标注边界未知

原则：**映射表改名 ≠ 全部**——签名级变革（方法移除、注入点实现变化）只能靠字节码核验，不凭记忆。

## 许可与致谢

- 知识内容整理自 [AnotherColorBlock](https://github.com/YuEr159/AnotherColorBlock) 社区维护版项目的开发历史（GPL-3.0）。
- 本技能文档以 MIT 许可发布。
