# SHOP_PORTAL_DEFAULT_PARAMS (v1.1 Draft)

```yaml
# NOTE:
# - key: sys_params.key に保存するキー名
# - type: 値の型 (int/string/bool/time/json)
# - default: 初期値
# - scope: global / shop
# - description: 説明

params:
  # ===== ダッシュボード・一覧 =====
  - key: "shop_portal.dashboard_default_range_days"
    type: "int"
    default: 7
    scope: "global"
    description: "オーナーダッシュボードで集計するデフォルト期間（日数）"

  - key: "shop_portal.announcement_max_items"
    type: "int"
    default: 5
    scope: "global"
    description: "ダッシュボードに表示するお知らせの最大件数"

  - key: "shop_portal.tickets_page_size"
    type: "int"
    default: 20
    scope: "global"
    description: "問い合わせ一覧（tickets）の1ページあたり表示件数"

  - key: "shop_portal.analytics_default_range_days"
    type: "int"
    default: 30
    scope: "global"
    description: "簡易売上分析のデフォルト対象期間（日数）"

  # ===== 備品・申請 =====
  - key: "shop_portal.supply_order_cutoff_day"
    type: "int"
    default: 25
    scope: "global"
    description: "当月請求に含める備品発注の締め日（この日までの発注を当月扱い）"

  - key: "shop_portal.enable_supply_order"
    type: "bool"
    default: true
    scope: "global"
    description: "SHOP_PORTAL で備品発注機能を有効にするか"

  - key: "shop_portal.supply_order_max_quantity_per_item"
    type: "int"
    default: 200
    scope: "global"
    description: "1回の発注で1品目あたり許可する最大数量の目安"

  # ===== オーナーAI相談（owner_ai） =====
  - key: "shop_portal.owner_ai_temperature"
    type: "float"
    default: 0.4
    scope: "global"
    description: "オーナー向けAIのサンプリング温度（0〜1）。高いほど表現が多様になる。"

  - key: "shop_portal.owner_ai_max_turns_before_escalation_suggest"
    type: "int"
    default: 5
    scope: "global"
    description: "同一テーマでAI応答が続いたとき、本部への相談を提案するまでのターン数"

  - key: "shop_portal.owner_ai_max_turns_before_force_escalation"
    type: "int"
    default: 8
    scope: "global"
    description: "一定以上解決しない場合に、強制的にHQにエスカレーションするターン数"

  - key: "shop_portal.owner_ai_quiet_hours_start"
    type: "time"
    default: "22:00"
    scope: "global"
    description: "夜間（深夜帯）開始時刻。深夜帯はAIからの即時回答を抑制する場合に使用。"

  - key: "shop_portal.owner_ai_quiet_hours_end"
    type: "time"
    default: "08:00"
    scope: "global"
    description: "夜間（深夜帯）終了時刻。"

  # ===== MTG 予約 =====
  - key: "shop_portal.meeting_slots_visible_days_ahead"
    type: "int"
    default: 30
    scope: "global"
    description: "本部ミーティングの空き枠を何日先まで表示するか"

  - key: "shop_portal.meeting_request_notify_chatwork"
    type: "bool"
    default: true
    scope: "global"
    description: "ミーティングリクエスト時に Chatwork 通知を送るか"

  - key: "shop_portal.meeting_request_notify_email"
    type: "bool"
    default: true
    scope: "global"
    description: "ミーティングリクエスト時にオーナー宛のメール通知を送るか"

  # ===== デザイン・キャンペーン関連 =====
  - key: "shop_portal.design.dr_code_prefix"
    type: "string"
    default: "DR-"
    scope: "global"
    description: "デザイン依頼ID（DR-ID）のプレフィックス"

  - key: "shop_portal.design.cr_code_prefix"
    type: "string"
    default: "CR-"
    scope: "global"
    description: "キャンペーン申請ID（CR-ID）のプレフィックス"

  - key: "shop_portal.design_designerA_offer_price_zero"
    type: "bool"
    default: true
    scope: "global"
    description: "デザイナーA（テンプレバナー）案件をオーナー請求なしとするか"

  - key: "shop_portal.design_default_estimate_expire_days"
    type: "int"
    default: 14
    scope: "global"
    description: "デザイン見積提示から何日で自動失効とみなすか"

  - key: "shop_portal.design_notify_chatwork_on_status"
    type: "json"
    default: '["approved","in_production","delivered"]'
    scope: "global"
    description: "どのステータス遷移で Chatwork 通知を送るか（配列で指定）。"

  # ===== オーナー情報・編集 =====
  - key: "shop_portal.owner_profile_editable_until_complete"
    type: "bool"
    default: true
    scope: "global"
    description: "オーナー情報を is_profile_complete=false の間のみ編集可能にするか"

  - key: "shop_portal.enable_owner_profile_change_request"
    type: "bool"
    default: true
    scope: "global"
    description: "オーナー情報修正の申請（hq_requests）を受け付けるか"

  # ===== スタッフロール・マスク =====
  - key: "shop_portal.staff_role_enable"
    type: "bool"
    default: true
    scope: "global"
    description: "スタッフロール機能（staff_basic / staff_multi_shop）を有効にするか"

  - key: "shop_portal.staff_default_role"
    type: "string"
    default: "staff_basic"
    scope: "global"
    description: "新規スタッフ作成時のデフォルトロール"

  - key: "shop_portal.staff_default_mask_fields"
    type: "json"
    default: '["billing.total_amount","billing.item_amounts","sales.gross_amount","kpi.detail"]'
    scope: "global"
    description: "全スタッフに対してデフォルトでマスクするフィールドキーの一覧"

  - key: "shop_portal.staff_multi_shop_allow_cross_shop_view"
    type: "bool"
    default: true
    scope: "global"
    description: "staff_multi_shop ロールに owner 配下の全店舗横断ビューを許可するか"

  # ===== 店舗別上書き用（例） =====
  - key: "shop_portal.dashboard_default_range_days"
    type: "int"
    default: 7
    scope: "shop"
    description: "特定店舗のみダッシュボード期間を変更したい場合に shop スコープで上書き可能。"

```
