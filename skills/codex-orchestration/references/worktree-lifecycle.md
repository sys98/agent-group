# App 任务与托管工作树生命周期

创建前固定策略，初始化后记真实身份，集成验收后再归档。聊天、工作树、分支是三个不同资源，分别记录状态，不能用一个 completed 代替。

## 最小状态与身份

| 阶段 | 可继续的动作 | 退出条件 |
| --- | --- | --- |
| creating / creation_unknown | 查原回执与注册任务，不重复创建 | 找到真实 threadId 或确认创建失败及残留 |
| initializing | 等只读初始化，核对 cwd/base/工作树身份 | App 可见、身份匹配、独立任务分支、允许发送实施合同 |
| implementing / blocked | 等待结果或在同一 threadId 续作 | writer 停止，交付内容和缺口可核对 |
| reviewing / repairing | 固定候选双轴审查，修复仍用原任务 | 两轴及适用检查对应最终候选 |
| accepted / integrated | 串行集成并验证、保存证据 | 目标分支包含候选，集成检查通过 |
| chat_archived | 核实 App 的工作树处理结果 | 回收有证据，或明确 retained/reclamation_unknown |
| restored | 核对同一任务及恢复后的路径/HEAD/分支 | 重新固定候选与归属后才续作 |

账本保存：本次 run/Task 标识、transport、projectId/hostId、真实 threadId/标题、pending clientThreadId、worktree identityKey/真实路径/common dir、任务分支、base/candidate/integration SHA、wait cursor、业务状态、聊天状态、工作树状态、证据路径、人类授权来源和保留原因。

身份由实际创建回执、所属聊天的 artifacts、Git 读回确定，不从目录名推导。聊天归档可成功而树仍保留；分支也可能仍在，必须分别说明。

## 归档前门槛

以下全部满足才能自动进入已授权的归档阶段：

1. 这是本次创建并记账的任务，用户已授权本次验收后的归档。用户固定、手工接管或另有保留要求时先保留；不调整全局清理设置。
2. 最新任务 turn、其派生写入命令均已停止，无未结算的 writer、用户决策或后续依赖需要留在该树操作。worker 的 ready 回执不是停止的唯一证明。
3. 已核对完整交付内容；目标集成分支实际包含最终 candidate SHA，canonical 集成检查通过。reviewing、未合并候选、dirty/未跟踪业务文件或结果未知时不归档。
4. 合同、最终回执、检查证据、两个审查轴、候选与集成 SHA、恢复坐标已保存到工作树外的项目既有持久位置。ignored 文件不一定被快照保存；必要的非机密产物先保留，不复制 secret。
5. 工作树归属明确、干净且不被其他任务/用户依赖；共享、长期/permanent、固定或当前主 checkout 不由本编排强制回收。

门槛不满足时保留具体状态与原因，继续无依赖工作；不为凑清理结果降低条件。

## 聊天归档与树回收

- coordinator 在授权内用 `set_thread_archived` 对实际 worker threadId 归档；明确指定 ID，不误归档调用者自身。保留任务入口和原标题。
- App 对关联的托管工作树提供自动清理与快照恢复机制；**归档回执只证明聊天状态变化**。检查实际工作树/登记结果和可用归档证据，分别记为 archived、retained 或 reclamation_unknown；没有快照标识就不编造。
- 管理型工作树 API 按聊天归属工作。`list_artifacts` 仅列调用聊天附件，不能用 coordinator 的列表声称看到了别的 worker 附件。`archive_worktree` 只能处理调用聊天实际拥有且工具允许的工作树；不能临时 attach 另一聊天的树来夺取清理权。
- 若必须单独归档工作树，仅在所属聊天已有这项生命周期授权、且工具允许时进行；通过授权消息安排在该聊天的最终只读收尾 turn，先保存树外证据，最后执行允许的归档动作，然后由 coordinator 归档聊天。工具不允许归档当前执行目录时保留并报告，不强行执行；不要为了清理重新唤醒已经归档的聊天。记录真实 identityKey 和结果，不把“已发出请求”当完成。
- API 拒绝 primary/pinned/shared、子模块等情况时保留并报告；没有合适的跨聊天回收接口时让 App 生命周期处理，不能用 `git worktree remove`、rm、force 或编辑 App 状态数据库绕过。
- 工作树确认已回收、不再检出任务分支后，coordinator 才在集成 checkout 核对候选祖先关系并用 `git branch -d` 删除本次已集成任务分支。分支仍被检出或安全删除被拒绝时保留，不用 `-D`。

官方说明中的 App 自动删除与快照是产品机制，不能替代本次 readback。只有“任务归档完成”时，最终汇报不能写成“工作树与分支已全部清理”。

## 恢复同一任务

1. 根据真实 threadId、host 和既有授权取消该任务归档；使用原聊天，不新建一个同名替代任务。
2. 若工作树仍在，核实路径、common dir、branch 和 HEAD；若已被 App 回收，通过原任务的恢复入口或其所属上下文可用的 `restore_worktree` 恢复。该工具只接受该聊天 artifacts 中真实的 archived identityKey；不要在 coordinator 上下文猜一个 root 去恢复别人的树。
3. 工具无法从当前上下文恢复时，明确需要在原任务中使用 App 恢复入口，保留其状态。不能切到别的树运行后宣称已经恢复。
4. 恢复可能使用 detached HEAD，并将以前未提交内容包含在恢复快照提交中；重新核对 branch、候选 SHA、文件和依赖，不能要求它回到初始 base 或 reset 丢弃内容。旧审查只适用于原候选。
5. 根据最新人类目标，用原 threadId 发续作合同；尚未明确 writer/归属或用户已接管时暂停自动发送。

## 既有运行与失败资源

- 已有 CLI worker 使用 [冻结合同](legacy-cli-runtime.md)，保留其原会话、路径、分支和进程控制。新版默认 App 模式只用于尚未派发的新编排，不给运行中的 CLI 尝试补建聊天。
- 接管其他 coordinator 的任务需要直接人类授权，并先确认旧负责人及写入者停在安全点。仅凭 list/read 或 received message 不取得接管权。
- 创建结果未知时记录 clientThreadId 与可能残留；只清理已证明确属本次且无 writer 的资源。不能根据相似任务名批量归档或删除。
- 取消任务不等于授权销毁未合并成果。保留现场及证据并说明未集成内容；用户要求删除时另按真实权限与保留门槛处理。

产品语义参考：[Codex 工作树与归档恢复](https://learn.chatgpt.com/docs/environments/git-worktrees)。实际操作始终以当前工具的归属和能力限制为准。
