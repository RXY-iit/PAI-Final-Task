# Mission / JEV / Recovery 测试与问题闭环

日期：2026-10-09。环境：Isaac Sim；范围：接入 thinking LLM 之前的基础功能、可预知任务、已知故障恢复和调查资料归纳。

这是可执行测试计划，不是测试完成报告。本文提出的验收标准是下一轮的目标，不代表当前代码已经满足。

## 1. 方法与文件入口

“测试暴露问题 → 集中修改 → 再测试”的方向正确。建议把广泛随机测试改为：**先固定预期和基线，小批次单变量测试，复现一个问题后归类，再集中修改；修复后跑原用例和邻近反例，最后扩大组合。**

| 文件 | 用途 |
|---|---|
| [01-mission-cases.md](01-mission-cases.md) | 输入什么 command，检查什么任务图与动作 |
| [02-recovery-cases.md](02-recovery-cases.md) | 如何制造/解除故障，检查恢复、人工交互、续行 |
| [03-evidence-and-comparison.md](03-evidence-and-comparison.md) | 采集证据、对照实验、统计、判断设计收益 |
| [04-problems-and-solutions.md](04-problems-and-solutions.md) | 已知问题种子、修复顺序、问题与解决记录模板 |
| [run-template.json](run-template.json) | 每次执行一份结构化记录；未知值用 null，不填估计成功 |
| [05-implementation-notes.md](05-implementation-notes.md) | 数据复用、快捷测试、可信位姿恢复和 RTF 边界 |
| [06-mission-recording-and-sim-health.md](06-mission-recording-and-sim-health.md) | 最新：任务报告边界、启动目录、日志保留、按步骤触发恢复与 Sim 监测 |
| [07-sim-localization-debug-20261010.md](07-sim-localization-debug-20261010.md) | 10/10 实测：四个小范围运动、积累失败锁死、单次初值恢复输出、静止 GICP 不稳定与过期 hint |
| [08-localization-navigation-fix-20261010.md](08-localization-navigation-fix-20261010.md) | 10/10 后续修复：定位时间/几何证据、积累恢复、home 导航回归、仍存在的旋转时间预算问题 |
| [09-sim-time-and-localization-20261010.md](09-sim-time-and-localization-20261010.md) | 最新：sim 时间积累、IMU/odom 时序、可信底盘锚点、FAST-LIO 队列、转向反馈和自动恢复实测 |
| [10-stability-validation-20261010.md](10-stability-validation-20261010.md) | 本轮实现与验收：20 分钟观察、三轮运动、真实 JEV、定位故障恢复、导航/视觉分级、报告与待解决项 |
| [14-domain-agent-suite-20261012.md](14-domain-agent-suite-20261012.md) | 四任务 Agent 的 Classic 实测：委派、续行、取消、未知 Arm pose 修复、OCR 与里程计剩余问题；本轮真实 JEV 对照待授权 |
| [16-primary-suite-20261012.md](16-primary-suite-20261012.md) | 新一轮真实 JEV Primary 与 Classic 对照；失败分流、原 Mission 人工继续、Recovery 协议隔离回放、API 在途取消与证据统计 |

每个 case 可执行多次，每次有独立 run_id；同一根因只建一个 problem_id。命令、故障、模式都相同时才是重复试验。

## 2. 一轮怎么做

1. **冻结本轮版本**：记录 git HEAD、dirty 文件列表、场景/map/profile、配置、JEV 实际返回模型、Classic 的 embedding/LLM 开关。测试中不改代码、不调阈值；需修改时结束本轮，开启新 revision。
2. **定义成功**：执行前写任务图、单位/默认值、允许动作、失败时继续策略、运动容差和观测期限。不要看结果后修改预期。含糊的依赖关系以澄清为合格，不以“猜中”冒充正确。
3. **建立基线**：先 M01–M07。导航基线必须在定位证据一致时做；定位不健康时做相对运动/感知测试，标注上下文，不混入健康导航成功率。
4. **单故障**：先 R01–R08，每次只操作一个来源。故障是否真正注入要先被证据确认。
5. **复现并归类**：同一失败通常复现 2–3 次就足够提交修复，不在已知阻塞点上反复浪费整轮时间。危险继续、取消后运动、重复执行出现一次就停止该类测试并修复。
6. **集中修改**：优先修正健康证据与协议，再改监督策略，最后优化 question、显示和成本；不要只改 prompt 来掩盖丢失的输入证据。
7. **回归**：原失败用例 + 正常路径 + 相反条件 + Cancel。通过后再做跨模式对照、组合故障和语言变体。
8. **封存**：保留原始失败、修改说明、修复后记录和仍未解决的问题。单元测试通过不能代替 Sim 端到端通过。

开发时每个关键用例先重复 3 次；稳定后核心用例重复 5 次。对照评估建议预先冻结每个条件 10 次以上的试验预算并报告实际 n；这些是工程采样建议，不是统计显著性保证。不同模式交错执行，避免负载/温度随时间变化只影响某一模式。

这里的 thinking LLM 指后续深度推理/调查模块，与现有 Classic parser 的 LLM fallback 不是同一个开关。第一轮建议关闭 parser LLM fallback，固定并记录 embedding 状态；后续需要比较完整 Classic 时，另建配置组，不能把两组结果混为一个 baseline。若现有启动方式不能独立控制开关，记录这一限制，不假设已经关闭。

## 3. 推荐第一轮顺序

| 顺序 | case | 主要暴露的问题 |
|---|---|---|
| 1 | R01、M01 | 启动后证据更新、页面与 report、基本命令通路 |
| 2 | M02、M03、M04、M05 | 相机/机械臂区别、参数、相对动作稳定性 |
| 3 | M08、M09 | Classic 漏步骤是否被 JEV 补全，单位是否正确 |
| 4 | M14、R04 | Cancel 是否终止执行与恢复，是否产生错误重试 |
| 5 | M07、R02 | 定位 Skill / Health / Recovery 是否依据一致 |
| 6 | M11、M12、M13 | 找不到目标时，独立、条件、含糊任务的区别 |
| 7 | R03、R05、R06 | AUTO/MANUAL、软件急停、Done 不等于验证成功 |
| 8 | R07、R08 | 低频/断流分级；忽视、复发、升级、恢复提醒 |
| 9 | R09、R10、R11 | 节点恢复、定位恢复、路径受阻后真正续行 |

上一步失败不一定阻止后续全部测试：例如定位失败可继续测试静止感知，但不能把此时的导航当作健康基线。

## 4. 当前实现与设计边界

已核对 `recovery_agent.py`、`engine.py`、`executor.py`、`envs/sim.yaml`、相关 playbook、parser templates 和最近测试日志。

- Dashboard 的 Classic / JEV Primary / JEV Shadow 以及 recovery request/response 已有实现。Shadow 的执行行为由 Classic 决定，不能用于证明 JEV 建议执行后成功。
- `recovery_execute=false` 是“诊断、不自动执行 playbook”，不是完全关闭 Recovery。Mission 定位兼容自动 reset 已移除，其余等待/重试仍需记录，不把配置名当作动作事实。
- `fault_injection` YAML 目前是描述。没有找到统一 `recovery_fault_injector` 节点；不能把 `/recovery_test/inject` 当成可用接口。
- `PB-SIM-RTF-LOW` 已接入 `/clock`/monotonic 的 RTF 证据，仅 advisory；不自动修改控制周期，详见 06。
- PB-LOC-LOST 已改为 last_trusted_pose → 新证据验证 → 人工；不是语义地标检测。代码回归与真实 Sim 验证分开记录。
- 10 月 7 日发现的定位证据矛盾、请求空 metrics、取消后的多余 JEV 判断等是本轮优先复测项，不假设已修好。
- 基础阶段遇到未知原因应终止/等待人工，输出证据包和明确的能力缺口；不依赖尚未接入的 thinking LLM 给答案。

## 5. 本轮统一原则

- user Cancel、软件急停、执行器互斥是代码约束；JEV 不能改变这些决定。
- 是否继续下一子任务由依赖关系和当下执行前置条件共同决定，不只看 `then/and`。
- 只忽视 warning 的提醒，不把故障健康状态改成 OK，也不跳过 skill 的前置检查。
- 定位成功不能只用 GICP 自评分证明；导航可用还需要位姿新鲜度、静止漂移/一致性等证据。若真值未接入，报告“未验证真实定位精度”。
- 机器人状态正常和任务完成分开统计；场景自己恢复、用户解除故障、自动动作恢复分开统计。

## 6. 依据

- [共通设计](../../recovery-agent/design.md)、[Sim 设计](../../recovery-agent/sim-guideline.md)、[JEV 设计](../jev-agent-design.md)、[论文评估草稿](../../../paper/paper2-mission-supervision.md)。本目录服务于开发验收，尚未开展 LLM-only、R3 等研究对照。
- [用户的公开设计总结](https://rxy-iit.github.io/Plan26/subpages/agent-design/recovery-agent/recovery-agent-design-ja.html)：与本轮相关的原则是证据驱动排查、恢复后验证和 Mission 续行。网页的定位示例还要求检查静止漂移；当前 PB-LOC-LOST 的 verify 主要检查 GICP score/pose，两者差异通过 R02/R10 验证。页面仍标注设计稿，不能用作“功能已实现”的依据。
