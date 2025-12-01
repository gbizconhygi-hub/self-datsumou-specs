# AI_INBOX_DEFAULT_PARAMS.md (v1.0 Draft)

AI-INBOX（[ai.self-datsumou.net](http://ai.self-datsumou.net/)）における

sys_params／shop_params 等で管理するデフォルトパラメータ定義。

※ 実際の sys_params への登録は、初期セットアップスクリプトで行う想定。

---

## 1. INBOX 一覧関連

```yaml
ai_inbox:
  default_list_days:
    value: 7
    type: int
    scope: "global"
    description: "INBOXに表示するデフォルトの期間（日数）。現在から過去何日分までを対象とするか。"

  page_size:
    value: 50
    type: int
    scope: "global"
    description: "INBOX一覧の1ページあたりの表示件数。"

  autorefresh_interval_sec:
    value: 0
    type: int
    scope: "global"
    description: "INBOX画面の自動更新間隔（秒）。0の場合は自動更新しない。"

  show_ai_flags:
    value: true
    type: bool
    scope: "global"
    description: "AIエスカレーション／自己解決フラグを INBOX 上に表示するか。"

  op_mail_badge_category_prefix:
    value: "operator_mail"
    type: string
    scope: "global"
    description: "OP_MAIL由来のチケットを識別するためのカテゴリプレフィックス。"
```

---

## 2. チケット詳細／AIアシスト関連

```yaml
ai_inbox:
  ai_draft_temperature:
    value: 0.4
    type: float
    scope: "global"
    description: "HQ向け返信ドラフト生成時のTemperature。"

  ai_summary_max_tokens:
    value: 512
    type: int
    scope: "global"
    description: "チケット要約生成時の最大トークン数。"

  ai_assist_enable:
    value: true
    type: bool
    scope: "global"
    description: "AIアシスト（要約／ドラフト生成）機能を有効にするか。"

  incident_suggestion_enable:
    value: true
    type: bool
    scope: "global"
    description: "類似チケット候補（incident候補）を自動表示するか。"

```

---

## 3. SLA／優先度関連

```yaml
ai_inbox:
  sla_minutes_by_parent_category:
    value:
      faq_basic: 1440        # 24時間
      reservation_related: 1440
      trouble_customer: 240  # 4時間
      contract_fc: 720       # 12時間
      system_trouble: 240
      other: 1440
    type: json
    scope: "global"
    description: "parent_categoryごとのSLA（初回レスポンス目安）を分単位で定義。"

  sla_minutes_opmail_default:
    value: 60
    type: int
    scope: "global"
    description: "OP_MAIL（電話代行）由来チケットのデフォルトSLA（分）。"

  sla_overdue_warning_margin_minutes:
    value: 60
    type: int
    scope: "global"
    description: "SLA期限の何分前からINBOX上で警告表示するか。"

```

---

## 4. FAQ候補（解決済み → ナレッジ）関連

```yaml
ai_inbox:
  faq_candidate_enable:
    value: true
    type: bool
    scope: "global"
    description: "FAQ候補機能を有効にするか。"

  faq_candidate_min_same_intent_count:
    value: 3
    type: int
    scope: "global"
    description: "同一インテントで何件以上解決済みチケットがある場合に、FAQ候補として抽出するか。"

  faq_candidate_lookback_days:
    value: 90
    type: int
    scope: "global"
    description: "FAQ候補の抽出対象とする期間（過去何日分）。"

```

---

## 5. AI設定管理・ページング関連

```yaml
ai_inbox:
  max_intents_per_page:
    value: 100
    type: int
    scope: "global"
    description: "インテント一覧画面の1ページあたりの最大件数。"

  max_qa_entries_per_page:
    value: 100
    type: int
    scope: "global"
    description: "FAQ・Q&A一覧画面の1ページあたりの最大件数。"

  intent_edit_role:
    value: "hq_admin"
    type: string
    scope: "global"
    description: "インテントの追加・削除・重大変更が可能なロール。"

  ai_param_edit_role:
    value: "hq_admin"
    type: string
    scope: "global"
    description: "AIパラメータ（モデル・Temperature等）の編集が可能なロール。"

  hot_edit_allowed:
    value: false
    type: bool
    scope: "global"
    description: "本番環境での即時パラメータ変更（ホットエディット）を許可するか。falseの場合はデプロイ等を経て反映する運用を前提とする。"

```

---

## 6. オーナー・店舗・請求関連

```yaml
ai_inbox:
  owner_list_page_size:
    value: 50
    type: int
    scope: "global"
    description: "オーナー一覧の1ページあたりの表示件数。"

  shop_list_page_size:
    value: 100
    type: int
    scope: "global"
    description: "店舗一覧の1ページあたりの表示件数。"

  default_billing_month_range:
    value: 3
    type: int
    scope: "global"
    description: "請求書一覧のデフォルト表示対象とする月数（過去何ヶ月分）。"

  invoice_default_due_days:
    value: 0
    type: int
    scope: "global"
    description: "支払期日までの日数。0の場合は「当月末」を固定とする運用とし、システム上は日付計算で扱う。"

  invoice_issue_time:
    value: "09:00"
    type: string
    scope: "global"
    description: "1日発行時に想定する発行時間（UI上の表示・バッチ起動の目安）。"

  invoice_notify_email_template_key:
    value: "invoice_issued_owner_v1"
    type: string
    scope: "global"
    description: "請求書発行通知メールに使用するテンプレートキー。"

  invoice_show_on_portal_profiles:
    value:
      - "fc_standard"
    type: json
    scope: "global"
    description: "SHOP_PORTAL上に請求書一覧を表示する billing_profile のリスト。"

```

---

## 7. MTG・電話相談関連（hq_mtg）

```yaml
hq_mtg:
  default_visible_days:
    value: 30
    type: int
    scope: "global"
    description: "SHOPポータル側に公開するMTG・電話相談枠の表示期間（未来何日分）。"

  max_requests_per_month_per_shop:
    value: 3
    type: int
    scope: "global"
    description: "1店舗あたり1ヶ月に受付けるMTG・電話相談リクエストの上限。"

  ai_suggestable_types:
    value:
      - "routine_checkin"
      - "trouble_consult"
      - "fc_contract"
    type: json
    scope: "global"
    description: "AI／HQがチケット詳細から提案可能なMTG種別の一覧。"

  enable_phone_light:
    value: true
    type: bool
    scope: "global"
    description: "軽め電話相談（phone_light）スロットを運用するか。"

```

---

## 8. システム・監視／履歴ビュー関連

```yaml
ai_inbox:
  api_error_threshold:
    value: 10
    type: int
    scope: "global"
    description: "APIエラーを「要注意」として監視ビューに強調表示する閾値（一定期間内の件数）。"

  sys_events_display_days:
    value: 7
    type: int
    scope: "global"
    description: "システムイベント（sys_events）の簡易集計に使用する期間（日数）。"

  show_raw_logs:
    value: false
    type: bool
    scope: "global"
    description: "AI-INBOX上で生ログ（リクエスト・レスポンス全文）を表示するか。falseの場合は集計値のみ。"

```

---

## 9. HQタグ（ticket_tags）関連

```yaml
ai_inbox:
  tag_edit_role:
    value: "hq_admin,hq_operator"
    type: string
    scope: "global"
    description: "HQタグの追加・削除が可能なロールのカンマ区切りリスト。"

  max_tags_per_ticket:
    value: 5
    type: int
    scope: "global"
    description: "1チケットに付与できるHQタグの最大数。"

```

---

## 10. タイムライン／履歴ビュー関連

```yaml
ai_inbox:
  owner_timeline_lookback_days:
    value: 365
    type: int
    scope: "global"
    description: "オーナー単位タイムラインに表示するイベントの期間（過去何日分）。"

  param_history_lookback_days:
    value: 180
    type: int
    scope: "global"
    description: "パラメータ変更履歴ビューで表示する期間（過去何日分）。"

```

---

## 11. 権限・ロール関連（補足）

```yaml
ai_inbox:
  allowed_roles_for_param_edit:
    value: "hq_admin"
    type: string
    scope: "global"
    description: "sys_params／shop_params 等のシステムパラメータを編集できるロール。"

  allowed_roles_for_invoice_issue:
    value: "hq_admin"
    type: string
    scope: "global"
    description: "請求書の発行（sentステータスへの変更）が許可されるロール。"

```
