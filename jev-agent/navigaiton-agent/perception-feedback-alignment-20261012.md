# Navigation Agent：参考 Perception 分层的修正建议

实施更新：2026-10-12 已按此方向加入目标解析、语义/实际路径审查、异常监督、同 Task 澄清和到达/身份分离反馈。当前实现及明确未支持的路线约束见 [README](README.md)。以下保留原设计依据，不将所有远期能力视为已经实现。

## 分层建议

| 阶段 | 负责 | 不负责 |
| --- | --- | --- |
| navigation_command_parse | 目标身份、区域/点、关系、到达方向、用户限制和歧义 | 重排整个 Mission；编造坐标 |
| navigation_plan_review | 候选目标、路线规则、狭窄段策略、到达契约和视觉确认要求 | 覆盖碰撞/定位/控制权硬门 |
| navigation_step_supervision | 确定性局部策略未处理的异常、失败段和已尝试动作 | 每个成功反馈都调用 API；反复放宽容差 |

Mission 看 navigate_to 的进度和结果，Navigation 内部保留有序 waypoint 与 Perception 子任务，owner/id、父子关系、取消和人工回复仍属于同一 Mission。

## 不能直接照搬的部分

- 感知可 all/any 聚合，路线却有几何依赖和顺序；前一段失败不能无条件继续下一段。
- 仅测到红色可 PARTIAL；目标门不可达不能静默换另一扇门并报成功。
- 解析不用整份健康快照，但定位、障碍物、控制权在执行前/过程中必须持续验证。
- 视觉描述没有发现危险不等于导航安全放行。

## 当前缺口与下一轮修改

修改前只有 `nav_target_review`。其中 `route_policy=existing_validated_nav2_policy` 只是策略标识，不证明本次 Nav2 路径已经生成并检查。现已改成分阶段问题，并在 goto_pose 路径计算后传入实际路径；仍不等于自由几何禁区已支持。

1. parse 输出 destination_candidates、goal_region/pose、approach_requirement、rule_refs；候选来自语义地图，真实歧义交人工。
2. plan_review 分语义目标/规则审查与 Nav2 路径出来后的路径审查。规则有版本、范围、优先级、有效期。
3. 只在目标、规则、路径发生有意义变化或异常时重审，不跟每次局部规划调用 JEV。
4. 明确 shelf 到位与 OCR 身份确认的 goal contract。视觉失败不能抹掉已到位事实，也不能假称确认了货架。
5. 人工选择绑定 mission_id + wait_id 在当前 Task 续行。目前 Navigation 部分候选仍是新 `go to ...` 命令，需像本轮 Perception 一样迁移。
6. 缺 semantic_map 不可称目标已校准。当前 refine 可进入 resolve_semantic_target，由执行端失败关闭；可以前移能力缺失提示，但不删除执行端校验。
7. 路径、规则摘要和分段动作通过 ID 引用，避免证据复制。Nav2 严格成功、语义容差成功、降级到达、视觉确认成功仍分别统计。

## 验收建议

同名两个门仅询问目标；维修规则阻断而非静默改目标；窄道按批准策略；到货架但 OCR NOT_FOUND 保留到位事实；恢复后位置变化重新规划；人工确认/取消与迟到回调不能复活旧任务。
