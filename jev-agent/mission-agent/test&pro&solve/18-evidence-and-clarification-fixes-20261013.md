# 2026-10-13：证据与人工澄清修复

依据：16-primary-suite 第 7 节的用户实测和离线审计。只修已确认的问题；不扩充 Classic 的连接词规则，不调整 Recovery PB，不改模型、检测阈值或机器人安全门。

## 修复清单

| 问题 | 修改 | 验证及边界 |
| --- | --- | --- |
| 组合匹配接受错误类别/跨图片 | PerceptionSkills 要求同 image_path、目标类别/颜色、matched_query、有效框/置信度，拒绝未批准近似匹配；输出双方证据框 | 历史 6 个匹配回放为 3 个；bus/truck、不同照片同坐标、NaN/空路径等负例拒绝。仍是 bbox 重叠启发式，不是实例分割或真实类别证明 |
| 支持颜色时丢失对象纠正选项 | Perception 在计划前返回确认提案，或仅检测已支持部分；确认后同 Task 重新审查 | red suitecase 已记录回答回放可选 suitcase 或仅 red；后者保留缺口/PARTIAL。确认、拒绝替代、取消三分支测试无预先采集、无重复任务 |
| 缺少上下视角 | view 的选项与执行 pan/tilt 从现有 camera_scan_profiles.yaml 九视角读取；task catalog 增加 band | 九个视角参数化测试通过；unknown band 不再默认中间。仅采图与描述分开，未接入 image_describe |
| alternative_type 重复主类型 | 增加按需 command_disambiguation，一批追问只包含重复的任务，明确主解释并排除该选项 | none 可继续，另一类型交用户选择；缺项、非法、unclear 阻止绑定。不无限重试；取消后不进入绑定或执行 |
| 请求版本、probe 落后 | schema_version=jev_robot_agent.v2；日志 questions_sha256、purpose；probe 复用正式 parse/bind/review builders | 三次只读 probe 要求全部问题有合法答案；空 answers 不显示 available。probe 不执行任务，记录 usage/latency，不能计作 Mission 调用或语义准确性验收 |

## 代码入口

- `src/robot_skills/robot_skills/executors/perception.py`：组合匹配证据过滤。
- `src/robot_agent/robot_agent/agents/perception_plan.py`：对象解释确认与未知修饰保留。
- `src/robot_agent/robot_agent/agents/perception_agent.py`：Ask Human 选项、九视角解析及计划生成。
- `src/robot_agent/robot_agent/command_grounding.py`：排除已选类型的消歧问题生成。
- `src/robot_agent/robot_agent/jev_agent.py`：有界追问、v2、probe、问题哈希日志。
- `src/robot_agent/robot_agent/agents/mission_agent.py`：向解析器传递取消状态，沿用同 Mission/Task 的人工等待协议。

## 验证

- 针对性回归：**272 passed，19.69s**。`debug-output/validation/20261013-restart-review/fix-regression.xml`。
- 实际历史证据回放：`replay_fixes.py` / `fix-replay.json`，与旧 audit.json 分开，保留来源 SHA256 和当前源文件指纹。没有重写原始检测、JEV 回答或 human report。
- 回放重用已有模型检测结果和 JEV 回答；人工选择由测试模拟。不新增真实 API、模型推理、ROS 命令或机器人动作。
- 人工确认后计划从 color-only 变为 composite，该分支复用的 supported 回答只是测试替身，不证明真实 JEV 已审查过新计划。
- 新的 command_disambiguation、view 问题及 v2 probe 只有离线契约验证；不能因此宣称真实 API 已稳定理解。
- `colcon build --packages-select robot_skills robot_agent --symlink-install` 两包成功，`git diff --check` 无问题。未重启用户进程；应用修改只需重启 Agent stack，不需要重启 Isaac Sim。

## 下一轮现场验证

1. 重启 Agent stack 后测试 `go back 0.2 meter, next turn left, after that set pose home`；若消歧，确认/取消都不得重放已执行步骤。
2. `find red suitecase` 应出现对象解释确认。确认后检测 red suitcase；仅支持部分应保留 PARTIAL；Cancel all 不采图。
3. `look ground side` / `look upper left`：检查 view 对应配置与实际相机反馈；危险描述仍不能冒充采图成功。
4. 检测 `red suitcase` 时检查每个 composite_match 的双方路径和类别；同图检测正确也不等于场景资产真实类别已经人工确认。
5. 新出现的歧义、未知名词或模型误识别继续记录，不靠追加特定拼写绕过模型能力和人工确认边界。

Classic 截断后的整体成功显示、模型真实误分类、复杂颜色修饰、相对云台调整等仍有边界；本轮不宣称完整语言覆盖或实机验收。
