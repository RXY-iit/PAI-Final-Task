# アーキテクチャと設計方針

[README に戻る](../README.md)

## 設計の中心

自然言語の理解とロボットの実行を、明確な契約でつなぎます。Mission Agent はユーザーの目的と依存関係を保持し、各領域 Agent はその Task をどのように実行するかを決めます。JEV は定義された選択肢への判断を返し、実行側がパラメータ、状態、完了証拠を検証します。

この分担により、たとえば物体探索の視点や検出器を変更しても、Mission 側でカメラ操作の全手順を組み直す必要がありません。領域固有の処理と失敗理由を、Task の結果として上位へ返せます。

## Mission / Task / Step / Robot Skill

| 概念 | 意味 | 実装上の入口 |
| --- | --- | --- |
| Mission | 一回のユーザー依頼と、その全体の達成状態 | [MissionState](../src/robot_agent/robot_agent/schemas.py) |
| Task | 独立した目標・完了条件を持ち、一つの owner が責任を持つ単位 | [TaskCatalog](../src/robot_agent/robot_agent/task_catalog.py) |
| Step | Task 内で実行する一回の Robot Skill 呼び出し | [SkillCall](../src/robot_agent/robot_agent/schemas.py) |
| Robot Skill | 型付き引数、前提条件、結果を持つ実行能力。制限された複合処理も含む | [SkillCatalog](../src/robot_skills/robot_skills/skill_catalog.py) |
| TaskRefinement | 領域 Agent が返す Skill 列、判断 trace、質問、outcome、選択肢 | [domain.py](../src/robot_agent/robot_agent/agents/domain.py) |

`SkillCall.kind` は `task`、通常の Skill、内部完了マーカー `task_end` を区別します。`task_end` 自体はロボットへ送る Skill ではありません。`mission_id`、`task_id`、`parent_task_id`、`step_id` で委派と実行の対応を追います。

たとえば複数視点で対象を探す場合、ユーザーの目的は一つの `search_object` Task です。画像枚数をそのまま Mission の Task 数にはしません。

## 実行の流れ

1. Mission が依頼を受け取り、今回の `agent_mode` を固定する。
2. Primary では JEV が Task 構造を判断し、`command_bind` で原文と引数・依存関係を結び付ける。コードが境界、数値、単位、条件を検証する。
3. `plan_review` でユーザーの目的が欠けていないかを審査する。Classic の計画との比較もこの段階で行う。
4. Task の実行順が来たら、owner の `refine()` に委派する。領域側は現在の能力・目標・証拠に基づき具体的な Skill 列を返す。
5. Mission の共有スケジューラが [SkillClient](../src/robot_agent/robot_agent/skill_client.py) 経由で実行する。実行中もキャンセルや必要な安全・状態条件を確認する。
6. `task_end` で内部 Step と子 Task の結果を集約し、`task_feedback` を出す。
7. 依存する後続 Task の可否を判断し、Mission 全体の結果をまとめる。証拠不足は質問、未達、失敗として残す。

実装は [MissionAgent](../src/robot_agent/robot_agent/agents/mission_agent.py) の `run()`、`_expand_task()`、`_complete_task()` に集約されています。Task の細化を実行直前まで遅らせる構造であり、任意の並列 TaskRunner や無制限の動的再計画を実装したものではありません。

## 例：目的地への移動と対象の確認

「登録済みの棚へ移動する」という依頼では、Mission が `navigate_to` を Navigation に渡します。Navigation は地図内の目的地、接近方向、適用ルールを解決し、停止・AUTO・自己位置推定の前提確認と `goto_pose` を構成します。

Nav2 の計算経路が得られた時点で、目標とルールに整合するかを審査します。地図で視覚確認が要求されている場合、到達後に Perception の `search_object` 子 Task を実行します。OCR が対象ラベルを確認できなかった場合も、「位置には到達した」という事実は保持し、視覚確認の未達を別に報告します。

同梱地図で任意の棚ラベルの校正が完了しているという意味ではありません。目的地と認識契約は、その環境で実際に使用する semantic map に設定します。

## JEV に判断させる範囲

[JevStrengthenAgent](../src/robot_agent/robot_agent/jev_agent.py) は各段階に必要な状態と質問を組み立て、選択式の回答を検証します。モデルに ROS コマンドや任意の実行コードを直接生成・実行させる構造ではありません。

| 判断の単位 | 主な stage / 定義場所 |
| --- | --- |
| 依頼の構造と原文への対応付け | `command_parse`、`command_bind`：[jev_agent.py](../src/robot_agent/robot_agent/jev_agent.py)、[command_grounding.py](../src/robot_agent/robot_agent/command_grounding.py) |
| Mission の計画と Task 結果 | `plan_review`、`task_outcome_review`：[mission_agent.py](../src/robot_agent/robot_agent/agents/mission_agent.py) |
| 観測の意味と能力 | `perception_view_parse`、`perception_command_parse`、`perception_plan_review`：[perception_agent.py](../src/robot_agent/robot_agent/agents/perception_agent.py) |
| 目的地と経路 | `navigation_command_parse`、`navigation_plan_review`：[navigation_agent.py](../src/robot_agent/robot_agent/agents/navigation_agent.py) |
| 登録姿勢の意図 | `arm_task_review`：[arm_agent.py](../src/robot_agent/robot_agent/agents/arm_agent.py) |
| Skill 失敗時の監督 | `step_supervision`、領域別の supervision：[jev_agent.py](../src/robot_agent/robot_agent/jev_agent.py)、[skill_catalog.v2.yaml](../src/robot_skills/config/skill_catalog.v2.yaml) |

毎制御周期や、正常終了した全 Skill でモデルを呼び出す設計ではありません。意味解釈、計画審査、結果・例外処理など、判断が必要な境界で使います。質問は Python / YAML で定義され、JSONL はその実行記録です。

## 完了を証拠で判定する

- **Perception**：画像取得、対象の検出、要求された属性の成立を区別する。`NOT_FOUND` と不十分な観測による `PARTIAL` も区別する。
- **Navigation**：`navigation_arrival` と `visual_confirmation` を分ける。Nav2 の厳密到達、Task 許容差、degraded、dry-run などの `completion_class` を混同しない。
- **ARM**：コマンドの受付だけでは物理的な完了としない。現在の自律能力を超える接触・把持・昇降は、能力不足を返す。
- **条件付き Task**：指定した依存元の探索証拠を使う。`FOUND` と目標充足が確認できない条件分岐に、そのまま移動を許可しない。

モデルの `execute` 判断が返っても、AUTO、安全、キャンセル、自己位置推定、実行器の制約を解除しません。Nav2 の経路審査も、足跡の全領域での衝突検証や実行中の全再計画を保証するものではありません。

## 人への確認と Recovery

曖昧な目的地、能力不足、審査の保留は、元の Mission と Task に紐づく質問として返します。再開操作は `mission_id` と `wait_id` に結び付け、古い回答で別の実行が復活しないようにします。キャンセルはモデルへの問い合わせを待たずに扱います。

環境・機器の異常は、[Recovery Agent](../src/robot_agent/robot_agent/nodes/recovery_agent.py) と [playbook](../src/robot_agent/config/recovery/playbooks) が補助します。これは三つの領域 Agent とは別の復旧機構です。Mission からの引き渡しは [recovery_supervision.py](../src/robot_agent/robot_agent/recovery_supervision.py) と Mission 実装、状態観測は [health_monitor.py](../src/robot_agent/robot_agent/nodes/health_monitor.py) を参照してください。

## 拡張時に変更する場所

| 変更内容 | 主な変更先 |
| --- | --- |
| 新しいユーザー Task と owner | [task_catalog.yaml](../src/robot_agent/config/task_catalog.yaml) と担当 Agent の `refine()` |
| 新しい実行能力 | [skill_registry.yaml](../src/robot_skills/config/skill_registry.yaml)、[skill_extensions.yaml](../src/robot_skills/config/skill_extensions.yaml)、対応する [executor](../src/robot_skills/robot_skills/executors) |
| 前提条件・完了意味・監督用質問 | [skill_catalog.v2.yaml](../src/robot_skills/config/skill_catalog.v2.yaml) |
| 検出対象・モデル・観測姿勢 | [perception_targets.yaml](../src/robot_skills/config/perception_targets.yaml)、[perception_models.yaml](../src/robot_skills/config/perception_models.yaml)、[sim_camera_poses.yaml](../src/robot_skills/config/sim_camera_poses.yaml) |
| 目的地・接近姿勢・到達後の視覚確認 | 使用する地図の `semantic_map.yaml`：[同梱例](../maps/sim_real_lab_001/semantic_map.yaml) |

Task 名や Skill 名を追加するだけでなく、実際の実行能力、キャンセル、完了証拠、未対応時の結果まで揃えることを設計上の単位とします。
