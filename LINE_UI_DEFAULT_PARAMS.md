# LINE_UI_DEFAULT_PARAMS.md - LINE版チュートリアル初期パラメータ（v1.4 Draft）

本ファイルは、LINE版チュートリアル UI（LINE_UI）で使用する

各種パラメータの初期値を定義する。

- 実運用では、`sys_params`／`shop_options` から上書き可能とする前提。
- ここに記載する値は「v1.4時点の暫定値」であり、運用しながら調整してよい。

---

## 1. 状態・タブ・プロファイル関連

```yaml
line_ui:

  # state_code ごとの初期タブ
  # （現時点では全て HOME を想定。将来的に変更可）
  default_tab_for_state:
    anonymous_no_store: "tab_home"
    anonymous_with_store: "tab_home"
    linked_with_store: "tab_home"
    store_line_redirect_only: "tab_home"

  # 1タブあたりの「大きめボタン」の最大個数（HOMEなど）
  max_main_slots_per_tab: 3

  # リッチメニュープロファイルのデフォルト割り当て（参考）
  # 実体は line_richmenu_profile_assignments で管理し、
  # ここは初期生成・ドキュメント用の定義とする。
  richmenu_profile_by_state:
    anonymous_no_store: "home_anonymous_no_store"
    anonymous_with_store: "home_anonymous_default_store"
    linked_with_store: "home_linked_default_store"
    store_line_redirect_only: "home_store_line_redirect"

```

## 2. 名寄せ・匿名店舗リンクTTL

```yaml
line_ui:

  # 名寄せ済みかどうかに応じて、AIの挙動を切り替える前提。
  # identity_token のTTLは AI_CORE / API_COMM 側の設定に準ずる。

  # 名寄せされていないLINEユーザーの店舗紐づけ情報を保持する日数（暫定値）
  anonymous_shop_link_retention_days: 365

  # 名寄せ完了後、直前の質問を自動で再実行するか
  auto_retry_after_identity: true

```

## 3. ローディング・レスポンス演出

```yaml
line_ui:

  # 「考え中…」メッセージを表示するかどうか
  show_thinking_indicator: true

  # API_CORE / API_COMM の処理時間予測がこのmsを超えそうなら、
  # ローディングメッセージを先に出す目安（暫定値）
  thinking_threshold_ms: 600

```

## 4. 解錠番号・セキュリティ関連

```yaml
line_ui:

  # 解錠番号表示の1時間あたりの最大回数（暫定値）
  unlock_code_max_attempts_per_hour: 5

  # 解錠番号を出す前に必ず予約日時を確認させるフローを有効にするか
  unlock_code_require_reservation_confirmation: true

  # 過去予約（本日より前）の解錠番号再表示を許可するか（通常false）
  unlock_code_allow_past_reservations: false

```

## 5. エラー時・設定不足時の挙動

```yaml
line_ui:

  # 予約系スロットで必須URLが存在しない場合の挙動
  # - "hide" → スロット自体を表示しない
  # - "show_placeholder" → 「オンライン予約は未設定です」的なガイドを出す
  slot_fallback_policy: "hide"

  # LINE_UIを店舗単位で無効化するかどうかのデフォルト（shop_optionsで上書き）
  default_line_ui_enabled: true

```

## 6. ログ・トラッキング関連（最小）

```yaml
line_ui:

  # LINE_UIの操作を sys_events に送るかどうか
  track_events_enabled: true

  # 将来用：イベント名のprefixなど
  event_name_prefix: "line_ui."

```

## 7. その他（将来拡張用のプレースホルダ）

```yaml
line_ui:

  # 将来のクーポンタブ有効フラグ
  coupon_tab_enabled: false

  # クーポン関連処理を有効にする最低条件（例：名寄せ済みのみ 等）
  coupon_require_identity: true

```
