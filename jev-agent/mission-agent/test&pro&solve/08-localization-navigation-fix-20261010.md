# 2026-10-10 定位与导航修复验收

## 1. 结论与边界

本轮延续 [07 的故障复现](07-sim-localization-debug-20261010.md)，实际修改代码、构建并在运行中的 Sim 验证。没有降低 GICP RMSE 门槛、静止漂移门槛、覆盖率要求，也没有修改地图或提高仿真速度来隐藏故障。

定位从静止漂移告警恢复到持续健康；初值重设后的点云积累可正常结束；附近 home 导航通过。尚不能声称任意货架、任意初值、自动 Recovery 全流程或整套 Full 冷启动已验收。

原始证据：`debug-output/localization_fix_20261010/`，派生指标 `summary.json`。每次命令的 `*.manifest.json` 引用 session `20261010T072312Z-b9b895b1` 下原 mission report 和 skill_events，不重复生成另一份 LLM 调查报告。各次测试初始朝向/版本不同，不能当作相同条件的统计对照。

## 2. 0.112 与 RTF 的关系

当前 Open3D 后端 score 是 inlier RMSE，本项目点云单位为米。0.112 表示参与匹配的内点残差约 0.112 m，既不是定位误差的真值，也不是 11.2% 置信度。RTF 不参与这个几何公式，不能按 RTF 乘除修正 score。

RTF 下降会间接改变现实时间消息频率、处理积压和动作耗时。是否有影响，需要分别检查扫描源时间、TF 时间、hint 时间、匹配覆盖/重叠、accepted pose，而不是只看最后一个低 score。

依据：[Open3D ICP 指标](https://www.open3d.org/docs/release/tutorial/pipelines/icp_registration.html)。small_gicp 的优化目标 `error` 不等于 Open3D inlier RMSE；本轮也修正了备用后端的指标计算与调用方向，但本机实际验收后端只有 Open3D。

## 3. 已修改的确定性问题

| 问题 | 修改 | 主要文件 |
| --- | --- | --- |
| 初值积累超时永久拒收后续数据 | 保留超时诊断，但继续观察有界的新数据；旧覆盖过期；运动后重新等待静止窗口 | `localization_pkg/scripts/initialpose_scan_window.py`、`gicp_localizer_node.py` |
| 高频时保留 30 帧不足以跨满 10 s | 连续观察时间与有界点云缓存分开计数；长时间断流后重新计时 | `initialpose_scan_window.py` |
| FAST-LIO 新收到的消息实际已过期 | 同时检查源时间与接收年龄，过期 hint 不参与扫描初猜；TF 按扫描时间获取 | `localization_evidence.py`、`gicp_localizer_node.py` |
| GICP 重复使用平面化先验，丢失 3D 地图高度/倾斜 | 内部保留完整 3D registration prior；导航仍使用平面 map->odom | `gicp_localizer_node.py` |
| GICP 误用窄高度切片 | Sim 启动独立使用 `GICP_SCAN_MIN_HEIGHT/MAX_HEIGHT`，默认不切片；原 laser slice 参数不再决定 GICP 几何 | 两个 `tools/sim/open_sim_*loc/nav2*` 启动脚本 |
| Health 和 RViz 使用不同位姿表示 | `/gicp_loc/pose` 与 accepted 平面导航 TF 一致；3D 匹配输出单列 `/gicp_loc/raw_pose` | `gicp_localizer_node.py` |
| 零/低 RMSE 但没有有效对应点也可能通过 | 增加有限值、最低重叠率和对应点数量验证 | `localization_evidence.py` |
| 正常移动超过锚点距离被误报定位冲突 | 新鲜且正常的定位不被 wheel travel 覆盖成 divergence；明确该字段是锚点以来里程，不是匹配残差 | `robot_skills/executors/system.py` |
| 导航在正常转向时误认为停滞 | 在目标区域内，将目标朝向误差持续减小计入进度；先更新进度，再判断降级。噪声、往复摆动不持续刷新进度 | `robot_skills/executors/navigation.py` |
| 监控以接收新鲜掩盖源时间过旧 | gicp_pose/fast_lio_odom 增加源时间检查；signal 记录实际 topic，避免对 `A or B` 字符串做 ROS graph 查询 | `robot_agent/nodes/health_monitor.py` |

GICP 线程默认限制 OMP=2 / BLAS=1，可用 `GICP_OMP_THREADS` 配置，避免与 Isaac 争用过多 CPU。`with_fast_lio=true` 不再覆盖明确的 `use_fast_lio_hint=false`；Sim 启动脚本仍明确开启 hint，但每次都校验其时间有效性。

恢复诊断 `gicp_recovery_status` 归入同一 GICP producer 及其依赖，减少独立的 unmapped 告警；PB-LOC-LOST 文案明确“定位尚未验证”，不把过期证据直接描述为机器人物理位置已经丢失。

## 4. 本轮证据

### 静止与初值重设

- 重载定位及 agent/health/recovery 后观察 60 s：30 个健康快照全部 `OK`；静止漂移中位 0.0191 m、最大 0.0605 m；不是绝对定位精度。
- 同窗口 RMSE 0.1156-0.1180，fitness=1；匹配耗时中位 0.279 s，最大 0.496 s。
- FAST-LIO 源时间仍落后约 13.8-17.4 仿真秒，但没有一次被用作有效 hint。这是隔离陈旧输入，不是修好了 FAST-LIO 的积压。
- 单次当前估计 `/initialpose` 后，11.516 s 收到新 accepted pose，11.940 s 健康回到 `OK`；覆盖达到 36/36，状态从 accumulating 转到 matching。45 s 窗口里除 5 次正常 settling 外，其余 17 次健康快照均 OK。
- 初值来自当前 TF，不是 Isaac 真值，也不是自动恢复动作。这只证明积累/重新匹配链路通过，不能算 PB-LOC-LOST 自主恢复成功。
- 同一捕获点云的离线重放中，窄高度切片重复匹配可产生约 0.8 m 变化；完整几何并保留 3D prior 后变化为厘米量级。没有做所有因素独立的在线 A/B，不能将全部改善单独归因于某一项。

### 命令执行

| mission_id | 命令 | 结果与验收解释 |
| --- | --- | --- |
| dba917fa00bc | go back 0.3 m | DONE；wheel_odom 实测 0.29856 m；3.94 s；定位仅 OK/MOVING |
| df23521031a1 | go to home | 修改导航进度前：任务 DONE，但由 relaxed tolerance 提前结束，朝向仍差约 104 度；不能记为正常朝向到位 |
| 5c6727669bb5 | go to home | 修改后：10.59 s；XY 误差 0.02295 m，yaw 误差 0.4226 rad；5 次朝向进度重置；没有放宽容差 |
| afa2536dceb4 | turn around | FAILED_TIMEOUT；30.17 s 转约 166.4 度，剩约 13.6 度；定位始终 OK/MOVING。暴露独立的相对动作 wall-time 预算问题 |
| bcc5f3f5bfa9 | go to home | 大朝向误差复测：19.41 s；XY 误差 0.03458 m，yaw 误差 0.4385 rad；9 次朝向进度重置；没有放宽容差 |

正常 home 任务容差是现有 `semantic_near` 的 XY=0.25 m / yaw=0.45 rad，因此本轮没有宣称严格 Nav2 goal SUCCEEDED 或高精度朝向对齐。任务达到自身容差后主动取消 Nav2 goal 是现有设计，与用户取消或恢复失败不同。

测试期间 RTF 仍约 0.45。没有把 watchdog 改为无限等仿真时间，也没有更改控制发布周期。180 度相对动作超时仍需单独设计“仿真动作预算 + 现实时间上限 + 无进展超时”，不能简单放宽所有安全超时。

诊断采集器最初未将 FAILED_SKILL 识别为终态，在任务停止后到观察期限才发 cancel；已修正采集器。失败任务没有自动重试，相应记录保留，不计为成功。

## 5. 验证与后续

- 定位、health、可信位姿、recovery engine/executor、mission handoff、相对移动证据、朝向进度等定向回归：173 passed。
- `robot_skills`、`robot_agent`、`localization_pkg` 构建通过；启动脚本语法和相关 diff whitespace 检查通过。
- 运行中仅重载相关 ROS 节点，未重启 Isaac、Nav2、FAST-LIO 或 Dashboard；这是局部重启验收，不是整机 Full 冷启动试验。最后任务已经停止，后台诊断采样不会持续运行。
- 尚需验证：完整 Full 冷启动、多个附近/货架目标、多次自动恢复与 mission 续行、移动中定位故障的延后恢复、Stop/Play 时钟回退、长时间负载。
- 未解决：FAST-LIO 源时间积压；CPU 温度/资源告警反复；大角度相对动作固定 wall timeout。没有证据证明是 Dashboard/arm 的追加单独导致。
- P014 静止收敛基准策略本轮没有完整重写；修复输入/先验后本场景已稳定，但一般化的收敛分类仍需后续测试。

优先下一轮：Full 冷启动检查健康 -> M04/M05 小相对动作 -> 已知近目标导航 -> 单故障恢复与续行。不要在定位尚未验证时直接把长距离货架导航作为 parser/JEV 成功率样本。
