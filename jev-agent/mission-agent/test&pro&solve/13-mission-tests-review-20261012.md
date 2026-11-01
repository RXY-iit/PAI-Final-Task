# M01-M17 运行审查与修正

日期：2026-10-12。依据会话 `debug-output/sessions/20261012T095147Z-ec8696d8`，23 个用户 Mission 和一个独立 system-recovery report。本轮不改写历史报告，不把 CANCELLED/SUPERSEDED 当作执行成功。

## 证据入口

- 原始 JEV 输入/答案：会话内 `agent/jev_calls.jsonl`。
- 执行事件：`agent/skill_events.jsonl`。
- 每次 Mission：`missions/<id>/human-report-<id>.json`。
- 本轮只读 API 回放：`debug-output/validation/20261012-m01-m17/`，不含运动或图像外发。

## 逐项审查

| Case / Mission | 真实执行 | 问题及处理 |
| --- | --- | --- |
| M01 / 2b785209956b | get_robot_status 完成 | final_message 仍匹配旧 robot_status 名，丢失状态概要；已兼容新名称 |
| M01b / 77d3b102a42e | 12 个已知目标列表返回 | 本次正常 |
| M02 / 3a5be11d37cd | 静止、转相机、拍摄、归位完成 | 本次正常 |
| M03 / a8ef5f297101 | 静止、arm home 完成 | 本次正常 |
| M04 / 6f5889ad16de；cefe27c633bf | 后退完成；后者经过 AUTO 等待后同任务续行 | 报告与 Mission 终态一致，前轮等待状态修复有运行证据 |
| M05 / M05b | 本会话无单独用例 | M14 中有 turn left，不能替代独立/180 度验收 |
| M06 / ba0a774c7f6c | 少量前进完成 | 本次正常 |
| M07 / 046a7a92e437 | goto_pose 成功，completion_class=task_tolerance；子任务拍照前澄清，随后用户取消 | 不是 Nav2 严格成功，也不是 OCR 未看见目标；Perception 错把 shelf_label_06 选成 object/none，生成 0 Step。修复地图确认合约及 OCR 问题结构 |
| M08 / 7214a5cd34be | 后退、向左观察完成 | 本次正常 |
| M09 / M09b / M10 | 本会话未找到对应命令 | 不计覆盖 |
| M11 / 6507196a7942 | 搜索 PARTIAL 后 home 完成，整体 PARTIAL | 明确 regardless 的独立归位正常，未检测完整目标不能算全成功 |
| M12 / 1084b2d69a14、18bd100e7261；car 变体 e2b7e01d7b2e | 未执行，条件句 guard 拒绝；后续新命令覆盖为 SUPERSEDED | 补齐有界条件图与基于实际检测结果的执行分支；red box 能力不足仍不能被当成 FOUND |
| M13 / 91d805f6bb38、4e66a396caf6 | NOT_FOUND 后自动后退，整体 PARTIAL | 不符合先询问的测试约定。JEV independent 标签不再自动授权运动；增加同 Mission 的确认/取消流程 |
| M14 / edfac04b23b5 | 后退与左转全部完成 | 没有中途取消证据，因此不能算取消测试通过；本轮补线程回归验证等待时取消及旧按钮失效 |
| M15 | 本会话没有对应三任务命令 | 不计覆盖 |
| M16 / c3e8e9250f64 | 四任务被 >3 限制拦截，无动作 | 预期边界，不放宽解析规模来掩盖问题 |
| M16b / d3e7ed5ab8fd | 不完整 go back ... 被拦截 | 正常安全边界 |
| M17 / 58214267b422 | live API 拒绝天气请求，无动作 | 误保留 local_fallback ambiguity；已改为真实拒绝原因 |
| M17b / bd69b954ff70 | shelf_9999 无地图匹配，无运动，用户取消 | 原“选择目标”却无候选；改 UNKNOWN_TARGET 与可操作说明，不猜替代目标 |
| 06 letter / 6a7285ab744c | object/none，未拍照就澄清 | 文字别名/参数编译缺口；补完整文字表达及 OCR 合约 |
| red box / 99d6e9565d21 | 仅支持颜色，PARTIAL | 正常表达能力缺口；不能将颜色检出当成复合对象检出 |
| car / 76f98409cc84 | 执行器报 FOUND | 不能因此认定正确识别：下文有模型/类别风险 |

## 明确缺陷与修正边界

1. **地图规则没有贯穿调用链**：活动地图已有 visual_arrival_confirm，但代码主要猜 shelf 命名。现补 query + recognition（OCR tokens/aggregation），Navigation 将它传给 Perception；不改变 approach 坐标或到达容差。
2. **父命令污染感知输入**：子 Task 的 raw_text 原是 go to shelf_06；现在审查当前感知查询。明确文字合约的合法类型为 OCR/unknown，不允许错误物体类绕过。
3. **别名计数与 pipeline 语义不清**：实际 API 会把 06/6 视为一个目标；代码改为一个目标、多个检查。plan_review 区分 HSV/YOLO/OCR pipeline 和支持片段，不把 runtime evidence 缺省混成语义不支持；实际能力和图像新鲜度保护保留。
4. **依赖与运行控制**：M13 问询后可同任务继续；M12 有界正条件分支基于当前搜索的完整证据，不依赖模型即兴授权。失败、PARTIAL、未知、矛盾不属于条件 false。
5. **记录**：Task 展开失败保存具体 message；OCR matched_tokens/required_tokens 不再因 key 含 token 被误脱敏。access_token/api_key 等仍脱敏。

## 检测风险，不伪装成修复完成

`debug-output/perception/76f98409cc84:t1:obs-50a1fa785e2f/annotated_observation_007_2b2961a431b2.jpg` 中黄色叉车被模型标成 car，图片也能看到 06 标牌。另一次 bus 检测被 car 查询接纳，因为当时 perception_targets.yaml 的 car.classes 包含 car/truck/bus。这是两类问题：模型误分类与过宽类别定义。本轮把 car 收紧为 car，单独增加 vehicle 接受 car/truck/bus，并补回归；但这不能解决叉车被模型标 car，单纯加阈值也不能证明正确。后者保留为模型能力/确认规则的待解决项，不声称模型已可靠。

M07 这次根本没有进入 OCR，因此不能把 0 Step 归因于相机分辨率；但也不能据此声称修复后 OCR 必然成功。须重新到位拍摄验收。

## 验证记录

- 全量后端：408 passed / 8 failed；8 项均为既有 test_real_lab_parser_semantic.py 资源/标定断言问题（6 项缺失真实地图相关资源，2 项旧相机标定预期）。未伪造实机地图或回退标定。
- 最终针对性回归：121 passed，包括 OCR 合约、M12 条件分支、M13 三模式确认/取消、未知目标、状态返回、报告脱敏、严格 car/vehicle 类别。
- TypeScript/Vite 构建、mission-view / mission-logic 测试通过；条件 false 的运动 Task 显示“条件不成立・スキップ”。Playwright 固定 WebSocket 回放在 1920/1440/1280/390 宽度通过层级、等待解除、取消和溢出检查，未连接 ROS 控制。
- 真实 JEV 共 1 probe + 6 次语义调用。前五次纠正了 OCR 类型，但暴露别名计数与总 decision 歧义。第六次仅调用修订后的 shelf plan_review，parse 使用记录答案重放：两个 check 均 supported，总 decision 仍 ask_human。没有把它计为端到端通过。
- 根据这个实际矛盾，去掉重复的总 decision API 问题，改为逐 check 的 supported/unsupported/unclear 汇总。原始答案本地重放、缺失答案拒绝均有回归；最新结构还没有新鲜的真实 API 验证（追加授权另行询问），也未执行新的机器人任务。
- 本轮尚未重启正在运行的 Agent，不修改正在执行的 Mission；没有发布新机器人动作。需要重启 Agent Dashboard 启动组加载 Python/地图规则，Isaac Sim 无须为这些修改重启。

## 复测顺序

1. 先 `find 06 letter`，检查 Perception 生成 search_at_place，parse=OCR，plan_review 说明可执行检查，不再 0 Step 无理由。
2. `go to shelf_06`：看 navigation_arrival 的分级，再看明确的 OCR 06/6 合约；同一批图像复用，两者 any；导航成功与视觉确认单独统计。
3. M13 在 NOT_FOUND 后不后退；Continue 后同 Mission 只执行后续运动；Cancel all 后旧 Continue 无效。
4. M12 先用 bottle 等直接类别测试 FOUND 与 NOT_FOUND，再测试 red box 的 PARTIAL 阻塞；错误不能当阴性，不能靠检测一个红色区域放行。
5. M14 真正中途取消一次；补齐本会话缺失的 M05/M05b/M09/M09b/M10/M15。相同条件重复三次前不宣称整体稳定。
