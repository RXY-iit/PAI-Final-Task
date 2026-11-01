# Mission Agent：Task 委派与 JEV 编排

更新：2026-10-13。第一版已接入代码；本文明确区分当前实现与后续设计。
本轮修改与真实 API 验证：[结构化绑定与能力契约](test&pro&solve/17-command-grounding-and-perception-capabilities-20261013.md)。
统一能力表：[Task / Skill](../task-skill-redefine.md)。
领域：[Navigation](../navigaiton-agent/README.md)、[Perception](../perception-agent/README.md)、[Arm](../arm-agent/README.md)。

## 1. 概念与职责

- Mission 是一次用户请求，不穷举 Mission 类型。
- Task 有独立目标与完成条件。一个 Task 由一个 owner 负责，必要时委派子 Task。
- Step 是一次 Robot Skill 调用；Task 内称 task step。**不再使用 Mission Step。**
- Robot Skill 是可调用执行能力，可以是受限组合流程。Skill 定义与 Step 实例分开。
- Agent Skills 知识包不是 Robot Skill，不列入执行白名单。

Mission 负责理解用户目标、任务顺序、依赖、委派、取消和汇总。领域 Agent 决定当前 Task 的具体方法；Skill 执行端负责参数、实时条件、停止和完成证据。
九视角搜索是一个 search_object Task，调用一个 search_at_place 组合 Skill，内部子调用仍可追踪。

## 2. 当前代码入口

| 内容 | 定义来源 |
| --- | --- |
| 12 类 Task、owner、slots、旧名兼容 | [task_catalog.yaml](../../../src/robot_agent/config/task_catalog.yaml)、[task_catalog.py](../../../src/robot_agent/robot_agent/task_catalog.py) |
| Mission 调度与委派 | [mission_agent.py](../../../src/robot_agent/robot_agent/agents/mission_agent.py)：_build_plan、_expand_task、_complete_task |
| 领域细化返回值 | [domain.py](../../../src/robot_agent/robot_agent/agents/domain.py)：TaskRefinement |
| JEV 请求、问题、真实回复、脱敏 trace | [jev_agent.py](../../../src/robot_agent/robot_agent/jev_agent.py) |
| Primary 结构问题、原文边界绑定、单位抽取、依赖编译 | [command_grounding.py](../../../src/robot_agent/robot_agent/command_grounding.py) |
| Task/Step 实例与状态 | [schemas.py](../../../src/robot_agent/robot_agent/schemas.py)：SkillCall、MissionState |
| Robot Skill 名单、契约、兼容别名 | robot_skills/config/skill_registry.yaml、skill_catalog.v2.yaml、skill_extensions.yaml |
| 取消、执行及观测证据域 | [skill_client.py](../../../src/robot_agent/robot_agent/skill_client.py) |
| 展示和报告 | web/src/components/MissionPanel.tsx、web/src/lib/missionLogic.ts、human_report.py |

Task 清单为 get_robot_status、list_known_targets、cancel、move_relative、rotate_relative、move_lift、set_arm_pose、observe_scene、search_object、navigate_to、interact_with_object、manipulate_object。
新增 Task 要改 catalog 和对应 owner，不应另在 JEV 维护一份平行名称列表。旧解析名字经 TaskCatalog.normalize 转换，现有 Skill API 不强制重命名。

## 3. 已实现的执行流程

M12/M13 执行契约补充：

- 支持有界控制流：搜索成功后执行一个相对平移/转向，否则保持静止。Primary 由 JEV 判断这种语义，不再要求句子命中特定 if/else 正则。Classic 仍保留原有有界句式。多分支、循环、超过三项目标等仍需澄清，不是通用条件编程语言。
- Primary 先选 Task 类型，再独立调用 command_bind 选择原文 token 边界和每个 Task 的依赖。代码验证边界、顺序、单位、参数和条件图；不会用 Classic 或连接词切分结果否决/覆盖合法绑定。Classic 只在 plan_review 比较。
- 条件执行读取同一父 Task、depends_on 指定搜索的 search_at_place 完成证据；旧格式兼容紧邻前一次搜索。FOUND 且目标满足才允许运动；完整 NOT_FOUND 则标记后续 Task 为 SKIPPED/CONDITION_FALSE，Mission 的 stay-still 分支满足；PARTIAL、未完成、失败、矛盾证据均不放行。
- 搜索记录仍保留 goal_satisfied=false，不伪造检出。条件式 Mission 可以因为正确执行了不运动分支而成功，这与无条件搜索 Mission 的 PARTIAL 不混算。
- M13 `check red object and go back a little` 在未确认目标后，`task_catalog.yaml` 的 `unmet_perception_policy: ask_before_motion` 先要求人工确认。批准继续绑定当前 mission_id/wait_id，重做正常的 AUTO/安全前置检查，不重新搜索、不重开 Mission。旧按钮和 Cancel all 不得复活任务。required_condition 不提供绕过条件的继续选项。
- `red box` 当前可能只可检测 red，box 无直接类别；PARTIAL 条件等待是能力不足，不再是“条件执行尚未实现”。完整条件链可先用 bottle 等直接类别验证。

1. 接收命令，冻结本次 agent_mode；运行中切换模式只影响下次 Mission。
2. command_parse 识别形态、数量、Task 类型；command_bind 绑定原文及每项依赖，编译 canonical Task intent/sequence。
3. plan_review 审查任务级计划，不预先执行相机/导航/机械臂。
4. 执行到 kind=task 时调用 owner.refine。之前完成的运动、当前位置/目标解析都在此边界重新考虑。
5. refine 返回合法 SkillCall 列表，或问题/能力不足。仅明确计划会进入动作执行。
6. 内部 Step 保留 task_id、step_id、owner，Mission 共用一个串行调度器调用 SkillClient。
7. task_end 是内部完成标记，不是 Robot Skill；聚合 Step 最新尝试和子 Task 结果，发 task_feedback。
8. 全部结束汇总 Mission outcome；错误按原安全/异常路径处理，用户取消优先。

**第一版是延迟细化 + 共用执行器，不是四个独立线程/ROS 节点，也不是完整动态 TaskRunner。**
领域 Agent 当前拥有计划细化，运行错误仍主要由共用 Mission 执行器处理。局部多轮重规划、独立预算树与资源租约尚未实现，不能据 owner 标签宣称完整领域自治。

当前内部 Task 节点示例（代码组装，不是 JEV 自由输出）：

~~~json
{
  "name": "navigate_to",
  "kind": "task",
  "task_id": "m1:t1",
  "owner": "navigation",
  "payload": {
    "task_type": "navigate_to",
    "target": "door",
    "constraints": {}
  }
}
~~~

展开后 Step 仍使用原 Skill 名，例如 goto_pose；新别名 navigate_to_pose 也可调用。Navigation 需要视觉确认时委派 search_object 子 Task，保存 parent_task_id。Perception 不会反向触发导航。

## 4. JEV 各阶段

| 阶段 | 当前输入/问题 | 执行影响 |
| --- | --- | --- |
| command_parse | 原文、catalog 类型及动作/参数角色定义；command_shape、task_count、task_1..3_type，以及各项 alternative_type | 识别目标结构及有意义的替代解释；有歧义先问人，不枚举所有类型的参数 |
| command_disambiguation（按需） | 原文、已选 Task 类型；仅对 alternative_type 重复主类型的项追问，选项排除主类型 | 最多一批追问；另一类型转人工选择，none 表示无第二解释；非法/缺失/unclear 停止绑定 |
| command_bind | 原文 token 边界、已选类型、控制流；每项 source/goal 起止与依赖，只有相对运动问方向/参数来源 | 数值由代码在当前片段抽取，默认值来自 task_catalog.defaults；目标原文交领域 Agent，不在 Mission 选择 arm pose/model |
| plan_review | JEV/Classic 具体 TaskGraph、原文及代码结构校验；三问：两个 graph_faithful + plan_source_decision | 选中的图必须 faithful 且通过代码验证；缺失/矛盾答案不放行，不能把单一来源投票当批准 |
| Navigation refine | 当前 navigate_to、实体、接近关系和适用规则；执行时加入实际路径摘要 | navigation_command_parse、navigation_plan_review（semantic / computed_path）；同 Task 消歧；异常才 navigation_step_supervision |
| Perception refine | 当前目标、分类/颜色/OCR 能力和模型/profile | perception_command_parse、perception_plan_review；Primary 的合法选择写入计划，异常才 perception_step_supervision；详见领域 README |
| step_supervision | 失败 Step 及相关上下文、后续任务依赖、必要健康证据 | 可请求恢复、停止/问人，或跳过失败 Task 继续独立目标；不跳过失败前置直接执行同 Task 后续动作 |

Primary command_parse 不接收 Classic 解析候选。Primary 的对比项只运行 deterministic parser，在 plan_review 才使用，避免 embedding/LLM 先阻塞 JEV。
Classic/Shadow 保留 Hybrid 的确定性、embedding、本地 LLM 回退；embedding 默认只加载本地缓存，缺失不会自动联网下载。明确需要下载时使用 ROBOT_EMBEDDING_ALLOW_DOWNLOAD=1。

JEV 的 choice 是类型/策略/数量选项，不是任意 JSON 或物理参数生成器。step count 沿用最多三项的解析策略；实现兼容转换可能把旧复合任务展开为最多六项 canonical Task，不代表支持六项自由语言解析。

### 4.1 Task 类型消歧

`set pose home` 省略了执行器/参考系，允许 Arm 姿态与底盘地图目标两种解释；拦截是正确的，不用规则强行归为 Arm。
结构阶段共八问，各项 `alternative_type` 选择另一种真正合理的 Task 或 `none`，不是增加一个任务。候选类型和按钮标签来自 `task_catalog.yaml`，不是针对某个句子的替换规则。

- 若有明确的替代解释，Primary 在 command_bind 前提供两个明确按钮，例如 Arm named pose / Base map location，以及 Cancel all。
- 若 alternative_type 重复主类型，不再默默忽略；用 command_disambiguation 带入已知主解释，并排除该选项。不能假设同一次 API 的问题按顺序共享答案。
- 若 API 提供实际 choice probabilities，也可用前两项接近作为保守澄清提示：第二项至少 0.25、差距不超过 0.20，且主选项在前两项中。这只是启发式，不是经过校准的置信度，不要求两个互斥选项都接近 1。
- 保留同一 mission_id；用户只确认有歧义的那一项，其余原文和任务保持不变。按钮按 wait_id 校验，旧按钮无效。
- 选择后继续 command_bind / plan_review，不重发整条命令。`human_confirmed_task_types` 同时约束 JEV 与 Classic 候选，review 不能反转人工确认。
- 选导航不代表地图必有 home，选 Arm 不代表姿态必可执行，仍由领域 Agent 验证目标、能力和安全条件。
- Cancel all 直接结束整个 Mission，不继续绑定、不执行已解析的前置运动。Shadow 只记录歧义，不额外询问或改变 Classic 执行。

实现：`robot_agent/command_grounding.py`、`robot_agent/jev_agent.py`、`robot_agent/agents/mission_agent.py`。10/13 用户实测已验证 M15 的人工确认和同 Mission 继续；重复类型的新增追问只做了离线回归，尚未真实 API 复验。
领域问题依照职责分组，详见领域 README。传入 unresolved_view 的观察目标在 Perception 的 perception_view_parse 中判定方向、范围和分析目的；未支持的相对云台调整仍需澄清。

### 4.2 请求版本与启动 probe

请求 state 统一使用 `schema_version: jev_robot_agent.v2`。它是项目契约标记，不是模型版本，也不会让 JEV 自动加载本地文件。
每条 `jev_calls.jsonl` 增加 `questions_sha256`，按实际问题/指令/选项规范化计算；便于区分同 stage 的措辞或动态选项变更。哈希记录在本地日志，不加入模型输入；不是整份代码/配置版本的替代。

启动 probe 仍是三次只读请求，但改为复用正式的 command_parse、command_bind、plan_review 问题生成器，不再维护旧 single_step/sequence_step_count/noul 示例。所有必答项必须有合法选项值；空 answers 不算可用。
probe 日志以 `purpose=probe` 区分，普通调用为 `purpose=decision`，保留原始用量、延迟与问题哈希。按 Mission 统计成本时排除 probe，启动成本另计。Live 仅证明 API 连通且回答结构有效，不证明解析准确或所有领域 Agent 已验收。

## 5. 模式与反馈

- Classic：采用确定性领域细化；不调用领域 JEV。
- Primary：JEV 选择经代码验证后影响任务计划。未配置 API 时明确为 local。已配置 API 时，解析、绑定、计划审查或领域审查调用失败/答案不合法，不采用 local 默认值作为模型许可。执行异常阶段已有确定性停止/恢复保护独立保留，不算 JEV 成功。
- Shadow：记录 JEV 建议，但仅执行 Classic 领域计划；拒绝建议也不额外问人或运动。当前调用仍是串行等待，不宣称无延迟后台执行。
- dry_run / stub：只验证编排，不证明检测/硬件目标达成。

事件新增 task_delegated、task_plan_created、task_feedback；Skill 结果带归属信息。MissionState.tasks 显示 parent_task_id、state、outcome、goal_satisfied。
当前 outcome 是 SATISFIED / UNMET / UNSUPPORTED / UNAVAILABLE / NEEDS_HUMAN 等简化标签；更细的 NOT_FOUND/身份关联结果仍在 Skill metrics 中。
DONE 且存在未满足任务时 Mission outcome=PARTIAL；失败记录不会被后续成功覆盖。重试完成判断使用同 step_id 的最后尝试，历史不删除。
导航严格/语义容差/降级仍在 goto_pose 的 completion_class 中分别保留。

## 6. 依赖、人工、恢复和取消

- then/and 不自动授权忽略失败。完整依赖图尚未实现，当前保守停止。
- 旧 near_place 复合命令转为 navigate_to + 后续 Task，标 required_condition。
- 用户明确“无论找到与否都 arm home”的收尾沿用明确 cleanup 规则；整体保留 PARTIAL。
- JEV 建议继续时跳过整个失败 Task 的剩余内部步骤，不在失败的相机定位后继续拍照。
- 领域有限选项和失败分流使用 `mission_id + wait_id` 的 `mission_control`。AUTO、目标消歧、对象替代、失败后重试/继续可在原 Mission 恢复，旧 wait 不可复用。保留必选 Cancel all。
- **尚未实现**同 Task 多轮自由文本澄清、任意局部 checkpoint。有限选项续行不等于完整 checkpoint；旧式 `command` 选项仍是新任务。
- Recovery 中间 `IN_PROGRESS/final:false` 不结束等待，最终回复后重查失败步骤；不能跳过失败的前置条件。同一步自动 Recovery 最多一次，持久故障转人工。
- Primary 的失败分流先于旧运动失败分支。相对运动可能已完成一部分，不自动重放原距离/角度；恢复后仍待人工明确后续独立目标或取消。剩余位移自动续行未实现。
- `retry_failed_step` / `continue_after_failed_task` 是当前 wait 的受限动作；后者保留原任务未完成结果，跳过其内部剩余步骤，后续仍过依赖/安全门。显式选择不会在同一依赖边界重复询问。
- `retry_with_alternate_view` 尚无独立的新视角重规划实现，转人工，不伪称已换视角。
- Cancel 传播到 SkillClient run tombstone、组合 Skill、相机等待和 Arm 等待；旧 JEV 返回后再次检查取消。取消后不自动归位、不发下一张或后续 Task。
- 发送 stop/cancel 与已确认物理静止是不同证据；新取消链仍需 sim 实测。Health/Recovery 不能代替执行端安全门。

定位依赖只属于需要定位的执行链；相对运动仍使用 odom，不新加全局定位门。恢复何时介入仍沿用已有策略，本轮后置。

## 7. 可扩展性与待办

新增 Task：catalog 语义/owner/slots -> normalize/legacy（仅需兼容时）-> owner.refine -> 完成证据 -> JEV slots -> 委派/失败/取消测试。
新增 Skill：registry/catalog 合同或扩展 manifest -> SkillClient dispatcher/执行器 -> mode/参数/取消/结果测试。manifest 别名只迁移接口名，不提高物理能力。
新组合 Skill 的内部图不得读取整句命令；语义和模型选择属于领域 Agent。

优先待办：类型化 TaskResult/契约验证器、逐边依赖图、多轮自由文本反馈、无损 checkpoint、领域局部重新规划、版本化规则、资源冲突和父子预算。
本轮不启用 thinking LLM，不重构 Recovery，不扩大导航容差。
验收记录：[11-domain-agents-20261011.md](test&pro&solve/11-domain-agents-20261011.md)。

2026-10-12 Primary 实测与失败分流修复见 [16-primary-suite-20261012.md](test&pro&solve/16-primary-suite-20261012.md)。JEV 原始 API 问答在 session `agent/jev_calls.jsonl`；显示用 trace 不再混入本地假设概率，缺失回答标 `missing`。Human report 保留用量统计，真正凭据仍脱敏。
