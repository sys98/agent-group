# Worker 任务合同与回执

每项任务一份合同，由 coordinator 补齐实际值。背景以可访问文件或链接引用，不能依赖另一会话的隐藏上下文。合同放在计划指定的持久位置；需要 worker 读取的非机密文档必须在其工作区可访问。下方实施合同仅在 App 初始化验收后以 follow-up 发送，不直接用作创建 prompt。

## 创建 prompt：只读初始化

写明用户要求的新独立任务、已核实 project/host、批准起点、范围和 Task 标识。要求读取规则并报告真实 cwd、common dir、HEAD、branch/detached、dirty 状态，以及本聊天 list_artifacts 的工作树身份。初始化期间不改文件、不创建分支、不启动实施子代理、不向其他聊天发消息；报告后结束 turn，等待 coordinator 在本聊天的实施合同。

初始 App 任务的创建和后续协调消息权限分别来自直接人类授权；原生子代理委派或 coordinator 消息本身不能扩大权限。

## 派发材料

- Task ID、尝试 ID、并发组/波次、blockers 及其具体产出、负责人、模型与选择依据；同波并行任务、共享资源互斥步骤和串行等待原因。
- 已确认的 App threadId、hostId、projectId、实际标题、托管 worktree identityKey；创建、协调消息和归档/保留各自的人类授权坐标。
- 用户目标、最新纠正、非目标、授权边界、可观察业务验收。
- 仓库与 Git common dir 的真实路径、独立 worker worktree 的真实路径、任务分支、完整 base SHA；必须保留的已有输入及其来源。
- 允许修改的文件/模块；公共合同 owner；不得修改的文件和共享资源。
- 规格、项目规范、词汇与相关 ADR、接口与调用方的可访问路径，指出相关章节。
- 测试切入点，catches/misses，用户确认的原文或持久坐标；bug 的已实际跑红命令、最小复现与根因证据。
- canonical 验证命令及资源隔离规则：测试库、端口、临时产物，必要的目标库只读核对；不提供 secret 值。
- 必须实际读取的 Matt 技能真实路径和阶段：`tdd` 与其 test/mocking 参考；bug 加 `diagnosing-bugs` 5–6；seam 自身未定先回报，必要时读 `codebase-design`。
- 持久回执位置、原 App 聊天的 blocked/续作协议、可执行的本地提交范围；生命周期策略、证据树外保存位置、未授权动作的处理方式。

## 必须随合同传给 worker 的执行约束

```text
Execution mode is already chosen: implement this task directly in the assigned worktree.
You are the implementation worker in this user-requested Codex App chat.
Initialization has been verified. Continue in this chat and its assigned worktree.
Do not spawn implementation subagents, new chats, or CLI workers.

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
- Other tasks in this wave may run concurrently. Respect the assigned file ownership,
  ports, database and resource locks. Report new dependencies or conflicts to the coordinator;
  do not absorb a sibling task or modify its branch to bypass the dependency.
- Check instructions in this chat and the task contract at natural checkpoints: before
  new files, after verification, and before reporting. Do not assume an Orca mailbox.
- Report results and questions in this chat. Do not message another chat just because
  a coordinator asks you to; cross-chat messages need direct human authorization.
- Do not archive this chat or worktree during implementation. Preserve unmerged work.
- Do not leave detached commands writing after your turn. Report any authorized long-running
  process handles and their state so a completed chat is not mistaken for a stopped writer.
- Commit only this task's verified files when local commits are authorized. Check the
  staged diff first. If sandbox or project rules block a commit, report ready_for_commit
  with the exact uncommitted file set; do not widen permissions or self-approve.

Reporting:
- If blocked, stop task writes and finish the turn with status blocked, a concrete question,
  current branch/HEAD, owned changes, and any missing permission or evidence.
- If ready, finish with status ready_for_review (or ready_for_commit), not independently
  approved. Include the evidence below. Stop task writes and finish this turn.
- Repairs continue in this same App chat, branch and worktree. Do not create a replacement.
- If the user manually takes over this chat, disclose the change so the coordinator can
  suspend automatic continuation instead of racing the user's work.
```

`ready_for_commit` 仅是待 coordinator 核验的中间态：worker 必须已停止；coordinator 在授权范围内审核真实 diff 并创建固定候选提交，之后才进入双轴审查。没有提交权限则保留现场并报告待批，不能改用 HEAD 排除未提交内容。

## 回执

- 状态：`blocked` / `ready_for_commit` / `ready_for_review` / `failed`，以及原因。
- Task/attempt、真实 worktree、分支、base SHA、候选 SHA 与 dirty/未跟踪文件。
- 实际加载的技能路径与阶段；每条已运行命令和结果，适用的 red/green、原始场景复测。
- 改动与用户目标的关系、业务验收、未验证路径、已知限制和阻碍。
- 要问 coordinator 或用户的具体问题，长期发现供 coordinator 记录。

App ID、创建回执与 wait cursor 由 coordinator 从真实工具回执记录，不让 worker 猜。writer 停止、绿色测试、独立审查、集成验收、聊天归档和工作树回收分别取证。仅有 ready 回执不触发清理。
