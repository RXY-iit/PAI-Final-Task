# 2026-10-10 Sim 定位主动排查

## 1. 范围与基线

- 用户允许在 sim 内进行 3 m 范围的相对运动和任务调试。本轮仅执行前后各 0.1 m、左右各 20 度，未导航到货架、未操作机械臂。
- 环境经 Dashboard 和 ROS 参数确认：`env=sim`、`profile=nav2`、`use_sim_time=true`、AUTO。地图为 `sim_factory_mid360_like_gpu_trueimu_011/map_3d.ply`。
- 测试时间约 16:42-16:49 JST；Git HEAD `d7a9a6337ec48322878f3699819c8e2e6c047ee0`，工作区有大量未提交修改。测试期间没有修改生产代码、参数、阈值，也没有重启节点。
- 实际运行参数为 `use_fast_lio_hint=true`，不是配置文件默认的 false；高度过滤为 `[-0.8, -0.3] m`，初值覆盖要求 100%，超时 75 s。
- 基线不是健康导航环境：GICP 已处于 `failed/fresh_scan_coverage_timeout`，旧覆盖率 29/36，旧 pose/score 停止更新；Recovery 已达到人工处理状态。任务模式保持 Classic。
- 证据目录：[localization_debug_20261010](../../../debug-output/localization_debug_20261010/)。[summary.json](../../../debug-output/localization_debug_20261010/summary.json) 是派生指标及源文件 SHA256，不是第二份 Human Report。
- 四个 `*.manifest.json` 复用 `run-template.json`，引用原 session 的 mission report 与 skill_events。每个命令只有一次试验，不代表统计稳定性，也不是 JEV/Classic 对照。

## 2. 相对运动结果

| command | mission_id | 结果 | Skill 反馈测量 | 耗时（现实秒） |
| --- | --- | --- | --- | --- |
| `go back 0.1 m` | `aa70e29458f3` | DONE | 0.10059 m | 3.84 |
| `turn left 20 degrees` | `d60305af3853` | DONE | 转动 17.97 度，误差 2.03 度 | 5.95 |
| `turn right 20 degrees` | `c0169cf99f29` | DONE | 转动 17.73 度，误差 2.27 度 | 5.70 |
| `go forward 0.1 m` | `859e08e1bbb6` | DONE | 0.10113 m | 3.63 |

测量来自 skill_finished 的 wheel_odom metrics，不使用 Dashboard 低频健康快照作为终点。旋转在当前 5 度容差内。目标平移总计 0.2 m，反馈合计约 0.202 m，另有少量仿真静止滑移。

本轮验证：GICP 输出中断时，相对运动仍能独立完成；没有观察到恢复初值抢占上述四个任务。
没有验证：新出现的定位故障在任务结束后的自动调度、导航步骤提前介入、Cancel 竞态；基线恢复已处于 chronic/manual，不能用本轮证明这些场景通过。

## 3. 定位恢复对照

### 3.1 新点云正常，但旧失败状态不会退出

`raw_stationary.jsonl` 的 20 s 只读采样显示，原始 `/point_cloud` 与 GICP 输入 `/livox/lidar_pc2` 都在更新；当前最近 30 帧在高度过滤后也可覆盖 36/36 扇区。
但 `/gicp_loc/recovery_status` 仍保留旧的 29/36、30 帧、17860 点，整个窗口没有 GICP pose/score。

代码原因：`gicp_localizer_node.py::_scan_cb` 在 `window.failed` 时直接返回，而 `_latest_scan` 已被 initialpose 清空。新数据到来不能解除失败，只能等待另一个 initialpose。
这确认了“输出中断是恢复状态机锁住”的问题；不能据此证明此前为何只达到 29/36，也不能宣称当前所有方向始终缺数据。

### 3.2 不改参数，单次重设当前估计，11.286 s 后恢复输出

16:46:19.019 JST 发布一次 `/initialpose`，位姿取当时 `map -> base_footprint`：x=0.728219、y=-0.066616、yaw=3.084067 rad。
这是当前估计，不是 Isaac 绝对真值，不是地图全局搜索；没有物理移动底盘。

- 保留原 100% 覆盖门槛、10 s 最小现实积累时间。
- 累积到 21 帧、23230 点，覆盖 36/36。
- 16:46:30.305 JST 收到第一个 accepted GICP pose，相隔 11.286 s。
- 随后的 110 s 采样窗口收到 73 个 score、71 个 pose；`phase=matching`，不再是旧积累失败。

归因：一次诊断用初值发布解除了输出阻塞。**不是自动 Recovery 成功，也不是定位健康已经恢复。**

### 3.3 输出恢复后，静止定位不稳定确实存在

在上述 110 s 窗口：

| 指标 | 实测 |
| --- | --- |
| wheel_odom 相对窗口首帧的最大平移 | 0.00837 m |
| accepted GICP pose 相对其首帧的最大 XY 变化 | 0.75186 m |
| GICP score 最小 / 中位 / 最大 | 0.10877 / 0.11537 / 0.12591 |
| FAST-LIO 输出源时间落后 `/clock` | 12.58-18.53 仿真秒，中位 16.68 s |

两组位移的首帧时间不同，不能直接相减作为定位误差；整个轮式里程计窗口几乎静止，足以说明 GICP 变化不能用同期底盘运动解释。没有 map 对齐的绝对真值，因此不能确定哪个 GICP 位姿才正确，仍需区分初值收敛、退化匹配与错误 hint。

本轮补充了上一轮仅靠历史摘要无法确认的证据：不能把 `POSSIBLE_STATIONARY_DRIFT` 全部当误报，更不能用低 RMSE 单独判定恢复成功。

## 4. 设计与实现问题

| ID | 已确认事实 | 修正方向 / 验收条件 |
| --- | --- | --- |
| P011 | 积累超时永久拒收新点云；本轮数据已达到 36/36 仍停留旧失败 | 失败后允许受控的新窗口/重试；区分缺数据、运动、覆盖不足。不能在失败时默认导航可用 |
| P012 | FAST-LIO hint 实际开启，源时间明显滞后；`_fast_lio_cb` 不保存时间戳，匹配时直接使用最近收到的 hint | 对齐 scan、TF、hint 的时间；过期 hint 不参与当前扫描初猜；检查 FAST-LIO 内部积压和输入同步，不能简单改新时间戳 |
| P013 | 健康输入是 `/gicp_loc/pose` 原始 accepted 匹配；RViz 用的是平面化并按 alpha 平滑的 TF | 明确 raw_match、accepted/published estimate、base/lidar frame 的不同角色；健康同时检查可解释的一致性，不把不同表示视为同一证据 |
| P014 | 新初值之后开始收敛时也立即建立静止基准；之后长期拿当前位置与首个基准比较 | 区分初始化/收敛/稳定跟踪/退化，用证据建立稳定基准；不得因低 score 直接放行，也不能无限延长“收敛”掩盖持续漂移 |

P013 的具体证据：末段静止样本中，TF `map -> livox_frame` 的 z 约 1.4925 m，`/gicp_loc/pose` 的 z 约 0.0977 m。这不是同一个物理时刻的精确残差，但差异远大于毫秒级采样差可解释的量。代码将六自由度匹配转换成平面 map->odom，再发布未作同样处理的原始匹配 pose；需要核对地图、外参、扫描几何和对 roll/pitch/z 的约束，不能把此现象归因于 Dashboard 显示延迟。

另有代码审查发现，尚未作独立运动复现：`SystemSkills._trusted_pose_delta()` 计算的是距可信锚点以来的 wheel_odom 运动量，并非“当前 GICP 与 odom 传播结果之间的残差”。字段/告警名 `GICP_TRUSTED_POSE_DIVERGENCE` 容易被误解为已经测得两者不一致，需单独明确“运动后旧定位变旧”与“定位冲突”。

### 未证实的根因

- 迟到 hint 很可能影响匹配初猜，但本轮没有关闭 hint 的受控 A/B，不能认定它独自造成 0.75 m 变化。
- 高度过滤、重复环境、点云几何退化、地图/外参不一致都需要区分；没有凭告警直接修改外参或地图。
- RTF 较低会影响现实时间频率，但不是锁死的充分解释。当前 LiDAR/IMU 新数据到达不能证明 FAST-LIO 没有内部积压。

## 5. 下一轮顺序

1. 先修 P011，并做覆盖不足后数据改善、移动后重新静止、超时、取消的定向回归。
2. 固定静止场景与初值，对相同采样比较有效新鲜 hint / 不使用 hint；保留原始时间戳，记录队列/处理延迟。
3. 同时记录 raw GICP、平面 TF、实际 base 运动与 scan stamp，验证它们的 frame、时间和物理约束。绝对精度需要 map 对齐真值。
4. 再完善收敛/漂移/恢复状态判断和提示；按静止证据稳定恢复后，才进行需要地图定位的导航回归。

## 6. 结束状态与存储

- 四个运动任务均 DONE，所有本轮诊断进程已结束，没有挂起任务或持续发布的测试控制器。
- 机器人回到测试起点附近，未保证精确回原点；最后一次操作为重新设定当前估计的初值。
- GICP 输出已恢复，但最终定位健康仍为 `POSSIBLE_STATIONARY_DRIFT`，Recovery 仍要求人工处理；没有声明恢复成功，也没有启动导航。
- 生产代码和运行参数未改。本目录只追加调查记录、问题入口；调试工具在 `/tmp`，原始证据在 debug-output，约 7.3 MB，无 rosbag、图像或完整点云转储。
- 本轮采样保留逐帧小型数值证据用于定位问题，不作为后台常驻日志策略。
