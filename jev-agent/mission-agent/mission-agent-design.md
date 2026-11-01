# JEV Stage / Question Design Review

> 文档定位更新：本文件保留旧版阶段/问题模板和当时的实现说明，不能据其“当前实现”标题判断最新状态。任务层级、职责、委派与 JEV 触发机制以 [Mission README](README.md)、[Navigation README](../navigaiton-agent/README.md)、[Perception README](../perception-agent/README.md) 为准。
> 2026-10-13：Primary 已改为 command_parse → command_bind → plan_review，删除旧单次请求中的全类型 slots/parse_confidence 问题。原文绑定不依赖 Classic 的连接词切分。最新设计、证据和待验证项见 [本轮记录](test&pro&solve/17-command-grounding-and-perception-capabilities-20261013.md)。
>
> 覆盖关系：Mission plan_review 审查 TaskGraph，不再拼平领域内部 SkillCall；感知问题迁入 Perception；step_supervision 区分 Task 内部处理与 Mission 的 task_outcome_review；用户 Cancel 直接执行，不交 JEV 判断。下文 single_step/sequence_step_count 按用户级 Task 理解，不按九视角拍照步骤计数。旧 per-skill 问题可供领域/Recovery 参考，不要求每个成功 skill 都调用 API。

目标：重新设计 JEV Strengthen Agent 的阶段、输入、问题、输出和执行挂钩方式。这个文档后续作为开发修改依据。

核心原则：

- JEV 的价值不是复读 local parser，而是弥补 deterministic/local command parse 与硬编码 if/else 的不足。
- JEV input 要告诉模型“当前 stage 的目标是什么、需要判断什么、可选输出是什么”，而不是堆无关系统字段。
- 每个 stage 的 questions 要分层：先判断任务形态，再判断子任务、依赖关系、继续/停止/recovery。
- 没有执行动作的 stage 不需要 robot health snapshot，除非这个 health 信息会改变 parse/plan 判断。
- 有执行结果的 stage 必须给具体 skill result/log/artifact/metrics，并按 skill 类型使用不同 question template。
- JEV 不能直接执行 ROS/skill。它输出 typed decision，MissionAgent/RecoveryAgent 负责执行。
- 安全硬门仍由 code 保底。JEV 可以建议继续/恢复/询问，但不能绕过 safety BAD/E-stop。

---

## 1. 运行时在哪里看 JEV 数据

当前可看：

- Dashboard: `JEV Decision Trace`
- Dashboard: `Raw mission state -> jev_trace`
- ROS topic: `/agent/mission_state`
- ROS topic: `/agent/mission_events`
- Log: `debug-output/agent/<timestamp>/skill_events.jsonl`

命令：

```bash
source /opt/ros/humble/setup.bash
source /home/ruan-x/robot_ws/install/setup.bash
export ROS_DOMAIN_ID=20

ros2 topic echo /agent/mission_state
ros2 topic echo /agent/mission_events

ls -td debug-output/agent/* | head
rg -n "jev|JEV|intent_parsed|plan_created|jev_supervision_decision|mission_needs_human" debug-output/agent/<latest>/skill_events.jsonl
```

Raw JEV call log：

```text
debug-output/agent/<timestamp>/jev_calls.jsonl
```

当前状态：已实现。`JevStrengthenAgent._write_call_log()` 会把 redacted request body、raw response、latency 写入 `jev_calls.jsonl`。Dashboard 只显示 summary；review 真实 JEV input/output 时应优先看这个文件。

每次 JEV call 写入 redacted raw data：

```json
{
  "time": "...",
  "mission_id": "...",
  "stage": "command_parse",
  "provider": "live",
  "model": "jev-latest",
  "request": {
    "state": {},
    "questions": {}
  },
  "response_raw": {},
  "normalized_answers": {},
  "adopted_decision": {},
  "redaction": {
    "api_key_removed": true,
    "large_images_removed": true,
    "max_string_chars": 2000
  },
  "latency_ms": 123.4,
  "cost_usd": 0.00001
}
```

这样 review 时能看到 JEV 真实收到什么，而不是只看 dashboard 摘要。

---

## 2. Stage Overview

推荐 stages：

```text
command_parse
  -> plan_review
  -> perception_query_review / perception_plan_review (only for perception tasks)
  -> pre_step_review
  -> step_supervision
  -> perception_result_review (only for perception results)
  -> recovery_choice
  -> final_assessment
```

当前优先实现：

1. `command_parse`
2. `plan_review`
3. perception subdecisions for `search_object` / `look_direction`
4. `step_supervision`
5. redacted `jev_calls.jsonl`

后续接入：

- `recovery_choice`：等 RecoveryAgent 设计完成。
- `progress_watch`：等 running-step metrics 更完整后做。
- `final_assessment`：用于任务完成质量评估和数据集统计。

### 2.1 Source of truth: task types vs robot skills

`supported_task_types` 不是自动等于当前全部 robot skills。它是 command_parse 暴露给自然语言的任务类型集合，应该从 catalog/schema 生成，而不是手写散落在 JEV prompt 里。

当前代码里的信息源：

- Robot skill 执行白名单：`src/robot_skills/config/skill_registry.yaml`
- Robot skill 语义/JEV catalog：`src/robot_skills/config/skill_catalog.v2.yaml`
- Perception model/provider：`src/robot_skills/config/perception_models.yaml`
- Perception target/model capability：`src/robot_skills/config/perception_targets.yaml`
- Parser examples/task types：`src/robot_agent/config/parser_templates.yaml`
- Deterministic parser 原始规则：`src/robot_agent/robot_agent/parsers/deterministic_parser.py`
- LLM parser prompt 允许的 task_type：`src/robot_agent/robot_agent/parsers/llama_cpp_parser.py`
- Perception task 到 skill plan 的映射：`src/robot_agent/robot_agent/agents/perception_agent.py`
- Mission task 到 skill plan 的总入口：`src/robot_agent/robot_agent/agents/mission_agent.py`

推荐长期设计：

- `skill_catalog.v2.yaml` 继续定义 robot skill contract。
- 新增或扩展一个 task catalog，定义 command_parse 可输出的 `task_type`，例如 `move_relative`、`rotate_relative`、`look_direction`、`search_object_here`、`search_object_near_place`、`set_pose`、`navigate_to_place`。
- JEV `supported_task_types` 从 task catalog 生成，只暴露面向用户任务的类型；内部 gate/recovery skill 如 `await_robot_stationary`、`check_localization_health` 不直接暴露给 command_parse。
- `plan_review` 再把 task graph 映射到 robot skill plan，并用 `skill_catalog.v2.yaml` 检查参数、precondition、failure semantics。

---

## 3. Stage: `command_parse`

### 3.1 Stage 目标

把用户自然语言命令解析成 typed task intent 或 ordered task graph。

这个阶段不执行 robot skill，所以不需要 robot health snapshot。

JEV 要判断：

- 这是不是 robot command。
- 是 single step 还是 ordered sequence。
- 如果是 sequence，有几步，每一步是什么。
- 每一步之间是什么关系：独立、依赖、条件、cleanup。
- 每一步对应的 robot task type 是什么。
- 参数是否已经足够，还是需要交给 code/catalog default policy 补默认值。
- 是否存在歧义、无法支持、不应执行。
- 如果 local parser 给出了候选，可以作为 candidate 之一，但不能成为 JEV 的主要依据。

### 3.2 不推荐输入

这些不是核心，不应作为主要 state：

- `schema_version`
- `stage`
- `local_candidate_intent` 作为中心字段
- 完整 robot health snapshot
- 大量历史 mission logs

`schema_version/stage` 可以保留在外层 metadata 方便日志和路由，但不要指望它帮助 JEV 判断。

### 3.3 推荐输入 state

```json
{
  "stage_goal": "Parse the user command into a supported robot task or ordered task graph. Prefer preserving all user subgoals over matching a local parser result.",
  "user_command": "check red object and go back a little",
  "language_hints": ["en", "ja", "zh"],
  "supported_task_types": [
    {
      "task_type": "move_relative",
      "description": "Move a short relative distance in the base frame.",
      "slots": {
        "direction": ["forward", "back", "left", "right"],
        "distance_m": "float optional"
      },
      "examples": ["go back a little", "前進少し"]
    },
    {
      "task_type": "rotate_relative",
      "description": "Rotate in place by a relative angle.",
      "slots": {"yaw_deg": "float"},
      "examples": ["turn left", "turn right 90 degrees"]
    },
    {
      "task_type": "search_object_here",
      "description": "Look for an object from current location.",
      "slots": {"object_query": "string"},
      "examples": ["find red box", "check red object"]
    },
    {
      "task_type": "look_direction",
      "description": "Move camera/view direction and capture an observation.",
      "slots": {"direction": ["left", "right", "front"], "band": ["ground", "middle", "upper"]}
    },
    {
      "task_type": "set_pose",
      "description": "Move arm to named pose.",
      "slots": {"pose": ["home", "see_front", "see_left", "see_right", "wave"]}
    },
    {
      "task_type": "navigate_to_place",
      "description": "Navigate to known semantic place."
    },
    {
      "task_type": "cancel",
      "description": "Cancel active task/motion."
    },
    {
      "task_type": "robot_status",
      "description": "Report robot status."
    }
  ],
  "default_policy_summary": {
    "move_relative": {
      "distance_words": {
        "a little": 0.2,
        "少し": 0.2
      },
      "missing_distance": "ask_human_or_safe_small_default_by_policy"
    },
    "rotate_relative": {
      "left": 90,
      "right": -90,
      "turn around": 180,
      "missing_angle": "catalog_default_if_direction_word_is_clear"
    },
    "look_direction": {
      "missing_band": "middle"
    }
  }
}
```

### 3.4 Slot/default responsibility boundary

不需要 `best_candidate_plan` 这种过渡实现。长期设计中，JEV 不应该在若干 local candidate 里投票，而应该按 stage 目标直接输出 task graph decision。

当前实现状态（2026-10-06）：

- `command_parse` 的 live JEV input 不再包含 `local_parse_result`、`local_candidate_intent`、`candidate_plans`、`best_candidate_plan`、`code_extracted_task_graph`。
- Classic/local parse 仍会在 code 内部作为 local fallback 使用，保证 JEV API 不可用时系统可运行；但在 JEV live call 的 state 里不暴露给 `command_parse`。
- JEV 的 task graph 不是自由 JSON 输出，而是由固定 typed questions 组合出来，然后 code 根据 answer 抽取/补齐 slots。
- 当前支持最多三步 sequence；`more_than_three` / `unclear` 会进入 ask human / thinking LLM 分支。

当前 `command_parse` live request 核心 state：

```json
{
  "stage_goal": "Parse the user command independently into a supported robot task or ordered task graph...",
  "user_command": "go back a little, after that see right",
  "language_hints": ["en", "ja", "zh"],
  "supported_task_types": [],
  "default_policy_summary": {}
}
```

当前固定 step/slot questions：

```json
{
  "step1_task_type": "choice",
  "step1_move_direction": "choice",
  "step1_distance_policy": "choice",
  "step1_turn_policy": "choice",
  "step1_look_direction": "choice",
  "step1_pose_name": "choice",
  "step1_slot_resolution_action": "choice",
  "step2_task_type": "choice",
  "step3_task_type": "choice"
}
```

也就是说：JEV 选择 task type / slot policy；具体 `distance_m`、`yaw_deg`、`object_query` 等由 parser/code/catalog policy 从原始文本或默认策略补齐。

责任边界：

- JEV 负责判断 intent、command shape、task type、sequence step count、step dependency。
- parser/code 负责从原始文本抽取明确 slot，例如 `back`、`left`、`red object`、`home`。
- catalog/default policy 负责把自然语言 slot 映射成可执行默认值，例如 `a little -> 0.2m`、`turn left -> 90deg`、`look left -> pan=-35, tilt=25, band=middle_left`。
- JEV 可以用 slot questions 判断某个 slot 是否 present、defaultable、需要 ask human。
- JEV 不应该直接自由生成危险数值参数。数值类参数要么来自文本明确数值，要么来自 catalog/default policy，要么 ask human。
- `plan_review` 负责二次检查补完后的 executable plan 是否仍忠实、完整、合理。

典型流程：

```json
{
  "user_command": "go back a little and turn left",
  "command_parse": {
    "command_shape": "ordered_sequence",
    "sequence_step_count": "two",
    "steps": [
      {"task_type": "move_relative", "slots": {"direction": "back", "distance_m": "DEFAULT:small"}},
      {"task_type": "rotate_relative", "slots": {"direction": "left", "yaw_deg": "DEFAULT:left_quarter_turn"}}
    ]
  },
  "code_slot_resolution": {
    "move_relative.distance_m": 0.2,
    "rotate_relative.yaw_deg": 90
  },
  "plan_review": "check the executable plan after defaults are applied"
}
```

### 3.5 分层 Questions 设计

#### Layer 1: Command validity

Command validity 不需要 local parse 结果。它只判断用户输入是否应该被 robot agent 处理。

```json
{
  "is_robot_command": {
    "type": "noul",
    "instructions": "Is the user text an actionable robot command or robot status/cancel request?"
  },
  "unsupported_or_unsafe_parse": {
    "type": "noul",
    "instructions": "Is the command outside supported robot capabilities or too unsafe/unclear to map without human help?"
  }
}
```

#### Layer 2: Command shape

Step count 用 choice，不用 score。`score` 更适合置信度；step count 是离散结构判断。

```json
{
  "command_shape": {
    "type": "choice",
    "instructions": "Classify the command structure.",
    "criteria": {
      "single_step": "One robot task with one main goal.",
      "ordered_sequence": "Multiple tasks with an explicit or implied order.",
      "parallel_or_independent_goals": "Multiple goals where later goals may still matter if earlier goals fail.",
      "status_or_cancel": "Robot status or cancel request.",
      "ambiguous_or_unsupported": "Cannot reliably map to supported tasks."
    }
  },
  "sequence_step_count": {
    "type": "choice",
    "instructions": "If command_shape is ordered_sequence or parallel_or_independent_goals, choose the number of task-level subgoals.",
    "criteria": {
      "one": "Only one executable task-level subgoal.",
      "two": "Two task-level subgoals.",
      "three": "Three task-level subgoals.",
      "more_than_three": "More than three task-level subgoals; ask human or thinking LLM before execution.",
      "unclear": "The number of subgoals is unclear; ask human or thinking LLM."
    }
  }
}
```

#### Layer 3A: Single step detail

Only meaningful when `command_shape=single_step`.

```json
{
  "single_step_task_type": {
    "type": "choice",
    "instructions": "Choose the best supported task type for a single-step command.",
    "criteria": {
      "move_relative": "Relative base movement.",
      "rotate_relative": "Relative base rotation.",
      "look_direction": "Camera/view observation direction.",
      "search_object_here": "Search/detect an object from current location.",
      "set_pose": "Arm pose/gesture.",
      "navigate_to_place": "Navigate to known place.",
      "robot_status": "Report status.",
      "cancel": "Cancel active task.",
      "ask_human": "Ask user to clarify."
    }
  },
  "single_step_slots_complete": {
    "type": "noul",
    "instructions": "Are the required slots for the selected single-step task present or safely defaultable?"
  },
  "single_step_slot_resolution_action": {
    "type": "choice",
    "instructions": "Choose how slot values should be resolved before plan_review.",
    "criteria": {
      "use_extracted_slots": "The command contains explicit usable slots.",
      "use_catalog_defaults": "The command contains enough intent words for safe catalog/default policy.",
      "ask_human_for_slots": "Required slots are missing or too ambiguous.",
      "unsupported_slots": "The slots requested are outside supported robot skill arguments."
    }
  }
}
```

Slot questions must be designed per robot task/skill family. If a task has no user-facing parameter to set, skip its slot questions.

Examples:

```json
{
  "move_relative_slots": {
    "direction": {
      "type": "choice",
      "criteria": {
        "forward": "Move forward.",
        "back": "Move backward.",
        "left": "Move left if supported by robot skill.",
        "right": "Move right if supported by robot skill.",
        "unclear": "Direction is missing or unclear."
      }
    },
    "distance_policy": {
      "type": "choice",
      "criteria": {
        "explicit_numeric": "User gave a numeric distance.",
        "catalog_small_default": "User used a small-distance phrase such as 'a little' or '少し'.",
        "catalog_named_default": "User used another phrase covered by catalog policy.",
        "ask_human": "Distance cannot be safely defaulted."
      }
    }
  },
  "rotate_relative_slots": {
    "turn_direction": {
      "type": "choice",
      "criteria": {
        "left": "Turn left.",
        "right": "Turn right.",
        "around": "Turn around.",
        "explicit_angle": "User gave numeric angle.",
        "unclear": "Turn direction/angle unclear."
      }
    },
    "angle_policy": {
      "type": "choice",
      "criteria": {
        "explicit_numeric": "Use user-provided numeric angle.",
        "catalog_direction_default": "Use catalog default for left/right/around.",
        "ask_human": "Angle cannot be safely defaulted."
      }
    }
  },
  "search_object_here_slots": {
    "object_query_present": {"type": "noul"},
    "query_is_specific_enough": {"type": "noul"},
    "query_resolution_action": {
      "type": "choice",
      "criteria": {
        "use_text_query": "Use object text from command.",
        "ask_human": "Object query too vague.",
        "unsupported_query": "Requested perception target is unsupported."
      }
    }
  },
  "set_pose_slots": {
    "pose_name": {
      "type": "choice",
      "criteria": {
        "home": "Set arm pose home.",
        "see_front": "Set arm/camera observation pose front.",
        "see_left": "Set arm/camera observation pose left.",
        "see_right": "Set arm/camera observation pose right.",
        "wave": "Wave pose/gesture.",
        "unknown": "Pose is unknown or unsupported."
      }
    }
  }
}
```

#### Layer 3B: Ordered sequence detail

Only meaningful when command has multiple tasks.

```json
{
  "sequence_steps_identified": {
    "type": "noul",
    "instructions": "Can the command be split into clear task-level steps?"
  },
  "all_subgoals_preserved": {
    "type": "noul",
    "instructions": "Does the parsed task graph preserve every requested subgoal, not just the first or easiest one?"
  },
  "subgoal_dependency_policy": {
    "type": "choice",
    "instructions": "Classify how later subgoals depend on earlier subgoals.",
    "criteria": {
      "independent_goals": "Later tasks can still be meaningful if earlier tasks fail.",
      "ordered_but_independent": "Order matters, but later tasks may still be useful after earlier failure.",
      "required_condition": "Later tasks only make sense if earlier task succeeds.",
      "cleanup_after_attempt": "Later task is cleanup/reset/home pose and may still be useful after failure.",
      "ambiguous_dependency": "The dependency is unclear and may require human choice."
    }
  }
}
```

For sequence, JEV output should include task types and dependency policy. It should not invent an opaque combined task type such as `move_relative_then_rotate_relative`.

```json
{
  "steps": [
    {
      "step_id": "s1",
      "task_type": "move_relative",
      "slot_resolution": {"direction": "back", "distance_policy": "catalog_small_default"},
      "dependency_role": "independent_goal"
    },
    {
      "step_id": "s2",
      "task_type": "rotate_relative",
      "slot_resolution": {"turn_direction": "left", "angle_policy": "catalog_direction_default"},
      "dependency_role": "ordered_but_independent_goal"
    }
  ],
  "connector": "and",
  "dependency_policy": "ordered_but_independent"
}
```

If `sequence_step_count=more_than_three` or `unclear`, treat it as abnormal for now:

- ask human for confirmation/splitting, or
- call thinking LLM as a second opinion, or
- stop as unparseable if the command is outside supported skills.

### 3.6 Unsupported / ambiguous behavior

`ambiguous_or_unsupported` should not always mean the same thing. It should branch:

```json
{
  "unsupported_resolution": {
    "type": "choice",
    "instructions": "Choose how to handle ambiguous or unsupported parse.",
    "criteria": {
      "ask_thinking_llm": "Use a slower LLM as a second opinion because the command may still be supported.",
      "ask_human": "Ask the human to choose or clarify.",
      "stop_unparseable": "Stop and report that the command cannot be parsed/supported.",
      "safe_status_only": "Only report robot status; do not execute physical skills."
    }
  }
}
```

This is where JEV's typed intuition should help decide whether to call a thinking LLM, ask human, or stop.

### 3.7 Command parse output

Recommended normalized output:

```json
{
  "parse_status": "accepted | ask_human | ask_thinking_llm | unsupported",
  "task_graph": {
    "steps": [
      {
        "step_id": "s1",
        "task_type": "search_object_here",
        "slots": {"object_query": "red object"},
        "slot_resolution": {"object_query": "use_text_query"},
        "dependency_role": "independent_goal"
      },
      {
        "step_id": "s2",
        "task_type": "move_relative",
        "slots": {"direction": "back", "distance_m": "DEFAULT:small"},
        "slot_resolution": {"distance_policy": "catalog_small_default"},
        "dependency_role": "independent_goal_after_observation"
      }
    ],
    "connector": "and",
    "dependency_policy": "ordered_but_independent"
  },
  "human_choices": []
}
```

---

## 4. Stage: `plan_review`

### 4.1 Stage 目标

把 command_parse 的 task graph 转成可执行 robot skill plan，并判断 plan 是否忠实、完整、合理。JEV Primary 模式下，`plan_review` 是唯一允许比较 JEV plan 与 Classic/local plan 的阶段。

这个阶段还没有执行动作，所以通常不需要 robot health snapshot。

需要检查：

- JEV parse/task graph 结果和对应 executable plan。
- Classic/local parser 的 task graph 和 executable plan。Classic 可以是 `null` 或 empty。
- parser/code 抽取出的 slots。
- catalog/default policy 补完后的参数。
- robot skill catalog 的 compact 信息。

JEV 要判断：

- JEV plan 是否覆盖所有用户 subgoals。
- Classic plan 是否覆盖所有用户 subgoals。
- plan step 顺序是否正确。
- plan 是否用了支持的 robot skill。
- slot/default 补完是否合理，是否应该 ask human。
- JEV 和 Classic 哪个更适合执行，或者两者都不应执行。

### 4.2 当前实现输入 state

```json
{
  "stage_goal": "Review plans before any robot action. Compare the JEV Primary plan with the Classic parser plan only in this stage, then choose JEV, Classic, ask a human/LLM, or reject.",
  "user_command": "check red object and go back a little",
  "jev_task_graph": {
    "steps": [
      {"step_id": "s1", "task_type": "search_object_here", "slots": {"object_query": "red object"}, "dependency_role": "independent_goal"},
      {"step_id": "s2", "task_type": "move_relative", "slots": {"direction": "back", "distance_m": 0.2}, "dependency_role": "independent_goal_after_observation"}
    ]
  },
  "classic_task_graph": null,
  "jev_slot_resolution": {
    "task_graph_slots": [],
    "plan_args": {
      "0:detect_known_object": {"query": {"source": "code_or_catalog_default", "value": "red object"}}
    }
  },
  "classic_slot_resolution": {
    "task_graph_slots": [],
    "plan_args": {}
  },
  "jev_plan": [
    {"index": 0, "name": "await_robot_stationary", "payload": {"settle_sec": 1.0, "timeout_sec": 180.0}, "timeout_sec": 180.0, "risk_label": "unknown"},
    {"index": 1, "name": "detect_known_object", "payload": {"query": "red object", "provider": "yolo11s"}, "timeout_sec": 3.0, "risk_label": "perception"}
  ],
  "classic_plan": []
}
```

为了降低 latency/cost，`jev_plan` / `classic_plan` 是 compact plan，不传完整 `SkillCall.to_dict()`、完整 catalog、完整 health snapshot。

### 4.3 Questions

```json
{
  "plan_covers_all_subgoals": {
    "type": "noul",
    "instructions": "Does the JEV plan cover all user subgoals?"
  },
  "classic_plan_covers_all_subgoals": {
    "type": "noul",
    "instructions": "Does the Classic parser plan cover all user subgoals?"
  },
  "plan_step_order_correct": {
    "type": "noul",
    "instructions": "Does the selected plan preserve the user's intended step order?"
  },
  "plan_uses_supported_robot_skills": {
    "type": "noul",
    "instructions": "Does every JEV plan step map to supported robot skills?"
  },
  "classic_plan_uses_supported_robot_skills": {
    "type": "noul",
    "instructions": "Does every Classic plan step map to supported robot skills?"
  },
  "slot_defaults_are_reasonable": {
    "type": "noul",
    "instructions": "Are all catalog/default-policy parameters reasonable for the user's command and safe operating defaults?"
  },
  "slots_need_human_confirmation": {
    "type": "noul",
    "instructions": "Do any missing/defaulted parameters require human confirmation before execution?"
  },
  "plan_dependency_policy_clear": {
    "type": "noul",
    "instructions": "Is the dependency/continuation policy between subgoals clear enough to use during step supervision?"
  },
  "plan_source_decision": {
    "type": "choice",
    "instructions": "Choose which plan, if any, should be executed.",
    "criteria": {
      "use_jev_plan": "Execute the JEV Primary plan.",
      "use_classic_plan": "Execute the Classic parser plan because it is better.",
      "ask_human": "Ask human before starting.",
      "ask_thinking_llm": "Use slower LLM second opinion before starting.",
      "reject": "Reject because it is unsafe or unsupported."
    }
  }
}
```

### 4.4 执行挂钩

当前 MissionAgent 行为：

- `use_jev_plan`：继续执行 JEV Primary plan。
- `use_classic_plan`：切换为 Classic intent/plan 后执行。
- `ask_human` / `ask_thinking_llm`：不发布/执行 plan，进入 `NEEDS_HUMAN`。
- `reject`：不发布/执行 plan，进入 terminal failure。

执行逻辑只能消费 typed answers / normalized `adopted_decision`，不消费 JEV 自由文本 reason。解释文本由 code/template 根据 answers 生成。

### 4.5 Current gap

已修复：

- `plan_review` 不再以 runtime health 为中心。
- `command_parse` 不参考 Classic；Classic 只在 `plan_review` 对比。
- `plan_start_decision` 已替换为 `plan_source_decision`。
- 如果 `plan_review` 不通过，MissionAgent 不会继续发布/执行 plan。

仍需完善：

- `plan_source_decision=ask_thinking_llm` 目前仍按 ask human 处理，thinking LLM 二次判断尚未接入。
- `plan_step_order_correct` 现在主要检查 selected/current plan，未来可以分别检查 `jev_plan_step_order_correct` 和 `classic_plan_step_order_correct`。
- `slot_defaults_are_reasonable` 仍是通用问题，未来应细分到 per-skill slot policy。

---

## 5. Perception Agent JEV Design

### 5.1 Why perception needs its own JEV decisions

`search_object_here` / `search_object_near_place` 虽然在 MissionAgent 看起来是一个 task，但 PerceptionAgent 内部已经有自己的分层语义：

- query 是否可识别。
- 应该只 capture 当前 view，还是做 bounded multi-view scan。
- 应该用 `yolo11s`、`yolo11s_ocr`，还是直接返回 unsupported/needs human。
- object word 是否是模型可检测类别、颜色/属性、OCR/text/label、shelf label，还是完全未知。
- 没检测到目标时，这是有效的“未找到”，还是 camera/model/provider 故障。

JEV mode 下，PerceptionAgent 也应该切换成 typed JEV decision，而不是只靠 `_provider_for_query()` 和 `_is_fully_unknown_query()` 这些 hardcoded rules。

### 5.2 Perception stages

推荐增加 perception 子阶段：

```text
perception_query_review
  -> perception_plan_review
  -> perception_result_review
```

这些阶段只在涉及 perception task 时触发。

### 5.3 Stage: `perception_query_review`

目标：判断用户给出的 object/query word 是否可以交给当前 perception pipeline。

Input:

```json
{
  "stage_goal": "Review the object query before creating a perception plan.",
  "task_type": "search_object_here",
  "object_query": "red box",
  "perception_capability_summary": {
    "providers": ["yolo11s", "yolo11s_ocr", "stub"],
    "known_colors": ["red", "blue", "green", "yellow", "orange", "black", "white"],
    "known_objects_examples": ["box", "bottle", "car", "person", "chair"],
    "ocr_supported": true,
    "shelf_label_supported": true
  }
}
```

Questions:

```json
{
  "query_is_perception_request": {
    "type": "noul",
    "instructions": "Is the query asking for visual perception rather than navigation or camera direction only?"
  },
  "query_contains_detectable_signal": {
    "type": "noul",
    "instructions": "Does the query contain at least one visual signal supported by current perception capabilities?"
  },
  "query_type": {
    "type": "choice",
    "criteria": {
      "object_class": "Known/open-vocab object class.",
      "color_plus_object": "Color-qualified object such as red box.",
      "text_or_label": "Text/OCR/label query.",
      "shelf_label": "Shelf label / number label query.",
      "scene_observation_only": "No detection target; just capture/look.",
      "unsupported_or_abstract": "No current model can reasonably detect this."
    }
  },
  "query_resolution_action": {
    "type": "choice",
    "criteria": {
      "use_query": "Use query as detector text.",
      "normalize_query_by_catalog": "Normalize aliases/classes using perception_targets catalog.",
      "ask_human": "Query is too vague or unsupported.",
      "skip_detection_capture_only": "Only capture observations; do not run detector."
    }
  }
}
```

### 5.4 Stage: `perception_plan_review`

目标：选择 perception plan，不执行 physical base motion；只允许 camera pose/capture/detect 这些 perception-related skills。

Input 应包含：

- reviewed `object_query`
- current task type: `look_direction` / `search_object_here` / `search_object_near_place`
- current robot stationary status，because perception requires stationary robot
- available providers from `perception_models.yaml`
- capability summary from `perception_targets.yaml`
- candidate perception plans generated by code/catalog:
  - `capture_current_view_only`
  - `directional_view_capture`
  - `bounded_3_view_scan`
  - `bounded_9_pose_scan`
  - `detect_without_motion_for_unsupported_query`

Questions:

```json
{
  "perception_task_kind": {
    "type": "choice",
    "criteria": {
      "capture_only": "User only asked to look/observe.",
      "detect_in_current_view": "Run detector on current or requested view.",
      "bounded_search_scan": "Move camera through bounded scan and detect.",
      "arrival_confirmation": "Detect label/object to confirm arrived place.",
      "unsupported": "Cannot form a meaningful perception plan."
    }
  },
  "provider_choice": {
    "type": "choice",
    "criteria": {
      "yolo11s": "Object/color detection without OCR requirement.",
      "yolo11s_ocr": "Object detection plus OCR/text/label support.",
      "stub": "Only stub is available; should be marked low confidence.",
      "none_capture_only": "No detector provider needed."
    }
  },
  "scan_policy": {
    "type": "choice",
    "criteria": {
      "current_view_only": "Do not move camera; capture/detect current view.",
      "directional_view": "Move camera to left/right/front/up/down then capture.",
      "three_view_middle": "Short left-center-right middle scan.",
      "nine_pose_bounded": "Full bounded shelf/object scan.",
      "no_camera_motion": "Do not move camera because query is unsupported or robot is not stationary."
    }
  },
  "perception_plan_start_decision": {
    "type": "choice",
    "criteria": {
      "start": "Start perception plan.",
      "ask_human": "Ask human to clarify target/query.",
      "capture_only": "Capture observations but skip detection.",
      "reject": "Reject unsupported perception request."
    }
  }
}
```

### 5.5 Stage: `perception_result_review`

目标：检测完成后判断结果含义。这个阶段可作为 `step_supervision` 的 skill-specific 子模板，也可以由 PerceptionAgent 内部先做一次 typed review。

Questions:

```json
{
  "perception_result_category": {
    "type": "choice",
    "criteria": {
      "target_found": "Requested target was found.",
      "target_not_found_valid": "Images/model ran normally but target was not found.",
      "ambiguous_match": "Some evidence exists but identity is ambiguous.",
      "unsupported_query": "The target cannot be recognized by current models.",
      "camera_fault": "Image capture/camera stream failed.",
      "provider_fault": "Model/provider/dependency unavailable.",
      "stale_or_missing_artifact": "Images/results missing or stale."
    }
  },
  "show_observation_images": {
    "type": "noul",
    "instructions": "Should dashboard show captured images even when target is not found?"
  },
  "perception_next_action": {
    "type": "choice",
    "criteria": {
      "mark_success": "Target found or observation-only task complete.",
      "ask_human_continue_retry_stop": "Ask human whether to continue, retry from another view, or stop.",
      "retry_alternate_view": "Retry perception from another bounded view.",
      "call_recovery_agent_camera": "Camera/model health problem; call recovery.",
      "mark_target_not_found": "No target found, but no health recovery needed."
    }
  }
}
```

### 5.6 RecoveryAgent trigger boundary

RecoveryAgent should be triggered when the failure is likely a health/system problem, not when the robot simply did not find the requested semantic target.

Use recovery when evidence points to:

- localization/map pose unhealthy and current skill requires map-frame navigation.
- local planner/controller fault.
- wheel odom missing for relative motion feedback.
- camera stream missing/stale, image artifact missing, or repeated capture timeout.
- perception provider/model dependency unavailable when detection is required.
- arm/lift gateway missing, stale, or command rejected.
- safety/control-mode issue where a bounded recovery or human operator action is needed.
- system load/timeouts that make a retry/wait meaningful.

Do not treat these as RecoveryAgent by default:

- valid perception result with `target_not_found`.
- valid detection ambiguity that needs human semantic choice.
- unsupported object word/query when camera/model is otherwise healthy.
- command parse ambiguity.
- user asked for an unsupported task.

In those cases, preferred actions are `ask_human_continue_retry_stop`, `mark_target_not_found`, `ask_thinking_llm`, or `reject`, not recovery.

---

## 6. Stage: `step_supervision`

### 6.1 Stage 目标

每个 robot skill step 结束后，或遇到 error/stall 后，判断：

- 这个 step 是否正常完成。
- 结果是否合理、是否符合该 skill 的 contract。
- 是否需要 recovery agent。
- 是否可以继续下一步。
- 如果失败，是否应该 ask human / retry / skip failed subgoal / continue independent subgoal / stop。

原则：

```text
在没有安全隐患的前提下，尽可能完成更多任务。
```

但这不等于失败后自动继续。对 physical action，尤其 base motion/contact arm motion，如果前一步失败导致语义不确定，应问人。

执行策略：

- 正常成功的普通 step 默认不调用 blocking JEV；用 code-level invariant check 记录结果即可。
- 成功但高风险/结果可疑/最后一步质量评估时，可以用 async shadow JEV，不阻塞执行。
- error、timeout、CANCELLED、NEEDS_HUMAN、stall、artifact missing、result inconsistent 时，调用 blocking `step_supervision`。
- `step_supervision` 不需要重新解析 `user_command`。它应主要使用已经 review 过的 task graph、current step、skill result、next step group、dependency policy。

### 6.2 输入 state 必须包含什么

和 parse/plan 不同，这里必须包含具体运行结果。但为了降低 latency/cost，当前实现不会传完整 plan / 完整 artifacts / 大量 recent events，只传当前 step 周围的小窗口和 compact result。

```json
{
  "stage_goal": "Supervise the current robot skill result and decide whether to continue, recover, ask human, retry, or stop.",
  "task_graph": {
    "steps": [
      {"step_id": "s1", "task_type": "search_object_here", "dependency_role": "independent_goal"},
      {"step_id": "s2", "task_type": "move_relative", "dependency_role": "independent_goal_after_observation"}
    ]
  },
  "plan_window": {
    "total_steps": 6,
    "current_index": 3,
    "window": [
      {"index": 2, "name": "capture_observation", "payload": {}, "timeout_sec": 3.0, "risk_label": "perception"},
      {"index": 3, "name": "detect_known_object", "payload": {"query": "red object", "provider": "yolo11s"}, "timeout_sec": 3.0, "risk_label": "perception"},
      {"index": 4, "name": "await_robot_auto_mode", "payload": {"timeout_sec": 0.1}, "timeout_sec": 0.1, "risk_label": "gate"},
      {"index": 5, "name": "move_relative", "payload": {"direction": "back", "distance_m": 0.2}, "timeout_sec": 30.0, "risk_label": "base_motion"}
    ]
  },
  "current_step": {
    "index": 3,
    "skill": "detect_known_object",
    "position": "4/6",
    "skill_contract": {
      "risk": "perception",
      "expected_success_codes": ["OK"],
      "known_failure_semantics": {
        "NEEDS_HUMAN": "target_not_found_or_ambiguous"
      }
    }
  },
  "skill_result": {
    "success": false,
    "code": "NEEDS_HUMAN",
    "message": "No confident match for red object.",
    "metrics": {
      "detections_count": 0,
      "provider": "yolo11s",
      "image_topic": "/camera/color/image_raw"
    },
    "artifact_summary": [
      {"type": "image", "topic": "/camera/color/image_raw"}
    ]
  },
  "next_step_group": {
    "skill": "move_relative",
    "risk": "base_motion"
  },
  "safety_summary": {
    "hard_safety_block": false,
    "safety_status": "OK"
  }
}
```

Full images / large artifacts should not go into JEV input. Dashboard should show them from mission/perception artifacts; JEV only needs counts, topics, provider, and result category.

### 6.3 通用 Questions

这些 questions 主要用于失败/异常分支。成功分支可以只做本地 invariant check，除非需要 shadow evaluation。

```json
{
  "step_completed_normally": {
    "type": "noul",
    "instructions": "Did the current robot skill step complete normally according to its expected result?"
  },
  "step_result_reasonable": {
    "type": "noul",
    "instructions": "Is the result plausible and internally consistent with the skill logs/metrics/artifacts?"
  },
  "failure_category": {
    "type": "choice",
    "instructions": "Classify the step result or failure.",
    "criteria": {
      "none_success": "The step succeeded normally.",
      "target_not_found": "Object/search target was not found.",
      "ambiguous_detection": "Detection exists but target identity is ambiguous.",
      "camera_or_perception_fault": "Camera or perception pipeline failed.",
      "control_mode_block": "AUTO/MANUAL control mode blocked the skill.",
      "safety_block": "Safety system blocks continuation.",
      "localization_unrelated": "Localization is unhealthy but irrelevant to this skill.",
      "localization_related": "Localization blocks this skill.",
      "local_planner_fault": "Nav2/local planner failed.",
      "arm_fault": "Arm gateway/motion fault.",
      "timeout_or_load": "Timeout or system load issue.",
      "logic_or_parse_error": "Wrong skill/plan was selected.",
      "unknown": "Insufficient evidence."
    }
  },
  "needs_recovery_agent": {
    "type": "noul",
    "instructions": "Should a recovery agent be invoked before continuing or retrying?"
  },
  "recovery_target": {
    "type": "choice",
    "instructions": "If recovery is needed, choose the recovery target.",
    "criteria": {
      "none": "No recovery needed.",
      "localization": "Recover localization.",
      "local_planner": "Recover Nav2/local planner.",
      "camera": "Recover camera/perception stream.",
      "arm_gateway": "Recover arm gateway/state.",
      "control_mode": "Ask human/operator to change control mode.",
      "system_load": "Wait/retry due to load.",
      "unknown": "Recovery target unclear."
    }
  },
  "failure_blocks_current_subgoal": {
    "type": "noul",
    "instructions": "Does this failure block the current task subgoal?"
  },
  "next_subgoal_still_meaningful": {
    "type": "noul",
    "instructions": "Can the next task subgoal still be meaningful after this result?"
  },
  "continue_requires_human": {
    "type": "noul",
    "instructions": "Should a human choose whether to continue because the next step is physical or semantically value-dependent?"
  },
  "supervision_action": {
    "type": "choice",
    "instructions": "Choose the next control action.",
    "criteria": {
      "continue_next_step": "Continue automatically to the next skill.",
      "ask_human_continue_retry_stop": "Ask human whether to continue, retry, or stop.",
      "retry_same_step": "Retry the same skill once.",
      "retry_with_alternate_view": "Change view and retry perception.",
      "skip_failed_subgoal_continue_independent_goal": "Skip failed subgoal but continue a later independent subgoal.",
      "call_recovery_agent": "Invoke recovery agent before retry/continue.",
      "stop_failed": "Stop because continuing is unsafe or meaningless.",
      "record_design_gap": "Stop/ask and record that planner logic should be improved."
    }
  }
}
```

### 6.4 Skill-specific templates

Skill-specific questions 后续应按 robot skill 单独补全。没有参数要设置、或失败处理完全通用的 skill 可以 skip 专用 slot/supervision questions。

#### `detect_known_object`

Use when current skill is perception detection.

Input should include:

- query
- provider
- captured images count
- detections count
- matched count
- annotated image artifacts
- no-target images

Questions:

```json
{
  "detection_query_supported": {
    "type": "noul",
    "instructions": "Is the requested object/query supported by the current detector/provider or rules?"
  },
  "target_found": {
    "type": "noul",
    "instructions": "Was the requested target found with enough evidence?"
  },
  "no_target_result_valid": {
    "type": "noul",
    "instructions": "If no target was found, is that a valid perception result rather than a camera/model fault?"
  },
  "retry_with_alternate_view_useful": {
    "type": "score",
    "instructions": "Score whether another view/capture is likely to improve detection."
  },
  "observation_should_be_shown": {
    "type": "noul",
    "instructions": "Should captured images be shown to the user even if no target was found?"
  }
}
```

Expected default:

- If target not found but images are valid:
  - current subgoal failed or incomplete.
  - show images.
  - next independent subgoal may continue only with human confirmation if physical.

#### `capture_observation`

Input:

- image path
- image timestamp
- camera topic
- min_image_time
- freshness

Questions:

```json
{
  "image_captured": {"type": "noul"},
  "image_fresh_enough": {"type": "noul"},
  "image_artifact_available": {"type": "noul"},
  "capture_fault_category": {
    "type": "choice",
    "criteria": {
      "none_success": "Image captured normally.",
      "stale_image": "Image is stale.",
      "missing_image": "No image artifact.",
      "camera_topic_missing": "Camera topic unavailable.",
      "unknown": "Unclear."
    }
  }
}
```

If success: continue.  
If stale/missing: retry capture or camera recovery.

#### `set_camera_pose`

Input:

- requested pan/tilt
- feedback pan/tilt
- tolerance
- scan_band
- require_feedback

Questions:

```json
{
  "camera_reached_pose": {"type": "noul"},
  "feedback_required_and_valid": {"type": "noul"},
  "pose_error_blocks_next_capture": {"type": "noul"},
  "camera_recovery_needed": {"type": "noul"}
}
```

If pose not reached and next is capture/detect, retry or ask/recover.

#### `move_relative` / `move_relative_xy`

Input:

- requested distance/dx/dy
- control mode used
- command topic
- duration
- start/final wheel odom pose
- travelled distance
- canceled flag
- safety status

Questions:

```json
{
  "relative_motion_completed": {"type": "noul"},
  "travel_matches_request": {"type": "noul"},
  "localization_issue_irrelevant": {
    "type": "noul",
    "instructions": "Is any localization issue irrelevant because motion used direct relative feedback?"
  },
  "motion_failure_category": {
    "type": "choice",
    "criteria": {
      "none_success": "Motion completed.",
      "control_mode_block": "AUTO/MANUAL mode blocked motion.",
      "safety_block": "Safety stopped motion.",
      "wheel_odom_missing": "No relative feedback.",
      "timeout_or_load": "Motion timed out.",
      "unknown": "Unclear."
    }
  }
}
```

If success: continue.  
If failed due to control mode: ask human.  
If safety block: stop.  
If wheel odom missing: recovery or ask.

#### `rotate_relative`

Input:

- requested yaw
- final yaw change
- command topic
- duration
- canceled flag
- wheel odom yaw

Questions:

```json
{
  "rotation_completed": {"type": "noul"},
  "yaw_matches_request": {"type": "noul"},
  "rotation_cancel_reason": {
    "type": "choice",
    "criteria": {
      "none_success": "Rotation completed.",
      "user_cancel": "User canceled.",
      "safety_block": "Safety stopped rotation.",
      "control_mode_block": "Control mode blocked rotation.",
      "wheel_odom_missing": "No yaw feedback.",
      "unknown": "Unclear."
    }
  }
}
```

#### `goto_pose`

Input:

- target pose
- nav2 result
- route/candidate
- timeout
- distance progress
- local planner state
- localization health

Questions:

```json
{
  "navigation_goal_reached": {"type": "noul"},
  "navigation_progress_reasonable": {"type": "noul"},
  "navigation_failure_category": {
    "type": "choice",
    "criteria": {
      "none_success": "Goal reached.",
      "localization_related": "Localization blocks map navigation.",
      "local_planner_fault": "Controller/planner fault.",
      "goal_blocked": "Target/route blocked.",
      "timeout_or_load": "Timed out or overloaded.",
      "unknown": "Unclear."
    }
  },
  "alternate_goal_useful": {"type": "noul"},
  "recovery_agent_needed": {"type": "noul"}
}
```

#### `set_pose`

Input:

- pose name
- arm gateway result
- feedback/status

Questions:

```json
{
  "arm_pose_completed": {"type": "noul"},
  "pose_is_cleanup_or_goal": {
    "type": "choice",
    "criteria": {
      "main_goal": "This pose is the user's main goal.",
      "cleanup": "This pose is cleanup/reset/home.",
      "observation_pose": "This pose enables camera/observation.",
      "unknown": "Unclear."
    }
  },
  "arm_recovery_needed": {"type": "noul"}
}
```

#### `touch` / `grasp`

Input:

- target
- detection result
- arm gateway status
- contact/grasp result
- safety/human confirmation

Questions:

```json
{
  "target_evidence_sufficient": {"type": "noul"},
  "contact_action_completed": {"type": "noul"},
  "human_confirmation_required": {"type": "noul"},
  "contact_failure_category": {
    "type": "choice",
    "criteria": {
      "none_success": "Action completed.",
      "target_not_found": "No target evidence.",
      "arm_fault": "Arm failed.",
      "safety_block": "Safety blocked.",
      "unsupported": "Skill unsupported in current mode.",
      "unknown": "Unclear."
    }
  }
}
```

### 6.5 What to confirm on successful steps

Success should not blindly continue, but most checks should be code-level, not blocking JEV:

- Result code and `success=true` agree.
- Required artifact exists for perception steps.
- Measured motion roughly matches requested motion.
- Step result is relevant to its task step.
- No safety/recovery warning appeared in metrics.
- If this was the last skill for a subgoal, mark subgoal status:
  - satisfied
  - attempted_with_no_target
  - failed
  - skipped

Optional shadow success questions:

```json
{
  "success_result_consistent": {
    "type": "noul",
    "instructions": "Are success flag, result code, message, and metrics internally consistent?"
  },
  "subgoal_satisfied_by_this_step": {
    "type": "noul",
    "instructions": "Does this step complete its parent task subgoal?"
  },
  "continue_after_success": {
    "type": "choice",
    "criteria": {
      "continue_next_step": "Continue normally.",
      "mark_subgoal_done_continue": "Mark current subgoal done and continue.",
      "ask_human": "Ask human because success is ambiguous.",
      "stop_done": "Mission is complete."
    }
  }
}
```

---

## 7. Execution Hook Requirement

如果 JEV 判断可以继续下一步，执行必须真的挂钩。

当前实现状态：

- `NEEDS_HUMAN` / failed step / abnormal success code 会触发 blocking `step_supervision`。
- `supervision_action=continue_next_step` / `skip_failed_subgoal_continue_independent_goal` 会实际推进 `index += 1`。
- `retry_same_step` / `retry_with_alternate_view` 当前最多 retry 一次。
- `call_recovery_agent` 已接入 MissionAgent 的 recovery request/response 通道。
- `FAILED_LOCALIZATION` 分支现在会优先尊重 JEV `call_recovery_agent`，不会直接绕过 JEV supervision。

仍需注意：

- `RecoveryAgentNode` 当前是 read-only v1。它能诊断并返回 advice，但不会真正执行 A1-A3 playbook action。
- 如果没有 `/agent/recovery_response`，MissionAgent 会 fallback 到旧的 localization/local planner recovery，以便 sim 测试不中断。
- `ask_human_continue_retry_stop` 的 dashboard choice 执行闭环仍待增强。

目标：

```text
step failed or NEEDS_HUMAN
  -> JEV step_supervision
  -> action=continue_next_step
      -> MissionAgent increments index and executes next step
  -> action=ask_human_continue_retry_stop
      -> Dashboard choices
  -> action=call_recovery_agent
      -> MissionAgent publishes /agent/recovery_request
      -> wait /agent/recovery_response
      -> MissionAgent decides retry/continue/ask/stop
  -> action=stop_failed
      -> terminal failure
```

For physical next steps after failed semantic/perception subgoal:

```text
continue_next_step should usually become ask_human_continue_retry_stop
```

unless dependency policy clearly says independent and risk is read-only/low.

### 7.1 Recovery request schema currently sent by MissionAgent

```json
{
  "request_id": "rr-<mission>-<timestamp>",
  "caller": "mission_agent",
  "mission_id": "...",
  "trigger": "jev_step_supervision | precheck_failed",
  "step": {
    "index": 3,
    "skill": "goto_pose",
    "payload": {}
  },
  "result": {
    "code": "FAILED_LOCALIZATION",
    "success": false,
    "message": "...",
    "metrics": {
      "localization_warning": "GICP_SCORE_BAD",
      "localization_requires_recovery": true
    }
  },
  "required_capabilities": ["auto_navigation"],
  "suspected_components": ["localization"],
  "urgency": "blocking",
  "resources_held": ["base"],
  "user_context": "go to 02-03棚",
  "agent_context": {
    "agent_mode": "jev_primary",
    "intent": {},
    "supervision_action": "call_recovery_agent"
  }
}
```

Recovery response handling:

- `RESOLVED` or advice `resume_from_step` / `retry_step` -> MissionAgent resumes its own mission logic.
- `NO_FAULT_FOUND` / `NOT_RELATED` / `handle_as_task_failure` -> MissionAgent does not pretend recovery solved it; it asks human or handles as task/plan failure.
- `NEEDS_HUMAN` / `FAILED` / timeout -> MissionAgent asks human unless an old compatibility fallback exists.

---

## 8. Case: `check red object and go back a little`

Desired parse:

```json
{
  "task_graph": {
    "connector": "and",
    "dependency_policy": "ordered_but_independent",
    "steps": [
      {
        "step_id": "s1",
        "task_type": "search_object_here",
        "slots": {"object_query": "red object"},
        "dependency_role": "independent_goal"
      },
      {
        "step_id": "s2",
        "task_type": "move_relative",
        "slots": {"direction": "back", "distance_m": 0.2},
        "dependency_role": "independent_goal_after_observation"
      }
    ]
  }
}
```

If detection fails:

```json
{
  "failure_category": "target_not_found",
  "failure_blocks_current_subgoal": true,
  "next_subgoal_still_meaningful": true,
  "continue_requires_human": true,
  "supervision_action": "ask_human_continue_retry_stop",
  "human_choices": [
    {
      "label": "Continue: go back a little",
      "continue_from_task_step": "s2"
    },
    {
      "label": "Retry perception from another view",
      "retry_policy": "alternate_view"
    },
    {
      "label": "Stop because red object was not found",
      "terminal": "stopped_target_not_found"
    }
  ]
}
```

Reason:

- Red object not found blocks the first subgoal.
- The second subgoal is still meaningful because connector is `and`.
- But second subgoal is physical base motion, so ask human before continuing.

---

## 9. Current Implementation Gaps

Implemented / mostly implemented:

- Redacted request/response logging to `jev_calls.jsonl`.
- `command_parse` state is centered on stage goal + supported task types + default policy, not Classic/local result.
- `best_candidate_plan` removed from current JEV command_parse surface.
- Sequence parse is represented as task graph steps; opaque candidate IDs may exist inside local fallback/candidate generator but are not the JEV decision surface.
- `sequence_step_count` is a choice: `one/two/three/more_than_three/unclear`.
- `more_than_three` / `unclear` are treated as abnormal ask-human style cases.
- `dependency_policy` is attached to sequence intent.
- `plan_review` compares JEV plan vs Classic plan and uses `plan_source_decision`.
- `plan_review` no longer receives normal robot health snapshot.
- `step_supervision` is called mainly on error/abnormal result, not every success.
- `step_supervision` action is hooked into MissionAgent continuation/retry/recovery branches.
- Valid target-not-found is not treated as RecoveryAgent by default.
- MissionAgent can publish `/agent/recovery_request` and wait for `/agent/recovery_response`.
- JEV input has been trimmed: compact plan, compact result, no full artifacts/images.

Still open:

- PerceptionAgent JEV subdecisions are partial: `perception_plan_review` exists, but `perception_query_review` and `perception_result_review` are not fully implemented.
- Per-skill slot questions exist for key early skills (`move_relative`, `rotate_relative`, `look_direction`, `set_pose`) but need a full catalog-driven expansion.
- `dependency_role` per individual step is still mostly inferred in summaries, not a first-class persisted field in every MissionIntent step.
- `ask_thinking_llm` branch is not connected; it currently behaves like ask human before execution.
- RecoveryAgent is read-only v1; it diagnoses and advises but does not execute playbook recovery actions.
- Dashboard human-choice execution loop for `ask_human_continue_retry_stop` needs expansion.
- `hard_safety` / `precondition_atoms` are visible in catalog review but should become stronger runtime gates where appropriate.
- `supported_task_types` is still generated from code helpers, not yet from a dedicated task catalog.

Nice to have:

- Dashboard expandable JEV raw call details.
- Human labels for JEV better/classic better.
- Stage schema version is already included in the raw request metadata; dashboard display can expose it if useful.
