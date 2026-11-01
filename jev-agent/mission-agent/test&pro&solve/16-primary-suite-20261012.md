# JEV Primary 实测与 Classic 对照

初始测试日期：2026-10-12。追加审计：2026-10-13，用户全重启后的 19:16–19:23 JST 测试，见第 7 节。范围：Mission / Perception / Navigation / Arm 的解析、委派、失败分流、人工反馈和取消。Recovery 的独立 PB 优化后置。

**最新结论：M15 变体的三任务解析、Ask Human 消歧和同 Mission 继续已经有真实运行证据；本次审计发现的组合匹配、纠正交接、上下视角及重复消歧问题已做后续代码修复与离线回归（第 7.5 节），新版本仍待现场复测。不能以 DONE/SATISFIED 直接统计用户目标成功率。** 第 1–5 节保留 10/12 历史口径，不与追加样本混算。

## 1. 环境和证据边界

- Full / nav2，地图 `sim_factory_mid360_like_gpu_trueimu_011`，真实 JEV endpoint 使用现有配置。每次启动 probe 成功才显示 live；本轮执行记录逐条检查 provider 和 fallback。
- 保留 Isaac、Nav2、定位、驱动和 Arm gateway；为加载修复重启 Agent / Dashboard / Health / Recovery 栈，未同时启动第二套。
- 移动前读取局部 costmap。最初前障约 0.496m，先向已检查的空闲后方退 0.5m；之后检查前障约 0.971m，再做 0.2m 复合任务。后续旋转前最近各向障碍约 1.21m 以上。没有放宽碰撞、odom 新鲜度、导航容差或安全门。
- Arm gateway 为 MOCK；home/front 成功只证明调度和仿真反馈，不等于实机验收。
- **EasyOCR、wheel_odom_stale 根因、M07 的导航/OCR 执行验收按用户要求暂缓**。本轮没有为消除这些问题而改阈值或检测成功条件。
- Classic 与 Primary 的起点、视角、代码版本不同，本表是语义/执行契约对照，不是随机化同场景成功率实验。

证据根目录：`debug-output/validation/20261012-primary-suite/`。

| 文件 | 内容 |
| --- | --- |
| `runs.jsonl` | 真实 Dashboard 命令、mission_id、前后 pose、状态转换、任务与 Skill 结果、JEV 调用摘要 |
| `evidence-index.json` | 每个 Mission 的 canonical human report、session JEV 原始记录引用和 SHA256；不复制完整证据 |
| `source-manifest.json` | 最终 dirty-worktree 文件指纹；不声称是早期测试的冻结版本 |
| `failure-replays.jsonl` / `failure-replays-fixed.jsonl` | 修复问题设计前后，真实 JEV + 合成执行/Recovery 的隔离回放；不是现场传感器故障 |
| `failure-replay-jev-calls.jsonl` | 回放原始 request/response，已脱敏；按时间、命令和 case 关联 |
| `navigation-replay.json` / `navigation-replay-jev-calls.jsonl` | 已记录路线的只读 JEV 规则审查、导航到感知的委派契约 |
| `pytest-final.txt` / `build-final.txt` | 回归与 build 结果 |
| `launcher.log` / `latest-state.json` | 重启记录及最后运行状态快照 |

真实任务的每一步 question、answer、API 用量可从 `evidence-index.json` 的 `jev_log` 找到。Human report 继续作为 Mission 证据主体；run-template 仅提供测试评价结构，不生成另一份完整日志。

## 2. Classic / Primary 对照

Classic 详情见 [14-domain-agent-suite-20261012.md](14-domain-agent-suite-20261012.md)。下面“符合”包括合理拒绝、等待和取消，不代表所有目标被找到。

命令列保留实际拼写和标点；同组用例按编号顺序列出。以 `runs.jsonl` 为本轮实发命令依据；未实跑项目列的是设计命令/操作，不表示新增实测。

| 用例 | 具体 command / 操作 | 上轮 Classic | 本轮真实 Primary | 结论 / 边界 |
| --- | --- | --- | --- | --- |
| M01 / M01b | `ロボット状態`<br>`list known targets` | 状态、已知目标正常 | 同样正常，JEV parse/review 都 live | 只读路径符合 |
| M02 | `左を見て` | 观察、拍照、相机归位 | 3 次成功，每次 4 个 JEV 决策，正常步骤无 supervision | 相机职责清晰，不转底盘；不是同一初始 pose 的三次重复 |
| M03 / M27 | `set pose home`<br>`set pose see_front` | home/front 成功 | home 3 次成功、front 1 次成功，包含 arm_task_review | MOCK 调度符合 |
| M04 / M06 | `go back 20cm`<br>`前進少し` | 前后 20cm 成功 | 前后均成功；实际位移在 report 中 | 无条件放宽、无多余 supervision |
| M05 / M05b | `turn left`（本轮在复合任务中执行）<br>`turn around` | 左右转成功；180° 曾 1 次 stale、2 次完成 | 左转作为 M10/M15 子任务完成；独立 180° 完成，约 35.6s 含 runner 收尾 | 单次 180° 成功不证明 stale 根因消失 |
| M07 | `go to shelf_06`（本轮只读回放） | 到达 task_tolerance，OCR 未确认标签 | 不再执行整段导航；只读 5 次 JEV 决策通过，导航 7 步、感知 1 个 search_at_place，06/6 共用采集 | 证明契约连通，不计导航到达或视觉确认成功 |
| M08 | `go back a little, 左を見て` | 后退 + 左看成功 | `798ccd375830` 两 Task 完整执行 | 混合语言、跨 Agent 委派符合 |
| M09 | `go back 20cm, then set pose home`<br>M09b: `go back 0.2 m, then set pose home`（未重跑） | 后退 + home 成功 | `dc433d548b4c` 完整执行 | 20cm 参数保留；M09b 本轮未单独重跑 |
| M10 | `go back a littlt and turn left` | typo 无法理解，转人工，LLM 不可达 | `93389821fc38` 正确后退 0.2m + 左转 | 本轮明确观察到的 JEV 解析增益；不是所有拼写扰动的稳定率证明 |
| M11 | `find red box, then set pose home regardless of whether it is found` | PARTIAL 后按 regardless 执行 home | `f9048ac98d7f` 同样继续 home，Mission PARTIAL | 不虚报找到 red box、不启动无关恢复 |
| M12 | `if you find a red box, go back 20cm; otherwise stay still` | 不完整检测阻止条件运动 | `432d8599926e` PARTIAL -> CONDITION_UNKNOWN -> 人工/取消，无后退 | 未知不等于 NOT_FOUND |
| M12b | `if you find bottle, go back 20cm; otherwise stay still` | 完整 NOT_FOUND 后保持不动 | `d61bc7627f9a` 同样 SKIPPED/CONDITION_FALSE | 阴性条件分支符合；真阳性条件分支本轮仅离线覆盖 |
| M13 | `check red object and go back a little`<br>阴性对照：`check bottle and go back a little` → 人工批准继续 | 未检出 -> 人工继续/取消 | 原命令 `ae008939ad5d` 检测到红色后正常后退；换 bottle 的阴性对照 `2fbc2f0bbd15` 请求人工，批准后同 Mission 后退，整体 PARTIAL | 原红色命令因真实 FOUND 不应强制问人；阴性对照只改目标，不混称同一场景结果 |
| M14 | `go back 20cm, then turn left` → 后退中 Cancel all | 运行中取消，不转向 | `008d2d87b389` 取消后退，无后续转向 | 实际取消链符合 |
| M15 | `go back 20cm, then turn left, then set pose home` | 三任务全部完成 | `dd5982936be9` 完成，plan_review 选择 Classic 计划 | 这是 Primary 允许的方案比较，不是 API fallback；不能称为纯 JEV 计划成功 |
| M16 / M16b | `go back 20cm, then turn left, then set pose home, then look right`<br>`go back ...` | 四任务/缺词拒绝 | 同样拒绝并可取消，没有截断执行 | 上限边界符合 |
| M17 / M17b | `今日の天気は？`<br>`go to shelf_9999` | 非指令提示偏 llm_unreachable；未知货架拒绝 | 天气问题识别 not_robot_command；未知货架明确未注册 | 提示更贴近原因，无编造目标 |
| M19 | MANUAL 下 `go back 20cm` → 选择 set_auto → 原 Mission 继续 | MANUAL -> set_auto -> 原 Mission 继续 | `b287a9f99b17` 真实 supervision=ask_human_enable_auto；新 AUTO 反馈后重查前置，运动仅一次 | 三个 SkillResult：失败前置、成功前置、成功运动；无无关 Recovery |
| M21 | `go back 20cm then turn left` → JEV parse 在途 Cancel all | 只做离线取消隔离 | `5bf3d27ade5f` 发现取消后多做 review；修复后 `d76e4ae5fd4e` 只有 parse 1 次，无动作 | 真实 API 在途取消符合；不可达 API 场景仍是离线覆盖 |
| M22 | 取消旧任务后发送 `ロボット状態` | 取消后新任务完成 | `9a6c0efcae60` 新 Mission 完成，旧任务未复活 | 非毫秒级并发压测 |
| M23 | `find 06 letter`（未重跑） | OCR 目标映射正确，检测召回不足 | 本轮不重复跑 OCR；映射由 M07 只读委派确认 | 按要求暂缓模型执行问题 |
| M24 / M25 | `find red bottle`<br>`find cup or bottle` | red bottle / cup OR bottle，共用 9 图，NOT_FOUND | 同样复用采集并返回未满足目标；整体 PARTIAL | 未检出是正常检测结果，不误唤醒 error supervision |
| M26 | `check left side of robot if anything dangerous?` | image_describe 未配置，拒绝假安全 | `7f2e67d86f81` 明确 UNSUPPORTED，未报“环境安全” | 能力边界符合 |
| M28 | `set pose unknown_pose` | 修复旧 candidate 假成功后拒绝未知 pose | `a5fc204b4871` 在 JEV parse 低置信度处已等待，无 Skill 执行 | 合理拒绝，但不是 Arm 未知 pose 防线本轮再次触发 |
| M29 / M30 | `touch blue dot`<br>`grasp bottle` | 自主 touch/grasp 未完成，拒绝 | 同样在 Arm 能力边界等待 | 不把 command 入队当完成 |
| M31 | `find bottle` → search_at_place 采集中 Cancel all | 扫描中取消，后续不复活 | `914346f8778e` search_at_place 返回 CANCELLED，无后续动作 | 真实组合 Skill 取消符合 |
| M18 / M20 / M32 | M18: arm_test 下 `go back 20cm` / `turn left` / `左を見て`（未实跑）<br>M20: 相机断流后 `左を見て`（本轮隔离回放）<br>M32: 注入空/非法 API 回答（离线回归） | Arm-only / 断流 / 非法 API 主要离线 | 保持 Full；断流用隔离 F01/F04，非法答案用回归 | 不冒充真实 Arm-only、相机节点重启或完整 PB 验收 |

## 3. 本轮发现并修复

| 缺陷 | 影响 | 修正 / 验证 |
| --- | --- | --- |
| 旧运动失败分支抢先返回 | JEV 已选择 Recovery，但没有真正调用 | Primary 失败处理统一入口；注入 FAILED_NAVIGATION 的回归先失败后通过 |
| NEEDS_HUMAN 分支 Recovery 后 `index += 1` | 跳过失败的静止等前置条件，直接执行后一步 | 成功后重查同一步；前置再次失败时不得继续 |
| 部分通用失败没有恢复次数上限 | 同一故障可能反复恢复 | 同步骤最多一次自动 Recovery、一次普通自动重试；持续失败转人工 |
| 失败后人工选项仍是重发整条命令 | 重跑已完成任务，不能可靠保留后续计划 | scoped retry / continue 原 Mission；批准继续只对当前依赖边界生效，不重复询问 |
| 相对运动恢复后可能完整重放 | 已走 8cm 再走原 20cm，超出原目标 | 禁止相对运动自动重放原 payload，恢复后等待；剩余距离重规划待设计 |
| 相对运动仍被描述/映射成 Nav2 | 可能把 odom 故障错送全局定位恢复 | 契约增加 relative_motion_fault/drive，传入 wheel_odom_rejection；Recovery 请求不再因 relative_motion alias 授权 GICP 恢复；仍保留 AUTO 门 |
| task_end 和旧 Task 名的风险标签错误 | 内部结束标记像动作；Arm home 被标只读 | 监督窗口排除 task_end，canonical Arm/navigation Task 标物理运动 |
| live 问答沿用本地概率字段 | API noul 已返回低值，但旁边仍显示本地高概率 | live trace 仅保留真实 API 概率，缺失项标 missing；缺监督 action 转人工 |
| 解析/审查返回后未立即检查取消 | 取消后浪费一次 API，可能短暂发布过时阶段 | parse/review 返回处检查取消；M21 修复后只一次调用、零动作 |
| FAILED_SKILL 根 outcome 可为 null | report 无法直接区分失败与未判定 | 终态设 FAILED；Task/Skill 原因仍保留 |
| report 误脱敏 input_tokens_est | 不能复查计费用量 | 保留用量键，access_token 等仍脱敏；历史按 API log 核对，不重写 |

监督输入裁剪没有删除故障原因。所有 Skill 的问题仍从 `skill_catalog.v2.yaml` 读取；本轮只补充相对运动的真实证据字段和分类。

## 4. 真实 JEV + 隔离失败回放

| Case | 最终结果 | 证据边界 |
| --- | --- | --- |
| F01 拍照断流 | JEV -> Recovery；IN_PROGRESS 后等待 final；重试原 capture；再回相机姿态、Arm home，SATISFIED | 真实 JEV；故障和修复回复为测试替身 |
| F02 部分运动 + odom stale | 原问题设计误判 localization/stop；修正后 relative_motion_fault + drive，实际调用 Recovery；返回成功后仍 NEEDS_HUMAN，不重放位移 | 未修复/注入真实 wheel_odom_stale，也未验证 Drive PB 能修好它 |
| F03 Recovery 中取消 | CANCELLED，迟到 RESOLVED 不执行后续 Skill | 验证 Mission 协议，不是硬件停止耗时 |
| F04 持久相机故障 | 1 次 Recovery、2 次 capture，第二次失败后 NEEDS_HUMAN | 无无限自动恢复循环 |
| F05 MANUAL | ask_human_enable_auto，0 次 Recovery、0 次 move | 真正 AUTO 切换/继续另由 M19 现场验证 |

人工 retry / continue / cancel 三分支、错误 wait_id、缺失回答、Recovery 最终回复关联和 Shadow 不采纳由离线回归补充。不要把合成恢复成功计入 Recovery PB 成功率。

## 5. 统计与限制

最终准确数量、ID、调用耗时和费用见 `evidence-index.json.summary`。统计口径：

| 指标 | 本轮记录 |
| --- | --- |
| Dashboard 实际执行 | 39 次，含空间调整、重复、预期拒绝和取消 |
| 终态 | SATISFIED 22、PARTIAL 4、CANCELLED 13；不能据此报 100% 任务成功 |
| Mission trace API 决策 | 107 次 live，0 次 fallback；另有 2 轮 x 5 个隔离故障 case、5 次 Navigation/Perception 只读决策，均单列 |
| 计划选择 | 1 次 M15 选 Classic；其余已批准任务选择 JEV，不含解析拒绝 |
| 单次决策耗时 | 中位约 211.8ms，P95 约 264.8ms，范围 157.4–343.6ms；不是整任务耗时 |
| Mission API 输入用量 | 344,047 tokens；输入成本估算约 $0.01445，来自 API log / adapter 统计，不是账单 |
| 回归 | 436 passed；排除已有 real-map/camera 测试依赖文件 `test_real_lab_parser_semantic.py`，其历史 8 个失败见上轮报告，不声称全仓测试通过 |
| Build / 部署 | robot_agent + robot_skills build 成功；最终已重启加载，四节点各一份 |
| 最终状态 | `71aa6cd85ce1` 只读任务 DONE；Primary/live、AUTO、定位 OK；未回初始位置，map 约 `(2.01, 4.26, 1.00rad)` |

- 真实命令的 live/fallback 与隔离回放分开；probe 不在 Mission trace 内，不把其费用算成零。
- DONE/SATISFIED、DONE/PARTIAL、预期取消、能力拒绝不同类，不把它们混成“全部成功率”。
- M15 采用 Classic 计划单列；这正是设计允许的 plan_review 选择，不隐藏在纯 JEV 成果里。
- 没有本轮导航到达执行，因此 strict/task_tolerance/degraded 都没有新增样本。M07 只读通过不计导航成功。
- 没有实时测量全部恢复动作的因果贡献；温度或同步告警自然消退不算 Recovery 修复成功。
- 费用为现有 adapter 的输入用量估算，不是账单；不包含 probe、隔离回放、输出费用和服务商调整。

## 6. 继续测试前需知道（历史限制及更新）

1. 10/12 的依赖关系以全局 policy 为主，M15 曾由 review 选 Classic。10/13 已增加原文绑定、逐 Task depends_on / dependency_policy 和人工类型确认，见 [17-command-grounding-and-perception-capabilities-20261013.md](17-command-grounding-and-perception-capabilities-20261013.md)。本次 M15 变体采用 JEV 计划并完成，不表示任意 DAG/条件结构已经支持。
2. 部分动作失败后的“剩余距离/角度继续”尚未实现，目前保守不重放；是否在恢复后自动跳到明确独立的 Arm home，还是始终人工确认，留待后续确定。
3. 新视角重试没有动态重规划器；不会把原视角重试伪称为换视角。
4. EasyOCR 缺模型、OCR 召回、相机当前 640x480、wheel_odom_stale 根因和 M07 完整成功仍单独待办，**不代表本轮 Agent 解析/分流失败**。
5. Host 温度、低 RTF、RGB/depth 同步抖动等只记录，未改 PB 阈值或 real verified。仅修复会直接干扰 Mission 的 relative_motion 能力映射。
6. Full 冷启动、长期多轮同起点、真实断流、所有条件真阳性、Shadow 全矩阵和真实 Recovery 修复仍未完整验收。本轮支持“特定解析和失败分流有收益”，不支持“复杂任务均稳定”结论。

## 7. 2026-10-13 全重启后追加复核

### 7.1 证据和验证方式

用户实际执行 session：`debug-output/sessions/20261013T101452Z-2bc01fff/`。本次审核覆盖 **8 个 Mission：6 Primary、2 Classic**；另一个 `manual-1791886498887` 是 system_incident，不算用户任务。

- 原始问题/选项/回答：`agent/jev_calls.jsonl`；执行和人工选择：`agent/skill_events.jsonl`。
- 每个任务仍以 `missions/<mission_id>/human-report-<mission_id>.json` 为证据主体，不复制完整 report。
- 汇总、源文件 SHA256、按 Mission 关联的阶段、耗时和离线复现结果：[audit.json](../../../../debug-output/validation/20261013-restart-review/audit.json)。审计时读取的是终态任务文件，不宣称整个启动配置曾冻结。
- 可复跑的只读审计：[audit.py](../../../../debug-output/validation/20261013-restart-review/audit.py)。重放已记录 JEV 回答，直接调用当前本地编译/匹配函数；不发 API、不读实时相机、不发布 ROS 命令、不运行新视觉推理。
- 用户真实 API 记录已包含新版 alternative_type、command_bind、human_confirmed_task_types，说明这些修改在新进程生效。响应实际模型为 `jev-1.13.0`，请求别名为 `jev-latest`。

### 7.2 追加 Classic / Primary 对照

原样保留命令拼写。这里的“Classic 未测”是本 session 未测，不用上轮不同条件结果补成同期对照。

| 用例 / 具体 command | 本次 Classic | 本次真实 Primary | 审核结论 |
| --- | --- | --- | --- |
| M15 变体：`go back 0.2 meter, next turn left, after that set pose home` | `56404b309368`：仅后退 0.2m；系统仍报 DONE/SATISFIED，遗漏左转与 home | `0c916991776d`：识别三项；第 3 项提出底盘导航 / Arm 姿态两种解释，用户选择 Arm；同 Mission 后退、左转 90°、home 均完成，review 选 JEV | Primary 在本样本保全目标且人工续行有效；Classic 的 SATISFIED 是相对截断计划，不是完整请求成功。两次起点不同，不比较运动效率 |
| M15 / M19 / M02 组合变体：`go back 0.2 meter, then look ground side, after that set pose home` | 未测 | `61914358b019`：完整三任务；AUTO 前置失败，用户 set_auto 后重查并后退一次；Perception 的 ground side 返回 direction=unclear、analysis=image_describe；等待后用户取消，未执行 home | AUTO 修复续行、取消通过；观察任务未完成。不能归因于拍照失败，问题发生在视角解析前 |
| M11 / M15 变体：`go back 0.2 meter, then search bottle after that set pose home` | 未测 | `a976de7e3d30`：后退完成；同组 9/9 图搜索 bottle 为 NOT_FOUND；task_outcome_review=continue_independent，然后 Arm home；DONE/PARTIAL | 没找到物体不是 skill error；独立后续任务正确执行，不假报找到瓶子。不等同于 M12 条件为真/假测试 |
| M24 拼写变体：`find red suitecase` | 未测 | `2f32bd71ee7e`：JEV 得到 composite、suitcase、red；编译器因原文不精确匹配仅生成 color_red，扫描 9/9 图；DONE/PARTIAL | 模型已理解，但代码交接和 Ask Human 选项丢失，不能说是 JEV 没识别 suitcase，也不能说目标不存在 |
| M24 组合对象变体：`find red suitcase` | 未测 | `d03333b8e919`：一项同实例 composite 检查，共享 9/9 图；运行报 FOUND/SATISFIED、6 个 composite_match | 有 3 对同图红色＋suitcase 有效候选，同时混入 2 个错误类别和 1 个跨图匹配。调用/分解已通，但证据污染，暂不计为干净的视觉验收 |
| M27：`set pose see_front` | `2610fe9750c1`：静止检查 + set_pose，SATISFIED | `c0897f8a2e30`：parse/bind/review/arm_task_review，静止检查 + set_pose，SATISFIED | 两种模式的任务调度均完成；只证明本次 sim Skill 反馈，不证明实机姿态精度 |

**M15 消歧的实际时间线（JST）**：19:21:26.603 接收，19:21:26.800 Ask Human，19:21:52.013 确认 Arm，然后绑定、审查并执行；19:22:18.226 完成。总计 51.623s，其中约 25.213s 为人工等待；4 次 trace API 合计约 854.384ms，不把人工等待算成 API 延迟。确认之前无 skill_started。主解释 navigate_to 的 choice probability=0.67，Arm=0.20；替代解释问题明确选 Arm，因此不需要两者概率都很高才允许澄清。

### 7.3 新暴露的问题（本轮审计未修改生产代码）

| 优先级 / 问题 | 实际证据及定位 | 修复方向 / 验收要求 |
| --- | --- | --- |
| P1：组合检测不限制物体类别 | `_match_composite_detections` 把所有 object 与 color 配对，没检查对象是否 matched_query。red suitcase 真实结果中包含 object_label=bus、truck；离线“红色 + bus”阴性样本同样被接受 | 组合证据必须满足目标类别、颜色、置信度等条件，不允许其他类别证明 suitcase。新增只有 bus/truck 的负例；不得靠 JEV review 掩盖 Skill 错误 |
| P1：组合检测跨图片配对 | 同一函数没比较 image_path；真实结果有第 6 个重复/跨图匹配。离线把红色放 a.jpg、suitcase 放 b.jpg，相同像素框也被接受 | 先按同 observation/image 关联，再判断空间重叠；保留双方证据引用。跨图不能证明同一实例 |
| P2：JEV 的纠正结果进入了编译器，但建议未到用户 | `perception_plan.compile_checks` 对 suitecase→suitcase 产生 proposal 和 gaps；`PerceptionAgent._refine_detection` 在还有 color_red 可执行时返回计划，丢掉 choices。回放得到 proposal=1、surfaced_choices=[] | 保留“建议目标解释/替换”的人工确认入口，确认后同 Task 继续；不要单纯追加 suitecase 正则，也不能默默将 box 改为 suitcase |
| P2：垂直观察能力没进入问题结构 | `perception_view_parse` 只有 left/right/front 等水平方向，没有 ground/upper 或 band；底层 `_direction_camera_pose` 已有 ground=80°/upper=-20°。本次选择 unclear/image_describe | 从配置暴露水平/垂直视角与采图/解释目的；相机能力表和问题选项一致。仅“看地面”不应必然需要未来的 image_describe API |
| P2：替代解释问题可能返回主解释本身 | 6 次 command_parse 中有 3 次主类型和 alternative_type 同为 set_arm_pose：ground-side 组合、bottle 组合、see_front。独立问题看不到前一问输出；代码忽略同值，前两个也未触发近分数兜底 | 不表示三条命令都必须问人，而是当前“另一解释”答案无效。结构校验须区分 none 与重复主类型；可在需要时用已选类型约束后续歧义审查，避免声称消歧已完全稳定 |
| P2：Classic 截断后仍显示整体成功 | 同一 M15 变体只保留第一任务，report SATISFIED。这是用户已知的 Classic 范围限制，但会污染评测 | 开发评价必须按原请求核对目标覆盖，不能按内部状态算成功；本次不要求 Classic 达到 Primary 的语言覆盖 |

相关源文件：`src/robot_skills/robot_skills/executors/perception.py`、`src/robot_agent/robot_agent/agents/perception_plan.py`、`src/robot_agent/robot_agent/agents/perception_agent.py`、`src/robot_agent/robot_agent/command_grounding.py`。

图像复核：已查看 `debug-output/perception/d03333b8e919:t1:obs-e2e5fd6b0fac/annotated_observation_003_2976851a329b.jpg`。可见红色物体，检测器标签为 suitcase；未核对 USD 对象类别，不能把检测器标签当真实标注。该保存图为 **640×480**，不是 1280×720；仅说明本次落盘输入，未据此认定启动环境变量未生效或重新归因于分辨率不足。

### 7.4 统计与本次补测

| 指标 | 10/13 本 session |
| --- | --- |
| Mission / 模式 | 8：Primary 6，Classic 2；无 Shadow |
| 系统上报 outcome | SATISFIED 5、PARTIAL 2、CANCELLED 1。Primary 为 3/2/1；Classic 为 2/0/0。含 Classic 截断和视觉污染，不能换算为用户目标成功率 |
| 真实 JEV 调用 | 30，provider 全 live，均有实际响应；6 次 parse、6 次 bind、6 次 review、其余 12 次领域/监督。启动 probe 不在这些 Mission 调用中 |
| 监督 | step_supervision 仅 1 次：AUTO 前置失败；task_outcome_review 1 次：bottle NOT_FOUND 后决定独立 home 继续。正常动作没有逐步重复监督 |
| API 延迟 | 原始日志中位 173.351ms，范围 146.639–243.984ms；不是任务耗时，也不是整个冷启动耗时 |
| 用量 | 实际响应 usage 合计 input 68,640、output 16,013 tokens；trace 输入费用估算 $0.00288288。输出成本字段为 0 不表示服务商免费，非完整账单 |
| 本次新增动作 / API | 审核期间 0 次；保留用户环境，不重启、不改变控制模式 |
| 本地专项回放 | 人工确认前无动作、原 Mission 三任务完成、PARTIAL 后独立归位、取消后无 home 均有断言；错误类别、跨图匹配、丢失纠正选项均已复现。缺陷仍开放，不把“成功复现错误”算功能通过 |
| 既有针对性回归 | 227 passed，20.41s；覆盖 Mission/Domain/Perception 编排和能力契约，未调用真实 API |

现有针对性回归的结果见 `debug-output/validation/20261013-restart-review/regression.xml`；此类回归不能覆盖全部现场新发现，尤其不能用全部既有测试通过否定上述阴性回放。

本 session **没有重跑 M12 的条件真阳性/假阴性两分支，没有 red box、shelf_06 整段导航、恢复动作或 Shadow 同命令对照**。M15 证明了一个真实样本的目标保全与人工续行增益，但不足以证明整体稳定。下一轮优先补组合检测的类别/同图防线、Perception 纠正选项交接及视角参数契约，再做条件正负例；不要先扩充特定句式。

### 7.5 审计后的修复

按用户后续授权，已修复第 7.3 节前五项；Classic 语言覆盖限制保持原样。记录见 [18-evidence-and-clarification-fixes-20261013.md](18-evidence-and-clarification-fixes-20261013.md)。历史 API 回答、任务 outcome 和 audit.json 不重写。
追加证据为 `debug-output/validation/20261013-restart-review/fix-replay.json` 和 `fix-regression.xml`。272 项离线回归通过；原始检测回放去掉两个错误类别和一个跨图配对，保留三个匹配。类型重复改为有界追问，视角问题改用配置中的九视角；新问题的真实 API 测试未执行，不计新一轮现场成功率。
