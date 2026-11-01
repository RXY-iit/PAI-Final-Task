# Mission / JEV 命令测试

更新：2026-10-12，按 Mission / Navigation / Perception / Arm 的实际分工验收。
Classic 执行证据在 `14-domain-agent-suite-20261012.md`，真实 Primary 对照和失败分流在 `16-primary-suite-20261012.md`；本文件定义期望，不把期望写成已通过。

返回 [README](README.md)。每行都是预定义 case，不是已通过记录。除非专测权限，基础运动使用 AUTO、空旷区域、已确认的有效反馈；Arm mock 成功只证明调度/仿真接口，不证明实机动作成功。

## 1. 执行前准备

Dashboard 的 `Mission Tests` 快捷按钮包含 M01–M17、M23–M30 及命令变体，点击即发送。M14 的 Cancel 仍由测试者按时机操作；M18–M22、M31–M32 含环境操作/故障注入，不做成单次发送按钮。命令源为 `src/robot_agent/robot_agent/mission_test_commands.py`，修改时同时核对本文预期。

1. 确认 Full 对应 `nav2`，Arm Only 对应 `arm_test`；确认 map 与 Isaac 场景匹配。
2. 记录 safety、robot_mode、相机数据、轮里程计、GICP score/pose/age、静止漂移。用 `ロボット状態` 保存一次 Skill 侧数据，同时保存 Health 页。
3. 在“已知位置”列表确认 `shelf_06` 存在，地图导航范围、目标是否占用可见。不存在就记为场景前提缺失，不算解析失败。
4. JEV 用例先确认真实 API probe 成功，调用记录 provider=live；若 fallback，单列 LOCAL_FALLBACK，不计入 Live 对照。
5. 结束前一个 mission/恢复流程，再切换模式。记录当前 warning 忽视规则；新 Dashboard session 应清零，若旧规则残留记为缺陷。
6. 每个命令分别保存 parse、plan、skill results、最终状态。看到 plan 正确不等于动作正确；看到 DONE 也要核对实际反馈。

## 2. 单任务基线

M01–M07 先 Classic，再 JEV Primary；Shadow 用于检查建议差异，不算第二次 JEV 执行。

| ID | command / 操作 | 正确语义与检查点 | 复位 |
|---|---|---|---|
| M01 | `ロボット状態`；另一次 `list known targets` | 只读；不发布运动。Classic 与 JEV 的支持集合不同也要记录，不得静默替换指令 | 无 |
| M02 | `左を見て`，先 MANUAL 再 AUTO | 等待静止 → 相机向左 → 新图像 → 相机回导航姿态；不旋转底盘、不依赖全局定位。两种模式各独立一次 | 相机回正 |
| M03 | `set pose home` | Arm pose 任务，不能误解为底盘回地图 home；检查 gateway/关节反馈 | 保持 home |
| M04 | `go back 20cm` | 单个后退 0.2 m；同时记录目标、wheel odom 位移、实际容差、耗时 | 恢复固定测试起点 |
| M05 | `turn left`；另一次 `turn around` | 默认 +90° / 180°；记录带方向的累积角度、误差和 skill 容差。170° 附近被容差判完成时，不能描述成精确转过 180° | 恢复测试朝向 |
| M06 | `前進少し` | 明确记录“小幅”当前 policy 值，预期 0.2 m；重复 3 次但每次恢复同一起点，不连续累积距离作对照 | 恢复起点 |
| M07 | `go to shelf_06` | 定位和路线检查通过才发 Nav2 goal；按目标 YAML 的 approach_policy 接近，再按 visual_arrival_confirm 委派 Perception。OCR 06/6 为一个身份目标，可复用一批图像检查。区分 strict/task_tolerance/degraded 到达与视觉确认，不把未确认标签算整体成功 | 同一已知 pose、相机姿态；路线先检查 |

M05 的容差在执行前从 skill 参数/配置抄到记录中；不得为使结果通过而事后改容差。手动驾驶回起点后重新取得状态快照，不能默认状态与上一轮相同。

## 3. 分解、参数、子任务依赖

| ID | command | 预期任务图 / 失败策略 | 对照重点 |
|---|---|---|---|
| M08 | `go back a little, 左を見て` | 后退 0.2 m → 向左观察；保留两个子目标 | 10 月 7 日 Classic 候选曾漏后退；比较两份 plan，再比较真实执行 |
| M09 | `go back 20cm, then set pose home` | 后退 0.2 m → Arm home；不是 20 m，也不是地图 home | 与 `go back 0.2 m, then set pose home` 做等价测试 |
| M10 | `go back a littlt and turn left` | 明确理解时为后退小幅 → 左转；无法确定时澄清可接受，但漏一个子目标后声称完成不接受 | 拼写扰动，仅改一个词；不混入系统故障 |
| M11 | `find red box, then set pose home regardless of whether it is found` | 明确“搜索尝试后回 home”；完整检测阴性为 NOT_FOUND，box 类别不支持为 PARTIAL，二者不混淆。若手臂前置条件满足则继续 home；整体部分完成不能声称找到目标 | 目标不存在/语义能力缺口不应重启相机或定位 |
| M12 | `if you find a red box, go back 20cm; otherwise stay still` | 有界条件句已实现：完整 FOUND 才后退，完整 NOT_FOUND 跳过运动；PARTIAL/检测失败等待澄清。red box 受 box 类别缺失限制，追加 bottle 作为完整能力链对照 | 不能按普通 sequence 忽略条件；未知不是 false；条件不成立标 SKIPPED，不能伪称运动成功 |
| M13 | `check red object and go back a little` | 失败依赖未说清；本轮预定策略为搜索未找到后询问是否后退，不能静默自动执行运动 | 先冻结策略，勿把任何猜测都算对；测试继续/取消两种用户选择 |
| M14 | `go back 20cm, then turn left`；后退运行中点 Cancel | 停止当前任务、不执行左转、无迟到重试；后续恢复完成也不得复活已取消 mission | 分别在解析等待、skill 运行、recovery 等待时取消 |
| M15 | `go back 20cm, then turn left, then set pose home` | 三个 task，内部 skill 数可以大于三；顺序与参数完整 | task count 与 skill count 不能混淆 |
| M16 | `go back 20cm, then turn left, then set pose home, then look right` | 当前三 task 上限：明确澄清/拒绝；不得执行截断的前三步后报 DONE | 边界、不完整命令：另测 `go back ...` |
| M17 | `今天的天气怎么样？`；另一次 `go to shelf_9999` | 前者不发布机器人动作；后者报告未知目标/澄清，不编造地图坐标 | 非指令与未知实体分开 |

M11–M13 在受控场景里确认无 red box，再提交；模型没检出“实际存在的目标”是另一类感知失败，不能混为目标不存在。若当前解析器不支持这些表达，标为能力缺口/预期澄清，不能无限重试同一命令。

## 4. 跨模块与故障边界

| ID | 前提与操作 | 预期 |
|---|---|---|
| M18 | arm_test 无全局定位；依次测 `go back 20cm`、`turn left`、`左を見て` | 只检查各自所需反馈和安全/权限；不能因缺 Nav2/GICP 全部阻断。相对动作仍需本地里程计 |
| M19 | MANUAL + 静止，`左を見て`；另一次 MANUAL + `go back 20cm` | 观察不应要求 AUTO；后退需明确权限条件。不要将 MANUAL 视为全系统损坏 |
| M20 | 有效拍照后断开该 camera publisher，再请求 `左を見て` | 不可把旧缓存图当作新观察；记录图像时间、拍摄步骤边界及超时 | 
| M21 | JEV 请求中取消；或在独立测试启动配置中指定不可达的本地 API 端口 | 不假 live、不静默把 fallback 算 JEV；不得在取消后因迟到结果执行。恢复 endpoint 后重新 probe。不要通过断开整个网络制造多重故障 |
| M22 | A 任务取消后立即提交 B=`ロボット状態` | A 的迟到 skill/recovery/JEV 消息不能覆盖 B、恢复 A 或混入 B 的 report；用 mission_id/request_id核对 |

## 5. 解析专项与现场执行分开

- 首先固定 10–20 条原始命令，对照预先标注的 task graph/slots；用离线 harness/dry-run 检查，不需要每条都移动机器人。先检查现有 dry-run 入口支持的参数，不能假设它已支持 JEV 录制回放。
- 自动化回归覆盖解析/委派/取消/依赖；`tools/sim/run_mission_case.py` 通过真实 Dashboard 执行单条指定命令，记录摘要及 mission_id，完整证据复用 session/mission report。它不伪造检测结果、不绕过控制门。启动前检查场景和其他 Mission，超时只取消本次任务。
- 每条标准命令生成少量“单变化”表达：逗号/then、英文/日文混合、单位、拼写。另留一组未参与修复的新表达，用来检查是否只是记住了原句。
- 解析正确但 skill 超时，记执行失败；解析把距离弄错，记参数失败；不要全部记“JEV failed”。

## 6. 领域 Agent 契约用例

| ID | 命令 / 操作 | 责任链和验收 |
| --- | --- | --- |
| M12b | `if you find bottle, go back 20cm; otherwise stay still` | 直接支持的类别对照：完整 FOUND 才运行一个相对运动；NOT_FOUND 显示 CONDITION_FALSE，PARTIAL/错误不可冒充阴性。条件 true/false 都需覆盖，场景不能提供的分支由模拟结果回归补充并明确标记 |
| M23 | `find 06 letter` | Mission 只委派搜索；Perception 识别 OCR、解析字面 token；一个 search_at_place、共享图像，文字与物体类型不混淆 |
| M24 | `find red bottle` | 同一个实例满足颜色与 bottle；两个要求不等于两次扫描；分离的红色物体和瓶子不能拼成一个成功 |
| M25 | `find cup or bottle` | OR 两个替代检测，共享采集；不能错成必须同时出现；没有目标也应返回图像 |
| M26 | `check left side of robot if anything dangerous?` | Mission 保留环境理解目标，Perception 判 image_describe；当前 API 未配置时明确 UNSUPPORTED，不把拍照或未检出当成安全证明 |
| M27 | `set pose see_front`，随后 `set pose home` | Arm 的 named pose、反馈、取消和恢复到 home；不动底盘。实际 gateway/mock 情况必须记录 |
| M28 | `set pose unknown_pose` | 未注册 pose 在执行前澄清，不能下发未知关节目标，不能报已完成 |
| M29 | `touch blue dot` | 当前自主 touch 完成契约未验收，Arm 返回 UNSUPPORTED；不能错误宣称完成、误用底盘相机坐标 |
| M30 | `grasp bottle` | 当前自主 grasp 未实现，Arm 返回明确能力边界；不得靠 stub/命令入队就报告抓取成功 |
| M31 | 搜索展开/图像采集期间 Cancel all，随后 M01 | 取消当前组合 Skill；相机停止/归位按可实现策略，后续检测不得复活旧 Mission；新 M01 report 不混入旧任务 |
| M32 | 模拟 API 空回复、错误选项、缺少某个 check 的回答 | 离线故障回归：Primary 对该检查阻塞/记录缺口；Shadow 不采纳也不额外动作；不得将缺失答案用本地默认掩盖成 live 成功 |

### 失败分流隔离回放

`tools/sim/replay_primary_failure_cases.py`：真实 JEV + 合成执行器/Recovery 回复，**不发布 ROS 动作，不算真实 PB 恢复成功**。需显式配置 API，原始请求脱敏保存。

| ID | 注入 / 预期 |
| --- | --- |
| F01 | 拍照失败，Recovery 中间/最终回复；重试拍照，而不是跳到后续步骤；再 Arm home |
| F02 | 相对运动部分完成、odom 过期；监督识别 direct drive/odom，不错归 Nav2/GICP；恢复后不自动重放完整距离 |
| F03 | Recovery 等待中取消，之后迟到 RESOLVED 不恢复任务 |
| F04 | 拍照故障持续；一次 Recovery 后仍失败，停止自动循环并求助人工 |
| F05 | MANUAL 阻塞；提供 AUTO 人工选项，不调用无关 Recovery、不直接移动 |

另外由离线回归覆盖：人工重试当前步、继续独立目标、取消三条分支均保留 mission_id；不能跳过失败前置执行同 Task 后续动作；缺失 live 监督 action 必须求助而非冒用本地答案。

## 7. 本轮执行与统计约束

- 先 M01/M01b、能力边界和领域解析，再相机/Arm、短距离运动，最后经检查的 M07；移动不超过单次 3 m，相对运动前后配对回位。API 和长扫描顺序执行，避免人为制造负载混淆。
- 常用命令至少比较 Classic / Primary / Shadow。Shadow 仅观察 JEV；不能把一次 Classic 执行算成两套执行对照。模式切换必须等 acknowledgement。
- M04/M06、M02/M03、取消和条件核心路径重复；若未达到 3 次同起点或未达到某模式，覆盖表写实，不能宣称完成全矩阵。
- 每次记录执行模式、任务 ID、父子关系、选定计划、每个 skill 的结果、当前 human wait、检测/到达分级、JEV live/fallback、耗时、控制前后状态及证据路径。实测、模拟结果回归、只读 API 回放分开。
- 发现确定性缺陷：固定失败证据 → 最小修正 → 针对性回归 → 同样现场用例重跑。待讨论项不偷偷变更规则；缺工具/缺场景/模型能力不足标注原因。
