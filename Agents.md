# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目上下文

| 文档 | 说明 |
| ----- | ----- |
| [STACK.md](.planning/codebase/STACK.md) | 技术栈、开发命令、部署流程 |
| [STRUCTURE.md](.planning/codebase/STRUCTURE.md) | 目录结构、命名规范 |
| [ARCHITECTURE.md](.planning/codebase/ARCHITECTURE.md) | 架构模式、术语表 |
| [CONVENTIONS.md](.planning/codebase/CONVENTIONS.md) | 代码风格、开发约定 |
| [TESTING.md](.planning/codebase/TESTING.md) | 测试规范 |
| [INTEGRATIONS.md](.planning/codebase/INTEGRATIONS.md) | 外部服务、环境变量 |
| [CONCERNS.md](.planning/codebase/CONCERNS.md) | 技术债务、注意事项 |

更新文档时优先更新到 `.planning/codebase/`。


## Detected stack
- Languages: Rust.
- Frameworks: none detected from the supported starter markers.

## Verification
- Run Rust verification from `rust/`: `cargo fmt`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`
- `src/` and `tests/` are both present; update both surfaces together when behavior changes.

## Repository shape
- `rust/` contains the Rust workspace and active CLI/runtime implementation.
- `src/` contains source files that should stay consistent with generated guidance and tests.
- `tests/` contains validation surfaces that should be reviewed alongside code changes.

## Working agreement
- Prefer small, reviewable changes and keep generated bootstrap files aligned with actual repo workflows.
- Keep shared defaults in `.claude.json`; reserve `.claude/settings.local.json` for machine-local overrides.
- Do not overwrite existing `CLAUDE.md` content automatically; update it intentionally when repo workflows change.
