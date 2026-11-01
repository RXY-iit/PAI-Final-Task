# 四个任务 Agent 实测与修复

日期：2026-10-12。本文只记录证据，不把测试定义当作测试通过。

后续真实 Primary 对照及新增修复见 [16-primary-suite-20261012.md](16-primary-suite-20261012.md)。本文保留当时 Classic 的环境/授权状态，不代表当前仍关闭 API；后续已按用户新请求恢复 live Primary。

## 环境与边界

- Full / `nav2`，地图 `sim_factory_mid360_like_gpu_trueimu_011`。保留 Isaac、Nav2、定位和驱动；为加载 Python 修复重启过 Agent/Dashboard/Health/Recovery 四节点。
- 本轮现场任务使用 **Classic**。新的真实 JEV 测试授权尚未收到；此前有限次回放授权已用完，未继续发送任务数据。后续测试进程显式移除 JEV key，防止启动 probe。配置文件中的 key 和默认 Primary 设置未改。
- 因此本轮现场数据证明确定性规划/执行/委派的行为，不证明真实 JEV 比 Classic 更好。Primary/Shadow 的异常回复、委派和采纳边界另由离线回归覆盖。
- RGB 实际输出 640x480，不是预期的高分辨率 1280x720。Arm gateway 为 MOCK，pose 成功不等于实机验证。
- 运动前读取局部 costmap；旋转前前移 0.5m，最近致命障碍距离约 1.09m。M07 只读路径预检：约 1.26m、48 点、中心采样最大代价 0、无 unknown；这不替代 Nav2 footprint/collision 保护。
- 回退/转向/取消前后的 wheel pose 记录在摘要中。取消后新命令须有新的 mission_id，不继续旧任务。

## 证据入口

相对工作区根目录：

- `debug-output/validation/20261012-domain-suite/runs.jsonl`：命令、模式、mission_id、步骤结果、前后 pose、等待/取消、耗时；完整证据不重复复制。
- `debug-output/validation/20261012-domain-suite/evidence-index.json`：51 次现场执行的 report 引用；50 次有 runner 摘要，另一次 `91ae3c9e9b9e` 在停止测试驱动前已完成，依据持久化 report 复核，不编造 runner 时序。
- `debug-output/validation/20261012-domain-suite/source-manifest.json`：最终源码 SHA256 和 git HEAD `d7a9a6337ec48322878f3699819c8e2e6c047ee0`；worktree 有既有未提交改动，早期 mission 在修复前运行，按本表 ID 区分版本，不能仅用 HEAD 声称可完全复现。
- `debug-output/sessions/<session>/missions/<mission_id>/human-report-<mission_id>.json`：完整任务、子任务、图像路径和运行历史。
- `debug-output/validation/20261012-domain-suite/navigation-preflight.json`：只读路线检查。
- `debug-output/validation/20261012-domain-suite/ocr-replay.json`：已保存 9 张图的 OCR 重放。
- `debug-output/validation/20261012-domain-suite/odom-probe.json`：独立只读里程计采样，不能替代故障发生瞬间的进程内证据。
- `debug-output/validation/20261012-domain-suite/health-observation.jsonl`：约 180s、每 5s 一次定位抽样。首个样本尚未收到状态，其余 35 个样本为 OK；不能把空样本计作定位丢失。
- `debug-output/validation/20261012-domain-suite/pytest*.txt`：回归结果。
- `tools/sim/run_mission_case.py`：可复用现场单例驱动，支持指定 Skill 开始后延迟取消；返回结果可用于批次 fail-fast。必须先检查场景，不能盲目连续运行恢复位姿命令。

## 已确认问题与修正

| 项目 | 失败证据 | 原因 / 修改 | 验证边界 |
| --- | --- | --- | --- |
| 未知 Arm pose 被报告成功 | M28 `ce6ca62be68c` | gateway BLOCKED 后保留旧 candidate；ArmSkills 先看 candidate.valid，误用旧候选。gateway 拒绝时失效候选；Skill 优先处理失败回复且仅接受 PLANNED；Agent/Skill 都检查网关实时 preset 清单和 allowed_modes | `451935363b26` 在规划阶段等待，不调用 set_pose；home/front 仍正常。gateway 新代码单测通过，常驻 gateway 未重启，现场由新 Agent/Skill 防线保护；完整重启后加载 gateway 修复 |
| `go to` 被当成目的地 | M16b `14406dddaa96` | incomplete target 在共享 command guard 中拒绝，避免掉进 embedding/LLM/地点匹配 | `b044a0c943fe` 未产生 navigation task 或运动 |
| M12/M13 阻塞时看不到第二个任务 | `90d37f334de1`、`179447cb5d09` 仅记录搜索任务 | 条件/依赖检查在创建 task record 之前提前返回；现在保留 WAITING_HUMAN、CONDITION_UNKNOWN / DEPENDENCY_UNCONFIRMED 及原因 | `c5f557b39dca` 等待时含两个 task，后退不执行；`f757c9ed9c9e` 同 Mission 批准继续，无重复搜索 |
| OCR 漏掉拍到的 06 | `491f696cad8c` 的 observation_007.jpg 清楚拍到 06，却返回 NOT_FOUND | 2x INTER_CUBIC 对小数字笔画不利；同图同后端线性插值识别 06=0.85。仅将 2x 插值改为 INTER_LINEAR，不降低阈值、不放宽 token 匹配 | 旧图 9 张回放命中第 7 张；新视角仍会漏检，不能宣称感知已稳定 |
| 转向失败原因不可细分 | M05b `1d8caf6c0f64` 转约 98° 后 wheel_odom_stale | 旧结果不记录究竟 wall 接收过期、source 过期还是 stamp 超前。新增故障瞬间 receive/source age、stamp、阈值与 rejection reason | 未放宽 0.75s 接收/源时间上限和 0.05s future 容差；根因尚待同进程复现 |

相关实现：

- `src/robot_agent/robot_agent/agents/arm_agent.py`
- `src/robot_skills/robot_skills/executors/arm.py`
- `src/so101_arm_control/so101_arm_control/arm_gateway.py`
- `src/robot_agent/robot_agent/parsers/deterministic_parser.py`
- `src/robot_agent/robot_agent/agents/mission_agent.py`
- `src/robot_skills/robot_skills/executors/perception.py`
- `src/robot_skills/robot_skills/executors/navigation.py`

## 现场覆盖

下表的“符合”指符合用例契约，包括合理拒绝/取消；不等于所有用户目标成功。

| 用例 | 实际结果 | 判定 |
| --- | --- | --- |
| M01 / M01b | 状态和 12 个注册目标正常返回，不运动 | 符合 |
| M02 / M19-observe | AUTO、MANUAL 静止状态都可左看、拍照、归位，底盘不转 | 符合 |
| M03 / M27 | home 和 see_front 完成，MOCK gateway 有对应执行反馈 | 仿真调度符合，不是实机验收 |
| M04 / M06 | 20cm 前后运动完成，wheel odom 约 0.20m | 首轮符合；重复见 runs.jsonl |
| M05 | 左转、右转完成 | 符合当前 0.05rad 容差，不是精确 90° |
| M05b | 首次转约 98° 因 wheel_odom_stale 停止；第二次从停止点发新 180°，`91ae3c9e9b9e` 完成；增加诊断后 `bb95cd7da2e5` 也完成 | 1 失败、2 成功，不同起始朝向，不是恢复原任务。第二次约 177.94°、32.5s wall；第三次未再次触发 stale，根因仍未知 |
| M07 | `f323f6f84aff` 导航后委派 Perception，06/6 共用 9 图，未识别标签，整体 PARTIAL | 委派符合；目标未完成。到达 task_tolerance：XY 0.162m、yaw 0.424rad。strict=0、task_tolerance=1、degraded=0 |
| M08 / M09 / M09b | 后退接观察、后退接 home、cm/m 等价表达均完成；分别执行前进复位 | 符合。三次前进实测约 0.207/0.206/0.204m；起点有小漂移，不是精确同位姿重置 |
| M10 | typo 命令进入澄清，Classic 本地 LLM 不可达，无运动 | 未漏步骤后假报成功；拼写理解能力尚未用 live JEV 比较 |
| M11 | `7bbb8f372f64` 搜索 PARTIAL 后，按显式 regardless 继续 Arm home | 符合，整体仍 PARTIAL |
| M12b | `ceafd4b08271` bottle 完整 NOT_FOUND，后退 SKIPPED/CONDITION_FALSE | 阴性分支符合；真实 FOUND 分支未验证 |
| M12 | red box 只有部分可检测能力，PARTIAL 后阻止条件运动 | 符合“不把未知当 false”；取消正常 |
| M13 | 取消分支不运动；`f757c9ed9c9e` 批准 continue_unmet_goal 后，同 Mission 仅执行后续后退，不重新搜索 | 两种用户选择符合；整体保留 PARTIAL |
| M14 / M22 | 后退约 0.6s 后取消，只移动约 3.3cm，无后续左转；新状态任务成功 | 普通取消符合。新命令在终态后数秒发出，不算毫秒级竞态实测；本轮未覆盖 recovery/API 在途取消 |
| M15 | `af3b9bd7c99d` 后退 → 左转 → Arm home 全部完成 | 符合，3 Task / 6 个实际 Skill，内部 task_end 不应算动作 |
| M16 / M16b / M17 / M17b | 四任务、缺参数、非机器人命令、未知货架均未运动，询问后取消 | 安全边界符合；Classic 非机器人命令的 llm_unreachable 提示不够语义化 |
| M19-auto-resume | `a9fd2e8db1e2` MANUAL 时后退被拦，明确选择 set_auto 后在同一 Mission 复查并完成 | 符合，不重发整条命令、不跳过 AUTO 检查 |
| M23 | 第一次 OCR 未检出；旧图回放修正后命中；换位重新采集仍未检出 | 尚未达到稳定识别 |
| M24 | red bottle 一个复合检测，共用 9 图，NOT_FOUND | 不虚报发现；未验证实物真阳性 |
| M25 | cup OR bottle 两个 check，共用 9 图，NOT_FOUND | 采集复用符合；真阳性 OR 另需场景 |
| M26 | image_describe API 未配置，明确等待，不输出“安全” | 符合能力边界 |
| M28 | 修复后未知 pose 无技能执行，询问后取消 | 符合 |
| M29 / M30 | touch/grasp 无完整自主执行契约，明确 UNSUPPORTED | 符合能力边界，不算触碰/抓取成功 |
| M31 | 采集约 2s 后取消，search_at_place 返回 CANCELLED；新 M01 正常 | 符合，旧任务未复活 |
| M32 | Primary 缺失/错误回答、Shadow 不采纳等离线回归 | 离线覆盖，不等于真实 API 测试 |
| M18 | 当前保持 Full，没有切换 arm_test | 未现场验证无全局定位环境，相关依赖边界仅回归覆盖 |
| M20 | 未主动断开相机 publisher | 未做本轮真实断流注入；旧缓存保护仅回归覆盖 |
| M21 | 本轮未发送真实 JEV 任务请求 | 不可达/缺回复/取消隔离仅离线回归，live 在途取消未验证 |

## 统计与交付状态

- 50 条 runner 记录，加 1 条依据持久化 report 核对的执行。包含修复前后、复位命令、预期拒绝和取消，不能直接混算整体成功率。
- runner 原始终态为 DONE/SATISFIED 27、DONE/PARTIAL 7、CANCELLED 15、FAILED_SKILL 1；其中修复前 M28 是已证实的假成功，绝不能计入正确完成率。
- 180 度动作共 3 次，2 次达当前角度容差、1 次 wheel_odom_stale 停止；不是同起点统计对照，不证明整体稳定。
- 导航 1 次：严格成功 0、task_tolerance 1、degraded 0；视觉确认未成功，完整导航目标成功 0/1。
- 2 次明确的操作员选择已验证：set_auto、continue_unmet_goal。其余 NEEDS_HUMAN 多由测试驱动按计划取消，不代表用户实际接受了替代目标。
- 本轮没有主动执行恢复注入，不统计“恢复成功率/恢复耗时”；后台温度告警自然消退不记作 Recovery 动作功劳。
- 回归：排除既有 real-map/camera 测试文件后 **422 passed**；完整测试 **433 passed / 8 failed**，失败均在该历史测试文件中，详见 pytest.txt。没有改动真实机器人校准以迎合测试。
- 最终任务 `bb95cd7da2e5` 已 DONE；AUTO、localization OK，最后可信 pose 约 map `(2.428, 5.604, -0.584rad)`。这不是最初位置，继续试验前重新核对空间。
- Dashboard `http://127.0.0.1:8765/` 保持运行，**当前测试进程是 Classic，JEV key 临时为空**；永久启动配置的 Primary/key 没改。真实 JEV 对照需要新授权并恢复正常 Agent 启动环境，无需为此重启 Isaac。
- 新 Agent/Skill/诊断代码已在最后一次进程重启中加载；Arm gateway 自身仍为旧进程，完整重启 Arm gateway 后才加载其额外候选失效防线。Agent/Skill 防线已实测有效。

## 待完成与待讨论

1. 新的 live JEV 对照授权后再做 Primary/Shadow。不能用本轮 Classic 成功声称 JEV 有收益。
2. EasyOCR 模型 `models/perception/easyocr/craft_mlt_25k.pth` 缺失且禁止自动下载。当前 Tesseract 可完成推理但跨视角召回不足。需决定模型部署、原始 1280x720 重新采集以及 OCR 角度/区域策略；保持“拍到但漏检”和“没拍到”分开。
3. M07 observation_007.jpg 在导航后明确拍到 06。当前问题不是 task delegation，也不能只归因于未拍到。到达 yaw 容差是否应受标签视角要求约束，后续另议，不在本轮偷改到达验收。
4. wheel_odom_stale 根因未证明。独立 20s 探针：740 条、最大到达间隔 0.051s、source age -0.0167..0s；它未复现 Agent 内瞬时故障，不能用来否定保护触发或证明传感器一直健康。
5. 后台 host 温度和 RGB/depth 同步告警有反复开关；Recovery 的 health evidence 自然恢复不能算 Agent 修复成功。本轮不修改 PB/告警门槛。
6. Arm-only、相机断流、JEV 在途取消、组合故障、真阳性检测场景，以及所有模式同起点三次重复尚未全面覆盖。
7. 部分既有 real-map/camera 回归依赖缺失文件或旧校准值。保留失败，不为本轮 sim 用例篡改真实配置。
8. FAILED_SKILL 根级 outcome 仍可能为 null（本次转向失败），调查时应以 terminal status 和 SkillResult 为准；后续统一 failure outcome schema，不能当作未执行或成功。
