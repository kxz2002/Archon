# Workflow Distillation：从 CC 执行记录沉淀 Archon 工作流

> 实习答辩项目设计文档（进行中）。本文档记录头脑风暴讨论结果、调研结论和待决细节。

## 1. 项目目标

利用 Archon 已有的基础设施，开发一个 **Claude Code Skill**，让 CC 能够**从自身执行过的复杂流程中沉淀出 Archon 工作流（YAML）**。

### 与已有方案的差异

| 维度 | `archon-workflow-builder.yaml`（已存在） | 本方案 |
|------|---------------------------|---------|
| 输入模态 | 自然语言描述 | **CC 执行记录（context / 历史 trace）** |
| 触发方式 | `archon workflow run archon-workflow-builder` | **CC 中调用 Skill** |
| 用户负担 | 用户必须先想清楚流程再描述 | 用户只需做完任务即可 |
| 准确性 | 取决于描述完整性 | **由真实执行轨迹决定** |
| 适用场景 | 已知流程显式化 | **隐式流程结晶化** |

理论概念：trace-to-workflow synthesis（执行轨迹到工作流的合成），比 NL-to-workflow 更前沿。

## 2. 核心价值主张

**CC 是探索引擎（灵活但不确定），Archon 是固化引擎（确定且可复用）。** 两者协同：

```
[复杂任务]
   ↓
CC 灵活探索（不可重复、不可保证）
   ↓
Skill 沉淀
   ↓
.archon/workflows/my-task.yaml（确定性、版本化、可复用）
   ↓
archon workflow run my-task（任何人、任何时间，结果一致）
```

### 为什么 Skill 不能替代 Archon

| 能力 | Skill | Archon Workflow |
|------|-------|-----------------|
| 可重复性 | 每次 AI 自由发挥，结果可能不同 | YAML 定义死，bash/script 节点是真实进程 |
| 团队复用 | 每人各自调 prompt | 一个 YAML，全员共用 |
| 审批门 | AI 无法被"硬停" | approval 节点字面阻塞进程 |
| 并行执行 | 串行决策 | DAG 同层节点真正并发 |
| 断点续传 | 重来或 AI 自行判断 | 引擎级 checkpoint |

**叙事**：今天用 CC 调试复杂问题花了 10 轮 tool call。下次遇到类似问题，让 AI 重新摸索一遍既慢又不确定。Skill 把刚才的流程沉淀成确定性的工作流，下次 `archon workflow run` 就能复用同样的过程。

## 3. 关键发现：仓库已有的基础设施

### 3.1 archon-workflow-builder.yaml（参考价值高）

**位置**：`.archon/workflows/defaults/archon-workflow-builder.yaml`，249 行

**核心模式**：两阶段生成
1. **Phase 1（intent extraction）**：用 `output_format` 强制 AI 输出结构化 JSON 描述意图
2. **Phase 2（YAML synthesis）**：基于结构化意图生成 YAML

**技巧**：
- 用 `denied_tools` 限制 AI 行为边界
- 用 `$ARTIFACTS_DIR` 输出工件
- 用 `$<nodeId>.output` 把上游节点输出注入下游

**借鉴价值**：本方案直接复用这个两阶段模式，仅替换 Phase 1 的输入（从用户描述 → CC 执行记录）。

### 3.2 校验端点

**`POST /api/workflows/validate`**
- 入参：`{ definition: Record<string, unknown> }`（YAML 对象，非字符串）
- 出参：`{ valid: boolean, errors?: string[] }`
- 实现：序列化 → `parseWorkflow()` → Zod + 语义校验

这是 AI 自我修正闭环的天然入口：生成 YAML → POST 校验 → 若失败读取错误 → 重新生成。

### 3.3 持久化端点

**`PUT /api/workflows/:name`**
- 入参：`{ definition: Record<string, unknown> }`
- 行为：校验通过后写盘到 `.archon/workflows/` 或 `~/.archon/workflows/`
- 来源：可选 `?source=project` 或 `?source=global`

### 3.4 用户定义工作流的所有渠道

| 渠道 | 入口 | 体验 |
|------|------|------|
| 1. 手写 YAML | `.archon/workflows/*.yaml` | 完全手动 |
| 2. Web UI 可视化构建器 | `WorkflowBuilderPage` → `@xyflow/react` | 拖拽 DAG，仅支持 3/7 节点类型 |
| 3. REST API | `PUT /api/workflows/:name` | JSON 提交 |
| 4. CLI marketplace | `archon workflow install <slug>` | 下载现成工作流 |

**Skill 是第 5 个渠道**——从 CC 上下文沉淀。

## 4. Schema 复杂度评估

### Node 字段统计
- **共享字段**：23 个（id, depends_on, when, trigger_rule, model, provider, context, output_format, allowed_tools, denied_tools, retry, hooks, mcp, skills, agents, …）
- **类型字段**：7 种互斥（command/prompt/bash/script/loop/approval/cancel）
- **类型特有**：script 必须有 `runtime`，interactive loop 必须有 `gate_message`，loop 不允许 `retry`

### 复杂度评估
- Schema 复杂度：6/10
- 校验反馈质量：7/10（有字段路径，但只返回第一个错误且是扁平字符串）
- AI 一次性生成正确率：约 30-50%（凭印象，需实验确认）
- 经过 2-3 轮校验闭环后正确率：约 80-90%

### 三大风险
1. **节点互斥违反**：AI 可能同时写 `prompt:` 和 `bash:`
2. **跨节点引用拼写**：`$nodeId.output` 中 nodeId 拼错
3. **嵌套对象扁平化**：把 `loop.gate_message` 写成 `loop_gate_message`

### 三大缓解策略
1. **Few-shot 提示**：用 `.archon/workflows/defaults/` 下 20 个真实工作流作为示例
2. **迭代校验闭环**：POST /api/workflows/validate → 读错误 → 修正 → 重试
3. **两阶段生成**：先输出结构化 intent，再生成 YAML（参考 archon-workflow-builder.yaml）

## 5. 两条实现路径

### 路径 A：In-Context Distillation（推荐，3 天可行）

在**当前 CC 会话**中触发 Skill，让 CC 回顾刚做过的事。CC 已经持有完整上下文（tool calls、决策、文件操作），无需读外部日志。

**用户体验**：
```
用户: [在 CC 中完成一个复杂调试任务，经历了 10+ 轮工具调用]
用户: /archon-distill 把刚才的流程沉淀成工作流
CC: [分析当前上下文 → 生成 YAML → 调用 validate → 修正 → 保存]
```

**优点**：
- 上下文齐全，无需解析外部日志格式
- 当场可演示，叙事清晰
- 实现简单：Skill 只需调用 Archon API

**缺点**：
- 只能沉淀当前会话的内容
- 受 CC context window 限制

### 路径 B：Log-Reading Distillation

读取 `~/.claude/projects/<project>/` 下的历史对话日志（JSONL 格式），跨会话挖掘流程。

**优点**：
- 可挖掘任意历史会话
- 价值更高，可批量沉淀

**缺点**：
- 需研究 CC JSONL schema（未公开文档）
- 需要 session 选择 UI/CLI
- 3 天偏紧

**结论**：选 **路径 A**。如时间允许，可在演示叙事中提到"未来可扩展到历史日志"。

## 6. 待决细节

进入实现前需用户确认：

### 6.1 演示场景

选择什么样的"前置任务"让 CC 先做、再沉淀？

**条件**：
- 在 2-3 分钟内可演示
- 天然有 DAG 结构（不是线性的）
- 有 bash/script 节点（体现"真实进程"优势）
- 不依赖外部服务（演示稳定性）

**候选**：
- 候选 1：扫描代码 → 跑测试 → 生成报告（线性，差）
- 候选 2：并行跑 lint+typecheck+test → AI 综合 → 生成报告（有并行）
- 候选 3：克隆某仓库 → 并行分析多个模块 → 生成依赖图 + AI 概览（DAG 丰富）
- 候选 4：[待用户提议]

### 6.2 沉淀产物的落盘方式

- 选项 A：直接写文件到 `.archon/workflows/`
- 选项 B：调用 `POST /api/workflows/validate` 校验 → 通过后 `PUT /api/workflows/:name` 保存

**推荐 B**：复用 Archon 自己的校验逻辑，避免重复实现 Schema。但需要 archon server 在跑（`bun run dev:server` 或 `archon serve`）。

### 6.3 校验失败的处理

CC 生成的 YAML 校验不通过时：

- 选项 A：Skill 自动重试，最多 N 次，最终失败则报错
- 选项 B：显示错误给用户，由用户决定下一步
- 选项 C：交互式修正——每次报错都问用户是否继续

**倾向 A，N=3**：自动化体验更好，演示更流畅。失败案例可在演示中故意展示一次（"看，校验机制让 AI 自我修正"）。

## 7. 后续工作

1. 用户决定 6.1/6.2/6.3 三个细节
2. 进入 `writing-plans` 阶段，产出详细实现计划
3. 开始实现：
   - Skill 入口（`.claude/skills/archon-distill/SKILL.md`）
   - 沉淀逻辑（Skill 内嵌的 prompt 模板）
   - 校验+保存逻辑（调用 Archon API）
4. 准备演示素材：录屏、PPT、demo 仓库

## 8. 时间估算（3 天）

| 阶段 | 估时 |
|------|------|
| Skill 设计 + 第一版 SKILL.md | 0.3 天 |
| 跑通最小闭环（CC → YAML → 校验 → 保存） | 0.5 天 |
| 调优 prompt（few-shot 示例、两阶段、错误反馈） | 0.5 天 |
| 准备演示场景 + 反复跑通 | 0.4 天 |
| 录屏 + PPT | 0.8 天 |
| 缓冲 + 兜底 | 0.5 天 |
| **合计** | **3 天**（无富裕） |

风险点：prompt 调优可能反复；演示场景选择不当导致重做。

## 9. 关键源码索引（待实现时参考）

| 文件 | 内容 |
|------|------|
| `.archon/workflows/defaults/archon-workflow-builder.yaml` | 现有元工作流，可直接借鉴模式 |
| `packages/workflows/src/schemas/dag-node.ts` | 节点 Schema 定义（30+ 字段） |
| `packages/workflows/src/loader.ts` | YAML 解析 + Zod + 语义校验 |
| `packages/workflows/src/validator.ts` | 命令/script 存在性校验 |
| `packages/server/src/routes/api.ts` | `POST /api/workflows/validate`、`PUT /api/workflows/:name` |
| `packages/workflows/src/schemas/workflow.ts` | WorkflowDefinition 顶层 Schema |
