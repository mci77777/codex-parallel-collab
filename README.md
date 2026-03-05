# codex-parallel-collab

面向 Codex CLI 的并行协作 skill。  
目标是让 Main Codex 先把需求拆成可执行 `CSV TODO`，再调度并行 Codex 执行，最后汇总为可验证交付。

## 正确触发方式（重点）

在提示词里显式写上：

- `$codex-parallel-collab`

不写该前缀时，模型可能不会进入本 skill 的流程。

## Codex CLI 调用方式

### 1) 新任务（推荐）

```bash
echo '$codex-parallel-collab
目标: 实现 X 功能并完成回归
范围: backend/api + service 层
验收: 列出可执行验证命令并全部通过
约束: 不改 CI，不新增三方依赖
请先输出 CSV TODO，再按依赖分批并行执行。' \
| codex exec --skip-git-repo-check --model yolo --sandbox workspace-write --full-auto -C /path/to/repo 2>/dev/null
```

### 2) 仅做拆解与计划（不改代码）

```bash
echo '$codex-parallel-collab
仅做需求拆解，输出任务包与 CSV TODO，不做落盘改动。' \
| codex exec --skip-git-repo-check --model yolo --sandbox read-only -C /path/to/repo 2>/dev/null
```

### 3) 继续上一轮（resume）

```bash
echo '$codex-parallel-collab
继续上一轮，只处理 CSV 中 status=blocked 的任务并给出解除阻塞方案。' \
| codex exec --skip-git-repo-check resume --last 2>/dev/null
```

## 你需要提供的最小输入

- 目标：要交付什么
- 范围：允许改哪些模块，不改哪些模块
- 验收：如何判定完成
- 约束：依赖、性能、安全、时间窗口等

## Skill 会产出什么

- 任务包目录：`.codex/<timestamp>-<slug>/`
- 任务清单：`issues/<timestamp>-<slug>.csv`
- 并行执行过程中的状态更新：`status/notes/evidence`

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

## 执行模型

- Main Codex：需求澄清、拆解、调度、汇总、最终落盘
- Parallel Codex：按分配的 CSV 行执行并回报
- 默认单写者：主 agent 串行写文件，子 agent 优先返回建议，降低冲突

## 性能优化建议

- 按依赖分层并行，避免把可并行任务放进串行链
- 并发建议从 `min(ready_tasks, 4)` 起步，按任务类型动态调节
- 子 agent 只读取必要上下文，避免全仓库灌入
- 子任务先做最小验证，全量验证放到最终阶段

## 常见误用

- 忘记在提示词里写 `$codex-parallel-collab`
- 一次输入多个互不相关的大目标，导致拆解发散
- 并行子任务写入同一文件，造成冲突和回滚成本上升

## 仓库结构

- `SKILL.md`：完整规范（权威）
- `references/subtask-contract.md`：子任务输入输出契约
- `agents/openai.yaml`：展示配置
