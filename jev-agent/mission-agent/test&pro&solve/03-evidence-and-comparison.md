# 证据、对照与收益判定

返回 [README](README.md)。目标不是收集更多截图，而是让每个失败都能回答：原指令是什么、当时证据是什么、谁做了什么判断、什么动作真正执行、为何停止/续行。

## 1. 每次执行的最小证据包

### 数据只生成一次（2026-10-09 更新）

`run-template.json` v1 的 actual.skill_results、operator_actions、recovery_actions、task_graph 与 Human Report 的 mission / recovery_history / user_interactions 重复。v2 改为评估索引，不再另写一份运行报告，也不因此再调用 LLM。

- **运行事实**：复用 Human Report 的 plan、skill_results、人工选择、恢复历史和可用的 api_summary；精细时间、JEV request/questions/answers/usage 仍以 JSONL 为准。Human Report 不替代全部原始 API 数据。
- **测试独有信息**：run.json 保存用例/重复次数、执行前预期、实验配置引用、故障操作、真值、验收判定及问题 ID。预期不能从实际结果反向生成。
- **关联粒度**：同一未清除的 mission 维持一份 report；多个测试片段可引用同一报告，以 mission_id、request_ids 和日志行范围区分。manual report 的 mission_id 可以为空，不能硬匹配到下一任务。
- **内容版本**：记录 report_id、updated_at 和文件 SHA-256；报告会继续更新。封存时保存一份最终快照或整个会话归档，后续引用封存版本，不能覆盖旧 run 的证据。
- **计算复用**：统计用代码读取已有证据。未来按“源文件哈希 + 提取器版本”缓存 derived_metrics；这是缓存约定，本轮未新增自动统计服务。LLM 总结复用已有 api_summary，不拿摘要代替事实。
- **缺数据**：没有 Human Report 的正常任务直接引用 JSONL，不为填模板额外制造 ask-human 报告；未知用 null。

Dashboard M01–M17 是命令选项，不是实验 runner；不会自动生成 run.json、切换 mode、注入故障或替你点击 Cancel。

一个 case / mode / repetition 对应一个 run 目录，建议放在 `debug-output/agent-validation/<run_id>/`。本目录保存计划、问题与解决摘要；原始大日志、图片、rosbag 留在 debug-output，不把 API key 或完整进程环境写入证据包。

| 内容 | 保存要求 |
|---|---|
| run.json | 以 [run-template.json](run-template.json) 为基础；记录执行前的预期，不事后补写成实际结果 |
| 版本 | HEAD + dirty 文件列表 + 与此问题相关的脱敏修改/配置副本；只写 HEAD 不足以定位本工作区版本 |
| 启动上下文 | nav2/arm_test、地图/场景、机器人起点、模式、ack 规则、LLM/embedding/API开关、实际 model、实测数据率/温度 |
| Skill / Mission | `debug-output/agent/<session>/skill_events.jsonl`，保留本 run 的 mission_id 对应事件 |
| JEV | 同 session 的 `jev_calls.jsonl`：每个 stage 的 request/questions/raw answers、provider、latency、fallback；缺 request_id 的调用按命令/时间关联并标注关联可靠度 |
| Recovery | `/agent/recovery_request`、`response`、`state`、`events`、`command`；原始恢复动作日志 `debug-output/recovery/`；没有文件时明确缺失 |
| 当前与历史 | `debug-output/human_reports/human-report-*.json`，配套 Health/任务/Recovery 截图；报告不能替代原始 topic |
| 感知 | 所有 capture/检测图片及时间，含没有检测出目标的图片；记录人工核对“确实不存在”还是“存在但漏检” |
| 运动/定位 | 目标参数、配置容差、wheel odom、GICP pose/score/age、静止漂移；有真值就另存，只供评估，不作为 agent 偷看答案的输入 |

本轮只做资料归纳，不开 thinking LLM 自动调查。未知原因生成调查包：观测事实、相互矛盾的证据、已尝试动作、未验证假设、建议的下一项只读检查。

## 2. 轻量 rosbag 记录（可选，但推荐用于恢复协议）

以下在另一个终端手动运行。确认 `rosbag2_transport` 已安装；保持与本次 Sim 相同 ROS_DOMAIN_ID/RMW，不另起同名 agent。`RUN_DIR` 示例需每次换唯一 run_id。

```bash
source /opt/ros/humble/setup.bash
source /home/ruan-x/robot_ws/install/setup.bash
export ROS_DOMAIN_ID=20
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
unset CYCLONEDDS_URI
ros2 pkg prefix rosbag2_transport
export RUN_DIR=/home/ruan-x/robot_ws/debug-output/agent-validation/20261009-R02-classic-01
mkdir -p "$RUN_DIR"
ros2 bag record -o "$RUN_DIR/rosbag" \
  /agent/text_command /agent/cancel /agent/mode_command /agent/mode_state \
  /agent/mission_state /agent/mission_events \
  /agent/health_state /agent/health_dashboard_state \
  /agent/recovery_request /agent/recovery_response /agent/recovery_state \
  /agent/recovery_events /agent/recovery_command \
  /robot_mode /safety_status /cmd_vel /nav2/cmd_vel \
  /wheel_odom /gicp_loc/pose /gicp_loc/score /initialpose /clock
```

任务结束后对 recorder 按 Ctrl+C 正常封存。此记录默认不订阅原始图像/点云，避免额外负载改变待测问题；需要时另开专项采集并记录开关。`/agent/human_report` 是 Dashboard websocket 内部转发状态，不能假设它是 ROS topic，保存 JSON 文件即可。

不要将 rosbag 中的 command、initialpose、recovery_command 回放到 live domain。后续回放评估应使用隔离 domain、只读/假 backend，并排除动作 topic。

若缺 rosbag 工具，先保存已有 JSONL、human report、launcher/ROS 日志和人工操作时间表；把未记录的协议细节标为 UNKNOWN，不根据截图推断完整时序。

## 3. 时间与结果分类

人工记录用 JST ISO 时间带 `+09:00`；原始日志保留 UTC，不重写。ROS 模拟时间用于对齐仿真状态；Pause/恢复超时和 API 延迟用墙钟/单调时间，避免暂停导致统计时间不前进。

记录 t_command、t_plan_ready、t_failure、t_request、t_recovery_action_start/end、t_verified、t_resumed、t_terminal、operator 等待时长。

| outcome | 含义 |
|---|---|
| PASS | 达到预先写下的预期；拒绝不支持/有条件的危险命令也可以是合格行为 |
| FAIL | 证据充分，实际行为违反预期 |
| PARTIAL | 明确只完成部分子目标，不伪称全部成功 |
| NOT_REPRODUCED | 未观察到要复现的症状，不证明修复成功 |
| INJECTION_INEFFECTIVE | 做了注入操作但没有形成目标故障，不进入恢复成功率分母 |
| BLOCKED_TOOLING | 缺注入/观测/评估工具，不算 agent 失败 |
| CANCELLED_BY_TESTER | 非计划的手动取消，成功率中单列；若 case 本来测试取消，则正确取消记 PASS |
| UNKNOWN | 日志缺失、状态冲突未能解释，不猜通过 |

额外单列 failure_layer：parse、slots、plan、precondition、skill、health_evidence、recovery_request、diagnosis、action、verification、resume、ui_report、api、test_setup。

## 4. 有效对照：不要一次改变所有模块

### A. JEV 的增量价值

- 固定同一 parser fallback 设置、skill/catalog、安全门和 Recovery 配置，只切 Classic / JEV Primary。
- Shadow 用于同状态下记录 JEV 建议与 Classic 的差异；不把它执行成功记为 JEV 成功。
- 已知 M08 的 Classic 漏步骤是可复现种子，但不能只测试这一句。保留未参与修改的表达集。
- Primary 必须记录最终采用的是 JEV plan 还是 Classic plan；“处于 Primary”不等于“JEV 独立计划执行成功”。
- Live、local fallback、API失败分别统计；JEV confidence 不是实测准确率。

### B. Recovery 的增量价值

- 固定 agent mode、场景起点、目标、故障类型和注入时机，只比较 recovery 自动执行与 diagnose-only。
- 当前 launch 支持 `recovery_execute:=false`；quickstart clean 脚本目前没有对应环境变量转发开关。使用完整的原启动参数切换 launch 参数，不能仅设置一个未经接线的 `RECOVERY_EXECUTE` 就声称关闭了自动恢复。
- diagnose-only 仍有诊断。2026-10-09 已移除 Mission 定位兼容自动 reset，但仍需核对其余等待/重试逻辑和实际动作日志，不能称之为“无恢复”基线；完全关闭的研究对照仍需明确开关/假 backend。
- 注入后，测试者不得在“自动恢复组”悄悄手动修好。必须记录 agent action / operator action / 自然恢复三种归因。
- 状态自己变好、operator 解除问题、自动重启确实产生恢复、重新发一条任务成功，分别是四种事实。

### C. 交互效应（后做）

基础稳定后做 Classic/Primary × auto/diagnose-only 四格实验。每格恢复相同场景/起点/故障，轮换顺序；未恢复初始条件的 run 不能直接成对比较。

## 5. 统计哪些指标

| 指标 | 分母 / 解释 |
|---|---|
| 完整解析率 | 完整任务图 + 参数正确的条数 / 有预定义正确答案的命令数；漏任务算错 |
| 合理澄清率 | 对预定义含糊/不支持命令，正确澄清/拒绝的比例；与可执行任务完成率分开 |
| 端到端完成率 | 全部要求达成的任务 / 有效可执行试验；同时报告 PARTIAL、取消、超时数量 |
| 误阻断 / 错误继续 | 预先标注应该继续却停止、应该停止却继续的次数；安全违规逐条列出 |
| 根因与动作正确率 | 有确认故障真值的 run；没有真值时只能报“与当前证据一致” |
| 恢复率 | 有效故障中被验证解决的比例；按自动动作/人工/自然恢复分列 |
| 恢复后续行率 | 已验证恢复且原任务未取消的 run 中，原 mission 正确重查并完成/部分完成的比例 |
| 人工成本 | operator 请求数、重复弹窗数、点击数、实际人工操作数、等待时长 |
| 效率 | parse/review/API/skill/recovery/总墙钟分别记录；报告中位数、p95 和 n，不能以 API 总耗时代替任务耗时 |
| cost | 真实 usage 与本地估算分列；保留计价来源和日期，缺 usage 写 null，不能写 $0 |

目前脱敏规则会将包含 `token` 的计数字段一起隐藏；因此部分 usage 无法从原始日志审计。把它列为观测问题，修复前只能报告估算，不能推断实际账单。

能支持设计收益的证据是：在相同任务/故障条件下，完整解析和任务完成更多、人工更少、无观察到的安全违规、额外延迟在预设预算内。小样本零事故不证明系统绝对安全；只报观察结果。thinking LLM 尚未接入时，不宣称解决未知故障或比 LLM-only 更快。

## 6. 每轮小结模板

```text
round_id / code_revision / config_revision:
cases planned / executed / blocked:
valid runs per condition:
PASS / FAIL / PARTIAL / UNKNOWN / cancellations:
new problem IDs / repeated problem IDs:
confirmed facts:
hypotheses still unverified:
automated recovery / operator recovery / natural recovery:
comparison result and confounders:
fixes selected for next round:
required regression cases:
```
