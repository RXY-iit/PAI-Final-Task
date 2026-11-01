# Arm Agent：确定性第一版与后续设计

更新：2026-10-11。本轮扩展现有 ArmAgent，不新增平行进程、不接 Arm JEV。共享术语以 [Task / Skill](../task-skill-redefine.md) 和 [Mission](../mission-agent/README.md) 为准。

## 1. 职责与当前能力

Arm 收到完整操作目标，负责具体动作、目标来源、参数、前置条件和完成证据。Mission 不把抓取/搬运/释放拆成多个用户 Task，也不猜像素坐标或关节姿态。

| Task | 当前处理 | 未完成部分 |
| --- | --- | --- |
| set_arm_pose | 静止检查 -> 既有预设姿态/手势；别名 pose_label/name；等待 gateway 状态 | 实际 sim 取消/姿态到位仍需本轮后续测试 |
| interact_with_object(operation=touch/press) | 明确 UNSUPPORTED/NEEDS_HUMAN，不先扫描/运动再失败 | Touch Anything 现有接口主要入队；相机来源、执行关联 ID、物理完成/接触证据不足 |
| manipulate_object(desired_end_state=held/placed_at/released) | 一个 Task，明确 UNSUPPORTED | 无完整策略、持物状态、放置/释放验证，不拼占位 skill 冒充能力 |
| move_lift | 自主 Task 暂拒绝，提示受监督控制 | 现有 bounded 技能返回 COMMAND_ACCEPTED，计时 STOP 不能证明目标高度已达 |

旧 touch/press/move_lift 的低层接口保留兼容，不代表通过新自主任务验收。没有把它们删成“无法解析的命令”；Task 仍被识别并有 owner/能力不足反馈。

## 2. 源文件与取消

- [arm_agent.py](../../../src/robot_agent/robot_agent/agents/arm_agent.py)：refine 的能力边界、预设计划；旧 build_plan 保留兼容。
- [arm.py](../../../src/robot_skills/robot_skills/executors/arm.py)：预设、Touch command、gateway 状态与取消。
- [arm_gateway.py](../../../src/so101_arm_control/so101_arm_control/arm_gateway.py)：既有轨迹/visual servo 控制。
- [touch_observation_node.py](../../../src/touch_anything/scripts/touch_observation_node.py)：已有视觉/接近流程，不等于统一 TaskResult。
- [skill_extensions.yaml](../../../src/robot_skills/config/skill_extensions.yaml)：set_arm_pose、touch_target、press_target、move_lift 别名。
- [capabilities.py](../../../src/robot_agent/robot_agent/capabilities.py)：check_arm_capability，保留 supported/available/ready 的 unknown。

本轮取消加入 Touch exit、visual_servo_stop、gateway cancel；不附带 release_hold。等待循环/连续预设间检查 run 的取消状态。发布者不存在时不再在非 dry_run 环境伪报 dry-run 成功。
这些是停止请求与软件调度保证，不替代底层 action 取消确认及真实关节静止测量。独立 cancel_arm_motion Skill 暂不注册。

## 3. 之后补齐的确定性闭环

1. 明确目标 evidence 的相机、时间、图像 ID、坐标系、标定版本。禁止把底盘 RGB bbox 直接交给 arm-camera 操作。
2. Arm 委派 Perception 获取实例/交互点，结果必须满足所需坐标/置信范围；不满足则澄清或补证。
3. 已授权动作策略决定接近方向、关节/速度/作用范围；IK、限位、碰撞、硬件安全独立验证。
4. 每次操作建立 action ID 与进度/终态回传，区分 accepted/running/completed/failed/cancelled。
5. 完成由目标契约判定：姿态到位、接触证据、按钮状态、持物/放置状态分别验证；无传感器不声称验证施力。
6. manipulate_object 内部自行编排抓取/搬运/放置/释放。仅 held 目标不附带放置，仅 released 不默认移动到未知位置。
7. 失败保留手臂/持物状态、动作历史和证据。向 Mission 报告，不在领域重复创建 Recovery 请求。

缺少可验证闭环的动作维持 UNSUPPORTED。将简单预设做好比添加一条能够发消息但无法取消/验收的“能力”更重要。

## 4. 后续 JEV（未接入）

| 决策阶段 | 最小输入 | 有限选项与代码边界 |
| --- | --- | --- |
| interaction_goal_review | 当前 Task、目标词、操作规则、用户约束 | touch/press/hold/place/release 的已登记语义或 ask_human；不得扩展原目标 |
| target_evidence_review | 目标实例、图像/坐标来源、缺失证据 | use_validated_target / request_perception / ask_human；不得生成坐标 |
| interaction_strategy_review | 可达性检查结果、已批准策略与资源约束 | 选择 policy ID / unsupported / ask_human；不可覆盖 IK/碰撞拒绝 |
| interaction_result_review | 原目标契约、真实动作/物理证据、已尝试动作 | finish / bounded_reobserve / ask_human / report_failure；不能将 ACK 判为成功 |

先完成确定性反馈与故障边界，再加这些判断。JEV 无法补出缺失的传感器/抓取策略。预设姿态不应每次额外调用 JEV。

## 5. 下一轮重点测试

- set pose home：一个 Arm Task，静止检查与姿态完成有归属，Mission 只执行一次。
- look left then set pose home：Perception -> Arm，证据不串用，模式在整次 Mission 冻结。
- 预设等待中 Cancel：无下一动作，无旧命令复活；检查实际 gateway/action 停止。
- touch blue dot / pick up box / lift up：识别对应 Task，但能力不足清楚反馈且没有自主运动。
- gateway 断开：明确失败而非 DRY_RUN_OK；恢复后不自动重放接触动作。

本轮只有 mock/dry-run 回归，未操控机械臂验收。Recovery 按用户要求后置。
