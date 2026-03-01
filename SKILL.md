---
name: agent-teams-playbook
version: "4.6"
maintainer: rebecca554owen
description: Agent Teams编排手册，用于多agent协作、团队组建、并行任务处理。当用户提到"agent teams"、"多agent协作"、"拉团队"、"并行处理"、"分工协作"、"swarm编排"、"agent团队"、"多代理"、"agent task"等关键词时触发。特别适用于跨文件重构、大规模代码生成、需要多角色协作的复杂任务。即使没有明确说"组建团队"，只要任务复杂度明显需要多agent并行处理，也应该使用此技能。
---
# Agent Teams 编排手册
作为 Agent Teams 协调器，你的职责包括：明确每个角色的职责边界、把控执行过程、对最终产品质量负责。
> **核心理解（铁律）**：Agent Teams 是"并行处理 + 结果汇总"模式，不是扩大单个 agent 的上下文窗口。每个 teammate 是独立的 Claude Code 实例，拥有独立的上下文窗口，可以并行处理大量信息，但最终需要将结果汇总压缩后返回主会话。
## 适用 vs 不适用
| 适用 | 不适用 |
|------|--------|
| 跨文件重构、多维度审查 | 单文件小修改 |
| 大规模代码生成、并行处理 | 简单问答、线性顺序任务 |
| 需要多角色协作的复杂任务 | 单agent可完成的任务 |
**边界处理**：用户输入模糊时，先引导明确任务再决策；任务太简单时，主动建议使用单agent而非组建团队。
## 用户可见性铁律
1. 每个阶段启动前输出计划，完成后输出结果
2. 子agent在后台执行，但进度必须汇报给用户
3. 任务拆分计划必须经用户确认后再执行
4. 失败时立即通知：`❌ [角色名] 失败: [原因]`，提供重试/跳过/终止选项
5. 全部完成后输出汇总报告（见阶段5格式）
## 场景决策树（含协作模式选择）
**执行顺序**：
1. **强制执行阶段0和阶段1**（所有场景必经）
2. **根据阶段1结果选择场景**（使用以下决策树）：

| 问题 | 路径 |
|------|------|
| Q0: 阶段1找到完全匹配的单个Skill？ | 是 → 场景2（Skill直接复用）<br>否 → Q1 |
| Q1: 任务复杂度？ | 简单(1-2步) → 场景1（提示增强）<br>中等(3-5步) → Q2<br>复杂(6+步) → Q3 |
| Q2: 任务间有依赖？ | 是 → Agent Team 模式<br>否 → Subagent 模式 |
| Q3: 需要明确团队分工？ | 是 → 场景4（Lead-Member）<br>否 → 场景5（复合编排） |

**使用规则**：
- 用户直接指定场景编号时，跳过决策树直接执行
- 未指定场景时，默认用**场景3（计划+评审）**
- **协作模式选择**：根据任务依赖关系自动选择 Subagent 或 Agent Team
- **注意**：阶段0（planning-with-files）和阶段1（Skill搜索，包含 find-skills）是所有场景的强制前置步骤

## 5大编排场景
| # | 场景 | 适用条件 | 默认协作模式 | 核心策略 |
|---|------|---------|------------|---------|
| 1 | 提示增强 | 简单任务，1-2步 | 不组队 | 优化单agent提示词，不拆分不组队 |
| 2 | Skill直接复用 | 任务可由单个Skill完全解决 | 不组队 | 执行规划和Skill搜索后，直接调用匹配的Skill，无需组建Agent Teams |
| 3 | 计划+评审 | 中等/复杂任务（**默认**） | **Subagent** | 出计划 → 用户确认 → 并行执行 → Review验收 |
| 4 | Lead-Member | 需要明确团队分工 + 任务间有依赖 | **Agent Team** | TeamCreate → Leader协调分配 → Member并行执行 → SendMessage协调 |
| 5 | 复合编排 | 复杂任务，无固定模式 | 动态切换 | 动态组合上述场景，按阶段切换协作模式 |
**模型分工**（所有场景通用）：通过Agent工具的`subagent_type`参数选择合适的agent类型——`Plan`处理复杂规划任务，`Explore`处理代码搜索，`general-purpose`处理常规任务。
## 协作模式选择（关键决策点）
**【铁律】**：阶段1任务分析后，必须根据任务依赖关系选择正确的协作模式。
| 模式 | 通信方式 | 适用场景 | 启动方式 | 成本 |
|------|---------|---------|---------|------|
| **Subagent** | 子agent → 主协调器单向汇报 | 并行独立任务，任务间无依赖 | `Agent`工具 | 低 |
| **Agent Team** | 成员间可双向通信(SendMessage) | 需要协作的复杂任务，任务间有依赖 | `TeamCreate` + `Agent(team_name, name)` | 高 |

### 协作模式判断流程
```
任务拆分后，检查任务间依赖关系：
┌─ 所有任务独立无依赖？
│  ├─ 是 → Subagent 模式（高效）
│  └─ 否 → Agent Team 模式（可协调）
│
├─ Subagent 模式实现：
│  1. 使用 TaskList 创建任务清单
│  2. 用 Agent() 工具并行启动 subagent
│  3. 通过 TaskUpdate 分配任务（owner字段）
│  4. 等待结果汇总
│
└─ Agent Team 模式实现：
   1. 使用 TeamCreate 创建团队
   2. 用 Agent(team_name, name) 生成团队成员
   3. 通过 SendMessage 实现成员间通信
   4. 通过 TaskList + TaskUpdate 协调任务
```

### 两种模式的代码示例
**Subagent 模式**（独立任务）：
```
# 步骤1: 创建任务清单
TaskCreate(subject="任务A", description="...")
TaskCreate(subject="任务B", description="...")

# 步骤2: 并行启动 subagent（使用 Agent 工具）
Agent(description="处理任务A", prompt="...", subagent_type="general-purpose")
Agent(description="处理任务B", prompt="...", subagent_type="general-purpose")

# 步骤3: 等待结果汇总
# subagent 完成后自动汇报
```

**Agent Team 模式**（协作任务）：
```
# 步骤1: 创建团队
TeamCreate(team_name="my-project", agent_type="general-purpose")

# 步骤2: 生成团队成员（使用 Agent 工具，指定 team_name 和 name）
Agent(team_name="my-project", name="frontend", description="前端开发", prompt="前端开发，等待后端API...")
Agent(team_name="my-project", name="backend", description="后端开发", prompt="后端开发，完成后通知frontend")

# 步骤3: 成员间通信协调
SendMessage(type="message", recipient="frontend", content="API已就绪", summary="API就绪通知")

# 步骤4: 通过 TaskList 协调任务
TaskList()  # 查看团队任务状态
```
## 6阶段工作流（含强制规划和Skill搜索）
**重要说明**：阶段0和阶段1是**所有场景的强制前置步骤**，场景选择（1-5）只影响阶段2-5的执行方式。
### 阶段0：规划准备（Planning Setup）**【硬性标准 - 所有场景必经】**
**使用 Skill 工具调用 planning-with-files**：
```
Skill(skill="planning-with-files", args="")
```
或
```
Skill(skill="planning-with-files")
```
> **注意**：`planning-with-files` 不需要 args 参数，args 可以留空或不传。

这将在项目目录创建三个核心文件：
- `task_plan.md` - 任务计划和阶段追踪
- `findings.md` - 研究发现和知识积累
- `progress.md` - 执行日志和进度记录

**Skill 工具参数说明**：
- **skill**: Skill 名称（必需）
- **args**: 传递给 Skill 的参数（可选，不需要时留空）

**关键规则**（规划文件创建后遵循）：
- 每个阶段开始前读取task_plan.md，完成后更新状态
- 每2次搜索/浏览操作后立即保存发现到findings.md
- 所有错误必须记录到task_plan.md的"Errors Encountered"表格
- 3次失败后升级给用户
> **铁律**：没有task_plan.md就不能开始执行。这是Manus工作流的核心，确保上下文持久化。
### 阶段1：任务分析 + Skill发现（Discovery）**【硬性标准 - 所有场景必经】**
先质疑再执行：
- 需求不合理时主动挑战假设，建议更好的方案
- 区分"现在必须做"和"以后再说"，排除非核心范围
- 任务太大时建议更聪明的起点
输出任务总览：
| 字段 | 内容 |
|------|------|
| 任务目标 | [一句话描述] |
| 预期结果 | [具体交付物] |
| 验收标准 | [可量化的通过条件] |
| 范围界定 | [must-have vs add-later] |
| 预计Agent数 | [N个，建议≤5] |
| 选定场景 | [场景编号+名称] |
| 协作模式 | [Subagent/Agent Team] |
**信息获取完整回退链**（强制执行，不可跳过）：
对每个子任务执行以下4步fallback chain：
1. **本地Skill扫描**：
- 读取system-reminder中的"available skills"列表
- 提取每个skill的名称和触发词/描述
- 将子任务关键词与skill触发词比对
- 匹配成功 → 标注`[Skill: skill-name]`，进入阶段2直接调用
2. **MCP联网搜索**（本地无匹配时）：
> **MCP（Model Context Protocol）**是与外部系统交互的标准化接口，允许Claude搜索GitHub代码、查询技术文档等
- 使用本地MCP工具进行搜索：
  - `mcp__grep__searchGitHub` - GitHub代码搜索（搜索真实代码示例）
  - `mcp__plugin_context7_context7__resolve-library-id` + `query-docs` - 技术文档查询
- 搜索到相关解决方案 → 标注`[MCP: tool-name]`，进入阶段2调用
- 无结果 → 继续第3步
3. **外部Skill搜索**（MCP也无结果时）：
- 使用 Skill 工具调用 find-skills（需要传递搜索关键词）：
```
Skill(skill="find-skills", args="子任务关键词")
```
- 搜索到 → 向用户推荐：`npx skills add -g -y`
- 用户确认安装 → 标注新skill，进入阶段2调用
- 用户拒绝 → 继续第4步
4. **通用Subagent回退**（外部也无匹配时）：
- 该角色改用`Agent`工具生成通用subagent
- 在团队蓝图中标注`[Type: general-purpose]`
> **铁律**：这4步必须按顺序执行。优先使用MCP联网搜索（成本低、响应快），Skill搜索作为补充。
### 阶段2：团队组建（根据协作模式执行）

**首先判断协作模式**：
- 分析任务间是否存在依赖关系
- 独立任务 → Subagent 模式
- 协作任务 → Agent Team 模式

#### 模式A：Subagent 组建（独立任务）
输出团队蓝图（无需 TeamCreate）：
| 编号 | 角色 | 职责 | 模型 | subagent_type | 解决方案来源 |
|------|------|------|------|---------------|--------------|
| 1 | [角色名] | [具体职责] | [opus/sonnet/haiku] | [agent类型] | [Skill: name] 或 [MCP: tool] 或 [Type: general-purpose] |

**执行步骤**：
1. 使用 `TaskList` 创建任务清单
2. 准备并行启动 subagent（阶段3执行）

#### 模式B：Agent Team 组建（协作任务）
输出团队蓝图：
| 编号 | 角色 | 职责 | 模型 | 通信需求 | 解决方案来源 |
|------|------|------|------|---------|--------------|
| 1 | [角色名] | [具体职责] | [opus/sonnet/haiku] | [需通信的角色] | [Skill: name] 或 [MCP: tool] 或 [Type: general-purpose] |

**执行步骤**：
1. 调用 `TeamCreate(team_name="xxx", agent_type="general-purpose")`
2. 准备生成团队成员（阶段3执行）

> **说明**：最后一列"解决方案来源"标注该角色使用的Skill名称、MCP工具名称（阶段1已匹配）或通用类型（fallback）。
### 阶段3：并行执行（根据协作模式执行）

#### 模式A：Subagent 并行执行
- **Skill任务**：用`Skill`工具调用 → `Skill(skill="skill-name", args="任务描述")`
- **MCP工具任务**：直接调用MCP工具获取信息
- **通用任务**：用`Agent`工具并行启动，独立任务同时启动，有依赖的按序执行
- 混合编排时skill、MCP工具和subagent可并行运行
- 每个agent/skill完成后汇报：`✅ [角色名] 完成: [一句话结果]`
- 遇到问题时给用户选项，而不是自己默默选一个

**Subagent 分配任务示例**：
```
# 并行启动多个 subagent（使用 Agent 工具）
Agent(description="处理任务A", prompt="...", subagent_type="general-purpose")
Agent(description="处理任务B", prompt="...", subagent_type="general-purpose")
# 不使用 team_name 参数，subagent 独立运行后汇报结果
```

#### 模式B：Agent Team 协作执行
- **生成团队成员**：用`Agent(team_name, name)`生成成员，必须指定 team_name 和 name
- **成员间通信**：用`SendMessage`实现成员间协调
- **任务协调**：通过 TaskList + TaskUpdate 协调任务状态
- 每个成员完成后通过 SendMessage 汇报

**Agent Team 启动成员示例**：
```
# 团队已在阶段2创建，现在生成成员
Agent(team_name="my-project", name="frontend", description="前端开发", prompt="前端开发...")
Agent(team_name="my-project", name="backend", description="后端开发", prompt="后端开发...")

# 成员间通信协调
SendMessage(type="message", recipient="frontend", content="API已就绪", summary="API就绪通知")
```
**Agent → Skill 委派**（子agent调用skill的3种模式）：
`general-purpose`类型的subagent拥有所有工具权限，包括`Skill`工具。
| 模式 | 流程 | 适用场景 |
|------|------|---------|
| 协调器直调 | 协调器 → `Skill(skill="name")` → 结果 | 单步Skill任务，无需并行 |
| 委派式调用 | 协调器 → `Agent(prompt="请使用 /skill-name 完成 X")` → subagent → `Skill` → 汇报 | 并行多个Skill，或Skill耗时较长 |
| 团队成员调用 | `TeamCreate` → 分配任务 → member → `Skill` → `SendMessage`汇报 | 需要成员间协调的复杂任务 |
委派式调用关键点：Agent prompt中写明要调用的Skill名称和参数，subagent会自动识别并调用。
### 阶段4：质量把关 & 产品打磨（增强版）

**自动验收工具链**（根据改动类型自动选择）：
| 改动类型 | 推荐工具/Agent | 触发方式 | 检查内容 |
|---------|---------------|---------|---------|
| 前端代码（`.tsx/.jsx/.vue`等） | `web-design-guidelines` skill | `Skill(skill="web-design-guidelines")` | UI合规性、可访问性、设计规范、UX审计 |
| 所有代码改动 | `code-simplifier` agent | `Agent(subagent_type="code-simplifier:code-simplifier")` | 代码简化、一致性维护、可读性优化 |
| PR 代码审查 | `code-review` skill | `Skill(skill="code-review:code-review")` | Pull request 全量审查 |

**验收流程**：
1. **自动检测改动类型**
   - 扫描修改的文件扩展名
   - 匹配对应的验收工具
2. **并行执行专项检查**
   - 前端改动 → 调用 `web-design-guidelines` skill 验收
   - 所有改动 → 调用 `code-simplifier` agent（`Agent(subagent_type="code-simplifier:code-simplifier")`）优化代码
   - 有 PR → 调用 `code-review:code-review` skill
3. **功能完整性验收**
   - 对照阶段1的验收标准逐项检查
4. **产品打磨**（不仅功能完整，更要用户体验优秀）
   - 边界处理：异常输入、空值、极端情况是否覆盖
   - 专业度：命名规范、代码风格、错误提示是否友好
   - 完整性：文档、配置说明、使用示例是否齐全

**验收检查清单**：
- [ ] 功能完整性（对照阶段1验收标准）
- [ ] 代码质量（通过 code-simplifier）
- [ ] 前端专项检查（web-design-guidelines）
- [ ] 边界处理（异常输入、空值）
- [ ] 专业度（命名、风格、错误提示）
- [ ] 完整性（文档、配置、示例）

全部通过 → 进入阶段5。不通过 → 打回修改，最多2轮，仍不通过则通知用户人工介入。
### 阶段5：结果交付 & 部署移交
输出执行报告：
| 项目 | 内容 |
|------|------|
| 总任务数 | X个，成功Y个，失败Z个 |
| 各Agent结果 | [角色]: [状态] - [关键产出] |
| 汇总结论 | [综合所有结果的最终结论] |
| 后续建议 | [当前未覆盖但值得做的改进方向] |
**部署移交**（按需提供）：
- 运行方式：启动命令、环境要求、配置说明
- 验证步骤：用户可自行验证的操作清单
- 已知限制：当前版本的边界和约束
## 执行底线
**【硬性标准】**：
0. **强制使用 planning-with-files**：任何复杂任务必须先调用 `Skill(skill="planning-with-files")` 创建 task_plan.md、findings.md、progress.md
1. **强制执行信息获取完整回退链**：本地Skill扫描 → MCP联网搜索 → `Skill(skill="find-skills", args="...")` → 通用subagent，不允许跳过任何步骤
2. **代码提交只执行 commit，不 push**：需要提交代码时只调用 `/commit` skill 或 `Skill(skill="commit")`，绝不自动推送到远程仓库
**【其他原则】**：
3. 先目标，后组织结构——任务不清晰时先澄清，再决定是否组建团队
4. 队伍规模由任务复杂度决定，并行Agent建议不超过5个
5. 关键里程碑必须有质量闸门和回滚点
6. 不默认任何外部工具可用，执行前先验证（含find-skills）
7. 浏览器多窗口默认互相独立，不共享上下文
8. 成本只是约束，不是固定承诺——不做不切实际的成本预估
9. 危险操作、大规模变更必须先获得用户确认
## 故障处理
| 故障类型 | 处理策略 |
|---------|---------|
| Agent执行失败 | 通知用户，提供重试/跳过/终止选项 |
| Skill/MCP不可用 | 按回退链降级：本地Skill → MCP联网搜索 → find-skills → 通用subagent |
| 模型超时 | 调整任务复杂度或拆分为更小的子任务 |
| 质量不达标 | 打回修改最多2轮，仍不通过则人工介入 |
| 上下文溢出 | 拆分为更小的子任务，分批执行 |

### Agent执行失败时的错误处理示例
```
❌ [前端开发] 失败: 依赖安装失败 (npm install error)

请选择处理方式：
1. 重试 - 修复依赖问题后重新执行
2. 跳过 - 继续执行其他任务，前端工作暂时搁置
3. 终止 - 停止整个任务

请回复选择（1/2/3）或提出其他建议。
```

### 代码示例：带错误处理的subagent调用
```python
# 使用 Agent 工具调用 subagent，带错误处理
Agent(
    description="处理用户认证模块",
    prompt="实现用户登录和注册功能...",
    subagent_type="general-purpose"
)
# 注意：Agent工具执行后返回结果或通知

# Agent Team 成员失败时的处理
SendMessage(
    type="message",
    recipient="team-lead",
    content="[后端API] 执行失败: 数据库连接超时，正在重试...",
    summary="后端任务失败预警"
)
