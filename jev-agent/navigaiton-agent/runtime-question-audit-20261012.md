# Navigation Agent 实际检查与问题审计

日期：2026-10-12。下方旧运行记录用于解释当时的问题；本节为后续代码更新，不能将旧 live 记录当作新代码实测。

## 09:18 UTC Session 后续检查

最新修正与证据见 [Dashboard/Agent 协作回归](../mission-agent/test&pro&solve/12-dashboard-agent-review-20261012.md)。本轮新增 blocking_issue 问题、到达后视觉确认时机以及原 Step 的人工重试。路径审查暂停不再唤醒异常监督。下方 372/8 是前一轮结果；本轮为 380/8，8 项为同一组既有测试问题。

## 本轮更新：实际问题来源

当前执行链已改为 navigation_command_parse -> navigation_plan_review(semantic) -> 执行前 navigation_plan_review(computed_path)；未处理异常才 navigation_step_supervision。完整输入和合法选项见 [当前 README](README.md#2-阶段输入与问题)。

| 检查 | 当前问题 | 定义源 |
| --- | --- | --- |
| 目标解析 | goal_binding / goal_kind / visual_confirmation | agents/navigation_agent.py::refine |
| 语义计划审查 | decision：execute / ask_human | 同上 |
| 实际路径审查 | decision / route_strategy / blocking_issue；只能使用当前已实现策略 | navigation_agent.py::review_path；navigation.py::_review_before_motion |
| 失败原因和后续动作 | common_supervision_questions | skill_catalog.v2.yaml |
| 当前 skill 证据 | supervision.questions；证据字段白名单 | 同一 catalog，alias/composite 继承；不另藏在 Navigation |

同名多门的歧义在代码已明确时直接问人；候选值写入原 Task 的 navigation_selected_target，旧的 command='go to ...' 不再使用。未知/未标定目标不发运动。

请求参数和返回仍在 jev_calls.jsonl，stage/phase 区分两次 plan_review；trace 绑定 task_id/step_id。path_id 是真实路径摘要，但采样路径不是实时净空保证，无法据此声称动态禁区已生效。

本轮测试使用 mock，不调用真实 API、不移动机器人。已验证各注册 skill 问题进入请求、同 Mission 选择/取消、缺地图、规则拦截和有效期、显式接近限制、路径 API 后取消/定位变化、OCR 未确认时保留到位事实。

本轮回归：robot_agent/test + robot_skills/test 共 372 passed、8 failed。8 项均在 test_real_lab_parser_semantic.py：6 项依赖缺失的 maps/l402_2d_map_clean_0509 地图文件，2 项旧相机标定断言与现有配置不一致；未修改真实标定或补造地图来通过测试。其余 361 项全通过，该旧文件另有 11 项通过。

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
PYTHONPATH="$PWD/src/robot_agent:$PWD/src/robot_skills:$PYTHONPATH" python3 -m pytest -q src/robot_agent/test src/robot_skills/test --tb=short
```

仅 source install/setup.bash 的 Python 导入也已验证：新的 navigation_plan 模块可见，34 个注册 skill 的 catalog_errors=[]。需重启 Agent 进程才加载新 Python 逻辑；本轮没有自动重启运行中的 robot 栈。

## 历史运行记录（修改前）

## 1. 当前 Task 与 Step

`navigate_to(shelf_06)` 由 Mission 在执行时委派 Navigation，当前展开为：
1. resolve_semantic_target
2. await_robot_stationary
3. set_camera_pose（导航相机归位、反馈验证）
4. await_robot_auto_mode
5. check_localization_health
6. goto_pose
7. search_at_place（来自嵌套 Perception search_object Task）

最后两条 task_end 是父子 Task 的内部汇总标记，不是 Robot Skill/Step。UI 之前隐藏标记但总数仍加了它们，导致 7/9；现仅按 kind=skill 编号和计数。Task 深化是动态的，未深化 Task 不应伪装成已经规划出的 Skill。

源码：`src/robot_agent/robot_agent/agents/navigation_agent.py`、`agents/mission_agent.py`；执行器 `src/robot_skills/robot_skills/executors/navigation.py`；实体/approach 来源实际地图 semantic_map.yaml，通用解析 `robot_skills/semantic_map.py`。

## 2. 修改前 JEV 唯一领域问题

Stage: `nav_target_review`。

```json
{
  "state": {
    "stage_goal":"nav_target_review",
    "task":{"task_type":"navigate_to","target":"shelf_06","relation":null,"constraints":{}},
    "evidence":{"entities":[{"id":"shelf_06","name":null}],"route_policy":"existing_validated_nav2_policy"}
  },
  "questions": {
    "decision": {"type":"choice","criteria": {
      "execute":"Use the uniquely resolved registered destination and existing navigation policy.",
      "ask_human":"Requested relation or destination needs clarification."
    }}
  }
}
```

真实 Task 还包含 raw_text 等字段；共同说明来自 `jev_agent.py::domain_review`，禁止虚构坐标/模型，只选提供的策略。上述 JSON 用于解释内容，完整 wire format 请看 jev_calls.jsonl。

确定性先检查 target 是否存在、SemanticMap 是否能唯一解析/标定；多候选/未标定时先返回人工选项，不发送运动。Primary 采纳 choice；Classic 不调用；Shadow 不改变执行。

**当前 JEV 没有审查 Nav2 的实际路径、窄通道净空、临时维修规则或接近朝向完整性。** `existing_validated_nav2_policy` 是策略标签，不是路线已通过安全证明。输入实体只有 id/name，语义信息仍偏少。

## 3. 两次历史任务的实际问题

session `20261012T015308Z-d41d5959`：
- `2ab225e70c8a`：4 次 goto_pose 失败/替代点重试后最终 degraded，到达后 OCR 未检出。
- `e4de9fe1352d`：4 次失败后 task_tolerance 到达（Nav2 cancelled_after_task_tolerance_reached），不是严格 Nav2 SUCCEEDED；随后 OCR 未检出。
- 部分内部微调报告 swept_footprint_blocked；不能放宽碰撞门来追求成功。
- 同时存在旧、新两套 Agent，每套都会收到同一 text_command，竞争 Nav2 goal 的风险真实存在。清除了旧套后只有一个 mission_state 发布者。重复实例是计划跳动/旧恢复记录的重要原因，但不能仅据此把所有历史 abort 都归为抢占。

## 4. 为什么 step_supervision 多次唤醒

每次 goto_pose 失败后都调用了 JEV，然后确定性分支换 approach 重试；每次未检出也被编码为 NEEDS_HUMAN，引发一次监督。所以每条历史导航有 4 次导航失败监督 + 1 次搜索监督，并不是顺利执行时的周期检查。

这 9 个问题来自 Mission 的 supervise_step，而非 Navigation 的 nav_target_review：
| ID | 类型 | 用途 |
| --- | --- | --- |
| step_completed_normally | noul | 当前执行是否正常完成 |
| step_result_reasonable | noul | 当前结果是否合理 |
| failure_category | choice | 错误分类 |
| needs_recovery_agent | noul | 是否需要健康恢复 |
| recovery_target | choice | 需要恢复的能力/组件 |
| failure_blocks_current_subgoal | noul | 是否阻断当前目标 |
| next_subgoal_still_meaningful | noul | 后续独立目标是否有意义 |
| continue_requires_human | noul | 是否需人工决策 |
| supervision_action | choice | 重试、独立继续、恢复、人工、终止等受限动作 |

输入是当前 Skill/result、task graph/plan window、能力契约和相关安全健康摘要，不包含图像推理。

修复后，已有有界 Navigation 重试先处理并写 step_supervision_skipped；只有未处理异常升级到 JEV。NOT_FOUND 作为完成的结果，不进入异常监督。jev_supervision_decision 的 action 明确是 proposal_pending_execution_policy（Shadow 为 shadow_only），不能把回答 call_recovery_agent 当作恢复确实执行的证据。真正恢复应有 request_id、Recovery 中间/最终响应和执行事件。

## 5. 后续优先补充

- 显式路由选择/净空与规则数据输入，不让“策略名称”替代路径证据。
- 本轮已禁止候选耗尽后重新尝试已失败原点；候选有效期/远端路线审查仍待补，不应无检查跳到远处另一端。
- 同 Task 对不同失败的局部决策与重试预算归属，最终从 Mission 的兼容执行分支收敛到 Navigation 内部。
- Arrival 合约：导航到位成功与视觉身份确认未满足分别报告。
- 人工澄清的原 Task 内恢复，而非依赖新发 command 重建整个任务。

Recovery PB 优先级：RTF 约 0.4 的 advisory、sim/real 相机分辨率不同，不自动等价于“必须先修才能 OCR/导航”。必须先处理的是急停、控制权错误、真实定位失效、碰撞/数据过期。RGB/depth 不同步在深度驱动的 costmap/3D操作中有关，单独 RGB OCR 不应因此被阻断。CPU 持续高温需留意，但不证明任务算法坏了。

## 6. 本轮在线复测

单实例 Full + JEV Primary，`5f21e132df25`（新 session `20261012T022650Z-4f03f3df`）：go to shelf_06 -> task_tolerance 到达 -> 九视角 OCR FOUND 06 -> 父子 Task SATISFIED；0 次 step_supervision、0 次替代目标重试、0 次 Recovery 调用。实测页面 7/7、无 JS 异常。
这是当前位置附近的到达/视觉确认协作验收，不是长距离或严格 Nav2 成功验收。
