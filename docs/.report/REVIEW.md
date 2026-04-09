# Unit Review Log

## Unit 51 — token-budget-feature

- **Status**: Approved
- **原始文档**: https://ccb.agent-aura.top/docs/features/token-budget
- **输出报告**: `docs/.report/51-token-budget-feature.md`
- **审校结果**:
  - 源码锚点精确到文件路径 + `#LXX-LYY` 行号范围。
  - 解析层 (`src/utils/tokenBudget.ts`)、状态层 (`src/bootstrap/state.ts`)、决策层 (`src/query/tokenBudget.ts`)、主循环集成 (`src/query.ts`)、UI 层 (`PromptInput.tsx`, `Spinner.tsx`, `REPL.tsx`)、系统提示 (`src/constants/prompts.ts`)、API 附件 (`src/utils/attachments.ts`) 均已覆盖。
  - 明确区分了 `+500k` 自动续接 feature 与 API-side `task_budget` beta 机制，避免概念混淆。
  - 关键设计决策（90% 阈值、收益递减保护、子 agent 豁免、无条件缓存提示、用户取消清预算）均有对应源码引用。

---

## 36-buddy.md

- **审校时间**: 2026-04-09
- **审校结果**: 通过
- **说明**:
  1. 源码锚点已逐行核对，行号与 `packages/ccb/src/buddy/` 下 8 个文件及 `commands/buddy/buddy.ts` 一致。
  2. 报告覆盖了 Buddy 宠物系统的完整链路：Feature Flag 门控、命令处理、物种/稀有度/属性生成、确定性哈希算法、Mulberry32 PRNG、18 物种的 3 帧精灵图、爱心动画、语音气泡淡出、API 反应触发。
  3. 格式遵循既有报告结构：引言、分节源码映射、源码索引表、附录动画说明。
  4. 关键设计点：
     - 宠物 bones 基于用户 ID 哈希确定性生成，无法通过编辑 config 篡改稀有度
     - `roll()` 函数使用 `rollCache` 缓存，避免重复计算
     - `companionReact.ts` 调用 Anthropic OAuth API 生成宠物反应（需 accessToken）
     - 2026 年 4 月 1-7 日为 teaser window，仅在此期间显示彩虹色 `/buddy` 启动提示
  5. 源码索引表覆盖 36 处精确锚点，状态全部为 ✅。
