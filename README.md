# codex-parallel-collab

`codex-parallel-collab` 是一个面向 Codex 的并行协作 skill。  
核心目标是把复杂需求拆成可执行、可验证、可追踪的 `CSV TODO`，由 Main Codex 编排并行 Codex 执行，最终稳定落地。

## 核心定位

- Main Codex：吃透需求、定义验收、拆解任务、调度并行、汇总决策、最终落盘
- Parallel Codex：只执行分配到的 CSV 行，按契约返回结果
- CSV TODO：任务单一事实源（SSOT），承载状态、依赖、Ownership、验证证据

## 标准流程

1. Main Codex 澄清目标、边界、风险、验收标准
2. 生成任务包 `.codex/<timestamp>-<slug>/`
3. 生成 `issues/<timestamp>-<slug>.csv`
4. 按依赖分层并行调度子任务
5. 汇总结果、处理冲突、回填 `status/notes/evidence`
6. Main Codex 串行合并与最终验证

## CSV TODO 最小字段

- `id`
- `task`
- `owner`
- `write_scope`
- `read_scope`
- `depends_on`
- `verify`
- `status`
- `notes`
- `evidence`

## 性能优化要点

- 分层批处理：按依赖拓扑并行，不做无效等待
- 自适应并发：默认 `min(ready_tasks, 4)`，按任务类型上下调
- 最小上下文：子 agent 仅读取 `mission-context.md`、`project.md`、本行 CSV
- 单写者策略：默认 Main Codex 串行落盘，减少冲突与回滚成本
- 快速失败：`BLOCKED` 立即回主 agent 决策，避免整批阻塞
- 验证最小化：子任务做最小验证，最终阶段再做全量验证

## 适用场景

- 多文件独立改动
- 信息收集与分析并行推进
- 需要多视角评审、调试、验证

## 仓库结构

- `SKILL.md`：完整技能规范与执行细节
- `references/subtask-contract.md`：子任务输入/输出契约模板
- `agents/openai.yaml`：技能展示信息

## 触发示例

- “用并行施工把这个需求拆成 CSV TODO，再派发给并行 codex 执行。”
- “请按 codex-parallel-collab 工作流推进这次多模块改造。”
