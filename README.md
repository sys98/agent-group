<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="agent-group 通过 Orca 或 Codex 独立 worker 在各自分支与工作树实施，由 coordinator 调查、监督、独立审查并集成验证">
</p>

**agent-group** 提供两种中文多代理交付技能：用 [`orca-dispatch`](skills/orca-dispatch/SKILL.md) 协调 [Orca](https://github.com/stablyai/orca) worker，或用 [`codex-orchestration`](skills/codex-orchestration/SKILL.md) 启动独立 Codex worker。实施任务在各自的分支与工作树中完成，由 coordinator 调查、监督、复核候选，再串行集成。

## 选择适合你的技能

| | [Orca 版](skills/orca-dispatch/SKILL.md) | [Codex 版](skills/codex-orchestration/SKILL.md) |
|---|---|---|
| 宿主 | Claude Code / Codex | Codex |
| 执行器 | Orca worker | Codex CLI |
| 恢复坐标 | Run + Git | 账本 + 会话 + Git |

两版都在独立分支与工作树实施。Orca 负责创建并管理其 worker；Codex 版由 coordinator 先建树，再启动独立 CLI 会话。具体调用见下方快速开始。

## 为什么使用 coordinator

多个 agent 并行工作时，常见问题是任务边界互相冲突、进度难以追踪，以及把 worker 自述的“测试通过”当成最终验收。两个技能都要求 coordinator 先理解项目和需求，再划分可验证任务，最后亲自复核交付证据。

- 修 bug 时先复现并定位原因；做功能时先读现有需求，只澄清会改变实现的决策。
- 任务按窄切片和依赖波次划分，明确负责人、文件归属与验收标准。
- 每个实施任务使用独立工作树；同一共享合同仍需统一归属，端口和数据库另行隔离。
- Codex 版将明确确认的测试切入点交给 worker，按 Matt 的逐条红绿循环实现，并独立审查规范符合度与需求符合度。
- coordinator 重跑适用检查，按固定候选完成双轴审查，集成验证通过后才启动依赖波次。
- 合并、推送、部署和生产操作仍受项目规则与用户授权约束。

## 从问题到集成

两个技能共享以下交付链，使用各自的 worker 执行器。Codex 的启动、监督、指定会话恢复和安全清理见 [worker 运行说明](skills/codex-orchestration/references/worker-runtime.md)。

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

### 使用独立 Codex worker

在 Codex 中调用：

```text
$codex-orchestration 修复素材生成点击后一直转圈的问题
$codex-orchestration 实现 docs/PRD.md 第 3 节「批量生成」 先确认
$codex-orchestration 继续 <账本路径>
```

coordinator 为每个任务核对独立分支与工作树，再以 `codex exec -C <worktree>` 启动独立会话。原生子代理只承担只读调查和 Matt 双轴审查。缺少独立 worker 或工作树能力时停止实施派发；worker 需要决策时以 blocked 回执结束当前 turn，收到答复后在原工作树恢复指定会话。

支持 `先聊清楚`、`先确认`、`推送` 和 `worker 用 <实际模型 ID>`。未指定模型时沿用 CLI 配置。账本记录真实进程、会话与 Git 坐标，帮助中断后恢复；它不提供 Orca FIFO ack，也不会在 coordinator 关闭后自动监督。

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

# Independent Codex workers
mkdir -p ~/.codex/skills
ln -s "$PWD/skills/codex-orchestration" ~/.codex/skills/codex-orchestration

# Alternatively, use ~/.agents/skills if your host reads that directory.
# Install codex-orchestration in only one discovery directory.
```

如目标链接已存在，请先按本机技能管理方式处理，避免覆盖已有安装。

## 依赖与边界

- **Orca 版本**需要 Orca 与其 `orchestration`、`orca-cli` 技能。
- **Codex 版本**需要 Git、Codex CLI、可跟踪的独立进程执行能力，以及用于 Matt 双轴审查的两个独立只读上下文。文档命令已对照本机 Codex CLI 0.163.0 帮助核对；其他版本应先检查实际能力。
- 按阶段读取 [Matt Pocock 技能](https://github.com/mattpocock/skills)：`grilling`、`prototype`、`diagnosing-bugs`、`to-tickets`、`tdd` 与 `code-review`；公开接口尚未确定时还需 `codebase-design`。Codex 版的责任分工和显式流程覆盖见 [Matt 联动说明](skills/codex-orchestration/references/matt-skill-routing.md)。
- 这些是交付流程指令。CLI 帮助核对、技能格式校验和 Git 隔离检查不等于真实项目的完整多 worker 交付验收。

适合单个功能或 bug 的一次交付；跨多天工作应结合项目的 tracker 与交付流程。生产部署始终单独授权。Orca 的模型映射、命名和消息机制属于其宿主约定，不套用到 Codex CLI。
