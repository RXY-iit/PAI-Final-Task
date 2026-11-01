# Navigation 人工选择、编号确认与 Routing 状态修复

## 本次运行证据

原始文件保持不变，未复制完整日志：

- `debug-output/sessions/20261014T081834Z-000d140d/agent/skill_events.jsonl`
- 同目录 `jev_calls.jsonl` 包含实际请求与响应。下表时间为 UTC。

| Mission | 时间 | 实际结果 | 原因 |
| --- | --- | --- | --- |
| `e1389329fef9` | 08:21:55–08:22:15 | `go to shelf 05，then find red object` 等待目标选择后被取消 | `shelf 05` 匹配两个已登记区域；需要选择本身是合理的 |
| `a0b26741c205` | 08:23:10–08:23:55 | 人工选 `navigation_target_0`，仍再次询问，然后取消 | 08:23:14 已收到正确 mission_id/wait_id 的选择，payload 已含 `navigation_selected_target=shelf_05_06_region`。之后 JEV 又问原目标能否唯一匹配，答 `ask_human`。不是按钮消息丢失 |
| `8bcc3420bd8a` | 08:24:00–08:25:23 | 改为 `shelf 05-06`；AUTO 选择成功恢复，导航后在编号确认解析阶段等待 | 导航产生 `shelf_label_05 shelf_label_06`，没有分组或 recognition contract。代码只建立一个 clause，JEV 判断两个 requirement，校验不一致 |

`8bcc3420bd8a` 卡住的是 Navigation 内部委托的编号确认，尚未执行顶层 `find red object`。本次证据不能证明红色识别失败。

另：该次导航记录 `completion_class=degraded`，XY 误差约 1.321 m、yaw 误差约 0.497 rad。不可记作 Nav2 严格成功；本次未调整导航验收条件，也未验证实际拍摄位置。

## 修复

1. **已确认的目标不再重复消歧。** Navigation 对选择的 ID 重新验证精确登记、approach 和规则。传递 `target_binding` 标明人工选择及原请求。省略已经解决的 `goal_binding` 问题，继续检查目标几何、视觉确认及完整计划。缺失或非法 API 答案仍阻止运行；人工选择不能绕过规则或执行安全门。
2. **编号确认显式配置。** `maps/sim_factory_mid360_like_gpu_trueimu_011/semantic_map.yaml` 的 `shelf_05_06_region` 增加 OCR `recognition`，tokens 为 `05`、`06`，aggregation 为 `all`。共享一次拍摄，执行两项检查。保持原来的两个标签都需确认的要求，不擅自改成任一标签即成功。其他旧区域的多编号 fallback 改用明确 `and` 连接，不再只用空格拼接。
3. **Routing 假等待修正。** 旧前端 `/ask_/` 会匹配 `task_graph_grounded` 内部的字符串。改为决策词边界匹配，同时避免把 `structure_human_clarified` 当作未确认。对于同 Task、同 Step、同 stage 后续已通过的复审，早先的等待记录标为历史，保留原答案；其他 Task 成功不消除当前等待。
4. **Task 状态一致。** 重新 refinement 时清掉旧询问摘要；返回 question 时不再把默认 `READY` 原样作为 `WAITING_HUMAN` 的 outcome。

## 验证及边界

- 新增：人工选择后仍使用同一 Mission；一次导航；嵌套编号检查后再执行红色检查；过期/重复人工选择拒绝；取消；规则阻止；已选 ID 不允许模糊匹配。
- 新增：地图 recognition 透传、两个 OCR checks 共用一次 capture、旧区域多标签显式 AND、`red object` 仅生成 `color_red` 而不虚构物体类别。
- 前端：`mission-view.test.cjs`、`mission-logic.test.cjs`；浏览器 `dashboard-layout.cjs` 使用隔离 WebSocket，在 1920、1440、1280、390 像素宽度验证状态与布局。截图 `/tmp/dashboard-layout-*.png`、`/tmp/dashboard-resolved-*.png`，为临时测试产物，不是实机记录。
- Python 回归：200 passed（22.92 秒）；结果 `debug-output/validation/20261014-domain-resume/regression.xml`。前端构建与 `robot_agent` symlink 构建通过。
- 本轮使用受控答案和 FakeClient 测试新协议；没有新增真实 JEV 调用或机器人运动。真实 JEV 对修改后问题的稳定性、OCR 是否实际检出编号、摄像位置是否合适，仍须现场重测。

## 重测步骤

重新启动 Agent stack 并刷新 Dashboard 即可加载新代码和地图，**不需要重启 Isaac Sim**。不要用旧页面的 wait_id 操作新 Mission。

1. JEV Primary 发送 `go to shelf 05，then find red object`。
2. 选定区域后应保持同一 mission_id，`navigation_command_parse` 不再重新询问区域身份；`command_bind` 应为完成。
3. 到达后应出现嵌套 Perception Task，一个 `search_at_place` 内检查 `text_05`、`text_06`。没有检出不等于解析失败；结果由任务依赖规则处理。
4. 只有前置目标满足或明确经过后续决策允许，才执行顶层 `red object`。不得为了演示继续而绕过失败的依赖。
5. 全部取消后不应再执行后续动作。若 JEV 再次询问，检查对应新 request/response 和 task_id，而不是仅看历史 Trace 的原始回答。
