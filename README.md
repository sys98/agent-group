<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="orca-dispatch: one command takes a bug or feature through investigation, splitting, parallel Orca workers, per-branch review and merge">
</p>

**orca-dispatch** 是一个给 [Orca](https://github.com/stablyai/orca) 用的 agent skill。你在 Claude Code 或 Codex 里用一条命令描述一个 bug 或新功能，它会以 coordinator 的身份自己完成排查、拆分、并行派发 Orca worker、逐分支审查和合并，只在需要你决策的地方停下来问你。

> 本仓库目前只有一个技能：[`skills/orca-dispatch/SKILL.md`](skills/orca-dispatch/SKILL.md)。技能正文是中文写的，面向中文工作流。

## 它解决什么问题

直接用多个 agent 并行写代码，常见的麻烦是：任务拆得互相踩文件；worker 做完了没人知道；收件箱里的消息被反复读到同一批；"测试通过"只是 worker 自己说的；最后合并时一团乱。

orca-dispatch 把这些都写成了 coordinator 的固定职责：

- **派发前先想清楚**：修 bug 先造一条能稳定复现的命令再找根因；做新功能先读已有 PRD，没想好就先问清楚、写成 PRD。
- **任务互不重叠**：每个任务是一条能独立验证的窄切片，同一波次内文件不重叠，有依赖就分波次。
- **不漏消息**：建一个 Run，逐批处理并确认（`--ack`）worker 的消息，连续空等三次就去查 worker 状态，只凭确凿证据才停止或重派。
- **验收不靠自述**：coordinator 亲自重跑验证命令，用双轴审查（是否符合规范、是否符合需求）看每个分支，再逐个合并、跑全量验证、清理 worktree 和终端。
- **能接着做**：会话中断后，用 `继续 <run_id>` 按 Orca 和 Git 的实际状态恢复，不重做已合并的任务。

## 流程

<p align="center">
  <img src="./assets/readme/workflow.svg" width="100%" alt="Workflow: sync and investigate, clarify and plan, split and dispatch, supervise workers, then review, merge and clean up">
</p>

| 步骤 | coordinator 做什么 | 你要做什么 |
|---|---|---|
| 0–1 | 同步到最新的默认分支，读项目的 `AGENTS.md` / `CLAUDE.md` / `MEMORY.md`，找出测试和构建命令 | — |
| 2 | **修 bug**：造出会变红的复现命令，列假设、找根因<br>**新功能**：读 PRD 和代码，只问会改变实现的决策 | 回答选项式提问 |
| 3 | 汇报根因或决策、方案、测试切入点、任务拆分表 | 确认计划（加 `先确认` 或需求没想好时） |
| 4–5 | 按窄切片拆任务，建 Run，整波启动 worker | — |
| 6 | 等 `worker_done`，回答 worker 的提问，逐批确认消息 | 只回答转给你的问题 |
| 7–8 | 重跑验证、双轴审查、合并、全量验证、清理 | 推送需要你另外点头 |
| 9 | 把新发现写回 `MEMORY.md` 和 PRD，给出最终汇报 | 看结果 |

## 用法

在 Orca 的终端或 Chat 里打开项目：

```text
/orca-dispatch 素材生成点击后一直转圈，不出结果
/orca-dispatch 实现 docs/PRD.md 第 3 节「批量生成」 先确认
/orca-dispatch 小说素材按免费章节生成文案和图片 先聊清楚
/orca-dispatch 继续 run_3f9c
```

Codex 里把 `/` 换成 `$`：`$orca-dispatch …`

| 参数 | 作用 |
|---|---|
| `先聊清楚` | 需求还没想好：不限轮数地问，必要时做一次性原型，问完先写 PRD |
| `先确认` | 计划出来后停下，等你确认再派发 |
| `推送` | 合并验证通过后允许 `git push`；部署永远不会自动执行 |
| `worker 用 opus\|sonnet\|fable\|codex` | 所有 worker 统一用指定模型；不指定时按任务难度自动分配 |
| `继续 <run_id>` | 恢复一次中断的编排 |

需要你选择时，它会优先弹出宿主的原生选择题（Claude Code 的 `AskUserQuestion`、Codex 的 `request_user_input`）；没有时用 `Q1 / A B C` 的文字格式，回 `yes` 或 `1A 2C` 就能答完。

## 安装

这是一个技能目录，复制或软链到你的 agent 会读取的技能目录即可：

```bash
git clone https://github.com/sys98/agent-group.git
# Claude Code
ln -s "$PWD/agent-group/skills/orca-dispatch" ~/.claude/skills/orca-dispatch
# Codex 等读取 ~/.agents/skills 的 agent
ln -s "$PWD/agent-group/skills/orca-dispatch" ~/.agents/skills/orca-dispatch
```

### 依赖

| 依赖 | 用途 | 来源 |
|---|---|---|
| Orca 与 `orchestration`、`orca-cli` 技能 | worker 派发、收件箱、worktree | Orca 自带 |
| `grilling`、`prototype` | 澄清需求 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `diagnosing-bugs`、`tdd` | 排查 bug、worker 先写测试 | 同上 |
| `to-tickets`、`code-review` | 切片原则、双轴审查 | 同上 |

Codex 想用原生选择题，需要在 `config.toml` 的 `[features]` 下加 `default_mode_request_user_input = true`（开发中的实验功能）。

## 限制

- 已在 Orca 1.4.222、Claude Code 2.1.296、Codex CLI 0.162.1 上按命令帮助逐项核对过；完整的多 worker 编排仍在实际使用中打磨。
- worker 默认是 Claude 模型，分支名要求用中文，这些是作者的工作习惯，按需改 `SKILL.md`。
- 适合单个功能或 bug 的一次交付；跨多天、需要 issue tracker 管理的大工程不是它的目标。
