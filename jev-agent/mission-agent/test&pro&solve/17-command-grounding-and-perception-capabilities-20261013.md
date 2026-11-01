# 2026-10-13：Mission 原文绑定与 Perception 能力契约

## 范围

修复分层设计，不追加 next/after-that 等连接词正则。未启动新的机器人动作，未重启用户的 Agent；没有替换视觉模型。Recovery、控制器、运动安全门不在本轮修改范围。

## 1. 发现与修改

| 原问题 | 系统性修改 | 边界 |
| --- | --- | --- |
| JEV 已选出三项 Task，却被旧代码切句失败否决 | 独立 command_bind 从原文 token 索引选择每项完整片段，代码校验、抽取数值 | 最多三项、96 个词法 token；不是无限长或通用程序语言 |
| 同次问题混合全部类型参数，产生大量 not_applicable | 先结构分类，再按已选类型提问；不问 Mission 的 arm pose 枚举 | 单次结构请求对三个可能位置做有界并行分类，不存在的位置返回 absent |
| parse_confidence 被误作 0..1 概率 | 新解析不问这个多档 score；保留 API 实际 confidence/probabilities，不自造 live 分数 | 结构/语义许可分开，不凭一个置信度放行 |
| 全局 dependency_policy 表达不了各项关系 | 每项 task_ordinal、depends_on、dependency_policy、explicit_after_attempt；执行时核对指定来源 | 有序不等于成功依赖；失败不会被无关 Task 的成功覆盖 |
| 条件句被特定 if/else 正则限制 | JEV 选 if_found_else_hold，编译搜索→有条件相对运动 | PARTIAL/矛盾/旧证据不会当成 FOUND 或完整 NOT_FOUND |
| review 只投票选来源可能批准错误图 | 审查两份具体图，各自 faithful/incorrect/unclear，再选来源；选中图必须 faithful 且参数/类型合法 | 三问同批互相独立，不假设模型看过其他问题答案 |
| YAML 的 box 映射声称超过模型能力 | schema v2 分开 closed-set 类别、别名、已知缺口、HSV/OCR 契约 | 当前仍不能可靠检测 box；不会自动改查 suitcase |
| API 失败沿用默认许可或显示本地答案 | 已配置 API 的解析/领域审查失败转澄清；错误 trace 结果为空并标 api_error | 未配置 API 的明确 local 模式保留；确定性安全/恢复逻辑不依赖 API |

## 2. 新问题与信息流

| 阶段 | 信息与问题 | 输出用途 |
| --- | --- | --- |
| command_parse | 原文；Task catalog 的动作/owner 定义；command_shape、task_count、task_1..3_type、各项 alternative_type | 识别目标数及类型，有歧义提供明确选择；不访问 Classic/健康快照 |
| command_bind | 原文、带 offset 的 token、已选类型；每个 Task 的 start/end、目标短语 start/end、dependency；仅运动问方向和 numeric/default policy | 代码拼装 intent。跨片段、重叠、未知/多义数值、互相冲突的方向拒绝；数值来自原文，默认值来自 Task catalog |
| plan_review | 原文、两份 TaskGraph、代码结构校验结果；jev_graph_faithful、classic_graph_faithful、plan_source_decision | 只有完整正确的图可以发布；无图/只识别第一项的 Classic 不能补成通过 |
| domain refine | 当前 Task 的完整原文、目标和领域能力/规则 | Arm 验证姿态；Navigation 解析地图；Perception 分析模型、颜色/文字以及相机方向 |
| task_outcome_review / step_supervision | 当前任务结果、正确的依赖来源、当前异常和必要证据 | 正常结果不重复监督；未满足目标不等于执行错误；独立后续任务仍需通过自己的安全门 |

slot 就是结构化参数，例如 distance_m；它不是另一种模型。新请求用参数来源选择和 source/goal span 表达，不再为每个 Task 问所有类型的 slot_resolution_action。

条件图示例：search_object(red box) + move_relative(back,0.2,depends_on=[1],run_if=previous_search_found)，else_action=stay_still。完整 FOUND 才运动；完整 NOT_FOUND 跳过运动；只有 red 可检测时为 PARTIAL，不能冒充条件为假。

每次模型调用都记录独立 stage、延迟与输入费用估算。raw questions/criteria 和 response 仍在 jev_calls.jsonl；界面是摘要，不是完整请求。绑定位置字段改为 source_words，避免旧脱敏规则将 tokens 当密钥隐藏；未发 API 的 local 路径不再记录估算 API tokens/费用，API 错误不伪装成 local 推断答案。

## 3. 本轮真实 API，只读 12 次

原始证据不重复复制：`debug-output/validation/grounding-20261013/jev_calls.jsonl`、`results.json`。无额外 probe、无图像/地图发送、无机器人动作。

**这批调用发生在最后一轮措辞/plan_review 精简之前，不代表最终版本全部通过。**

| 用例与完整命令 | 调用 | 实际结果 | 结论 |
| --- | --- | --- | --- |
| M15：go back 0.2 meter, next turn left, after that set pose home | parse + bind + review | 三项数量/原文/0.2m/左转均绑定；第三项选 navigate_to(home)；review 返回 ask_human | 用户确认末项确有歧义，拦截合理；当时缺少 Arm / Navigation 明确选项。不是旧连接词切分问题 |
| M12：if you find a red box, go back 20cm; otherwise stay still | parse + bind + review | search_object(red box) → back 0.2m，成功依赖和静止 false 分支完整；review 选 JEV | 解析/审查通过，不证明实物 red box 可检测 |
| M11：find red box, then set pose home regardless of whether it is found | parse + bind | 正确两类 Task、home 和原文；home 无成功依赖。第一项也被多余标为 cleanup | 主要目标正确，依赖标注仍有问题；未调用完整 review |
| 等价表达：move backward 20 cm; subsequently rotate left; finally return the arm to home | parse + bind | 三类 Task/参数正确，但添加了 success_1/success_2 成功依赖 | 部分通过；顺序与成功依赖仍混淆，未调用完整 review |
| Perception：red box | parse + review | composite、box、red、unit_count=2；只执行 color_red，保留 box 缺口 | 能力分层通过，不能据此声称找到了箱子 |

12 次调用延迟约 157–241 ms/次。输入合计 37,218 tokens（trace 统计），费用估算约 $0.001563；不是账单。增加绑定阶段的好处是显式追踪原文归属，不能宣称零成本或比旧单次请求更低延迟。

复盘后依赖选项区分先后与结果依赖，第一项不提供 after_attempt；review 明确拒绝虚构成功依赖。最终 review 精简为三问，并移除重复的图/plan/slot 输入；binding 不重复在每个选项打印 token offset，offset 已在 state 提供。曾将命名姿态强制解释为 Arm 的措辞已按用户纠正撤回：省略执行器且导航解释也合理时必须澄清，不能强选 Arm。

上述最后修改需新一轮真实 API 复验；不得把离线模拟通过当成 JEV 理解已稳定。已申请额外最多六次只读验证，未获回复前不发送。

## 4. 离线回归与模型事实

- 回归覆盖：原文归属、边界错误、单位、方向矛盾、API 缺项/失联、来源投票与图判定冲突、非相邻依赖、M12 的 FOUND/NOT_FOUND/PARTIAL、M11 独立归位、取消/人工恢复/Recovery 转交、领域计划、同图组合检测与证据。
- 实际加载本地 `models/perception/yolo11s.pt`，`model.names` 与 v2 YAML 的 80 类集合一致，box 不在其中。加载环境有 Matplotlib Axes3D 包版本警告，不影响本次类表核对；没有执行图像推理。
- 新增核心测试：test_command_grounding.py、test_perception_capabilities.py；扩展 test_domain_tasks.py。
- 最终针对性回归：**219 passed，19.52 s**，JUnit 证据 `debug-output/validation/grounding-20261013/regression.xml`。这些是离线/模拟执行器回归，不计入真实 API 成功率。
- `colcon build --packages-select robot_skills robot_agent --symlink-install`：两包成功。没有重启用户 Agent，也没有执行机器人动作。

## 5. 已知边界与下一轮验收

1. 真实 JEV 仍可能选错类型/依赖，review 必須拒绝而不是让后续技能猜测。重点重复 M15、同义表达和不含明确 arm 的命名姿态请求。
2. 先用 red bottle 等受支持类别验证条件运动；red box 用于验证 PARTIAL/能力不足处理。增加 box 能力需要模型，不是 YAML 同义词。
3. Perception 仍用有界 clause 候选和词表；任意复杂括号、混合 AND/OR、多语言 OCR、相对云台微调、image_describe API 尚不支持完整执行。
4. 新绑定支持方向+单一距离及相对角度，不宣称已支持 dx/dy 复合位移或任意自然语言数词；缺参/多值拒绝，不默默采用第一个数字。
5. 本轮没有 sim 运动验收。YAML schema 已更新，继续测试前须在 Quick Start Stop Agent stack 后重新启动，避免旧 Python 代码混用新 YAML；Isaac Sim 不需要因本轮修改而重启。

## 6. 用户纠正后的类型消歧

新增每个 Task 的 alternative_type 问题，语义替代解释或接近的实际 choice 概率触发澄清。只展示 catalog 中合法的候选，不自行生成分数，也不凭目标词强选动作。Ask Human 保留原 Mission，提供 Arm / Navigation 明确动作与 Cancel all；选择后才绑定参数并审查。选导航但地图没有 home 时仍须确认目标，不能虚构坐标。

人工确认以 human_confirmed_task_types 进入绑定与审查，代码阻止 Classic 候选反转该选择。测试发现并修复了解析等待返回后把 CANCELLED 覆盖为 NEEDS_HUMAN 的竞态。回归覆盖两种选择、等待期间不执行、旧 wait_id 拒绝、Cancel all 后不再绑定/执行、原序列保留和人工选择不可被 review 反转。

本次未新增真实 API 调用或机器人动作；原 12 次调用是历史证据，不能作为新增问题的在线验证。新问题措辞仍需后续真实 JEV 测试。

追加验证：同组 Mission / Domain / Perception 针对性离线回归 **227 passed，20.10 s**；`colcon build --packages-select robot_agent --symlink-install` 成功，`git diff --check` 无问题。未重启运行中的 Agent。
