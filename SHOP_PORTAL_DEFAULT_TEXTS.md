# SHOP_PORTAL_DEFAULT_TEXTS (v1.0 Draft)

```yaml
shop_portal:
  # ===== ダッシュボード系 =====
  dashboard:
    no_data_notice: "まだ十分なデータがありません。オープン後しばらく経過すると集計が表示されます。"
    section_title_announcements: "本部からのお知らせ"
    section_title_billing: "請求・支払い状況"
    section_title_sales_summary: "売上サマリー"
    section_title_ticket_summary: "店舗ごとの問い合わせ状況"

  # ===== お知らせ =====
  announcements:
    unread_badge_label: "未読のお知らせがあります"
    empty: "現在、お知らせはありません。"

  # ===== オーナー情報ウィザード =====
  owner_profile:
    wizard_title: "オーナー情報のご登録"
    wizard_intro: "請求書の宛名・住所など、今後のご案内に必要な情報をご入力ください。"
    complete_notice: "オーナー情報の登録が完了しました。以降の修正は本部にて承ります。"

  # ===== 問い合わせ（tickets） =====
  tickets:
    list_title_open: "要対応の問い合わせ"
    list_title_closed: "完了した問い合わせ"
    empty_open: "現在、対応が必要な問い合わせはありません。"
    reply_placeholder: "オーナー様からのお返事をこちらにご記入ください。"
    reply_saved: "返信を保存しました。お客様への案内が完了しました。"
    status_changed_to_closed: "この問い合わせを「完了」に変更しました。"

  # ===== 備品発注 =====
  supplies:
    list_title: "備品発注"
    empty_items: "現在、発注可能な備品は登録されていません。"
    order_confirm_message: "この内容で発注を送信します。よろしいですか？"
    order_completed: "発注を受け付けました。請求書に反映されるまで少しお時間をください。"
    order_error_stock: "在庫状況の確認でエラーが発生しました。本部までお問い合わせください。"

  # ===== 各種申請（hq_requests） =====
  requests:
    list_title: "本部への申請一覧"
    empty: "現在、申請中の案件はありません。"
    new_request_title: "新しい申請を送る"
    submitted: "申請を受け付けました。本部で内容を確認いたします。"
    status_updated: "申請のステータスが更新されました。詳細をご確認ください。"

  # ===== キャンペーン申請（campaign_requests） =====
  campaigns:
    new_title: "キャンペーン申請"
    wizard_intro: "店頭・SNSで告知したいキャンペーン内容をご入力ください。"
    submitted: "キャンペーン申請を受け付けました。本部で内容を確認します。"
    needs_revision: "本部から修正のご依頼があります。内容をご確認のうえ、再申請をお願いいたします。"
    approved: "キャンペーン内容が承認されました。バナー制作に進行します。"
    banner_completed: "キャンペーンバナーのご用意ができました。ダウンロードしてご活用ください。"

  # ===== デザイン依頼（design_requests / DR） =====
  design:
    list_title: "デザイン依頼（DR）"
    new_title: "新しいデザイン依頼を作成"
    submitted: "デザイン依頼を受け付けました。本部で内容とお見積りを確認します。"
    estimate_proposed: "お見積りをご提示しました。内容をご確認のうえ、承認またはキャンセルをお選びください。"
    estimate_approved: "お見積りが承認されました。デザイナーが制作を開始します。"
    delivered: "デザインの納品が完了しました。内容をご確認ください。"
    completed: "このデザイン依頼は完了済みです。追加の修正が必要な場合は、新しい DR を起票してください。"
    dr_required_notice: "この内容は新しいデザイン案件になります。まず SHOP_PORTAL からデザイン依頼（DR）を起票してください。"

  # ===== オーナー向け AI 相談 =====
  owner_ai:
    greeting: "いつも店舗運営ありがとうございます。本部サポートAIです。ご不明点やお困りごとは、まずこちらにお気軽にご相談ください。"
    escalation_to_hq: "この内容は本部で個別に確認が必要なため、担当者に引き継ぎます。少しお時間をいただきます。"
    contract_fc_warning: "FC 契約や違約金など、個別契約条件の最終判断は必ず契約書と本部担当の説明を優先してください。ここでは一般的な考え方のみお伝えします。"
    quiet_hours_notice: "現在は夜間帯のため、詳しいご回答に少しお時間をいただく場合があります。お急ぎの場合は翌営業日の本部からの連絡をお待ちください。"

  # ===== 本部 MTG =====
  meetings:
    list_title: "本部ミーティングの予約"
    empty_slots: "現在ご予約可能な枠はありません。しばらくしてから再度ご確認ください。"
    request_sent: "ミーティングのご希望を受け付けました。本部で日程を確定し、改めてご連絡いたします。"
    confirmed_notice: "本部ミーティングの日程が確定しました。詳細はSHOP_PORTALとメールでご確認ください。"

  # ===== エラー／共通 =====
  common:
    generic_error: "処理中にエラーが発生しました。お手数ですが、時間をおいて再度お試しください。"
    permission_denied: "この画面にはアクセス権限がありません。ご契約状況については本部までお問い合わせください。"
    not_found: "指定されたデータが見つかりませんでした。既に削除・キャンセルされている可能性があります。"
