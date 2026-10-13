---
name: orca-dispatch
description: Orca 监督式编排快捷入口——修 bug 时先排查根因，做新功能时按想法成熟度澄清需求，汇报计划后把工作拆给 Orca worker 在新分支上执行，由 coordinator 复核、review 并合并。仅在用户显式调用时运行（Claude Code 输入 /orca-dispatch，Codex 输入 $orca-dispatch）。
argument-hint: "<问题或功能描述> [先聊清楚] [先确认] [推送] [worker 用 opus|sonnet|fable|codex] | 继续 <run_id>"
disable-model-invocation: true
---

问题或需求：$ARGUMENTS

（Codex 等不替换上面占位符的宿主里，以用户调用本技能时附带的文字为准，下文说的"调用参数"同理。）

用户输入本命令，即已明确选择执行方式：子代理执行（通过 Orca worker）。不要再询问"子代理执行还是直接执行"，也不要另写 `PRD.md`/`Tech-Spec.md`：第 3 步的汇报就是本次的计划。先加载 `orchestration` skill，并遵守其中的安全底线。

你是 coordinator：排查、拆分、监督、验收、代码 review、合并都是你的职责，不要把 review 或合并推给用户。全程按全局指令（AGENTS.md / CLAUDE.md）的进度播报要求，在每批工具调用前用中文说明上一批的发现和下一步。

### Coordinator 宿主适配

先按 `orchestration` 解析本会话的 CLI，再检查 `status --json` 的 caller 和 `run-create` 回执。CLI 示例中的 `orca` 都替换为本次解析出的同一个 executable。

- 有 `ORCA_TERMINAL_HANDLE` 时，用真实句柄补齐下方终端唤醒步骤。
- 没有终端句柄、但 Orca 已识别当前 App session 时，不把缺 PTY 当成缺 coordinator。建 Run 后验证 `orchestration check --run <run_id> --json` 可消费收件箱，再用 `send --to dispatch:<id>` 和 worker 按 live preamble 回传的 `status` 验证双向消息。以 session/Run 通路监督，不伪造句柄、不借用其他终端。
- App session 模式不注入 `terminal send`。worker 的 question、escalation 和 worker_done 仍按 live preamble 发送；coordinator 持续执行带 `--run` 的 check，逐批处理和 ack。双向验证失败时查精确回执与 runtime 绑定，不另起重复 worker。
- worker 在自然检查点读取自己的 follow-ups 时也保留完整回执；若返回 `deliveryId`，处理整批后用同一 caller 带 `--ack` 继续读到空。只打印消息正文而丢掉 deliveryId，可能反复读旧批次并漏掉后续指令。没有 deliveryId 不伪造 ack；coordinator 不冒充 worker consumer 代读或代 ack。
- `structworker_*` 是结构化 agent session，不能当作 PTY 传给 `worker-start --terminal`。需要续作而该路径被明确拒绝时，先查 mutation 回执与 Task/Dispatch，确认没有副作用；旧 worker 已发送有效 worker_done 后，用 `worker-release` 确认关闭，再在原工作树派发 fresh worker。未结算的旧 writer 不释放，也不启动第二个 writer。
- 缺 caller/session 绑定或 check 明确拒绝时，先定位 CLI/runtime 适配；不能把空列表或未核实的状态当成通路成功。

## 可选参数

- `先聊清楚`：新功能还没想好时用，第 2 步改走不限轮数的拷问，见第 2 步。
- `先确认`：第 3 步汇报计划后停下，等用户确认再派发；未指定则汇报后直接派发。

凡是要用户做选择的地方（第 2 步澄清、第 3 步确认计划、第 6 步转问 worker 的问题），都按第 2 步的规定优先用原生选择题工具。
- `推送`：第 8 步合并验证通过后，允许 `git push` 默认分支；未指定则不推送。
- `worker 用 <模型>`：见文末"Worker 模型"。
- `继续 <run_id>`：恢复一次中断的编排，见文末"恢复"。

## 步骤

0. **同步到最新代码**（排查之前完成，保证排查和 worker 基于同一份代码）：
   - `git fetch origin --prune`，确定默认分支（`git symbolic-ref refs/remotes/origin/HEAD`，或 `orca repo show` 中的 base ref）。
   - 用 `git rev-list --left-right --count <默认分支>...origin/<默认分支>` 比较本地与远程：
     - 相同或本地落后：基线 = `origin/<默认分支>`；若当前就在默认分支且工作区干净，执行 `git pull --ff-only` 让本地排查也看到最新代码。
     - 本地领先（有未 push 的提交）：基线 = 本地 `<默认分支>`，并在汇报中注明。
     - 已分叉：停止，把两边差异告诉用户，由用户决定基线。
   - 工作区有未提交改动时，提醒用户这些改动不会进入 worker 分支，且会阻碍第 8 步合并。
   - 记下基线提交的短哈希，后面每个 worker 都用它。
1. **先读项目规范**：读取项目根目录的 `AGENTS.md`、`MEMORY.md`、`CLAUDE.md`（大小写不敏感），以及本次涉及目录下的同名文件。提取项目实际使用的测试、类型检查、lint、构建命令，以及编码、提交、合并约定。
2. **只读调查**：先判断这是修 bug 还是新功能。
   - **修 bug**：加载 `diagnosing-bugs`，在当前 worktree 做它的第 1–4 阶段：先造出一条能在这个 bug 上变红的命令（已实际跑过、稳定、几秒内出结果），复现并缩到最小，列出 3–5 个可证伪的假设，再逐个验证找到根因。第 5 阶段（修复 + 回归测试）交给 worker，coordinator 不改代码。真的造不出这条命令时，按它的要求停下来告诉用户缺什么。
   - **新功能**：先找项目里已有的需求文档（用户点名的文件、`PRD.md`、`docs/` 下的需求或 spec）并读相关代码，事实自己查。有 PRD 时以它为准，它就是 spec，不再另跑 `to-spec`：PRD 已写明的不再问，只问 PRD 没写到、写得模糊或与现有代码冲突、而且会改变实现方式的点；PRD 覆盖多个功能时只做用户这次点名的部分。然后加载 `grilling`，用它的提问逻辑（每轮只问前置条件已定的问题、每题给推荐答案），事实自己查，只问需要用户拍板的决策。**提问形式覆盖 `grilling` 的纯文本格式**：宿主有原生选择题工具时一律用它：Claude Code 的 `AskUserQuestion`（一次最多 4 题）、Codex 的 `request_user_input`（一次最多 3 题，本机已开启默认模式可用的实验开关）。每题 2–4 个互斥选项，推荐项放第一个并在标签末尾标「（推荐）」，选项说明写清取舍；一轮问题超过单次上限就分几次连续调用，问完一轮再统一处理答案。工具当轮不在可用列表里，或调用后没有返回答案，就退回下面的文字格式，不要反复重试。真的列不出选项的开放问题（比如"产品叫什么"）才用文字问。宿主没有这类工具时改用可一行作答的文字格式：题目编号 `Q1`、`Q2`…，每题的选项编号 `A`、`B`、`C`，推荐项固定放 `A` 并标「（推荐）」；一轮末尾写一句"回 yes 全部采纳推荐，或回 `1A 2C 3B` 逐题选，不在选项里的直接写文字"。用户回复时按这个编号解析，没提到的题按推荐处理。按想法的成熟度选一种问法，并在第一轮开头告诉用户用的是哪种：
     - **大体想好了**（有 PRD，或描述里已说清要做什么、给谁用、怎么算做完）：**最多两轮**，到了就按当前结论继续，没问到的用合理默认值并在第 3 步写明假设。
     - **还没想好**（用户带了 `先聊清楚`；或者没有需求文档，而且用户流程、数据怎么存、用哪个外部服务这类决定整体做法的问题还没定）：按 `grilling` 原本的规则不限轮数，一直问到没有待定的决策。某个问题光靠聊说不清时（界面长什么样、状态流转对不对），加载 `prototype` 做一个一次性原型给用户看；原型不提交到默认分支，结论记下后按 `prototype` 的规则处理掉。问完后把结论写成需求文档（项目已有存放需求的位置就放那里，否则 `docs/<功能名>-prd.md`）：目标、用户流程、已定的决策、不做的范围、验收标准，之后把它当 PRD 用。这种情况下第 3 步无论有没有 `先确认` 都要停下等用户确认。
     - 两种都是用户说"够了"就立即收尾，没问到的写成默认假设。
   - 无论哪种，如果改动很简单（一处小改动），直接在这里完成、验证并汇报，不必派 worker。
3. **汇报调查结果与计划（必须，派发前）**：先用中文向用户输出一段完整汇报，再做任何派发：
   - 修 bug：根因（是什么、为什么会发生、代码位置）、证据（日志、数据、复现结果），以及那条会变红的复现命令和它的输出。新功能：已确认的决策（标明出自 PRD 哪一节，还是本次问到的）、采用的默认假设，以及 PRD 与现有代码不一致的地方。
   - 方案：打算怎么改、为什么这样改、影响范围和风险。
   - 测试切入点（seam）：每个任务在哪个公开接口上写测试、能抓到什么、抓不到什么。优先用已有的切入点，越靠上层越好，越少越好。这就是 `tdd` 要求的用户确认，用户不反对就视为同意，worker 不再问。
   - 任务拆分表：每个任务的中文分支名、改动内容、所有权文件、验收标准、使用的模型、所属波次。
   - 指定了 `先确认`，或第 2 步走的是"还没想好"，在此停下等待确认；否则汇报完成后继续第 4 步。
4. **拆分任务**：按 `to-tickets` 的切片原则拆，但不发布到 tracker、不单独问用户（第 3 步汇报就是确认）：每个任务是一条贯穿所需各层的完整窄切片，做完能独立验证；需要的预重构单独作为最先做的任务；一次改动波及全仓的机械改名类工作按 expand–contract 分批。同一波次内各任务改动的文件不能重叠，重叠的切片放到不同波次。任务之间有真实依赖（后一个要用前一个的代码）时分成多个波次，依赖链不超过三四层；能并行的尽量放同一波。任务说明全文用中文书写（命令、路径、代码标识符保持原样）。每个任务说明必须写清：目标范围、要做的改动、约束、所有权（可改哪些文件）、可观察的验收标准，并列出第 1 步提取的验证命令。背景资料（需求文档、相关代码、日志）用绝对路径或链接指给 worker，说明读哪一部分，不要整段粘贴。并行任务若共用数据库、端口等本地资源，在说明里为每个 worker 分配独立的资源。任务说明末尾附上下面这段；PTY 模式把 `<COORDINATOR_HANDLE>` 换成真实 `$ORCA_TERMINAL_HANDLE`，App session 模式将「唤醒 coordinator」三行换成「coordinator 通过 Run 收件箱持续检查；按 live preamble 发消息，不执行 terminal send」，其余规则原样保留：

   ```
   执行方式已选：直接执行，不要询问执行方式。遇到阻塞用 orchestration ask 问 coordinator，不要在本地弹问题等待。
   唤醒 coordinator（仅 PTY 模式）：每次向 coordinator 发出 worker_done、escalation 或 question 后，紧接着执行
   orca terminal send --terminal <COORDINATOR_HANDLE> --enter --text "<分支名> <消息类型>，请结算"
   PTY coordinator 可能需要这条唤醒。发送失败不影响已发出的消息，在 worker_done 里注明即可；结构化 worker 没有终端时也不要伪造句柄。
   项目规范（必须严格遵守）：
   - 每次自然检查点和 worker_done 前读取自己的完整 follow-up 回执；有 deliveryId 时，处理整批后 ack，再继续读到空，不漏后续批次。
   - 动手前读项目根目录及所改目录的 AGENTS.md / MEMORY.md / CLAUDE.md，遵守其中的架构、命名、编码、测试、提交规范；与本任务说明冲突时以项目规范为准，并在 worker_done 里说明。
   - 精准修改，匹配现有代码风格；不重构、不格式化无关代码，不新增依赖/抽象/配置，除非任务明确要求。
   - 加载 `tdd` skill，在本任务说明给定的测试切入点上按红→绿循环实现，一次一个测试；切入点已由 coordinator 和用户确认，不要再问。修 bug 时用任务说明里的复现命令作为第一个失败测试的依据，并加载 `diagnosing-bugs` 按它第 5、6 阶段修复和清理。完成前按项目规定运行测试、类型检查、lint、构建，全部通过才可报 succeeded，失败如实报 failed 并附输出。
   - 只在本 worktree 分支内改动所有权范围内的文件；不 push、不合并、不改其他分支。按项目提交规范 commit。
   - 不要修改 MEMORY.md；把值得长期记录的发现写进 worker_done 摘要，由 coordinator 统一写入。
   - 不读取、打印或提交 .env、密钥、凭据。
   ```

5. **建 Run，先全部启动再等待**：
   - 先建一个 Run 并记下 `run_id`，在汇报里告诉用户（中断后靠它恢复）：
     `orca orchestration run-create --objective "<一句话目标>" --json`
   - 当前波次所有任务都启动之后，再开始等待：
     `orca orchestration worker-start --spec "<中文任务说明>" --run <run_id> --worktree new-top-level --name <中文简短分支名> --base-branch <本波基线> --agent claude --model <模型> --effort high --json`
   - 第一波的基线是第 0 步的基线；后面每一波都在上一波合并、验证通过后再启动，基线换成合并后默认分支的最新提交（记下新的短哈希，第 7 步用它核对）。
   - `--name` 必须是中文短语（如 `修复-飞书表头超宽`），禁止用英文命名；词之间用 `-` 连接，不含空格和 `~ ^ : ? * [ \`，不含 `..`。Orca 会自动加上分支前缀，不要自己加。
   - 必须显式传 `--base-branch`，不依赖 Orca 的默认基线。
   - `worker-start` 返回非零时**不要直接重试**：读回执里的 `failedStage` 和 `residualResources`，按 `orchestration` skill 的 `references/recovery-and-cleanup.md` 处理，避免留下两个 worker 改同一份东西。
6. **监督**：循环等待，执行这条命令时把命令超时显式设到 600000ms（Claude Code 的 Bash 工具最多就是这么长；Codex 等宿主同样要显式设置，否则默认超时会提前杀掉等待）：
   `orca orchestration check --run <run_id> --wait --types "worker_done,escalation,question" --timeout-ms 540000 --json`
   - 返回的是最早一批未确认的消息，**不确认会一直重复返回同一批**。逐条处理完整批：回答 question，需要用户决策的转问用户；`worker_done` 进入第 7 步。处理完再带上 `--ack <delivery_id>` 继续等待：
     `orca orchestration check --run <run_id> --ack <delivery_id> --wait --types "worker_done,escalation,question" --timeout-ms 540000 --json`
   - 超时或返回空不是失败，继续等。**连续三次空等待**后，用 `orca orchestration worker-list --run <run_id> --include-remote --json` 查看每个 worker，按每行的 `projection.attention` 和 `projection.nextAction` 行动；`nextAction` 为 `none` 时看 `liveness.reason` 后继续等。
   - 只有确凿证据（`liveness` 为 `exited`，或 worker 的最后一轮没有发 `worker_done`）才可以停止、放弃或重派；`unverifiable` 和"没动静"都不算证据，继续等或查看终端。
   - 还有 worker 没有结算时，不要结束自己这一轮回复。
7. **验收与代码 review**：收到 `worker_done` 后不能直接采信，逐项完成：
   - 用 `git merge-base --is-ancestor <本波基线哈希> <worker 分支>` 确认该分支基于正确的基线。
   - 在对应 worktree 里亲自重跑验证命令。
   - 按 Matt Pocock 的 `code-review` 技能（双轴审查；如果它和宿主内置的 `/code-review` 同名冲突，按它的实际文件路径读取）审查该分支：固定点用本波基线哈希，Spec 用第 3 步的计划和该任务说明；它的 Standards 与 Spec 两个只读审查子代理并行跑。在两轴之外，coordinator 自己再看正确性、边界条件、错误处理、回归风险、安全，以及改动是否都在所有权范围内、有没有无关改动或新增依赖。
   - 不合格：回复具体修改意见让 worker 修，或重新派发，直到合格。
   - 合格后执行 `orca orchestration worker-release --dispatch <dispatch_id> --json`（只关闭 worker 终端，保留 worktree 和分支供合并）。回执显示释放不确定或被保留时，按回执给出的步骤处理，不要自己用 `terminal close` 代替。
8. **合并与集成验证**（当前波次全部验收通过后；有下一波时合并验证完再回到第 5 步启动下一波）：
   - 在默认分支上按项目合并约定逐个合并 worker 分支（项目无约定时用 `git merge --no-ff`）。
   - 合并后在默认分支上跑全量验证命令。
   - 出现冲突或验证失败：停止合并，已合并部分不要强行回退，向用户报告具体冲突或失败输出。
   - 合并并验证通过后，清理已合并的 worker：`orca worktree rm --worktree <id> --force --json`，再 `git branch -d <分支>`（`-d` 会拒绝删除未合并的分支，不要用 `-D`）。
   - 最后一波完成后，确认 `orca orchestration worker-list --run <run_id> --terminal-state reclaimable --json` 返回空；不为空说明还有 worker 终端没处理，处理完再汇报。
   - 指定了 `推送` 才执行 `git push`；部署永远不自动执行，只给出部署步骤。
9. **记录与汇报**：在项目允许写入的前提下，把 worker 上交的长期有效发现合并写入项目 `MEMORY.md`；项目有 PRD 时，把本次澄清新增的决策补进 PRD 对应章节，让它继续是唯一的需求来源。两者按项目提交规范一起提交。最终汇报：根因或已实现的功能、每个任务的结果与 review 结论、合并提交、集成验证结果、剩余风险、是否已推送、部署或数据修复等待用户确认的具体操作，以及是否更新了 `MEMORY.md`。

## 恢复

coordinator 会话中断（关掉、崩溃、上下文压缩后丢了状态）时，用户用 `继续 <run_id>` 作为参数调用本技能。不要凭记忆判断进度，以 Orca 和 Git 的实际状态为准：

1. `orca orchestration worker-list --run <run_id> --include-remote --json` 和 `orca orchestration task-list --run <run_id> --json` 查每个任务的状态。
2. 先不带 `--ack` 执行一次 `orca orchestration check --run <run_id> --json`，处理积压的消息批次。
3. 用 `git branch`、`git log <默认分支>` 判断哪些 worker 分支已经合并。
4. 从最早没完成的步骤接着做；已经合并的任务不要重做，已有 worker 在跑的任务不要再派一个。

## Worker 模型

- 默认按任务难度自动分配：第 4 步拆任务时逐个判断，并在第 3 步任务拆分表的"使用的模型"列写明每个任务的模型和理由。
  - `claude-sonnet-5`：边界清晰的机械性任务——小范围改动、改文案/样式、加日志、按明确方案补测试、纯搬迁式重构。
  - `claude-opus-5-5`：其余一切——跨模块改动、并发/时序/状态类 bug、涉及钱或数据的逻辑、安全相关、需要权衡设计的任务，以及判断不准时（宁高勿低）。
  - 所有 worker 统一 `--effort high`。
- 如果调用参数里指定了模型，全部 worker 按指定使用，不再逐个分配：`opus` → `claude-opus-5-5`，`fable` → `claude-fable-5-1`，`sonnet` → `claude-sonnet-5`，`codex` → `--agent codex`（不传 `--model`）。
- 实现工作只交给 Orca worker，禁止用宿主自带的子代理（Claude Code 的 Agent 工具、Codex 的子代理）改代码。唯一例外是第 7 步 `code-review` 的两个只读审查子代理。
