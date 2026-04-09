# 21-skills.md 审校报告

## 审校时间
2026-04-09

## 审校范围

### 已覆盖的内容
1. **Tool vs Skill 本质差异** - 对比表完整，包含粒度、触发方式、本质、注册位置、执行器五个维度
2. **Skill 的五个来源** - 内置命令、磁盘 Skills、Legacy Commands、MCP Skills、用户全局 Skills 均有覆盖
3. **加载链路** - 从 `discover_skill_roots()` 到 `parse_skill_frontmatter()` 的完整流程
4. **核心 API** - `handle_skills_slash_command()`、`classify_skills_slash_command()`、`execute_skill()` 等
5. **路径解析** - `resolve_skill_path()` 两级查找机制
6. **Frontmatter 解析** - YAML 头部解析逻辑
7. **CLI 集成** - REPL 和命令行参数处理
8. **测试验证** - 两个关键测试用例

### 源码锚点质量

| 文件 | 锚点数量 | 精确度 |
|------|----------|--------|
| `commands/src/lib.rs` | 6 处 | L54-L60, L247-L251, L2262-L2291, L2325-L2343, L2654-L2796, L3085-L3159, L3186-L3225 |
| `tools/src/lib.rs` | 6 处 | L558-L569, L1992-L1997, L3021-L3043, L3167-L3197, L6511-L6543, L6568-L6612 |
| `rusty-claude-cli/src/main.rs` | 3 处 | L178-L181, L3652-L3658, L4026-L4037 |

所有锚点均已验证存在于源码中。

## 遗漏内容（与原文对比）

### 原文档有但报告未覆盖的内容

1. **Bundled Skills（编译时打包）** - `src/skills/bundledSkills.ts` 的延迟文件提取机制
   - Rust 版本中对应的实现位置待确认

2. **两条执行路径：Inline vs Fork**
   - 原文档提到 `SkillTool.call()` 根据 `command.context` 分流
   - Rust 版本中 `execute_skill()` 仅返回 prompt 内容，未实现 fork 模式

3. **权限模型：Safe Properties 白名单**
   - 原文档提到 28 个属性的白名单和四层权限检查
   - Rust 版本中 `Skill` 工具的权限为 `PermissionMode::ReadOnly`

4. **Prompt 预算：1% 上下文窗口的截断策略**
   - 原文档提到 `formatCommandsWithatBudget()` 的三级降级
   - Rust 版本中未发现对应的预算截断逻辑

5. **动态发现与条件激活（paths frontmatter）**
   - 原文档提到 `paths` 字段实现条件激活
   - Rust 版本中 `parse_skill_frontmatter()` 仅解析 `name` 和 `description`

6. **使用频率排名**
   - 原文档提到 `skillUsageTracking.ts` 的指数衰减算法
   - Rust 版本中未发现对应的使用追踪逻辑

7. **远程技能加载（Experimental）**
   - 原文档提到 `EXPERIMENTAL_SKILL_SEARCH` feature flag
   - Rust 版本中未实现此功能

## 架构差异说明

### TypeScript 上游 vs Rust 重写版

| 功能 | TypeScript 上游 | Rust claw-code |
|------|----------------|----------------|
| Skill 执行模式 | inline / fork 双模式 | 仅 inline（返回 prompt） |
| Frontmatter 字段 | 17 个字段 | 仅 name, description |
| 权限检查 | 四层权限 + Safe Properties 白名单 | 仅 ReadOnly 权限 |
| 条件激活 | paths 字段 | 未实现 |
| 使用追踪 | 指数衰减算法 | 未实现 |
| 远程技能 | S3/GCS/AKI 加载 | 未实现 |
| Bundled Skills | 编译时打包 + 延迟解压 | 未实现 |

## 结论

本报告**忠实地反映了 `claw-code` Rust 重写版的 Skills 系统实现现状**，而非盲目照搬原文档。

**优点：**
- 所有源码锚点精确且已验证
- 如实描述了 Rust 版本的当前能力边界
- 测试用例覆盖了核心路径解析和加载逻辑

**待补充：**
- 如需实现原文档的高级功能（fork 模式、条件激活、权限白名单等），需在 Rust 版本中新增对应模块

**建议：**
1. 考虑实现 Frontmatter 多字段解析（`allowed-tools`、`context`、`paths` 等）
2. 添加 Skill 执行的 fork 模式支持
3. 实现条件激活机制（基于文件路径的动态 Skill 发现）

---

审校者：AI Assistant
审校方式：源码比对 + 原文对照
