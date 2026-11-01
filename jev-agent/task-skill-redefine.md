# Task / Step / Robot Skill 重新定义

修订日期：2026-10-11。本文按本轮讨论重新建立概念和能力清单，覆盖此前的术语设计。**已完成第一版 Task 委派与部分 Skill 扩展；仍保留未实现的目标设计。** 第 4 节最后一列是本轮实际实现状态，不代表 sim/real 动作已经验收。旧 task-step-skill-inventory.md 仍可作为代码现状索引，但不再约束这里的设计。

## 1. 统一概念

| 概念 | 新定义 | 示例 |
| --- | --- | --- |
| Mission | 一次用户请求，从提交到完成、取消或失败。计划可以包含直接执行的 Step 和委派的 Task | go to shelf 06, then find red box, and touch blue dot |
| Task | 有独立目标与完成条件的任务。由负责的 Agent 进一步解释并生成执行计划；简单任务可只有一个固定 Step | 到达 shelf 06；寻找 red box；接触 blue dot |
| Step | 一次具体执行步骤；本设计中的可调用执行单元是一次 Robot Skill 调用。Task 内的 Step 称 task step | 调用 set_camera_pose 或 search_at_place |
| Robot Skill | Agent 可调用、参数可校验、过程可观察、可取消的执行能力；可以是动作、查询、检查或可复用的组合流程 | capture_observation、goto_pose、search_at_place |
| Agent | 对自己领域的目标、方法、参数与完成证据负责的规划/执行协调模块 | Mission、Navigation、Perception、Arm |
| JEV Stage | Agent 内部的判断阶段，不是用户 Task，也不自动登记为 Robot Skill | command_parse、plan_review、step_supervision |

不再使用“Mission Step”这个概念。需要定位执行位置时使用 mission_id、task_id（直接 Step 可以为空）、step_id、attempt_id。

**Skill 是能力定义；Step 是调用实例。** 同一 Skill 可以被不同 Agent、多次 Step 调用。用户口语中可以把可调用 Skill 称为 Step，但配置中分别保存 skill 名称和 step 实例 ID。

Task 与组合 Skill 的区别不是动作数量：
- Task 表达目标和完成条件，可需要语义判断、问人和更新策略。
- 组合 Skill 表达已经选定的、参数化的执行方法，有明确边界和结果契约。
- “找到 red box”是 Task；“按 fixed_nine_views 采集并用指定 pipeline 检测 red box”可以是一个 search_at_place Skill。
- 组合 Skill 的内部子调用仍要记录、校验和传播取消，但不冒充新的用户 Task。

查询型 Skill 通常立即完成；取消协议应定义取消发生在开始前、执行中、完成后的行为，不要求已经返回的查询能被撤销。Agent Skills 知识/流程包不属于本文的 Robot Skill。

## 2. 执行结构与职责

### 2.1 示例

用户请求：hi, please go to shelf 06, then find red box, and touch blue dot.

Mission Agent 的 command_parse 构建如下计划草案，而不是在解析阶段执行动作：

| 顺序 | 节点种类 | 内容 | 负责人 |
| --- | --- | --- | --- |
| 1 | 直接 Step | await_robot_auto_mode | Mission 的确定性执行器 |
| 2 | Task | navigate_to(target=shelf_06) | Navigation Agent |
| 3 | Task | search_object(target=red box, scope=current_place) | Perception Agent |
| 4 | Task | interact_with_object(target=blue dot, operation=touch) | Arm Agent |

随后 plan_review 检查顺序、目标、依赖、参数来源与能力约束，再开始执行；运行阶段统一称 step_supervision。

- Step：直接调用 Skill。明确的检查和动作不需要再问 JEV。
- Navigation Task：执行到该 Task 时才解析具体 shelf 实体、接近方向、路线和到达条件；必要时委派 Perception 做身份确认。
- Perception Task：解析对象词、颜色/实例关系、模型和观测方式，然后生成 task steps。
- Arm Task：解析接触对象和操作，获取感知证据，检查可达性与授权，执行并验证。
- Task 内部正常进展只向 Mission 上报进度。需要人工、恢复、扩大权限或最终失败时才上报相应事件。
- 普通成功按确定性流程推进；错误/结果未满足时才按需调用 JEV，不能每个成功 Step 都阻塞一次 API。

await_robot_auto_mode 只在确实需要 AUTO 的执行链前插入，不作为所有 Mission 的全局前置。只有观察任务且底盘静止时可以不要求 AUTO。执行端仍重查持续条件，不能因为计划开头通过就永久信任。

“then”先只表示顺序，不能默认后一个目标依赖前一个目标成功。是否独立、需要前一步产物、无条件收尾，应在任务关系里明确；无法确认且影响行为时问人。

### 2.2 执行图的最小区分

计划节点使用两种明确形状，不再同时用 task_type 和 skill_name 猜层级：

~~~json
[
  {
    "kind": "skill",
    "step_id": "s1",
    "skill": "await_robot_auto_mode",
    "args": {}
  },
  {
    "kind": "task",
    "task_id": "t1",
    "task_type": "navigate_to",
    "owner": "navigation",
    "goal": {"target_ref": "shelf_06"},
    "completion_policy": "registered_arrival_policy"
  }
]
~~~

这是概念协议示例。第一版已用 SkillCall.kind=task/skill 表达委派与调用，任务目标暂存在 payload（MissionIntent），内部增加不执行动作的 task_end 标记。已记录父子 ID；独立 TaskContract、revision/资源租约/预算树尚未实现。复杂 Task 可以继续委派子 Task；Navigation/Arm 可以请求 Perception，Perception 不得自行导航；禁止循环委派。

简单用户请求也可以保留一个 Task 包装，例如 get_robot_status Task -> get_robot_status Step；Mission 插入的前置检查可以直接是 Step，不必伪装成用户目标。每个执行动作只保留一个调用实例，不在 Task 和 Mission 层重复执行。

## 3. 重新命名后的 Task 清单

以下 12 类已登记到 `src/robot_agent/config/task_catalog.yaml`，供当前 JEV command_parse 选择。可识别不等于可执行；Arm 的交互/操作/升降 Task 暂明确返回 UNSUPPORTED。task type 是预先设计的；JEV 从中选择，不临时发明类型。类型是否可识别与设备/后端当前是否可用是两件事。

| Task Type（新） | 目标与主要输入 | 负责 Agent / 方式 | 完成条件 | 旧类型对应 |
| --- | --- | --- | --- | --- |
| get_robot_status | 查询当前状态及指定范围 | Mission；固定查询 Step | 返回带时间和未知项的状态 | robot_status |
| list_known_targets | 查询已登记实体，可按类别/区域筛选 | Mission；固定查询 Step | 返回当前地图下的实体列表 | list_known_targets |
| cancel | 取消当前 Mission/指定授权范围 | Mission；立即控制路径 | 在途执行已停止/取消被确认 | cancel |
| move_relative | 底盘相对移动；direction+distance 或 dx/dy | Mission；简单执行模板 | odom 位移符合目标，安全条件持续成立 | move_relative、move_relative_xy |
| rotate_relative | 底盘相对旋转；direction/angle | Mission；简单执行模板 | 实际角进度/最终误差符合目标 | rotate_relative |
| move_lift | 升降机构移动；方向、距离/允许的位置目标 | Arm；简单模板，必要时检查联动约束 | 反馈或明确的估计契约满足，不能把计时估计标为测量完成 | move_lift_bounded |
| set_arm_pose | 预设手臂姿态/手势；pose_label、repeat | Arm 确定性模板，可由 Mission 快捷调用 | 对应姿态/手势的完成证据成立 | set_pose |
| observe_scene | 观察场景；direction/pose_label/relative_delta/view_profile、scope | Perception | 所需视角到位且有效图片已取得；不要求存在特定对象 | 合并 look_direction、observe_scene、around 分支 |
| search_object | 查找或确认对象；query、scope、goal_mode=find/verify、证据要求 | Perception | 按目标返回 FOUND/CONFIRMED/NOT_FOUND/UNCONFIRMED/PARTIAL 等 | 合并 search_object_here、verify_target、到达视觉确认 |
| navigate_to | 去地点/对象附近/区域/明确 pose；target_ref、relation、arrival_policy | Navigation | 达到原目标契约，保留严格/容差/降级类别 | navigate_to_place、navigate_to_object |
| interact_with_object | 接触或按压；target_ref、operation=touch/press、交互条件 | Arm | 接触/按压结果有足够证据；命令入队不算完成 | touch_object、press_object |
| manipulate_object | 操作物体至目标状态；object_ref、desired_end_state、可选 destination、已有持物状态 | Arm | 用户指定的最终状态经过验证；内部抓取/放置/释放由 Arm 规划 | 合并 grasp_object、place_object、release_object；完整能力待开发 |

本表现在是 12 种 Task Type。任务族是有限的，参数可扩展，不为每种对象/方向/模型新增 Task Type。例如 red box、blue dot、shelf_06 都不是 Task Type。

**合并类型不等于丢掉目标差别：**
- “向左看”“向上看一点”“看看周围”都交给 observe_scene，P 决定一个视角还是视角集。
- “找 red box”“确认这里是不是 shelf 06”都交给 search_object，P 决定对象/颜色/OCR/身份关联证据。find 和 verify 是目标参数，不再是两种 Task。
- “把物体从 A 放到 B”是一个 manipulate_object Task，Arm 内部规划抓取、搬运、放置、释放、验证；Mission 不把这些强拆成多个 Task。
- 只要求“抓住它”时 desired_end_state=held；“放到 B”时为 placed_at；只要求“松开”时为 released。不能因为合成一种 Task 就默认执行整套动作，未知持物状态或释放位置不安全时必须澄清/阻断。
- 抓取、放置、释放依然保留为不同底层 Skill，便于 Arm 灵活组合、监控和取消。

### 3.1 不再保留的组合 Task 名称

- search_object_near_place -> navigate_to + search_object。
- inspect_place -> 按实际语义拆成 navigate_to + observe_scene / search_object(goal_mode=verify)；不能只靠 inspect 一词硬定识别内容。
- touch_object_near_place / press_object_near_place / grasp_object_near_place -> navigate_to + 对应 Arm Task。
- 移动中需要标签确认是 Navigation 的内部子 Task，不必总增加一个顶层 Task；用户独立要求的检查才单独编排。
- sequence 是计划结构，不是业务 Task；ask_human 是控制/澄清事件，不是机器人任务类型。
- recover_localization 属于 Recovery 能力，不默认作为普通业务 Task 暴露给自然语言任意调用。显式恢复请求需走授权恢复入口。

### 3.2 可解析与可执行分开

command_parse 可以识别 manipulate_object(desired_end_state=held)，即使当前没有抓取策略。能力检查返回结构化 NOT_AVAILABLE/UNSUPPORTED，不把它改判成“不是指令”。

已知不可用应在实际动作前报告，不必先移动、扫描再证明它不可用。Mission step_supervision/异常处理决定停止、询问或继续独立任务；**skip 是编排决定，不是 Skill 自动隐藏失败**。局部失败及未满足目标保留在最终结果里。不可逆动作尤其不能在已知缺少后续关键能力时盲目开始。

## 4. Robot Skill：可调用 Step 的目标清单

以下不是简单沿用现有 19 项，而是按可复用执行能力重新分组。名字为标准 API 的目标清单；只有“本轮状态”标明已注册/别名已注册的新增名字可以调用，其余仍不能发送。旧名字继续兼容。

状态：
- 已有：当前有相近的注册 Skill/执行器，仍需按新契约核对。
- 组合：已有子调用/辅助逻辑，可封装，但还没有这个注册 Skill。
- 新增：没有独立可调用实现；不能据表格宣称已经支持。
- 迁移：现有 Recovery/backend 等有部分实现，需统一适配；不代表所有环境可运行。
- 占位：当前接口存在，但实际能力尚不可用。

新增的“现有能力 / 补齐难度”列以本次代码核对为依据，不是在线健康状态或实测认证：

- 现有完整旧接口：可复用原能力；新命名或扩展参数仍可能未实现。
- 部分：有底层函数、节点或组合流程，但没有本表要求的独立完整 Skill。
- 无/占位：没有可执行闭环。
- 低：适配/注册已有接口及针对性回归，通常相对快；中：还需状态机、取消、证据或集成测试；高：新算法、策略、标定、碰撞/物理结果验证，不能承诺快速完成。
- 难度按本表完整契约估计，不按“能发出一条命令”估计；无工时承诺。本轮通过 mock/dry-run 与代码回归核对，未验证实际模型推理或机器人动作，sim/real 必须分别验收。

调用者缩写：M=Mission，N=Navigation，P=Perception，A=Arm，R=Recovery。下表是预想调用范围，**不是无条件授权**；实际还受资源、控制权、任务约束和安全门限制。

### 4.1 通用查询、条件与控制

| Skill | 参数 / 输出要点 | 预想调用者 | 基线与来源（改动前） | 能力范围 / 剩余补齐难度 | 本轮状态（2026-10-11） |
| --- | --- | --- | --- | --- | --- |
| get_robot_status | scope -> 当前状态、时间、unknown | M/N/P/A/R | 已有：S | 有旧接口；低：投影 scope/时间字段 | 复用旧接口；scope 投影暂缓，现有状态查询不变 |
| list_known_targets | map/category/region -> 实体列表 | M/N/P/A | 已有基本列表：E；筛选需扩充 | 有基本列表；低：增加确定性筛选 | 复用基本列表；地图/类别筛选暂缓，现有语义条目缺统一筛选字段 |
| await_robot_auto_mode | 等待策略 -> AUTO 证据；不自动切换模式 | M/N/A/R | 已有：S | 有旧接口；低：契约适配 | 复用原实现与执行门 |
| await_robot_stationary | 稳定条件/预算 -> 静止证据 | M/N/P/A/R | 已有：S | 有旧接口；低：契约适配，保留现有证据逻辑 | 复用；观测组合每个视角重查静止 |
| check_localization_health | 必要定位能力/证据新鲜度 -> 结果 | N/R，M 按需 | 已有：L | 有旧接口；低：能力需求参数适配，不放宽阈值 | 复用；未放宽定位门限或更改恢复策略 |
| check_navigation_capability | operation=relative_motion/global_navigation，必需数据/服务 -> 分项能力结果 | M/N/R | 新增领域入口，复用 S/L/N | 部分：已有模式/定位/odom/Nav2 检查；中：按操作投影，不能全局要求 loc | 已注册；按 relative/global 区分证据，相对运动不要求 loc；完整 costmap/TF 探针后续补 |
| check_perception_capability | camera、query/pipeline/profile -> 采集与推理分别是否可用 | M/P/A/N/R | 新增领域入口，复用 P/C/模型配置 | 部分：有 provider 配置及错误检查；中：部署/类别/相机时效统一 | 已注册；相机新鲜度、query、权重/依赖只读检查；实际模型加载/OCR 就绪不足时 unknown |
| check_arm_capability | operation、mode、所需反馈/策略 -> 分项能力结果 | M/A/R | 新增领域入口，复用 A/G/T/F | 部分：有 gateway/硬件状态；中：区分 preset、touch、grasp、lift | 已注册；区分预设与不支持的操作，gateway 时效；不宣称通用碰撞/物理完成 |
| request_human_input | 具体问题、选项、上下文、关联 ID -> 有效回答或取消 | M/N/P/A/R 经统一 broker | 新增统一可调用契约；不能每个 Agent 自建弹窗协议 | 部分：已有 Dashboard/恢复人工请求；中：统一关联、过期与取消 | 暂缓独立 Skill；复用 Mission NEEDS_HUMAN/选项入口，同 Task 回答与 revision broker 尚需实现 |
| cancel_execution | scope/执行 ID -> 取消确认及未停止项 | M/N/P/A/R 仅限授权子执行 | 新增统一协议；现有 cancel_motion 仅覆盖部分运动路径 | 部分：各模块取消入口；高：跨任务/子动作确认与迟到回调隔离 | 未独立注册；本轮加固现有 Mission/SkillClient 取消，不声称所有物理停止已确认 |
| stop_base_motion | 执行上下文 -> 停止请求及停止证据 | M/N/R；P/A 仅经资源协调 | 组合；现有 cancel_motion/导航停止逻辑：N | 部分：已有取消/零速度；中：统一实际停止证据 | 暂缓独立 Skill；停止请求已有，统一新鲜 odom 停止证据/执行所有权需补 |
| check_execution_evidence | 原始完成契约 + 结果证据 -> satisfied/unsatisfied/unknown | M/N/P/A/R | 新增按领域 validator 分派；不是让 JEV 自由宣布成功 | 部分：各执行器结果判断；中：统一分派和 unknown 语义 | 暂缓通用 Skill；已按 Task 聚合每个 Step 最新结果与子 Task 结果，领域完整 validator 尚需补 |

安全门是执行框架不可跳过的基础机制，不靠 Agent“记得选择一个安全检查 Skill”才存在。request_human_input 是可复用交互流程，但不是物理机器人动作。

#### capability 检查：领域分开，底层共用

建议对 Agent 暴露上面三个领域 Skill，而不是只有一个含义模糊的 check_capability。底层共用一个只读检查框架/证据库；内部可以保留 check_capability(domain, operation, requirements) 分派函数，但不再把它与三个入口重复登记为第四个需要执行的 Step。

| 领域 | 基于当前机器人应该检查的内容 | 不应要求的内容 |
| --- | --- | --- |
| Navigation | relative_motion 检查 odom/控制与必要安全数据；global_navigation 另查定位/TF、地图、Nav2 action/lifecycle 与相关 costmap | 相对运动不能因为 GICP 不健康一律拒绝 |
| Perception | 所选 RGB/深度/CameraInfo 数据是否新鲜；provider/权重/依赖实际可用；query 所需 OCR/类别/颜色能力；需要转云台时才查反馈 | 纯 RGB 观察不强制深度/OCR，也不强制 AUTO；不在检查时自动下载或运行重模型 |
| Arm | 对应 preset/轨迹/Touch Anything/夹爪/升降接口、模式、反馈、标定及所需策略；grasp 占位明确标 unsupported | 有关节动作不等于有抓取/完整碰撞/接触验证能力 |

结果至少分开 supported（实现上支持吗）、available（当前部署可用吗）、ready（当前前置满足吗），每项允许 unknown，并附 reason_code、evidence_refs、observed_at/valid_until。健康不是全部能力：模型没有目标类别、抓取未实现也属于能力缺口，但不应触发节点重启。

这三个 Skill 只检查，不切 AUTO、不运动、不启动恢复。Mission 可做低成本预检，领域 Agent 细化参数后只补查变化/缺失项，避免重复整套检查。缓存按时间、配置/证据 revision 失效；执行端的实时安全门始终重查。

### 4.2 底盘、导航与路线

| Skill | 参数 / 输出要点 | 预想调用者 | 基线与来源（改动前） | 能力范围 / 剩余补齐难度 | 本轮状态（2026-10-11） |
| --- | --- | --- | --- | --- | --- |
| resolve_navigation_target | target_ref/relation -> 可区分实体候选及歧义 | N | 已有 resolve_semantic_target：E；新接口需保留多候选 | 有 resolve_semantic_target；中：多候选/歧义契约 | 别名已注册；复用 resolve_semantic_target；多候选消歧在 Navigation.refine 返回 choices |
| compute_navigation_goal | 实体、arrival/approach policy -> 合法 pose/region 候选 | N | 组合；现有语义/导航 helper：E/N | 部分：已有 semantic/approach helper；中：提取与几何验证 | 暂缓；区域/接近计算耦合 SemanticMap，贸然独立会改变当前到达策略 |
| plan_navigation_path | 当前 pose、goal、允许路点 -> 实际路径及 planner 结果 | N | 组合；Nav2 调用封装待独立注册 | 部分：N._compute_global_path/ComputePathToPose；中：独立取消与返回契约 | 暂缓；需独立 action 取消/路径结果契约，不能直接暴露私有 helper |
| inspect_navigation_path | 实际路径、footprint、规则 -> 窄段/冲突/未知证据 | N | 新增独立契约；已有部分 annotation/路径逻辑不等于完整实现 | 部分：地图/footprint/annotation 检查；高：完整路径净空与规则验证 | 未实现完整接口；需几何净空与规则证据，非快速工作 |
| navigate_to_pose | 校验后的 pose/arrival policy -> 分类到达结果 | N；M 可执行已审查固定计划 | 已有 goto_pose：N | 有 goto_pose；低：名称适配；新 arrival 契约为中 | 别名已注册；复用 goto_pose，保留原完成分类，不新增到达算法 |
| follow_waypoints | 已校验路点序列、各段契约 -> 分段进度/结果 | N | 新增组合 Skill；每段重新验证 | 部分：已有路线/路点逻辑；中：独立序列状态机、取消和续行 | 暂缓；需分段 checkpoint、取消和不重放契约，继续使用现有导航内部路线 |
| apply_navigation_profile | 已登记 profile ID -> 生效/恢复确认 | N/R | 新增受限接口；禁止任意修改 footprint/安全阈值 | 部分：N._publish_narrow_motion_profile/_clear_narrow_motion_profile；中：白名单和退出回滚 | 暂缓；需配置生效确认、资源所有权和退出回滚，不能只封装 publish |
| move_base_relative | direction+distance 或 dx/dy -> 实际位移结果 | M/N/R；A 仅获准底盘联动时 | 已有 move_relative / move_relative_xy：N；统一参数入口 | 有两个相对移动接口；低：统一参数和回归 | 别名已注册；direction 或 dx/dy 分派，复用原 odom/安全逻辑 |
| rotate_base_relative | angle/policy -> 实际旋转结果 | M/N/R | 已有 rotate_relative：N | 有 rotate_relative；低：名称适配 | 别名已注册；复用原旋转执行器 |
| verify_navigation_arrival | 原完成契约 + pose/Nav2/视觉证据 -> 分类结果 | N | 组合；现有部分到达逻辑：N | 部分：N._task_goal_reached 等；中：提取并保留成功分类 | 暂缓独立 Skill；原 goto_pose 的确定性验收保留，避免复制阈值 |

相对运动不因全局定位丢失自动失效；但 Navigation 不能用相对运动绕过自身“必须有可靠定位”的任务约束。R 的移动也需要受限 playbook 授权，不恢复旋转找定位的旧策略。

### 4.3 相机、观测与识别

| Skill | 参数 / 输出要点 | 预想调用者 | 基线与来源（改动前） | 能力范围 / 剩余补齐难度 | 本轮状态（2026-10-11） |
| --- | --- | --- | --- | --- | --- |
| set_camera_pose | label 或绝对 pan/tilt 或相对 delta；反馈策略 | P；N/A/R 用已确定姿态 | 已有绝对姿态：C；label/relative 统一契约需补 | 有绝对角度及姿态配置映射；中：相对反馈坐标、限位、时效与取消 | 复用；新增取消检查。label/relative delta 统一接口暂缓，须核实反馈坐标与限位 |
| capture_observation | camera/view ID、新鲜度条件 -> 图片及元数据引用 | P/A；N 委派或已审查调用 | 已有：P | 有旧接口；低：原相机接口适配；多相机支持为中 | 复用并补 Task 证据域、view_id 和取消检查；多相机路由未补 |
| move_camera_and_capture | 指定姿态 + 图片条件 -> 到位后的新鲜图片 | P/A | 组合：set_camera_pose + capture_observation | 部分：builder 已串联；中：到位后新帧保证和取消 | 已注册；静止 -> 到位反馈 -> 新帧，传播取消 |
| capture_view_set | 登记 profile/view 列表 -> 各 view 的图片与有效覆盖 | P | 组合：现有 Perception builder | 部分：已有固定视角循环；中：覆盖计数/失败/取消统一 | 已注册；从 camera_scan_profiles.yaml 读取视角，返回子调用与覆盖；不动态创设视角 |
| detect_object | 图片引用、类型化 query、provider/pipeline、policy -> 检测实例/错误 | P/A | 已有 detect_known_object：P；改名并完善契约 | 有 detect_known_object；中：类型化 query、指定图片集和后端检查 | 别名已注册；复用现有检测和 Task 图片域；显式任意图片集/统一类型化 query 暂缓 |
| read_text | 图片/ROI、语言/数字匹配 policy -> OCR 文本及位置证据 | P | 组合；现有 OCR 后端：P，待独立接口 | 部分：已有 OCR 后端；中：提取接口和部署检查；可读率另验 | 暂缓；OCR 与现有 detector 聚合耦合，需单独 ROI/后端/错误契约 |
| measure_object_attributes | 实例 ROI、颜色等已支持属性 policy -> 属性证据 | P | 组合/扩充；已有部分颜色分析：P | 部分：已有颜色分析；中：限定现有颜色范围；通用属性为高 | 暂缓；颜色规则尚非独立实例属性契约，不能只返回颜色就验证对象 |
| associate_target_evidence | 对象/文字/颜色/ROI -> 同实例关联及未满足条件 | P | 新增统一契约；防止跨图误拼 red + box | 部分：已有组合 query 分析；高：跨实例/跨视角身份关联不能仅拼结果 | 未实现通用接口；跨图/跨实例关联需要额外数据与负例 |
| locate_interaction_target | 实例/像素点 + 标定/深度 -> 交互目标及不确定性 | P/A | 新增统一契约；现有 touch pipeline 需适配，不保证已有通用 3D 输出 | 部分：T 的视觉/几何/TF 流程；高：通用目标输出及标定不确定性验证 | 未实现通用接口；底盘相机 bbox 不能直接作为 arm camera 交互坐标 |
| search_at_place | query、camera、profile、model pipeline、完成策略 -> 图片集和搜索结果 | P | 组合；封装现有固定扫描+识别，当前未独立注册 | 部分：已有九视角+识别计划；中：组合执行器、证据及取消，不新增识别算法 | 已注册；一个组合 Skill 封装九视角与指定 provider，失败/取消保留已取得图像 |
| verify_visual_target | 原身份/属性契约 + 现有图像/检测 -> confirmed/unconfirmed/ambiguous | P，受 N/A 委派 | 新增/组合；区别于一次 detect_object 调用 | 部分：检测/OCR 输出可复用；中至高：取决于是否要求货架/实例关联 | 暂缓独立接口；现有 OCR 匹配不能证明通用目标/货架实例身份 |

**固定视角是参数，不是 Skill 类型。** ground_left、middle_center、return_home 等是 profile/label；“往上看一点”由 P 解析成 relative delta/policy。set_camera_pose 必须基于新鲜当前姿态把相对目标转换为受限绝对目标；反馈缺失则等待/失败，不能从默认姿态猜当前位置。

本轮已注册的 search_at_place 是确定的可复用方法，不在内部重新理解整句用户命令。P 先确定 query、模型及 profile，再调用它；固定九视角默认完成九个有效视角，底盘保持静止。需要灵活补拍/换模型时，也可直接组合 move_camera_and_capture、detect_object 等较小 Skill。

组合 Skill 自身有一个外部 Step ID，内部每次调用有 child_step_id；取消传播到所有子调用，不在取消后自动归位或继续下一张。复用图片必须核对 query/model/观测有效期，不复制完整证据文件。

### 4.4 手臂与升降

| Skill | 参数 / 输出要点 | 预想调用者 | 基线与来源（改动前） | 能力范围 / 剩余补齐难度 | 本轮状态（2026-10-11） |
| --- | --- | --- | --- | --- | --- |
| set_arm_pose | pose_label、repeat -> 姿态/动作完成证据 | A/M；P/R 使用已批准姿态 | 已有 set_pose：A | 有 set_pose/gateway preset；低：原预设适配，不新增任意姿态 | 别名已注册；兼容 pose_label/name，预设姿态通过 Arm Task 执行 |
| move_lift | 方向/距离或受支持目标高度 -> 反馈/估计明确区分 | A/M；P/N 仅经任务授权 | 已有 move_lift_bounded：F；高度闭环不能假定已具备 | 有 bounded 命令/计时 STOP，返回 COMMAND_ACCEPTED；中：等待结果；可靠高度闭环需另核实，非低 | 别名已注册但仍 COMMAND_ACCEPTED；自主 Task 暂拒绝，计时 STOP 不等于高度完成 |
| check_arm_reachability | 目标、当前状态、标定、约束 -> 可达/不可达/未知 | A | 新增统一检查接口 | 部分：T 局部 IK、G 关节校验；高：通用可达性/环境碰撞不完整 | 未实现通用接口；IK helper 不等于环境碰撞验证 |
| plan_arm_interaction | 已定位目标、touch/press/grasp policy -> 允许的动作方案 | A | 新增；现有 touch pipeline 仅为部分来源 | 部分：T 接近/按压求解及 gateway 验证；高：通用交互规划，不能套用为抓取 | 未实现通用接口；需要交互目标/标定/结果闭环 |
| touch_target | 明确交互目标、touch policy -> 接触结果证据 | A | 已有 touch(mode=touch)：A；当前完成语义需补强 | 有触发接口和 T 流程；中至高：闭环接触证据/标定及安全验收 | 别名已注册；保留原 touch 队列语义，新自主 Task 暂不执行 |
| press_target | 明确目标、已批准按压 policy -> 按压结果 | A | 已有 touch(mode=press)：A；不是任意施力接口 | 有 press 触发和局部推进逻辑；中至高：动作结束不等于物理按压成功 | 别名已注册；mode 固定 press，不宣称已物理按压成功 |
| grasp_target | 明确目标、已配置 grasp policy -> 抓取结果 | A | 占位 grasp：A；不能计为已实现抓取 | 仅 grasp 占位；高：缺抓取策略、持物验证与整链验收 | 未实现；原 grasp 仍占位，自主 manipulate_object 明确 UNSUPPORTED |
| place_held_object | 已持物状态、目标、放置 policy -> 放置/释放结果 | A | 新增 | 无完整放置 Skill；高：持物状态、轨迹、目标与释放验证 | 未实现；无持物/放置闭环 |
| release_gripper | 授权位置/释放 policy -> 夹爪反馈 | A/R 在授权流程内 | 新增统一接口；先确认硬件与能力支持 | 部分：夹爪关节可经 gateway 控制；中：受控开夹适配；安全释放/已释放证明为高 | 暂缓；虽可动夹爪，但释放授权、持物状态和停止反馈不完整 |
| verify_arm_interaction | 交互契约 + 反馈/视觉等 -> 完成/失败/未知 | A，可委派 P | 新增；不能用 COMMAND_ACCEPTED 代替完成 | 部分：关节/流程状态；高：接触、持物、放置的物理证据未形成统一契约 | 未实现物理完成统一契约；不拿命令 ACK 代替 |
| cancel_arm_motion | 当前 action ID -> 停止/取消证据 | A/M/R 通过 broker | 新增统一契约，需核实各底层接口 | 部分：G._cancel_active_goal/_visual_servo_stop；中：联动 T、停止确认及保持策略 | 未独立注册；已把 Touch exit + visual_servo_stop + gateway cancel 纳入 cancel_motion，停止确认待验 |

Skill 的力、距离、速度等参数只能来自硬件已支持且校验过的配置。没有力传感器不能要求“验证施力成功”；证据不足必须返回 unknown，而非用超时结束冒充完成。

### 4.5 Recovery 专用能力

| Skill | 参数 / 输出要点 | 预想调用者 | 基线与来源（改动前） | 能力范围 / 剩余补齐难度 | 本轮状态（2026-10-11） |
| --- | --- | --- | --- | --- | --- |
| request_recovery | 故障证据、失败执行、能力需求、checkpoint -> 中间/最终回复 | M；N/P/A 经 M 协调 | 迁移：已有 Mission/Recovery 请求协议；不让领域重复发请求 | 有请求/中间及最终回复机制；中：封装成统一可取消 Skill | 后置；保留原 Mission 请求协议，不新设领域恢复入口 |
| recover_localization | 已批准恢复策略 -> 定位验证结果/人工需要 | R | 迁移：PB-LOC-LOST/R；旧 recover_localization_spin 不作为新语义 | 有 PB/后端恢复流程；中：适配统一 Skill，必须保留验证与延后调度 | 后置；保留原 Recovery，不新增别名 |
| compute_recovery_pose | 可信 pose+同时间 odom+当前 odom -> 恢复参考及可信度 | R | 迁移：已有可信 pose/odom helper，需接口化 | 有 trusted_pose/helper；低：只读接口化及时间配对回归 | 后置；本轮不改 Recovery 的可信 pose/odom 逻辑 |
| publish_initial_pose | 已验证 seed、frame/time -> 发布记录，不等于恢复成功 | R | 迁移：R backend | 有 R._publish_trusted_pose；中：受控 seed 参数化与取消竞态 | 后置；不新增独立任意初值发布入口 |
| verify_localization_recovery | 新匹配、时间戳、有效扫描/稳定条件 -> 验证结果 | R | 迁移：已有 health/probe 逻辑 | 部分：已有健康 probe/验证；中：整理新证据窗口，禁止旧健康值误通过 | 后置；沿用原验证，不改证据窗口 |
| restart_component | 白名单 component/env -> 重启结果及证据 | R | 迁移：R；没有重启命令时转人工 | 有配置化后端；低：仅已有 restart 命令组件；新增组件为中/待确认 | 后置；保留既有白名单 backend |
| reset_navigation_lifecycle | 授权 lifecycle manager -> 状态结果 | R | 迁移：R lifecycle_reset | 有后端；低：适配及状态验证回归 | 后置；保留原 backend |
| clear_navigation_costmap | local/global scope -> 服务结果 | R/N 受策略限制 | 迁移：R clear_costmap | 有后端；低：接口适配，清图不保证故障解决 | 后置；保留原 backend |
| collect_diagnostic_evidence | incident/task 范围、时间窗 -> 脱敏证据索引 | R/M | 新增统一接口；复用现有 report，不复制所有 heartbeat | 部分：已有 human_report/事件记录；中：范围裁剪、脱敏及引用复用 | 未独立注册；Human report 已追加 tasks/outcome/owner/step 引用，不复制图片 |

恢复不默认调用 move_base_relative/rotate_base_relative，也不能随意重设初值。普通定位故障在非定位依赖任务期间延后；任务结束/取消且停止确认后，或进入定位依赖任务前，按原恢复策略介入。

## 5. 实现来源与迁移对应

以下代号仅用于上表引用，不是新包名。

| 代号 | 当前源文件 / 定义 |
| --- | --- |
| S | [src/robot_skills/robot_skills/executors/system.py](../../src/robot_skills/robot_skills/executors/system.py)：SystemSkills |
| E | [src/robot_skills/robot_skills/executors/semantic.py](../../src/robot_skills/robot_skills/executors/semantic.py)：SemanticSkills |
| L | [src/robot_skills/robot_skills/executors/localization.py](../../src/robot_skills/robot_skills/executors/localization.py)：LocalizationSkills |
| N | [src/robot_skills/robot_skills/executors/navigation.py](../../src/robot_skills/robot_skills/executors/navigation.py)：NavigationSkills |
| C | [src/robot_skills/robot_skills/executors/camera.py](../../src/robot_skills/robot_skills/executors/camera.py)：CameraSkills |
| P | [src/robot_skills/robot_skills/executors/perception.py](../../src/robot_skills/robot_skills/executors/perception.py)：PerceptionQueryAnalyzer / PerceptionSkills |
| A | [src/robot_skills/robot_skills/executors/arm.py](../../src/robot_skills/robot_skills/executors/arm.py)：ArmSkills |
| F | [src/robot_skills/robot_skills/executors/lift.py](../../src/robot_skills/robot_skills/executors/lift.py)：LiftSkills |
| G | [so101_arm_control/arm_gateway.py](../../src/so101_arm_control/so101_arm_control/arm_gateway.py)：关节候选验证、trajectory action、取消、visual servo stop |
| T | [touch_anything/README.md](../../src/touch_anything/README.md)、[touch_kinematics.py](../../src/touch_anything/scripts/touch_kinematics.py)、[touch_observation_node.py](../../src/touch_anything/scripts/touch_observation_node.py)：局部 IK、目标跟踪与接近/按压流程；README 明确无完整环境碰撞模型 |
| 模型配置 | [perception_models.yaml](../../src/robot_skills/config/perception_models.yaml)：YOLO/OCR 配置不等于权重/依赖已经部署；本轮未验证实际加载 |
| R | [recovery/backends.py](../../src/robot_agent/robot_agent/recovery/backends.py)、[recovery/executor.py](../../src/robot_agent/robot_agent/recovery/executor.py)、[PB-LOC-LOST.yaml](../../src/robot_agent/config/recovery/playbooks/PB-LOC-LOST.yaml) |
| 当前组合计划 | [agents/perception_agent.py](../../src/robot_agent/robot_agent/agents/perception_agent.py)、[navigation_agent.py](../../src/robot_agent/robot_agent/agents/navigation_agent.py)、[arm_agent.py](../../src/robot_agent/robot_agent/agents/arm_agent.py) |

当前默认运行时注册表与契约表由旧 19 项 + [skill_extensions.yaml](../../src/robot_skills/config/skill_extensions.yaml) 合成，共 34 个可调用名字：新增 9 个兼容别名、3 个能力检查、3 个观测组合。**别名不是新增物理能力。**
[extensions.py](../../src/robot_skills/robot_skills/extensions.py) 同时供 registry、catalog、SkillClient 使用，避免三份名单不同步。显式加载自定义 registry/catalog 文件仍只加载该文件，未自动混入扩展。
[task_catalog.yaml](../../src/robot_agent/config/task_catalog.yaml) 与 [task_catalog.py](../../src/robot_agent/robot_agent/task_catalog.py) 管理 12 类 Task 及旧名适配；SkillCall 名与 Task 类型不共享白名单。
组合执行器：[observation.py](../../src/robot_skills/robot_skills/executors/observation.py)；能力证据：[capabilities.py](../../src/robot_agent/robot_agent/capabilities.py)。
本轮记录与扩展流程见 [实施记录](mission-agent/test&pro&solve/11-domain-agents-20261011.md)。

Task/Skill 可以同名，但必须由 kind 和独立 catalog 区分。为避免语义混淆，复杂 Task 尽量用目标名（navigate_to、interact_with_object），底层 Skill 用具体操作名（navigate_to_pose、touch_target）。

## 6. 是否需要 Arm Agent

**建议需要，但扩展现有 ArmAgent，而不是再创建一个平行重复的模块。**

现有 [agents/arm_agent.py](../../src/robot_agent/robot_agent/agents/arm_agent.py) 已有固定计划 builder。真正需要补的是领域责任：

1. 确定 touch blue dot 指哪个可交互目标、触碰还是按压，是否存在歧义。
2. 委派 P 获取目标实例/交互位置，确认标定、坐标系、可达性与碰撞约束。
3. 选择已经实现且授权的接触/抓取策略，编排观察姿态、检测、接近、动作与验证；manipulate_object 内部的抓取、放置、释放由 Arm 规划及监控，不交回 Mission 拆分。
4. 处理本 Task 内有限重拍/重定位目标；系统故障请求 Recovery；不能把“看到了目标”当“可以触碰”。
5. 返回完成证据和未解决问题，由 Mission 决定其他任务是否继续。

set_arm_pose(home) 这种固定任务无需 JEV，可由 M 直接调用同一 Skill；复杂交互才进入 Arm 的细化流程。Arm Agent 不必独立 ROS 进程，也不需要每次多一次网络调用。

拟议 JEV 决策点：interaction_goal_review、target_evidence_review、interaction_strategy_review、interaction_result_review。只在语义/合法策略/结果存在判断需求时使用；可达性、碰撞、关节极限、硬件状态由确定性检查决定，不让 JEV 投票覆盖。

## 7. 后续实现边界

- 以本机器人已有能力为第一批：状态/静止/AUTO、相对移动、goto_pose、云台与 RGB 采集、已配置识别后端、预设手臂动作、已有 Recovery backend。已有接口也必须保留完成语义与回归，不因重命名自动算验收。
- 第二批补领域能力检查、固定观测/搜索组合、human/recovery broker 和跨任务取消，属于集成工作，不能全部归为“简单封装”。
- 通用环境碰撞、完整抓取/放置、物理接触/持物验证作为后续能力，不纳入当前 baseline 的已支持范围。类型仍可解析，但应明确能力不足。
- 本表是分期设计，不要求一次实现所有候选 Skill；优先做代码里已有且能形成闭环的能力。
- Task Catalog 保存目标、owner、参数、完成条件及可识别类型；Skill Catalog 保存可调用接口、权限、资源、前置/持续/完成条件、结果、取消和预算。
- 同一组合 Skill 只维护一份执行实现，P 可选整套调用或较小 Skill 组合，不在 Mission/Perception 分别复制九视角代码。
- Task/Skill 的可用性由运行时能力检查反馈，不通过删掉自然语言选项掩盖能力缺失。
- Mission step_supervision 是跨计划的监督阶段；Task 内的执行与局部决策由 owner 负责。各层不能并发对同一失败重复恢复、问人或重试。
- 新结果契约至少区分 execution_state、goal_satisfied、outcome、evidence_refs、error/required_capability。NOT_FOUND、UNSUPPORTED、健康故障、取消分别处理。
- 本轮已同步 Mission / Navigation / Perception README，并新增 Arm README；旧 inventory 是历史实现索引。以本表实现列、对应源码和本轮测试记录区分已实现与规划。
