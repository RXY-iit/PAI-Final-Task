# Mission 记录边界与 Sim 健康证据

更新：2026-10-09。以本轮实现为准，待 Full 实机仿真回归。

## 1. 调查结论与唯一执行事实源

旧 `human-report-cfcfa662d348.json` 按 mission_id 命名，但实际按 dashboard 当前显示的 mission 延长生命周期。
160 条事件中 159 条是 heartbeat，早期任务事件已被挤出；结束后的全局恢复也会改写旧报告。这不是预期的 mission 边界。
旧报告保留，不伪造或追补其中已经丢失的事件。

新的组织结构：

```text
debug-output/sessions/<UTC启动时间-随机ID>/
  agent/skill_events.jsonl       # 本次启动的原始任务事件，按 mission_id 关联
  agent/jev_calls.jsonl          # 脱敏 API 请求与回答
  sim_health.jsonl              # 每 10 秒的时钟/RTF/流摘要
  missions/<mission_id>/human-report-<mission_id>.json
  .pid-<PID>                    # 清理时保护仍存活的进程
```

- ROS dashboard launch 每次产生新 session ID，各进程共享。浏览器刷新不是新 session。
- 每次接受新命令产生新的 mission。完成、失败、取消后封存；`NEEDS_HUMAN`、恢复中和等待用户选择不封存。
- `cancel_requested` 只是请求，直到实际 `CANCELLED` 才封存。
- 等待人工时若 Mission worker 已退出，Cancel 由 AgentNode 补发 CANCELLED 状态并清空人工提示，避免报告一直不结束。
- 跨 topic 的迟到终态事件可以补入一次；后台 heartbeat/旧 mission 状态不能重新打开报告。
- 未启动任务时的人工恢复报告标记为 `scope=system_incident`，不伪装成任务记录。
- ROS 自身日志、Isaac 日志、已有图像目录仍保留自己的目录，不复制到 session；证据通过路径引用。

## 2. 应记录与不应记录

记录：用户命令、解析结果和来源、计划、每步实际参数与结果、错误码、人工选择、恢复尝试和验证结果、最终状态；起始健康异常快照、健康状态变化；原始 JEV 调用日志引用。

不记录到任务事件数组：每 2 秒重复的健康 heartbeat、仅时间戳变化的健康状态、以前任务已结束的恢复记录。健康监测和 UI 实时更新不因减日志而关闭。

正常结束的报告也生成，供 debug 和成功率统计。Thinking LLM 使用同一份报告的裁剪视图；不是重新运行一遍调查、再生成一份同功能事实记录。
API 默认仍由 `HUMAN_REPORT_API_ENABLED` 控制；相同求助原因/用户选择不因重复状态推送反复调用。
与任务无关的全局故障列为 background_incidents，不冒充任务的求助原因，也不把正在执行的任务报告改成 NEEDS_HUMAN。

`run-template.json` 是人工评价清单，不是第二份运行日志：

- 保留测试编号、重复轮次、预期行为、禁止动作、ground truth、通过/失败评价和问题编号。
- `session_id`、`mission_id`、`evidence_refs` 指向现有报告和日志。
- 不手工复制完整执行步骤、recovery 历史、JEV request/response 或 heartbeat。
- 每个命令可有一份评价；无需为普通非测试操作强制创建评价清单。

## 3. Mission 与定位恢复

- 定位始终监测和显示；不依赖定位的 Mission 执行期间，定位恢复是延后而非关闭，告警不打断当前任务。
- `move_relative`/`rotate_relative` 不因定位异常而停止，原有急停、控制权限等保护仍有效。
- `goto_pose` 依赖定位，由 Mission 的健康检查停止运动并请求 Recovery。
- Mission 完成、失败或取消后，若故障仍存在，自动尝试 PB-LOC-LOST，不需要再次点击 Retry；空闲时同样自动恢复。到达依赖定位的步骤时，通过当前 Mission 请求提前解除等待。
- 等待覆盖整个不依赖定位的任务，不在 move_relative 与后续 perception 之间抢先发布初值。Dashboard 的 execution_blocked 显示 Deferred 及解除等待条件。
- Cancel Mission 不等于 Abort Recovery：前者解除定位恢复等待，已开始的同一恢复可继续；后者保留人工中止后的禁止自动重试状态。恢复前仍要求新鲜停止证据，并遵守冷却、次数和安全限制。
- 初值使用丢失前的可信位姿加上等待期间累计的 wheel odom 运动增量；不直接复用丢失点的旧 pose。证据失效或超出可信种子范围时转人工，不盲目放宽有效性限制。
- 故障在等待期间自行恢复则不再发布初值。若新任务启动且不依赖定位，则再次延后恢复；旧 Mission 请求不能使新 Mission 意外解除等待。
- 自动种子不可用/验证失败后明确显示等待 RViz 位置与朝向，Done 不是恢复成功证据。已封存的 Mission 报告不因任务后后台恢复而重新打开。
- `/initialpose` 后的初值修正默认不限制平移/转角跳变（`initialpose_limit_correction=false`）。正常跟踪和自动连续拒绝触发的有界窗口保持原限制；评分与有效变换检查不取消。
- 大距离初值不保证 GICP 能找到正确位置：局部匹配仍需要合理的初始位置、朝向与有效点云。

## 4. Dashboard

- 原右下快捷指令改为 M01-M17 全部变体按钮，点击即发送，不自动批量执行。
- 新启动默认不加载跨 session 的忽略告警；显式 `ack_store` 参数仍可选择持久化。
- 启动 profile 以本次 launch 参数为准，不默认读取上次 GUI settings；显式指定 `launcher_settings` 才允许兼容回退。
- `当前未触发` 不再写成 `当前正常`。没有证据不等于健康。

## 5. Sim 健康监测

本轮已接入 Health Dashboard 的 H2 证据：

| 项目 | 依据与边界 |
| --- | --- |
| RTF | `/clock` 与 monotonic 的约 10 秒窗口斜率；低于 0.8 是 warning，不自动改控制周期 |
| 暂停/回退 | 区分无消息、消息仍来但时间不推进、时间回退；无消息不能断言一定按了 Pause |
| use_sim_time | 异步 GetParameters 查询配置的 GICP/Nav2 节点；超时/未启动标为未验证，不 spin |
| 传感器频率 | 现实时间接收 Hz、源时间接收 Hz、源数据年龄、重复/回退时间戳、最大接收间隔；payload 抽样不再丢掉接收计数 |
| 相机同步 | 最近 5 秒 RGB/Depth/CameraInfo 时间戳匹配、尺寸与 frame 比较；默认 0.1 秒容差，未收到时 unknown |
| 重复发布 | 传感器 publisher endpoint 检查；不能把多个来源误认成单个正常传感器 |
| 定位积累 | `/gicp_loc/recovery_status` 提供覆盖率、帧数、点数、时长、失败原因、匹配结果和地图/frame |
| 主机负载 | 复用已有 CPU/GPU/显存/温度监测，不增加重复采集器 |

尚无足够仪表的内容明确显示 `NOT_INSTRUMENTED`：USD 场景身份/单位与地图外参的自动一致性验证、发布端序号支持的精确丢帧数与队列深度。接收间隔和延迟只是线索，不能替代这些证据。
`PB-SIM-RTF-LOW` 现为 advisory，建议检查时钟和主机负载；不会自动重启 Isaac 或修改控制器周期。

## 6. 存储与清理

本轮检查：debug-output 约 12 MB，~/.ros/log 约 92 MB；仓库内未找到 rosbag 文件。没有实际删除历史记录。
空闲期间 ROS/Isaac 自身仍可能写日志；新监测每 10 秒写小型摘要，不保存图像/点云内容，不自动录 bag。
任务事件日志单文件约 10 MB 轮转，保留 3 个备份；Sim 摘要约 5 MB 轮转，保留 1 个备份。
GICP 正常跟踪的逐次成功修正降为 debug；初值窗口成功和错误仍保留。ROS/Isaac 自身的其他日志不受本工具轮转控制。

```bash
python3 tools/prune_debug_logs.py --days 7          # 只预览
python3 tools/prune_debug_logs.py --days 7 --apply  # 明确删除过期的新 session
```

只处理 `debug-output/sessions`，保护活跃 PID、KEEP 文件标记、包含 rosbag 的 session。不会清理地图、标定、训练数据、旧格式输出或 ~/.ros/log。
需要保留问题证据时，在 session 下放置 `KEEP`。清理整个 session 会同时删除其报告与原始证据，避免只剩失效引用。

## 7. 本轮验收重点

1. 启动两次检查 session 不同、忽略列表为空，profile 对应本次选择。
2. 执行 M01、M02，检查两份报告；结束后等待一分钟，旧报告不再随 heartbeat 改写。
3. 定位异常时相对运动仍可执行，Mission 运行期间不发布恢复初值；完成或 Cancel 后等待停止证据并自动恢复。任务中遇到导航步骤则提前停下并请求恢复。
4. 人工 Retry 后看积累进度，运动中失败原因应为 base_moved，不能再显示过时的 cancel_motion。
5. Pause/Play 看 RTF 与时钟状态；不把历史 RTF 推算成新测量值。
6. 设一个较远但合理的初值，确认不再因为初值跳变距离被拒绝，但不合格匹配仍被评分保护拒绝。

## 8. 开发验证

- 定位延后恢复修正后的定向回归：90 项通过，覆盖 Mission 完成/取消后自动恢复、导航步骤提前请求、相机请求不解除定位等待、故障自行恢复、停止证据、显式 Abort 与 odom 初值传播；robot_agent 已重新构建。以下 148 项为此前整轮验证记录。
- 定向 pytest 148 项通过：任务报告边界、恢复与 Mission 交接、人工等待 Cancel、监测证据、RTF、日志保留、点云窗口等。
- TypeScript/Vite 构建通过，robot_agent / robot_skills / localization_pkg 构建安装通过。
- 独立 ROS Domain 121 初始化只读 HealthMonitor，新增项目正常构建；未连接 Domain 20 的机器人控制。
- 隔离浏览器以 mock WebSocket 检查 1440x1000 和 390x844：22 个按钮各发送一次精确命令；模拟 RTF=0.5、覆盖=75% 正确显示，无 JS 错误和横向溢出。
- 浏览器数据是假数据，不是实际 RTF。尚未进行 Isaac Full 端到端恢复测试，也未调用 Thinking API 或删除旧日志。

### 2026-10-10 实测补充

上述“尚未实测”描述的是实现当时。随后已在 Full sim 完成四个小范围相对运动和一次定位初值对照，见 [07 实测报告](07-sim-localization-debug-20261010.md)。相对运动均完成；定位输出可经新初值恢复，但静止匹配仍不稳定，不能认定定位恢复通过。积累失败锁死、过期 FAST-LIO hint、原始匹配与平面 TF 表示不一致列为 P011-P014，当前未修改生产实现。

以上为第一轮诊断时的状态。后续已修改生产实现并重新构建，最新结果见 [08 定位与导航修复验收](08-localization-navigation-fix-20261010.md)：静止定位健康、初值积累后重新匹配和附近 home 导航已实测通过；FAST-LIO 积压及低 RTF 下大角度相对转向的时间预算仍未解决。不能把本轮局部节点重载等同于完整 Full 冷启动验收。
