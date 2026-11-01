# 本轮实现与测试入口

日期：2026-10-09。代码变化不代替 Sim 实测结果。

同日后续修改以 [06-mission-recording-and-sim-health.md](06-mission-recording-and-sim-health.md) 为准：快捷按钮直接发送、任务结束封存报告、定位恢复按 Mission 请求触发、RTF 已监测。本文下面的开发验证数字是前一轮记录。

## 1. 快捷命令与报告复用

- Dashboard 右下快捷区为 Mission Tests：M01–M17，共 22 个主命令/变体，点击直接发送；不自动改变控制/Agent mode，不自动 Cancel。
- 命令源：`src/robot_agent/robot_agent/mission_test_commands.py`，经 `/api/config.mission_test_commands` 传到浏览器。
- `run-template.json` v2 是轻量评估索引，不是第二套 Human Report。已有报告、日志、图像只引用；预期、真值、判定独立保存。没有新增报告 API 调用。详见 [数据复用](03-evidence-and-comparison.md)。

## 2. 定位恢复实现

按本轮要求实现“丢失前可信位置恢复”，命名 `last_trusted_pose`。没有检测语义地标，不应称为 semantic_landmark 已实现；真正的视觉/语义地标定位仍待设计地图关联与观测验证。

1. 复用 SystemSkills 的可信 GICP 位姿及 wheel odom 锚点，经 `/agent/health_state` 传给 Recovery。只使用本次运行历史，不从旧日志盲目载入。
2. PB-LOC-LOST 检查停止指令、取消导航；使用相对里程计推算当前 seed。不使用 Isaac 真值，不全局盲搜，不发旋转或底盘移动命令。
3. 默认锚点最大年龄 120 秒、wheel odom 年龄 3 秒、报告年龄 4 秒；相对锚点位移不超过 1 m、旋转不超过 0.5 rad。缺失/越界直接人工。这是 seed 有效性限制，不是启动宽限时间。
4. GICP 输出为 LiDAR pose，而 `/initialpose` 需要 base pose：使用 `livox_frame -> base_footprint` 外参转换，TF 不可用就失败，不猜坐标系。
5. GICP 收到 seed 后清除旧扫描与 FAST-LIO 修正参考；Sim 至少积累 10 秒墙钟、10 份不同且晚于重置的点云，达到 36/36 个十度方位扇区覆盖。重复时间戳不计数，只延长时间不等于完整扫描。
6. 覆盖是有效回波的离散方位覆盖，不是厂家完整 scan 标志，也不证明匹配正确。窄视野/遮挡可能一直不足，此时转人工，不静默降低要求。现场需要核对 LiDAR frame 和实际点云覆盖。
7. 最多缓冲 30 份，每份约不超过 20,000 点；75 秒仍不足则拒绝本次积累。积累时底盘移动超过 3 cm / 0.05 rad、TF 缺失或 frame 错误也拒绝，避免未经补偿地混合移动点云。
8. Recovery 最多等 100 秒新定位验证，要求 GICP score、accepted pose、Mission 定位一致性都正常。Done 不代表通过。自动尝试一次后转 RViz 人工，retry_budget=0，避免自动循环重设初值。
9. 健康输入加入 `localization_consistency`，避免正常 score 掩盖静止漂移。重置后短暂 UNKNOWN 保留有期限的恢复；根因 item 临时变好不能绕过 verify。
10. Mission 保留失败 skill/metrics；恢复失败回复不再被“未检测到漂移”的通用提示覆盖。最终仍由 Mission 重查权限/安全/定位并决定续行。Recovery 无回复/超时不再调用 Mission 旧兼容自动 reset，直接人工，防止两套恢复并行重设初值。

配置：`src/robot_agent/config/recovery/envs/sim.yaml` 的 `actions.last_trusted_pose / localization_success_timeout_sec`、`PB-LOC-LOST.yaml`、`src/localization_pkg/config/gicp_localizer.yaml` 的 `initialpose_*`。Real 没有自动可信 seed backend，playbook 仍未验证，不能将 Sim 测试等同实机验证。

优先复测 R02/R04/R10：有 seed / 无 seed / 过期 / TF 缺失 / 低频足覆盖 / 长等仍缺覆盖 / 人工 Done 但不合格 / Cancel 后不恢复原任务。score 正常仍不证明绝对精度，真值仅作评估。

## 3. RTF 与 topic 低频

`RTF = 模拟时间增量 / 墙钟时间增量`。理想情况下 `wall Hz ≈ sim Hz × RTF`：模拟时钟下 10 Hz、RTF=0.2，墙钟约 2 Hz，不一定是丢帧。RTF 正常但相机慢，可能是渲染频率、publisher、DDS、处理积压或传感器配置。主机负载是常见原因，但不能仅凭 topic Hz 下结论。

PB-SIM-RTF-LOW 用于识别上游仿真时钟问题。同日后续已加入 RTF health item 和 advisory playbook；没有 reduce_sim_load backend，不自动调频。详见 06。

后续建议：

- 先采集 `/clock` 相对 monotonic time 的 RTF、各 topic 模拟时间频率/墙钟频率/age，再区分整体减速和局部停流。
- 物理相关控制统一 use_sim_time；已经按模拟时间计时的控制循环随 clock 自动减速，不能再统一乘 RTF 二次降频。
- 点云/图像任务用新鲜样本量、覆盖和有上限的墙钟预算；必要时降低运动速度或渲染负担，不复用旧数据冒充新反馈。
- 急停、取消、链路丢失、死锁 watchdog 保留墙钟/单调时间保护，不能随低 RTF 无限延长。
- 低频有效数据可 warning；停流、坏数据或观测不足以满足当前任务，仍需阻断相应任务。

官方依据：[Isaac Sim RTF](https://docs.isaacsim.omniverse.nvidia.com/6.0.0/ros2_tutorials/tutorial_ros2_rtf.html)、[发布频率与模拟时间](https://docs.isaacsim.omniverse.nvidia.com/latest/ros2_tutorials/tutorial_series/tutorial_ros2_publish_rate.html)、[ROS 2 Clock and Time](https://github.com/ros2/design/blob/gh-pages/articles/130_ros_time.md)。文档版本可能不同于本机，这里引用时间概念，不直接照搬最新版 Graph 操作。

## 4. 启用与边界

重新启动 GICP/localization 和 Agent Dashboard 栈，再刷新浏览器；仅刷新页面不会更新 Python 节点。先取得健康定位以建立可信锚点，再观察自然漂移；刚启动就定位丢失、没有可信历史时，直接人工是预期行为。

本轮未向机器人发运动、重定位或故障注入命令。fault injector 与真正的 semantic_landmark 留作后续，不把它们作为本轮已实现功能。

## 5. 开发验证（不是 Sim 端到端结果）

- 定向 pytest：133 项通过，覆盖 Recovery engine/executor、Health monitor、可信位姿边界、Mission 交接、命令目录、点云窗口。包括重置前旧证据不能通过、UNKNOWN 不能中断积累、覆盖不足超时人工、无重复 reseed。
- 前端 TypeScript/Vite 构建通过；隔离浏览器在 1440×1000、390×844 下检查全部 22 个选项，文本正确、选择不发动作、无页面 JS 异常。截图 `/tmp/mission-picker-1440.png`、`/tmp/mission-picker-390.png` 为临时开发证据。
- robot_agent、robot_skills、localization_pkg 已 colcon 构建安装；运行中的节点需重启才能使用新代码。
- 全量 pytest：175 通过、8 项 real-lab 测试失败。6 项涉及缺失的 `maps/l402_2d_map_clean_0509` 地图/标注，2 项涉及已有相机坐标映射或标定预期差异。本轮未修改这些文件/标定值，未伪造地图来使测试通过。结果保存于 `debug-output/agent-validation/20261009-code-regression.xml`。真实定位恢复效果、全周点云覆盖可达性和任务续行仍待你在 Sim 测试。
