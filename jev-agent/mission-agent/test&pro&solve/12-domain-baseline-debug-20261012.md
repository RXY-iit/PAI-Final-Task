# Full 模式领域协作基线修复与验证

日期：2026-10-12。现有工作树包含其他开发改动，本轮未提交、未回滚它们。
场景/map：sim_factory_mid360_like_gpu_trueimu_011；Full/nav2；ROS_DOMAIN_ID=20；RTF 采样约 0.4。
测试使用现实时间记录，未改仿真时钟、碰撞门、定位要求或到达容差。

## 1. 首要根因：同一 ROS Domain 有两套 Agent

旧进程 agent_node=2230110、dashboard_node=2225799、health_monitor=2214888、recovery_agent=2194870，来自 10 月 10 日。
新 Full 进程来自 10 月 12 日。ROS 图直接观察到各两个节点及两个 mission_state 发布者。
旧进程命令名为 python3，单纯按 comm 查 agent_node 会漏掉它们。

影响：同一用户指令可被两个执行器处理，旧/新计划与状态交替发布，两个 Nav2 goal 相互干扰风险，8765 被旧 Dashboard 占用，旧 Recovery incident 重新出现。
已核对具体可执行脚本与 PID 后 SIGINT 旧四进程；随后仅重启本次 Agent/Dashboard 四服务，不重启 Isaac/Nav2/localization。ROS mission_state 发布者验证为 1。
新增 `tools/agent_stack_guard.py`，sim clean launcher 启动前检测同工作区、同 domain 的脚本 argv，包括 Python 孤儿进程；发现已有实例拒绝启动并列出 PID，不自动误杀。
这是标准 launcher 的保护，不是禁止所有直接 ros2/manual 启动方式的跨机器单例锁。

## 2. 本轮代码修复

| 问题 | 修改 | 验证 |
| --- | --- | --- |
| 7/9、隐藏 8/9 | UI 只显示/计算 kind=skill，排除 task/task_end | Playwright 桌面 1365x900、窄屏 390x844 均为 7/7，无 JS 错误/横向溢出 |
| check_localization 被误画为 Recovery 工作 | Agent 状态采用实际 owner，task_end 不参与 | 前端纯函数回归 |
| 搜索正常未检出被当成故障 | OK + NOT_FOUND + goal_satisfied=false；Task 汇总保留 UNMET | 单测、真实九视角负例 |
| 后续任务是否继续 | 独立/cleanup 与依赖失败分开；必要时 task_outcome_review，required_condition 不允许模型绕过 | 独立/条件/不明确单测；实测 unconditional home |
| 重试时反复 JEV fault check 后建议未采用 | 有界 Navigation 重试先处理；仅未处理异常监督；建议标为 proposal | Shadow 回归确保内部重试不调用 supervise_step |
| 所有候选失败后回到已失败原点 | 候选 key 已在失败集时禁止再次重试 | 耗尽回归；未做远端候选实测 |
| OCR 对棚号6/06不一致、小字漏检 | 仅 shelf label 允许前导零等价；完整数字 token；额外一次2x OCR | 数字负例；18张历史图片回放；在线正例 |
| 日志把 input_tokens/required_tokens 当秘密 | 保留非凭据 token 字段，仍脱敏密钥/访问令牌 | 脱敏回归及 live usage |

## 3. 真实执行记录（不是 mock/dry-run）

证据复用：`debug-output/sessions/20261012T022650Z-4f03f3df/agent/skill_events.jsonl`、`jev_calls.jsonl`，以及同 session missions/human_reports。完整图像仍在 debug-output/perception，不额外复制。

| Mission ID | 请求/操作 | 结果 | 墙钟时长 | JEV 异常监督 |
| --- | --- | --- | --- | --- |
| dcc76b090262 | find shelf_label_06 | 九视角，FOUND，SATISFIED | 15.84s | 0 |
| 14fe7cfae015 | find shelf_label_99, then set pose home regardless of whether it is found | NOT_FOUND 正常返回；arm home 成功；PARTIAL | 16.65s | 0 |
| 5f21e132df25 | go to shelf_06 | Navigation + 嵌套 Perception 满足；七个 Skill | 14.93s | 0 |
| 107e495b3473 | find shelf_label_06，采集开始后取消 | CANCELLED，无后续 Skill/旧任务复活事件 | 1.15s（含解析） | 0 |

前三条共 10 次真实 JEV 调用：command_parse=3，plan_review=3，perception_model_review=3，nav_target_review=1；provider 全部 live。实际单次 API 延迟 152.7–215.8ms。日志已保留 API usage，不把 UI 估算 cost 视为账单。

导航分级：本轮一条 goto_pose 为 **task_tolerance**，Nav2 状态 cancelled_after_task_tolerance_reached；严格 Nav2 成功 0、语义容差成功 1、降级成功 0。机器人本来就在目标附近（约 0.12m），本轮不证明长距离导航稳定；不改容差把它包装成严格成功。
相机扫描与 Arm home 为实际执行；不引入底盘探索或未检查的远端路线。导航试验若触发替代点则取消的观察脚本未触发该分支。

## 4. 自动化验证

- 本轮针对性 Python 回归 65 passed（领域委派、依赖、取消、恢复 handoff、schema/candidate、OCR/composite、重复进程保护）。
- 前端 `node --test test/mission-logic.test.cjs` 通过；`npm run build` 通过。
- 真实 Dashboard 浏览器验证显示 7/7，无 pageerror。
- 拓展 real_lab 回归仍有原有 8 个失败：6 项依赖缺失 real 地图夹具，2 项相机标定旧期望。本轮不改 real 地图/标定来消除这些失败。
- 历史两组 OCR 图片共18张：一组恢复匹配，另一组仍漏检。不把一次在线成功视为完整识别稳定性验收。

## 5. Recovery 告警与优先级

重启后 incident 日期已变为本次启动，旧忽略列表不再通过残留发布者重现。
RTF 约0.4、sim相机640x360 vs预期640x480为 advisory；CPU约82°C是资源 degraded。fast_lio_odom、RGB/depth 同步有间歇 degraded，需检查新鲜度/同步影响，不能据“有 PB 卡片”断定需要先修所有卡片。
运动前直接证据：localization OK、GICP约0.119、最新可信 pose+odom有效、AUTO；WATCHDOG 在当前系统健康门中为 OK。
急停、实际定位失效、陈旧 costmap/传感器、碰撞门阻断需要优先处理；RGB OCR 不应被无关深度告警阻断。未执行任何 Recovery 修复动作，因此没有计作自动恢复成功。

## 6. 留待后续

1. 第一组历史图的 OCR 倾斜/小字漏检；EasyOCR 权重尚未部署，不自动联网下载。
2. 货架数字与具体货架实例关联尚未证明。
3. Navigation 的路径/窄通道/临时人类规则 JEV 仍未实现。
4. 当前 query/领域 JEV 比较简单，具体真实问题见两个 Agent 的 runtime-question-audit-20261012.md。
5. 本轮不是多次冷启动、长距离或 Recovery 故障注入验收；也不能据这些单次结果证明 JEV 比 Classic 更优。
