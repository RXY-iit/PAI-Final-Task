# Dashboard: Mission / Task / Skill 分层显示

## 布局

- 左栏：Robot Health Summary、Recovery Agent；real 模式的硬件信息仍保留。
- 中栏：Mission 概要、Intent Routing、分层任务执行、结果图片、折叠的原始证据。
- 右栏：机器人状态、Joy-Con、Agent Mode、Command、Mission Tests、人工选择。
- 桌面三栏独立滚动，中栏最宽。1199px 以下依次排列中栏、操作栏、健康栏。
- Foxglove 开启时折叠在中栏，不挤占第四栏；Health / Arm 独立页面不改动。

## 显示语义

1. command_parse 显示实际解析源、Task 数量、目标概要及原始 Intent。
2. pre_step_review 是 UI 标签，当前对应真实后端 `plan_review`，没有添加新的审核阶段。绿框表示计划已确定，不等于所有运行时安全条件持续成立。
3. Primary 显示实际采用的 JEV / Classic 计划；Shadow 建议标注为比较，不冒充执行决定。
4. step_supervision 仅展示已发生的调用，关联 Task、Skill、问题 ID、答案和判断。判断标注为提案，实际结果以 Mission/Skill 状态为准。
5. task_outcome_review 单独保留原名，不将未发现目标后的依赖判断称为执行故障监督。
6. ミッションイベント 左侧是审核后的顶层 Task 或直接 Skill，右侧是该 Task 展开的步骤。子 Task 按真实 parent_task_id 嵌套；内部 task_end 不作为可执行 Step。
7. 未展开的 Task 保留占位，不杜撰内部计划。相同步骤的重试聚合，详情保留历次结果。
8. 完成、目标未达、失败、人工等待、恢复等待和取消分开显示；不因 current_step_index 越过就将旧步骤标成成功。
9. 原始 JSON 日志折叠，只显示当前 Mission，过滤 heartbeat。照片仍分为原始观测、匹配、未匹配相册，支持 search_at_place 的复合结果。

没有新增“Mission Step”业务对象，图中顶层序号只是展示顺序。

## 数据契约与源文件

- `src/robot_agent/robot_agent/schemas.py`: MissionState 增加 `parsed_intent` 和 `initial_plan` 快照。
- `src/robot_agent/robot_agent/agents/mission_agent.py`: 审核前保存解析结果、审核通过后保存主计划；在 PARSE、PLAN、Task REFINING 开始时发布状态；附加 JEV 调用所属 Task / Step 信息。
- `src/robot_agent/robot_agent/jev_agent.py`: JevCallTrace 增加 task_id、step_id、step_index、skill_name，均为可选关联字段，不修改 API 问题或执行策略。
- `src/robot_agent/web/src/lib/missionView.ts`: 纯函数构建层级、关联重试和阶段状态。旧进程缺少快照时优先使用事件，再使用 Task 记录重建；原始解析缺失明确标记，不冒充已保存。
- `src/robot_agent/web/src/components/MissionProgress.tsx`: 阶段概要及任务执行层级。
- `src/robot_agent/web/src/components/MissionPanel.tsx`: Mission 概要、图片与原始证据。
- `src/robot_agent/web/src/App.tsx`、`styles.css`: 三栏布局。

前端已构建到现有 Dashboard 使用的 dist，刷新页面即可更新布局。新增后端快照及解析中状态需在下一次 Agent 服务启动后生效；本轮不打断当前机器人会话来强制重启。

## 验证

- Python：领域任务、能力及 Mission/Recovery handoff 共 38 项通过。
- 前端：已有 2 项 + 新增 9 项纯逻辑测试通过，TypeScript / Vite 构建通过。
- `web/test/dashboard-layout.cjs`：1920、1440、1280、390 四种宽度，检查三栏顺序、嵌套 Task、待执行项、Primary 拒绝、Shadow 比较、人工等待、取消清空、页面异常及横向溢出。WebSocket 全部拦截，不发送控制指令。
- 历史真实证据回放：`debug-output/sessions/20261012T022650Z-4f03f3df` 中 Mission `5f21e132df25`，成功还原 1 个 navigation Task、嵌套 perception Task、7 个 Skill；原始观测及检测标注图片可加载。
- 历史 report 中的证据经过精简，图片回放引用同 session 的 skill_events.jsonl，未生成重复完整报告。

本轮验证的是显示与遥测关联，不将历史回放计为新的机器人实测，也没有改动导航、恢复或 JEV 的判断策略。
