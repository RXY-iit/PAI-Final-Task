# ログと判断トレース

[README に戻る](../README.md)

## 通常の保存構造

このコピーに実行済み `.jsonl` はありません。通常の [Dashboard launch](../src/robot_agent/launch/agent_dashboard.launch.py) は共通の `ROBOT_AGENT_SESSION_ID` を各ノードに渡します。[session.py](../src/robot_agent/robot_agent/session.py) が扱うセッションディレクトリは次の構造です。

```text
<ROBOT_WS>/debug-output/sessions/<session_id>/
├── agent/
│   ├── skill_events.jsonl
│   └── jev_calls.jsonl
└── sim_health.jsonl
```

`session_id` の自動生成形式は UTC 時刻とランダムな識別子です。`ROBOT_WS` が未設定なら、ロガーはプロセスの作業ディレクトリを基準にします。`sim_health.jsonl` はシミュレーションの監視処理が書き込みます。設定や実行内容により、すべてのファイルが必ず作られるとは限りません。

## 三種類の記録

| ファイル | 主なフィールドと用途 | 実装 |
| --- | --- | --- |
| `skill_events.jsonl` | `time`、`mission_id`、`status`、`type`、`payload`。委派、判断採用、Skill 結果、Mission 完了を追う | [MissionLogger](../src/robot_agent/robot_agent/mission_log.py)、[MissionAgent._event](../src/robot_agent/robot_agent/agents/mission_agent.py) |
| `jev_calls.jsonl` | `stage`、`purpose`、`questions_sha256`、`provider`、`model`、`request`、`response_raw`、`latency_ms`。質問と実際の応答を調べる | [JevStrengthenAgent._write_call_log](../src/robot_agent/robot_agent/jev_agent.py) |
| `sim_health.jsonl` | `at`、`clock`、`streams`。時計の停止やセンサーデータの鮮度など、実行環境の証拠を調べる | [health_monitor.py](../src/robot_agent/robot_agent/nodes/health_monitor.py) |

AgentNode 経由のイベントには `ros_time_sec` も追加されます。`skill_events.jsonl` は heartbeat を除外し、約 10 MiB を超えると次の書き込み時に `.1`〜`.3` へローテーションします。`sim_health.jsonl` もサイズに応じて `.1` に退避します。調査時には該当する世代も確認してください。

## Agent ごとの対応

四つの Agent がそれぞれ専用 JSONL を持つわけではありません。実行の所属はイベントの Task / owner、モデルへの質問は stage で見分けます。

| Agent | イベントで見る owner | JEV の主な stage |
| --- | --- | --- |
| Mission | `mission` と Mission 全体のイベント | `command_parse`、`command_bind`、`plan_review`、`task_outcome_review`、`step_supervision` |
| Perception | `perception` | `perception_view_parse`、`perception_command_parse`、`perception_plan_review`、`perception_step_supervision` |
| ARM | `arm` | `arm_task_review`、共通の `step_supervision` |
| Navigation | `navigation` | `navigation_command_parse`、`navigation_plan_review`、`navigation_step_supervision` |

supervision は対象となる失敗などで呼ばれるため、正常な実行で記録がないことは異常ではありません。`classic` ではモデル判断のログがない場合があります。

`task_delegated` → `task_plan_created` → `skill_finished` → `task_feedback` を辿ると、Task がどの Skill に展開され、何を根拠に完了したかが分かります。子 Task は `parent_task_id` で関連付けます。`domain_decision` は owner と `trace` を持ち、領域の判断を確認できます。

raw の `jev_calls.jsonl` はトップレベルに共通の `mission_id` / `call_id` を保存する形式ではありません。厳密な所属の確認にはイベント側と MissionState の `jev_trace` を使い、raw 応答は時刻・stage・request 内の文脈も照合します。raw ログだけで全レコードを ID 結合できるとは限りません。

## 抽出例

以下はログを生成した元環境で使う読み取り例です。`RUN` は実在するセッションに置き換えます。`jq` が必要です。

```sh
RUN="debug-output/sessions/<session_id>"

# Mission 単位でイベントを読む
jq -c 'select(.mission_id == "<mission_id>")' "$RUN/agent/skill_events.jsonl"

# Perception の委派・判断・フィードバックを読む
jq -c 'select(.payload.owner == "perception")' "$RUN/agent/skill_events.jsonl"

# 上で得た Task ID から Skill 結果を含めて追う
jq -c 'select(.payload.task_id == "<task_id>")' "$RUN/agent/skill_events.jsonl"

# Navigation の質問と応答を読む
jq -c 'select((.stage // "") | startswith("navigation_"))' "$RUN/agent/jev_calls.jsonl"

# ARM の姿勢審査を読む
jq -c 'select(.stage == "arm_task_review")' "$RUN/agent/jev_calls.jsonl"
```

owner の抽出は、そのフィールドを持つイベントだけが対象です。すべての関連 Skill 結果を調べる場合は Task ID で追います。Perception のモデル判断なら `perception_`、Mission の計画審査なら `plan_review` を同様に抽出できます。

## 保存先の例外と単体実行

- `JEV_CALL_LOG_PATH` を指定すると、JEV の raw ログをそのファイルへ保存します。
- それがなく `ROBOT_AGENT_SESSION_ID` があれば、通常のセッション内 `agent/jev_calls.jsonl` に保存します。
- どちらもなければ、`JEV_LOG_ROOT`（既定値 `debug-output/agent`）配下の最新ディレクトリ、または新しい時刻ディレクトリを使います。旧メモにある `debug-output/agent/<timestamp>/jev_calls.jsonl` はこの経路です。
- `MissionLogger` に既定値以外の `root` を直接渡す場合、イベントログは `<root>/<timestamp>/skill_events.jsonl` になります。

[agent_dry_run.py](../src/robot_agent/robot_agent/nodes/agent_dry_run.py) は最終 MissionState を標準出力へ出すツールです。AgentNode のイベントロガーを接続していないため、実行すれば上記のイベント JSONL がすべて揃うという意味ではありません。

## 判断と実行を読み違えないために

`provider=live`、`local`、`api_error` と、trace の `fallback_used`・`adopted_decision` を併せて確認します。raw 応答が返ったことと、その判断が検証されて実行に採用されたことは別です。`jev_shadow` の判断も実行採用とは区別します。

JEV ロガーは API key などのキー名を伏せ、大きな画像文字列を除外・切り詰めます。ただしユーザーの依頼、場所の名称、パス、モデル応答の自由文を完全に匿名化する仕組みではありません。共有ログを追加する場合は内容を確認してください。現状では [`.gitignore`](../.gitignore) により実行ログとローカル認証情報を公開対象から外しています。
