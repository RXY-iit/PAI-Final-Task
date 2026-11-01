# Primary、人工续行与告警检查（2026-10-12）

## 本轮范围

- Dashboard/Agent 启动默认 `jev_primary`。用户已确认 Mission、Navigation、Perception、Arm 四个任务 Agent 统一启用 JEV；Recovery Playbook、安全门仍独立。
- Primary 会使用各域实际需要的 JEV 检查，不意味着每个简单 Skill 都额外调用 API。Arm 增加 `arm_task_review`，不开放尚未实现的抓取等能力。Shadow 只记录比较结果，不采纳模型建议改变执行。
- API 可用状态与 mode 是不同信息；选中 Primary 不等于证明 API 正常，仍使用现有 probe / 调用结果的状态判断。
- 本轮没有调用真实 API、移动机器人或切换现场控制模式。验证使用离线客户端和浏览器 WebSocket mock。

## Cancel 的证据与修复

现场日志：`debug-output/sessions/20261012T023653Z-728196f2/agent/skill_events.jsonl`。

- `7898a09bcaf3` 等待 incomplete_command_placeholder 后，英文 Cancel 被当成自然语言新请求，产生 `dd7117c7a25d`，重新解析并执行取消 Skill。
- `6bb8d7cbaadd` 的通用取消产生 `cancel_requested` 和 `mission_cancelled`，后端确实到达 CANCELLED；没有找到与该操作对应的异常堆栈。因此不能把用户看到的异常直接归因为后端抛错。
- 前端存在另一项确定性风险：旧 Mission 的迟到取消状态会清掉新 Mission 的显示。

现已统一为控制路径：人工选择面板移除模型生成的重复 cancel，只保留一个通用取消；纯取消文本在 AgentNode 中也不再新建 Mission。旧 Mission 的迟到状态不能清除当前 Mission，取消后的旧任务也不能被迟到的 received/state 重新展示。

## 同一 Mission 内修复与续行

### 取消范围补充

- Ask Human 固定按钮和命令区共用 `MissionCancelButton`，显示 `キャンセル（全部） / Cancel all`，绑定同一个 `sendCancel`。含义是取消当前 Mission 的全部执行中/未执行 Step，不是仅取消当前 Step，也不是清空独立 Recovery 的故障。
- 动态选项由 JEV/执行策略提供，不能只凭按钮文字决定取消范围。自然语言 `cancel step` 不属于全取消的控制别名。
- 用户已确认未来单步控制的语义：停止当前 Step，保留 Mission，然后由 Agent 重新检查后续依赖和安全条件，决定继续或再次询问。不能默认跳过依赖检查，更不能把取消 AUTO 前置条件当成允许执行后续运动。
- 本次固定取消按钮已实现；独立的 `cancel_step` 执行协议尚未实现，不宣称改名即拥有单步取消能力。后续仅在后端提供绑定 Mission/Step/等待令牌的有效动作时显示该选项；不得把它转发为自由文本再误解析为全取消。
- 现有 `set_auto`、`resume` 属于带作用域的同 Mission 控制；旧语义澄清选项的 `command` 仍是自然语言新请求，两者不能混为同一步续行。

### 相机 Quick Start 检查

- `tools/sim_robot_quick_start.py` 默认 `low_camera_resolution=False`；本次读取保存设置也为 False，profile 为 nav2，`start_isaac=False`。
- `src/robot_sim/robot_sim/camera_profile.py` 标准值为 640x480；真实底盘 RGB 启动配置 `src/robot_bringup/launch/test_all.launch.py` 为 `640,480,30`。这里只核对分辨率，不承诺仿真与真实频率/标定完全相同。
- Full/Arm 的 Play 前后复制脚本均写入显式 profile，覆盖旧的 640x360 等环境变量；D455 RGB/Depth render product 共用该分辨率。
- 未勾选启动 Isaac 时，Quick Start 不会自动更改正在运行的 Isaac 相机；需要重新复制并执行对应 Script Editor 配置。低分辨率只有显式选择时使用 320x240。

实现入口：

- `src/robot_agent/robot_agent/agents/mission_agent.py`：`_wait_for_step_repair`、`human_action`。
- `src/robot_agent/robot_agent/nodes/agent_node.py`：`/agent/mission_control` 订阅、`/nav_mode/command` 发布。
- `src/robot_agent/web/src/components/HumanChoices.tsx`：带作用域的人工操作按钮。

控制消息示例（不是自然语言命令）：

```json
{"action":"set_auto","mission_id":"current-mission","wait_id":"current-wait"}
```

`action` 只能选择当前 wait 提供的 `set_auto` 或 `resume`；Mission ID、唯一 wait ID、不处于取消状态均需匹配。旧按钮不能影响后来开始的等待。

1. AUTO 前置条件不满足时，保留原 worker、计划索引、已完成结果和目标，显示「AUTO に変更して続行」。
2. 只有操作员点击才请求 AUTO；JEV 可建议 `ask_human_enable_auto`，不能自行切换模式。
3. 发布请求不等于成功，必须收到新鲜的真实 AUTO 反馈，再重新执行原步骤的前置条件检查。手动从其他控制入口切换 AUTO 也可满足该反馈条件。
4. 健康检查、定位门等支持「条件を再確認して続行」。点击仅重查当前步骤，不绕过健康/安全门，也不重新解析整条命令。
5. 等待期间取消会唤醒并终结 worker；随后收到 AUTO 不能复活任务。

边界：语义歧义仍需澄清；已经执行一部分的相对移动、接触/抓取等不盲目重放。当前通用续行不是所有错误的自动重试策略。离线 MissionAgent 默认不启用阻塞人工等待，实际 AgentNode 显式启用。

## Decision Model Routing

Intent Routing 更名为 Decision Model Routing。除 command_parse / pre_step_review 外，逐次追加真实 trace 中的域检查和异常监督，例如 nav_target_review、perception_model_review、arm_task_review；显示决策、耗时及 Task/Skill 关联。未调用的阶段不伪造卡片。

## Recovery 告警：本次采样证据

以下是本轮读取的快照，不代表持续实时数值。

| 项目 | 证据与含义 | 处理方式 |
| --- | --- | --- |
| PB-HOST-RESOURCE | CPU package/core 达约 80°C，sysfs max 为 80°C、crit 为 100°C；CPU 约 39.5%，内存约 40%，GPU 约 60°C。本次是温度门限，不是 CPU/内存一起耗尽 | 减少非必要负载、检查散热，等指标稳定恢复。Retry 仅重新打开处理流程，不会自动降温或杀进程 |
| chronic human only | 同类问题反复触发，进入重复发生次数限制 | 不是严重程度自动升级。确认原因后 Retry，或对可接受的 warning 暂时忽视；忽视不改变原始健康数据 |
| PB-SIM-RTF-LOW | RTF 约 0.466，窗口约 10 秒，clock 接收 age 约 0.004 秒、reset=0，时钟在推进 | 仿真约比现实慢 2.15 倍，不是暂停或 clock 失联；目前低于 0.8 的 advisory。降低仿真负载，或接受慢速并暂时忽视；不自动等幅修改所有控制周期 |
| PB-CAMERA-MODEL-MISMATCH | 底盘 RGB CameraInfo 为 640x360；Arm 为 640x480。旧 sim item 名 camera_temperatures 错误暗示温度问题 | 改为 sim_camera_model，并显示实际尺寸证据。需要标准分辨率时重新应用 Quick Start 对应相机 Script Editor 配置；仅重启 Dashboard 不改变 Isaac 相机 |

相机 640x480 通过只证明当前尺寸检查满足要求，不能证明内参、外参和标定准确；不能仅篡改 CameraInfo 数字消除告警。host 的通用提示也已改成「某项测量超过门限」，Dashboard 显示根健康项的具体摘要。本轮未放宽阈值、修改 real verified 状态或改变 Recovery 执行授权。

待继续观察：温度在阈值附近反复穿越，可能造成 operator 流程重复开启和 chronic 计数增长；历史 “resolved by health evidence” 可能只是自然降温，不是 Recovery 动作有效的证明。本轮没有重构该迟滞/计数状态机。

## 验证与部署边界

- 179 项针对性 Python 测试通过：域 Agent、人工续行、Recovery handoff/engine/executor、健康监测和 report。
- 前端单元测试、生产构建通过；1920/1440/1280/390 布局检查通过，1440/390 人工操作/取消竞态检查通过。
- 回归包含：请求 AUTO 但无反馈不执行、陈旧 AUTO 不执行、新鲜 AUTO 后只执行一次；取消后不复活；健康仍坏时重查不越过安全门；旧取消不清空新 Mission；Arm Shadow 不采纳 JEV 拒绝建议。
- 前端 dist 已构建。现场 Python 进程未重启，新后端逻辑需要重新启动 Agent Dashboard 启动项，不需要为这些逻辑修改重启 Isaac。
- 下一轮现场验收：MANUAL 下发 `go back 20cm`，点击 AUTO 续行；等待时取消后再手动 AUTO；健康修复后同一步重查；检查四个任务 Agent 的真实 API trace。上述实体仿真行为本轮尚未验证。
