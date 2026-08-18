# /serena-archive Command

**Description**: 将 .serena/memories/ 中**已实现的事实型**记忆落库到项目文档（AGENTS.md / docs/ / docs/shipped/），然后整体删除 .serena 目录。严格过滤计划、规划、spec、design 等未实现内容——只迁移反映代码现状的事实。无参数自动执行全流程；`--dry-run` 只评估不写入；`--keep` 落库后保留 .serena（不删除）。

**Agent**: build

**Scope**: global

---

## Command Instructions

# Serena 记忆归档命令

**核心契约**：只把 .serena 中**已实现的事实**转化为项目文档，**严格丢弃**计划/spec/规划/设计类内容。落库完成且用户确认后删除整个 .serena 目录。

**用户输入**：``

## 参数

| `` | 行为 |
|---|---|
| 空 | 完整流程：盘点 → 过滤 → 核实 → 转化 → 删除 |
| `--dry-run` | 只做盘点+过滤+核实+输出建议，不写入、不删除 |
| `--keep` | 落库后保留 .serena（仅转化，不删除） |
| `--verify-only` | 跳过转化，只核实 .serena 记忆与代码现状的漂移 |

未识别参数按空处理。

---

## 执行流程

### 阶段 0：前置检查

1. **项目根**：`git rev-parse --show-toplevel`（或回退到 cwd）。
2. **.serena 存在性**：`Test-Path .serena/memories/`。不存在 → 报告"无可归档内容"并退出。
3. **shipped root 发现**：用 `/shipped` 命令同样的发现逻辑（读 AGENTS.md → README.md → 候选目录 → 默认 `docs/shipped/`）。
4. **AGENTS.md 存在性**：不存在则提示用户先建立 AGENTS.md（命令不擅自创建根 AGENTS.md）。

### 阶段 1：盘点 + 过滤

枚举 `.serena/memories/**/*.md`（不含 cache、project.yml 等工具产物），逐个评估：

#### 1.1 内容分类（每份记忆判定一类）

| 类别 | 判据 | 处理 |
|---|---|---|
| **已实现事实** | 含 `[shipped]` 标记、明确文件路径/API/字段/行为描述，且 grep 代码能验证 | **转化** |
| **代码现状契约** | 描述已实现函数/协议/接口的精确行为 | **转化** |
| **构建/工具事实** | terser import / .gitignore 规则 / devDeps 清理记录等工程事实 | **转化** |
| **工作流约束** | 影响后续执行的硬约束（lint 规则、编码风格） | **转化** |
| **技术债/待办** | 标 `[tech-debt]` 或明确"未实现"的修复点 | **核实后转化**到 tech-debt 文档 |
| **冗余** | 内容已被 AGENTS.md / README.md / docs/ 完整覆盖 | **丢弃** |
| **纯规划/spec** | 含 `[plan]` / `[designed]` / "未来" / "规划" / "spec" / "TODO" 关键词且无对应代码 | **丢弃** |
| **过程记录** | 一次性排障日志、临时假设、会话笔记 | **丢弃** |
| **工具产物** | cache/* / project.yml / project.local.yml | **丢弃** |

#### 1.2 关键过滤原则

- **不落库规划性内容**：含 `[plan]` / `[designed]` / "未来计划" / "待实现" / "spec" / "design proposal" 关键词的内容一律丢弃，除非已有代码实现。
- **不复制规格漂移**：记忆描述与代码现状不符时，以代码为准重写或丢弃——绝不原样复制记忆。
- **不迁移"应有"的内容**：只迁移"现在确实是"的事实。理想态描述 → 丢弃。
- **冗余优先丢弃**：如果项目文档已有更详细版本，不复制旧摘要。

#### 1.3 grep 核实（防漂移）

对每条要转化的记忆，grep 引用到的文件/函数/字段，确认：

- 文件路径存在
- 函数签名匹配
- 行为与记忆描述一致
- 标记类型正确（`[shipped]` 还是 `[tech-debt]`）

任何一项不符 → 按代码现状修正后转化，或丢弃。

### 阶段 2：落库写入

按内容类型路由到对应文档：

| 内容类型 | 目标位置 | 写入方式 |
|---|---|---|
| 项目级规格（协议契约、API 契约） | `docs/<topic>-contract.md`（新建或追加） | 整段规格化重写，按代码现状 |
| shipped 功能单元事实 | `<shipped-root>/<unit>.md` | 追加 [shipped] 标记条目 |
| 构建/工具/工作流约束 | `AGENTS.md` 对应节 | 在现有节追加，不新建节 |
| 技术债/测试待办 | `docs/tech-debt-cleanup.md` | 新增"测试质量改进待办"或"补充清理"节 |
| 编码规范 | 项目已有 AGENTS.md Working Rules 节 | 仅追加独特规则，丢弃已被覆盖的 |

#### 写入规则

- **先读后写**：每个目标文件先 Read 现有内容，在其结构内追加，不覆盖。
- **外科手术式**：只写入独特事实，已被现有文档覆盖的不重复。
- **代码现状优先**：转化内容必须与 grep 核实的代码现状一致，规格漂移时按代码修正。
- **不创建文档树**：除非确有必要（如 webrtc-sdp-contract 这种独立规格），优先追加到现有文档。

### 阶段 3：报告 + 用户确认

输出报告：

```
## .serena 归档报告

### 已转化（N 条）
| 来源记忆 | 目标文档 | 摘要 |
|---|---|---|
| memories/project/build-packaging.md | AGENTS.md Build Pitfalls | terser default import + pnpm-lock + devDeps 清理 |
| ...

### 丢弃（N 条）
| 来源记忆 | 原因 |
|---|---|
| memories/project/overview.md | 冗余，已被 README + AGENTS.md 覆盖 |
| memories/contract/webrtc-planned.md | 纯规划，无代码实现 |
| ...

### 工具产物（不转化）
- cache/typescript/*.pkl (~26MB LSP 缓存)
- project.yml / project.local.yml (Serena 配置)

### 待删除的 .serena 文件
- [列出全部 .serena/**/*]

### .gitignore 清理
- L40: 移除 `.serena/`
- L67: 移除 `.serena`（重复条目）
```

`--dry-run` 模式到此结束，不写入。

正常模式：写入完成后，**询问用户确认**："确认全部有用记忆已落库？将删除 .serena 整体。"

### 阶段 4：删除 .serena

用户确认后：

1. `Remove-Item -Recurse -Force .serena`
2. 清理根 `.gitignore`：移除所有 `.serena` 相关规则（grep `.serena` 找全部出现位置）
3. **不修改**项目根 `.gitignore` 之外的其他配置
4. 报告：删除的文件数、清理的 .gitignore 行

`--keep` 模式跳过此阶段。

---

## 输出格式

最终报告必须包含：

1. **盘点总数**：扫描的记忆文件数
2. **分类统计**：转化 / 丢弃 / 工具产物 各类文件数
3. **转化清单**：每条转化的来源→目标映射
4. **丢弃清单**：每条丢弃的原因（冗余 / 规划 / 过程记录 / 已被覆盖）
5. **核实漂移清单**：发现记忆与代码不符的位置，及处理方式（按代码修正 / 丢弃）
6. **删除清单**：实际删除的 .serena 文件数 + .gitignore 改动
7. **后续建议**：用户需手动核实的点（如规格漂移争议项）

---

## 安全约束

- **绝不**删除项目根之外的任何文件。
- **绝不**修改 AGENTS.md 已有内容（只追加）。
- **绝不**覆盖 docs/ 已有文档（只追加/新建独立规格）。
- **绝不**复制未 grep 核实的内容到文档（防漂移）。
- **删除前必须用户确认**（`--dry-run` 和 `--keep` 是默认安全选项）。
- **遇到规格漂移**：按代码现状重写或丢弃，不原样复制记忆描述。
- **不擅自创建 docs/planned/**：纯规划内容直接丢弃，不归档为 `[designed]`（避免规划膨胀）。

---

## 通用规则

- **工具优先**：用 PowerShell（Windows）/ bash（Unix）操作文件系统；用 grep/Read 评估记忆内容。
- **跨平台**：删除命令根据 OS 选择（Windows: `Remove-Item`，Unix: `rm -rf`）。
- **原子更新**：每次落库写入应可独立验证；用户确认后再删除 .serena。
- **报告精确**：每个文件、每条记忆、每个改动点都要在报告中列出，便于用户审阅。
- **不编造**：无法确认的记忆标"无法核实"并询问用户，不猜测后写入。

---

## 模板

### 阶段 1 评估表（命令内部使用）

| 记忆文件 | 大小 | 关键词 | 内容类型 | 处理 | 目标 |
|---|---|---|---|---|---|
| memories/project/overview.md | 706B | 已实现事实 | 冗余 | 丢弃 | — |
| memories/project/build-packaging.md | 498B | terser, pnpm-lock | 构建事实 | 转化 | AGENTS.md Build Pitfalls |
| memories/contract/xyz-plan.md | 1.2KB | `[plan]`, 未来 | 纯规划 | 丢弃 | — |
| ... | | | | | |

### 阶段 3 报告模板

```markdown
## .serena 归档报告（dry-run / 已执行）

**项目根**: `<root>`
**shipped root**: `<shipped-root>`
**扫描时间**: <timestamp>

### 1. 盘点
- 记忆文件总数：N
- 工具产物（不入库）：cache/* + project.yml 等
- 评估记忆数：M

### 2. 分类结果

| 处理 | 数量 | 说明 |
|---|---|---|
| 转化 | X | 已实现事实/契约/约束 |
| 丢弃（冗余） | Y1 | 已被现有文档覆盖 |
| 丢弃（规划） | Y2 | 纯规划/spec/未实现 |
| 丢弃（过程记录） | Y3 | 一次性排障日志 |
| 丢弃（工具产物） | Y4 | cache/yml 配置 |

### 3. 转化详情
[逐条来源→目标→摘要]

### 4. 漂移核实
[记忆与代码不符的位置 + 处理]

### 5. 删除清单
[实际/将删除的 .serena 文件 + .gitignore 改动]
```

---

## 与其他命令的关系

- **`/shipped`**：本命令转化时优先追加到 `<shipped-root>/<unit>.md`，遵循 shipped 清单工作流。
- **`/cleanup-sisyphus`**：本命令专注 .serena 归档；其他工作区清理用 cleanup-sisyphus。
- **`/doc-sync`**：本命令一次性归档；doc-sync 用于持续同步。
