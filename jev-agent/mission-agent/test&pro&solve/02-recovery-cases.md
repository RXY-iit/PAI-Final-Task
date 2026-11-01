# Recovery 故障与操作测试

返回 [README](README.md)。以下只面向当前 Sim。执行前保存场景与启动配置；每个 case 的恢复/复位方式要先明确。这里的操作由测试者逐项执行，本计划没有向运行中的机器人注入故障。

## 1. 可直接做的单故障测试

编号约定：`Rxx` 是现场恢复用例；`P0/P1` 是优先级（P0 先做），不是 playbook 或测试编号；`Cxx` 是跨模块协议/安全约束测试；`Txx` 是测试工具与观测能力待办，未实现时不是可执行用例。另一文件的 `P001` 是 problem ID，与优先级 P0 无关。C 主要验证 playbook 以外的通用逻辑，也验证与 playbook 的接口。fault_injection 实现暂后置。

| ID / 优先级 | 操作步骤 | 预期与失败判据 | 解除/复位 |
|---|---|---|---|
| R01 / P0 | 正常 Full 启动，保留启动诊断，启动完成后对照 Health、Recovery、Mission | 随新证据更新；已恢复 incident 不应长期仍算当前故障；不依赖固定启动宽限期。记录打开/恢复时间 | 不人为延迟节点；保持正常运行 |
| R02 / P0 | 静止时运行 `ロボット状態`，保存 GICP 分数/age/漂移；若已有漂移告警，提交 `go to shelf_06` | Skill 拒绝导航时 request 必须包含失败 skill/code/metrics；Recovery 不得仅凭 score 正常忽略漂移。若未复现，标 NOT_REPRODUCED，不算修复通过 | Cancel；人工核对定位，不反复发目标 |
| R03 / P0 | 静止切 MANUAL，再发 `go back 20cm`；在确认窗口未出现有效 AUTO 前先点 Done，然后用 A 切 AUTO | Done 不能伪造 AUTO；有实际权限证据后 Mission 重查并只执行一次。是否走 PB-MODE-MANUAL 或 Mission 权限确认要记录 | Cancel 或明确切回约定模式 |
| R04 / P0 | 等待恢复/operator request 时点任务 Cancel；随后解除原故障 | 原任务保持 CANCELLED，最终 recovery 回复不得让原 mission 续行；Report 保留取消历史，当前面板清理 | 故障解除后单独发新任务 |
| R05 / P0 | 静止并取得 Dashboard control，按 Y 触发软件 ESTOP；发一个运动指令 | 软件门阻断；A2+ 恢复等待。UI 的“全停止/Cancel”不是同一测试。不能直接向 `/safety_status` 发伪 OK 来解除 | 按真实控制入口解除 Y，再检查状态；不可因解除自动执行旧 Cancel 任务 |
| R06 / P0 | R05 的人工请求中，保持 ESTOP 点 Done；另一轮点 Cannot/Abort；有 Approve/Reject 的请求才测对应按钮 | 无证据时仍 waiting/Not detected；Cannot/Abort 有明确终态且不自动循环弹窗；批准仅代表同意，不代表前置条件已满足 | 解除真实门状态；每个分支新 run |
| R07 / P1 | 对一个已确认的 Isaac camera/LiDAR publisher 降低发布频率，再单独做完全停流变体 | 有效低频 WARN，持续断流 STALE/故障；不把坏载荷或从未收到数据当低频。记录实际接收率、age、publisher、clock | 恢复原频率/enable，确认新消息到达 |
| R08 / P1 | 对已有温度/分辨率/低频 warning 点暂时忽视；消除再复现同类 warning；点恢复提醒 | 同类复发进摘要；健康证据仍真实。另测低频忽视后完全停流：升级后必须重新显示，不沿用忽视 | 恢复原配置、恢复提醒；记录 ack_store 状态 |
| R09 / P1 | MANUAL、静止、软件安全门满足 playbook 条件时，只停止 `sim_omni_drive_command_node` 的确认 PID | 记录进程死亡和对应 health item 是否真变化；若不能触发，记观测缺口。能触发时只重启目标组件，验证反馈，不能重启整栈冒充局部恢复 | 自动恢复失败时用原参数重启该组件，确认没有双实例 |
| R10 / P1 | 静止、点云正常且 map 正确；先保存健康可信位置，再用 RViz 给一个已记录偏移的初始估计，观测是否确实形成定位故障 | 有有效锚点时：可信位置 + wheel odom 小范围推算 → 新点云 → GICP/一致性验证 → Mission 重查；没有有效锚点直接人工。不允许 spin，不能仅凭 initialpose 报成功 | 自动失败时用正确 RViz 初值恢复，保留日志 |
| R11 / P1 | 健康导航中，在明确可观测路径上放置一个带碰撞/可被相机感知的障碍，保持到目标受阻；另一轮移除 | 目标失败证据传到 PB-NAV-BLOCKED；controller active 不等于路径可通。Done 后仍堵塞不能算任务恢复；移除后需重查/重试目标并记录完成 | Cancel 后移除障碍，恢复同一场景 |
| R12 / P1 | 静止时 Pause Isaac，再 Play；检查 clock 与派生传感器告警 | 优先识别上游时钟停止，避免同时自动重启大量下游；Play 后证据恢复并收敛。暂停不等于运动中取消测试 | Play，确认消息、时间、定位恢复 |

操作补充：

- R07 的低频和停流是两次独立 run。只操作已确认发布目标 topic 的 Graph 节点/频率设置；无法识别时标 BLOCKED_TOOLING，不能用 Pause 整个 Isaac 代替单传感器停流。记录原值、修改值、改回方式。
- R08 的温度告警使用自然出现的告警，不用额外压力程序制造过热。无现成告警时用可控相机配置偏差；不具备条件就先离线测。
- R09 不在包含多个子进程的启动终端随意 Ctrl+C，也不使用主机范围 `pkill -f`。先 `pgrep -af sim_omni_drive_command_node`，核对单一 PID 的命令及 ROS_DOMAIN_ID 与当前 Sim 相同，再对该 PID `kill -TERM <已核对的PID>`。没有核对就不执行。
- R09 后备重启必须复用 `envs/sim.yaml` / 本次 launcher 的参数。先确认自动恢复没有已经启动另一实例。A2 被 WATCHDOG/非 OK 安全门挡住时应记录前置条件等待，不伪造安全状态或归类为重启失败。
- R10 的错误 initialpose 不保证形成故障：GICP 可能拒绝或自行纠正。没有可观察故障时是 INJECTION_INEFFECTIVE，不能计为 Recovery 自动成功。物理 kidnap 与错误初值是不同故障，不能互相替代结论。
- 当前实现为 `last_trusted_pose`，不是视觉语义地标识别。RViz 重置会使 Mission 的旧 wheel odom 锚点失效，因此 R10 不保证覆盖“自动可信位姿恢复成功”路径；直接转人工可能符合前提检查。优先用 R02 自然漂移复现自动路径，未来再用隔离 fault injector 精确测试。
- 定位细节、低频窗口与限制见 [05-implementation-notes.md](05-implementation-notes.md)。

## 2. 每个故障必须检查完整链路

```text
注入操作与时间
  -> 原始证据确实改变
  -> health item / capability 更新
  -> incident + 诊断路径 + playbook
  -> 前置条件 / operator 选择 / 实际恢复动作
  -> 新证据验证（不是按钮确认）
  -> recovery final response
  -> Mission 重查前置条件并续行/终止
  -> 最终任务结果 + 当前 UI + 历史 report
```

任一环节缺失，记录在哪一环断开。至少保存 mission_id、request_id、incident_id、execution_id、operator_request_id；不能只拿一个 SUCCEEDED 截图算端到端通过。

## 3. 协议专项：先离线、再受控端到端

这些可以通过现有 engine/executor 单元测试增加 FakeBackend/快照序列，不需要操纵 Isaac。集成回放工具未实现，禁止把伪 health/response 混发到 live ROS domain。

| ID | 输入/事件序列 | 合格条件 |
|---|---|---|
| C01 | request X → final:false → 多次处理中 → final:true | 始终关联 X；不会 5 秒就误判失败；最终只恢复一次；计时采用单调墙钟 |
| C02 | 等待 X 时先收到 Y 回复；再重复 X final；或 Cancel 后收到 X final | 忽略无关/重复/迟到回复；不重放已完成动作 |
| C03 | 健康表面正常，但失败 metrics 含漂移；反例为 score/pose 真正常且任务目标不存在 | 前者保留矛盾证据并调查/恢复，后者不自动执行系统重启 |
| C04 | 一个 A2 恢复运行中，另一个 A2 请求到达；中途 ESTOP | 互斥；新动作等待。区分“禁止新动作”和“中止已经运行的动作”的实际能力，不假设 runner 能撤销所有子进程 |
| C05 | 恢复接近超时、超时后才 final；健康反复好/坏；连续多次同故障 | 有限重试、cooldown/chronic、终态明确；没有无穷重启/询问；不通过重置 mission 无限绕过预算 |
| C06 | JEV 超时、字段缺失、低置信度、成功判定但输出矛盾 | 明确失败/降级/人工，日志可解释；不以 local 模拟结果冒充 live；仅传 schema 合法值仍不足以证明语义正确 |
| C07 | 完成 recovery action 后但验证未通过；人工点 Done | 不报 RESOLVED；恢复“执行完成”和“故障解决”分开 |
| C08 | 相机低频 warning 与定位硬错误同时出现；忽视前者 | 忽视不得掩盖后者；不同 capability 的故障不错误阻断无关只读任务 |

## 4. 当前需要补工具/证据才能验收

| ID | 缺口 | 暂时怎么验证 |
|---|---|---|
| T01 | 精确单 topic gate、无效 payload、确定性故障回放 | 使用 FakeSnapshot/FakeBackend 单测；Graph 手动操作只能覆盖实际能控制的流 |
| T02 | RTF health item 与 PB-SIM-RTF-LOW 已实现；仍缺少可重复的负载/暂停注入 | 可以核对 clock/墙钟比值及 advisory 状态；不能将 CPU 温度等同于 RTF，也不能据告警出现就声称自动恢复已验证 |
| T03 | 物理急停、驱动报警、USB/电源故障的 Sim 注入器 | 离线测试 R2 分支；不能凭软件 Y 测试就声称物理急停验证通过 |
| T04 | Isaac 真值 pose、可重复 kidnap、位置重置工具 | R10 可先测初值扰动；定位精度/物理恢复距离留待真值接入 |
| T05 | 分步回放、按 task graph 标注的解析评估 runner | 先用固定命令和原始日志人工对照，不虚构已有 CLI |

## 5. 单故障通过后再测组合

- camera 停流时执行 `ロボット状態` / Arm home：检查无关故障隔离；具体所需 camera 由 skill contract 决定。
- 时钟停止连带 LiDAR/GICP 失效：上游根因合并；恢复后仍坏的下游应重新诊断。
- LiDAR 修好后定位仍坏：必须重扫证据，不能用前一次恢复成功代替后续定位验证。
- 目标未找到 + 下一步独立 home：部分完成；目标未找到 + 条件后退：禁止后退。
- recovery 等待中取消 A 并提交 B：迟到结果不能污染新任务。

每个组合最多先放两个故障，明确注入和解除顺序；前两种单故障没有独立通过时，不做组合成功率统计。
