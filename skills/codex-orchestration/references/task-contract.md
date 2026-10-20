# Worker 任务合同与回执

每项任务一份合同，由 coordinator 补齐实际值。背景以可访问文件或链接引用，不能依赖另一会话的隐藏上下文。合同放在计划指定的持久位置；需要 worker 读取的非机密文档必须在其工作区可访问。

## 派发材料

- Task ID、尝试 ID、波次、blockers、负责人、模型与选择依据。
- 用户目标、最新纠正、非目标、授权边界、可观察业务验收。
- 仓库与 Git common dir 的真实路径、独立 worker worktree 的真实路径、任务分支、完整 base SHA；必须保留的已有输入及其来源。
- 允许修改的文件/模块；公共合同 owner；不得修改的文件和共享资源。
- 规格、项目规范、词汇与相关 ADR、接口与调用方的可访问路径，指出相关章节。
- 测试切入点，catches/misses，用户确认的原文或持久坐标；bug 的已实际跑红命令、最小复现与根因证据。
- canonical 验证命令及资源隔离规则：测试库、端口、临时产物，必要的目标库只读核对；不提供 secret 值。
- 必须实际读取的 Matt 技能真实路径和阶段：`tdd` 与其 test/mocking 参考；bug 加 `diagnosing-bugs` 5–6；seam 自身未定先回报，必要时读 `codebase-design`。
- 持久回执位置、coordinator 的 blocked/续作协议、可执行的本地提交范围；未授权动作的处理方式。

## 必须随合同传给 worker 的执行约束

```text
Execution mode is already chosen: implement this task directly in the assigned worktree.
You are an independent implementation worker. Do not spawn implementation subagents.

Before writing:
- Read the project rules and the actual Matt skill files named in the contract.
- Verify the physical cwd, Git common directory, task branch, and HEAD against the contract.
- On the first attempt require a clean checkout at the assigned base; on a repair verify
  the recorded candidate and explicitly owned changes. Report any mismatch as blocked.
- Use only the confirmed public test seams. If confirmation is missing or a seam must
  change, report the exact decision needed instead of inventing user approval.

During implementation:
- For behavior changes, work one public behavior test -> observed red -> minimal fix -> green.
- For bugs, perform diagnosing-bugs phases 5-6 and rerun the original reproduction.
- Do not use private implementation mocks or tautological assertions as acceptance evidence.
- Edit only owned paths in this worktree; do not merge, push, deploy, change shared services,
  read credentials, or edit project MEMORY.md. Do not switch another worktree or branch.
- Check the task's persisted instructions at natural checkpoints: before new files,
  after verification, and before reporting. Do not assume a live Orca mailbox exists.
- Commit only this task's verified files when local commits are authorized. Check the
  staged diff first. If sandbox or project rules block a commit, report ready_for_commit
  with the exact uncommitted file set; do not widen permissions or self-approve.

Reporting:
- If blocked, stop task writes and finish the turn with status blocked, a concrete question,
  current branch/HEAD, owned changes, and any missing permission or evidence.
- If ready, finish with status ready_for_review (or ready_for_commit), not independently
  approved. Include the evidence below. The coordinator verifies the real process has ended.
- Repairs resume this same task in the same worktree. Do not create a replacement writer.
```

`ready_for_commit` 仅是待 coordinator 核验的中间态：worker 必须已停止；coordinator 在授权范围内审核真实 diff 并创建固定候选提交，之后才进入双轴审查。没有提交权限则保留现场并报告待批，不能改用 HEAD 排除未提交内容。

## 回执

- 状态：`blocked` / `ready_for_commit` / `ready_for_review` / `failed`，以及原因。
- Task/attempt、真实 worktree、分支、base SHA、候选 SHA 与 dirty/未跟踪文件。
- 实际加载的技能路径与阶段；每条已运行命令和结果，适用的 red/green、原始场景复测。
- 改动与用户目标的关系、业务验收、未验证路径、已知限制和阻碍。
- 要问 coordinator 或用户的具体问题，长期发现供 coordinator 记录。

CLI 会话 ID、宿主进程句柄和进程退出状态由 coordinator 从真实执行事件记录，不让 worker 猜。回执、进程退出、绿色测试、独立审查和集成验收分别取证。
