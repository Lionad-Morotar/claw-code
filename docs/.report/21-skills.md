# Skills 技能系统技术报告

> 本报告基于 [Claude Code 中文文档 - Skills 技能系统](https://ccb.agent-aura.top/docs/extensibility/skills) 的内容，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。

---

## 一句话定义

**Skill（技能）** 是 **"Prompt 即能力"** 的声明式封装——将复杂任务的最佳实践固化为可复用的 Markdown 文件，通过 `SkillTool` 工具加载并注入到对话流中。

### Tool vs Skill 的本质差异

| 维度 | Tool | Skill |
| --- | --- | --- |
| 粒度 | 单个原子操作（读文件、执行命令） | 一套完整的工作流（代码审查、创建 PR） |
| 触发方式 | AI 自主选择 | 用户 `/skill-name` 或 AI 通过 `SkillTool` 自动匹配 |
| 本质 | TypeScript/Rust 执行逻辑 | **Prompt + 权限配置** 的声明式封装 |
| 注册位置 | `tools/src/lib.rs` → `mvp_tool_specs()` | `commands/src/lib.rs` → `discover_skill_roots()` |
| 执行器 | 各 Tool 的 `run_*()` 函数 | `SkillTool.call()` → 两条分支（inline / fork） |

**核心洞见**：复杂任务的关键不在代码逻辑，而在 Prompt 质量。一个代码审查 Skill 不需要审查引擎，只需告诉 AI "审查什么、按什么顺序、输出什么格式"——Skill 把这种"经验"封装为可复用的 Markdown。

---

## Skill 的五个来源与加载链路

### 1. 内置命令（Built-in Commands）

硬编码在 [`commands/src/lib.rs#L247-L251`](/rust/crates/commands/src/lib.rs#L247-L251) 的 `COMMANDS` 数组中，包含 70+ 条命令（`/commit`、`/review`、`/compact` 等）。这些是 Rust 模块而非 Markdown，但实现了相同的 `SlashCommand` 接口。

### 2. 磁盘 Skills（`.claw/skills/`）

这是最重要的加载路径，由 [`commands/src/lib.rs#L2654-L2796`](/rust/crates/commands/src/lib.rs#L2654-L2796) 的 `discover_skill_roots()` 函数实现：

```rust
fn discover_skill_roots(cwd: &Path) -> Vec<SkillRoot> {
    let mut roots = Vec::new();
    for ancestor in cwd.ancestors() {
        // 项目级：向上遍历至 home
        push_unique_skill_root(&mut roots, DefinitionSource::ProjectClaw,
            ancestor.join(".claw").join("skills"), SkillOrigin::SkillsDir);
        push_unique_skill_root(&mut roots, DefinitionSource::ProjectClaw,
            ancestor.join(".omc").join("skills"), SkillOrigin::SkillsDir);
        push_unique_skill_root(&mut roots, DefinitionSource::ProjectClaw,
            ancestor.join(".agents").join("skills"), SkillOrigin::SkillsDir);
        // ... .codex, .claude 等
    }
    // 用户全局目录
    if let Ok(claw_config_home) = env::var("CLAW_CONFIG_HOME") { ... }
    if let Ok(codex_home) = env::var("CODEX_HOME") { ... }
    if let Some(home) = env::var_os("HOME") { ... }
    roots
}
```

**加载协议**：只识别 `skill-name/SKILL.md` 目录格式，不再支持单文件 `.md`。加载流程：

1. `readdir` 扫描目录 → 仅保留 `isDirectory()` 的条目
2. 在每个子目录中查找 `SKILL.md`，未找到则跳过
3. `parse_skill_frontmatter()` 解析 YAML 头部
4. `create_skill_command()` 构造 `Command` 对象

**去重机制**：使用路径比较避免通过重叠父目录导致的重复加载。

### 3. Legacy Commands（`/commands/` 目录）

向后兼容的旧格式，由 [`commands/src/lib.rs#L2688-L2818`](/rust/crates/commands/src/lib.rs#L2688-L2818) 加载。同时支持 `SKILL.md` 目录格式和单 `.md` 文件格式。标记为 `SkillOrigin::LegacyCommandsDir`。

### 4. MCP Skills（动态发现）

通过 MCP 协议动态发现的 Skills，标记为 `loadedFrom: 'mcp'`。**安全边界**：MCP Skills 的 Prompt 内容禁止执行内联 shell 命令，因为远程内容不可信。

### 5. 用户全局 Skills

通过环境变量加载的全局技能目录：
- `CLAW_CONFIG_HOME/skills`
- `CODEX_HOME/skills`
- `HOME/.claw/skills`
- `HOME/.omc/skills`
- `HOME/.claude/skills`
- `HOME/.claude/skills/omc-learned`

---

## Skill 工具实现：`tools/src/lib.rs`

### Skill Tool Schema

[`tools/src/lib.rs#L558-L569`](/rust/crates/tools/src/lib.rs#L558-L569) 定义了 `Skill` 工具的输入 schema：

```rust
ToolSpec {
    name: "Skill",
    description: "Load a local skill definition and its instructions.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "skill": { "type": "string" },
            "args": { "type": "string" }
        },
        "required": ["skill"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
}
```

### Skill 执行逻辑

[`tools/src/lib.rs#L1992-L1997`](/rust/crates/tools/src/lib.rs#L1992-L1997) 的 `execute_skill()` 函数：

```rust
fn execute_skill(input: SkillInput) -> Result<SkillOutput, String> {
    let skill_path = resolve_skill_path(&input.skill)?;
    let prompt = std::fs::read_to_string(&skill_path).map_err(|error| error.to_string())?;
    let description = parse_skill_description(&prompt);

    Ok(SkillOutput {
        skill: input.skill,
        path: skill_path.display().to_string(),
        args: input.args,
        description,
        prompt,
    })
}
```

### Skill 路径解析

[`tools/src/lib.rs#L3021-L3043`](/rust/crates/tools/src/lib.rs#L3021-L3043) 的 `resolve_skill_path()` 函数实现了两级查找：

```rust
fn resolve_skill_path(skill: &str) -> Result<std::path::PathBuf, String> {
    let cwd = std::env::current_dir().map_err(|error| error.to_string())?;
    // 第一级：委托 commands crate 的多根查找
    match commands::resolve_skill_path(&cwd, skill) {
        Ok(path) => Ok(path),
        // 第二级：兼容模式回退
        Err(_) => resolve_skill_path_from_compat_roots(skill),
    }
}
```

**兼容模式回退** 由 [`tools/src/lib.rs#L3029-L3043`](/rust/crates/tools/src/lib.rs#L3029-L3043) 的 `resolve_skill_path_from_compat_roots()` 实现，遍历所有可能的技能目录。

### 技能目录解析

[`tools/src/lib.rs#L3167-L3197`](/rust/crates/tools/src/lib.rs#L3167-L3197) 的 `resolve_skill_path_in_skills_dir()`：

```rust
fn resolve_skill_path_in_skills_dir(
    root: &std::path::Path,
    requested: &str,
) -> Option<std::path::PathBuf> {
    // 直接匹配：root/requested/SKILL.md
    let direct = root.join(requested).join("SKILL.md");
    if direct.is_file() {
        return Some(direct);
    }
    // 遍历匹配：通过 frontmatter name 匹配
    for entry in std::fs::read_dir(root)? {
        if !entry.path().is_dir() { continue; }
        let skill_path = entry.path().join("SKILL.md");
        if !skill_path.is_file() { continue; }
        if entry.file_name().eq_ignore_ascii_case(requested)
           || skill_frontmatter_name_matches(&skill_path, requested) {
            return Some(skill_path);
        }
    }
    None
}
```

### Frontmatter 解析

[`commands/src/lib.rs#L3186-L3225`](/rust/crates/commands/src/lib.rs#L3186-L3225) 的 `parse_skill_frontmatter()`：

```rust
fn parse_skill_frontmatter(contents: &str) -> (Option<String>, Option<String>) {
    let mut lines = contents.lines();
    if lines.next().map(str::trim) != Some("---") {
        return (None, None);
    }
    let mut name = None;
    let mut description = None;
    for line in lines {
        let trimmed = line.trim();
        if trimmed == "---" { break; }
        if let Some(value) = trimmed.strip_prefix("name:") {
            let value = unquote_frontmatter_value(value.trim());
            if !value.is_empty() { name = Some(value); }
        }
        if let Some(value) = trimmed.strip_prefix("description:") {
            let value = unquote_frontmatter_value(value.trim());
            if !value.is_empty() { description = Some(value); }
        }
    }
    (name, description)
}
```

---

## Skills Slash Command：`commands/src/lib.rs`

### 命令入口

[`commands/src/lib.rs#L2262-L2291`](/rust/crates/commands/src/lib.rs#L2262-L2291) 的 `handle_skills_slash_command()`：

```rust
pub fn handle_skills_slash_command(args: Option<&str>, cwd: &Path) -> std::io::Result<String> {
    match normalize_optional_args(args) {
        None | Some("list") => {
            let roots = discover_skill_roots(cwd);
            let skills = load_skills_from_roots(&roots)?;
            Ok(render_skills_report(&skills))
        }
        Some("install") => Ok(render_skills_usage(Some("install"))),
        Some(args) if args.starts_with("install ") => {
            let target = args["install ".len()..].trim();
            let install = install_skill(target, cwd)?;
            Ok(render_skill_install_report(&install))
        }
        Some(args) if is_help_arg(args) => Ok(render_skills_usage(None)),
        Some(args) => Ok(render_skills_usage(Some(args))),
    }
}
```

### 技能分类与分发

[`commands/src/lib.rs#L2325-L2343`](/rust/crates/commands/src/lib.rs#L2325-L2343) 的 `classify_skills_slash_command()` 和 `resolve_skill_invocation()`：

```rust
pub fn classify_skills_slash_command(args: Option<&str>) -> SkillSlashDispatch {
    match normalize_optional_args(args) {
        None | Some("list" | "help" | "-h" | "--help") => SkillSlashDispatch::Local,
        Some(args) if args == "install" || args.starts_with("install ") => {
            SkillSlashDispatch::Local
        }
        Some(args) => SkillSlashDispatch::Invoke(format!("${}", args.trim_start_matches('/'))),
    }
}
```

**SkillSlashDispatch 枚举**（[`commands/src/lib.rs#L54-L60`](/rust/crates/commands/src/lib.rs#L54-L60)）：

```rust
pub enum SkillSlashDispatch {
    Local,           // 本地处理（list/help/install）
    Invoke(String),  // 技能调用（格式："$skill [args]"）
}
```

### 技能加载与渲染

[`commands/src/lib.rs#L3085-L3159`](/rust/crates/commands/src/lib.rs#L3085-L3159) 的 `load_skills_from_roots()`：

```rust
fn load_skills_from_roots(roots: &[SkillRoot]) -> std::io::Result<Vec<SkillSummary>> {
    let mut skills = Vec::new();
    let mut active_sources = BTreeMap::<String, DefinitionSource>::new();

    for root in roots {
        let mut root_skills = Vec::new();
        for entry in fs::read_dir(&root.path)? {
            // 根据 origin 类型加载 SkillsDir 或 LegacyCommandsDir
            match root.origin {
                SkillOrigin::SkillsDir => {
                    if !entry.path().is_dir() { continue; }
                    let skill_path = entry.path().join("SKILL.md");
                    if !skill_path.is_file() { continue; }
                    let contents = fs::read_to_string(skill_path)?;
                    let (name, description) = parse_skill_frontmatter(&contents);
                    root_skills.push(SkillSummary { ... });
                }
                SkillOrigin::LegacyCommandsDir => { ... }
            }
        }
        root_skills.sort_by(|left, right| left.name.cmp(&right.name));
        // 去重：后加载的会被标记为 shadowed
        for mut skill in root_skills {
            let key = skill.name.to_ascii_lowercase();
            if let Some(existing) = active_sources.get(&key) {
                skill.shadowed_by = Some(*existing);
            } else {
                active_sources.insert(key, skill.source);
            }
            skills.push(skill);
        }
    }
    Ok(skills)
}
```

---

## CLI 集成：`rusty-claude-cli/src/main.rs`

### Skills 命令解析

[`rusty-claude-cli/src/main.rs#L178-L181`](/rust/crates/rusty-claude-cli/src/main.rs#L178-L181) 的 CLI 入口：

```rust
CliAction::Skills {
    args,
    output_format,
} => LiveCli::print_skills(args.as_deref(), output_format)?,
```

### Skills 打印实现

[`rusty-claude-cli/src/main.rs#L4026-L4037`](/rust/crates/rusty-claude-cli/src/main.rs#L4026-L4037)：

```rust
fn print_skills(
    args: Option<&str>,
    output_format: CliOutputFormat,
) -> Result<(), String> {
    let cwd = std::env::current_dir().map_err(|e| e.to_string())?;
    match output_format {
        CliOutputFormat::Text => {
            println!("{}", handle_skills_slash_command(args, &cwd)?);
        }
        CliOutputFormat::Json => {
            println!(
                "{}",
                serde_json::to_string_pretty(&handle_skills_slash_command_json(args, &cwd)?)?
            );
        }
    }
    Ok(())
}
```

### REPL 中的 Skills

[`rusty-claude-cli/src/main.rs#L3652-L3658`](/rust/crates/rusty-claude-cli/src/main.rs#L3652-L3658) 的 REPL 处理：

```rust
SlashCommand::Skills { args } => {
    match classify_skills_slash_command(args.as_deref()) {
        SkillSlashDispatch::Invoke(prompt) => self.run_turn(&prompt)?,
        SkillSlashDispatch::Local => {
            Self::print_skills(args.as_deref(), CliOutputFormat::Text)?;
        }
    }
}
```

---

## 测试验证

### Skill 加载测试

[`tools/src/lib.rs#L6511-L6543`](/rust/crates/tools/src/lib.rs#L6511-L6543) 的 `skill_loads_local_skill_prompt()`：

```rust
#[test]
fn skill_loads_local_skill_prompt() {
    let home = temp_path("skills-home");
    let skill_dir = home.join(".agents").join("skills").join("help");
    fs::create_dir_all(&skill_dir).expect("skill dir should exist");
    fs::write(
        skill_dir.join("SKILL.md"),
        "# help\n\nGuide on using oh-my-codex plugin\n",
    ).expect("skill file should exist");
    std::env::set_var("HOME", &home);

    let result = execute_tool("Skill", &json!({
        "skill": "help",
        "args": "overview"
    })).expect("Skill should succeed");

    let output: serde_json::Value = serde_json::from_str(&result).expect("valid json");
    assert_eq!(output["skill"], "help");
    assert!(output["path"].as_str().expect("path").ends_with("/help/SKILL.md"));
    assert!(output["prompt"].as_str().expect("prompt").contains("Guide on using oh-my-codex plugin"));
}
```

### 项目级技能测试

[`tools/src/lib.rs#L6568-L6612`](/rust/crates/tools/src/lib.rs#L6568-L6612) 的 `skill_resolves_project_local_skills_and_legacy_commands()`：

```rust
#[test]
fn skill_resolves_project_local_skills_and_legacy_commands() {
    let root = temp_path("project-skills");
    let skill_dir = root.join(".claw").join("skills").join("plan");
    let command_dir = root.join(".claw").join("commands");
    fs::create_dir_all(&skill_dir).expect("skill dir should exist");
    fs::create_dir_all(&command_dir).expect("command dir should exist");
    fs::write(skill_dir.join("SKILL.md"), "---\nname: plan\n...").expect("skill file");
    fs::write(command_dir.join("handoff.md"), "---\nname: handoff\n...").expect("command file");

    std::env::set_current_dir(&root).expect("set cwd");

    let skill_result = execute_tool("Skill", &json!({ "skill": "$plan" }))
        .expect("project-local skill should resolve");
    assert!(skill_result.contains(".claw/skills/plan/SKILL.md"));

    let command_result = execute_tool("Skill", &json!({ "skill": "/handoff" }))
        .expect("legacy command should resolve");
    assert!(command_result.contains(".claw/commands/handoff.md"));
}
```

---

## 完整生命周期总结

```
磁盘 SKILL.md
  ↓ parse_skill_frontmatter() → name, description
  ↓ load_skills_from_roots() → SkillSummary
  ↓ 去重（name 大小写匹配 + shadowed_by 标记）
  ↓ render_skills_report() → 格式化输出
  ↓ getSkillDirCommands() memoize 缓存
  ↓ getAllCommands() 合并 local + MCP
  ↓ AI 选择匹配的 Skill
  ↓ SkillTool.validateInput() → 名称校验 + 存在性检查
  ↓ execute_skill() → 读取文件内容
  ↓ SkillOutput { skill, path, args, description, prompt }
```

---

## 源码索引

| 文件 | 核心内容 | 行号 |
|------|----------|------|
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `discover_skill_roots()` | L2654-L2796 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `handle_skills_slash_command()` | L2262-L2291 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `classify_skills_slash_command()` | L2325-L2343 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `load_skills_from_roots()` | L3085-L3159 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `parse_skill_frontmatter()` | L3186-L3225 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `SkillSlashDispatch` | L54-L60 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `Skill` ToolSpec | L558-L569 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `execute_skill()` | L1992-L1997 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `resolve_skill_path()` | L3021-L3043 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `resolve_skill_path_in_skills_dir()` | L3167-L3197 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | Skill tests | L6511-L6765 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI `print_skills()` | L4026-L4037 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | REPL Skills 处理 | L3652-L3658 |
