<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="agent-group 提供 Orca 派发与 Codex 原生编排两种技能，协助调查、拆分、监督、审查和集成多代理任务">
</p>

**agent-group** 提供两种中文多代理交付技能：用 [`orca-dispatch`](skills/orca-dispatch/SKILL.md) 协调 Orca worker，或用 [`codex-orchestration`](skills/codex-orchestration/SKILL.md) 协调 Codex 原生子代理。两者都让 coordinator 对调查、任务边界和结果验收负责，并遵守各自宿主的能力与授权边界。

## 选择适合你的技能

| | `orca-dispatch` | `codex-orchestration` |
|---|---|---|
| 适用环境 | 配置了 Orca 的 Claude Code 或 Codex | Codex 原生代理协作 |
| 执行方式 | Orca Run、worker 和独立分支 | Codex 子代理与任务账本 |
| 恢复方式 | `继续 <run_id>`，依据 Orca 和 Git 状态 | 检查代理、Git 与项目账本，从最早未完成步骤继续 |
| 技能入口 | `/orca-dispatch` 或 `$orca-dispatch` | `$codex-orchestration` |

## 为什么使用 coordinator

多个 agent 并行工作时，常见问题是任务边界互相冲突、进度难以追踪，以及把 worker 自述的“测试通过”当成最终验收。两个技能都要求 coordinator 先理解项目和需求，再划分可验证任务，最后亲自复核交付证据。

- 修 bug 时先复现并定位原因；做功能时先读现有需求，只澄清会改变实现的决策。
- 任务按窄切片和依赖波次划分，明确负责人、文件归属与验收标准。
- coordinator 重新运行适用检查，并分别审查规范符合度和需求符合度。
- 合并、推送、部署和生产操作仍受项目规则与用户授权约束。

## Orca 工作流

下图展示 `orca-dispatch` 的 Orca 生命周期。Codex 原生协作的工具、监督与恢复方式不同，见 [`codex-orchestration`](skills/codex-orchestration/SKILL.md)。

<p align="center">
  <img src="./assets/readme/workflow.svg" width="100%" alt="orca-dispatch 从同步和调查、澄清计划、按波次派发，到监督 worker、审查分支并集成验证的流程">
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

### 使用 Codex 原生子代理

在 Codex 中调用 `$codex-orchestration` 并描述目标。技能使用 Codex 当前提供的原生协作工具，按任务维护负责人、文件归属、基线、验收和证据；它不依赖 Orca Run，也不提供 Orca FIFO 消息确认、PTY 唤醒或持久 Run 的等价保证。

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

# Codex native orchestration
mkdir -p ~/.codex/skills
ln -s "$PWD/skills/codex-orchestration" ~/.codex/skills/codex-orchestration

# Shared agent skill directory, if your host reads it
ln -s "$PWD/skills/codex-orchestration" ~/.agents/skills/codex-orchestration
```

如目标链接已存在，请先按本机技能管理方式处理，避免覆盖已有安装。

## 依赖与限制

| 技能 | 依赖与边界 |
|---|---|
| `orca-dispatch` | 需要 Orca 的 `orchestration`、`orca-cli` 技能；按阶段使用 `grilling`、`prototype`、`diagnosing-bugs`、`tdd`、`to-tickets`、`code-review`。在 Codex 中使用原生选择题时，可按需启用实验性 `default_mode_request_user_input` 功能。 |
| `codex-orchestration` | 需要当前 Codex 会话实际提供原生代理与任务协调工具；按阶段使用 `grilling`、`prototype`、`diagnosing-bugs`、`to-tickets`、`tdd`、`codebase-design`、`code-review`。 |

两个技能都以单个功能或 bug 的一次交付为目标；需要跨多天管理的大工程应结合项目自己的 issue tracker 和交付流程。`orca-dispatch` 的命令兼容性已按 Orca 1.4.222、Claude Code 2.1.296 和 Codex CLI 0.162.1 的帮助信息核对；完整多 worker 流程仍在实际使用中打磨。worker 默认模型和中文分支命名是作者的工作习惯，可按需修改技能正文。
