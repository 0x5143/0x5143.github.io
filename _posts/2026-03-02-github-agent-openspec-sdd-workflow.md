---
title: GitHub Agent + OpenSpec SDD 纯 Issue 驱动工作流整合方案
date: 2026-03-02
type: engineering
status: stable
tags:
  - github-actions
  - openspec
  - sdd
  - copilot
  - agent-skill
  - workflow-automation
  - methodology
---

## 1. 这是什么问题？

我希望探索一种将 **GitHub Copilot Coding Agent**、**OpenSpec SDD（Spec-Driven Development）** 和 **GitHub Actions CI** 整合在一起的工作流，实现：

- 代码与文档分离（单仓库双目录）
- 以 Issue 为核心驱动 SDD 全生命周期
- 新增 Issue 自动触发 CI 生成 SDD 骨架
- SDD 完成后可自动分配给 Copilot Agent 实现

**核心约束：必须可落地、可执行，零额外依赖。**

---

## 2. 核心结论 / 判断

**纯 Issue 驱动 > Project Board 同步展示。**

经过多轮迭代验证，最终方案砍掉了 GitHub Project Board，理由是：

- Project Board 在此场景中仅作为"只读镜像"，无决策价值
- 引入 Project 需要 PAT Token、GraphQL mutation、组织级权限等大量额外依赖
- Issue Label 本身就是天然的状态机，完全可以替代 Board 的状态流转

砍掉后，整个方案：

| 维度 | 含 Project Board | 纯 Issue（最终版） |
|------|-----------------|-------------------|
| Secrets | 1 PAT + 5 Variables | 0（仅 GITHUB_TOKEN） |
| GraphQL 调用 | 4 处 mutation | 0 |
| 落地难度 | 需 Org Admin 配合 | 仓库 Maintainer 即可 |
| 可移植性 | 依赖组织级 Project | 任何仓库即开即用 |

---

## 3. 设计理念

```
Issue 是"第一公民"，Label 是"状态机",
Milestone 是"版本线”，PR 是"交付物"。
不需要一个额外的看板来"展示"这些已经存在的东西。
```

---

## 4. 前置条件

| # | 条件 | 确认方式 |
|---|------|---------|
| 1 | Node.js ≥ 20.19.0 | `node -v` |
| 2 | OpenSpec CLI | `npm install -g @fission-ai/openspec@latest` |
| 3 | Copilot Coding Agent 已启用 | Repo → Settings → Copilot → Agent ✅ |
| 4 | Actions 读写权限 | Repo → Settings → Actions → General → Read and write ✅ |

> 不再需要 PAT Token、PROJECT_TOKEN Secret、任何 Project Variables。所有 Workflow 仅使用自动注入的 `GITHUB_TOKEN`。 

---

## 5. 仓库目录结构（单仓库双目录）

```
my-project/
├── src/                                # 💻 代码
│   └── ...
├── .openspec/                          # 📄 SDD 文档
│   ├── config.json
│   ├── specs/                          #   源真相 (Source of Truth)
│   │   └── <capability>/spec.md
│   └── changes/                        #   活跃变更
│       └── <change-id>/
│           ├── proposal.md
│           ├── tasks.md
│           ├── design.md              (可选)
│           └── specs/<domain>/spec.md
├── .github/
│   ├── workflows/
│   │   ├── 01-sdd-init.yml
│   │   ├── 02-sdd-validate.yml
│   │   ├── 03-sdd-implement.yml
│   │   └── 04-sdd-archive.yml
│   ├── ISSUE_TEMPLATE/
│   │   └── sdd-request.yml
│   └── copilot-instructions.md
└── README.md
```

**关键区分：**

- `src/` — 纯代码，Copilot Agent 的工作区域
- `.openspec/` — 纯文档，SDD Pipeline 的工作区域
- `.github/` — 自动化胶水层

---

## 6. Label 状态机（替代 Project Board）

```
Issue 生命周期：

[sdd] ─── Workflow 01 ───▶ [sdd:spec-ing]
                                 │
                           团队完善 SDD
                           Review & Merge PR
                                 │
                                 ▼
                          [sdd:spec-ready]
                                 │
                           Workflow 03
                           创建实现 Issue
                                 ▼
                       [sdd:implementing]
                       + [copilot-assigned]
                                 │
                          Copilot / 人工
                          提交 & 合并实现 PR
                                 ▼
                           [sdd:done]
                           Issue 自动关闭
```

**所需 Labels：**

| Label | 颜色 | 说明 |
|-------|------|------|
| `sdd` | `#7057ff` | SDD 流程入口 |
| `sdd:spec-ing` | `#fbca04` | SDD 文档编写中 |
| `sdd:spec-ready` | `#0e8a16` | SDD 文档已就绪 |
| `sdd:implementing` | `#1d76db` | 代码实现中 |
| `sdd:done` | `#0e8a16` | 已完成并归档 |
| `copilot-assigned` | `#8b5cf6` | 已分配给 Copilot Agent |
| `auto-implement` | `#d93f0b` | SDD 合并后自动交给 Copilot |

**日常查询：**

```
所有 SDD 进行中:    is:open label:sdd:spec-ing
等待实现:           is:open label:sdd:spec-ready
实现中:             is:open label:sdd:implementing
Copilot 处理中:     is:open label:copilot-assigned
已完成:             label:sdd:done is:closed
```

---

## 7. 两套命令体系（避免混淆）

| Shell CLI（CI/CD 中使用） | AI IDE 内 OPSX 指令（开发者在 IDE 中使用） |
|--------------------------|------------------------------------------|
| `openspec init` | `/opsx:new <change>` |
| `openspec validate` | `/opsx:ff` |
| `openspec archive` | `/opsx:continue` |
| `openspec --version` | `/opsx:apply <change>` |
| | `/opsx:verify <change>` |
| | `/opsx:archive <change>` |

> ⚠️ CI/CD 工作流中只能使用 Shell CLI，不能使用 `/opsx:` 指令。OPSX 指令是 AI 编码助手（Claude/Cursor/Windsurf）中的交互式命令。

---

## 8. 四个核心 Workflow

### 8.1 Workflow 01：Issue[sdd] → 生成 SDD 骨架 → 创建 Draft PR

**触发条件：** Issue 被创建或被打上 `sdd` 标签

**执行逻辑：**

1. 解析 Issue 模板中的 Change Name、Summary、验收标准
2. 幂等性检查（分支是否已存在）
3. 在 `.openspec/changes/<change-id>/` 生成 `proposal.md`、`tasks.md`、`specs/`
4. 创建 `sdd/<change-id>` 分支，提交并推送
5. 创建 Draft PR
6. Issue 标签从 `sdd` → `sdd:spec-ing`
7. 在 Issue 中评论 SDD Pipeline 状态

```yaml
name: "01 📐 SDD Init"

on:
  issues:
    types: [opened, labeled]

permissions:
  contents: write
  issues: write
  pull-requests: write
```

### 8.2 Workflow 02：SDD 文件变更 → 自动校验

**触发条件：** PR 中有 `.openspec/**` 文件变更

**执行逻辑：**

1. 检查每个 change 目录的文件完整性（proposal.md、tasks.md、specs/）
2. 检查内容非空（proposal 至少 5 行，tasks 至少 2 个任务项）
3. 运行 `openspec validate --all`（如果已初始化）
4. 在 PR 中评论校验报告
5. 有缺失必需文件时 CI 失败

```yaml
name: "02 ✅ SDD Validate"

on:
  pull_request:
    paths:
      - '.openspec/**'

permissions:
  contents: read
  pull-requests: write
```

### 8.3 Workflow 03：SDD PR 合并 → 创建实现 Issue → 可选分配 Copilot

**触发条件：** `sdd/*` 分支的 PR 被合并到 main

**执行逻辑：**

1. 从分支名提取 change-id
2. 读取 `tasks.md` 和 `proposal.md` 内容
3. 检查原始 SDD Issue 是否有 `auto-implement` 标签
4. 创建实现 Issue，body 中包含完整的 SDD 上下文
5. 如果 auto-implement，`assignees: ['copilot']` → 触发 Coding Agent
6. SDD Issue 标签 → `sdd:spec-ready`
7. 在 SDD Issue 中评论链接

```yaml
name: "03 🤖 SDD Merged → Implement"

on:
  pull_request:
    types: [closed]
    branches: [main]

permissions:
  contents: read
  issues: write
```

> **关于 Copilot Agent 触发方式：** 通过 REST API 将 Issue 的 assignee 设为 `"copilot"` 即可触发。Agent 会自动分析仓库、读取 SDD 文件、创建分支、编写代码、运行测试，最终提交一个 Draft PR。迭代反馈发生在 PR 评论中，而非 Issue 评论中。

### 8.4 Workflow 04：实现 PR 合并 → 归档 SDD → 关闭 Issue

**触发条件：** `copilot/*` 分支或带 `sdd:implementing` 标签的 PR 被合并

**执行逻辑：**

1. 从 PR body 中提取 change-id
2. 运行 `openspec archive <change-id> --yes`（CLI 失败时手动回退）
3. 合并 delta specs 到 `.openspec/specs/`
4. 提交归档（commit message 含 `[skip ci]` 防止循环）
5. 所有关联 Issue 标签 → `sdd:done`，自动关闭
6. 评论归档完成

```yaml
name: "04 📦 Impl Merged → Archive"

on:
  pull_request:
    types: [closed]
    branches: [main]

permissions:
  contents: write
  issues: write
```

---

## 9. 完整生命周期时序

```
👤 PM/Dev                  GitHub Actions                🤖 Copilot Agent
────────                   ──────────────                ────────────────

1. New Issue [SDD]
   label: sdd
   │── issues.opened ─────▶│
                            │ 01-sdd-init
                            ├ 幂等检查
                            ├ 生成 SDD 骨架
                            ├ 创建 sdd/ 分支
                            ├ 创建 Draft PR #2
                            ├ label → sdd:spec-ing
                            └ 评论 Issue #1
   ◀── 通知 ───────────────│

2. 完善 SDD (PR #2)
   │── push ───────────────▶│
                            │ 02-sdd-validate
                            ├ 文件检查
                            ├ openspec validate
                            └ 发布报告
   ◀── 报告 ───────────────│

3. Merge SDD PR #2
   │── merge ──────────────▶│
                            │ 03-sdd-implement
                            ├ 解析 tasks.md
                            ├ 创建 Issue #3
                            │  assignees: [copilot]
                            ├ label #1 → sdd:spec-ready
                            └ 评论 #1
                            │
                            │── Issue #3 ──────────────▶│
                                                        │ 分析仓库
                                                        │ 读取 SDD
                                                        │ 创建分支
                                                        │ 编写代码
                                                        │ 运行测试
                            │◀── Draft PR #4 ──────────│

4. Review & Merge PR #4
   │── merge ──────────────▶│
                            │ 04-sdd-archive
                            ├ openspec archive
                            ├ 合并 delta specs
                            ├ commit [skip ci]
                            ├ label → sdd:done
                            ├ 关闭 Issue #1 & #3
                            └ 评论: archived
   ◀── done ───────────────│
```

---

## 10. 关键设计决策与 Review 记录

### 10.1 为什么砍掉 Project Board？

| 问题 | 详情 |
|------|------|
| `projects_v2_item` 仅组织级 | webhook 仅在组织级别生效，repo 级别不收该事件 |
| Actions 触发不稳定 | 社区广泛报告延迟、丢失事件等问题 |
| 权限要求高 | 需要 Organization Admin + PAT (project scope) |
| 只读镜像无价值 | Board 仅同步展示已有 Issue 状态，不产生新决策 |

### 10.2 OpenSpec 命令纠错

| 错误 | 正确 |
|------|------|
| `npm install -g openspec` | `npm install -g @fission-ai/openspec@latest` |
| 目录 `openspec/` | 目录 `.openspec/`（CLI 默认） |
| `/opsx:` 在 CI 中使用 | `/opsx:` 仅在 AI IDE 中使用 |

### 10.3 Copilot Agent 触发方式

- 通过 REST API `assignees: ['copilot']` 触发
- Agent 自动创建 Draft PR，不在 Issue 中回复
- 迭代反馈发生在 PR 评论中
- 需要 Copilot Pro+/Business/Enterprise 订阅
- 仓库 Settings → Copilot → Coding Agent 需提前启用

### 10.4 幂等性与健壮性

- 每个 Workflow 都有幂等性检查（分支已存在则 skip）
- `openspec archive` CLI 失败时有手动回退方案
- 归档提交使用 `[skip ci]` 防止无限循环
- Label 操作 `removeLabel` 使用 `.catch(() => {})` 容错

---

## 11. 已知限制

| # | 限制 | 应对策略 |
|---|------|---------|
| 1 | `assignees: ['copilot']` 要求 Agent 已启用 | Workflow 中 `continue-on-error` + 回退为手动 |
| 2 | OpenSpec CLI 仍在快速迭代 | archive 步骤有手动回退方案 |
| 3 | `[skip ci]` 提交不触发后续 workflow | 这是期望行为，防止循环 |
| 4 | Copilot Agent 只适合中小型任务 | 大型任务手动实现，不打 auto-implement |
| 5 | branch protection 可能阻止 bot push | 需配置允许 `github-actions[bot]` |

---

## 12. 日常使用速查

| 你想做什么 | 操作 |
|-----------|------|
| 发起新需求 | New Issue → 选 "SDD Request" 模板 → 自动触发 |
| 查看 SDD 进行中 | `is:open label:sdd:spec-ing` |
| 查看等待实现 | `is:open label:sdd:spec-ready` |
| 查看实现中 | `is:open label:sdd:implementing` |
| 查看 Copilot 处理中 | `is:open label:copilot-assigned` |
| 查看已完成 | `label:sdd:done is:closed` |
| 本地编辑 SDD | `git checkout sdd/<change-id>` → 编辑 → push |
| IDE 中用 OPSX | `/opsx:new <id>` → `/opsx:ff` → `/opsx:apply` |

---

## 13. 总结

这个方案的核心是：

> **Issue 是入口，Label 是状态机，SDD 是规格标准，Actions 是自动化引擎，Copilot Agent 是可选执行者。**

它不依赖任何额外的 Token、Project Board 或组织级权限。Clone 仓库 → 跑初始化脚本 → 提 Issue → 自动运转。

这不是一个理论方案，而是一个可以直接 copy-paste 到任何 GitHub 仓库中立即使用的工程实现。

---

## 参考链接

- [OpenSpec - Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [OpenSpec CLI Reference](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)
- [OpenSpec Workflow Docs](https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md)
- [GitHub Docs: Assign Copilot to an Issue](https://docs.github.com/copilot/how-tos/use-copilot-agents/coding-agent/assign-copilot-to-an-issue)
- [GitHub Docs: Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- [GitHub Docs: Automating Projects using Actions](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/automating-projects-using-actions)