# HELP_UI_DEFAULT_PARAMS.md

v1.4 初期パラメータセット（すべて可変前提）

> このファイルは「運用開始時の初期値」をまとめたものです。
仕様ではなく、いつでも本部管理 UI から変更してよい値とします。
> 

---

## 1. メディア関連

- `help_ui.max_media_per_message`
    - 説明：1メッセージあたりの最大添付数
    - 初期値：3
- `help_ui.max_media_per_session`
    - 説明：1セッションあたりの最大添付数
    - 初期値：10
- `help_ui.media_retention_days`
    - 説明：メディアファイルの保存日数
    - 初期値：30
- `help_ui.allow_video`
    - 説明：動画添付を許可するか（true: 許可 / false: 禁止）
    - 初期値：true
- `help_ui.image_max_long_edge_px`
    - 説明：画像リサイズ時の長辺最大ピクセル
    - 初期値：1920
- `help_ui.video_max_seconds`
    - 説明：動画1件あたりの最大秒数
    - 初期値：30

---

## 2. 本人確認・セッション関連

- `help_ui.identity_retry_limit`
    - 説明：本人確認（STORES特定）リトライ上限回数
    - 初期値：3
- `help_ui.identity_token_ttl_hours`
    - 説明：identity_token（本人確認済み状態）の有効期間（時間）
    - 初期値：72
- `help_ui.session_idle_timeout_minutes`
    - 説明：無操作でセッションを終了扱いにするまでの時間（フロント側の目安）
    - 初期値：30

---

## 3. エスカレーション・ステート関連

- `help_ui.max_turns_before_escalation_suggest`
    - 説明：同じ intent で解決しない場合に「人への引き継ぎ」を提案するまでのターン数
    - 初期値：4
- `help_ui.max_turns_before_force_escalation`
    - 説明：「まだ解決していない」が続いた場合に、強制的にエスカレーションに誘導するターン数
    - 初期値：7

---

## 4. 時間帯・夜間対応

- `help_ui.quiet_hours_start`
    - 説明：夜間帯開始時刻（ローカル時間、"HH:MM"）
    - 初期値："22:00"
- `help_ui.quiet_hours_end`
    - 説明：夜間帯終了時刻（ローカル時間、"HH:MM"）
    - 初期値："08:00"

---

## 5. 障害フラグ・運用スイッチ

- `help_ui.force_ai_off`
    - 説明：AI_CORE を強制停止するフラグ（true の場合、常に fallback）
    - 初期値：false
- `help_ui.force_api_comm_off`
    - 説明：API_COMM（STORES等）を強制停止するフラグ
    - 初期値：false

---

## 6. 外部URL

- `help_ui.stores_mypage_url`
    - 説明：STORES マイページのベースURL
    - 初期値："https://coubic.com/u"

※店舗別URLに切り替えたい場合は、scope=shop のレコードで上書きします。
