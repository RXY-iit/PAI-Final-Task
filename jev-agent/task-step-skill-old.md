# Task / Step / Robot Skill 总清单

核对日期：2026-10-10。范围：当前工作区代码、配置及三个 Agent 的最新设计文档。本文是索引和实现状态快照，不是新的运行时注册表，也不表示所有登记能力已经通过实机验证。不列举 Mission，因为 Mission 是一次用户请求及其生命周期。

## 1. 先区分层级

| 概念 | 本文含义 | 当前是否有独立定义 |
| --- | --- | --- |
| Task Type | 可选择的任务类型，如 `search_object_here` | 有分散的代码分支和解析模板；**没有统一 Task Catalog** |
| Task | 一个类型的实例，带目标和参数 | 当前主要由 `MissionIntent` 表达；新的 TaskContract/TaskResult 尚未实现 |
| Mission Step | 用户任务图里的一个 Task 节点 | 最新设计已明确；当前执行计划仍扁平化为 SkillCall |
| Task Step | Task 内部的动作、检查或判断节点 | 当前由各 Agent 的 plan builder 展开；**没有独立 Step Catalog** |
| Robot Skill | 执行白名单里的函数契约 | 有 registry、catalog、dispatcher 和 executor |
| SkillCall | 一次 Robot Skill 调用 | 已有数据类；同一 skill 可以在一个 Task 中调用多次 |
| JEV Stage | 判断阶段，如 plan_review | 不是 Robot Skill，也不直接等于一个用户 Task |
| Recovery Step | playbook 内的动作、人工请求或等待证据 | 有独立 PB 定义；不是 Mission 可选 Task，也不等于 SkillClient 的 skill |

Agent Skills package 指知识和工作流程的可复用包规范，与上述 Robot Skill 不同。

**当前数量：20 种普通可建计划 Task 类型，另有 sequence 编排类型和 ask_human 澄清结果；19 个注册 Robot Skill，其中 18 个出现在 Agent 的静态计划构建调用中。** 参数、对象、重复次数产生的所有实例无法穷举，本文列全类型、现有展开模板及其来源。

## 2. 定义入口

以下路径均相对仓库根目录；链接相对本文位置。定位代码时以类/函数/配置键为准，不依赖可能变化的行号。

| 内容 | 源文件与定义 |
| --- | --- |
| 共享术语、未来任务协议 | [mission-agent/README.md](mission-agent/README.md)，§1、§4、§5 |
| 当前任务与步骤数据结构 | [src/robot_agent/robot_agent/schemas.py](../../src/robot_agent/robot_agent/schemas.py)：`MissionIntent`、`SkillCall`、`MissionState.plan` |
| Task 到执行计划的主分派 | [src/robot_agent/robot_agent/agents/mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py)：`_build_plan`、`_build_sequence_plan` |
| 自然语言模板 | [src/robot_agent/config/parser_templates.yaml](../../src/robot_agent/config/parser_templates.yaml)：`templates[].intent.task_type`，包含下表全部 20 种普通类型 |
| 确定性解析 | [src/robot_agent/robot_agent/parsers/deterministic_parser.py](../../src/robot_agent/robot_agent/parsers/deterministic_parser.py)：解析入口及各专用解析函数 |
| JEV 可选 Task 类型与 slots | [src/robot_agent/robot_agent/jev_agent.py](../../src/robot_agent/robot_agent/jev_agent.py)：`_supported_task_type_criteria`、`_supported_task_types`、`_task_type_slots`、`_intent_for_jev_task` |
| Robot Skill 执行白名单 | [src/robot_skills/config/skill_registry.yaml](../../src/robot_skills/config/skill_registry.yaml)：`skills` |
| Robot Skill 契约 | [src/robot_skills/config/skill_catalog.v2.yaml](../../src/robot_skills/config/skill_catalog.v2.yaml)：`robot_skills`、`precondition_atoms` |
| Skill 配置加载 | [src/robot_skills/robot_skills/registry.py](../../src/robot_skills/robot_skills/registry.py)、[skill_catalog.py](../../src/robot_skills/robot_skills/skill_catalog.py) |
| Skill 分派与参数传递 | [src/robot_agent/robot_agent/skill_client.py](../../src/robot_agent/robot_agent/skill_client.py)：`SkillClient.execute_skill` |
| 新 Task Catalog / TaskContract / TaskResult / 领域 TaskRunner | **尚无独立实现文件**；目前仅有三个 Agent README 的设计 |

`MissionIntent.task_type` 目前是 `str`，不是集中约束的 Enum。一个名字出现在设计文档、parser 模板或 registry 中，不能单独证明它已经有完整执行闭环。

## 3. 当前所有普通 Task 类型

“JEV 可选”仅指 `_supported_task_types()` 显式提供该类型；不保证所有措辞/参数都能解析，也不表示没有列出的类型绝对不会经其他 fallback 到达。“计划来源”的文件就是当前定义位置，不是未来拟议文件。

表中 A/S/N/L/Q/H/C/G 为下一节定义的展开模板，不是注册名称。

| # | Task Type | 含义与主要参数 | 当前计划展开 | JEV 可选 | 计划来源 |
| --- | --- | --- | --- | --- | --- |
| 1 | `robot_status` | 查询机器人状态 | `get_robot_status` | 是 | [mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py)：`_build_plan` |
| 2 | `list_known_targets` | 查询已知语义地点/对象 | `list_known_targets` | 否 | 同上：`_build_plan` |
| 3 | `cancel` | 取消运动/任务的控制意图 | `cancel_motion`；另有运行时取消入口 | 是 | 同上：`_build_plan`，取消处理不能只看此静态计划 |
| 4 | `move_lift_bounded` | 升降；direction、distance_m、duration_sec | 静止检查 -> `move_lift_bounded` | 否 | 同上：`_build_plan` |
| 5 | `move_relative` | 相对平移；direction、distance_m | A -> `move_relative` | 是 | 同上：`_build_plan`、`_relative_motion_preflight` |
| 6 | `move_relative_xy` | 相对位移；dx_m、dy_m | A -> `move_relative_xy` | 是 | 同上 |
| 7 | `rotate_relative` | 相对旋转；yaw_deg | A -> `rotate_relative` | 是 | 同上 |
| 8 | `look_direction` | 看指定方向；direction、band、可选 object_query | S -> L | 是 | [perception_agent.py](../../src/robot_agent/robot_agent/agents/perception_agent.py)：`build_look_direction_plan`；Mission 加前置 |
| 9 | `search_object_here` | 当前位置搜索/观察；object_query | S -> Q | 是 | 同上：`build_search_plan`；Mission 加前置 |
| 10 | `navigate_to_place` | 去已知地点；target、relation | N；部分目标追加到达视觉确认 Q | 是 | [navigation_agent.py](../../src/robot_agent/robot_agent/agents/navigation_agent.py)：`build_plan`；Mission 决定追加确认 |
| 11 | `navigate_to_object` | 去已知语义对象附近；target、relation | N；部分目标追加 Q | 是 | 同上；当前与 navigate_to_place 共用主要计划 |
| 12 | `inspect_place` | 到指定地点并检查其标签/目标；target | N -> Q（query 来自目标名） | 是 | [mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py)：`_build_plan`；Perception `build_plan` |
| 13 | `search_object_near_place` | 去某处找对象；target、object_query | N -> Q（query 来自 object_query） | 是 | 同上 |
| 14 | `set_pose` | 手臂预设姿态/手势；target 或 pose、repeat | H -> `set_pose` | 是 | [arm_agent.py](../../src/robot_agent/robot_agent/agents/arm_agent.py)：`build_plan`；Mission 加前置 |
| 15 | `touch_object` | 接触当前对象；object_query 或 target | H -> C(mode=touch) | 否 | 同上：`build_plan`、`_touch_plan` |
| 16 | `press_object` | 按压当前对象；object_query 或 target | H -> C(mode=press) | 否 | 同上；没有独立 press Robot Skill |
| 17 | `grasp_object` | 抓取；目标、可选 destination | H -> G；执行端仍为占位 | 否 | 同上：`build_plan` |
| 18 | `touch_object_near_place` | 去指定地点后接触对象；target、object_query | N -> H -> C(mode=touch) | 否 | Mission `_build_plan` + Arm `_touch_plan` |
| 19 | `press_object_near_place` | 去指定地点后按压对象 | N -> H -> C(mode=press) | 否 | 同上 |
| 20 | `grasp_object_near_place` | 去指定地点后抓取对象 | N -> H -> G；抓取端仍为占位 | 否 | Mission `_build_plan` + Arm `build_plan` |

最后三行的完整源文件仍为 [mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py) 和 [arm_agent.py](../../src/robot_agent/robot_agent/agents/arm_agent.py)。复合类型是现有 API；未来是否拆成 Navigation Task + Arm/Perception Task 需要显式迁移，本文不将它们重复计数。

### 3.1 编排、澄清和非任务值

| 名称 | 性质 | 定义位置 / 状态 |
| --- | --- | --- |
| `sequence` | 多任务容器，不是新机器人能力；`constraints.steps` 含 1-3 个子 intent；当前展开成同一个平面 SkillCall 列表 | [mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py)：`_build_sequence_plan`；[deterministic_parser.py](../../src/robot_agent/robot_agent/parsers/deterministic_parser.py) |
| `ask_human` | 澄清/拒绝直接执行的解析结果；不进入普通 skill 计划 | Mission 运行入口；[jev_agent.py](../../src/robot_agent/robot_agent/jev_agent.py) 与各 parser |
| `not_applicable` | JEV 问题中的“不适用”选项，不是已注册执行 Task | JEV `_supported_task_type_criteria`；`_supported_task_types` 将其排除 |

JEV 显式类型列表目前缺 8 项：list_known_targets、move_lift_bounded 和六个 touch/press/grasp 类型。另一个独立列表是 [llama_cpp_parser.py](../../src/robot_agent/robot_agent/parsers/llama_cpp_parser.py) 的 13 种允许类型，不含七个手臂类型和 sequence。**三种解析路径的能力范围尚未统一由一份 Task Catalog 驱动。**

## 4. 当前所有 Step 展开模板

当前 `MissionState.plan` 中每个条目都是 `SkillCall`，并没有另外注册一组 Step 名称。因此可枚举的是下面的模板及其调用类型，而不是每条命令产生的所有序号。18 种已用于建计划的 skill 名称见 §5；唯一未用于这些 builder 的注册项是 `recover_localization_spin`。

### 4.1 前置模板

| 代号 | 展开内容 | 源文件 / 函数 |
| --- | --- | --- |
| A | `await_robot_auto_mode` | [mission_agent.py](../../src/robot_agent/robot_agent/agents/mission_agent.py)：`_relative_motion_preflight` |
| S | `await_robot_stationary`，settle=1s、模板 timeout=180s | 同上：`_perception_preflight` |
| H | `await_robot_stationary`，settle=1s、模板 timeout=180s | 同上：`_arm_preflight` |
| 升降前置 | `await_robot_stationary`，settle=1s、模板 timeout=30s | 同上：`_build_plan` 的 move_lift_bounded 分支 |

这里列的是计划参数，不替代 executor 内部的持续安全检查、进展判断及预算策略。相对运动前置没有 check_localization_health；Perception 前置没有 await_robot_auto_mode。

### 4.2 N：导航模板

定义：[src/robot_agent/robot_agent/agents/navigation_agent.py](../../src/robot_agent/robot_agent/agents/navigation_agent.py)，`NavigationAgent.build_plan`。

| 内部序号 | Robot Skill | 作用 |
| --- | --- | --- |
| 1 | `resolve_semantic_target` | target_name/relation 解析为地图目标及候选 |
| 2 | `await_robot_stationary` | 在相机归位前确认静止 |
| 3 | `set_camera_pose` | return_home，要求反馈 |
| 4 | `await_robot_auto_mode` | 导航控制模式前置 |
| 5 | `check_localization_health` | 全局导航定位前置 |
| 6 | `goto_pose` | 使用解析后的 pose、候选和导航策略 |

部分目标由 Mission `_needs_visual_arrival_confirmation` 再追加 Perception `build_arrival_confirmation_plan`。目标名包含 shelf/棚/部分数字等会命中当前字符串规则；这不是独立注册的 `visual_confirmation` Task。Nav2 内部重规划/微调也不是额外登记的 Mission Step。

### 4.3 L：定向观察模板

定义：[perception_agent.py](../../src/robot_agent/robot_agent/agents/perception_agent.py)，`build_look_direction_plan`。

`set_camera_pose(direction, band) -> capture_observation -> [有 object_query 才 detect_known_object] -> set_camera_pose(return_home)`。

不含前置是 3 或 4 个调用；加 S 后是 4 或 5 个。`look left` 与底盘 `turn left` 是不同 Task，不能混为同一个 Robot Skill。

### 4.4 Q：搜索模板及全部固定视角

定义：[perception_agent.py](../../src/robot_agent/robot_agent/agents/perception_agent.py)，`build_search_plan`、`_is_fully_unknown_query`、`_provider_for_query`。

| query 情况 | 实际展开 |
| --- | --- |
| 普通可尝试识别的目标 | 九视角各 set_camera_pose + capture_observation，之后归位，再 detect_known_object：20 次调用 |
| around / surroundings / 周り / 周囲 | middle_left、middle_center、middle_right 各转动+拍照，之后归位，不做检测：7 次调用 |
| 当前 analyzer 判定完全未知/abstract/unsupported | 只调用 detect_known_object 报告不可识别/需要人工；不先转动拍九张 |

普通九视角模板的**现有顺序**如下；角度是代码中的逻辑输入，实际硬件映射由相机 executor/config 处理。

| 调用序号（不含 S） | view / scan_band | pan_deg | tilt_deg | 操作 |
| --- | --- | --- | --- | --- |
| 1-2 | ground_left | -35 | 80 | set_camera_pose -> capture_observation |
| 3-4 | ground_center | 0 | 80 | 同上 |
| 5-6 | ground_right | 35 | 80 | 同上 |
| 7-8 | middle_left | -35 | 25 | 同上 |
| 9-10 | middle_center | 0 | 25 | 同上 |
| 11-12 | middle_right | 35 | 25 | 同上 |
| 13-14 | upper_left | -35 | -20 | 同上 |
| 15-16 | upper_center | 0 | -20 | 同上 |
| 17-18 | upper_right | 35 | -20 | 同上 |
| 19 | return_home | 0 | 67 | set_camera_pose |
| 20 | 汇总识别 | 不适用 | 不适用 | detect_known_object(query, provider) |

`search_object_here` 加 S 后共 21 个 SkillCall，但设计上仍只有一个 Task。`inspect_place` / `search_object_near_place` 是 N + Q；不能假设它们自动再加 S，当前组合以 Mission 分支为准。失败/取消可能提前结束，计划调用数不是实际成功调用数。

`PerceptionAgent.build_plan` 还保留通用 fallback：capture_observation -> 可选 detect_known_object。它是 helper 分支，**没有另外登记一个同名 Task**。

### 4.5 手臂模板

定义：[arm_agent.py](../../src/robot_agent/robot_agent/agents/arm_agent.py)，`build_plan`、`_touch_plan`、`_normalize_pose_name`。

| 模板 | 展开 | 限制 |
| --- | --- | --- |
| 预设姿态 | set_pose(name, repeat) | home/see_front/see_left 等是参数值，不是各自的 Robot Skill |
| C 接触/按压 | set_pose(see_front) -> capture_observation -> detect_known_object(provider=yolo11s_ocr) -> touch(mode=touch 或 press) | 当前 requires_human_confirm 标记在 touch 调用；物理完成证据仍需单独核实 |
| G 抓取 | grasp(target, destination) | 已有计划和接口，但执行器没有配置实际 GR00T 策略端点 |

当前 helper 名为 ArmAgent；新版三 Agent 职责设计并不意味着该现有模块已删除。

## 5. 全部 19 个 Robot Skill

每一项均有两个配置定义路径：

- 白名单 **R**：[src/robot_skills/config/skill_registry.yaml](../../src/robot_skills/config/skill_registry.yaml)，查 `skills[].name=<下表名称>`。
- 契约 **C**：[src/robot_skills/config/skill_catalog.v2.yaml](../../src/robot_skills/config/skill_catalog.v2.yaml)，查 `robot_skills` 下同名条目。

统一分派路径：[src/robot_agent/robot_agent/skill_client.py](../../src/robot_agent/robot_agent/skill_client.py)，`execute_skill`。下表参数仅为阅读索引，完整参数/默认值应对照 C、dispatcher 和函数签名，不能仅凭此表调用机器人。

| # | Robot Skill / R、C 同名定义 | 作用 / 主要参数 | executor 源文件及函数 | 当前注意事项 |
| --- | --- | --- | --- | --- |
| 1 | `get_robot_status` | 状态快照；无业务参数 | [executors/system.py](../../src/robot_skills/robot_skills/executors/system.py)：`SystemSkills.get_robot_status` | 查询，不是持续监控节点 |
| 2 | `await_robot_auto_mode` | 等待 AUTO；timeout_sec | 同上：`await_robot_auto_mode` | 注册允许 sim/real；不代表自动切 AUTO |
| 3 | `await_robot_stationary` | 等待静止；settle_sec、timeout_sec | 同上：`await_robot_stationary` | 实际运动证据门槛 |
| 4 | `check_localization_health` | 定位检查；settle_timeout_sec、allow_warmup_pass | [executors/localization.py](../../src/robot_skills/robot_skills/executors/localization.py)：同名函数 | 不负责主动重定位 |
| 5 | `recover_localization_spin` | 遗留恢复；timeout_sec、angular_z | 同上：同名函数 | 名称过时；实际停止并发布可信 pose + odom 种子，静止验证，不旋转；real 禁用 |
| 6 | `resolve_semantic_target` | target_name、relation -> 目标及接近信息 | [executors/semantic.py](../../src/robot_skills/robot_skills/executors/semantic.py)：同名函数 | 不是移动动作 |
| 7 | `list_known_targets` | 列出语义目标 | 同上：同名函数 | 与同名 Task 属于不同层 |
| 8 | `goto_pose` | pose、tolerance、timeout、recovery_policy 等 | [executors/navigation.py](../../src/robot_skills/robot_skills/executors/navigation.py)：`NavigationSkills.goto_pose` | 包含复杂导航执行逻辑，不是原子电机操作 |
| 9 | `move_relative` | direction、distance_m、timeout_sec | 同上：同名函数 | 当前走直接 odom 相对运动，不依赖全局定位通过 |
| 10 | `move_relative_xy` | dx_m、dy_m、timeout_sec | 同上：同名函数 | 同上，仍受自身安全门保护 |
| 11 | `rotate_relative` | yaw_deg、timeout_sec | 同上：同名函数 | odom 角进度与安全/预算逻辑 |
| 12 | `cancel_motion` | 取消运动 | 同上：`cancel_navigation` | registry 名与 Python 函数名不同 |
| 13 | `capture_observation` | 采集图像及 scan/view 元数据 | [executors/perception.py](../../src/robot_skills/robot_skills/executors/perception.py)：`PerceptionSkills.capture_observation` | 图片证据，不等于检测成功 |
| 14 | `detect_known_object` | query、provider、min_confidence 等 | 同上：`detect_known_object` | query 分析与模型 pipeline 在此，非新 Task 类型 |
| 15 | `set_camera_pose` | pan_deg、tilt_deg、scan_band、反馈策略等 | [executors/camera.py](../../src/robot_skills/robot_skills/executors/camera.py)：同名函数 | 云台姿态，不是手臂 set_pose |
| 16 | `move_lift_bounded` | direction、distance_m、duration_sec | [executors/lift.py](../../src/robot_skills/robot_skills/executors/lift.py)：同名函数 | 有界升降，具体限制见实现 |
| 17 | `set_pose` | 手臂 name、repeat | [executors/arm.py](../../src/robot_skills/robot_skills/executors/arm.py)：`ArmSkills.set_pose` | 预设姿态名是参数 |
| 18 | `touch` | target、mode=touch/press 等 | 同上：`touch` | COMMAND_ACCEPTED 表示命令入队，不直接证明接触已完成 |
| 19 | `grasp` | target、destination | 同上：`grasp` | **占位实现**，返回 NEEDS_HUMAN / 策略端点未配置；real 禁用 |

“19 个注册”不等于“19 个已经可用且验证过”。`recover_localization_spin` 是兼容接口，不在当前 Agent 静态计划中；PB-LOC-LOST 也明确不调用它。恢复动作在 Recovery backend 有独立实现。

### 5.1 已确认的定义漂移

- R 的 move_relative / move_relative_xy / rotate_relative 仍有 Nav2/定位依赖描述，而当前 executor 使用直接相对运动。Mission 部分 reason 也仍写 through Nav2。不能据这些旧说明推断实际门槛。
- R 的 recover_localization_spin 描述包含旋转，与当前静止恢复实现不一致。
- R 的 goto_pose 禁止行为描述与 executor 内已存在的直接速度 fallback 需要进一步统一契约；此处只标出差异，不视为允许绕过定位/碰撞门。
- 新设计将 Task 边界与内部执行分开，但 `schemas.py` 的 plan 仍是平面 SkillCall 列表。文档中“一个 Task”的定义不能被当作已经完成 UI/执行器迁移。

## 6. 设计中提出但尚未独立实现的 Step

这一节的名字是**设计阶段/模板**，没有独立注册的同名 Robot Skill；现有低层 helper 有部分对应功能，不等于新领域 TaskRunner 已实现。

| Owner | 设计中的内部阶段 | 定义路径 | 对应代码状态 |
| --- | --- | --- | --- |
| Mission | command_parse -> 任务级 slots -> plan_review -> Task 委派 -> 异常时 task_outcome_review | [mission-agent/README.md](mission-agent/README.md)，§3、§6 | 已有 JEV parse/review 等旧机制；新的 Task 图/结果协议和 task_outcome_review 闭环待实现 |
| Navigation | resolve_target -> resolve_arrival_contract -> propose_goal_candidates -> Nav2 plan -> analyze_route -> choose_allowed_strategy -> validate_plan -> execute_segment(s) -> verify_arrival -> optional_visual_confirmation | [navigaiton-agent/README.md](navigaiton-agent/README.md)，§3 | 现有 semantic/navigation executor 和 builder 承担部分功能；没有这些阶段的统一领域执行器 |
| Perception | 目标/完成条件 -> query -> 模型选择 -> 观测计划 -> 静止/相机资源检查 -> 各视角采集 -> 推理及组合验证 -> 收尾 -> TaskResult | [perception-agent/README.md](perception-agent/README.md)，§2、§4 | 已有 analyzer/provider 规则和固定拍照计划；新的阶段协议、覆盖契约和动态重试闭环待实现 |

所有拟议领域 JEV 阶段也单独列出，避免误认成漏登记的 Robot Skill：

| Owner | 阶段名 | 定义路径与状态 |
| --- | --- | --- |
| Navigation | nav_target_review、nav_arrival_review、nav_route_review、nav_plan_review、nav_progress_decision、nav_arrival_review_result | [Navigation README §4](navigaiton-agent/README.md)；新领域阶段待实现 |
| Perception | perception_goal_review、perception_query_review、perception_model_review、perception_observation_review、perception_plan_review、perception_result_review | [Perception README §4](perception-agent/README.md)；新完整闭环待实现 |

已有 [jev_agent.py](../../src/robot_agent/robot_agent/jev_agent.py) 包含 `perception_plan_review` 等同名/相关旧逻辑；**名字相同不代表已经符合新领域职责及执行挂钩设计**。本节不是穷举 API trace 的每个问题 ID。

以下是缺少实现定义的位置：独立 Task Catalog、独立 Task Step schema/catalog、TaskContract/TaskResult、NavigationTaskRunner、PerceptionTaskRunner、版本化领域 human rules。来源目前仅为上述 README；本文不为未建文件伪造可点击实现路径。

## 7. Recovery Step 附录：与 Task/Skill 分开

当前 20 个 playbook 的 `recovery` 数组如下。`action`、`operator_request`、`wait_evidence` 是不同步骤类型；下表顺序是配置顺序，成功跳转 verify、on_fail 等仍按原 YAML，**不是所有步骤每次都执行**。

| Playbook / 定义文件 | recovery 顺序 |
| --- | --- |
| [PB-ARM-LINK-LOST.yaml](../../src/robot_agent/config/recovery/playbooks/PB-ARM-LINK-LOST.yaml) | 人工 replug_arm |
| [PB-CAMERA-MODEL-MISMATCH.yaml](../../src/robot_agent/config/recovery/playbooks/PB-CAMERA-MODEL-MISMATCH.yaml) | 人工 fix_isaac_camera_resolution |
| [PB-CAMERA-STALE.yaml](../../src/robot_agent/config/recovery/playbooks/PB-CAMERA-STALE.yaml) | 人工 restart_sim_camera |
| [PB-COMPONENT-NOT-STARTED.yaml](../../src/robot_agent/config/recovery/playbooks/PB-COMPONENT-NOT-STARTED.yaml) | 人工 confirm_launch_mode |
| [PB-DRIVE-ALARM.yaml](../../src/robot_agent/config/recovery/playbooks/PB-DRIVE-ALARM.yaml) | cancel_motion -> 人工 inspect_drive_alarm -> drive_alarm_reset |
| [PB-DRIVE-LINK-LOST.yaml](../../src/robot_agent/config/recovery/playbooks/PB-DRIVE-LINK-LOST.yaml) | 人工 check_drive_usb_power -> wait_evidence(device_present) |
| [PB-DRIVE-PROC-DOWN.yaml](../../src/robot_agent/config/recovery/playbooks/PB-DRIVE-PROC-DOWN.yaml) | restart_component(producer) |
| [PB-HOST-RESOURCE.yaml](../../src/robot_agent/config/recovery/playbooks/PB-HOST-RESOURCE.yaml) | 人工 reduce_host_load |
| [PB-LIDAR-STALE.yaml](../../src/robot_agent/config/recovery/playbooks/PB-LIDAR-STALE.yaml) | 人工 restart_sim_lidar |
| [PB-LOC-LOST.yaml](../../src/robot_agent/config/recovery/playbooks/PB-LOC-LOST.yaml) | cancel_motion -> publish_initialpose(last_trusted_pose) -> 未验证成功时人工 rviz_2d_pose_estimate |
| [PB-MODE-MANUAL.yaml](../../src/robot_agent/config/recovery/playbooks/PB-MODE-MANUAL.yaml) | 人工 switch_to_auto |
| [PB-NAV-BLOCKED.yaml](../../src/robot_agent/config/recovery/playbooks/PB-NAV-BLOCKED.yaml) | clear_costmap(local) -> 人工 remove_obstacle |
| [PB-NAV-LIFECYCLE.yaml](../../src/robot_agent/config/recovery/playbooks/PB-NAV-LIFECYCLE.yaml) | lifecycle_reset |
| [PB-NODE-DOWN.yaml](../../src/robot_agent/config/recovery/playbooks/PB-NODE-DOWN.yaml) | restart_component(producer) |
| [PB-SAFE-ESTOP-INCONSISTENT.yaml](../../src/robot_agent/config/recovery/playbooks/PB-SAFE-ESTOP-INCONSISTENT.yaml) | cancel_motion -> 人工 inspect_estop_wiring |
| [PB-SAFE-ESTOP-PRESSED.yaml](../../src/robot_agent/config/recovery/playbooks/PB-SAFE-ESTOP-PRESSED.yaml) | cancel_motion -> 人工 release_estop |
| [PB-SAFE-SOFTWARE-STOP.yaml](../../src/robot_agent/config/recovery/playbooks/PB-SAFE-SOFTWARE-STOP.yaml) | 人工 release_software_estop |
| [PB-SIM-CLOCK-STALL.yaml](../../src/robot_agent/config/recovery/playbooks/PB-SIM-CLOCK-STALL.yaml) | 人工 resume_or_restart_isaac |
| [PB-SIM-RTF-LOW.yaml](../../src/robot_agent/config/recovery/playbooks/PB-SIM-RTF-LOW.yaml) | 人工 inspect_sim_clock_and_load |
| [PB-STEER-HW-ERROR.yaml](../../src/robot_agent/config/recovery/playbooks/PB-STEER-HW-ERROR.yaml) | cancel_motion -> 人工 inspect_steer_motor -> dynamixel_reboot |

执行状态机：[src/robot_agent/robot_agent/recovery/executor.py](../../src/robot_agent/robot_agent/recovery/executor.py)。动作后端：[src/robot_agent/robot_agent/recovery/backends.py](../../src/robot_agent/robot_agent/recovery/backends.py)，`supports` / `start`。

后端有 restart_component、lifecycle_reset、clear_costmap、cancel_motion、publish_initialpose、notify_mission 的分派。是否能自动执行还依赖环境配置、前置条件和验证规则。drive_alarm_reset、dynamixel_reboot 在 PB 有声明，但此 backend 没有对应实现，不能写成已具备自动修复。人工模板名称也不表示程序已实现同名自动动作。

publish_initialpose 后端还保留 auto_initial_align 来源分支；当前 PB-LOC-LOST 使用的是 last_trusted_pose，并未把 auto_initial_align 或 semantic_landmark 列入当前 recovery 顺序。semantic_landmark 不能作为现有 Robot Skill/恢复步骤计入。

## 8. 如何维护这一份清单

1. 新增 Task 时同时检查 Mission 分支、parser templates、JEV 类型/slots 和可选 LLM parser；不能只改一个 supported_task_types 列表。
2. 新增 Robot Skill 时检查 R/C、SkillClient 分派、executor 与测试，再更新 §5；注册与实测可用分开标注。
3. 新增内部步骤先确定 owner；领域判断不必登记成 Robot Skill，九个 view 也不应登记为九种 Task。
4. 后续真正实现 Task Catalog 后，以它生成 Task 清单，避免继续维护多份不一致的枚举。本文暂时是人工核对索引，不是第二套执行配置。
5. 普通 Task、JEV 阶段、PB recovery 动作分别统计。保持 Mission 一个用户请求、Task 一个可交付目标、Task Step 内部过程、Robot Skill 可执行能力的边界。

本次仅创建此文档，没有修改控制逻辑、registry、catalog 或三个 Agent 的运行实现。
