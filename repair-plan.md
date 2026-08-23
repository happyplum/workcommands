---
description: Repair, normalize, and finalize a reviewed or imported execution plan before downstream execution.
subtask: false
---

You are executing the `/repair-plan` command.

Treat `$ARGUMENTS` as the target plan path, plan identifier, or the user's focused repair request.

# 计划修复

## 概述

现有计划的第二轮修复工作流。目标：修复计划结构、依赖真实性和验证对齐——而非重新界定产品意图。

**核心原则：** 先修复文档，再修复代码。
**执行标准：** 计划仅在所有硬关卡通过时才可安全交付 Atlas 执行。

**规范化入口：** 对 reviewed / imported plan 先判断它是否已经是本地治理认可的 execution-ready surface。若只是缺本地治理约束、仍可确定恢复，则进入 `normalize-before-execute`；若存在硬关卡失败或结构失真，再进入 `repair-before-execute`。

## 上游契约自检（必先执行）

**本 command 是二次质检器，不是契约来源。** 调用本 command 的 agent 必须在执行修复前**并行加载**下列 5 个上游 skill / prompt，按其**当前最新版本**做对齐检查，缺口进入关卡：

| # | 上游契约 | 加载方式 | 对齐检查抽象 |
|---|---|---|---|
| 1 | `ulw-plan` skill（导入兼容源） | `skill(name="ulw-plan")` | Plan artifact producer contract 的 checkbox 语法与 9 章节识别；仅作上游计划**导入**时的字段映射来源，输出收敛为本地五区块 |
| 2 | `omo-atlas-execution-constraints` skill | `skill(name="omo-atlas-execution-constraints")` | Atlas 执行边界（环境就绪、worktree 身份字段、`mode: worktree` 语义、worker 四态裁决） |
| 3 | `omo-adaptive-execution` skill | `skill(name="omo-adaptive-execution")` | `task()` 路由与委托契约（category XOR subagent_type、`[CONTEXT]/[GOAL]/[STOP WHEN]/[EVIDENCE]/[DOWNSTREAM]/[REQUEST]` 六段、worker 四态） |
| 4 | `prompts/atlas.md` | 直接读源 | 环境就绪检查、顶层 `workspaces` 身份解析（`name/path/branch`）、`mode: worktree` 处理、身份字段缺失即停止 |
| 5 | `prompts/prometheus.md` | 直接读源 | 顶层 `workspaces` 无条件存在、五区块正文结构、命名规范、`handoff`（路径/版本/状态/未决/入口） |
| 6 | `momus` | `subagent_type="momus"` | 测试时序裁决（四问判据：不读实现能否写测试/失败是否静默/是否语义变更/是否仅模式复制）与官方四类审查 |

**规则**：

- 本 command **不复制**上游契约原文。仅以下列抽象层级的「上游对齐检查」（见「必需检查 / 上游对齐层」）做缺口识别，缺口 → 修复方向 → 关卡代码。
- 加载失败（如某 skill/prompt 在当前环境不可用）→ 在修复报告中标注 `UPSTREAM_SOURCE_UNAVAILABLE` 并跳过该契约的对齐检查（**不**视为硬关卡失败）。
- 上游契约间的冲突由上游裁决；本 command 只识别「是否符合上游要求」，不裁定上游契约谁优先。

## 目标计划结构（强制对齐）

本 command 修复后的计划**必须**符合本地五区块 schema（与 prompts/prometheus.md「消费者与文档稳定性」一致）。不再使用 `Task N` / `Task N-V` / `CP0-CP3` / `Plan Size Audit` / `User Requirement Digest` / `Intent Anchor` / `Execution Skill Requirements` 等旧重型 schema，也不保留上游 9/11 章节输出结构。

修复后的计划文件**必须**按以下顺序包含五个静态区块（缺失区块由修复流程注入骨架）：

1. `# <plan-id> - Work Plan`
2. `## 需求与目标` —— 节首 3-5 行用户可读摘要；逐条需求附可追溯来源（用户原话引号 / 结论标注轮次 / 转述标注，不得混排）并标注 `core` / `preference`；未确认缺口保持未决不得自行补齐
3. `## Workspaces`（顶层身份）—— 每个 lane 一个条目，提供 `name` / `path` / `branch` 三项身份字段。**单 lane 计划也必须给全身份**（不省略此章节、不简化为 `single-lane` 字符串），否则 Atlas 环境就绪检查无解析入口 → `WORKSPACE_IDENTITY_MISSING`
4. `## 并发矩阵` —— 机器可消费区块：按 wave 分组呈现，逐 task 列出 cohort 归属、硬前驱、互斥写入与可变资源、workspace lane 与 route；每个 wave 节自带并发举证（输出依赖/唯一 owner/接口冻结/二元验收/资源隔离/墙钟论证）与本 wave 并发数（不超过 `concurrency_budget`）；task 恰好出现一次、硬前驱可解析、无环；单 writer 单 lane 可写 `cohorts: none`
5. `## Task 契约` —— 每个 task 一节（task ID 为顶层连续正整数，正文不写 checkbox），字段见「Task 契约字段契约」
6. `## 检查点与集成` —— 检查点声明（纳入 task 集合/放行条件/验收命令，检查点是唯一验收节点来源）；检查点断言标注证据强度；Final Wave 节点；全部检查点通过后才允许最终原子提交整理

### 上游导入映射

上游 9 章节（ulw-plan template）或 11 章节计划（含 `## Workspaces` / `## Handoff`）**只作导入识别与字段映射来源**，最终必须收敛为本地五区块，不得保留第二套输出结构：

| 上游章节 | 映射去向 |
|---|---|
| `## TL;DR (For humans)` | 需求与目标（节首摘要） |
| `## Scope` | 需求与目标（硬约束/非目标；Deferred → 未决项） |
| `## Verification strategy` | 检查点与集成（验收命令） |
| `## Workspaces` | Workspaces（身份字段原样保留） |
| `## Execution strategy`（Dependency matrix / Routing） | 并发矩阵 + Task 契约路由字段 |
| `## Todos` | Task 契约（无 checkbox，整数 task ID） |
| `## Final verification wave` | 检查点与集成（Final Wave 节点） |
| `## Commit strategy` | 检查点与集成（最终原子提交整理） |
| `## Success criteria` | 检查点与集成 |
| `## Handoff` | **不进正文**：handoff 只在交付消息中提供（计划路径/版本/状态/未决事项/执行入口），不写入计划文件 |

### Task 契约字段契约（每个任务节点必填）

每个 task 必须以加粗引用块或子列表形式提供以下字段（缺一触发 `TODO_FIELD_MISSING`）：

- **step_type** —— 步骤类型：`test-freeze`（前置红测试）/ `impl`（实现）/ `test-supplement`（后置补测试）/ `integration`（集成与汇合）
- **References** —— 涉及的精确文件路径与行号/区块（不可只有模糊描述）
- **Scope** —— 该任务做什么、不做什么（一句话边界）
- **Acceptance** —— 可观测的完成条件（命令、grep 结果、exit code 等）
- **Evidence** —— 该任务完成后产出的证据落点（产物路径 / diff 路径 / 测试输出路径 / 日志路径）；与 QA happy/QA failure 产出对齐；缺证据路径 → `EVIDENCE_PATH_MISSING`
- **QA happy** —— 成功路径的最小验证（含具体工具调用与预期结果）
- **QA failure** —— 失败路径的处理方式（不得静默继续；含具体工具调用与判定阈值）
- **Commit** —— 原子提交的中文动词短语（不含 task 编号/emoji/Co-Authored-By）
- **workspace_lane** —— 所属 lane 名（如 `main` / `gateway` / `runtime`）；single-lane 计划也必须显式写 `main`
- **category** 或 **subagent_type** —— 二选一（见「Routing 枚举」）。`category` 用于实现类任务；`subagent_type` 用于显式调用 explore/librarian/metis/oracle/momus 等只读专家
- **load_skills** —— 任务执行时预加载的 skill 名列表，可为 `[]`

可选字段（建议但非硬关卡）：
- **Pre-condition** —— 任务开始前必须成立的前置；若 BLOCKING 必须显式标注 `Pre-condition（BLOCKING）`

**并行裁决规则**（并行性以并发矩阵 wave 节为准，不靠本字段单独声明）：

1. 并行/串行关系写入并发矩阵的 cohort 归属与 wave 节并发举证；每 wave 并发数不超过 `concurrency_budget`
2. 同一 lane 内两个 task 的 References 写集（文件路径集合）存在重叠 → 不同 wave 或串行
3. 拆解产生的并行子任务的 References 必须显式互斥；不满足互斥的「并行」声明触发软警告 `PARALLEL_WRITESET_OVERLAP`

## 强制规则

1. **修复边界**：不得重新界定产品意图。仅修复结构、依赖真实性和验证真实性。未解决的硬关卡 → `REJECT`。高影响歧义 → 先问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`；不得猜测。
2. **确定性修复流程**：运行确定性两轮流程（先规范化，再硬关卡重评估）。保持输出可审计，带显式关卡码和固定章节。
3. **风险分层验证**：共享接口、跨模块集成、迁移、安全或高风险输出必须在 Acceptance + QA happy + QA failure 中显式覆盖；低风险本地任务可保持最简 QA。
4. **分解与路由纪律**：任务粒度采用最小内聚可验证结果。共享同一接口决策、不变量或验证面的工作保持同一任务。修复后的 Routing 表必须使用合法枚举值（见「Routing 枚举」）。贵价 category（`deep`、`ultrabrain`、`visual-engineering`、`artistry`）仅保留给确实需要专业能力的工作；保留时必须在该任务的 `WHY` 字段或 Routing 表 WHY 列写一行理由。
5. **用户侧防漂移锚点**：每个计划的用户可读摘要位于 `## 需求与目标` 节首（3-5 行）；不得另建摘要副本。
6. **执行命令格式**：当当前执行单位为 Prometheus/Atlas 时，输出的 `/start-work` 执行命令必须使用包含计划文件名（不含扩展名）的完整格式：`/start-work <filename>`。例如计划文件名为 `audit-p0-p1-fixes.md`，则执行命令为 `/start-work audit-p0-p1-fixes`。不得输出无文件名的裸 `/start-work`。
7. **平台到本地的收敛**：上游平台 runtime 术语（如 `sisyphus-junior`）可以作为 imported plan 的事实输入，但必须先被规范化成本地可执行的路由表达（Routing 表 + category 枚举），不能直接越过本地 schema 进入执行。

## Routing 枚举

Routing 表的 Task 行必须用 `category` **或** `subagent_type` 二选一声明执行者（与 `omo-adaptive-execution` 的 `task()` schema 一致）：

- **`category` 允许值**（实现类任务）：`visual-engineering` / `ultrabrain` / `deep` / `artistry` / `quick` / `unspecified-low` / `unspecified-high` / `writing`
- **`subagent_type` 允许值**（只读专家）：`explore` / `librarian` / `metis` / `oracle` / `momus`
- **兜底**：当任务执行需要由 executor 现场决定路由（例如 PLAN 标注 `executor_judgment` 或 `routing_by_executor`），Routing 表可在 WHY 列显式声明并留空 category/subagent_type 列；该兜底不得滥用，每个使用必须给出理由

`load_skills` 仅允许使用项目有效 skill 名（见 skills 索引）或 `[]`。

每个任务节点必须以 `category` 或 `subagent_type` 声明执行者（或显式兜底）。`load_skills` 默认 `[omo-adaptive-execution]`。

Routing 表若无法表达 `subagent_type` 或兜底语义 → 触发软警告 `ROUTING_DISPATCHER_UNSUPPORTED`（不阻断，提示按 omo-adaptive-execution 最新规范对齐）。

## 高成本任务拆解分析（必执行章节）

### 触发对象

对每个 `category ∈ {deep, unspecified-high, ultrabrain}` 的**可执行叶节点** task，必须执行拆解分析。**不是凡贵必拆**——分析后输出明确结论：可拆 / 不可拆。

**拆解终止规则**（防递归）：

1. **只分析可执行叶节点** —— 已被拆解为父节点的 task 不再触发分析（父节点通过 References 字段的 `decomposed_into: [...]` 标记为非执行，见「拆解输出」）
2. **单次修复最多拆一层** —— 子任务即使仍是高成本 category，也不再进入第二轮拆解分析
3. **残留高成本叶节点的强制结论** —— 若拆解后某子任务仍是 `deep` / `unspecified-high` / `ultrabrain`，必须在该子任务末尾追加 `[WHY_NOT_SPLIT]` 理由（如「再拆会破坏契约内聚」），不得留作隐式高成本

### 分析维度（逐项核对并记录结论）

对每个高成本任务，沿以下维度判断是否存在 ≥2 个**可独立验收、可独立失败、可并行**的子结果：

| 维度 | 倾向可拆 | 倾向不拆 |
|---|---|---|
| References 跨文件数 | ≥2 文件 / 跨包 | 单文件局部 |
| 估算改动行数 | ≥150 行 | <150 行 |
| Pre-condition | 含 BLOCKING 前置 / characterization test | 无前置 |
| 子结果独立性 | 多个独立可验收产出 | 共享同一不变量/接口决策 |
| 并行收益 | 子结果可并行执行显著缩短关键路径 | 强顺序依赖，拆开只增交接 |
| 风险隔离需求 | 子结果各自需要独立 QA failure 路径 | 同一验证面统一处理 |

### 拆解输出

**可拆** → 父任务改为**无 checkbox 的编排说明**（不参与执行），所有可执行子任务与中间校验点**重新分配顶层连续正整数 task ID**，同时原子更新并发矩阵、硬前驱、路由与检查点中的引用：

- 每个子任务**完整复用 Task 契约字段契约**（References / Scope / Acceptance / QA happy / QA failure / Commit / workspace_lane / step_type / category / load_skills）
- 子任务的 `category` 通常**降级**（`deep` → `unspecified-high` 或 `quick`；`ultrabrain` → `deep` 或 `unspecified-high`），并附降级理由；**例外**：子任务命中风险特征（lifecycle 恰好一次动作 / 生产装配点语义变更 / 需先钉住错误被吞没的现状）时路由不得低于 `unspecified-high`，风险下限优先于拆解降级
- 子任务**继承父任务的 wave 与 workspace_lane**（不新建 sub-Wave）；中间校验点表现为 Wave 内的阻塞依赖边
- 子任务之间的并行/串行关系写入并发矩阵的 cohort 归属与 wave 节并发举证
- 子任务的 References 写集若声明可并行则必须**互斥**（同一文件路径不可被多个并行子任务写入）
- **父任务转为非执行编排节点**：
  - References 字段改为 `decomposed_into: [<子任务编号列表>]`
  - **不进入**并发矩阵与路由（只列可执行叶节点）
  - Scope 改为「编排子任务」
  - 父任务不触发任何 `task()` 调用

**不可拆** → 在该 task 节点的 Routing 表 WHY 列或任务体末尾追加一行：

```
- **[WHY_NOT_SPLIT]**: <一行理由，如「单文件单接口决策，拆开增加交接不增加并行收益」>
```

### 拆解后中间校验点（强制）

凡被拆解为 ≥2 个子任务的高成本任务，必须在子任务序列中注入**中间校验点**。校验点本身是子任务（带完整字段契约），位于关键子任务之间，承担三类职责之一：

1. **基线锁定** —— 在改动前用 characterization test / git diff / wire 捕获锁定现有行为（如 KB 适配器迁移前的 status 映射锁定）
2. **契约对齐** —— 在两个子任务共享接口边界时验证双方契约一致（如 canonical port 补齐后再切消费者）
3. **diff 闭合** —— 在并行子任务汇合前验证无漂移（如 gateway + runtime lane 合并回 main lane 前的 parity test）

中间校验点的 `category` 通常为 `quick` 或 `unspecified-low`；`Acceptance` 必须是可运行的命令或可观测的检查；`QA failure` 必须 BLOCKING（拦截下游子任务继续）。

如果分析后认为该任务无需中间校验点（极少见），必须显式写 `- **[NO_MIDPOINT_JUSTIFIED]**: <理由>`，否则触发 `MIDPOINT_MISSING`。

## Worktree 环境校验阶段（强制 Wave 0 / Task 0）

### 触发条件

满足以下任一条件时，修复流程**必须**确保计划含一个 Wave 0 / Task 0「环境就绪」节点：

- 计划含多 lane 条目（在顶层 `## Workspaces` 表中）
- 任何 task 节点的 `workspace_lane` 字段不为 `main`
- Workspaces 中显式列出 ≥2 个 worktree 路径

`single-lane` 计划（仅 main lane、无多 lane 条目）不触发本阶段；但**顶层 `## Workspaces` 章节仍必须存在**并给出 main lane 的身份字段（见硬关卡 `WORKSPACE_IDENTITY_MISSING`）。

### Wave 0 / Task 0 字段契约

注入的 Task 0 必须满足 Task 契约字段契约，并额外覆盖：

- **Scope**：校验或创建所有 Workspaces 表声明的 worktree；不覆盖已占用路径；记录所有 lane 的共同 base SHA
- **References**：本计划 `## Workspaces` 表
- **package_manager**（额外字段，本任务专用）：显式声明项目包管理器与 install 命令。允许值：`pnpm: pnpm install --frozen-lockfile` / `npm: npm ci` / `yarn: yarn install --frozen-lockfile` / `cargo: cargo fetch --locked` / `go: go mod download` / `maven: mvn -o dependency:resolve` / `<other>: <command>`。未声明 → 触发软警告 `PKG_MANAGER_UNDECLARED`
- **Acceptance**：
  - `git worktree list` 显示 Workspaces 表中的每个路径
  - 每个 worktree 内 `git rev-parse --abbrev-ref HEAD` 返回 Workspaces 表对应 Branch 列
  - 每个 worktree 内运行 Task 0 声明的 install 命令并 exit 0
  - 多 lane 计划中每个 lane 的 base SHA 一致
- **QA happy**：上述检查全部通过
- **QA failure**：
  - worktree 路径已存在但 head 或 base 不匹配预期 → `BLOCKED_NEEDS_DECISION`，不强制切换
  - worktree 路径不存在 → 走「分支创建协议」
  - 路径被无关内容占用 → 报错并停止，不覆盖
  - 多 lane base SHA 不一致 → 硬关卡 `BASE_SHA_DIVERGENT`
- **Commit**：无（基础设施）
- **workspace_lane**：`main`（此任务负责创建/校验所有 lane）
- **category**：`quick`
- **load_skills**：`[omo-adaptive-execution]`

### 分支创建协议

当 worktree 缺失时，按以下顺序操作，**不得跳步**：

1. **声明解析**：从 Workspaces 表读取 `trunk=<branch>`（默认 `main`）与 `remote=<name>`（默认 `origin`）。未声明则用默认值并写入 Task 0 证据区
2. **plan-id 安全化**：将 `<plan-id>` 转 lowercase；非 `[a-z0-9-]` 字符替换为 `-`；连续 `-` 合并为单个；长度截断至 50 字符。结果用于 `work/<plan-id>/<lane>` 分支名
3. **主干状态校验**（在主仓执行）：
   - `git status --porcelain` 非空 → 软警告 `TRUNK_DIRTY`，让用户决定是否继续
   - `git rev-parse HEAD` ≠ `git rev-parse @{u}` → 软警告 `TRUNK_UNPUSHED`，让用户决定
4. **拉取远端引用**：`git fetch <remote>`；失败 → `BLOCKED_NEEDS_DECISION`
5. **解析并锁定 base SHA**：`git rev-parse <remote>/<trunk>` 写入 Task 0 证据区，作为本计划所有 lane 的共同 base SHA
6. **幂等复用**：`git worktree list` 已含目标路径 → 校验其 head 指向 `work/<plan-id>/<lane>` 且 base SHA 一致 → 通过；不匹配 → `BLOCKED_NEEDS_DECISION`，不强制覆盖
7. **创建分支与 worktree**：
   - `git branch work/<plan-id>/<lane> <remote>/<trunk>`
   - `git worktree add <path> work/<plan-id>/<lane>`
8. **依赖安装**：在新 worktree 内运行 Task 0 声明的 install 命令；exit ≠ 0 → `BLOCKED_NEEDS_DECISION`
9. **证据落盘**：将每个 lane 的（分支名、worktree 路径、base SHA、HEAD commit、install 命令、install exit code）写入 Task 0 Acceptance 证据区

任何步骤失败 → `BLOCKED_NEEDS_DECISION` 并停止后续任务；不得猜测错误恢复路径。

### 跨 lane 集成 task（强制注入）

**触发条件**（同时满足）：

- Workspaces 表含 ≥2 个非 main lane
- main lane 的某个 task 在并发矩阵中依赖其他 lane 任务

**注入规则**：在首个「消费其他 lane 产物」的 main lane task **之前**注入一个 lane-merge task，字段如下：

- **References**：本计划 Workspaces 表 + 各非 main lane 的最终 HEAD commit
- **Scope**：将所有非 main lane 的产出 merge 或 cherry-pick 到 main；不在本 task 实现业务逻辑
- **Pre-condition（BLOCKING）**：上游 lane 的所有 task 已完成且各自 Acceptance 通过；解除形式为各上游 lane task 输出的 commit SHA
- **Acceptance**：`git log <main>` 显示来自每个非 main lane 分支的 merge commit；main lane 运行检查点与集成区块声明的总门禁命令（如 `pnpm run verify` / `cargo test` / `go test ./...`）通过
- **QA happy**：merge 无冲突；总门禁 exit 0
- **QA failure**：merge 冲突或门禁失败 → BLOCKING；触发 `BLOCKED_NEEDS_DECISION`，不自动 resolve
- **Commit**：合并上游 lane 到主干
- **workspace_lane**：`main`
- **category**：`quick`
- **load_skills**：`[omo-adaptive-execution]`

### F2.5 Lane merge parity（最终复核）

`F2.5` **不替代**上述「跨 lane 集成 task」，只做最终复核：

- 每个 non-main lane 的提交已在 main lane 上可见
- main lane 通过项目总门禁
- 各 lane 分支相对 base SHA 无意外丢失
- 多 lane 计划中各 lane base SHA 一致

`F2.5` **阻塞** `F2` 与 `F3`：F2.5 未通过则 F2/F3 不得开始。

## 故障处理

1. 缺少输入 → 仅修复确定性格式问题后停止。
2. 高影响歧义 → 问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`。
3. 规范化后关卡反复失败 → `REJECT` 并附失败关卡集。
4. 不做叙述性信心声明；仅输出可执行的判定产物。

## 提示精简契约

本 command 是显式稳定计划的修复/验证权威来源。Prometheus 将修复语义委托至此。共享的分解、路由和提级原则来自 `omo-adaptive-execution`。不要在 Prometheus 中复制冗长的硬关卡规则块；将修复专用关卡保留在此。

## 第二轮修复模式（强制）

1. **第一轮——结构规范化**：
   - 识别/补齐目标计划结构为五区块（需求与目标 / Workspaces / 并发矩阵 / Task 契约 / 检查点与集成）；上游 9/11 章节导入计划按「上游导入映射」表收敛
   - 展平任何嵌套可执行项到 Task 契约列表（顶层连续正整数 task ID，无 checkbox）
   - 识别 Workspaces 表 → 注入或补齐 Wave 0 / Task 0
   - 识别高成本任务 → 执行拆解分析 → 注入子任务 + 中间校验点
   - 规范化 Routing 表（category 枚举、load_skills 名）
   - 统一并发矩阵引用的 task ID；补齐 wave 节并发举证与并发数

2. **第二轮——关卡重评估**：在规范化后的计划上重跑所有硬关卡；失败时输出带关卡码的 `REJECT`；仅当硬关卡失败数 = 0 时输出 `PASS`。

任何结构变更后不得跳过第二轮。

## 必需检查

### 计划结构层

1. **需求与目标摘要完整** —— `## 需求与目标` 节首含 3-5 行用户可读摘要；逐条需求附可追溯来源（用户原话引号 / 结论标注轮次 / 转述标注）并标注 `core` / `preference`；未确认缺口保持未决不得自行补齐。关卡：`TLDR_MISSING`。
2. **需求与目标边界完整** —— `## 需求与目标` 含硬约束 / 非目标 / 未决项；未决项必须显式标注 `BLOCKED on <决策项>`。关卡：`SCOPE_MISSING`。
3. **验收命令可执行** —— `## 检查点与集成` 中的验收命令必须可运行（非模糊叙述）。关卡：`VERIFICATION_STRATEGY_NOT_EXECUTABLE`。
4. **Workspaces 身份一致** —— `## Workspaces` 每个 lane 条目含 `name` / `path` / `branch` 三项身份字段；每个 lane 在 Task 契约中至少出现一次 `workspace_lane`；Task 契约中出现的 lane 必须在 Workspaces 声明。关卡：`WORKSPACE_TABLE_INCOMPLETE`。
5. **Routing 枚举合法 + 与可执行 task 一一对应** —— Routing 表的 category 列只允许使用「Routing 枚举」中的值；`load_skills` 只允许有效 skill 名或 `[]`。Routing 行必须与 Task 契约中的**可执行叶节点**一一对应：(a) 每个可执行叶节点（即 References 不含 `decomposed_into` 的 task）必须在 Routing 表中出现且仅出现一次；(b) Routing 表中每个 Task 必须对应一个存在的可执行叶节点；(c) 父编排节点（含 `decomposed_into` 的 task）不得出现在 Routing 表中。关卡：`ROUTING_SCHEMA_INVALID`。允许的 `category`：`visual-engineering` / `ultrabrain` / `deep` / `artistry` / `quick` / `unspecified-low` / `unspecified-high` / `writing`。
6. **Final Wave 完整** —— `## 检查点与集成` 含 Final Wave 节点（`F1` 计划合规 / `F2` 代码质量 / `F3` 全量 QA / `F4` 范围保真四项）；多 lane 计划额外含 Lane merge parity 关卡。关卡：`FINAL_VERIFICATION_MISSING`。
7. **检查点断言可观测** —— `## 检查点与集成` 每条断言必须可独立验证（命令、grep、文件存在、exit code 等），拒绝纯叙述，并标注证据强度（集成实测 / 切片单测拼装 / 类型检查）。关卡：`SUCCESS_CRITERIA_VAGUE`。

### Task 契约层

8. **Task 契约字段完整** —— 每个 task 节点含必填字段（References / Scope / Acceptance / QA happy / QA failure / Commit / workspace_lane / step_type / category / load_skills）。测试组织符合 Momus 裁决：test-first 计划含前置红测试 task 且其验收绑定契约 ID；test-first 任务测试先行、实现随后，tests-after 任务附判据依据。关卡：`TODO_FIELD_MISSING`。
9. **QA 可执行性** —— Acceptance / QA happy / QA failure 必须含具体命令 + 可观测预期 + 证据目标；拒绝纯叙述步骤。关卡：`QA_NOT_EXECUTABLE`。
10. **并发矩阵闭合** —— 并发矩阵引用的所有 task ID 必须在 Task 契约中存在；并行/串行关系必须在 wave 节并发举证中体现；每个 wave 节含并发举证与本 wave 并发数（≤ `concurrency_budget`），且 wave 节与全局并发矩阵一致；移除幻影依赖；依赖图必须无环（T→…→T 回路直接判 `DEPENDENCY_GRAPH_OPEN`）。关卡：`DEPENDENCY_GRAPH_OPEN`。
11. **Pre-condition 显式** —— 含 BLOCKING 前置的任务必须用 `Pre-condition（BLOCKING）` 显式标注，并指明 BLOCKING 解除的产出形式。关卡：`PRECONDITION_UNMARKED`。
12. **范围保真** —— Task 契约的 References 与需求与目标的硬约束/非目标不冲突；任何 task 不得触碰非目标或未决项。关卡：`SCOPE_LEAK`。

### 高成本任务与 Worktree 层

13. **拆解分析覆盖** —— 对每个 `deep` / `unspecified-high` / `ultrabrain` 任务，必须输出「可拆 / 不可拆」结论；可拆必须注入子任务 + 中间校验点；不可拆必须含 `[WHY_NOT_SPLIT]`。关卡：`DECOMPOSITION_ANALYSIS_MISSING`。
14. **中间校验点闭合** —— 被拆解为 ≥2 子任务的高成本任务，子任务序列必须含至少一个中间校验点（基线锁定 / 契约对齐 / diff 闭合）；若无则必须含 `[NO_MIDPOINT_JUSTIFIED]`。关卡：`MIDPOINT_MISSING`。
15. **子任务降级合理** —— 子任务的 `category` 不得高于父任务；降级必须在 Routing 表 WHY 列或任务体写一行理由。例外：子任务命中风险特征（lifecycle 恰好一次动作 / 生产装配点语义变更 / 需先钉住错误被吞没的现状）时路由不得低于 `unspecified-high`，风险下限优先于拆解降级。关卡：`SUBTASK_CATEGORY_INFLATED`。
16. **Worktree 前置就绪** —— 触发 worktree 阶段的计划必须含 Wave 0 / Task 0；Task 0 的 Acceptance 覆盖 `git worktree list` / 分支匹配 / 声明的 install 命令（来自 Task 0 的 `package_manager` 字段）三项；QA failure 含从主干建分支协议；多 lane 计划还需校验各 lane base SHA 一致。关卡：`WORKTREE_PREFLIGHT_MISSING`。

### 治理层

17. **平台术语收敛** —— imported plan 中的 `sisyphus-junior` 等上游 runtime 标签必须先规范化为 Routing 表 + category / subagent_type 枚举；不得直接进入执行。关卡：`PLATFORM_TERM_LEAKED`。
18. **执行命令格式** —— 输出含 `/start-work <plan-name>`（plan-name 为计划文件名不含扩展名），可附 options `--worktree <path>` / `--make-pr` / `--ship`；plan-name 必须与计划 `# <plan-id> - Work Plan` 中的 plan-id 一致。关卡：`START_WORK_COMMAND_MALFORMED`。

### 上游对齐层（按已加载的上游契约做缺口识别）

19. **顶层 workspaces 身份完整** —— `## Workspaces` 顶层章节存在；每个 lane 条目含 `name` / `path` / `branch` 三项身份字段；单 lane 计划也必须给全身份（不简化为 `single-lane` 字符串）。关卡：`WORKSPACE_IDENTITY_MISSING`。修复方向：把 `### Workspaces`（在 `## Execution strategy` 下）四列表的身份字段提升为顶层 `## Workspaces` 条目。
20. **Handoff 信息就位** —— handoff 五要素（计划路径 / 版本 / 当前状态 / 未决事项 / 执行入口）只在交付消息中提供，**不写入计划文件正文**；执行入口派生自 plan-id 与可选 options。关卡：`HANDOFF_MISSING`。
21. **Evidence 字段对齐** —— 每个可执行 task 含 `Evidence` 字段，指向具体产物路径。关卡：`EVIDENCE_PATH_MISSING`。
22. **task ID 体系一致** —— 全文 task ID 统一为顶层连续正整数，正文不写 checkbox；并发矩阵引用必须匹配该体系；不允许小数后缀。关卡：`TODO_ID_SCHEME_INCONSISTENT`（软警告）。
23. **`mode: worktree` 字段对齐** —— 若计划含多 lane，应在 `## Workspaces` 或顶层声明 `mode: worktree`（具体字段形态以上游 atlas.md 最新定义为准）；单 lane 可选。关卡：`WORKTREE_MODE_UNDECLARED`（软警告）。
24. **Routing 派发器表达** —— Routing 表能表达 `category` XOR `subagent_type` + executor 兜底；不能表达则触发软警告 `ROUTING_DISPATCHER_UNSUPPORTED`，提示按 omo-adaptive-execution 最新规范对齐。

## 修复顺序

1. `## 需求与目标`：节首 3-5 行用户可读摘要；逐条需求附溯源并标注 `core` / `preference`；硬约束/非目标/未决项（Deferred → 未决项，标注 BLOCKED）
2. `## Workspaces`：每个 lane 补齐 `name` / `path` / `branch` 身份字段
3. `## Task 契约`：字段完整（References / Scope / Acceptance / QA happy / QA failure / Commit / workspace_lane / step_type / category / load_skills）；测试组织按 Momus 裁决落位（test-freeze / impl / test-supplement）
4. `## 并发矩阵`：wave 分组、cohort 归属与硬前驱、互斥写集；每 wave 并发举证 + 并发数（≤ `concurrency_budget`）；task ID 闭合无环
5. **Worktree 触发判定** → 若触发，注入或补齐 Wave 0 / Task 0（含从主干建分支协议）
6. **高成本任务拆解分析**：识别 deep / unspecified-high / ultrabrain → 分析维度 → 可拆则注入子任务 + 中间校验点；不可拆则附 `[WHY_NOT_SPLIT]`
7. `## 检查点与集成`：检查点声明（放行条件 + 验收命令）；断言标注证据强度；Final Wave F1-F4；多 lane 计划强化 Lane merge parity；最终原子提交整理收敛于此
8. 第二轮硬关卡重评估

## 输出要求

产出：

- `Verdict`：`PASS` 或 `REJECT`
- `Gate Summary`：按关卡码统计的失败硬关卡数
- `Hard Gates`：关卡码 + 失败章节列表
- `Warnings`：非阻塞质量问题（含软警告码）
- `Fixed Sections`：已修改的确切章节
- `Needs Decision`：需要人工产品/契约决策的条目
- `需求与目标 Confirmed`：确认节首 3-5 行用户可读摘要 + 溯源 + `core` / `preference` 标注就位
- `边界 Confirmed`：确认硬约束 / 非目标 / 未决项就位且未决项标 BLOCKED
- `Worktree Preflight`：是否触发、注入了哪些 lane 的 Wave 0 / Task 0、主干分支名、哪些 lane 已就绪 / 待创建
- `Task Decomposition Report`：列出每个被分析的高成本任务——结论（可拆/不可拆）、拆出的子任务编号、注入的中间校验点编号、降级的 category、或不拆的 `[WHY_NOT_SPLIT]` 理由
- `并发矩阵`：wave 分组、并发举证、并发数与依赖图摘要
- `Routing Audit`：category / subagent_type 枚举合法性、贵价任务 WHY 列理由是否就位、executor 兜底声明是否滥用
- `Upstream Contract Self-Check`：6 个上游契约的加载状态（已加载 / `UPSTREAM_SOURCE_UNAVAILABLE`）+ 各自对齐检查的缺口摘要
- `Handoff Explanation`：六元素——`What this plan drives`（计划驱动什么）/ `End state`（最终态）/ `Shape`（N impl + F final-verifier）/ `Added beyond the request`（在用户请求外补强的内容）/ `Verification`（F1-F4 + 上游对齐检查）/ `Execution handoff`（`/start-work <plan-name>` 含可选 options）；**只在交付消息中提供，不进计划正文**
- `Start-Work Command`：完整 `/start-work <plan-name>` 命令（含 options `--worktree <path>` / `--make-pr` / `--ship` 若适用）

任何 `BLOCKED_NEEDS_DECISION` 条目仍开放 → 判定必须为 `REJECT`。

## 硬关卡 vs 软警告

### 硬关卡（必须为零才能 PASS）

`TLDR_MISSING` · `SCOPE_MISSING` · `VERIFICATION_STRATEGY_NOT_EXECUTABLE` · `WORKSPACE_TABLE_INCOMPLETE` · `ROUTING_SCHEMA_INVALID` · `FINAL_VERIFICATION_MISSING` · `SUCCESS_CRITERIA_VAGUE` · `TODO_FIELD_MISSING` · `QA_NOT_EXECUTABLE` · `DEPENDENCY_GRAPH_OPEN` · `PRECONDITION_UNMARKED` · `SCOPE_LEAK` · `DECOMPOSITION_ANALYSIS_MISSING` · `MIDPOINT_MISSING` · `SUBTASK_CATEGORY_INFLATED` · `WORKTREE_PREFLIGHT_MISSING` · `BASE_SHA_DIVERGENT` · `PLATFORM_TERM_LEAKED` · `START_WORK_COMMAND_MALFORMED` · `WORKSPACE_IDENTITY_MISSING` · `HANDOFF_MISSING` · `EVIDENCE_PATH_MISSING`

### 软警告

`ROUTING_UNDERKILL` · `ROUTING_OVERKILL` · `TASK_MAY_UNDER_DECOMPOSE` · `VERIFICATION_REDUNDANT` · `MIDPOINT_EXCESSIVE` · `TLDR_DRIFT_AFTER_SPLIT` · `PKG_MANAGER_UNDECLARED` · `TRUNK_DIRTY` · `TRUNK_UNPUSHED` · `PARALLEL_WRITESET_OVERLAP` · `TODO_ID_SCHEME_INCONSISTENT` · `WORKTREE_MODE_UNDECLARED` · `ROUTING_DISPATCHER_UNSUPPORTED`

`TLDR_DRIFT_AFTER_SPLIT` 触发条件：拆解改变了需求与目标节首摘要中**投入量级**（如 1 个 ultrabrain 拆为 3 个 unspecified-low，关键人力分布变化）、**风险等级**或**关键路径**（如拆解引入新的并行路径），但摘要未相应更新。

`ROUTING_DISPATCHER_UNSUPPORTED` 触发条件：Routing 表格式无法表达 `subagent_type` 或 executor 兜底语义（如计划只允许 category 列）；不阻断，提示按 omo-adaptive-execution 最新规范扩展列定义。

## 提级流程

1. 自动修复确定性文本问题（章节骨架、字段补齐、枚举修正、依赖边显式化）。
2. 高影响的模糊产品决策 → 问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`。示例：选择 `/api/` vs `/api/v1/`、是否将某功能纳入或排除出 Scope IN、Deferred 项的 BLOCKED 决策无法定位。
3. 绝不猜测业务意图来「强制通过」。

## 与 Atlas/Prometheus 集成

- 权威性结构修复的落地路径，用于权威性计划编辑。在执行前和审查驱动的缺陷发现后使用。
- 定义结构有效、修复完成的计划应是什么样；不负责运行时执行顺序、证据纪律或提交时机。
- Atlas 仅在动态执行连续两轮结构重排仍无法闭合、用户要求冻结计划，或需要长期计划资产时使用此规范。Prometheus 提供紧凑的路由意图；本 command 展开并执行具体修复关卡。
- `metis` 可能暴露遗漏，`oracle` 可能产出修订简报；局部依赖变化由 `omo-adaptive-execution` 在不改变用户目标的前提下重排，权威计划结构变更再通过本 command 落地。
- 输出在当前审查消息中行内发出；不需要单独的产物文件。
