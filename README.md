<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="agent-group 通过 Orca worker 或可见可续作的 Codex App 工作树任务实施，由 coordinator 调查、监督、审查、集成并管理归档恢复">
</p>

**agent-group** 提供两种中文多代理交付技能：用 [`orca-dispatch`](skills/orca-dispatch/SKILL.md) 协调 [Orca](https://github.com/stablyai/orca) worker，或用 [`codex-orchestration`](skills/codex-orchestration/SKILL.md) 协调用户明确要求的新 Codex App 工作树任务。每个 App 任务可在界面中看到、进入并继续；实施使用独立分支与工作树，由 coordinator 监督、审查、串行集成并管理归档恢复。

## 选择适合你的技能

| | [Orca 版](skills/orca-dispatch/SKILL.md) | [Codex 版](skills/codex-orchestration/SKILL.md) |
|---|---|---|
| 宿主 | Claude Code / Codex | Codex App |
| 执行器 | Orca worker | App 工作树任务 |
| 恢复坐标 | Run + Git | threadId + 账本 |

两版都在独立分支与工作树实施。Orca 负责其 worker 生命周期；Codex 版创建绑定工作树的独立 App 聊天，并使用同一 threadId 处理提问、修复与继续。

## 为什么使用 coordinator

多个 agent 并行工作时，常见问题是任务边界互相冲突、进度难以追踪，以及把 worker 自述的“测试通过”当成最终验收。两个技能都要求 coordinator 先理解项目和需求，再划分可验证任务，最后亲自复核交付证据。

- 修 bug 时先复现并定位原因；做功能时先读现有需求，只澄清会改变实现的决策。
- 先分析完整切片、真实依赖、文件冲突和共享资源，展示哪些任务能同时开始、哪些必须等待，再确定 worker 数量。
- 每个实施任务使用独立工作树；同一共享合同仍需统一归属，端口和数据库另行隔离。
- Codex 版将明确确认的测试切入点交给 worker，按 Matt 的逐条红绿循环实现，并独立审查规范符合度与需求符合度。
- coordinator 重跑适用检查，按固定候选完成双轴审查，集成验证通过后才启动依赖波次。
- 合并、推送、部署和生产操作仍受项目规则与用户授权约束。

## 从问题到集成

两个技能共享以下交付链，使用各自的任务执行器。Codex 的创建、可见性验收、监督与续作见 [App 任务运行说明](skills/codex-orchestration/references/worker-runtime.md)；收尾见 [工作树生命周期](skills/codex-orchestration/references/worktree-lifecycle.md)。

<p align="center">
  <img src="./assets/readme/workflow.svg" width="100%" alt="先调查与计划，再派发独立工作树中的 worker，最后固定候选进行双轴审查、串行集成与验证">
</p>

## 快速开始

### 使用 Orca

在已配置 Orca 的 Claude Code 或 Codex 项目中调用：

```text
/orca-dispatch 修复素材生成点击后一直转圈的问题
/orca-dispatch 实现 docs/PRD.md 第 3 节「批量生成」 先确认
/orca-dispatch 小说素材按免费章节生成文案和图片 先聊清楚
/orca-dispatch 继续 run_3f9c
```

Codex 调用 Orca 技能时使用 `$orca-dispatch`。常用参数：

| 参数 | 作用 |
|---|---|
| `先聊清楚` | 需求尚未成熟时继续澄清；必要时制作一次性原型并整理 PRD |
| `先确认` | 汇报计划后等待确认，再开始派发 |
| `推送` | 集成验证通过后允许推送；不会自动部署 |
| `worker 用 opus\|sonnet\|fable\|codex` | 指定本次 worker 使用的模型或 agent |
| `继续 <run_id>` | 按 Orca 与 Git 实际状态恢复中断的编排 |

### 使用可见的 Codex App 工作树任务

在 Codex App 的正确 Git 项目中，明确要求创建任务并说明协调范围：

```text
$codex-orchestration 新建 App 独立工作树任务，实现 docs/PRD.md 的批量生成。
允许本次持续协调；验收并集成通过后归档。先确认

$codex-orchestration 继续 <账本路径>
```

coordinator 通过 App 任务工具创建 worktree 环境中的独立聊天。新聊天先做只读初始化；App 中可见、真实 threadId 可读取、工作树与批准基线核对通过后，才发送实施合同。提问与修复继续原聊天，不为每次修复另建任务。原生子代理只承担只读调查和 Matt 双轴审查。

### 并发实施与顺序依赖

主协调者先给出任务表：每项交付什么、依赖谁的具体结果、负责哪些文件、共用哪些运行资源、属于哪个并发组。存在两项以上可独立验收、已授权且可隔离的就绪任务时，应并发启动多个 worker；不能默认由一个 worker 按清单做完所有事。真实不可拆、前置未完成或确有权限/资源上限时可以串行，但要给出具体理由。

调度按“整批创建 → 分别核对初始化 → 派发所有可实施任务 → 等待活跃集合”推进，不在第一个任务完成后才创建第二个。共享数据库重建和最终集成等冲突步骤串行，其他独立实施继续并发。计划并发、聊天数量与实际同时实施分别取证。

支持 `先聊清楚`、`先确认`、`推送` 和 `worker 用 <实际模型 ID>`。未指定模型时保留 App 任务的默认设置。创建聊天、协调消息和归档分别遵守直接人类授权及宿主限制；普通技能调用不自动授予这些权限。缺少正确的已保存项目或 App 工具时报告受阻，不改用 CLI 伪装 App 任务。

### 工作树从创建到恢复

- **创建与继续**：等待真实任务 ID 和初始化验收，在同一聊天、分支与工作树实施、修复；pending 创建结果先核对，避免重复 writer。
- **验收与保留**：固定候选完成双轴审查、串行集成与验证；运行中、未合并、dirty、用户接管或需保留的任务不进入自动清理。
- **归档与恢复**：先把证据保存到工作树外，再按授权归档聊天，分别核实托管工作树和分支结果。恢复使用原任务和 App 的快照/恢复机制，重新核对路径、HEAD 与审查基线。聊天归档不等于已证实所有资源回收。

升级前已派发的 CLI worker 保留原合同、会话和工作树，继续原有监督与收尾；新版不会给它们补建聊天或迁移执行器。账本提供恢复坐标，不承诺 coordinator 关闭后仍自动监督。

## 安装

克隆仓库后，将所需技能链接到 agent 读取的技能目录：

```bash
git clone https://github.com/sys98/agent-group.git
cd agent-group

# Claude Code + Orca
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/orca-dispatch" ~/.claude/skills/orca-dispatch

# Codex with Orca
mkdir -p ~/.agents/skills
ln -s "$PWD/skills/orca-dispatch" ~/.agents/skills/orca-dispatch

# Visible Codex App worktree tasks
mkdir -p ~/.codex/skills
ln -s "$PWD/skills/codex-orchestration" ~/.codex/skills/codex-orchestration

# Alternatively, use ~/.agents/skills if your host reads that directory.
# Install codex-orchestration in only one discovery directory.
```

如目标链接已存在，请先按本机技能管理方式处理，避免覆盖已有安装。

## 依赖与边界

- **Orca 版本**需要 Orca 与其 `orchestration`、`orca-cli` 技能。
- **Codex 版本**需要匹配的已保存 Git 项目、App 的任务创建/读取/等待/消息与生命周期工具，以及用于 Matt 双轴审查的两个独立只读上下文。任务工具若不允许这种用户请求的独立聊天，必须遵守限制。
- 按阶段读取 [Matt Pocock 技能](https://github.com/mattpocock/skills)：`grilling`、`prototype`、`diagnosing-bugs`、`to-tickets`、`tdd` 与 `code-review`；公开接口尚未确定时还需 `codebase-design`。Codex 版的责任分工和显式流程覆盖见 [Matt 联动说明](skills/codex-orchestration/references/matt-skill-routing.md)。
- 这些是交付流程指令。工具 schema、格式校验与离线场景检查不等于真实 App 中创建、续作和归档恢复已全部实测；每次交付应分别记录实际证据。

适合单个功能或 bug 的一次交付；跨多天工作应结合项目的 tracker 与交付流程。生产部署始终单独授权。Orca 的模型映射、命名和消息机制属于其宿主约定，不套用到 Codex App。
