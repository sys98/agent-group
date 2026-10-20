# Orca 流程与 Codex worker 对应

源流程为 agent-group 仓库中的 `skills/orca-dispatch/SKILL.md`；独立安装时不要求同时安装 Orca。以下保留交付不变量，只替换实际执行机制。

| 原流程 | Codex 对应 |
| --- | --- |
| 同步、项目规则与调查 | 相同阶段划分；显式基线、真实复现或需求证据，遵守当前项目权限 |
| grilling、prototype、计划 | 实际读取原技能；已有规格复用；测试切入点有明确确认记录 |
| to-tickets | 按窄完整切片与 blockers 排波次；沿用原编排集中计划，不自动发布 tracker |
| Run 与 worker-start new-top-level | 创建用户明确要求的新 App 工作树任务；核实真实 threadId、可见聊天和独立分支/worktree 后实施 |
| worker 消息与 FIFO ack | wait_threads/read_thread 收取状态；授权 follow-up 发回同一 threadId，不假装具有 Orca ack |
| worker_done 与验收 | 核对任务 turn、派生 writer、候选提交和 worktree，coordinator 重跑检查；完成和自述不能代替验收 |
| Matt code-review | 相同固定基线、候选、独立 Standards/Spec 上下文和分轴报告；原生子代理仅用于只读工作 |
| worker-release | 证明 App 任务 writer 已停止；没有同名 release 或 PTY nudge 能力时不编造 |
| 合并、全量验证、下一波 | 授权范围内串行集成；集成通过后更新下一波基线 |
| 清理与恢复 | 授权归档原 App 任务，分别核实托管树回收和分支；恢复同一 threadId 和其托管树 |

## 必须保留的边界

- App 可见任务与独立工作树是新编排的实施条件。缺少匹配 App 项目、创建/消息授权或真实工具时实施受阻；不能退回 CLI 或共享目录子代理。
- 工作树只隔离文件 checkout。Git refs、进程端口、数据库和外部系统仍可能共享，必须另行协调；不因有 worktree 就允许并发改共享合同。
- Orca 的中文分支名与固定模型映射是其宿主约定。Codex 按项目命名与 App 目标 host 的实际能力执行；未指定模型不传覆盖，不声称继承 coordinator 模型。
- Matt `tdd` 的切入点确认不能用沉默替代；原 Orca 文本中的此项宽松约定不复制。Matt 的 tracker 发布/重复拆票确认在本编排中被集中计划替代，必须披露该覆盖。
- `grilling` 等技能的只读调查、`code-review` 的双轴子代理，与实施 worker 是不同角色；不能因都叫 agent 就互换。
- 不复制 Orca 的等待超时、强制清理或 CLI 命令；托管工作树不能由 coordinator 越权 raw Git 删除。聊天归档、工作树回收与分支删除分别验收。
- 新建 App 聊天与跨聊天消息必须满足直接人类授权和宿主规则；技能不能把普通子步骤自动转为用户所有的新聊天。
- 升级前已派发 CLI 尝试使用 [冻结 CLI 合同](legacy-cli-runtime.md)，不迁移、不补建 App 聊天、不干预现有 writer。

App 创建与续作见 [worker-runtime.md](worker-runtime.md)，归档/恢复见 [worktree-lifecycle.md](worktree-lifecycle.md)；Matt 加载与覆盖见 [matt-skill-routing.md](matt-skill-routing.md)。
