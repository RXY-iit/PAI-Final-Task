# Dashboard / Agent 协作检查

日期：2026-10-12。范围：语言显示、历史等待、导航发布前澄清、父子任务依赖、报告终态乱序。未改动定位、碰撞门、真实硬件配置或 Recovery PB 判据。

## 1. 历史执行证据

会话：`debug-output/sessions/20261012T091838Z-4d598e91`。
原始 API 请求/响应在 `agent/jev_calls.jsonl`；事件在 `agent/skill_events.jsonl`；报告在 `missions/<id>/human-report-<id>.json`。没有复制完整报告或改写历史结果。

| Mission | 实际情况 | 结论 |
| --- | --- | --- |
| `984062618d39` / go back 20cm | await_robot_auto_mode NEEDS_HUMAN，随后同一步 OK，move_relative OK；Task SATISFIED | 执行已完成；step_supervision 持续显示确认待ちは UI 生命周期问题 |
| `6bf9ced1f7e8` / back, turn left, home | 三个 Task 都完成；六个技能结果 OK | 这一次 Mission/Arm 顺序协作正常，不等于全部动作稳定性验收 |
| `8b09d5051375` / find red box then home regardless | 感知正常结束但 goal_satisfied=false；home 完成 | 未检出不是技能异常；后续明确独立的 home 正常运行，整体不能统计为全部目标成功 |
| `728dbe8fe0e7` 等 shelf_06 | computed_path 审查 ask_human；随后监督提供 Continue: search_object | 未到达却推荐到达后检查，且该选项实际重发原 command，是明确的依赖与交互设计缺陷 |

四条 recorded live 路径审查均选择 ask_human，route_strategy 却选择 nav2。旧问题没有具体冲突分类，不能从这些 choice 结果断言模型究竟为什么拒绝。发现的输入缺口是：混合语义一致性与未提供的物理安全证据、没明确视觉确认发生在到达之后。新版问题修补这些缺口；下方补充经过授权的真实 API 只读回放，不将其计为机器人导航成功率。

## 2. 修正

- Readiness 标题、分项名称、状态、时间、说明、Recovery 编号说明/暂时忽视/恢复提醒统一 English；原本 Japanese 保留。原始用户输入与历史日志不强制翻译。
- UI 将原始 JEV 决策与当前等待分开：重试完成、已经进入后续步骤、Mission 终止或 Shadow 的旧 ask 判断不再显示当前确认待ち；原始问题/答案保留。
- failed Task 的子孙 Task 不作为后续独立目标候选；展开后的 task_end 保留 parent_task_id。禁止用“Continue: 某 Task”按钮暗中重发整条原始 command。
- navigation_plan_review 明确分工，新增 blocking_issue。语义计划包含到达后的 search_object；contract 标明 after_successful_arrival。32 个路径采样坐标仅在 JEV 摘要内取三位小数，路径 hash 和实际执行坐标不改。
- 发布前审查暂停直接等待原 Mission 的当前 Step 修复，不额外调用异常监督；resume 重新计算/审查当前导航，Cancel all 终止等待。不能以恢复按钮绕过安全检查。
- 报告跨 topic 乱序：终态 event 先到时，允许同 Mission 兼容的终态 state 补充最终任务/结果一次；后台健康或迟到非终态不能重开报告，原关闭时间保留。

定义源：

- `src/robot_agent/web/src/lib/missionView.ts`、`components/MissionProgress.tsx`、`ReadinessPanel.tsx`、`RecoveryPanel.tsx`
- `src/robot_agent/robot_agent/readiness.py`、`human_report.py`
- `src/robot_agent/robot_agent/agents/mission_agent.py`、`navigation_agent.py`、`navigation_plan.py`
- `src/robot_agent/robot_agent/jev_agent.py`
- `src/robot_skills/robot_skills/executors/navigation.py::_review_before_motion`

## 3. 验证与边界

| 检查 | 结果 |
| --- | --- |
| robot_agent/test + robot_skills/test | 380 passed / 8 failed |
| 已知 8 项失败 | test_real_lab_parser_semantic.py：6 项缺失真实地图资源，2 项旧相机标定断言；未改地图/标定来凑通过 |
| 新增后端回归 | 原 Step resume/cancel、无重复 supervision、依赖子任务不可继续、展开后父子关系、路径冲突 Primary/Shadow、报告终态乱序全部通过 |
| TypeScript / Vite build、mission-view 与 mission-logic | 通过 |
| Playwright 模拟 WebSocket | 1920/1440/1280/390 布局、等待解除、取消；1440/390 readiness 与 stale、JST 时间；无 JS 错误与横向溢出 |
| 在线只读核对 | sim / nav2 profile；只运行一套 Agent/Dashboard/Health/Recovery；当前任务 CANCELLED；AUTO，定位与 local planner OK |
| 新版真实 API | 初次因外发权限被阻止；用户明确批准后完成 4 次回放（另有启动可用性 probe）。3 次正常路径 execute/none/nav2；1 次规则冲突 ask_human/rule_conflict/ask_human；均 live、无 fallback |
| 新版机器人运动 | 本轮未发布运动或导航任务，不宣称 shelf_06 端到端已验收；未重启运行中的后端进程 |

在线 09:43 UTC 附近证据：RTF 约 0.485；CPU package 86°C（设备 max 80°C / crit 100°C），GPU 61°C；底盘 RGB 实际 640x480，约 3.17 Hz wall / 5.38 Hz sim。PB-HOST-RESOURCE 为 degraded/chronic，PB-SIM-RTF-LOW 已暂时忽视，fast_lio_odom 有 WARNING。实时 GICP pose/score age 约 0.55-0.65 s、score 约 0.125-0.127、定位 OK。性能告警不等于定位故障；640x480 是当下流证据，不声称是用户选择错误或新默认值错误。

### 真实 API 回放

原始请求/响应：`debug-output/agent-audit-20261012/route-review-replay-jev.jsonl`。来源是上述 session 最后一条 computed_path 的采样路径，不是新的 ComputePathToPose 或完整原始 247 点路径，因此新 path_id 不同，不能声称几何等价验收。

| 输入 | API 答案 | 耗时 | 估算输入 token |
| --- | --- | --- | --- |
| 原历史采样路径，无冲突规则 #1 | execute / none / nav2 | 163.6 ms | 1824 |
| 同上 #2 | execute / none / nav2 | 190.8 ms | 1824 |
| 同上 #3 | execute / none / nav2 | 173.4 ms | 1824 |
| 增加“唯一通路维修关闭，询问替代路线”的 advisory 规则 | ask_human / rule_conflict / ask_human | 191.7 ms | 1880 |

顺序为 decision / blocking_issue / route_strategy。四次估算输入费用合计约 $0.000309，不含 probe，也不是供应商账单。反例通过语义 advisory 测试模型选择，没有修改真实地图/规则；硬规则 deny_destination 仍由代码先拦截。返回内容进一步区分 implemented_route_strategy 与模型选择的 route_strategy，避免把被拒绝的 nav2 策略显示为已采纳。

## 4. 继续测试

前端已构建，刷新浏览器使用新 UI；后端 Python 和 readiness 文案需重启 Agent Dashboard 启动组，不需要为本轮修改重启 Isaac Sim。保留原启动组单实例保护。

1. `go back 20cm`：需要 AUTO 时批准，完成后原 ask trace 标为历史，不遗留等待。
2. `go to shelf_06`：导航未完成不得出现 Continue: search_object；审查澄清提供同一步 recheck/cancel，具体 blocking_issue 可在 trace 中复核。
3. 在上述等待期间取消，然后点击迟到的旧 resume：不得复活任务。
4. 真正到达后才执行货架标签确认；到位和 OCR 结果分别统计，缺目标不是导航成功证据。

待确认：新版 JEV 在更多路径/歧义/异常输入下的稳定性与长路线端到端运行；当前相机分辨率是否符合所选 Quick Start profile；既有实机测试资源/标定断言如何维护。本轮不放宽保护，也不因 WARN 自动重启节点。
