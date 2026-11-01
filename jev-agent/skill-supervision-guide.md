# Robot Skill 异常监督：统一入口

更新：2026-10-12。这里的 Robot Skill 是可调用执行能力，不是 Agent Skills 的 SKILL.md 包规范。

## 查哪里

唯一问题内容源：[skill_catalog.v2.yaml](../../src/robot_skills/config/skill_catalog.v2.yaml)。

| 配置 | 作用 |
| --- | --- |
| common_supervision_questions | 所有领域共享的完成一致性、故障分类、Recovery、依赖、人工与后续动作问题 |
| robot_skills[].supervision.questions | 当前 skill 的专属证据判断问题；发送为 skill_<key>，type=noul |
| robot_skills[].supervision.evidence_fields | 允许投影到请求里的 metrics 字段；没有值时不可推断为正常 |
| supervision_profiles.capability | 三个只读 check_*_capability 的共用问题 |
| skill_extensions.yaml | alias 和 composite 的继承关系，不另写问题文本 |

之前三个 perception skill 有专属问题，其他 skill 只有通用分类及前置条件。现在补齐 19 个基础 skill，连同运行时的 alias、检查和观察组合能力共 34 项都有监督契约。SkillCatalog.validate_registry 会报告缺少 questions/evidence_fields。

move_relative_xy 使用 YAML anchor 共享平移契约。search_at_place 合并采集和检测问题；若存在 failed_child，优先用具体失败子 skill。别名从原始 skill 继承，不维护另一份相同文本。

## 谁使用它

JevStrengthenAgent.supervise_step 从 catalog 读取通用问题和当前契约，经 _decide 发送。await_robot_auto_mode + NEEDS_HUMAN 时才保留 ask_human_enable_auto 选项，其余调用删去。代码只负责条件选择和确定性边界，不再内嵌另一份通用 API 问题。

stage 根据 owner 为 navigation_step_supervision / perception_step_supervision / step_supervision；不意味着每个成功 skill 都调用。Mission 的 _should_call_jev_supervision 控制异常触发；正常 NOT_FOUND/PARTIAL 是目标结果，不是服务异常。

JEV 只能建议。取消、硬安全门、依赖和动作权限仍由执行代码决定。hard_safety=false 不是“失败可以无条件跳过”，也不是“这个前置条件不重要”。

## 扩展模板

```yaml
supervision:
  questions:
    result_supported: Does fresh measured feedback support the reported result?
    failure_explained: Does evidence identify the failure rather than only a timeout?
  evidence_fields: [actual_value, target_value, source_stamp, source_age_sec, reason]
```

新增 skill 时同时定义 args_schema、preconditions、effects、failure_semantics、supervision，并登记 registry 和 executor。问题要对应真实可能提供的结果证据，不要求模型脑补传感器数据。

注意：本轮补齐的是问题契约，不代表所有底层 executor 已提供模板里每个可选指标。有些字段暂缺，此时必须 null；下一轮可按实际失败日志补测量来源。common safety_summary 和 failure code 仍保留，不能只依赖 metrics 名称。

回归入口：src/robot_agent/test/test_navigation_domain_plan.py::test_every_registered_skill_has_unified_supervision。它验证每个注册 skill 的问题确实进入 API 请求构造，不只是文档存在。

原始请求见 jev_calls.jsonl 的 redacted request_body.questions / state；Dashboard Trace 是摘要。非 live fallback 不能作为真实 JEV 判断效果样本。
