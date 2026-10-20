# Matt 技能联动与覆盖

这里给出读取时机和责任分工，不替代原技能。使用宿主技能目录解析同名或 `mattpocock-skills:` 命名技能，读取真实 `SKILL.md`。缺失时在已知技能根和用户指定来源查找；仍缺时报告受影响门槛，不自动下载、不用表格摘要冒充执行。

| 阶段 | 实际读取 | 负责人及要求 |
| --- | --- | --- |
| bug 调查 | `diagnosing-bugs` | coordinator 做 1–4：真实红色反馈命令、最小复现、公开 3–5 个可证伪假设与验证证据；读领域词汇和 ADR |
| 功能澄清 | `grilling` | coordinator 按前置已定的决策 frontier 提问，事实自行查；决策答复未到不推进依赖步骤 |
| 文字不足以确定 UI/状态 | `prototype` 及对应 `UI.md` / `LOGIC.md` | 明确原型回答的问题；保留完整原型为独立来源，正式分支只纳入已验证决定与合规实现 |
| 计划和拆任务 | `to-tickets` | 完整 tracer-bullet 切片、可独立验收、fresh context 可容纳、真实 blockers、必要 prefactoring、宽改 expand–contract |
| 派发前确定测试切入点 | `tdd` | coordinator 实际读 seam 规则，给出 catches/misses；取得明确确认或引用本任务已有确认，再写入合同 |
| 实施 | `tdd`、`tests.md`、`mocking.md` | 独立 worker 每次一个公开行为测试 → 实际红 → 最小实现 → 绿；预期来自独立需求或样例，不耦合私有实现，不批量先写测试 |
| bug 修复与清理 | `diagnosing-bugs` 阶段 5–6 | worker 使用正确 seam，首个回归先红，复测原始场景并清理诊断；没有正确 seam 就报告 |
| seam/公开接口本身未定 | `codebase-design` | 使用共享设计词汇，先向 coordinator 报告取舍；不以嵌套技能启动无关重构 |
| 每个候选验收 | `code-review` | coordinator 固定 base/candidate，读取非空 diff 与提交列表；两个独立只读上下文并行分别执行 Standards/Spec |

## 允许的编排覆盖

以下覆盖来自 `orca-dispatch` 或当前宿主约束，不能宣称是 Matt 原技能的完整默认流程：

- `grilling`：已有 PRD 只澄清当前范围的未决项；成熟需求最多两轮，`先聊清楚` 则继续到关键决策明确。原生选择题优先，按真实工具 schema；没有工具时用 Q1/A/B/C。不将未答复当批准，用户说“够了”立即停止，剩余默认明确记录。
- `to-tickets`：切片原则完整读取，拆分在同一次完整计划中展示并按已有确认门槛处理；不额外启动重复拆票问答、不自动建 Issue/标签或发布 tracker。用户明确要求 tracker 时另按其授权执行原技能发布步骤。
- `prototype`：完整原型按原技能留在 throwaway 分支并记录指针；本地提交未获授权时先保留隔离目录和来源坐标，明确尚未完成分支归档。不得丢弃完整来源或把“未提交”说成已归档。
- `tdd`：coordinator 统一完成用户 seam 确认，再把确认原文/坐标传给 worker；worker 不重复询问，但也不能把“coordinator 定了”说成“用户确认了”。原版“用户不反对就视为同意”的简化在本适配中不采用。
- `code-review`：固定点、规格与规范由 coordinator 提供，不重复向用户索取；用实际文件路径区分 Matt 技能和宿主同名内置 review。未配置 tracker 时使用已存在的项目规格，不为启动审查擅自建 Issue。

## 审查交接的必要材料

- coordinator 停止 candidate writer，记录本波 base 与候选 commit SHA；验证 ref、祖先关系和非空 diff，使用显式 `git -C <worker-path> diff <base>...<candidate>` 及对应提交列表。两轴均针对这个同一候选，不能各看活动 HEAD。
- Standards 收到项目规范来源与 Matt 原技能的**完整 smell baseline 原文**；不能只给一个 baseline 名称或本表链接。原规范优先，smell 是 judgement call，工具强制检查的事项不重复列。
- Spec 收到本次计划/任务合同、原始需求与最新用户纠正，逐项检查缺失、错误和范围扩张。缺 spec 明确报告 Spec 未完成，不报整体通过。
- 按 Matt 要求两个独立只读审查上下文并行，不让实施 worker 自审；保留两个轴各自的报告与计数，不混合重排。缺并行独立能力时报告受阻，不能以 coordinator 自述补齐。
- 修复产生新候选时两轴 verdict 均需对应新 SHA。测试证据仅在实现、输入、依赖与环境相关部分未变时复用；记录复用边界。
