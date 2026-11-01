# 定位、导航与 Agent 稳定性验证

日期：2026-10-10。环境：Isaac Sim、ROS_DOMAIN_ID=20、Fast DDS、Full/nav2、地图 sim_factory_mid360_like_gpu_trueimu_011。本轮不启用 thinking LLM，不更改 real playbook verified。

## 1. 结论与边界

完成确定性修复、有限仿真实验及后续 Agent 设计稿。相对运动与定位新鲜度基线有改善，但**没有完成整机稳定性或三模式收益的统计证明**。导航到达与视觉确认必须分开：三次近距离 shelf_06 导航都按原有任务容差到达，OCR 仅一次确认 06。

本轮属于诊断迭代，不是冻结版本 A/B：测试间修改过代码、重载过 Agent，没有真值重置到完全相同的初始状态。所有 evaluation manifest 标记 comparison_eligible=false。旧失败记录保留。末尾新增的导航转向碰撞门只做回归及只读现场检查，不计入之前运动的验证结论。

证据根目录：`debug-output/stability_20261010/`。`summary.json` 汇总 39 个有 stdout 结果的任务（包括复位动作和初始失败），不是成功率分母。`evaluations/` 复用 run-template，仅引用 canonical mission report、图片和日志，不生成第二套完整报告。

最终就绪检查：`final-readiness.json`。Dashboard HTTP 200；Mission CANCELLED、无后续任务复活；15 秒静止观察 wheel odom 位移最大约 1.05mm；定位 OK、score 0.11884、pose 源年龄 0.60 秒；RTF 0.4609。只读调用新转向/10cm 微调保护均允许当前空闲区域。FAST-LIO 进程运行，Health 中 /fast_lio/odometry 新鲜；诊断文件里的 laserMapping=0 是按旧名称查找计数，不代表进程退出。未启用 thinking API。

## 2. 本轮代码修复

| 问题 | 修改与限制 |
| --- | --- |
| 直线运动只看走过的距离，可能把反向/侧移当成功 | 起始 wheel odom 坐标系投影，检查终点误差、横向偏差、反向和过冲；没有新鲜 odom 不执行 |
| 旋转角度取绝对值/环绕导致方向和整圈歧义 | 有符号角增量累积，按剩余误差停止；单测覆盖正负 180°、360° |
| 低 RTF 下固定现实预算过短 | 延续 source/sim 执行预算；另保留墙钟硬上限、clock 停止/回退、odom 失联和无进展保护 |
| 新比例转向控制落入驱动死区 | 初次 20° 实测只转约 10.85°，无进展保护停止；修正 sim 最低角速度为 0.18 rad/s，后续三轮完成。该参数不是 real 验证结果 |
| 动态导航可因振荡持续续期 | 要求位置进入新网格/向目标累计改善，或终点朝向累计改善；总等待有硬上限 |
| 近目标直接速度兜底绕过部分 Nav2 保护 | goto_pose 全程校验定位；微调/预对齐逐周期检查新鲜 costmap、实际 footprint、同帧 TF、扫掠区域。未知格/越界/过期拒绝；保守矩形包围可能拒绝实际可通路径 |
| 解析漏步骤/条件变成无条件动作 | 修复明确逗号/and 分隔；未实现条件执行、四步以上、占位省略号统一 ask_human，不能默认移动 |
| M11 cleanup 后缀混入 home 参数 | 独立提取 cleanup policy；只有明确无条件 home、推理确实完成的未检出结果才可继续。底层故障不按未检出跳过；最终说明保留部分未完成 |
| OCR 后端错误被吞成“目标不存在” | 全后端失败为 FAILED_PERCEPTION；至少一个后端完成才标 inference_completed，缺失后端单列 warning |
| 恢复 IN_PROGRESS 等待期间不能及时取消 | 等待检查取消，request_id/mission_id 关联，终态后丢弃迟到回复；不得复活旧任务 |
| report 当前问题与历史混杂、重复健康快照 | 健康状态差量、相关正常证据、恢复回复变迁；已解决 incident 不进入当前阻断项 |

## 3. 超时与时间证据

| 类别 | 完成/进展证据 | 仍需保留的保护 |
| --- | --- | --- |
| GICP 初值积累 | 唯一、有效、时间戳推进的扫描；sim 时间跨度、点数、回波覆盖及静止 | 现实失联/最终等待上限；重复/空点不计入。基础修复详见 09 |
| 相对平移/旋转 | wheel odom 实际位移、带方向角增量和目标误差 | 无进展 5 秒源时间或 30 秒墙钟；源时间执行预算；墙钟硬上限；取消/时钟异常/odom 失联 |
| 默认 30 秒 sim 运动 | 不要求在 30 秒现实时间完成 | 当前 MotionClock 对应 90 秒现实硬上限；极低 RTF 仍可能有意停止，不无限等待 |
| 导航 | Nav2 结果、已有任务容差、路径/目标方向的实际进展 | 定位与碰撞证据失效立即停止；动态模式最终上限默认 360 秒墙钟 |
| Recovery 握手 | 同一 request 的 final:false → final:true | 单调墙钟默认 180 秒等待、可取消；超时不另起无互斥的定位重置 |
| 通信/健康 | 源 stamp 与接收 monotonic 分开 | Health/Recovery 是监督，不能替代 skill 的即时停止门 |

GICP score 是几何残差，不按 RTF 缩放；低 RTF 通过数据延迟、积累长度和时间配对间接影响定位。IMU 进入 FAST-LIO；wheel odom 用于连续相对运动与可信锚点传播，不假称两者均直接融合进 GICP。

## 4. 仿真结果

运动前检查当前位置和 local costmap，测试在 shelf_06 附近已检查区域进行；单次平移 0.2/0.3m，不向初始 arm 测试桌墙区域做长距离导航。

| 实验 | 实际结果 | 证据 |
| --- | --- | --- |
| 20 分钟连续只读观察 | 1200 秒、240 样本；169 OK、62 MOVING、5 SCORE_STALE、3 初值 settling、1 初始空样本。包含主动中断和重载，不是零故障 soak | soak.jsonl |
| 后退/前进 0.3m、左/右 20°、180°及转回，各三次 | 修复后 18/18 DONE；平移误差约 0.0118–0.0149m，最大横偏约 0.00367m；180°约 33.49–34.01 秒墙钟，终点 yaw 误差约 0.038–0.046rad | M04/M05/M05b/M06-classic-v2-* |
| 初版转向失败 | 1 次 FAILED_SKILL / motion_no_progress；未改报成功 | M05-left-classic-r1.* |
| FAST-LIO/IMU/odom 运动时序 120 秒 | FAST-LIO age 中位 0.100、最大 0.117 sim 秒；IMU 最大约 0.033；GICP pose 最大约 0.367；末次 LiDAR 队列 0、IMU 队列 6、backlog dropped=0 | motion-timing.jsonl |
| GICP 进程暂挂 16 秒，再由实验脚本恢复 | PB-LOC-LOST 发布可信初值并验证，约 34.0 秒，0 次操作员请求；无 spin | R02-loc-fault.jsonl，inc-20261010-190203-28 / exe-29 |
| 恢复后近距离导航 | goto_pose 成功，随后 OCR 未确认；当时已很接近目标，不能算长路线通过 | M07-post-recovery.jsonl |
| 三次后退后接近 shelf_06 | 导航实际路程约 0.043/0.311/0.265m，三次 task_tolerance；终点误差约 0.232/0.238/0.234m；视觉确认 1/3 | M07-approach-1/2/3.* |
| Arm home；后退再看左；后退再 home | 三条均 DONE，顺序步骤保留 | M03-final、M08-final、M09-final |
| 后退中 Cancel | CANCELLED，后续转向未执行 | M14-final，mission 50a5980cf601 |
| Classic/Primary/Shadow，各 M01+M02 | 6/6 DONE，Primary/Shadow API 可用；只读状态/相机动作，不能外推复杂导航收益 | M01/M02-各模式、*-availability.json |
| 真实 JEV 六条解析复测 | provider=live，约 178–195ms；typo 两步解析成功，M11 home 参数修正，条件/四步/省略号被确定性门拒绝 | live-parser-audit-v2.json、jev-parser-calls-v2.jsonl |

**分级导航统计，分母为三次近距离导航**：Nav2 严格成功 0/3，语义任务容差成功 3/3，降级成功 0/3，导航失败 0/3。三次都走原有容差策略，本轮没有放宽容差。端到端带视觉任务成功为 1/3，需人工处置的视觉未确认 2/3；两次均被测试脚本取消，没有人工补识别来计成功。

定位恢复归因必须分开：实验脚本 SIGCONT 让 GICP 重新运行，Recovery 自动发布初值并检查新的匹配证据。不能称 Recovery 自主重启了 GICP，也不证明绑架恢复或坏地图恢复。原 mission 自动续行的完整故障链尚未实测，此处是在恢复后发布了新导航任务。

## 5. 覆盖表

| 范围 | 本轮覆盖 | 仍未验证 |
| --- | --- | --- |
| M01–M09/M14 | 上表实际动作；关键相对运动三次；M09b 参数离线核对 | 每个语言变体三模式各三次尚未完成 |
| M10–M13/M15–M17 | 22 个按钮命令离线解析；六条真实 API；cleanup、条件拒绝与不完整输入回归 | red box 现场真假目标组合、完整条件任务执行、三任务现场执行 |
| R01/R02、PB-LOC-LOST | 持续健康观测；一次可撤销定位停流故障自动验证 | Full 冷启动；不同时刻锚点+人工运动的全链现场故障矩阵 |
| C01/C02/R04 | IN_PROGRESS/final、错误 request/mission、取消及时退出、迟到回复离线回归 | 等待恢复期间现场 Cancel + 解除故障的完整联动 |
| R03/R05/R06、C04/C07 | FakeBackend 验证 A2 安全门、审批、Done 不代替证据、互斥相关分支 | 未现场触发 ESTOP/模式切换/物理驱动报警，不能标硬件 verified |
| R07/R08、PB-SIM-RTF-LOW | 低频不自动恢复、忽视/升级/复发离线；在线观察 RTF advisory | 精确 topic gate、坏载荷和可重复负载注入工具缺失 |
| R09–R12、其他 PB | 节点 restart/人工退化、重试/冷却、上游根因、clock 回退/停止等离线分支 | 现场驱动节点重启、路径受阻、绑架、Stop/Play 重置及组合故障未执行 |

回归：`robot_agent/test + robot_skills/test + localization_pkg/test + robot_sim/test`，**281 passed / 8 failed**，见 regression-final.xml。8 项来自 test_real_lab_parser_semantic.py：6 项缺少 real 地图/路径资源，2 项旧相机期望与现有标定不一致。保留失败，不修改 real 配置或隐藏测试。

## 6. 后续设计与待解决项

| 分类 | 问题 / 下一步 |
| --- | --- |
| 实现缺口 | Perception JEV review 仍主要是 trace，没有实际重编模型/扫描计划；新架构仅设计稿 |
| 实现缺口 | 条件分支执行器尚未实现，当前明确 ask_human；不能把 guard 当成完成条件任务 |
| 工具/资源缺失 | EasyOCR craft 权重缺失；无统一故障注入器、独立 map 真值或完全同初态重放 |
| 证据不足 | 长距离导航、冷启动、组合故障、定位恢复后原 Mission 自动续行、强负载多小时 FAST-LIO；必须下一轮单独验收 |
| 待确认策略 | 导航容差足够到达区域但不足以保证 OCR 视角；是否追加有限视角搜索，而非放宽识别阈值 |
| 待确认策略 | sim 低速驱动阈值应否抽成配置；极低 RTF 的墙钟上限；保守 swept-AABB 是否改为精确多边形 |
| 调查输入 | 原 report 大 metrics 仍需按需投影；thinking 调用应异步、有预算，不直接使用同步 human-summary API |

详细设计：[Upper Agent 信息源](../../../upper-agent/LLM-call/README.md)、[Perception Agent](../../perception-agent/README.md)、[Navigation Agent](../../navigaiton-agent/README.md)。保留用户指定目录拼写 `navigaiton-agent`。

本轮没有录制新 rosbag 或持续写完整点云，也没有删除旧失败证据。短期调试日志仍有保留价值；最终清理由既有 retention 工具按 session/引用关系进行，不删除当前运行日志和仍被 report 引用的图片。

`final-source-fingerprints.json` 保存最终源代码/配置 SHA256，不冒充每次历史运行冻结版本。`tools/sim/observe_agent_stability.py` 可做有界只读采样；`tools/sim/summarize_stability_runs.py` 从本轮结果生成引用式评估清单。两者不负责自动故障注入或控制机器人。
