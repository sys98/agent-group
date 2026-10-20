# Matt 技能联动

本表是加载路由和 `orca-dispatch` 的流程覆盖说明，不是 Matt 原技能的替代文本。路径按本机安装；跨机器时按宿主技能目录找到同名技能，不自动下载或安装。技能名称可能带 `mattpocock-skills:` 前缀，识别后读取其真实 SKILL.md；有原生技能调用工具时用工具，否则直接读文件。阶段只需读一次，版本变化后重读。

| 阶段 / 触发 | 必须读取的技能 | 负责人和产出 |
| --- | --- | --- |
| 修 bug：调查 | [diagnosing-bugs](/Users/yimijiu/.agents/skills/diagnosing-bugs/SKILL.md) | coordinator 做 1–4 阶段；提供已实际跑红的紧反馈命令、最小复现、3–5 个有预测的假设和根因证据 |
| 新功能：需求澄清 | [grilling](/Users/yimijiu/.agents/skills/grilling/SKILL.md) | coordinator 沿决策树只问前置已定的 frontier，事实自行查；每题给推荐，等待答复再推进依赖决策 |
| 文字无法确认 UI / 状态模型 | [prototype](/Users/yimijiu/.agents/skills/prototype/SKILL.md)，再按问题读其 UI.md 或 LOGIC.md | 一次性原型回答明确问题；已验证决定及符合正式规范的纯逻辑可纳入正式实现，完整原型另行保留，不将原型整体当生产实现 |
| 形成波次和任务 | [to-tickets](/Users/yimijiu/.agents/skills/to-tickets/SKILL.md) | coordinator 采用 tracer-bullet 窄完整切片、真实 blocked-by、可独立验证、必要预重构优先、宽变更 expand–contract；不执行其 tracker 发布/标签/额外确认步骤 |
| 实施功能或修复 | [tdd](/Users/yimijiu/.agents/skills/tdd/SKILL.md)，按要求读 tests.md、mocking.md | worker 在计划确定的 seam 一次一个行为测试→红→最小实现→绿；不批量先写测试、不内部实现耦合、不把重构混入循环 |
| 修 bug：修复和清理 | [diagnosing-bugs](/Users/yimijiu/.agents/skills/diagnosing-bugs/SKILL.md) | worker 做 5–6 阶段，修复前回归先红，修后重跑原始场景、删除临时 debug；正确 seam 缺失必须报告 |
| seam / module interface 本身有设计问题 | [codebase-design](/Users/yimijiu/.agents/skills/codebase-design/SKILL.md) | 按 tdd 的嵌套要求读取词汇参考，确定 interface、seam 和真实可变点；不无故启动重构或并行设计会话 |
| 每项候选验收 | [code-review](/Users/yimijiu/.agents/skills/code-review/SKILL.md) | coordinator 固定基线、验证非空 diff，提供 spec、规范与完整 smell baseline；两个独立只读代理分别跑 Standards / Spec，分轴报告与统计 |

## 澄清、测试切入点与确认

- 有 PRD：只针对本次范围和真实歧义加载 grilling，不另跑 to-spec，不重新访谈已定决策。大体明确最多两轮，其余安全默认写入计划；无决策可问时记录无需提问。
- 尚未成熟或“先聊清楚”：按 grilling frontier 迭代，直到共享理解形成；用户说“够了”即停止。补齐需求文档里的目标、流程、决策、非目标、验收；计划后必须等待明确确认才实施。
- 选择题优先宿主当前原生工具，推荐项在前；数量与选项限制按真实工具 schema。必要答复未收到就不执行依赖步骤，不把未选项自动当作批准。无工具时用 Q1/A/B/C 可一行回答格式。
- coordinator 在派发前加载 tdd 的 seam 要求，计划给出每个测试 interface 能抓到和抓不到的行为。orca-dispatch 将 seam 讨论并入计划：已有明确授权且无待定 seam 决策时直接传给 worker，不逐票重问；存在新决策或明确批准门槛时等答复，不能以“用户不反对”替代批准。
- `先确认`：无论需求成熟度，派发前停在完整计划等待答复；`推送` 只授权通过集成后的既定推送，不含部署；`worker 用 <模型>` 仅映射当前宿主实际支持的配置；`继续 <账本路径或任务标识>` 从实际代理和 Git 事实恢复，不伪造 run_id。

## 原型、拆票和审查的流程覆盖

- prototype 要求保留原型主来源在 throwaway 分支、默认分支只留结论。提交未获授权时保留仓外/隔离目录产物与绝对指针，注明未提交，不自动删除、不创建 Issue 或提交默认分支；用户点名要保留的产品原型不按 throwaway 处理。
- to-tickets 在本编排中只加载切片原则。原技能的发布与 triage 标签配置不运行；任务表与账本即本轮计划，不另建一套 tracker。预重构必须能证明是实现所需；依赖尽量保持三四层内，做不到说明真实原因，不造假边。
- code-review 使用本波基线，不让 worker 重新向用户索取已定 fixed point。执行 `rev-parse`、祖先检查、diff 与提交列表；empty diff 或 ref 错误先定位。已明确允许 dirty 审查时必须另附冻结文件列表、未跟踪文件及内容指纹，说明三点 diff 不涵盖这些文件，向两轴交同一候选材料。
- Standards 传实际规范以及 code-review 全量 smell baseline；Spec 传本次计划/任务说明与最新用户纠正。缺 spec 先补来源，仍缺则明确跳过且不得报双轴通过；不要为获取 spec 自动创建 Issue 或配置标签。两轴保持分离，各自统计，不能一轴通过掩盖另一轴失败。
- 每个候选记录技能路径/阶段、基线、证据与未验证项。技能无法读取时在已知技能根和用户指定来源查找；仍缺时说明具体路径和受影响阶段，可继续独立调查，不能用概括流程冒充加载或静默略过必需门槛。
