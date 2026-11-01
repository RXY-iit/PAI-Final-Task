# Decision Model（Jev）を用いた Mission Agent

Isaac Sim 環境で稼働している ROS 2 ワークスペースから、Mission Agent とその周辺実装を抜き出したコピーです。Decision Model（Jev）を用いる判断層は、コード上では **JEV** / `JevStrengthenAgent` として実装されています。

自然言語の依頼を Mission として受け取り、**Perception Agent・ARM Agent・Navigation Agent** に Task を委派します。モデルは意味の解釈や計画の審査を担当し、型付きの Robot Skill と実行時の検証を通じてロボットを動かします。

このリポジトリの主目的は、設計と実装の対応を読めるようにすることです。元の実行環境の全ファイル、モデル重み、外部パッケージ、実行ログを含む完全な再現用配布物ではありません。

## 最初に読むもの

| 知りたいこと | 読む場所 |
| --- | --- |
| 責務分担、Task / Step / Skill、モデル判断の境界 | [アーキテクチャと設計方針](docs/architecture.ja.md) |
| 各 Agent の動作と現在の能力 | この README の「各 Agent の設計」 |
| JSONL の保存先、Agent ごとの追跡方法 | [ログと判断トレース](docs/logging.ja.md) |
| Task の種類と委派先 | [task_catalog.yaml](src/robot_agent/config/task_catalog.yaml) |
| 実行可能な Skill と完了条件・監督用質問 | [skill_registry.yaml](src/robot_skills/config/skill_registry.yaml)、[skill_catalog.v2.yaml](src/robot_skills/config/skill_catalog.v2.yaml)、[skill_extensions.yaml](src/robot_skills/config/skill_extensions.yaml) |

上記の日本語ドキュメントは、このコピーのコードと設計メモを照合して整理したものです。元の中国語メモを削除しても、ここから実装と設計資料を辿れます。

## 全体構成

```text
ユーザー / Dashboard
        │ 自然言語の依頼・確認への回答・キャンセル
        ▼
Mission Agent ───────── JEV 判断層
        │               意味解析・計画審査・例外時の判断
        ├── Perception Agent    観測・物体探索・属性 / OCR 確認
        ├── ARM Agent           登録済み姿勢・操作能力の判定
        └── Navigation Agent    目的地解決・経路審査・到達確認
                  │ TaskRefinement / SkillCall
                  ▼
        Mission の共有スケジューラ
                  │
              SkillClient
                  │
        Robot Skill executors
                  │
        ROS 2 / Nav2 / Isaac Sim
```

各領域 Agent は、現在は同じ Mission 実行系から呼ばれる Python クラスです。領域ごとの判断責務を分け、実行は共有スケジューラで直列に管理します。独立した複数の Agent プロセスを並列実行する構成ではありません。JEV アダプタも共有し、領域ごとの質問を渡します。

## 各 Agent の設計

### Mission Agent：依頼全体の管理

依頼を Task に分解し、順序、依存関係、担当 Agent、キャンセル、最終結果を管理します。Task の実行順が来た時点で担当 Agent の `refine()` を呼び、現在の状態に応じた Skill 列を受け取ります。途中の移動や感知結果を、後続 Task の判断へ反映するためです。

Primary モードでは `command_parse` で Task の構造を選び、`command_bind` で原文の範囲・引数・依存関係を結び付け、`plan_review` で計画を審査します。数値や単位、依存条件はコードで検証します。状態照会、既知の目的地一覧、相対移動・旋回など、Mission 自身が担当する Task もあります。

- 実装：[mission_agent.py](src/robot_agent/robot_agent/agents/mission_agent.py)（`run`、`_expand_task`、`_complete_task`）
- 原文との対応付け：[command_grounding.py](src/robot_agent/robot_agent/command_grounding.py)
- 型と状態：[schemas.py](src/robot_agent/robot_agent/schemas.py)、[domain.py](src/robot_agent/robot_agent/agents/domain.py)
- JEV 接続・質問・トレース：[jev_agent.py](src/robot_agent/robot_agent/jev_agent.py)

### Perception Agent：観測と認識の契約

`observe_scene` と `search_object` を担当します。依頼から「何を見るか」「どの属性を満たすか」「AND / OR のどちらか」を整理し、利用可能な検出器、色判定、OCR に変換します。複数の確認で観測画像を共有し、対象と色が同じ物体インスタンスに属するかなど、証拠の対応も確認します。

`perception_command_parse` と `perception_plan_review` で意味と実行能力を確認し、必要に応じて視点を解決します。探索結果は `FOUND`、`NOT_FOUND`、`PARTIAL` などを区別します。画像を取得できただけで対象を発見したとは扱いません。自由な画像説明や危険判定は、対応するバックエンドが未整備のため、観測だけで成功したことにはしません。

- 領域ロジック：[perception_agent.py](src/robot_agent/robot_agent/agents/perception_agent.py)、[perception_plan.py](src/robot_agent/robot_agent/agents/perception_plan.py)
- 実行：[observation.py](src/robot_skills/robot_skills/executors/observation.py)、[perception.py](src/robot_skills/robot_skills/executors/perception.py)
- 設定：[perception_targets.yaml](src/robot_skills/config/perception_targets.yaml)、[perception_models.yaml](src/robot_skills/config/perception_models.yaml)、[camera_scan_profiles.yaml](src/robot_skills/config/camera_scan_profiles.yaml)

### ARM Agent：検証できる操作だけを委派

現在の自律 Task は、主に `set_arm_pose` による登録済み姿勢・ジェスチャーです。停止を確認してから姿勢を実行し、実行側で gateway の状態や完了を確認します。JEV モードでは `arm_task_review` が姿勢・繰り返しの意図を審査します。

`interact_with_object`（接触・押下）、`manipulate_object`（把持・配置・解放）、`move_lift` は Task として認識しますが、現在の `refine()` は自律完了の検証が不足するため `UNSUPPORTED` を返します。旧低レベル API や互換用プランの存在は、自律操作の完成を意味しません。

- 領域ロジック：[arm_agent.py](src/robot_agent/robot_agent/agents/arm_agent.py)
- 実行：[arm.py](src/robot_skills/robot_skills/executors/arm.py)、[lift.py](src/robot_skills/robot_skills/executors/lift.py)
- 能力判定：[capabilities.py](src/robot_agent/robot_agent/capabilities.py)

Arm gateway や Touch Anything の外部実装は、このコピーには含まれていません。

### Navigation Agent：目的地・経路・到達証拠を分離

`navigate_to` と `follow_waypoint_list` を担当します。通常の目的地は semantic map の登録情報から解決し、同名候補は人に確認します。未知の地点、未校正の座標、未対応の接近方向には、推測した移動先を与えません。

通常のナビゲーションでは、`navigation_command_parse`、意味レベルの `navigation_plan_review`、Nav2 が計算した経路に対する `navigation_plan_review` を使います。到達後に対象の同一性確認が必要なら、Perception の子 Task を委派します。位置への到達と視覚的な確認は別々に記録します。登録済み waypoint 列は、通常の前提チェックを利用する専用分岐で順番に実行します。

- 領域ロジック：[navigation_agent.py](src/robot_agent/robot_agent/agents/navigation_agent.py)、[navigation_plan.py](src/robot_agent/robot_agent/agents/navigation_plan.py)
- 実行：[navigation.py](src/robot_skills/robot_skills/executors/navigation.py)
- 地図の読み取り：[semantic_map.py](src/robot_skills/robot_skills/semantic_map.py)
- 同梱設定例：[semantic_map.yaml](maps/sim_real_lab_001/semantic_map.yaml)、[navigation_annotations.yaml](maps/sim_real_lab_001/navigation_annotations.yaml)、[waypoint_list_a.yaml](src/robot_skills/config/waypoint_list_a.yaml)

## 判断モード

| `agent_mode` | 動作 |
| --- | --- |
| `classic` | 既存の parser とコードの実行方針を使用 |
| `jev_shadow` | Classic の実行を保ち、JEV の判断を比較用に記録 |
| `jev_primary` | 検証済みの JEV 判断を、定義された実行分岐に採用 |

判断モードは Mission 開始時に固定されます。実行環境を表す `mode=sim / real / dry_run` とは別の設定です。API 未設定時の local fallback は、実際のモデル呼び出しとログ上で区別します。

## JSONL はどこにあるか

**このコピーには実行済み JSONL は含まれていません。** 通常の Dashboard 起動では、次のファイルが実行時に生成されます。Agent ごとに独立した JSONL を作る方式ではなく、共有ログを Task の owner や stage で追跡します。

| ファイル（`ROBOT_WS` からの相対パス） | 内容 | 書き込み元 |
| --- | --- | --- |
| `debug-output/sessions/<session_id>/agent/skill_events.jsonl` | Mission、Task 委派、領域判断、Skill 結果、完了イベント | [mission_log.py](src/robot_agent/robot_agent/mission_log.py) |
| `debug-output/sessions/<session_id>/agent/jev_calls.jsonl` | JEV の質問、リクエスト、応答、遅延、provider | [jev_agent.py](src/robot_agent/robot_agent/jev_agent.py) の `_write_call_log()` |
| `debug-output/sessions/<session_id>/sim_health.jsonl` | シミュレーション時刻・センサーストリームの観測 | [health_monitor.py](src/robot_agent/robot_agent/nodes/health_monitor.py) |

保存先の上書き、単体起動時の旧パス、各 Agent の抽出例は [ログと判断トレース](docs/logging.ja.md) を参照してください。存在しないログへのリンクの代わりに、生成元のコードをリンクしています。

## 周辺実装と実行環境

| 場所 | 役割 |
| --- | --- |
| [src/robot_agent](src/robot_agent) | Agent、ROS ノード、監視・Recovery、Dashboard |
| [src/robot_skills](src/robot_skills) | 型付き Skill、実行器、能力・完了契約 |
| [src/robot_sim](src/robot_sim) | Isaac Sim との接続、センサー・駆動用スクリプト |
| [src/nav_pkg](src/nav_pkg)、[src/localization_pkg](src/localization_pkg) | Nav2 と自己位置推定 |
| [src/safety_layer](src/safety_layer) | 実行時の安全制御 |
| [maps](maps)、[glim_config_nav2](glim_config_nav2)、[rviz](rviz) | 地図・SLAM・表示設定 |
| [Dashboard UI](src/robot_agent/web/src) | Mission / Task の進捗、確認、キャンセル、判断履歴 |

[agent_dashboard.launch.py](src/robot_agent/launch/agent_dashboard.launch.py) が Agent と Dashboard の起動入口です。既定値には元環境の `mode=real`、workspace パス、地図 ID が残っています。Isaac Sim に接続する場合は `mode:=sim` と実際の `robot_ws`・`map_id` を指定し、センサー、TF、自己位置推定、Nav2 などを別途整える必要があります。この launch 単体では Isaac Sim を含む全体環境を構築しません。

モデル接続は `JEV_API_KEY`、`JEV_API_URL`、`JEV_MODEL` を環境変数で設定します。API の形式・既定値は [JEV アダプタ](src/robot_agent/robot_agent/jev_agent.py) を参照してください。認証情報と実行ログはリポジトリに含めません。

## 実装の検証を読む

設計上の境界は、[Task 委派テスト](src/robot_agent/test/test_domain_tasks.py)、[原文バインドのテスト](src/robot_agent/test/test_command_grounding.py)、[Perception のテスト](src/robot_agent/test/test_perception_domain_plan.py)、[Navigation のテスト](src/robot_agent/test/test_navigation_domain_plan.py)、[Arm の証拠テスト](src/robot_skills/test/test_arm_candidate_evidence.py) でも確認できます。

これらにはオフライン・代替実装を使うテストが含まれます。テストコードの存在や dry-run の成功を、実 API と Isaac Sim を接続した運動・認識の検証結果とは扱いません。一部テストや起動設定は、このコピーに含まれない元環境のツール・資産を参照します。
