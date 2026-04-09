# Unit 40 — Teammem 审校记录

## 审校时间
2026-04-09

## 审校结论

### 已修正 / 确认无误

1. **子项目归属明确**：报告开头已标注子项目为 `packages/ccb`，避免与主仓库 `rust/` 实现混淆。
2. **源码锚点精确到行**：所有 `#LXX–#LYY` 锚点均通过 `sed -n` 或文件 grep 在生产源码中二次验证，误差 ≤2 行。
3. **关键数值准确**：
   - `MAX_PUT_BODY_BYTES = 200_000`
   - `MAX_FILE_SIZE_BYTES = 250_000`
   - `DEBOUNCE_MS = 2000`
   - `MAX_CONFLICT_RETRIES = 2`
4. **flow 与文档一致**：Pull（server-wins）与 Push（local-wins）描述与 `index.ts` 函数体注释完全吻合。
5. **安全相关 PSR 编号正确**：M22174（密钥扫描不外传）、M22186（路径穿越/符号链接防护）。

### 发现与建议

- `packages/ccb` 中的 `src/services/teamMemorySync/index.ts` 现有 1256 行，原始文档称其约 1257 行，差异为 1 行（可能是行尾换行版本差异），不影响阅读。
- 原始文档中引用的 `src/services/teamMemorySync/watcher.ts` 实际位于 `packages/ccb/src/services/teamMemorySync/watcher.ts`（388 行），报告已正确映射。
- 仓库中未发现独立的 `__tests__` 目录覆盖 team memory 模块；测试仅通过 watcher 的测试辅助函数（`_resetWatcherStateForTesting`、`_startFileWatcherForTesting`）提供钩子。报告中未虚构测试覆盖率。

### 最终状态
通过。可直接归档。
