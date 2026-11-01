# Perception Agent 实际检查与问题审计

日期：2026-10-12。本文描述当前代码及实测，不把 README 中的目标架构当作已实现。

## 1. 实际调用链

Mission 的 `search_object` Task（包括 Navigation 的视觉确认子 Task）
-> `PerceptionAgent.refine` -> 确定性 query/provider/capability 检查
-> `perception_model_review` -> 一个 `search_at_place` Step
-> 内部九视角静止检查、相机到位、新帧保存、归位、检测
-> SkillResult -> Perception Task feedback -> 父 Navigation/Mission 汇总。

源码：
- `src/robot_agent/robot_agent/agents/perception_agent.py`: refine、_provider_for_query、_arrival_confirmation_query。
- `src/robot_agent/robot_agent/agents/domain.py`: Classic/Primary/Shadow 采纳边界。
- `src/robot_agent/robot_agent/jev_agent.py`: domain_review、_decide、脱敏请求日志。
- `src/robot_skills/robot_skills/executors/perception.py`: query plan、OCR、数字匹配、检测结果。
- `src/robot_skills/robot_skills/executors/observation.py`: 组合执行、取消、图片隔离。
- `src/robot_skills/config/camera_scan_profiles.yaml`: 固定视角。
- `src/robot_skills/config/perception_models.yaml`: provider 与 OCR 部署。

## 2. 当前真正问了哪些问题

共同说明（由 domain_review 发送）：只深化当前 Task、保持原目标、只能选择给出的策略；证据未知/冲突/缺失时 ask_human，不生成模型或坐标。

| Stage | 输入 state | 唯一 API question：decision (choice) | 回答如何生效 |
| --- | --- | --- | --- |
| perception_model_review | stage_goal、Task（object_query/goal_mode/raw_text）、query、capability 快照 | query 默认 provider / ask_human；默认 yolo11s 时额外提供 yolo11s_ocr | Primary 写入 search_at_place.provider；Shadow 仅记录，采用确定性 default |
| perception_observation_review | stage_goal、observe_scene Task、profile、base_motion=false | execute / ask_human | execute 使用既有方向/环视/当前画面计划；不让模型产生角度 |

模型选项的实际说明：
- default provider: `Use the query-compatible configured provider.`
- yolo11s_ocr（补充选项）: `Use the configured detector with OCR support.`
- ask_human: `Clarify the query/model requirement.`

观察选项的实际说明：
- execute: `Execute the requested bounded observation profile.`
- ask_human: `Direction or observation scope is ambiguous.`

不是每个搜索都会调用 observation_review：当前 `search_object` 的九视角 profile 在代码中固定，只有 model_review；observe_scene 才走 observation_review。
capability 包含 query plan、依赖/权重、RGB age 等，但不保证 EasyOCR 已经成功加载。API 成功也不等于模型能检测到目标。

## 3. shelf_label_06 具体发生了什么

历史 session：`debug-output/sessions/20261012T015308Z-d41d5959/agent/`。
任务 `2ab225e70c8a`、`e4de9fe1352d` 均已解析为：

```json
{"type":"text","target_text":"06","text_kind":"shelf_label","required_digits":["06"]}
```

JEV model_review 选择 yolo11s_ocr。数字 query 实际走 OCR，并非让 YOLO 寻找名为 shelf_label_06 的 COCO 类；JEV 没有看图片。
两组 `observation_007.jpg` 可见标签 06。Tesseract 原分辨率漏检；EasyOCR 的 `craft_mlt_25k.pth` 缺失且禁止自动下载。因此“query 解析错了”不是这两次失败的主要证据。

本轮修复：
- 棚号按完整数字 token 比较，允许 06 与 6 等价；106/60/07 不等价。普通 `text_06` 仍严格保留前导零。
- 不截取长编号末两位；不拼接多个分离数字伪造标签。
- 数字 OCR 原图未匹配时增加一次 2 倍上采样，bbox 映射回原图；不降低置信度阈值。
- 18 张历史图片离线回放：第二组第 7 张读到 06，confidence=0.89；第一组仍无可靠匹配。并未全面解决视角、倾斜、小字和 OCR 稳定性。
- 实测 `dcc76b090262`：Primary 原地 find shelf_label_06，九张新图、FOUND，Task/Mission SATISFIED。

## 4. 未检出不是执行异常

```json
{"success":true,"code":"OK","metrics":{"inference_completed":true,"outcome":"NOT_FOUND","goal_satisfied":false}}
```

FOUND 同样 success=true，但 goal_satisfied=true。后端全部失败、无相机反馈、取消/超时仍是执行错误；部分候选/不支持语义需要确认时仍 NEEDS_HUMAN。
Task 不能只看 success 来判断目标满足。Navigation 要求视觉确认时，子 Task UNMET 导致父 Task UNMET，不篡改 Nav2 本身的 completion_class。

Mission 对后续任务处理：
- 无后续目标：结束并报告 PARTIAL，无需强行 ask_human。
- 明确“无论找到与否回 home”：直接执行清理 Task，保留搜索 UNMET。
- 已声明独立目标：可以继续；Primary 在任务边界使用 task_outcome_review，不是 step_supervision。
- required_condition/依赖不明：不能猜测允许继续，询问人。

实测 `14fe7cfae015`：find shelf_label_99, then set pose home regardless of whether it is found；搜索 NOT_FOUND、home 成功、Mission PARTIAL，0 次 step_supervision。

## 5. 尚未实现与待设计

当前没有完整的 perception_goal/query/plan/result 多阶段 JEV。没有“看到数字就证明它属于目标货架”的实例关联；没有自适应重拍、OCR 模型动态部署或自由生成模型。请 review：
1. 棚号的补零 policy 是否只限 shelf_label？当前是。
2. 到达后确认是否必须关联货架本体，而不只是读到数字？当前不能证明关联。
3. 失败时先定点重拍/裁剪/deskew，还是问人？本轮只增加一次有界 2x OCR。
4. 正常 NOT_FOUND 后，哪些任务应明确标为独立，哪些依赖识别结果？

原始问题/答案：session 下 `agent/jev_calls.jsonl` 的 request.questions、request.state、response_raw.answers；`agent/skill_events.jsonl` 用 mission_id/task_id 关联。脱敏不再删除 input_tokens/required_tokens 等非凭据字段。
