# Robot Skill Catalog for Deterministic + JEV Agent

> 本文保留 catalog 建设历史。最新术语与 task catalog 分工见 [Mission README](README.md)：Task Type/Task 不等于 robot skill；Agent Skills package 专指知识/流程包规范。旧例的 agent_skills、layer: agent_skill、AgentSkillTemplate 是历史命名，不应作为新 schema 扩展，应迁移为 task_types/task_templates。任务模板由所属 Agent 展开，不再一律由 Mission 拼成扁平 SkillCall 列表。

目标：把现有 `robot_skills` 里的 **robot skill** 整理成一个给 Deterministic parser、JEV candidate generator、MissionAgent、Dashboard 共同使用的 typed catalog。

这里需要区分三层概念，避免把不同生态里的 `skill` 混在一起：

- `robot skill`：真实可执行的底层能力，最终由 `SkillClient.execute_skill()` 调用，例如 `move_relative`、`set_camera_pose`、`detect_known_object`。
- `Agent Skills package`：给 AI agent 注入专业知识、工作流程、参考资料、工具使用规范的可复用包规范。它是 agent 能力扩展的知识/流程包，不等于本项目里的机器人执行 skill。
- `task template`：某个已登记 Task Type 的合法内部执行模板，例如固定九视角搜索。一个用户命令可产生多个 Task；Task owner 在能力/资源边界内将模板编译为 robot skill 调用，不把整句命令叫 agent skill。
- JEV 不直接发 ROS command，不自由创造 skill。Mission command_parse 独立选择 Task Type 和结构；领域 Agent 在已登记实体、模型与合法策略中做 typed decision，不把 local candidate plan 当作解析的唯一来源。

当前仓库已有基础来源：

- `src/robot_skills/config/skill_registry.yaml`：现有 robot skill registry。
- `src/robot_agent/robot_agent/skill_client.py`：skill dispatch 到具体 executor。
- `src/robot_agent/robot_agent/agents/*.py`：agent 层把 `MissionIntent` 展开为 `SkillCall`。
- `src/robot_agent/robot_agent/schemas.py`：`MissionIntent` / `SkillCall` / `MissionState`。

现有 registry 已经有 `name/group/version/description/safety_level/allowed_modes/timeout`，但还不足以高效支持 JEV。缺少的是自然语言别名、slot schema、前置条件细分、哪些 health block 可忽略、失败结果语义、可组合关系、恢复建议、Dashboard 展示语义。

---

## 1. 可参考项目和可借鉴点

这些项目不需要直接引入，但它们的设计思想有用：

- BehaviorTree.CPP：机器人行为常用的模块化、可调试、可恢复执行框架。适合参考 “节点/条件/恢复/可视化 trace” 的结构。https://www.behaviortree.dev/
- MoveIt Task Constructor：把复杂 manipulation task 拆成可组合 stages，stage 之间传递候选解。适合参考 “候选解 + stage composition + failure isolation”。https://github.com/moveit/moveit_task_constructor/blob/ros2/README.md
- Open-RMF task system：用 schema 化 JSON 描述任务、dispatcher 分派任务、fleet adapter 执行具体 robot action。适合参考 “task/capability/action 分层，不同 robot capability 通过 adapter 连接”。https://index.ros.org/r/rmf_task/
- SkiROS2：skill-based robot control，强调 pre-condition、hold-condition、post-condition 来组织和组合技能。适合参考 “skill contract 不只描述输入输出，还描述执行期间必须保持什么条件”。https://arxiv.org/abs/2306.17030
- ROS-LLM：把 LLM 输出转为 sequence / behavior tree / state machine 执行，并结合 task feedback。适合参考 “LLM/JEV 只做高层结构化行为，执行仍落回 ROS action/service”。https://arxiv.org/abs/2406.19741

对本项目最合适的路线不是引入一个大框架，而是先做轻量 catalog：

```text
YAML skill registry v2
  -> Python typed loader/validator
  -> Deterministic parser candidate generation
  -> JEV candidate choice / supervision state
  -> MissionAgent build_plan / health gate
  -> Dashboard skill catalog view / debug trace
```

---

## 2. 当前 Robot Skills 概览

从 `skill_registry.yaml` 和 `SkillClient` 看，当前 robot skills 可分为：

| Group | Robot skill | 含义 |
| --- | --- | --- |
| health_safety | `get_robot_status` | 读取 robot mode、safety、health |
| health_safety | `await_robot_auto_mode` | 等待/确认 AUTO，主要用于 base motion |
| health_safety | `await_robot_stationary` | 等待机器人静止，主要用于 perception/arm |
| health_safety | `check_localization_health` | 检查 map/global navigation 前的 localization |
| health_recovery | `recover_localization_spin` | localization recovery，目前 sim-only |
| semantic_map | `resolve_semantic_target` | 把自然语言地点解析为 map-frame approach pose |
| semantic_map | `list_known_targets` | 列出已知地点/目标 |
| base_motion | `goto_pose` | Nav2 到 map-frame pose |
| base_motion | `move_relative` | body-frame 前后左右相对移动 |
| base_motion | `move_relative_xy` | body-frame dx/dy 相对移动 |
| base_motion | `rotate_relative` | 原地相对旋转 |
| base_motion | `cancel_motion` | 取消导航/运动 |
| view_camera | `set_camera_pose` | pan/tilt 到指定视角 |
| view_camera | `capture_observation` | 保存当前 observation image |
| perception_detection | `detect_known_object` | 对 captured observations 做 known object / OCR 检测 |
| manipulation_lite | `move_lift_bounded` | lift 限界移动 |
| arm | `set_pose` | SO101 预设 pose/gesture |
| arm | `touch` | Touch Anything 接触/按压目标 |
| arm | `grasp` | VLA grasp/place |

这个列表可以作为 catalog seed，但应该继续补充机器可读字段。

---

## 3. Catalog v2 的核心设计

### 3.1 Skill Contract

每个 robot skill 都应该有完整 contract：

```yaml
- name: move_relative
  group: base_motion
  version: 0.2.0
  layer: robot_skill
  description: Move relative to current base frame with direct/relative motion policy.
  intent_tags:
    - relative_motion
    - base_motion
  natural_language:
    verbs: ["move", "go", "back", "forward", "前進", "後退"]
    aliases:
      - "go back"
      - "back up"
      - "前進少し"
    negative_examples:
      - "go to shelf"
      - "look back"
  args_schema:
    type: object
    required: ["direction"]
    properties:
      direction:
        type: enum
        values: ["forward", "back", "left", "right"]
      distance_m:
        type: float
        default: 0.2
        min: 0.05
        max_sim: 2.0
        max_real: 0.5
      timeout_sec:
        type: float
        default: 30
  preconditions:
    required:
      - safety_ok_or_watchdog
      - robot_mode_auto
      - wheel_odom_fresh
    not_required:
      - global_localization_healthy
    notes:
      - Relative/direct motion should not be blocked by localization-only BAD if wheel odom feedback is fresh.
  effects:
    reads: ["wheel_odom", "robot_mode", "safety_status"]
    writes: ["nav2_goal_or_direct_motion"]
    physical_motion: true
    contact: false
  failure_semantics:
    FAILED_LOCALIZATION:
      blocks_goal: false_if_only_relative_motion
      default_recovery: ask_human_or_record_design_gap
    CANCELLED:
      classify_with: ["user_cancel", "watchdog_cancel", "planner_cancel"]
      default_recovery: inspect_health_then_retry_or_ask
  composition:
    can_follow: ["await_robot_auto_mode", "await_robot_stationary"]
    can_precede: ["rotate_relative", "set_camera_pose", "capture_observation", "set_pose"]
    independent_cleanup_after_failure: false
  jev:
    candidate_priority: high
    ask_when_confidence_below: 0.72
    expose_to_command_parse: true
    expose_to_step_supervision: true
  dashboard:
    display_name: "Move relative"
    risk_label: "base motion"
```

### 3.2 为什么要区分 Preconditions

当前最大的不稳定来自“前置条件太粗”：

- `goto_pose` 需要 map/global localization。
- `move_relative/rotate_relative` 更应该依赖 wheel odom/direct feedback，不应该被 localization-only BAD 阻止。
- `perception` 不应该要求 AUTO；只要 robot stationary + camera fresh 即可。
- `set_pose` 不应该依赖 base localization，但需要 arm gateway/feedback。

所以 catalog 里必须把 precondition 拆细：

```yaml
precondition_atoms:
  safety_ok_or_watchdog:
    hard_block_for: ["base_motion", "arm_motion", "lift_motion"]
  robot_mode_auto:
    hard_block_for: ["goto_pose", "move_relative", "rotate_relative"]
    irrelevant_for: ["capture_observation", "detect_known_object"]
  robot_stationary:
    required_for: ["capture_observation", "set_camera_pose", "set_pose", "touch", "grasp"]
  global_localization_healthy:
    required_for: ["goto_pose"]
    irrelevant_for: ["move_relative", "rotate_relative", "capture_observation", "set_pose"]
  wheel_odom_fresh:
    required_for: ["move_relative", "move_relative_xy", "rotate_relative"]
  camera_image_fresh:
    required_for: ["detect_known_object"]
  arm_gateway_ready:
    required_for: ["set_pose", "touch", "grasp"]
```

这样 JEV 的 `plan_review` 和 `step_supervision` 可以被清晰约束，而不是猜。

---

## 4. Task Template 与 Agent Skills Package 的边界

本项目内部建议把“自然语言任务模板”统一叫 `task template`，不要叫 `agent skill`。

`Agent Skills package` 是另一层：它可以保存机器人领域知识、操作流程、调试步骤、prompt/参考文档，让 AI agent 学会如何维护和扩展这个系统。例如未来可以做一个 `robot-skill-authoring` Agent Skill package，里面包含：

- 如何新增 robot skill。
- 如何补 `skill_registry.yaml`。
- 如何补 `skill_catalog.v2.yaml`。
- 如何写 parser/candidate/test。
- 如何在 dashboard 上验证。

但这个 package 本身不直接执行机器人动作，也不等同于 `move_relative` 这类 robot skill。

`task template` 才是自然语言到 robot skill sequence 的系统内概念。

Example：

```yaml
- task_type: look_direction
  layer: agent_skill
  description: Move camera to a direction, capture image, optionally detect object, then return camera home.
  slots:
    direction: ["left", "right", "front", "front_left", "front_right"]
    band: ["ground", "middle", "upper"]
    object_query: optional string
  expands_to:
    - skill: await_robot_stationary
      optional: true
    - skill: set_camera_pose
      bind:
        pan_deg: "$direction.pan"
        tilt_deg: "$band.tilt"
    - skill: capture_observation
    - skill: detect_known_object
      when: "$object_query exists"
    - skill: set_camera_pose
      payload: {pan_deg: 0, tilt_deg: 67, scan_band: "return_home"}
  success_policy:
    if_no_object_query: "capture_observation_success_means_done"
    if_object_query: "detection_required_or_ask_human"
  failure_policy:
    target_not_found: "show_all_images_and_ask_or_mark_observed"
```

这能解决 `左を見て` 的语义：它是 observation，不是 detection。没有目标物也应 DONE，只要 observation captured。

---

## 5. JEV 如何使用 Catalog

### 5.1 Command Parse

Deterministic parser 不应该硬编码所有技能知识。它应该读 catalog 提供的：

- `natural_language.verbs/aliases`
- `args_schema`
- `intent_tags`
- `agent_skill.expands_to`
- `negative_examples`
- `risk_label`

Candidate generator 产生：

```json
{
  "candidate_id": "back_then_turn_left",
  "source": ["sequence_splitter", "skill_catalog_alias", "typo_normalizer"],
  "task_type": "sequence",
  "steps": [
    {
      "agent_skill": "move_relative",
      "robot_skills": ["await_robot_auto_mode", "move_relative"],
      "slots": {"direction": "back", "distance_m": 0.2}
    },
    {
      "agent_skill": "rotate_relative",
      "robot_skills": ["await_robot_auto_mode", "rotate_relative"],
      "slots": {"yaw_deg": 90}
    }
  ],
  "precondition_summary": {
    "requires_auto": true,
    "requires_global_localization": false,
    "requires_stationary": false
  },
  "risk": "base_motion"
}
```

JEV 只在候选之间选择：

```json
{
  "best_candidate_plan": {
    "type": "choice",
    "criteria": {
      "back_then_turn_left": "Preserves both requested actions and order.",
      "turn_left_only": "Drops the go-back subgoal.",
      "ask_human": "Typo or action is too ambiguous."
    }
  },
  "chosen_plan_uses_supported_skills": {"type": "noul"},
  "chosen_plan_missing_subgoal_risk": {"type": "score"},
  "needs_human_confirmation": {"type": "noul"}
}
```

### 5.2 Plan Review

JEV 输入不要只给 raw health，要给 catalog 计算后的 facts：

```json
{
  "plan": [{"skill": "move_relative"}, {"skill": "rotate_relative"}],
  "catalog_precondition_review": {
    "hard_blocks": [],
    "warnings": ["global localization BAD but not required for relative motion"],
    "irrelevant_bad_components": ["localization"],
    "required_missing": []
  }
}
```

这比让 JEV 自己理解所有 ROS health 字段稳定得多。

### 5.3 Step Supervision

每个 skill 的 `failure_semantics` 应直接进入 JEV state：

```json
{
  "latest_step_result": {"skill": "detect_known_object", "code": "NEEDS_HUMAN"},
  "skill_contract": {
    "failure_semantics": {
      "NEEDS_HUMAN": {
        "possible_meanings": ["target_not_found", "ambiguous_target", "unsupported_query"],
        "default_human_choices": ["continue_next_safe_cleanup", "retry_with_alternate_view", "stop"]
      }
    }
  },
  "next_step": {"skill": "set_pose", "risk": "arm_motion", "role": "cleanup_or_independent_action"}
}
```

---

## 6. Skill Catalog 管理方式

推荐文件结构：

```text
src/robot_skills/config/
  skill_registry.yaml              # 当前执行白名单，继续保留
  skill_catalog.v2.yaml            # 新增：完整 robot skill contract

src/robot_agent/config/
  task_templates.yaml              # 新增：自然语言任务模板，不是 Agent Skills package
  command_candidate_policy.yaml    # 新增：候选数量、阈值、typo policy
```

加载结构：

```text
SkillCatalog
  - robot_skills[name] -> RobotSkillContract
  - agent_skills[task_type] -> AgentSkillTemplate
  - precondition_atoms[name] -> PreconditionRule
  - failure_semantics[skill][code] -> SupervisionHint
```

代码使用：

- `SkillClient` 仍使用 `skill_registry.yaml` 做执行允许和 dispatch。
- `CandidateGenerator` 使用 `skill_catalog.v2.yaml` 和 `agent_skill_templates.yaml` 生成候选。
- `MissionAgent` 使用 catalog 做 plan review / health gate，不再散落硬编码。
- `JevStrengthenAgent` 使用 catalog 摘要构造 state/questions。
- Dashboard 读取 catalog summary，显示 skill 风险、前置条件、失败建议。

---

## 7. 初始 Schema 草案

```yaml
schema_version: robot_skill_catalog.v2

precondition_atoms:
  robot_mode_auto:
    source: health.robot_mode
    satisfied_when: "AUTO"
    hard_safety: false
  robot_stationary:
    source: health.stationary
    satisfied_when: true
    hard_safety: false
  safety_ok_or_watchdog:
    source: health.safety_status
    satisfied_when: ["OK", "WATCHDOG"]
    hard_safety: true

robot_skills:
  - name: capture_observation
    group: view_camera
    args_schema:
      type: object
      properties:
        min_image_time_monotonic: {type: float, optional: true}
    preconditions:
      required: ["robot_stationary", "camera_ready"]
      irrelevant: ["robot_mode_auto", "global_localization_healthy"]
    effects:
      physical_motion: false
      creates_observation: true
      writes: ["mission_observations"]
    failure_semantics:
      FAILED_TIMEOUT:
        category: camera_or_perception_fault
        recovery: retry_or_reset_camera
    jev:
      expose_to_command_parse: false
      expose_to_step_supervision: true

agent_skills:
  - task_type: search_object_here
    candidate_triggers:
      aliases: ["find", "search", "探して", "見つけて"]
    slots:
      object_query: {type: string, required: true}
    expands_to:
      - skill: capture_observation
      - skill: detect_known_object
        bind: {query: "$object_query"}
    success_policy:
      target_found: done
      target_not_found: ask_human_or_done_with_observation
```

---

## 8. 当前实现差距

当前已有：

- `skill_registry.yaml` 基础执行注册。
- `SkillRegistry` loader。
- `SkillClient` dispatch。
- `NavigationAgent / PerceptionAgent / ArmAgent` 的手写 plan template。
- `MissionAgent._handle_health_before_step()` 的部分 health gate。
- `JevStrengthenAgent` 的基础 command/plan/step decision trace。

缺口：

- 没有统一 `args_schema`，payload 合法性主要靠 executor 内部默认值或抛错。
- 没有自然语言 alias/negative examples，parser 只能散落写 regex。
- 没有机器可读的 “localization 对 relative motion 可忽略” 规则。
- 没有标准 failure semantics，`NEEDS_HUMAN / CANCELLED / FAILED_TIMEOUT` 的含义按 skill 不同会变化。
- 没有 task template 文件，任务展开逻辑散在 `agents/*.py` 和 `MissionAgent._build_plan()`。
- 没有 skill composition graph，JEV 很难判断前一步失败后下一步是否仍有意义。
- Dashboard 无法展示 “这个 skill 为什么被允许/阻止/建议恢复”。

---

## 9. 推荐实现阶段

### Phase 1: Catalog 文档化

- 新建 `skill_catalog.v2.yaml`，先覆盖现有 19 个 robot skills。
- 每个 skill 至少补齐：
  - `args_schema`
  - `preconditions.required/irrelevant`
  - `effects`
  - `failure_semantics`
  - `jev.expose_to_*`

### Phase 2: Loader + Validation

- 新增 `robot_skills.skill_catalog.SkillCatalog`。
- 启动时校验：
  - registry 中每个 skill 都在 catalog 中。
  - catalog 中每个 skill 都能被 `SkillClient` dispatch。
  - required args 有 schema。
  - failure code 使用 `result_codes.py` 中已有 code。

### Phase 3: Candidate Generator

- 新增 `robot_agent/candidates/skill_candidate_generator.py`。
- 输入 user text、catalog、deterministic result、embedding top-k。
- 输出 typed candidate plans。
- JEV 只选择 candidate，不自由生成 plan。

### Phase 4: Health Gate Refactor

- 把 `MissionAgent._handle_health_before_step()` 中的 hard-coded group 判断迁移为 catalog precondition review。
- 输出 `hard_blocks / warnings / irrelevant_bad_components / recovery_hints`。

### Phase 5: Dashboard Skill Debug

Dashboard 每个 step 显示：

```text
move_relative
requires: AUTO, wheel_odom_fresh, safety_ok_or_watchdog
ignored: global_localization_healthy
status: allowed
reason: localization BAD is irrelevant for relative motion
```

这会极大降低调试成本。

---

## 10. 优先整理的 Skills

为了支持当前 JEV 测试，先整理这些：

1. `move_relative`
2. `rotate_relative`
3. `move_relative_xy`
4. `goto_pose`
5. `set_camera_pose`
6. `capture_observation`
7. `detect_known_object`
8. `set_pose`
9. `touch`
10. `await_robot_auto_mode`
11. `await_robot_stationary`
12. `check_localization_health`

这 12 个覆盖了当前问题：

- sequence parse：`go back ... then/and turn/look/set pose`
- relative motion 不受 localization 限制
- perception 不受 AUTO/MANUAL 限制
- target not found 后是否继续
- dashboard 显示 captured images / detection album
- arm pose/touch 的前置条件和失败语义

---

## 11. 设计判断

现阶段最稳的路线：

```text
保留现有 robot skill executor
保留 SkillClient dispatch
新增 skill catalog v2 作为语义/约束/候选生成层
让 Deterministic parser 和 JEV 都消费同一份 catalog
让 MissionAgent 用 catalog review preconditions
让 Dashboard 展示 catalog review 和 JEV decision trace
```

这样能同时满足：

- 高效：常见命令 deterministic 仍可直接命中。
- 正确：JEV 用 catalog 检查完整性、相关性、遗漏 subgoal。
- 安全：JEV 只能选择 catalog 中允许的 typed candidate。
- 可维护：新增 robot skill 时，只需补 executor + registry + catalog + agent template，不需要在 parser/JEV/MissionAgent 里到处加 if/else。
