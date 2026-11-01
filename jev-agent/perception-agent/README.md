# Perception Agent：领域解析、能力审查与执行监督

更新：2026-10-13。本文件覆盖旧设计；历史实测保留在 [runtime-question-audit-20261012.md](runtime-question-audit-20261012.md)，旧 stage 名称不代表当前实现。

## 1. 职责与概念

Perception 有两类职责：调用识别能力检查对象/文字/颜色；控制相机朝向并取得图像。后续图像描述、环境理解也是其职责，但 **image_describe API 本轮尚未接入**。

Mission 理解请求、安排 Task 和依赖；Perception 深化单个感知 Task，决定需要哪些证据、哪些检查可执行、如何采集及反馈。不自己移动底盘，也不通过视觉模型给底盘安全门放行。

| 概念 | 示例 | 责任 |
| --- | --- | --- |
| Mission | 看左边，找红色箱子，然后手臂归位 | Mission Agent |
| 感知 Task | search_object(red box)、observe_scene(left) | Perception Agent |
| 语义检查项 requirement | red、box、两者属于同一实例 | command_parse |
| Task Step / Robot Skill 调用 | search_at_place、set_camera_pose、capture_observation | Agent 规划、执行器执行 |
| 内部子调用 | 九个视角的静止→到位→拍摄，以及多次 detect_known_object | 组合 Skill，留证据 |

解析中的数量是**语义要求数量**，不是九个相机角度，也不是新增九个 Mission Task。red bottle 有两个要求，但编译成一个同实例组合检查；cup or bottle 是两个替代检查。相机采集有物理先后，识别检查重完整性。

## 2. 当前能力

| 类型 | 当前实现 | 限制 |
| --- | --- | --- |
| object | YOLO11s COCO 类别 | box 不是 COCO 直接类别；suitcase 不等价于 box |
| color | OpenCV HSV | 仅配置颜色；light blue 等未校准修饰不能默默省略 |
| ocr | Tesseract / EasyOCR | 部署和加载执行时验证；全部后端失败不是 NOT_FOUND |
| image_describe | 未来视觉语言 API | 当前 UNSUPPORTED，不能伪装成已经分析图片 |

这是三类识别能力，**不是三个独立 YOLO 模型**。perception_models.yaml 的 yolo11s/yolo11s_ocr 是两个 provider，stub 是占位。HSV/OCR 不依赖 YOLO 权重；此前一律检查 YOLO 权重的能力判断已修正。

权威配置：
- [perception_models.yaml](../../../src/robot_skills/config/perception_models.yaml)：provider / OCR 部署。
- [perception_targets.yaml](../../../src/robot_skills/config/perception_targets.yaml)：schema v2，实际类表、等价别名、能力缺口、HSV/OCR 契约。
- [perception_capabilities.py](../../../src/robot_skills/robot_skills/perception_capabilities.py)：统一加载与校验，Agent 和执行器共用。
- [task_catalog.yaml](../../../src/robot_agent/config/task_catalog.yaml)：Task 和 owner。

## 3. command_parse

2026-10-12 回归修正：委派检测的 `raw_text` 使用当前 `object_query`，不把父 Mission 的 `go to shelf_06` 当作感知指令。地图传来的 `recognition_contract` 是权威的目标属性，不是 Classic 的候选答案。明确 OCR 合约只允许 OCR/unknown；API 返回物体类型等越界答案必须澄清，不能静默改写。

`shelf_label_06` 默认识别货架编号；地图显式配置的 `06/6` 是同一个目标的可接受文字，不是两个物体。`06 letter`、`letter 06` 等完整文字表达可编译为 `text_06`。其他修饰如 `red label 06` 不能丢掉 red 后直接声称完全满足。普通 `text_06` 不自动替换成 6；只有显式合约给出的替代写法可以这样处理。

`perception_plan_review` 审查各个 check 的语义支持情况，不把一个不支持的 box 传播成 color_red 也不可执行。provider 是 pipeline：`yolo11s` 内部的 HSV 可以检查颜色，颜色不是 YOLO 类别；`yolo11s_ocr` 负责文字。部署可用性与新鲜图像仍由 capability/skill 检查，不因语义审查通过而跳过。PARTIAL 不等于目标成立，不能放行依赖完整目标的运动。

最新 plan_review 只问每个 `check_i_compatible = supported / unsupported / unclear`。去掉与逐项支持判断重复的泛化 `decision` 问题：真实 API 曾对全部检查给 supported，却同时返回 ask_human，无法形成可执行的调查原因。现在由明确答案汇总：保留 supported 检查，其余转 gap；至少一项支持则执行这些检查，零项支持则澄清。缺失/非法答案视为 unclear，不用默认值覆盖真实 API 的缺失。输入语义/分组歧义已在 command_parse 拦截；无可执行能力时不额外发送空 plan_review 请求。

stage：`perception_command_parse`。输入当前 Task 的原文片段和语义选项，不发送整份健康快照。Mission 必须保留解释目的、方向和修饰，不能提前降为单纯拍照。

代码划分显式 and/or/逗号 clause，提供有界词表；JEV 同一次请求回答：

| 问题 | choice | 作用 |
| --- | --- | --- |
| coverage | complete / unclear | source clauses 是否覆盖完整需求 |
| relation | all / any / unclear | AND 还是明确 OR；不能拆丢颜色与对象关系 |
| unit_count | 1..12 / unclear | 语义要求数，复合颜色+对象计二；和类型交叉校验 |
| kind_i | ocr / object / color / composite / image_describe / unknown | clause 类型 |
| object_i | 实际类别与配置名称 / none | 请求对象；选择不证明可用或等价 |
| color_i | 配置颜色 / none | 请求颜色，不生成 HSV 数值 |

object_i/color_i 均另有 unknown：未列入的名词、未校准色调不能靠“最接近”的可用类别替换。box 可以选为语义目标，但能力审查仍认定不支持。

Classic 用结构化确定性初值；Primary 采用合法 JEV 选择；Shadow 记录但不改变确定性执行。未配置 API 是明确 local；已配置 API 的无响应、损坏、缺项不偷偷补成成功。

数字/文字由代码从原文抽取，不是 JEV 自由生成 JSON。替代词非原文/批准别名时须人工确认。最多六个 clause；混合 AND/OR、数量不一致、边界不清需澄清。本版不是无限制生成式解析器，JEV 不能发明动态代码或未注册动作。

`shelf_label_06` 是 OCR 06；现有货架标签 policy 接受 06/6，普通 text_06 保持严格文字匹配。`text_06 or 6` 可作为两项显式替代检查。

## 4. plan_review

stage：`perception_plan_review`。输入 requirements、checks、缺口、替代提案、聚合关系和采集范围，不发送无关定位/CPU 状态。

代码先作能力校验。JEV 回答 `check_i_compatible`（supported / unsupported / unclear），逐项检查类型和能力是否相符；调度逻辑据此汇总 execute_supported_checks / ask_human_no_supported_check，不另问总开关。不通过的项移入缺口；不能把缺失模型变成可用：

1. color 必须有 HSV 配置且词语吻合；未知修饰保留为缺口。
2. object 必须匹配真实类表或批准的直接映射；fallback 不算直接类别。
3. OCR 需要明确 token；未解决上下文不能省略。
4. 权重/依赖明确不可用时移出检查；加载未知留给执行验证，不宣称 ready。
5. 还有可检测部分就执行，返回 PARTIAL + 未完成条件，不把能力不足当执行错误。
6. 完全不可执行时 ask human；替代候选标明“替换 X 为 Y，会改变目标”。回答绑定 mission_id + wait_id，只重新审查当前 Task，不重放此前动作。

10/13 修复：有替代/纠正提案时，即使颜色部分已可执行，也先保留 Ask Human 选项。用户可确认对象解释，或选 **Detect supported parts only**。后者只执行已支持部分，缺口保留、结果仍可为 PARTIAL；不会因没有看到确认选项而丢掉物体要求。没有提案的 red box 能力缺口仍按原策略检测支持部分，不强行制造替代项。
确认未知单词到已注册类别（例如 suitecase→suitcase）后，只解除该对象词的歧义；其他色调/属性仍须保留并验证，不新增针对这个拼写的正则或别名。

**不是高置信度就自动替换**。生产配置已移除 box→suitcase 的默认提案。若 JEV 提议一个并非原文或等价别名的受支持类别，只能作为改变目标的人工候选，不可自动执行；无需把所有近似词硬登记为模型能力。

### 能力登记与扩展

| YAML 项目 | 意义与限制 |
| --- | --- |
| detector_classes | 闭集类别。替代误导性的 yolo_open_vocab_classes 名称；本轮实际加载 yolo11s.pt，核对 80 类一致，无 box |
| detector_contract | 使用这些类别的权重/provider 声明；执行器加载时再次核对 model.names，不一致拒绝推理 |
| object_aliases | 可选语义别名或显式大类组合，例如 vehicle→car/truck/bus；每项 classes 必须是实际类别的子集，不能用 car 查询暗中覆盖三类 |
| unsupported_objects | box/bucket/forklift/pallet 等已知缺口的语义词表，不是可执行目标，也不是完整的未知名词黑名单 |
| colors + color_contract | 当前七种 HSV 掩码；不是完整颜色全集。红/橙/黄范围存在重叠，lighting_calibrated=false；light blue 等没有独立校准不能宣称支持 |
| text + text_contract | digit/letter/text 是查询类型，不是物体类别。当前资源为英文/数字、明确 token 匹配；多语言、任意短语、场景解释不作支持承诺 |

`objects` 分支在源码兼容层仍会出现，但新版 YAML 不再维护平行的“物体能力表”。统一 loader 把 object_aliases/unsupported_objects 转成旧消费者的内部字段，并校验越界映射；`yolo_open_vocab_classes` 也仅是内部兼容键。

新增物体类必须先有相应 checkpoint，不能只加 YAML 名字。新增别名不需训练但必须语义等价；新增 HSV 颜色需有阈值和图像测试；新增 OCR 语言需同时部署资源、扩展匹配规则及测试。资源可用性与目标是否在画面里，仍需执行证据，不能由登记表保证。

red box 仅支持 red 时检测红色并保留 box 缺口；是否继续后续 Task 由 Mission 依赖策略判断，不承诺无条件继续运动。

## 5. 一轮图像，多项检查

search_at_place 新增 `checks`、`aggregation`、`unresolved_requirements`，保留旧 query/provider 入口：

```json
{
  "profile": "shelf_label",
  "checks": [
    {"id": "c0", "kind": "object", "query": "cup", "provider": "yolo11s"},
    {"id": "c1", "kind": "object", "query": "bottle", "provider": "yolo11s"}
  ],
  "aggregation": "any",
  "unresolved_requirements": []
}
```

- 默认九张只采一次，checks 共用 Task/attempt evidence_id；方向请求仅采该方向。
- 每个位置仍检查静止、反馈、新帧，保留取消和最终预算。正常扫描沿用 profile 归位；取消/失败不启动新归位。
- 当前推理顺序执行，**没有声称并行或模型输出缓存**。cup/bottle 仍可能重复调用 YOLO，但不重复拍摄；原始输出缓存是后续优化。
- 标注文件包含 query hash，不同检查不覆盖标注图。
- red bottle 要同图同实例关联，不能跨图片拼凑。
- 10/13 执行器修复：两项必须 matched_query、非近似/未批准替代、颜色和对象类别符合当前 plan、置信度/框有效且 image_path 相同。匹配结果保留双方图片路径、框及 overlap_ratio；bus/truck 不能证明 suitcase。
- 聚合全部 check_results，不只返回最后一次。all 全满足，any 至少一项满足；存在缺口时保守 PARTIAL，不用 OR 掩盖未知。

### 定向观察参数

`perception_view_parse` 现在问 profile、view、analysis。view 直接来自 `src/robot_skills/config/camera_scan_profiles.yaml` 的 shelf_label 九视角，不再只列左右前；问题和执行的逻辑 pan/tilt 共用这份配置，实际硬件映射仍由 real/sim_camera_poses.yaml 管理。
ground/down 对应 ground_center，up 对应 upper_center，未指定高度时 middle；组合方向可以选 ground_left 等。未配置的 band/view 不默默回落到中间视角。
只要求看某方向默认属于采图意图；明确描述/危险判断才选择 image_describe。最终含义仍由 JEV 判断，新问题措辞尚待真实 API 验证；相对 pan/tilt 调整仍不冒充固定视角。

## 6. step_supervision

stage：`perception_step_supervision`。Perception 复用共享 JEV 适配器与 Mission 执行策略，不多调一遍监督 API。仅未处理的执行异常调用；正常 FOUND/NOT_FOUND/PARTIAL 不调用。取消优先，不让 JEV 争论取消后是否继续。

| Skill / 子调用 | 本轮加入的问题 |
| --- | --- |
| set_camera_pose | 新鲜反馈是否到位；控制权与静止条件是否满足 |
| capture_observation | 到位后新帧是否可读；是否静止；是否属于当前证据域 |
| detect_known_object | 后端是否完成；能力缺口还是系统故障；未完成属性是否保留；是否同实例 |
| search_at_place | 继承采集+检测问题；异常时按 failed_child 选择契约 |

**之前 catalog 没有逐 Skill 问题**，只有前置条件/失败语义/暴露标记。现在 [skill_catalog.v2.yaml](../../../src/robot_skills/config/skill_catalog.v2.yaml) 的 `supervision.questions` / `evidence_fields` 已覆盖所有注册 Robot Skill，通用监督问题也迁入同一文件。compact_skill 传给监督，API 实际包含对应 noul 问题，证据不足应回答 null。维护方法见 [统一监督目录](../skill-supervision-guide.md)。

输入为当前结果、有限子调用历史、有效视角数、推理状态、缺口和最多三条后端错误摘要，以及继续所需的依赖/安全摘要。安全门、恢复授权独立。系统健康故障可以请求 Recovery；没检出不等于相机需要重启。

仍由 Mission 执行有界重试/恢复/人工策略；**未实现任意重排内部检查、仅重试单个推理 checkpoint**。实际组合故障立即保留已有证据并返回上级。

## 7. Result 与未来 image_describe

| outcome | execution | goal | 含义 |
| --- | --- | --- | --- |
| FOUND / OBSERVED | 成功 | 按契约 | 满足检测 / 单纯拍摄完成 |
| NOT_FOUND | 成功 | false | 完成检测未检出，不证明目标不存在 |
| PARTIAL | 成功 | false | 部分满足或部分能力可执行，附缺口 |
| UNSUPPORTED | 未执行 | 未满足 | 无能力或描述 API 未接入 |
| EXECUTION_ERROR | 失败 | 未验证 | 相机/模型错误、图片不可读，不伪装 NOT_FOUND |
| CANCELLED | 取消 | 未验证 | 停止新动作，保留图片 |

`check left side of robot if anything dangerous?` → Mission observe_scene 保留 analysis_kind=image_describe 与问题 → Perception 解释型任务。目前在计划前明确告知 API 未配置，不拿一张照片当“没有危险”。

未来 API 契约建议，**本轮未实现、未调用**：

```json
{
  "request": {
    "task_id": "t1", "purpose": "assess_visible_hazards",
    "question": "Anything dangerous on the left side?",
    "image_refs": [{"image_id": "obs1", "stamp": "...", "view_id": "middle_left"}],
    "limitations": ["limited_field_of_view", "not_a_safety_clearance"]
  },
  "response": {
    "status": "completed",
    "findings": [{"description": "...", "evidence_image_ids": ["obs1"], "confidence": 0.7}],
    "unknowns": ["occluded_area"], "summary": "..."
  }
}
```

真实图像发给批准的视觉 API，不让 JEV 文本 choice 冒充视觉识别。校验 schema、证据引用、取消 epoch 后返回 Perception → Mission → Human。`not observed` 不等于 `safe`，不用于解除急停或绕过碰撞检查。

## 8. 维护与验收

| 文件 | 内容 |
| --- | --- |
| `src/robot_agent/robot_agent/agents/perception_plan.py` | clause、questions、词表、能力编译与缺口 |
| `src/robot_agent/robot_agent/agents/perception_agent.py` | 三阶段、TaskRefinement、替代选项 |
| `src/robot_agent/robot_agent/jev_agent.py` | domain_questions、API/trace、监督 |
| `src/robot_skills/robot_skills/executors/observation.py` | 一次采集、多检查聚合 |
| `src/robot_skills/robot_skills/executors/perception.py` | YOLO/HSV/OCR、同实例、实际后端错误 |
| `src/robot_skills/config/skill_extensions.yaml` | 组合 Skill 登记与契约继承 |
| `src/robot_skills/config/skill_catalog.v2.yaml` | 前置条件、失败语义、逐 Skill 问题 |

新增对象/颜色先更新能力配置并验证实际模型/HSV，不仅改 prompt。新增类型需要解析、能力审查、执行器、结果、监督及测试一起登记。

本轮离线/替身回归不等于真实 JEV + Isaac 验收。下一轮测试 red box、red bottle、cup or bottle、shelf_label_06、未知替代确认/取消、低帧率、缺模型；比较语义完整率、PARTIAL 正确率、不必要人工次数、采集次数和 API 成本。

待完善：复杂自由句/布尔树；OCR 与对象空间关联；模型输出缓存；推理 checkpoint；与 Navigation 的相机租约；image_describe API。以上不当作已实现能力。

### 本轮验证记录

10/13 后续修复验证：272 项针对性回归通过；保存的 red suitcase 原始检测从 6 个组合匹配收敛为 3 个同图/正确类别匹配，suitecase 提案可人工确认或选择仅支持部分。详见 [修复记录](../mission-agent/test&pro&solve/18-evidence-and-clarification-fixes-20261013.md)。这是离线回放与替身执行，不覆盖新问题的真实 JEV 理解和新一轮现场运动。下面保留前一轮历史验证。

- 105 项针对性 Python 回归通过：领域分工、JEV 选择/损坏回答、逐项计划审查、共享采集/聚合、替代词同 Mission 确认/取消、不能抹去颜色修饰、OCR、不可读图片、标注隔离、Mission/Recovery 交接。
- 前端构建、mission-logic/mission-view 测试通过；1440px/390px 浏览器模拟消息测试确认回复使用 mission_control 而非新 command，Cancel all 一致、无页面溢出。
- 扩展的 `test_real_lab_parser_semantic.py` 有 8 项失败：6 项依赖缺失的 `maps/l402_2d_map_clean_0509`，2 项期待旧相机标定/映射；本轮没有修改这些地图或相机硬件逻辑，也未通过改验收值掩盖。
- 未启动机器人动作、未调用真实 JEV/视觉 API，未把替身测试统计为真实识别成功率。重启 Agent Dashboard 加载新 Python 代码并刷新页面即可开始实测；不需要为本轮修改重启 Isaac。
