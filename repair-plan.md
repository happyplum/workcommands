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

**规范化入口：** 对 reviewed / imported plan 先判断它是否已经是本地治理认可的 execution-ready surface（`级别：轻量` 三节或 `级别：完整` 五区块，见 `omo-plan-structure`）。若只是缺本地治理约束、仍可确定恢复，则进入 `normalize-before-execute`；若存在硬关卡失败或结构失真，再进入 `repair-before-execute`。

## 上游契约自检（必先执行）

**本 command 是二次质检器，不是契约来源。** 调用本 command 的 agent 必须在执行修复前**并行加载**下列 4 个上游 skill，按其**当前最新版本**做对齐检查，缺口进入关卡：

| # | 上游契约 | 加载方式 | 对齐检查抽象 |
|---|---|---|---|
| 1 | `omo-plan-structure` skill（本地计划结构单一标准） | `skill(name="omo-plan-structure")` | 计划分级与判级行、轻量三节 / 完整五区块、Task 5 字段、验收条目机械语法、矩阵结构约束、Workspaces schema、计划/账本分离 |
| 2 | `ulw-plan` skill（导入兼容源） | `skill(name="ulw-plan")` | 上游计划**导入**时的章节识别与字段映射来源（checkbox 语法、`Recommended task executor category:` 路由锚）；输出收敛为本地结构，不保留上游 9/11 章节 |
| 3 | `omo-adaptive-execution` skill | `skill(name="omo-adaptive-execution")` | `task()` 路由与委托契约（category XOR subagent_type、上游六段 `TASK / EXPECTED OUTCOME / REQUIRED TOOLS / MUST DO / MUST NOT DO / CONTEXT`、worker 四态、Category 词表与 `WHY_NOT_LOWER_COST`） |
| 4 | `omo-atlas-execution-constraints` skill | `skill(name="omo-atlas-execution-constraints")` | Atlas 执行边界（环境就绪、worktree 身份字段、`mode: worktree` 语义、worker 四态裁决） |

**规则**：

- 本 command **不复制**上游契约原文。仅以上述抽象层级的「上游对齐检查」（见「必需检查 / 上游对齐层」）做缺口识别，缺口 → 修复方向 → 关卡代码。
- 加载失败（如某 skill 在当前环境不可用）→ 在修复报告中标注 `UPSTREAM_SOURCE_UNAVAILABLE` 并跳过该契约的对齐检查（**不**视为硬关卡失败）。
- 对齐基准只来自上述 skill；**不读取角色 prompt 文件作为执行依据**——角色 prompt 承载的是该角色的行为条款，读取会把他人角色条款注入执行上下文；计划对齐契约以 `omo-plan-structure` 为准，其余关卡所需要素（workspaces 身份、handoff 要素等）由「必需检查」条目自足承载。
- 上游契约间的冲突由上游裁决（结构类以 `omo-plan-structure` 裁决）；本 command 只识别「是否符合上游要求」，不裁定上游契约谁优先。

## 目标计划结构（强制对齐）

计划结构以已加载的 `omo-plan-structure` 为单一标准；本节只列关卡所需结构锚点，不复制定义。不再使用 `Task N` / `Task N-V` / `CP0-CP3` / `Plan Size Audit` / `User Requirement Digest` / `Intent Anchor` / `Execution Skill Requirements` 等旧重型 schema，不保留上游 9/11 章节输出结构，也不使用 `step_type` / `QA happy` / `QA failure` / 无 checkbox task ID 等旧字段体系。

修复后的计划**必须**：

1. 首行为判级行 `级别：轻量 | 完整`（判级判据、高风险特征与升格规则按 `omo-plan-structure`）
2. **轻量计划**三节：`## 摘要`（3-5 行 + core 二元清单 + `假设：` / `不做的事：` 子行 + 节尾 Workspaces 一行声明）、`## 任务清单`、`## 终态验收`（具名 gate + `- [ ] F1. 终态：<命令>` 行 + 基线预验单行；默认 `checkpoints: none`）
3. **完整计划**按序五个静态区块：
   - `# <plan-id> - Work Plan`（标题）
   - `## 需求与目标` —— 节首 3-5 行用户可读摘要；逐条需求附可追溯来源（用户原话引号 / 结论标注轮次 / 转述标注，不得混排）并标注 `core` / `preference`；未确认缺口保持未决不得自行补齐
   - `## Workspaces`（顶层身份）—— 标注 `vcs: git | none` 与 `mode: current | worktree`；`vcs: git` 时标明主分支与计划/账本存放路径（计划 `.omo/plans/<plan-name>.md`、账本 `.omo/ulw-execute/ledger.jsonl`）；`mode: current` 必须记录 `authorization_source`；每个 lane 条目提供 `name` / `path` / `branch` 三项身份字段，**单 lane 计划也必须给全身份**（否则 Atlas 环境就绪检查无解析入口 → `WORKSPACE_IDENTITY_MISSING`）
   - `## 并发矩阵` —— 唯一拓扑事实源：逐 task 列出 cohort 归属、硬前驱、仅集成关联、owner、互斥写入与可变资源、workspace lane 与 route；task 恰好出现一次、硬前驱可解析、无环；`concurrency_budget` 声明与上限；单 writer 单 lane 可写 `cohorts: none`
   - `## Task 契约` —— 每个 task 一个列表条目（带 checkbox 标题行），字段见「Task 契约字段契约」
   - `## 检查点与集成` —— 检查点声明（纳入 task 集合 / 放行条件 / 验收命令，检查点是唯一验收节点来源）或显式 `checkpoints: none`；断言标注证据强度；Final Wave 节点；全部检查点通过后才允许最终原子提交整理
4. 执行期超判据（第 4 个 task、第二 lane、新增高风险）以一次 `topology_remap` 升格为完整结构重排，既有验收条目 `ID` 不变

### 上游导入映射

上游 9 章节（ulw-plan template）或 11 章节计划（含 `## Workspaces` / `## Handoff`）**只作导入识别与字段映射来源**，最终必须收敛为本地轻量三节或完整五区块，不得保留第二套输出结构：

| 上游章节 | 映射去向 |
|---|---|
| `## TL;DR (For humans)` | 摘要 / 需求与目标（节首摘要） |
| `## Scope` | 摘要 / 需求与目标（硬约束 / 非目标 / `不做的事：`；Deferred → 未决项） |
| `## Verification strategy` | 终态验收 / 检查点与集成（验收命令 → 验收条目机械语法） |
| `## Workspaces` | Workspaces（身份字段原样保留，补 `vcs` / `mode` / 主分支 / 存放路径标注） |
| `## Execution strategy`（Dependency matrix / Routing） | 并发矩阵 + Task 契约路由行 |
| `## Todos` | Task 契约（带 checkbox 标题行 `- [ ] N. <标题>`，路由行保留 `Recommended task executor category:` 前缀） |
| `## Final verification wave` | 终态验收 / 检查点与集成（Final Wave 节点） |
| `## Commit strategy` | 检查点与集成（最终原子提交整理 / 一行提交意图） |
| `## Success criteria` | 检查点与集成（标注证据强度） |
| `## Handoff` | **不进正文**：handoff 只在交付消息中提供（计划路径/版本/状态/未决事项/执行入口），不写入计划文件 |

### Task 契约字段契约（每个实施 task 必填）

固定字段合计 ≤5 行（条件字段另计），密度以 decision-complete 为准：

- **标题行** —— `- [ ] N. <标题>`：一行内聚意图（交付什么可观察结果、服务哪个下游）；集成 task 用 `[integration]` 前缀，普通实现无前缀；**不设测试专用前缀**（测试组织由上游 QA per todo 契约承接，tdd 时序由 Momus 审查判定附件输出，不由计划结构承载）
- **路由行** —— `Recommended task executor category: <route>`（字面前缀保留，取值限「Routing 枚举」）；使用专用子代理时改写 `subagent_type=<name>`（二者选一）；`execution_mode` 以同行括注（如 `(background)`），无法预定时写 `executor_judgment` 及原因
- **上下文胶囊** —— 相关文件清单、关键符号与行区间、规划期已验证结论、生成时的代码 revision 锚；落点已知且为单点修改的 task 可只写目标路径与符号名
- **验收条目** —— 逐条一行机械语法 `- <ID>：<二元条件> → 命令=<命令> 预期=<结果>`，`ID` 惯例 `T<n>-A<m>`；高风险 task 在条目行尾追加 `scope=<作用域文件清单>`；`ID` 从不复用，语义替换以新条目 `supersedes` 旧条目表达，执行期修订 append-only
- **写域** —— 完整计划 = 并发矩阵写域清单引用 + 增量禁止项一行；轻量计划 = 唯一可写产物完整清单

条件字段（存在对应情形时必填）：`环境 preflight`（运行时前置：服务启动、手动视觉巡查所需 env 文件的复制源 → 目标，逐个列出文件名与路径，不写机密值）、`reviewer 安排`（命中独立 reviewer 条件时）、`放弃/风险判据`（高风险完整计划可选）。

计划级通用约定在 `## Task 契约`（或轻量 `## 任务清单`）节首引言一次承载，不在逐 task 重复：通用禁止范围、终止状态（`blocked` 必须附断点胶囊：已验证结论、已排除路径、卡点描述）、`package_manager`（缺 → 软警告 `PKG_MANAGER_UNDECLARED`）、默认 `load_skills`。

**并行裁决规则**（并行性以并发矩阵 wave / cohort 归属为准，不靠字段单独声明）：

1. 并行/串行关系写入并发矩阵的 cohort 归属与硬前驱；并行规模受并发预算约束（口径 = 运行中写入 worker 与未验收积压之和，默认 ≤3、隔离充分至 4，`concurrency_budget` 为唯一覆盖入口）
2. 同一 lane 内两个 task 的写域存在重叠 → 不同 wave 或串行
3. 拆解产生的并行子任务写域必须显式互斥；不满足互斥的「并行」声明触发软警告 `PARALLEL_WRITESET_OVERLAP`

## 强制规则

1. **修复边界**：不得重新界定产品意图。仅修复结构、依赖真实性和验证真实性。未解决的硬关卡 → `REJECT`。高影响歧义 → 先问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`；不得猜测。
2. **确定性修复流程**：运行确定性两轮流程（先规范化，再硬关卡重评估）。保持输出可审计，带显式关卡码和固定章节。
3. **风险分层验证**：共享接口、跨模块集成、迁移、安全或高风险必须在验收条目与 `环境 preflight` 中显式覆盖；低风险本地任务保持最小充分验证（NON-CUMULATIVE）。
4. **分解与路由纪律**：任务粒度以 `omo-plan-structure` 原子性契约为准（最小内聚可验证结果；共享同一接口决策、不变量或验证面的工作保持同一 task）。修复后的路由行必须使用合法枚举值。高价路由（`unspecified-high` / `deep` / `ultrabrain` / `artistry` 及实际落高价模型的领域路由）必须在路由行或任务体附 `WHY_NOT_LOWER_COST`（点名低一档缺的具体能力）；命中风险特征（lifecycle 恰好一次动作 / 生产装配点语义变更 / 需先钉住错误被吞没的现状）时路由不得低于 `unspecified-high`。
5. **用户侧防漂移锚点**：用户可读摘要只位于轻量 `## 摘要` 或完整 `## 需求与目标` 节首（3-5 行）；不得另建摘要副本。
6. **执行命令格式**：输出的执行命令必须使用完整格式 `/ulw-execute <plan-name>`（plan-name 为计划文件名不含扩展名），且与 `# <plan-id> - Work Plan` 中的 plan-id 一致。不得输出无文件名的裸 `/ulw-execute`。
7. **平台到本地的收敛**：上游平台 runtime 术语（如 `sisyphus-junior`）可以作为 imported plan 的事实输入，但必须先被规范化成本地可执行的路由表达（路由行 + category 枚举），不能直接越过本地 schema 进入执行。

## Routing 枚举

路由行必须用 `category` **或** `subagent_type` 二选一声明执行者（与 `omo-adaptive-execution` 的 `task()` schema 一致）：

- **`category` 允许值**（实现类任务，判定边界以 `omo-adaptive-execution`「Category 选择」表为准）：`quick` / `unspecified-low` / `unspecified-high` / `deep` / `ultrabrain` / `visual-engineering`（复杂 UI/UX/样式/动画且交付物以写用户可见界面为主——双维判定，简单界面改动走 `unspecified-*`）/ `writing` / `artistry`
- **`subagent_type` 允许值**（只读专家）：`explore` / `librarian` / `metis` / `oracle` / `momus`
- **兜底**：任务路由需由 executor 现场决定时，路由行写 `executor_judgment` 及原因；该兜底不得滥用，每次使用必须给出理由

`load_skills` 仅允许使用项目有效 skill 名（见 skills 索引）或 `[]`；默认 `[omo-adaptive-execution]`（在计划级通用约定声明一次）。

导入计划附带的独立 Routing 表若无法表达 `subagent_type` 或兜底语义 → 触发软警告 `ROUTING_DISPATCHER_UNSUPPORTED`（不阻断，提示收敛为 task 路由行）。

## 高成本任务拆解分析（必执行章节）

### 触发对象

对每个路由为 `deep` / `unspecified-high` / `ultrabrain` / `artistry`（或实际落高价模型的领域路由）的**可执行叶节点** task，必须执行拆解分析。**不是凡贵必拆**——分析后输出明确结论：可拆 / 不可拆。

**拆解终止规则**（防递归）：

1. **只分析可执行叶节点** —— 已被拆解为父节点的 task 不再触发分析（父节点以标题行 `decomposed_into: [...]` 标记为非执行，见「拆解输出」）
2. **单次修复最多拆一层** —— 子任务即使仍是高成本路由，也不再进入第二轮拆解分析
3. **残留高成本叶节点的强制结论** —— 若拆解后某子任务仍是高价路由，必须在该子任务末尾追加 `[WHY_NOT_SPLIT]` 理由（如「再拆会破坏契约内聚」），不得留作隐式高成本

### 分析维度（逐项核对并记录结论）

沿以下维度判断是否存在 ≥2 个**可独立验收、可独立失败、可并行**的子结果：

| 维度 | 倾向可拆 | 倾向不拆 |
|---|---|---|
| 写域跨文件数 | ≥2 文件 / 跨包 | 单文件局部 |
| 估算改动行数 | ≥150 行 | <150 行 |
| 前置约束 | 含基线锁定 / characterization 需求 | 无前置 |
| 子结果独立性 | 多个独立可验收产出 | 共享同一不变量/接口决策 |
| 并行收益 | 子结果可并行执行显著缩短关键路径 | 强顺序依赖，拆开只增交接 |
| 风险隔离需求 | 子结果各自需要独立失败路径 | 同一验证面统一处理 |

### 拆解输出

**可拆** → 父任务改为**无执行语义的编排说明**（不参与执行），所有可执行子任务与中间校验点**重新分配顶层连续正整数 task ID**（标题行 `- [ ] N.` 体系），同时原子更新并发矩阵、硬前驱、路由与检查点中的引用：

- 每个子任务**完整复用 Task 契约字段契约**（标题行 / 路由行 / 胶囊 / 验收条目 / 写域）
- 子任务的路由通常**降级**（`deep` → `unspecified-high` 或 `quick`；`ultrabrain` → `deep` 或 `unspecified-high`），并在路由行附降级理由；**例外**：子任务命中风险特征（lifecycle 恰好一次动作 / 生产装配点语义变更 / 需先钉住错误被吞没的现状）时路由不得低于 `unspecified-high`，风险下限优先于拆解降级
- 子任务**继承父任务的 wave 与 workspace lane**（不新建 sub-Wave）；中间校验点表现为矩阵中的硬前驱边
- 子任务之间的并行/串行关系写入并发矩阵的 cohort 归属
- 子任务写域若声明可并行则必须**互斥**（同一文件路径不可被多个并行子任务写入）
- **父任务转为非执行编排节点**：
  - 标题行改为 `- [ ] N. <标题>（decomposed_into: [<子任务编号列表>]）`，保持 checkbox 不勾选（随全部子任务验收通过一并勾选）
  - **不进入**并发矩阵与路由（只列可执行叶节点）
  - 不触发任何 `task()` 调用

**不可拆** → 在该 task 节点末尾追加一行：

```
- **[WHY_NOT_SPLIT]**: <一行理由，如「单文件单接口决策，拆开增加交接不增加并行收益」>
```

### 拆解后中间校验点（强制）

凡被拆解为 ≥2 个子任务的高成本任务，必须在子任务序列中注入**中间校验点**。校验点本身是子任务（带完整字段契约），位于关键子任务之间，承担三类职责之一：

1. **基线锁定** —— 在改动前用 characterization test / git diff / wire 捕获锁定现有行为
2. **契约对齐** —— 在两个子任务共享接口边界时验证双方契约一致
3. **diff 闭合** —— 在并行子任务汇合前验证无漂移

中间校验点的路由通常为 `quick` 或 `unspecified-low`；验收条目必须是可运行的命令或可观测检查；其阻塞语义以**并发矩阵硬前驱**表达（后继子任务的硬前驱指向校验点，校验点未通过则后继不就绪），不使用独立 Pre-condition 字段。

如果分析后认为该任务无需中间校验点（极少见），必须显式写 `- **[NO_MIDPOINT_JUSTIFIED]**: <理由>`，否则触发 `MIDPOINT_MISSING`。

## Worktree 环境校验阶段（强制 Wave 0 / Task 0）

### 触发条件

满足以下任一条件时，修复流程**必须**确保计划含一个 Wave 0 / Task 0「环境就绪」节点：

- 计划含多 lane 条目（Workspaces 声明中）
- 任何 task 的写域归属 lane 不为 `main`
- Workspaces 中显式列出 ≥2 个 worktree 路径

单 lane（`mode: current` 或仅 main lane）计划不触发本阶段；但 **Workspaces 声明仍必须存在**（轻量为节尾一行，完整为顶层区块）并给出身份字段（见硬关卡 `WORKSPACE_IDENTITY_MISSING`）。

### Worktree 命名与落位

worktree 目录与分支按 `omo-plan-structure` 规则命名，落位遵循全局产物落位规则（worktree 优先建在仓库根 `.worktrees/` 下）：

- 单 lane 主 workspace 或多 lane integration workspace：路径 `.worktrees/<plan-name>--main`，分支 `work/<plan-name>/main`
- 实施 lane：路径 `.worktrees/<plan-name>--<task-key>`，分支 `work/<plan-name>/<task-key>`
- 计划与账本只在主工作区（主分支检出）的声明路径下，**任何 lane worktree 内不得另建计划或账本副本**

### Wave 0 / Task 0 字段契约

注入的 Task 0 必须满足 Task 契约字段契约，并额外覆盖：

- **标题行**：`- [ ] 0. 环境就绪（校验或创建全部 lane）`
- **胶囊**：本计划 Workspaces 表 + 各 lane 共同 base SHA
- **`环境 preflight`**：从计划级通用约定的 `package_manager` 派生各 lane 的依赖安装命令（声明于通用约定，缺 → 软警告 `PKG_MANAGER_UNDECLARED`）；含视觉巡查的 lane 同时列出 env 文件复制清单（源 → 目标）
- **验收条目**：
  - `git worktree list` 显示 Workspaces 表中的每个路径
  - 每个 worktree 内 `git rev-parse --abbrev-ref HEAD` 返回对应分支
  - 每个 worktree 内运行声明的 install 命令并 exit 0
  - 多 lane 计划中每个 lane 的 base SHA 一致
- **写域**：仅 `.worktrees/` 下的 lane 目录与环境文件；不覆盖已占用路径
- **验收条目的失败预期**（并入验收条目预期侧，不另立字段）：
  - worktree 路径已存在但 head 或 base 不匹配预期 → `BLOCKED_NEEDS_DECISION`，不强制切换
  - worktree 路径不存在 → 走「分支创建协议」
  - 路径被无关内容占用 → 报错并停止，不覆盖
  - 多 lane base SHA 不一致 → 硬关卡 `BASE_SHA_DIVERGENT`
- **路由行**：`Recommended task executor category: quick`
- 并发矩阵中该 task 的 workspace lane 列为 `main`（此任务负责创建/校验所有 lane；lane 归属只写矩阵，不写 task 正文）

### 分支创建协议

当 worktree 缺失时，按以下顺序操作，**不得跳步**：

1. **声明解析**：从 Workspaces 声明读取主分支（默认 `main`）与 `remote`（默认 `origin`）。未声明则用默认值并写入 Task 0 胶囊
2. **plan-id 安全化**：将 `<plan-name>` 转 lowercase；非 `[a-z0-9-]` 字符替换为 `-`；连续 `-` 合并为单个；长度截断至 50 字符。结果用于 worktree 目录与分支名
3. **主干状态校验**（在主仓执行）：
   - `git status --porcelain` 非空 → 软警告 `TRUNK_DIRTY`，让用户决定是否继续
   - `git rev-parse HEAD` ≠ `git rev-parse @{u}` → 软警告 `TRUNK_UNPUSHED`，让用户决定
4. **拉取远端引用**：`git fetch <remote>`；失败 → `BLOCKED_NEEDS_DECISION`
5. **解析并锁定 base SHA**：`git rev-parse <remote>/<trunk>` 写入 Task 0 胶囊，作为本计划所有 lane 的共同 base SHA
6. **幂等复用**：`git worktree list` 已含目标路径 → 校验其 head 指向对应分支且 base SHA 一致 → 通过；不匹配 → `BLOCKED_NEEDS_DECISION`，不强制覆盖
7. **创建分支与 worktree**：
   - `git branch work/<plan-name>/<lane-key> <remote>/<trunk>`
   - `git worktree add .worktrees/<plan-name>--<lane-key> work/<plan-name>/<lane-key>`
8. **依赖安装**：在新 worktree 内运行声明的 install 命令；exit ≠ 0 → `BLOCKED_NEEDS_DECISION`
9. **证据落盘**：将每个 lane 的（分支名、worktree 路径、base SHA、HEAD commit、install 命令、install exit code）写入执行账本（`.omo/ulw-execute/ledger.jsonl`），不写入计划正文

任何步骤失败 → `BLOCKED_NEEDS_DECISION` 并停止后续任务；不得猜测错误恢复路径。

### 跨 lane 集成 task（强制注入）

**触发条件**（同时满足）：

- Workspaces 含 ≥2 个非 main 写入 lane
- main lane 的某个 task 在并发矩阵中依赖其他 lane 任务（含仅集成关联）

**注入规则**：在首个「消费其他 lane 产物」的 main lane task **之前**注入一个 integration task（标题行带 `[integration]` 前缀，即 `omo-plan-structure` 要求的唯一 integration task）：

- **标题行**：`- [ ] N. [integration] 汇合全部非 main lane 产物到 main`（不实现业务逻辑）
- **胶囊**：Workspaces 声明 + 各非 main lane 的最终 HEAD commit
- **阻塞语义**：上游 lane 的所有 task 已完成且各自验收条目通过（并发矩阵硬前驱 + 仅集成关联表达）；解除形式为各上游 lane task 输出的 commit SHA
- **验收条目**：`git log <main>` 显示来自每个非 main lane 分支的 merge commit；main lane 运行检查点与集成区块声明的总门禁命令（如 `pnpm run verify` / `cargo test` / `go test ./...`）通过；merge 冲突或门禁失败 → `BLOCKED_NEEDS_DECISION`，不自动 resolve
- **路由行**：`Recommended task executor category: quick`
- 并发矩阵中该 task 的 workspace lane 列为 `main`

### F2.5 Lane merge parity（最终复核）

`F2.5` **不替代**上述「跨 lane 集成 task」，只做最终复核：

- 每个 non-main lane 的提交已在 main lane 上可见
- main lane 通过项目总门禁
- 各 lane 分支相对 base SHA 无意外丢失
- 多 lane 计划中各 lane base SHA 一致

`F2.5` **阻塞** Final Wave 与终态验收的其余具名 gate：F2.5 未通过则 Final Wave 与其余具名 gate 不得开始。

## 故障处理

1. 缺少输入 → 仅修复确定性格式问题后停止。
2. 高影响歧义 → 问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`。
3. 规范化后关卡反复失败 → `REJECT` 并附失败关卡集。
4. 不做叙述性信心声明；仅输出可执行的判定产物。

## 提示精简契约

本 command 是显式稳定计划的修复/验证权威来源。Prometheus 将修复语义委托至此。共享的分解、路由和提级原则来自 `omo-adaptive-execution`，计划结构契约来自 `omo-plan-structure`。不要在 Prometheus 中复制冗长的硬关卡规则块；将修复专用关卡保留在此。

## 第二轮修复模式（强制）

1. **第一轮——结构规范化**：
   - 补判级行并按 `omo-plan-structure` 判级；已含判级行但结构与级别不符的先纠级（轻量超判据 → `topology_remap` 升格完整结构）
   - 识别/补齐目标结构（轻量三节或完整五区块）；上游 9/11 章节导入计划按「上游导入映射」表收敛
   - 展平任何嵌套可执行项到 Task 契约列表（`- [ ] N.` 标题行体系，带 checkbox）
   - 迁移旧字段体系（`step_type` / `QA happy` / `QA failure` / `Evidence` / 独立 Routing 表 / `Pre-condition` → 验收条目、路由行、胶囊、写域、矩阵硬前驱）
   - 识别 Workspaces 声明 → 注入或补齐 Wave 0 / Task 0
   - 识别高成本任务 → 执行拆解分析 → 注入子任务 + 中间校验点
   - 统一路由行枚举与 `load_skills` 名；补齐计划级通用约定
   - 统一并发矩阵引用的 task ID；补齐 cohort 归属、硬前驱与 wave 一致性

2. **第二轮——关卡重评估**：在规范化后的计划上重跑所有硬关卡；失败时输出带关卡码的 `REJECT`；仅当硬关卡失败数 = 0 时输出 `PASS`。

任何结构变更后不得跳过第二轮。

## 必需检查

### 计划结构层

1. **判级与结构完整** —— 首行判级行存在且结构与级别相符（轻量三节 / 完整五区块，区块名与顺序按 `omo-plan-structure`）。关卡：`PLAN_LEVEL_MISSING`。
2. **需求摘要完整** —— 摘要/需求与目标节首含 3-5 行用户可读摘要；逐条需求附可追溯来源并标注 `core` / `preference`；未确认缺口保持未决。关卡：`TLDR_MISSING`。
3. **需求边界完整** —— 硬约束 / 非目标（`不做的事：`）/ 未决项就位；未决项显式标注 `BLOCKED on <决策项>`。关卡：`SCOPE_MISSING`。
4. **验收命令可执行** —— 终态验收/检查点与集成中的验收条目为机械语法（命令= / 预期=），非模糊叙述。关卡：`VERIFICATION_STRATEGY_NOT_EXECUTABLE`。
5. **Workspaces 身份一致** —— `vcs` / `mode` 标注齐全；完整计划每 lane 条目含 `name` / `path` / `branch` 三项身份字段，每个 lane 在 Task 契约写域中至少出现一次，Task 写域引用的 lane 必须在 Workspaces 声明；轻量计划节尾声明含 `vcs`、`mode` 及路径要素；`mode: current` 含 `authorization_source`。关卡：`WORKSPACE_TABLE_INCOMPLETE`。
6. **路由枚举合法 + 与可执行 task 一一对应** —— 每个可执行叶节点恰有一条路由行且取值合法；`load_skills` 只允许有效 skill 名或 `[]`；父编排节点（`decomposed_into`）不再保留路由行。关卡：`ROUTING_SCHEMA_INVALID`。
7. **Final Wave 完整** —— 终态验收/检查点与集成含具名 gate 与 `- [ ] F1. 终态：<命令>` 行；完整计划含 Final Wave 节点；多 lane 计划额外含 Lane merge parity 关卡。关卡：`FINAL_VERIFICATION_MISSING`。
8. **检查点断言可观测** —— 每条断言可独立验证（命令、grep、文件存在、exit code 等），拒绝纯叙述，并标注证据强度（集成实测 / 切片单测拼装 / 类型检查）。关卡：`SUCCESS_CRITERIA_VAGUE`。

### Task 契约层

9. **Task 字段完整** —— 每个实施 task 含 5 字段（标题行 / 路由行 / 上下文胶囊 / 验收条目 / 写域）；计划级通用约定就位（含 `package_manager`）。关卡：`TODO_FIELD_MISSING`。
10. **验收条目二元可执行** —— 验收条目含命令 + 可观测预期；`ID` 唯一且不复用。关卡：`QA_NOT_EXECUTABLE`。
11. **并发矩阵闭合** —— 矩阵引用的所有 task ID 在 Task 契约中存在且恰好出现一次；硬前驱可解析、依赖无环（回路直接判 `DEPENDENCY_GRAPH_OPEN`）；wave 节与全局矩阵一致；并行规模不超并发预算上限（运行中写入 worker + 未验收积压口径）。关卡：`DEPENDENCY_GRAPH_OPEN`。
12. **范围保真** —— task 写域与硬约束/非目标不冲突；任何 task 不得触碰非目标或未决项。关卡：`SCOPE_LEAK`。
13. **计划/账本分离** —— 计划正文不含动态状态（执行证据、尝试次数、会话记录；checkbox 勾选投影 `- [x]` 除外）；账本统一为 `.omo/ulw-execute/ledger.jsonl` 且不在 lane worktree 内出现副本。关卡：`PLAN_DYNAMIC_STATE_LEAKED`。

### 高成本任务与 Worktree 层

14. **拆解分析覆盖** —— 对每个高价路由任务，必须输出「可拆 / 不可拆」结论；可拆必须注入子任务 + 中间校验点；不可拆必须含 `[WHY_NOT_SPLIT]`。关卡：`DECOMPOSITION_ANALYSIS_MISSING`。
15. **中间校验点闭合** —— 被拆解为 ≥2 子任务的高成本任务，子任务序列必须含至少一个中间校验点（基线锁定 / 契约对齐 / diff 闭合），阻塞语义为矩阵硬前驱；若无则必须含 `[NO_MIDPOINT_JUSTIFIED]`。关卡：`MIDPOINT_MISSING`。
16. **子任务降级合理** —— 子任务路由不得高于父任务；降级必须附理由。例外：命中风险特征时路由不得低于 `unspecified-high`，风险下限优先于拆解降级。关卡：`SUBTASK_CATEGORY_INFLATED`。
17. **Worktree 前置就绪** —— 触发 worktree 阶段的计划必须含 Wave 0 / Task 0；Task 0 验收条目覆盖 `git worktree list` / 分支匹配 / install 命令三项；多 lane 计划校验各 lane base SHA 一致；worktree 目录与分支命名符合规则且落位 `.worktrees/`。关卡：`WORKTREE_PREFLIGHT_MISSING`。
18. **env 复制声明** —— 含启动服务进行手动视觉巡查或人工运行时验证的 task，`环境 preflight` 必须列出从主工作区复制的 env 文件清单（文件名、源路径与目标路径，不写机密值）。关卡：`ENV_REPLICATION_MISSING`。

### 治理层

19. **平台术语收敛** —— imported plan 中的 `sisyphus-junior` 等上游 runtime 标签必须先规范化为路由行 + category / subagent_type 枚举；不得直接进入执行。关卡：`PLATFORM_TERM_LEAKED`。
20. **执行命令格式** —— 输出含 `/ulw-execute <plan-name>`（plan-name 为计划文件名不含扩展名）；plan-name 必须与 `# <plan-id> - Work Plan` 中的 plan-id 一致。关卡：`ULW_EXECUTE_COMMAND_MALFORMED`。

### 上游对齐层（按已加载的上游契约做缺口识别）

21. **顶层 workspaces 身份完整** —— 完整计划 `## Workspaces` 顶层区块存在；每个 lane 条目含 `name` / `path` / `branch` 三项身份字段；单 lane 计划也必须给全身份（不简化为 `single-lane` 字符串）。关卡：`WORKSPACE_IDENTITY_MISSING`。修复方向：把上游 `### Workspaces`（嵌于 `## Execution strategy` 下）的身份字段提升为顶层 Workspaces 声明。
22. **Handoff 信息就位** —— handoff 五要素（计划路径 / 版本 / 当前状态 / 未决事项 / 执行入口）只在交付消息中提供，**不写入计划文件正文**；执行入口派生自 plan-id。关卡：`HANDOFF_MISSING`。
23. **task ID 体系一致** —— 全文 task ID 统一为标题行 `- [ ] N.` 顶层连续正整数体系；并发矩阵引用必须匹配该体系；不允许小数后缀。关卡：`TODO_ID_SCHEME_INCONSISTENT`（软警告）。
24. **`mode: worktree` 声明对齐** —— 若计划含多 lane，Workspaces 声明 `mode: worktree`；单 lane 可选。关卡：`WORKTREE_MODE_UNDECLARED`（软警告）。
25. **路由派发表达** —— 导入计划的独立 Routing 表收敛为 task 路由行；无法表达 `subagent_type` 或 `executor_judgment` 兜底语义时触发软警告 `ROUTING_DISPATCHER_UNSUPPORTED`。

## 修复顺序

1. 判级行与结构骨架：轻量三节或完整五区块；上游导入按映射表收敛
2. 摘要 / 需求与目标：节首摘要 + 逐条溯源 + `core` / `preference` + 硬约束/非目标/未决项
3. Workspaces 声明：`vcs` / `mode` / 主分支与存放路径 / 每 lane 身份字段
4. Task 契约：5 字段迁移补齐 + 计划级通用约定
5. 并发矩阵：cohort 归属、硬前驱、仅集成关联、写域互斥；wave 一致性与 `concurrency_budget`
6. **Worktree 触发判定** → 若触发，注入或补齐 Wave 0 / Task 0（含分支创建协议要素）
7. **高成本任务拆解分析** → 可拆则注入子任务 + 中间校验点；不可拆则附 `[WHY_NOT_SPLIT]`
8. 检查点与集成 / 终态验收：检查点声明、证据强度、Final Wave（F1 行）、多 lane 强化 Lane merge parity、提交意图收敛
9. 第二轮硬关卡重评估

## 输出要求

产出：

- `Verdict`：`PASS` 或 `REJECT`
- `Gate Summary`：按关卡码统计的失败硬关卡数
- `Hard Gates`：关卡码 + 失败章节列表
- `Warnings`：非阻塞质量问题（含软警告码）
- `Fixed Sections`：已修改的确切章节
- `Needs Decision`：需要人工产品/契约决策的条目
- `需求确认`：摘要节首 3-5 行用户可读摘要 + 溯源 + `core` / `preference` 标注就位；硬约束 / 非目标 / 未决项就位且未决项标 BLOCKED
- `Worktree Preflight`：是否触发、注入了哪些 lane 的 Wave 0 / Task 0、主干分支名、哪些 lane 已就绪 / 待创建
- `Task Decomposition Report`：列出每个被分析的高成本任务——结论（可拆/不可拆）、拆出的子任务编号、注入的中间校验点编号、降级的路由，或不拆的 `[WHY_NOT_SPLIT]` 理由
- `并发矩阵`：cohort 归属、硬前驱、依赖图与 `concurrency_budget` 摘要
- `Routing Audit`：路由行枚举合法性、高价任务 `WHY_NOT_LOWER_COST` 是否就位、`executor_judgment` 兜底是否滥用
- `Upstream Contract Self-Check`：4 个上游 skill 的加载状态（已加载 / `UPSTREAM_SOURCE_UNAVAILABLE`）+ 各自对齐检查的缺口摘要
- `Handoff Explanation`：六元素——`What this plan drives`（计划驱动什么）/ `End state`（最终态）/ `Shape`（N impl + F final-verifier）/ `Added beyond the request`（在用户请求外补强的内容）/ `Verification`（Final Wave + 上游对齐检查）/ `Execution handoff`（`/ulw-execute <plan-name>`）；**只在交付消息中提供，不进计划正文**
- `Execution Command`：完整 `/ulw-execute <plan-name>` 命令

任何 `BLOCKED_NEEDS_DECISION` 条目仍开放 → 判定必须为 `REJECT`。

## 硬关卡 vs 软警告

### 硬关卡（必须为零才能 PASS）

`PLAN_LEVEL_MISSING` · `TLDR_MISSING` · `SCOPE_MISSING` · `VERIFICATION_STRATEGY_NOT_EXECUTABLE` · `WORKSPACE_TABLE_INCOMPLETE` · `ROUTING_SCHEMA_INVALID` · `FINAL_VERIFICATION_MISSING` · `SUCCESS_CRITERIA_VAGUE` · `TODO_FIELD_MISSING` · `QA_NOT_EXECUTABLE` · `DEPENDENCY_GRAPH_OPEN` · `SCOPE_LEAK` · `PLAN_DYNAMIC_STATE_LEAKED` · `DECOMPOSITION_ANALYSIS_MISSING` · `MIDPOINT_MISSING` · `SUBTASK_CATEGORY_INFLATED` · `WORKTREE_PREFLIGHT_MISSING` · `ENV_REPLICATION_MISSING` · `BASE_SHA_DIVERGENT` · `PLATFORM_TERM_LEAKED` · `ULW_EXECUTE_COMMAND_MALFORMED` · `WORKSPACE_IDENTITY_MISSING` · `HANDOFF_MISSING`

### 软警告

`ROUTING_UNDERKILL` · `ROUTING_OVERKILL` · `TASK_MAY_UNDER_DECOMPOSE` · `VERIFICATION_REDUNDANT` · `MIDPOINT_EXCESSIVE` · `TLDR_DRIFT_AFTER_SPLIT` · `PKG_MANAGER_UNDECLARED` · `TRUNK_DIRTY` · `TRUNK_UNPUSHED` · `PARALLEL_WRITESET_OVERLAP` · `TODO_ID_SCHEME_INCONSISTENT` · `WORKTREE_MODE_UNDECLARED` · `ROUTING_DISPATCHER_UNSUPPORTED`

`TLDR_DRIFT_AFTER_SPLIT` 触发条件：拆解改变了摘要节首中**投入量级**（如 1 个 ultrabrain 拆为 3 个 unspecified-low，关键人力分布变化）、**风险等级**或**关键路径**（如拆解引入新的并行路径），但摘要未相应更新。

`ROUTING_DISPATCHER_UNSUPPORTED` 触发条件：导入计划的独立 Routing 表无法表达 `subagent_type` 或 `executor_judgment` 兜底语义；不阻断，提示收敛为 task 路由行。

`ROUTING_UNDERKILL` 触发条件：路由低于任务性质所需能力（如共享接口任务标 `quick`），或绕过风险特征路由下限条款。

`ROUTING_OVERKILL` 触发条件：路由高于任务性质所需且无有效 `WHY_NOT_LOWER_COST` 举证。

`TASK_MAY_UNDER_DECOMPOSE` 触发条件：task 写域跨多个 owner / failure family，但既未拆解也无完整「非原子」举证三要素。

`VERIFICATION_REDUNDANT` 触发条件：同一 revision 内重复等价全量门禁，或叠加非必要验证项（违反 NON-CUMULATIVE）。

`MIDPOINT_EXCESSIVE` 触发条件：注入的中间校验点数量超出关键路径所需，存在无对应职责的校验点。

## 提级流程

1. 自动修复确定性文本问题（章节骨架、字段补齐、枚举修正、依赖边显式化）。
2. 高影响的模糊产品决策 → 问 1-3 个定向问题；若仍不确定 → `BLOCKED_NEEDS_DECISION`。示例：选择 `/api/` vs `/api/v1/`、是否将某功能纳入或排除出 Scope IN、Deferred 项的 BLOCKED 决策无法定位。
3. 绝不猜测业务意图来「强制通过」。

## 与 Atlas/Prometheus 集成

- 权威性结构修复的落地路径，用于权威性计划编辑。在执行前和审查驱动的缺陷发现后使用。
- 定义结构有效、修复完成的计划应是什么样；不负责运行时执行顺序、证据纪律或提交时机。
- Atlas 仅在动态执行连续两轮结构重排仍无法闭合、用户要求冻结计划，或需要长期计划资产时使用此规范。Prometheus 提供紧凑的路由意图；本 command 展开并执行具体修复关卡。
- `metis` 可能暴露遗漏，`oracle` 可能产出修订简报；局部依赖变化由 `omo-adaptive-execution` 在不改变用户目标的前提下重排，权威计划结构变更再通过本 command 落地；计划结构标准冲突以 `omo-plan-structure` 裁决。
- 输出在当前审查消息中行内发出；不需要单独的产物文件。
