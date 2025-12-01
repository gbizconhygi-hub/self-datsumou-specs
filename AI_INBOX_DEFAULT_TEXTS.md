# AI_INBOX_DEFAULT_TEXTS.md (v1.0 Draft)

AI-INBOX（[ai.self-datsumou.net](http://ai.self-datsumou.net/)）で使用する

画面ラベル・ボタン・システムメッセージのデフォルト文言定義。

※ 実装上は YAML／配列等に変換して利用する想定。

---

## 1. グローバルナビ

```yaml
nav:
  inbox: "INBOX"
  search: "チケット検索／履歴"
  ai_faq: "AI・FAQ管理"
  owners_shops_billing: "オーナー・店舗・請求"
  meetings: "MTG・電話相談"
  system_settings: "システム・設定"
  logout: "ログアウト"

```

---

## 2. INBOX 一覧

```yaml
inbox:
  title: "INBOX - 全チャネル問い合わせ一覧"
  description: "help／LINE／SHOPポータル／電話代行など、全ての問い合わせ・申請・アラートを一覧表示します。"

  tab_all: "すべて"
  tab_customer: "顧客起点"
  tab_owner: "加盟店起点"
  tab_opmail: "電話代行・本部起点"
  tab_system: "システム・アラート"

  column_created_at: "受付日時"
  column_updated_at: "最終更新"
  column_status: "ステータス"
  column_source: "起点"
  column_shop: "店舗"
  column_owner: "オーナー"
  column_subject: "件名"
  column_last_sender: "最終送信者"
  column_ai_status: "AIステータス"
  column_sla: "SLA"

  filter_title: "フィルタ"
  filter_date_range: "期間"
  filter_source: "起点チャネル"
  filter_status: "ステータス"
  filter_shop: "店舗"
  filter_owner: "オーナー"
  filter_category: "カテゴリ"
  filter_tag: "HQタグ"
  filter_ai_escalated: "AIエスカレーションあり"
  filter_reset: "フィルタをリセット"

  badge_customer: "顧客"
  badge_owner: "加盟店"
  badge_opmail: "電話代行"
  badge_system: "システム"
  badge_ai_escalated: "AI→本部"
  badge_ai_self_resolved: "AI自己解決済"

  empty_message: "表示対象のチケットはありません。条件を変更して再検索してください。"

```

---

## 3. チケット詳細・対応画面

```yaml
ticket_detail:
  title_prefix: "チケット詳細"
  breadcrumb_inbox: "INBOXに戻る"

  section_timeline: "やりとり"
  section_context: "コンテキスト"
  section_ai_assist: "AIアシスト"
  section_related_tickets: "関連チケット"
  section_tags: "HQタグ"

  label_status: "ステータス"
  label_assignee: "担当者"
  label_sla: "SLA"
  label_incident_key: "案件グループ"

  button_reply: "返信を書く"
  button_change_status: "ステータス変更"
  button_change_assignee: "担当者変更"
  button_add_tag: "タグを追加"
  button_remove_tag: "タグを削除"
  button_ai_summary: "やりとりを要約"
  button_ai_draft: "返信ドラフトを作成"
  button_incident_link: "このチケットを案件グループに関連付ける"

  ai_summary_title: "AI要約"
  ai_summary_placeholder: "AIによる要約がここに表示されます。"
  ai_draft_title: "返信ドラフト"
  ai_draft_notice: "送信前に必ず内容を確認・修正してください。"

  system_message_no_related_tickets: "関連付けられたチケットはありません。"
  system_message_incident_linked: "案件グループに関連付けました。"
  system_message_incident_unlinked: "案件グループとの関連付けを解除しました。"

  confirmation_change_status: "ステータスを「{new_status}」に変更します。よろしいですか？"
  confirmation_send_reply: "この内容で返信を送信します。よろしいですか？"

```

---

## 4. FAQ／AI設定管理

```yaml
ai_faq:
  title: "AI・FAQ管理"
  description: "AIの意図分類（intent）・FAQ・回答テンプレート・AIパラメータを管理します。"

  tab_intents: "インテント一覧"
  tab_qa: "FAQ・Q&A"
  tab_ai_profiles: "AIプロファイル"
  tab_texts_params: "UIテキスト／パラメータ"

  intents_list_title: "インテント一覧"
  column_intent_key: "インテントキー"
  column_super_category: "大分類"
  column_parent_category: "カテゴリ"
  column_routing_policy: "ルーティングポリシー"
  column_allowed_channels: "対象チャネル"

  button_new_intent: "インテントを追加"
  button_edit_intent: "編集"
  button_delete_intent: "削除"

  qa_list_title: "FAQ・Q&A一覧"
  column_channel: "チャネル"
  column_question: "質問（タイトル）"
  column_answer_template: "回答テンプレート"

  button_new_qa: "Q&Aを追加"
  button_edit_qa: "編集"
  button_delete_qa: "削除"

  ai_profile_title: "AIプロファイル設定"
  label_model_name: "モデル名"
  label_temperature: "Temperature"
  label_max_tokens: "最大トークン数"
  label_intent_confidence_threshold: "意図判定しきい値"

  system_message_intent_saved: "インテントを保存しました。"
  system_message_qa_saved: "Q&Aを保存しました。"
  system_message_ai_profile_saved: "AIプロファイルを保存しました。"

```

---

## 5. FAQ候補（解決済みチケット → ナレッジ）

```yaml
faq_candidates:
  title: "FAQ候補"
  description: "解決済みチケットの中から、FAQに掲載できそうなものを候補として表示します。"

  column_created_at: "受付日時"
  column_resolved_at: "解決日時"
  column_intent: "インテント"
  column_shop: "店舗"
  column_owner: "オーナー"
  column_preview_question: "質問の概要"
  column_preview_answer: "回答の概要"

  button_preview_detail: "詳細を確認"
  button_accept: "FAQとして採用"
  button_accept_with_edit: "編集して採用"
  button_reject: "候補から除外"

  system_message_no_candidates: "現在、FAQ候補はありません。"
  system_message_candidate_accepted: "FAQに登録しました。"
  system_message_candidate_rejected: "候補から除外しました。"

```

---

## 6. オーナー・店舗・請求

```yaml
owners_shops:
  title: "オーナー・店舗・請求"
  tab_owners: "オーナー一覧"
  tab_shops: "店舗一覧"
  tab_billing: "請求一覧"

  owners_list_title: "オーナー一覧"
  column_owner_code: "オーナーコード"
  column_owner_name: "オーナー名"
  column_owner_type: "区分"
  column_status: "ステータス"
  column_billing_email: "請求連絡先"
  column_chatwork_room: "ChatWorkルーム"

  shops_list_title: "店舗一覧"
  column_shop_code: "店舗コード"
  column_shop_name: "店舗名"
  column_ownership_type: "所有区分"
  column_billing_profile: "請求プロファイル"
  column_region: "エリア"
  column_shop_status: "店舗ステータス"

  button_new_owner: "オーナー新規作成"
  button_new_shop: "店舗新規作成"
  button_view_timeline: "タイムラインを見る"

billing:
  invoices_list_title: "請求書一覧"
  column_billing_month: "請求対象月"
  column_issue_date: "発行日"
  column_due_date: "支払期日"
  column_total_amount: "合計金額"
  column_status: "ステータス"
  column_issue_ready_flag: "一括発行対象"
  column_visibility: "表示先"

  button_open_issue_dialog: "請求書を発行する"
  button_issue_selected: "選択した請求書を一括発行"
  button_issue_single: "この請求書だけ発行"
  button_download_pdf: "PDFダウンロード"

  status_draft: "下書き"
  status_sent: "発行済み"
  status_paid: "入金済み"
  status_void: "無効"

  system_message_no_invoices: "該当する請求書はありません。"
  system_message_issue_completed: "請求書を発行しました。"
  system_message_issue_partial_warning: "一部の請求書は発行対象から除外されています。内容を確認してください。"

```

---

## 7. MTG・電話相談

```yaml
meetings:
  title: "MTG・電話相談"
  tab_slots: "公開スロット"
  tab_meetings: "予約一覧"

  slots_title: "本部側相談枠（公開スロット）"
  column_slot_time: "日時"
  column_meeting_type: "種別"
  column_mode: "方法"
  column_capacity: "受入数"
  column_status: "ステータス"

  meetings_title: "予約済みMTG・電話相談"
  column_owner: "オーナー"
  column_shop: "店舗"
  column_meeting_type: "種別"
  column_scheduled_at: "実施日時"
  column_status: "ステータス"

  button_link_from_ticket: "この案件でMTGを提案"
  badge_booking_mode_direct: "即予約"
  badge_booking_mode_request: "要調整"

  status_requested: "要調整"
  status_confirmed: "確定"
  status_done: "実施済み"
  status_cancelled: "キャンセル"

  system_message_no_meetings: "予約されているMTG・電話相談はありません。"

```

---

## 8. システム・設定／履歴ビュー

```yaml
system_settings:
  title: "システム・設定"
  tab_param_history: "パラメータ変更履歴"
  tab_monitoring: "監視（API・イベント）"

  param_history_title: "パラメータ変更履歴"
  column_changed_at: "変更日時"
  column_actor: "変更者"
  column_entity: "対象"
  column_key: "キー"
  column_before: "変更前"
  column_after: "変更後"

  monitoring_title: "簡易監視ビュー"
  section_api_errors: "APIエラー"
  section_sys_events: "システムイベント"

  system_message_no_history: "表示できる変更履歴はありません。"
  system_message_no_events: "表示できるイベントはありません。"

```

---

## 9. HQタグ（ticket_tags）

```yaml
tags:
  label_title: "HQタグ"
  label_input_placeholder: "タグを選択"
  system_message_no_tags: "タグは設定されていません。"
  system_message_tag_added: "タグを追加しました。"
  system_message_tag_removed: "タグを削除しました。"

```

---

## 10. 共通メッセージ

```yaml
common:
  error_generic: "エラーが発生しました。時間をおいて再度お試しください。"
  error_unauthorized: "この操作を実行する権限がありません。"
  error_validation: "入力内容を確認してください。"
  confirm_discard_changes: "編集内容が保存されていません。このページから移動してもよろしいですか？"

```
