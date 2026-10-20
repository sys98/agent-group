# orca-dispatch 流程适配

参考源：`/Users/yimijiu/.agents/skills/orca-dispatch/SKILL.md`。这是流程参考，未调用 Orca 执行引擎；同名 delivery-pipeline 内的 reference 不是本技能来源。

| 原技能步骤 | Codex 对应行为 |
| --- | --- |
| 0 同步最新代码 | 读取远程/本地状态，确定显式基线；无远程用本地，dirty 输入先解决归属，不自动 pull |
| 1 读项目规范 | 读取根与模块规则，提取实际 canonical 命令 |
| 2 诊断或需求澄清 | 实际加载 diagnosing-bugs 1–4；新功能复用 PRD 并加载 grilling；文字不能确定时加载 prototype 对应分支 |
| 3 先汇报计划 | 派发前说明证据、方案、测试切入点、所有权、模型与波次；明确要求确认时等待 |
| 4 拆任务 | 实际读取 to-tickets 切片原则，覆盖其发布步骤；合同强制 worker 加载 tdd，bug 再加载 diagnosing-bugs 5–6 |
| 5 建 Run 与启动波次 | 用任务账本恢复坐标和真实 agent ID，spawn_agent 启动独立实施；不伪造 Run/Dispatch |
| 6 监督与消息结算 | send_message / followup_task / list_agents / wait_agent；超时不判死；停止旧 writer 才接替 |
| 7 验收双轴审查 | 实际加载 code-review，核对物理基线和非空 diff，主代理重跑检查，传完整 smell baseline 并分轴报告；不通过交原实施者修复 |
| 8 合并与集成 | 授权后按项目约定串行集成，集成检查通过才进下一波；确认停止与已集成再清理 |
| 9 记录汇报 | 主代理更新项目长期事实、规格决策和验证报告；明确未合并、未推送及业务验收缺口 |
| 恢复与模型 | 实际 agent/Git/账本优先；宿主可用模型与约束优先，不复制外部模型名或假设 effective 参数 |

## 授权和宿主差异

原技能默认 worker commit、coordinator 合并，并仅使用 Orca worker 实施；本适配使用 Codex 原生实施代理、主进程不承担非琐碎业务实现。提交、合并等服从项目逐次授权，不从参考技能获得额外权限。Codex 原生协作没有 Orca FIFO ack、worker-release、PTY nudge 或持久 Run，不能声称具备这些原生命令和恢复保证。

模型按**派发时仍剩的难题**判断：机械变更、常规已定方案、仍有复杂判断分别说明。需重点评估并发/时序/状态机、公共合同破坏性变更或迁移、鉴权/资金/数据风险、留给 worker 的设计取舍、根因尚不确定五类；文件数量不是升级理由。用户明确指定时优先；不可用时报限制，不私自替换成外部模型名。原技能的固定 Claude/DeepSeek ID、high effort 和中文 Orca worker 名不照搬：使用宿主实际支持配置及命名约束（Codex agent task_name 等），任务表保留中文可读标题。两次因能力问题未过验收或主动升级请求时，核实旧 writer 停止后沿原任务续作，不新开重复写入者。

其余原生命令差异：有远程、默认分支干净且允许同步时可按项目规则 ff-only；否则保留显式本地基线并说明，分叉交用户确定。等待按 Codex 有界工具及沟通时限，不复制 540 秒阻塞。清理只处理本次拥有、已停止且通过集成的资源，不用 force 删除 dirty worktree。未获提交/合并授权时保留候选并报告门槛，不伪报完成。

Matt 技能逐阶段来源、嵌套 codebase-design、原型保存与流程覆盖，以 [matt-skill-routing.md](matt-skill-routing.md) 为准；worker 的强制加载合同见 [task-contract.md](task-contract.md)。

## 本次接管的流程差距

此前版本已做任务归属、独立审查和本地集成验证，但允许主代理写强耦合实现、采用共享 dirty 目录，且未要求严格按 orca-dispatch 的波次分支与双轴固定点流程。这些工程结果不能追认成已遵循原技能。后续按修订流程执行；不为补流程而重做已验证业务、补造提交、改历史记录或擅自拆动用户 dirty 工作区。
