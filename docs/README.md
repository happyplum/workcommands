# oh-my-opencode 命令集

oh-my-opencode 的文件式 OpenCode commands 仓库。这里存放显式手动触发的治理型命令；命令本身是主执行入口，负责完整工作流、完成标准与最终报告。

## 安装

将本目录放置于 OpenCode 配置根目录下的 `commands/`（例如 `~/.config/opencode/commands/`）。文件名会成为对应的斜杠命令：

- `repair-plan.md` → `/repair-plan`
- `restructure-memory.md` → `/restructure-memory`
- `cleanup-sisyphus.md` → `/cleanup-sisyphus`

## 设计定位

- `commands/` 是**手动执行面**：负责参数解释、操作协议、成功标准与最终报告。
- `/repair-plan`、`/restructure-memory` 与 `/cleanup-sisyphus` 现在都是**自包含 command**：命令文件自身持有完整执行协议，不再依赖匹配 skill 作为规则载体。
- command 文档、目录结构、入口命名与命令级行为说明继续由本仓库维护。

## 命令列表

| 命令 | 作用 | 默认目标 |
|------|------|----------|
| `/repair-plan` | 修复或规范化 reviewed / imported execution plan，使其进入本地 execution-ready 状态 | 从参数、当前会话或当前工作区推断计划目标 |
| `/restructure-memory` | 对项目持久化记忆做结构重组，保留 durable knowledge | 从参数、当前任务或可复用 memory inventory 推断范围 |
| `/cleanup-sisyphus` | 审计并清理 `.omo` / `.sisyphus` 工作区与 `docs/` 中的 Agent 执行产物，同时保留必要的长期知识和用户文档 | 当前工作区 `.omo` / `.sisyphus`，以及存在时的 `docs/` |

## 参考索引

- `/cleanup-sisyphus` 的 `docs/` 清理以命令文件中的 `Docs 保留标准` 为准；参考入口只记录索引。当前保留判定围绕目的、受众、时效、唯一性、行动价值和责任边界六项标准。

## 边界

- `/repair-plan`、`/restructure-memory` 与 `/cleanup-sisyphus` 的规则细节均以各自 command 文件为准，不再把同名 skill 作为主事实来源。
- 不把 command 降级成“仅调用某个 skill”的 wrapper；命令自身必须足够自洽，能直接驱动执行到完成。
- 若命令行为变化影响根配置事实或长期治理规则，需同步根 `README.md` 与相关 durable memory。
