# 2026-10-11 Task / Domain Agent 第一版

本轮范围：统一 12 类 Task、实现可快速完成的 Robot Skill 扩展、接入领域委派及 JEV decision、Arm 确定性边界。Recovery 保持独立，不重写 PB/backend，不启动 thinking LLM。

## 1. 变更清单

| 内容 | 实际状态 |
| --- | --- |
| Task Catalog | 新增 robot_agent/config/task_catalog.yaml；12 类；旧名/复合任务在 task_catalog.py 适配 |
| Mission 计划 | kind=task 延迟细化；kind=skill 执行；task_end 只聚合结果，不是能力 |
| 可调用名称 | 原 19 + 9 个旧能力别名 + 3 个只读能力检查 + 3 个观测组合 = 34；registry/catalog 由同一 manifest 扩展 |
| 观测组合 | move_camera_and_capture、capture_view_set、search_at_place；九视角固定 profile；新帧、取消与部分证据保留 |
| Navigation | 执行时目标消歧、nav_target_review、原 Nav2 计划、可委派 Perception 子 Task |
| Perception | 目标/后端快查、perception_model_review、perception_observation_review、指定 provider 写入执行计划 |
| Arm | set_arm_pose 确定性执行；touch/press/manipulation/lift 自主 Task 暂不支持，反馈真实能力不足 |
| UI/report | Tasks 归属、父子 ID、goal_satisfied/outcome；Arm Agent 状态；Skill 结果携带 task_id/step_id/owner |
| 取消 | Mission/SkillClient run tombstone；组合子调用/相机/Arm 等待检查；不在取消后归位 |

状态与跳过原因已逐项更新到 [task-skill-redefine.md](../../task-skill-redefine.md) 的第 4 节最后一列。该表共有 53 个目标 Skill，不表示 53 项全实现。

## 2. 本轮发现并修正

1. 父计划原先直接展开领域内部调用，无法清楚区分一次目标与二十多个相机动作。新增 Task envelope，在执行边界才 refine。
2. 同 Mission 多个搜索共用图片域。改为 Task evidence_id，组合每次尝试再分观测 epoch，重试不把旧尝试图片拼成新覆盖。
3. 失败后若直接 index+1，会继续同 Task 的后续动作。继续独立目标时跳过整个失败 Task 的剩余内部步骤。
4. Step 多次尝试全部参与 all(success)，重试成功仍未完成。结果聚合改为每个 step_id 最新尝试，历史保留。
5. 定位恢复成功被当成失败动作完成，直接跳过检查/导航。Mission 只对定位检查/绝对导航有界重试，不把 Recovery RESOLVED 当作 Task 成功，也不重放相对动作。
6. Arm 发布者不存在在非 dry_run 时也可能伪报 dry-run 成功；改为明确失败。取消时补 Touch/gateway stop/cancel，保持策略不附带 release_hold。
7. Classic unknown destination 在 Mission 阶段退回解析，领域无法澄清。Mission 启用 defer_target_resolution；明确 go to 的地点词传给 Navigation，旧独立 parser 默认行为保留。
8. 未缓存 embedding 模型可能联网下载并卡住。默认 local_files_only；加载失败缓存错误。Primary 的对比只用 deterministic，Classic 结果仅在 plan_review 参与。
9. 模式运行中变化可能混用策略。冻结本 Mission mode，Shadow 不采纳领域 decision。
10. plan_review 的 Classic 对比计划由临时状态生成，现明确继承当前 mission_id，保证切换方案后 Task/证据仍属于原 Mission。

## 3. 验证方法

本轮**未操控 Isaac/真实机器人，未实际调用 JEV 网络 API，未实际加载视觉模型**。以下只证明确定性结构与 mock/dry-run 行为。

### 3.1 自动化测试

新增测试：
- src/robot_agent/test/test_domain_tasks.py：Task 名称、延迟委派、嵌套 N->P、取消、失败停止、显式 cleanup、模式冻结、未知目标、重试与恢复反馈。
- src/robot_agent/test/test_domain_capabilities.py：相对运动不依赖 loc、感知不统一要求 AUTO、抓取不支持、别名/取消。
- src/robot_skills/test/test_observation_composites.py：34 项契约一致、九视角、无效 provider 无动作、取消、新尝试图片隔离。

命令（仓库根目录）：
~~~bash
source /opt/ros/humble/setup.bash
source install/setup.bash
export PYTHONPATH="$PWD/src/robot_agent:$PWD/src/robot_skills:$PYTHONPATH"
python3 -m pytest src/robot_agent/test src/robot_skills/test -q
~~~

最终全量结果：**284 passed / 8 failed**（8.84 秒）；新增的 27 项均通过。失败集中在 test_real_lab_parser_semantic.py 的 8 项：
- 6 项依赖 maps/l402_2d_map_clean_0509 下当前缺失的地图/标注配置，或该配置对应的别名选择。
- 2 项仍期望旧相机角度映射/return_home 标定值，与当前配置不符。

本轮未改这些地图/标定，不通过回滚用户标定来“修绿”测试；也未在修改前保存全量对照，故不宣称已证明它们全部是历史失败。需要后续单独整理测试 fixture。

### 3.2 三模式 dry-run（共 18 组）

使用 DryRunSkillClient，显式清空 JEV_API_KEY、HF_HUB_OFFLINE=1；不调用 ROS、不移动机器人。

| 命令 | 三个 mode 的实际结果 | 验证点 |
| --- | --- | --- |
| robot status | DONE / SATISFIED，get_robot_status owner=mission | 12 类新名字转换 |
| go back a little then set pose home | DONE / SATISFIED，move_relative -> set_arm_pose | 简单执行 -> Arm 委派 |
| 左を見て | DONE / SATISFIED，observe_scene owner=perception | 单方向观察委派 |
| find red box | DONE_WITH_STUB，search_object | 不将模拟推理计为真实检出 |
| go to shelf_06 | NEEDS_HUMAN，navigate_to owner=navigation | 本次默认语义地图没有该可直接执行目标；不猜坐标 |
| pick up red cube | NEEDS_HUMAN / UNSUPPORTED，manipulate_object owner=arm | 可识别而不可执行，无动作 |

真实 API choice 采用与 Shadow 不采用的分支用 mock 测试覆盖；上述 dry-run 不能用来评价真实 JEV 准确率或延迟收益。

### 3.3 构建与浏览器

- npm run build：通过 TypeScript/Vite 构建。
- colcon build --packages-select robot_skills robot_agent --symlink-install：两包构建通过。
- Playwright 注入 mock WebSocket 状态：1365x900、390x844；3 个 Task 折叠项均存在，Task 容器无水平溢出/纵向裁切、无 JS 错误。初次截图发现继承 section 样式造成内部裁切，已改为不带外框的普通列表并复测。
- 临时截图：/tmp/agent-task-desktop.png、/tmp/agent-task-mobile.png；仅界面模拟，不作为实际 ROS/机器人运行证据。

## 4. 下一轮 sim 验收

重启 Dashboard/agent 使新代码与配置生效，浏览器访问 http://127.0.0.1:8765 。本轮构建不会自动重启用户的在途机器人进程。

| 顺序 | 操作 | 应观察到的分工/反馈 |
| --- | --- | --- |
| 1 | robot status；set pose home | Mission 简单查询；Arm 单 Task，有真实姿态结束结果 |
| 2 | 左を見て；find red box | Perception Task；后者一个 search_at_place，9 个采集结果可追踪 |
| 3 | go back 0.2 m then set pose home | 相对位移后才委派 Arm；当前 Task 的 loc 要求不扩大 |
| 4 | go to door（地图多门时） | Navigation 提澄清，不自行换门；当前选项沿用重发机制，不保证保留剩余任务 |
| 5 | go to 一个已校准目标 | Navigation 既有完成分类不变；需要标签时出现 parent_task_id 关联的 Perception |
| 6 | 搜索中、预设中分别 Cancel | 停止后续子调用，无取消后归位，无旧 JEV 回复复活；检查实际停止反馈 |
| 7 | find red box then set pose home regardless of detection | 未检出时允许明确收尾，整体 PARTIAL，前目标未完成保留 |
| 8 | 同一命令对比 Classic/Primary/Shadow | API 真实成功才统计 live；Shadow 无额外动作，记录实际 provider 和采纳选择 |

## 5. 明确暂缓

- 完整领域自治：动态局部重规划、受限补拍、revision/资源租约/父子预算树。
- Human broker：同 Task 多轮澄清与原 Mission 剩余计划续行；现有 choices 为兼容入口。
- Navigation：新路线规则 DSL、路径净空/窄通道 JEV、动态维修禁行；原有导航规则保留。
- Perception：类型化 query 的全约束验证、同实例/货架身份关联、相对云台、多相机、OCR 独立 Skill。
- Arm：Touch/升降的 action 终态与物理结果、通用可达性、抓取/放置/释放。无完整证据前不冒充可执行。
- Recovery：不改变 PB verified，不新增故障注入。本轮只修 Mission 端成功回复不能跳过动作的问题。

## 6. 扩展流程

Task：task_catalog.yaml 登记目标/owner/slots -> task_catalog.py 兼容（如需要）-> owner.refine -> JEV 按类型追问 -> 完成/失败/取消测试。catalog 不负责声明动态硬件在线状态。

Robot Skill：旧 API 兼容用 skill_extensions.yaml 的 alias；独立能力补 registry/catalog 合同、执行器和 SkillClient/DryRun 分派。默认 loader 合并 manifest，自定义 registry 文件不自动扩展。

组合 Skill：在 observation.py 或对应领域模块定义有限执行图；输入已确定 query/policy，不再次理解整句命令；子调用带 ID、取消和证据，重试用新 epoch。

新领域 JEV：固定 stage/purpose、裁剪 state、有限合法 choices、非法/失败路径、Primary 的实际执行挂钩、Shadow 不采纳测试；必须说明 stage 是否已实现，不能只有文档/trace。
