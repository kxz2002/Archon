# Archon 深度解析

> 本文档基于对项目源码的系统性阅读和分析，记录了关于 Archon 项目的核心理解。适合想要深入理解 Archon 架构和原理的开发者阅读。

## 目录

1. [项目概述](#1-项目概述)
2. [核心概念：Workflow、Command、Skill](#2-核心概念workflowcommandskill)
3. [Archon 与 Claude Code 的关系](#3-archon-与-claude-code-的关系)
4. [Workflow 执行机制](#4-workflow-执行机制)
5. [Workflow 的数据结构：DAG](#5-workflow-的数据结构dag)
6. [Archon 编排的抽象层次](#6-archon-编排的抽象层次)
7. [Archon 在 Claude Code 中如何生效](#7-archon-在-claude-code-中如何生效)

---

## 1. 项目概述

**Archon** 是一个 AI 编码工作流引擎——它让开发者把软件开发流程（规划 → 实现 → 测试 → 审查 → 提交 PR）定义成 YAML 文件，然后由 AI agent 按这个确定性流程执行，而不是让 AI 自由发挥。

### 解决什么问题

直接让 AI "帮我修这个 bug"，每次结果都不一样——它可能跳过规划、忘记跑测试、写出不规范的 PR 描述。Archon 用版本化的 YAML DAG 流程约束 AI 的行为边界，流程结构由开发者控制，AI 只负责填充分智能的部分。

**类比：Dockerfile 之于基础设施，GitHub Actions 之于 CI/CD，Archon 之于 AI 编码流程。**

### 技术栈

Bun + TypeScript + Hono (HTTP server) + React (Web UI) + SQLite/PostgreSQL，monorepo 结构。

### 核心能力

| 能力 | 说明 |
|------|------|
| DAG 工作流 | 7 种节点类型，支持依赖关系、条件执行、并发调度、失败重试、变量替换 |
| 多 AI 后端 | Claude Code SDK、OpenAI Codex SDK、Pi（社区贡献，支持 ~20 种 LLM） |
| Git Worktree 隔离 | 每次工作流运行自动创建独立分支和目录 |
| 多平台接入 | CLI、Web UI、Slack、Telegram、GitHub Issues/PRs、Discord |
| 可视化工作流构建器 | Web UI 中拖拽编辑 DAG 工作流 |

---

## 2. 核心概念：Workflow、Command、Skill

Archon 中有三个名字相近但完全不同的概念：

| 概念 | 位置 | 格式 | 作用 |
|------|------|------|------|
| **Workflow（工作流）** | `.archon/workflows/*.yaml` | YAML | Archon 执行的 DAG 自动化流程 |
| **Command（命令）** | `.archon/commands/*.md` | Markdown | AI prompt 模板，被 workflow 节点引用 |
| **Archon Skill（技能）** | `.claude/skills/archon/` | Markdown | 教 Claude Code 怎么用 Archon 的说明书 |

**Workflow 不是 Skill。** Skill 是一套文档，告诉 Claude Code "你可以用 `archon workflow run` 命令来做 X"。Workflow 是真正被 Archon 引擎执行的东西。

### 如何使用

**方式一：直接用内置工作流**

```bash
archon workflow run archon-fix-github-issue "修复 #123 的登录超时 bug"
```

在 GitHub Issue 里 `@archon fix this bug`，AI 会自动路由到合适的工作流。

**方式二：自定义工作流**

在项目的 `.archon/workflows/` 下创建 YAML 文件。三种加载源按优先级从低到高：Bundled（内置）< Global（`~/.archon/workflows/`）< Project（`<repo>/.archon/workflows/`）。

**方式三：自定义 Command**

在 `.archon/commands/` 下创建 `.md` 文件，工作流的 `command:` 节点直接引用。Command 就是可复用的 prompt 模板。

### 三种调用方式

1. **CLI 直接调用**：`archon workflow run <name> [message]`
2. **斜杠命令**（Slack/Telegram/Web UI）：`/workflow run <name> [args]`
3. **AI 自动路由**（自然语言）：`@archon fix this bug`，AI 根据路由规则自动选择工作流

---

## 3. Archon 与 Claude Code 的关系

### 一句话结论

**Archon 是 Claude Code 的上层编排平台，不是替代品。** 这是双向关系：

### 方向 A：Archon 依赖 Claude Code SDK（主关系）

`packages/providers/package.json` 声明了 SDK 依赖：
```json
"dependencies": {
    "@anthropic-ai/claude-agent-sdk": "^0.2.121",
}
```

`ClaudeProvider.sendQuery()` 调用 SDK 的 `query()` 函数，SDK 内部 **spawn Claude Code 原生二进制作为子进程**。Archon 不直接调用 REST API，也不自己实现 agent loop——完全委托给 SDK。

### 方向 B：Claude Code 可以调用 Archon（辅助关系）

`archon skill install` 把 Archon Skill 安装到 `.claude/skills/archon/`。Claude Code 启动时自动加载该 Skill，从而学会通过 `archon workflow run` 命令委托任务给 Archon。

### 架构定位

```
┌──────────────────────────────────────────────┐
│                  Archon                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Workflow │ │ Platform │ │   Provider    │  │
│  │  Engine  │ │ Adapters │ │   Registry    │  │
│  │  (DAG)   │ │          │ │               │  │
│  └──────────┘ └──────────┘ └──────┬────────┘  │
│                                   │            │
│                          SDK query() 调用      │
└───────────────────────────────────┼────────────┘
                                    ▼
                    ┌───────────────────────────┐
                    │   Claude Code 二进制       │
                    │   (子进程，由 SDK 管理)     │
                    │   - Agent loop             │
                    │   - Tool use               │
                    │   - 文件读写               │
                    └───────────────────────────┘
```

**Archon 不是 Claude Code 的替代品，而是它的编排层。** 也可以通过 Codex 或 Pi 等其他 provider 运行，不必须依赖 Claude Code。

---

## 4. Workflow 执行机制

### 完整执行链路

```
用户消息 "fix this bug"
       │
       ▼
orchestrator-agent.ts: handleMessage()
       │
       ├─ 斜杠命令 → command-handler.ts → executeWorkflow()
       │
       └─ 自然语言 → AI 路由 → /invoke-workflow → dispatchOrchestratorWorkflow()
                                                         │
                                                         ▼
                                              executor.ts: executeWorkflow()
                                              - 加载配置，解析 provider
                                              - 路径锁守卫
                                              - 创建 WorkflowRun 数据库记录
                                                         │
                                                         ▼
                                              dag-executor.ts: executeDagWorkflow()
                                              - buildTopologicalLayers() 拓扑分层
                                              - 逐层 Promise.allSettled() 并发执行
                                                         │
                                              ┌──────────┼──────────┐
                                              ▼          ▼          ▼
                                          prompt节点  bash节点   approval节点
```

### 各节点类型的执行方式

**AI 节点（command/prompt）**：通过 `ClaudeProvider.sendQuery()` → SDK `query()` → spawn Claude Code 子进程。不是 REST API 调用。

**Bash 节点**：`execFileAsync('bash', ['-c', script])`，无 AI 参与。stdout 被捕获为 `$nodeId.output`，shell 变量用单引号转义防止注入。

**Script 节点**：`execFileAsync('bun', ['-e', code])` 或 `execFileAsync('uv', ['run', 'python', '-c', code])`，无 AI 参与。

**Loop 节点**：循环调用 `aiClient.sendQuery()`，每次迭代后检查完成信号（`<promise>COMPLETE</promise>` 或 `until_bash` 脚本 exit 0）。

**Approval 节点**：暂停工作流，等待用户通过 `/workflow approve/reject` 响应。

### Agent 工程规范

**Archon 不自己实现 agent loop。** Agent loop（think → tool use → observe → repeat）完全由 SDK 内部管理：

- **Claude SDK** 的 `query()` 函数 spawn 原生 `claude` 二进制子进程，该子进程内置完整的 agent loop
- **Codex SDK** 同理，`thread.runStreamed()` 处理 Codex 的内部 agent loop
- **Archon 唯一管理的循环层**是 `LoopNode`——它迭代整个 AI 调用，但每次迭代内部仍然是 SDK 管理自己的循环

### 会话管理

- `context: fresh`：强制新 AI 会话（`resumeSessionId = undefined`）
- `context: shared`（默认）：传递 `lastSequentialSessionId`，下一个节点恢复同一会话
- 并行层中的节点始终 `fresh`（不能共享会话）

---

## 5. Workflow 的数据结构：DAG

### 是 DAG，不是线性序列

旧版 `steps:` 格式已被移除，所有工作流使用 `nodes:`（DAG）格式。

**Schema 定义**（`packages/workflows/src/schemas/dag-node.ts`）：
```typescript
export const dagNodeBaseSchema = z.object({
  id: z.string(),
  depends_on: z.array(z.string()).optional(),  // 显式边列表
  when: z.string().optional(),
  trigger_rule: triggerRuleSchema.optional(),
  // ... 7 种互斥的节点类型字段
});
```

### 7 种互斥节点类型

| 类型 | 判别字段 | 说明 |
|------|---------|------|
| CommandNode | `command: string` | 引用 `.archon/commands/` 中的命令文件 |
| PromptNode | `prompt: string` | 内联 prompt，直接发给 AI |
| BashNode | `bash: string` | Shell 脚本，无 AI |
| ScriptNode | `script: string` + `runtime` | TS/Python 脚本，无 AI |
| LoopNode | `loop: {...}` | 循环执行直到满足条件 |
| ApprovalNode | `approval: {...}` | 暂停等待人工审批 |
| CancelNode | `cancel: string` | 终止工作流 |

### 拓扑排序：Kahn 算法

`dag-executor.ts:491-539` 的 `buildTopologicalLayers()`：

```typescript
export function buildTopologicalLayers(nodes): DagNode[][] {
  // Phase 1: 构建入度表和邻接表
  // Phase 2: 入度为 0 的节点为第 0 层（根节点）
  // Phase 3: 逐层剥离，同层节点并发执行
  // Phase 4: visited < totalNodes → 环检测报错
}
```

**输出是 `DagNode[][]`（分层数组）。** 同一层内节点无相互依赖，通过 `Promise.allSettled(layer.map(...))` 并发执行。层间串行。

### 可视化示例

```yaml
nodes:
  - id: A              # 入度 0 → Layer 0
  - id: B              # 入度 0 → Layer 0（与 A 并发）
  - id: C
    depends_on: [A, B] # 入度 2 → Layer 1
  - id: D
    depends_on: [A]    # 入度 1 → Layer 1（与 C 并发）
  - id: E
    depends_on: [C, D] # 入度 2 → Layer 2
```

```
Layer 0:   [A]  ⚡ [B]     ← 并发
              \  /   \
Layer 1:      [C]   [D]    ← 并发
                \   /
Layer 2:        [E]
```

### 节点的决策流程

每个节点执行前经过以下检查：

```
1. priorCompletedNodes 检查（resume 场景）→ 已完成? → 跳过
2. trigger_rule 检查 → skip? → 跳过
3. when: 条件求值 → false? → 跳过
4. 按节点类型调度执行
```

**trigger_rule 的四种模式：**

| 规则 | 含义 |
|------|------|
| `all_success`（默认）| 所有上游成功才执行 |
| `one_success` | 至少一个上游成功就执行 |
| `none_failed_min_one_success` | 无失败且至少一个成功 |
| `all_done` | 所有上游终态（含失败/跳过）就执行 |

### 环检测

在加载时（`loader.ts`）和运行时（`dag-executor.ts`）各做一次 Kahn 算法环检测。

### Resume（断点续传）

`hydrateResumableRun()` 从数据库事件表查询已完成节点的输出，预填充到 `nodeOutputs` Map。执行时每个节点检查 `priorCompletedNodes.has(node.id)`，若已存在则跳过。

---

## 6. Archon 编排的抽象层次

### 核心洞察：分层抽象

Archon 编排的不是单一事物，而是一个**多层抽象系统**。核心抽象是 **DAG 节点**——一个统一的处理步骤单元，抽象掉了 AI 调用、Shell 执行、脚本运行和人工审批之间的差异。

### 完整的抽象层次

| 层级 | 被编排的抽象 | 核心接口 |
|------|------------|---------|
| **消息层** | 跨平台的对话和命令 | `IPlatformAdapter` |
| **路由层** | AI 路由 vs 确定性命令的分发 | `handleMessage()` |
| **会话层** | AI provider 会话的生命周期和审计链 | `Session` + `transitionSession()` |
| **工作流层** | DAG 节点的拓扑调度、并发、重试、条件 | `executeDagWorkflow()` |
| **节点层** | 统一的任务单元（AI/bash/script/审批） | `NodeOutput` |
| **引擎层** | 不同 AI 后端的统一调用 | `IAgentProvider.sendQuery()` |
| **隔离层** | Git worktree 的创建/复用/销毁 | `IIsolationProvider` |
| **持久层** | 工作流状态和事件的完整审计追踪 | `IWorkflowStore` |

### Node：最核心的抽象

每个节点本质上是一个**函数**：

```
Node: (upstream_outputs, workflow_variables, config) → NodeOutput
```

`NodeOutput` 是 discriminated union：
```typescript
// 成功时:
{ state: 'completed', output: string, sessionId?: string, structuredOutput?: unknown }
// 失败时:
{ state: 'failed', output: string, error: string }
// 跳过时:
{ state: 'skipped', output: string }
```

**关键洞察：Node 把 AI 调用和非 AI 操作抽象成了同一种东西。** 下游节点不关心 `$upstream.output` 是来自 Claude Code 的文本输出还是 bash 脚本的 stdout。

### Orchestrator：一切的调度中心

`orchestrator-agent.ts:643` 的 `handleMessage()` 是整个系统的单一入口点：

```typescript
async function handleMessage(
  platform: IPlatformAdapter,      // ← 从哪来
  conversationId: string,           // ← 哪个会话
  message: string,                  // ← 用户说了什么
  context?: HandleMessageContext    // ← 附加上下文
): Promise<void>
```

它的决策树：斜杠命令 → 确定性处理（无 AI）；自然语言 → AI 路由 → 聊天回复或触发工作流。

### 依赖注入

每一层通过窄接口解耦。`WorkflowDeps` 接口只有 3 个字段：
```typescript
export interface WorkflowDeps {
  store: IWorkflowStore;
  getAgentProvider: AgentProviderFactory;
  loadConfig: (cwd: string) => Promise<WorkflowConfig>;
}
```

### 一句话总结

**Archon 不是"AI 工具"，而是"AI 工作流的操作系统"——管理进程（Node）、调度资源（Provider）、提供隔离（Worktree）、持久化状态（Store），通过统一的消息总线（Platform Adapter）对外暴露能力。**

---

## 7. Archon 在 Claude Code 中如何生效

### 机制：Claude Code 原生 Skill

**Archon 不会拦截 Claude Code 的斜杠命令。** 它的机制很简单：安装到 `.claude/skills/archon/SKILL.md` 的 Markdown 文件教 Claude Code 的 AI "遇到某些任务时，用 Bash 工具去调用 `archon` CLI"。

### SKILL.md 的核心

```yaml
description: |
  Use when: User wants to run Archon workflows, CREATE workflows or commands...
  Triggers (run): "use archon to", "run archon", "archon workflow", ...
  NOT for: Direct Claude Code work - only for delegating to Archon CLI.
```

**触发条件是自然语言短语，不是系统级 hook。** Claude Code 的 Skill 系统通过匹配用户输入来决定是否注入 Skill 内容到 AI 上下文。

### 实际运行流程

```
用户: "use archon to fix issue #42"
            │
            ▼
┌─ Claude Code Skill 匹配 ─────────────────────┐
│ 检测到 "use archon to" 匹配 skill trigger      │
│ 将 SKILL.md 全文注入 AI 上下文                  │
└──────────────────────────────────────────────┘
            │
            ▼
┌─ AI 遵循 SKILL.md 的指令 ────────────────────┐
│ 1. 运行 archon workflow list 查看可用工作流     │
│ 2. 匹配 "fix issue #X" → archon-fix-github-issue│
│ 3. 运行: archon workflow run \                │
│    archon-fix-github-issue \                  │
│    --branch fix/issue-42 \                    │
│    "Fix issue #42"                            │
│ 4. 使用 run_in_background: true                │
└──────────────────────────────────────────────┘
            │
            ▼
┌─ archon CLI 在独立 git worktree 中执行 ───────┐
│ - 创建隔离分支 fix/issue-42                     │
│ - 运行 DAG 工作流节点                            │
│ - Claude Code SDK spawn 子进程执行 AI 任务       │
└──────────────────────────────────────────────┘
```

### 两个方向的 Claude Code 使用

| 方向 | 说明 |
|------|------|
| **A: Archon → Claude Code** | 工作流执行时，ClaudeProvider 通过 SDK spawn 子进程 |
| **B: Claude Code → Archon** | 用户在 Claude Code 中说 "use archon to..."，AI 通过 Bash 运行 `archon` CLI |

**方向 B 中，Claude Code 的 AI 像一个"项目经理"——理解意图、选择工作流、下发命令、汇报结果。真正写代码的工作由 Archon 在隔离环境中完成。**

### 如果不用 Skill

也可以绕过 Skill，直接在 Claude Code 终端里手动运行：
```bash
archon workflow run archon-fix-github-issue --branch fix/issue-42 "Fix issue #42"
```

Skill 只是让这个过程自动化——让 AI 自己决定何时该调用 Archon、选哪个工作流、传什么参数。

---

## 关键源码文件索引

| 文件 | 内容 |
|------|------|
| `packages/providers/src/types.ts` | IAgentProvider 接口定义 |
| `packages/providers/src/claude/provider.ts` | Claude Provider 实现，SDK 子进程管理 |
| `packages/workflows/src/schemas/dag-node.ts` | 7 种节点类型 Zod schema |
| `packages/workflows/src/schemas/workflow.ts` | WorkflowDefinition schema |
| `packages/workflows/src/schemas/workflow-run.ts` | NodeOutput、WorkflowRun 运行时类型 |
| `packages/workflows/src/loader.ts` | YAML 解析 + DAG 验证 + 环检测 |
| `packages/workflows/src/dag-executor.ts` | DAG 执行引擎（拓扑分层、节点调度、重试） |
| `packages/workflows/src/executor.ts` | 工作流入口（配置、路径锁、DAG 调度） |
| `packages/workflows/src/executor-shared.ts` | 变量替换、命令加载、错误分类 |
| `packages/workflows/src/condition-evaluator.ts` | `when:` 条件表达式求值 |
| `packages/workflows/src/deps.ts` | WorkflowDeps 依赖注入接口 |
| `packages/workflows/src/store.ts` | IWorkflowStore 持久化接口 |
| `packages/core/src/orchestrator/orchestrator-agent.ts` | handleMessage() 单一入口点 |
| `packages/core/src/orchestrator/orchestrator.ts` | 后台工作流分发、隔离解析 |
| `packages/core/src/types/index.ts` | IPlatformAdapter、Conversation、Session 定义 |
| `packages/isolation/src/types.ts` | IIsolationProvider 接口 |
| `packages/cli/src/commands/skill.ts` | `archon skill install` 实现 |
| `.claude/skills/archon/SKILL.md` | Archon Skill 入口（教 Claude Code 用 Archon） |
