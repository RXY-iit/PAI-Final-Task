# 问题与解决记录

日期：2026-10-09。下表是既有观察形成的种子，不是本轮新测试结果。所有修复必须关联 failing run 和复测证据，不能因为代码修改完成就关闭问题。

## 1. 优先复测与修复顺序

| problem_id | 优先级 | 已有事实 / 待验证点 | 对应 case | 下一步与关闭条件 | 状态 |
|---|---|---|---|---|---|
| P001 | P0 | 10/07 Skill 漂移告警与 Health 的定位 AVAILABLE 冲突；Recovery request 丢失 skill/result metrics | R02、C03、M07 | 先统一/传全证据；同一失败上下文能够解释各层判定，恢复后 Skill 重查通过，不放宽门槛掩盖问题 | 待复测/修复 |
| P002 | P0 | `NOT_RELATED` 被变成人工等待，随后通用提示否认已出现的漂移 | R02、M11、C03 | 明确系统故障、任务失败、未证实故障；保留真实原因；不同返回状态有可测试的 Mission 后续行为 | 待复测/修复 |
| P003 | P0 | 用户取消导航后仍调用 JEV，曾得到低置信度 retry 建议；代码取消保护阻止了重试 | M14、M22、R04、C02 | Cancel 优先于监督；迟到回复不恢复旧任务；正常非用户取消仍可分类 | 待复测/修复 |
| P004 | P1 | Human Report 曾将 CPU warning 显示成定位任务主因 | R02、R08 | 当前阻塞、当前人工操作、历史 warning 分开；摘要引用正确 mission/step 和证据 | 待复测/修复 |
| P005 | P1 | JEV 解析固定提问不存在的后续步骤/无关 slots；任务依赖曾低置信度 | M08–M17、C06 | 按任务形态/类型选择问题；语义保持正确；减少 token/耗时须有对照，不能单凭问题变少宣称改善 | 待设计/复测 |
| P006 | P1 | SUCCEEDED 可能只是自然恢复；PB-NAV-BLOCKED 的验证不能单凭 controller active 证明路径可通 | R09–R11、C07 | 明确恢复归因与所恢复 capability；验证后原 mission 实际续行，仍堵塞不能算最终任务恢复 | 待复测 |
| P007 | P1 | token 计数被通用敏感字段规则隐藏，影响成本审计 | M08、M21 | 凭据继续脱敏，数值 usage 可审计；estimate 与实际 usage 分开 | 待修复 |
| P008 | P1 | 定位 playbook 验证比设计总结窄：主要看 GICP score/pose | R02、R10 | 验证加入必要一致性/漂移依据；真实精度留待 Isaac 真值评估，不把 score 当精度 | 待设计/复测 |
| P009 | P2 | 缺统一故障注入/回放、RTF item 和真值采集 | T01–T05 | 先实现可确认注入与可恢复的最小工具，独立 domain；不能依赖尚不存在的 topic | 工具缺口 |
| P010 | P1 | 前次修复了低频警告、忽视与资源白屏，但仍需当前版本端到端回归 | R01、R07、R08、M22 | 页面资源完整；warning 忽视持久/复发/升级正确；report 历史完整且当前准确 | 待回归，不预先记通过 |
| P011 | P0 | 10/10 实测：积累失败后新点云已达 36/36，GICP 仍永久拒收；再次 initialpose 后 11.286 s 恢复输出 | R10 | 有界重试/重新积累、错误分类，失败不能默认放行导航 | DIAGNOSED，未修复 |
| P012 | P1 | 10/10 实测：FAST-LIO hint 开启但源时间滞后 12.58-18.53 仿真秒，GICP hint 没有时间有效性检查 | R02、R07、R10 | 时间对齐、过期 hint 排除；队列与同步检查；受控 A/B | DIAGNOSED，漂移因果待验证 |
| P013 | P1 | Health 读原始 accepted GICP pose，RViz 使用平面化/平滑 TF；本轮还观测到明显高度差 | R02、R10 | 明确 frame/时间/估计语义，核对几何约束与外参 | DIAGNOSED，未修复 |
| P014 | P1 | 静止漂移比较基准在初值后首次匹配即建立；实测 GICP 最大变化 0.752 m，轮式里程计整个窗口仅 0.0084 m | R02、R10 | 区分收敛、稳定与漂移；不依赖低 RMSE 或单纯延长等待 | REPRODUCED，根因待区分 |

P011-P014 为 10/10 新实测/代码审查，详见 [主动排查](07-sim-localization-debug-20261010.md)。此前 P001/P008 的局部代码回归不等于本轮端到端定位恢复通过。

10/10 后续状态以 [08 修复验收](08-localization-navigation-fix-20261010.md) 为准：P011 已修复并通过有界窗口回归/Sim 初值重设；P012 已排除过期 hint，源时间积压尚未解决；P013 已分离 raw 与 accepted 导航表示；P014 本场景静止已稳定，但一般化收敛策略仍待验证。另修复导航将正常朝向对齐误判停滞的问题；新增待办为低 RTF 下 180 度相对旋转的固定 30 s wall-time 预算。

10/07 证据源：[skill_events](../../../debug-output/agent/20261007_181825/skill_events.jsonl)、[jev_calls](../../../debug-output/agent/20261007_181825/jev_calls.jsonl)。这些路径只对应当次测试，不替代本轮 run_id。

## 2. 归类规则

2026-10-09 修改状态（尚未关闭问题）：P001 已接通 Mission 一致性健康输入及失败 skill/metrics；P002 已修复定位恢复失败原因被通用文案覆盖，其他任务失败策略仍待验证；P008 已追加新证据/一致性验证、可信位姿恢复和有覆盖条件的点云积累。均为 **FIXED_PENDING_TEST**（P002 为部分修复），需 R02/R04/R10 的 Sim 证据才能关闭。当前局部测试、浏览器检查和全量测试环境缺口见 [实现记录](05-implementation-notes.md)。

- HEALTH_EVIDENCE：原始数据、单位、阈值、时间窗口或两套健康判断不一致。
- CONTRACT：request 丢失、字段缺失、id 关联错误、final 协议错误。
- POLICY：独立/条件任务、重试/跳过/人工的规则未明确或不一致。
- EXECUTION：skill、backend、取消、超时、互斥实际行为有误。
- JEV：输入/问题设计不当、选择错误、低置信度/矛盾未处理；不能把缺失输入归因于模型笨。
- OBSERVABILITY：report 主因错误、日志缺项、cost 不可审计、时间难关联。
- TOOLING / ENVIRONMENT：故障未触发、测试工具缺失、起点/负载变化造成无效对照。

一个问题可以跨多层，但应明确“最早出错的环节”。只修 UI 提示不能关闭协议错误；只补一个 if 分支不能关闭未知依赖策略。

## 3. 每个问题的记录模板

```markdown
## Pxxx - 简短标题

- status: NEW / REPRODUCED / DIAGNOSED / FIXED_PENDING_TEST / VERIFIED / DEFERRED
- severity / owner:
- related_case_ids / failing_run_ids:
- code/config revision:
- expected behavior (written before test):
- actual behavior:
- earliest incorrect stage:
- minimal reproduction: command, prerequisites, one fault/action, reset
- evidence: file + line/time + mission/request/incident/execution IDs
- confirmed facts:
- hypotheses (not facts):
- alternative explanation / next discriminating check:
- fix scope and rationale:
- changed files / diff reference (redacted):
- regression: original case + normal case + opposite condition + Cancel
- retest run_ids / repetitions / actual measurements:
- remaining limitations:
- close decision and evidence:
```

## 4. 批量修改的粒度

第一批：P001/P002/P008，解决定位证据、恢复协议与验证一致性。

第二批：P003 以及 C01–C07，解决取消、迟到/重复回复、恢复续行与执行边界。若发现取消后继续运动等实际错误，立即提升到第一批，停止相关运动测试。

第三批：M11–M13/P005，冻结任务依赖策略，再优化问题设计与置信度处理；不要用新措辞绕开原失败用例。

第四批：P004/P007/P010，完善当前摘要、证据可追溯性、成本与页面回归。影响定位根因的日志缺失应提前处理。

每批修复后跑固定核心集：M01/M02/M04/M08/M11/M12/M14、R02/R04/R08，以及被修改模块的单元测试；其他用例按影响范围补充。没有实测的项保留待测，不能用一个成功视频关闭整批。
