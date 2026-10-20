# Orca 流程与 Codex worker 对应

源流程为 agent-group 仓库中的 `skills/orca-dispatch/SKILL.md`；独立安装时不要求同时安装 Orca。以下保留交付不变量，只替换实际执行机制。

| 原流程 | Codex 对应 |
| --- | --- |
| 同步、项目规则与调查 | 相同阶段划分；显式基线、真实复现或需求证据，遵守当前项目权限 |
| grilling、prototype、计划 | 实际读取原技能；已有规格复用；测试切入点有明确确认记录 |
| to-tickets | 按窄完整切片与 blockers 排波次；沿用原编排集中计划，不自动发布 tracker |
| Run 与 worker-start new-top-level | 持久账本记录真实恢复坐标；每任务独立分支/worktree，再启动独立 Codex CLI worker |
| worker 消息与 FIFO ack | 运行日志与明确 blocked 回执；coordinator 答复后恢复指定 CLI session，不假装具有 Orca ack |
| worker_done 与验收 | 核对真实进程退出、候选提交和 worktree，coordinator 重跑检查；退出和自述不能代替验收 |
| Matt code-review | 相同固定基线、候选、独立 Standards/Spec 上下文和分轴报告；原生子代理仅用于只读工作 |
| worker-release | 证明 CLI worker 已停止；没有同名 release 或 PTY nudge 能力时不编造 |
| 合并、全量验证、下一波 | 授权范围内串行集成；集成通过后更新下一波基线 |
| 清理与恢复 | 只清理本次已停止且已集成资源；按账本、Git、进程与会话实际状态恢复 |

## 必须保留的边界

- 独立 worker/worktree 是默认实施条件，不是可选优化。无法创建工作树或启动独立会话时，实施受阻；不能退回共享目录的原生子代理。
- 工作树只隔离文件 checkout。Git refs、进程端口、数据库和外部系统仍可能共享，必须另行协调；不因有 worktree 就允许并发改共享合同。
- Orca 的中文分支名与固定模型映射是原流程的宿主约定。Codex 按项目命名与 CLI 实际支持模型执行，未指定模型沿用 CLI 配置，不声称继承 coordinator 模型。
- Matt `tdd` 的切入点确认不能用沉默替代；原 Orca 文本中的此项宽松约定不复制。Matt 的 tracker 发布/重复拆票确认在本编排中被集中计划替代，必须披露该覆盖。
- `grilling` 等技能的只读调查、`code-review` 的双轴子代理，与实施 worker 是不同角色；不能因都叫 agent 就互换。
- 不复制 Orca 的等待超时、强制清理或 CLI 命令；采用宿主有界等待和非强制资源清理，不能超越当前用户或项目授权。

实际启动、续作和 Git 核验见 [worker-runtime.md](worker-runtime.md)；Matt 原技能加载与覆盖见 [matt-skill-routing.md](matt-skill-routing.md)。
