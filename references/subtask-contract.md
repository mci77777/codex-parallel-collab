# 子任务契约模板（Parallel Collab）

本文件用于在并行施工时，给每个子 agent 设定一致的输入边界与输出格式，降低汇总成本与冲突概率。

## 1) 子任务输入（发给子 agent）

```text
ROLE: implement | review | debug | validate | docs

PURPOSE: 1 句话目标
TASK:
  - 任务 1
  - 任务 2

SCOPE (WRITE):
  - path/to/file1
  - path/to/dir/**
  - NONE（只读任务必须写 NONE）

SCOPE (READ):
  - path/to/entrypoint
  - path/to/spec

DO NOT:
  - 不改 build/gradle 配置
  - 不引入新依赖
  - 不改不在 SCOPE(WRITE) 内的文件

OUTPUT:
  - 用“按文件分组”的方式给出改动建议
  - 如涉及代码变更：给出最小 patch 片段（可用 unified diff 或关键函数替换）
  - 给出验证命令（越小越快越好）
  - 标出风险与回滚点
```

## 2) 子任务输出（子 agent 返回）

```text
STATUS: DONE | BLOCKED

SUMMARY: 3-7 行总结（面向主 agent 汇总）

CHANGES:
  - file: path/to/file
    why: 为什么要改
    what: 改什么（要点）
    patch: |
      (可选) unified diff/关键代码片段

RISKS:
  - 风险点 1（含影响面与缓解）

VERIFY:
  - 命令 1
  - 命令 2（可选）

OPEN:
  - 需要主 agent 决策的问题（没有则写 NONE）
```

## 3) Ownership（写冲突防线）

并行写入时强制遵守：

- 每个子 agent 只能修改 `SCOPE (WRITE)` 内文件
- 禁止 2 个子 agent 修改同一文件（除非已拆分为不重叠段落且主 agent 明确授权）
- 涉及同一配置源（如依赖版本、构建脚本、CI 配置）默认串行

## 4) 常用 ROLE 约定

- `implement`：只产出最小可合入变更建议 + 验证命令
- `review`：只做风险/一致性/回归面评估，给出需要确认点
- `debug`：定位根因与复现路径，提出最小修复方案（避免“加日志当修复”）
- `validate`：给出最小验证矩阵与可复现命令；发现阻塞则明确 BLOCKED 原因
- `docs`：只改文档/注释/说明（若允许写入）

