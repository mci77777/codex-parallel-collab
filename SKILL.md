---
name: codex-parallel-collab
description: Main Codex 并行编排工作流：先把用户需求理解透彻并拆解为可执行 CSV TODO，再用 `spawn_agent`/`wait` 调度并行 Codex 分批执行、汇总与递归推进，直到完成可验证交付。适用于多文件独立改动、信息收集+分析、评审/调试/验证并行推进。触发词：并行施工、并行调度、collab、多 agent、parallel、CSV TODO。
---

# Codex 并行施工（Parallel Collab）

> 核心定位：Main Codex 负责深度理解需求与拆解；Parallel Codex 负责按 CSV TODO 执行与回报；主 agent 统一汇总与落地。

## 何时使用

- 任务规模超过单 agent 的上下文或执行窗口
- 子任务可以按文件、模块或阶段解耦
- 需要“并行收集 + 串行决策 + 再并行执行”的节奏

## 角色分工（强约束）

- **Main Codex（主 agent）**：
  - 澄清需求、验收标准、边界与风险
  - 产出任务包（pack）和 CSV TODO（SSOT）
  - 按依赖拓扑分批调度并行子任务
  - 汇总冲突、做最终决策并串行落盘
- **Parallel Codex（子 agent）**：
  - 仅处理分配到的 CSV 行
  - 严格遵守 Ownership（写入范围）
  - 按统一输出契约返回 `STATUS/CHANGES/RISKS/VERIFY/OPEN`

## 工具映射（本环境）

- **并行调度（collab）**：`spawn_agent`、`wait`、`send_input`、`close_agent`
- **并行跑工具**：`multi_tool_use.parallel`
- **落盘写入**：默认 Main Codex 串行 `apply_patch`，避免并行写冲突

## Main Codex 标准执行链（CSV TODO 驱动）

1. **需求澄清**
   - 抽取目标、范围、非目标、验收标准、风险、限制条件
2. **任务包落盘**
   - 目录：`.codex/<timestamp>-<slug>/`
   - 文件（按读序）：`mission-context.md` → `project.md` → `overview.md` → `csv-todo.md`
3. **生成 CSV TODO（SSOT）**
   - 文件：`issues/<timestamp>-<slug>.csv`
   - 每行对应一个可闭环子任务，包含 Ownership 和 Verify
4. **按依赖分批并行调度**
   - 仅调度 `depends_on` 已满足的行
   - 一批完成后再推进下一批
5. **汇总与决策**
   - 合并结果、处理冲突、回填 CSV 状态与证据
6. **最终落盘与验证**
   - Main Codex 串行整合
   - 运行最小验证矩阵并输出最终交付

## CSV TODO 推荐列（最小集合）

- `id`：任务唯一标识
- `task`：可执行动作（1 行闭环）
- `owner`：子 agent 标识或角色
- `write_scope`：允许写入范围（NONE 表示只读）
- `read_scope`：关键读取入口
- `depends_on`：前置任务 id（可空）
- `verify`：最小验证命令
- `status`：`todo|doing|done|blocked`
- `notes`：关键说明
- `evidence`：验证结果或产出链接

## 性能优化策略（重点）

### 1) 分层批处理，不做无效等待
- 按依赖图做拓扑分层；每层一次性并行下发
- 避免把可并行任务放进串行链

### 2) 自适应并发，避免资源抖动
- 默认并发：`min(可执行任务数, 4)`
- IO/检索型任务可提高到 6-8；构建/重测型任务降到 2-3
- 若出现超时/失败潮，主动降并发重试

### 3) 最小上下文分发，降低 token 与歧义
- 子 agent 只读：
  - `mission-context.md`
  - `project.md`
  - 自己负责的 CSV 行
- 禁止把全仓库全文当输入

### 4) 单写者模型，减少冲突成本
- 默认子 agent 只产出建议，不直接落盘
- 必须并行写入时，强制独占文件清单

### 5) 快速失败与小步重试
- 子任务返回 `BLOCKED` 时立即进入主 agent 决策，不等待全批次结束
- 重试只带“新增信息”，避免原样重跑

### 6) 验证最小化
- 子任务只给最小验证命令
- 全量构建/回归留到主 agent 最终阶段

## 子任务输入/输出契约（建议强制）

子任务模板见：`references/subtask-contract.md`。

### 输入模板（发给子 agent）

```text
ROLE: implement | review | debug | validate | docs
ROW_ID: 对应 CSV 行 id
PURPOSE: 1 句话目标
TASK: 1-5 条执行项
SCOPE (WRITE): 允许写入文件/目录（无则 NONE）
SCOPE (READ): 允许读取入口
DO NOT: 禁止事项
OUTPUT: 返回 STATUS/CHANGES/RISKS/VERIFY/OPEN
```

### 输出模板（子 agent 返回）

```text
STATUS: DONE | BLOCKED
CHANGES: 按文件/函数分组
RISKS: 风险与回滚点
VERIFY: 最小验证命令
OPEN: 需要 Main Codex 决策的问题（无则 NONE）
```

## 串行任务处理

强依赖链（A→B→C）必须串行，不强行并行化。每一步先完成最小闭环，再进入下一步。

## 最佳阅读策略

- 子 agent 开工前：`mission-context.md` → `project.md` → 自己的 CSV 行
- MCP 索引只做 1-3 次高信号查询，产出“文件路径 + 不变量 + 风险”
- 默认不并行写入；确需并行写入时严格 Ownership

## 核心不可变原则（强约束）

### 语言规范
- 简体中文回答（本 skill 生效期间默认如此）

### 基本原则
1. 质量第一：代码质量与系统安全不可妥协
2. 思考先行：编码前先拆解与规划
3. Skills 优先：优先使用已存在 Skills 驱动执行
4. 透明记录：关键决策与变更可追溯

### 测试要求
- 覆盖公共函数与边界条件
- 外部依赖要 mock（能不打网络就不打）
- 后台跑单测时，设置最大超时 60s，避免卡死

## 危险操作确认机制（必须执行）

执行以下操作前，必须先向用户发起明确确认：
- 删除文件/目录、批量修改、移动系统文件
- 修改环境变量/系统设置/权限变更
- 数据库删除/结构变更/批量更新
- 发送敏感数据或调用生产环境 API
- 全局安装/卸载、更新核心依赖

确认格式模板：

**⚠️ 危险操作检测！**

**操作类型：** [具体操作]
**影响范围：** [详细说明]
**风险评估：** [潜在后果]

**请确认是否继续？**（需要明确的“是/确认/继续”）

## 终端输出风格指南（如用户要求“强视觉边界”）

- 核心原则：标题独占一行，内容分组明确，视觉边界清晰
- 语言与语气：友好自然、短句优先、避免长段落
- 代码展示：多行必须代码块；示例聚焦关键逻辑
- 反模式：复杂长表格；无意义长路径输出
