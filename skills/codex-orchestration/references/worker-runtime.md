# 独立 worker：启动、监督与恢复

在首次派发、修复续作、接管和清理前读取相关部分。这里的 worker 是**独立 Codex CLI 会话与其进程**，不是 `collaboration.spawn_agent`；后者只用于只读调查和独立审查。遵守宿主对执行器、委派和消息的限制；宿主不允许启动这种 worker 时，实施受阻，不绕过限制。

## 预检和持久记录

先检查 `codex --version`、`codex exec --help`、`codex exec resume --help`。CLI 不存在、版本缺功能、执行器不能保留运行句柄或不能访问目标工作树时，停止派发。不得通过重新登录、读取凭据、改全局配置或开启危险权限来“修复”预检。

在项目已有计划机制中使用一个持久账本，`update_plan` 仅同步显示进度。每个任务保存：合同、波次、base、分支、worktree 的真实路径与 common dir、每次尝试的目录、CLI session ID、宿主进程句柄、操作系统进程标识及启动时间、退出状态、回执、候选与审查 SHA、集成结果。运行日志放在任务专用证据目录；检查其不进入业务提交，不复制凭据和敏感运行配置。

启动前写入“启动待核对”记录。进程已启动但 session ID 尚未回传时，也视为可能已有 writer；不能重新派发来代替核对。PID 可能复用，恢复时需结合启动时间、命令、工作目录、宿主句柄与日志核验。

## 创建并核对 worktree

优先使用能正确定位本仓库和本波基线的宿主托管工作树工具，异步创建必须等 completed 后核验实际路径。工作树默认 detached HEAD 时，在自己的工作树创建任务分支后再启动 worker。

没有适合的托管工具、且项目允许原生 Git 时，采用等价明确操作。下列变量由 coordinator 填写为已经核验的实际值；路径在本次任务拥有的目录下，分支不得复用已有 writer 的分支：

```bash
git -C "$repo_path" rev-parse --verify "$base_sha^{commit}"
git -C "$repo_path" worktree add -b "$task_branch" "$worker_path" "$base_sha"

git -C "$worker_path" rev-parse --show-toplevel
git -C "$worker_path" rev-parse --path-format=absolute --git-common-dir
git -C "$worker_path" branch --show-current
git -C "$worker_path" rev-parse HEAD
git -C "$worker_path" status --porcelain=v1 --untracked-files=all
```

按平台解析路径，核对 worker 路径真实且不同于 coordinator 和其他 worker；common dir 属于同一仓库、任务分支互异、初始 HEAD 等于 base、初始状态干净。不是“执行了命令”就算通过。创建失败先查看 `git worktree list --porcelain`、分支和真实目录，保留归属不明资源，不盲目重建或强删。

原工作区有本任务依赖的 dirty 输入时，先明确如何纳入授权基线；未经授权不自动 commit/stash/reset，也不把干净但缺输入的 worktree 当成功。工作树依赖安装只运行项目认可的命令，不复制 .env 等凭据文件。数据库和端口另行隔离。

## 启动一个 worker

为每次尝试分配新证据目录。通过宿主提供的可跟踪执行工具运行下列命令，使用有界 yield 返回执行句柄；不要用脱管的 `nohup`、`disown` 或自行搭建后台守护进程。

以下为 POSIX shell 示例。它记录 supervisor PID、CLI 日志和最终退出码；`task.md` 是已完成的合同。并发任务各用不同的 `worker_path` 和 `attempt_dir`。权限示例仅在不超出宿主及项目既有授权时采用；更严格的现有约束优先。

```bash
printf '%s\n' "$$" > "$attempt_dir/supervisor.pid"
worker_exit=0
codex -a never -s workspace-write exec \
  -C "$worker_path" --json \
  --output-last-message "$attempt_dir/result.md" - \
  < "$attempt_dir/task.md" \
  > "$attempt_dir/events.jsonl" 2> "$attempt_dir/stderr.log" || worker_exit=$?
printf '%s\n' "$worker_exit" > "$attempt_dir/exit-code"
exit "$worker_exit"
```

coordinator 另记宿主返回的执行句柄、supervisor 的启动时间与进程信息。这里的 PID 是 supervisor，不是 Codex session ID。从真实 JSONL `thread.started` 事件取得 `thread_id`，登记到同一任务；已有非空文件不能当本次回执。CLI 无法持久记录 session 时，报告恢复能力缺失。

不使用 `--ephemeral`、`--skip-git-repo-check`、绕过审批/沙箱或认证覆盖参数。未指定模型时省略 `-m`，记录“CLI 配置，实际模型未核实”；用户指定模型时按实际 CLI 能力传入，不照搬 Orca 模型别名。若主进程在受管环境中禁止这种执行器，不能改用 shell 逃逸。

不在已经准备好的工作树里再传 `--worktree`，否则可能创建另一棵树，破坏记录的路径和基线。当前 CLI 的自动建树能力不能替代显式基线与任务分支核验。

## 监督与双向决策

当前波可启动的 worker 全部启动后，再通过宿主原进程句柄进行有界等待。按进度需要读事件和回执，避免全文反复读取日志；不要把凭据或敏感工具输出回显。shell 返回码、`turn.completed`、最终回执必须分别记录。

- 运行中：保持监督；无输出/等待超时不视为 worker 死亡。CLI 的原生子代理消息接口不能发送给本 worker。
- 需要输入：合同要求 worker 停止写入并以 `blocked` 完成当前 turn，说明具体问题、状态和缺口。coordinator 先证明该执行已结束，再回答或转问用户。CLI 回执不存在时不能假定只是缺答案。
- 验收失败：coordinator 将具体失败命令与输出、审查发现交还原 worker；不接手业务修补。
- 中断/非零退出：核对现有进程、可能存活的后代命令、日志、工作树 dirty 和 session。没有结束证明前不创建替代 writer。任何未停止进程都不得被清理的工作树“带走”。

不搭建假的 Orca mailbox。若当前 CLI 提供 live steering，也必须先查其帮助和消息语义并取得适用授权；本流程不依赖该能力。紧急停止用实际宿主支持的进程控制，确认 CLI 及其任务后代已停止，再记录保留的改动与下一步。

## 在原 worktree 续作

每次续作创建新的尝试证据目录，把答复或修复合同写入 `followup.md`。确认原 writer 已结束，核对同一任务分支、真实路径、原候选和现有改动；**续作不能要求重新变成初始干净 base**。

以下命令在原 worker 的工作目录运行，使用真实 session ID；`resume` 的参数以实际版本帮助为准。此处 `-C` 放在顶层，不能假设 `exec resume` 支持 `-C`。

```bash
printf '%s\n' "$$" > "$attempt_dir/supervisor.pid"
worker_exit=0
codex -a never -s workspace-write -C "$worker_path" exec resume \
  "$worker_session_id" --json \
  --output-last-message "$attempt_dir/result.md" - \
  < "$attempt_dir/followup.md" \
  > "$attempt_dir/events.jsonl" 2> "$attempt_dir/stderr.log" || worker_exit=$?
printf '%s\n' "$worker_exit" > "$attempt_dir/exit-code"
exit "$worker_exit"
```

核对续作事件与会话，不用 `--last`、不 fork 新 session 冒充恢复。原 session 确实无法恢复时，只有证明旧 writer 已停止后才能在原工作树启动替代独立会话；合同携带原候选、未提交变更与先前证据，账本记录显式移交。

## 固定候选、串行集成与清理

确认 worker 已停止且所有交付内容进入授权候选提交，再核验：

```bash
git -C "$worker_path" status --porcelain=v1 --untracked-files=all
git -C "$worker_path" merge-base --is-ancestor "$base_sha" "$candidate_sha"
git -C "$worker_path" diff --check "$base_sha" "$candidate_sha"
git -C "$worker_path" diff "$base_sha...$candidate_sha"
git -C "$worker_path" log --oneline "$base_sha..$candidate_sha"
```

空 diff、错误 ref、未包含的 dirty 文件、分支偏离或越界变更先处理，不能直接进入“通过”。Matt 双轴收到相同 base/candidate 和完整材料；两轴审查期间禁止 worker 继续改候选。

当前波验收通过后，在授权集成目标串行合并指定候选并运行 canonical 验证。共享 Git refs 的修改由 coordinator 串行控制；worker 仅修改自己的任务分支。通过后记录集成 SHA，下一波以该提交为新基线。

清理前复核任务归属、writer 与后代已停止、工作树干净、候选确实被目标分支包含；仅 diff 相同不等于已集成。托管工作树使用其托管清理工具。原生 Git 示例：

```bash
git -C "$repo_path" merge-base --is-ancestor "$candidate_sha" "$integration_sha" &&
  git -C "$repo_path" worktree remove "$worker_path" &&
  git -C "$repo_path" branch -d "$task_branch"
```

不加 force。被拒绝时保留资源并解释，不绕过 Git 的未合并保护。恢复时先读账本，再核实运行句柄/session/真实 worktree 与集成历史；从最早未完成步骤继续。账本和日志提供恢复坐标，不能代替持久调度器。

## 能力来源

- [Codex 非交互执行与指定 session 恢复](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Codex 工作树](https://learn.chatgpt.com/docs/environments/git-worktrees)

命令形状已对照 Codex CLI 0.163.0 的本地帮助；其他版本应重新核对。帮助检查不证明登录、模型调用、任务质量或整轮编排验收成功。
