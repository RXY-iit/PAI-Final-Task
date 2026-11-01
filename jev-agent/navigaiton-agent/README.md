# Navigation Agent：三阶段执行契约

更新：2026-10-12。本文件描述本轮代码，尚未做本轮真实 API / Isaac 运动验收。目录沿用 navigaiton-agent 拼写。

## 1. 分工与实现边界

2026-10-12 M01-M17 补充：到达确认的特殊对象应在活动地图的 `semantic_map.yaml` 中定义，而不是依赖 Perception 猜名字。
当前 `maps/sim_factory_mid360_like_gpu_trueimu_011/semantic_map.yaml` 的 `shelf_06.visual_arrival_confirm`：

```yaml
enabled: true
query: shelf_label_06
recognition:
  kind: ocr
  tokens: ['06', '6']
  aggregation: any
  meaning: Printed shelf number at the shelf_06 label-side approach
```

接近位置/方向继续使用同一实体的 `approach_policy`；这不是任意位置见到数字就算到达。Navigation 成功到达并停车后委派确认 Task，把 `recognition` 作为 `recognition_contract` 传给 Perception。两种数字写法属于一个身份目标，允许同批图片多次 OCR 检查。当前 recognition 实现仅支持 OCR；其他类型返回明确不支持，不忽略配置。旧目标无新字段时暂保留兼容规则，后续应逐项迁移，不能批量猜测标签和坐标。

未知地点现在明确返回 `UNKNOWN_TARGET`，不再给出没有候选的“选择地点”提示。导航到位、OCR 确认与 Mission 完成分别记录。

Mission 把 navigate_to Task 委派给 Navigation；Navigation 明确目标、接近关系、到达和视觉确认要求，输出有序 Robot Skill。内部依然由共享 Mission 调度器执行，不是新建独立 ROS 控制节点。导航到位后可以委派 Perception 子 Task；子任务仍有同一 Mission 和明确 parent_task_id。

| 环节 | 当前实现 |
| --- | --- |
| 缺少 semantic_map、未知地点、未标定目标 | 不生成运动计划，返回明确问题 |
| 多个同名门 | 同 Task 的人工选项；mission_id + wait_id 绑定，不创建新的 go to 命令 |
| 唯一实体 | navigation_command_parse 校核语义、几何类型和视觉确认目标 |
| 语义计划 | navigation_plan_review，phase=semantic；检查规则和有序 skills |
| 实际路径 | goto_pose 获取 ComputePathToPose 结果后，navigation_plan_review，phase=computed_path |
| 窄道 | 使用现有 navigation_annotations 分段策略；JEV 可接受或要求澄清，不能创造不存在的 profile |
| 失败 | 确定性有界重试后，navigation_step_supervision；正常成功不触发 |
| 结果 | navigation_arrival 与 visual_confirmation 分开，保留 completion_class |
| 取消 | API 返回后再次检查取消，旧结果不得发新 goal |

代码：[navigation_agent.py](../../../src/robot_agent/robot_agent/agents/navigation_agent.py)、[navigation_plan.py](../../../src/robot_agent/robot_agent/agents/navigation_plan.py)、[Mission 调度器](../../../src/robot_agent/robot_agent/agents/mission_agent.py)、[导航执行器](../../../src/robot_skills/robot_skills/executors/navigation.py)。

## 2. 阶段输入与问题

| Stage | 最小输入 | 问题 / 合法结果 | 执行影响 |
| --- | --- | --- | --- |
| navigation_command_parse | 本 Task、匹配实体 ID/名称/类别、已标定 pose、region/point、接近关系、规则 | goal_binding: matched / ask_human；goal_kind: region / point；visual_confirmation: required / not_required / ask_human | 不一致或缺失回答则澄清；不能凭模型制造坐标或把区域改成点 |
| navigation_plan_review / semantic | 同一目标契约、适用规则、有序 skill 名 | decision: execute / ask_human | 只有合法方案进入执行；此时明确尚无物理路径 |
| navigation_plan_review / computed_path | 实际路径 hash、长度、点数、最多 32 个采样点、目标、窄道标注、执行段角色、执行器检查职责、到达后视觉确认时机 | decision: execute / ask_human；route_strategy: 当前已实现策略 / ask_human；blocking_issue: none / goal_conflict / rule_conflict / unsupported_strategy / insufficient_route_evidence | 拒绝或非 none 冲突不发运动；递归 waypoint、alternate goal 也经过 callback |
| navigation_step_supervision | 当前失败 skill / failed_child、结果和指定 metrics、任务依赖窗口、相关安全证据 | catalog 通用问题 + 当前 skill 专属问题 | 交共享 Mission 策略决定重试、Recovery、人工或终止；不能跳过必需的前置路线段 |

命令解析和语义计划不携带整份健康快照。执行端在 API 返回后重验取消、定位、实际 pose 是否移动；Mission callback 重读 AUTO / safety 和新鲜度，原碰撞保护保留。

模型无回答/回答非法不能变成 execute。未配置 API 时沿用透明 local fallback，trace 不计作 live。Shadow 记录判断但不采纳模型改变，不额外产生运动。

2026-10-12 暂停语义修正：computed_path 审查只判断目标/路线规则一致性，不要求 JEV 从未提供的原始传感器判断净空。视觉确认是到达后的依赖子 Task，语义 plan_review 的 skills 也包含它。JEV 的 execute 不是安全许可，执行器仍检查取消、AUTO、安全、定位、位姿变化和碰撞条件。

审查返回 NEEDS_HUMAN 是明确的发布前暂停，不重复进入 navigation_step_supervision。Dashboard 提供原 Mission/current Step 的重新检查与 Cancel all；重新检查会重新求路径和审查。未成功到达时不能提供 Continue: search_object。真正的未处理 skill 失败仍进入异常监督。阶段原始判断永久保留，但已重试成功/任务终止的旧等待只作为历史，不显示为当前确认请求。

## 3. Robot Skill 顺序

默认计划：
1. resolve_semantic_target
2. await_robot_stationary
3. set_camera_pose（导航相机归位并验证反馈）
4. await_robot_auto_mode
5. check_localization_health
6. goto_pose
7. 可选 search_object 子 Task（由 Perception 拆解，不把 9 张采集摊到 Mission 顶层）

不是固定“7 个 API 调用”。第 6 步执行前才有实际路径审查；包含窄道或有界重试时可新增 computed_path review。它们是计划审查，不是异常 step_supervision，也不是每个控制周期请求模型。

现阶段具体参数继续使用既有容差和导航策略。没有降低定位/碰撞要求，也没有把语义容差成功混为 Nav2 严格成功。

## 4. 规则配置与限制

规则入口是当前加载的 semantic_map.yaml 顶层 navigation_rules。它随地图加载，不自动监听磁盘热更新；修改后需要重新加载对应进程。空列表保持原行为。

```yaml
navigation_rules:
  - id: front_door_maintenance
    revision: 1
    target_ids: [door_front]
    effect: deny_destination
    message: Front door destination is closed for maintenance.
    valid_from: "2026-10-12T00:00:00+09:00"
    valid_until: "2026-10-13T00:00:00+09:00"
  - id: shelf_identity
    revision: 1
    target_ids: [shelf_06]
    effect: require_visual_confirmation
```

| effect | 实际含义 |
| --- | --- |
| deny_destination | 禁止以指定实体为目的地，代码直接拒绝，JEV 无权豁免 |
| require_visual_confirmation | 到达后必须确认身份；JEV 不能删除确认要求 |
| advisory | 提供语义判断参考，不是可执行禁区或硬安全约束 |

target_ids 为空表示当前语义地图所有目标。时间为带时区的现实 UTC 时间，不是 sim clock；省略有效期表示持续有效。时间格式、规则结构不合法或未知 effect 都先澄清，不忽略。

硬拒绝优先，不提供优先级数字覆盖禁令。当前不支持通过自然语言文件自动生成禁区、单行道、宽度限制或维修绕行；未知 effect 返回 RULE_BLOCKED。**deny_destination 不等于禁止路径经过该门**，后者必须加入 planner/costmap 的几何约束后才可启用。

显式 approach relation 只接受已登记 approach_poses.id；不允许旧的“没找到就取第一个”回退。region 的显式接近面约束尚未落实，先返回 UNSUPPORTED_APPROACH。已明确接近面的任务禁止换面和放宽容差。

## 5. 路径审查能证明什么

输入路径确实来自 Nav2 ComputePathToPose，而非固定字符串 existing_validated_nav2_policy。path_id 为实际点序列摘要；对模型只发最多 32 点和确定性算出的长度、标注，不发整幅 costmap。

它是一次全局路径及既有策略的审查，**不是足迹净空证明，也不锁定 NavigateToPose 后续重规划的轨迹**。碰撞由现有 Nav2 / 直接微调保护执行。自由几何硬规则、动态禁区需要持续 planner/filter 验证，本轮没有伪装为已实现。

每次 goto_pose dispatch，包括内部递归段，重新检查；不在每次局部规划 tick 调 JEV。未来若要重用审查结果，必须绑定目标、路径、地图、规则与证据 revision，不能只按命令文本缓存。

## 6. 到达和感知反馈

navigation_arrival 在 goto_pose 返回后立即保存，因此后续 OCR 出错也不会抹去位置到达事实；记录 reached、completion_class、误差与原 result_step_id，避免复制大 metrics。

visual_confirmation：NOT_REQUIRED / CONFIRMED / UNCONFIRMED。
完成的 Perception 搜索没有找到对象时不是执行故障，但需要确认身份的导航 Task 为 UNMET，Mission 可据依赖关系决定下一个 Task，而不是假称全部成功。
仍分别统计 nav2_strict / task_tolerance / degraded / dry_run / not_successful；缺证据为 unclassified，不能推断为 strict。

## 7. 维护与验证

问题源和继承规则见 [统一 Skill 监督目录](../skill-supervision-guide.md)。导航领域问题在 navigation_agent.py，通用失败问题和各 skill 问题都在 skill_catalog.v2.yaml；无需去每个 Agent 再找一份同名 skill 问题。

当前回归覆盖：缺地图、多门消歧续行/取消、维修规则、失效时间、未支持规则、显式接近关系、Primary/Shadow 差异、实际路径摘要、递归段门控、API 后取消/定位变化、到达但 OCR 未确认。详见 [问题审计](runtime-question-audit-20261012.md)。

待现场验证：真实 API 延迟下的路径检查、Full sim 长距离导航/窄道、重定位后续行。
待扩展：多条真实候选路线比较、路径净空度量、规则热更新和执行中变更停止、领域任务独立 TaskRunner。没有独立 TaskRunner 不影响本轮 owner/Task/Step 分工，但也不表示任意动态重规划已实现。
