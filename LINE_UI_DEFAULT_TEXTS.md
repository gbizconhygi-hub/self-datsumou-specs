# LINE_UI_DEFAULT_TEXTS.md - LINE版チュートリアル初期テキスト集（v1.4 Draft）

本ファイルは、LINE版チュートリアル UI（LINE_UI）で使用する

ボタンラベル・ガイド文・案内メッセージの初期テキストセットを定義する。

- 実運用では、本部管理UIから店舗別に編集可能とする前提。
- ここに記載する文言は「デフォルト値」であり、ポリシーではない。

## 1. リッチメニュータブ名

```yaml
tabs:
  tab_home: "HOME"
  tab_booking: "予約/プラン"
  tab_support: "サポート"
  tab_account: "アカウント"
  tab_coupon: "クーポン"

```

## 2. 論理スロット（SLOT_◯◯）のラベル

```yaml
slots:

  SLOT_TRIAL_PURCHASE:
    label: "お試し体験購入"
    sub: "安い1回チケットを買う"

  SLOT_TRIAL_RESERVE_PAGE:
    label: "お試し体験予約"
    sub: "お試し専用予約ページへ"

  SLOT_SPOT_RESERVE_PAGE:
    label: "都度払い予約"
    sub: "都度払い専用予約ページへ"

  SLOT_PLAN_LIST:
    label: "チケット/定額プラン一覧"
    sub: "回数券や定額プランの商品一覧"

  SLOT_DEFAULT_RESERVE_PAGE:
    label: "予約（回数券/定額）"
    sub: "回数券・定額プラン専用の予約ページ"

  SLOT_RESERVATION_STATUS:
    label: "予約確認"
    sub: "日時・店舗のみ表示（鍵は出さない）"

  SLOT_RESERVATION_CANCEL_CHANGE:
    label: "予約キャンセル/変更"
    sub: "STORESマイページを開く"

  SLOT_UNLOCK_CODE:
    label: "解錠番号確認"
    sub: "最新の予約日時を確認してから表示"

  SLOT_FORGOTTEN_ITEMS:
    label: "忘れ物案内"
    sub: "忘れ物に関するご案内"

  SLOT_MESSAGE_CONTACT:
    label: "メッセージで問い合わせ"
    sub: "AIチャットで質問する"

  SLOT_HOW_TO_USE:
    label: "サロンの利用方法"
    sub: "基本的な流れ・注意事項"

  SLOT_ENTRY_NOTICE:
    label: "入室の注意事項"
    sub: "ビルの鍵/オートロックなど"

  SLOT_STORE_SELECT:
    label: "利用店舗の選択/変更"
    sub: "ご利用中・予定の店舗を選ぶ"

  SLOT_IDENTITY_LINK:
    label: "会員連携/ログイン"
    sub: "STORES会員とLINEをひも付け"

  SLOT_STORE_LINE_JUMP:
    label: "店舗専用LINEで予約/問合せ"
    sub: "この店舗は総合LINEでは予約受付なし"

  SLOT_COUPON_CENTER:
    label: "クーポン一覧"
    sub: "使えるクーポンを確認"

  SLOT_COUPON_PICKUP:
    label: "キャンペーン情報"
    sub: "店舗別キャンペーンなど"

```

## 3. 店舗前提・店舗変更まわりの文言

```yaml
context_messages:

  store_context_prefix:
    # 予約や店舗固有の話で使う
    text: "現在をご利用中という前提でお話ししますね。"

  store_context_change_hint:
    text: "もし別の店舗についてのご相談でしたら、下の「利用店舗の選択/変更」から変更できます。"

  ask_store_first_time:
    text: |
      ありがとうございます😊
      まずは「どの店舗」についてのご相談か教えてください。

  ask_store_options_title:
    text: "ご利用中、またはご利用予定の店舗をお選びください。"

  ask_store_option_hygi_all:
    text: "ハイジ全体への質問（店舗はまだ決めていない）"

  ask_store_option_no_store_yet:
    text: "まだどの店舗も利用していない"

```

## 4. 名寄せ（会員連携）まわりの文言

```yaml
identity_messages:

  suggest_identity_link_light:
    text: |
      会員情報とLINEをひも付けておくと、
      予約確認や解錠番号の確認がLINEから簡単にできるようになります😊

  identity_required_for_action:
    text: |
      この内容についてお調べするには、会員情報とのひも付け（本人確認）が必要です。
      下のボタンから会員連携をお願いします。

  identity_success:
    text: |
      会員情報とのひも付けが完了しました✨
      先ほどの内容について、そのまま続けてご案内しますね。

  identity_failed:
    text: |
      会員情報とのひも付けがうまくできませんでした。
      入力内容をご確認のうえ、もう一度お試しください。

```

## 5. 予約確認・解錠番号まわりの文言

```yaml
reservation_messages:

  reservation_status_title:
    text: "直近のご予約内容はこちらです。"

  reservation_status_empty:
    text: "現在、今後のご予約は確認できませんでした。"

  reservation_status_item_format:
    # プレーンテキストで表現する場合のテンプレ（実際はFlexで整形）
    text: "{date} {time}\nメニュー：{menu_name}"

  unlock_confirm_intro:
    text: |
      以下のご予約でお間違いないでしょうか？
      日時と店舗をご確認のうえ、「解錠番号を表示する」を押してください。

  unlock_no_reservation:
    text: "解錠番号をお出しできるご予約が見つかりませんでした。"

  unlock_shown_notice:
    text: |
      入室に必要な解錠番号は以下の通りです。
      ご来店時にお間違えのないようご注意ください。

  unlock_security_warning:
    text: |
      ※この画面のスクリーンショットや解錠番号の共有はお控えください。
      第三者による不正利用の原因となる可能性があります。

```

## 6. 忘れ物・サポート系文言

```yaml
support_messages:

  how_to_use_basic:
    text: |
      サロンのご利用方法は、ざっくり次の流れです。
      1. ご予約の時間にご来店
      2. 入室・ロッカーでお着替え
      3. マシンの準備・照射
      4. 片付け・退室
      詳しい手順は店舗内の案内もあわせてご確認ください。

  entry_notice_default:
    text: |
      入室方法についてのご案内です。
      ・ビル入口のオートロック番号
      ・エレベーターの階数
      ・店舗ドアの位置
      など、店舗からの個別案内もあわせてご確認ください。

  forgotten_items_default:
    text: |
      お忘れ物に関するご案内です。
      ・一定期間（例：30日間）保管のうえ処分させていただきます。
      ・衛生上の理由から、一部のお品物は当日中に処分させていただく場合があります。
      詳細は店舗ごとのルールに準じます。

```

## 7. 店舗LINE丸投げ店舗用の文言

```yaml
redirect_messages:

  store_line_redirect_intro:
    text: |
      こちらの店舗のご予約・チケット購入は、
      総合サポートLINEではなく「店舗専用LINE」にて承っております。

  store_line_redirect_cta:
    text: "店舗専用LINEを開く"

  store_line_redirect_note:
    text: |
      総合サポートLINEでは、解錠番号・忘れ物・全体的なご質問のみ承ります。
      予約の新規/変更/キャンセルは店舗専用LINEからお願いいたします。

```

## 8. エラー・障害時の文言（LINE版）

```yaml
error_messages:

  api_unavailable:
    text: |
      ただいまシステムが混み合っているため、
      一部の情報（予約や解錠番号など）をお調べできない状況です。
      お手数ですが、少し時間をおいてから再度お試しください。

  line_ui_disabled_for_shop:
    text: |
      現在、この店舗は総合サポートLINEからの受付を停止しております。
      大変お手数ですが、店舗専用LINEまたは店舗案内に記載の連絡先からお問い合わせください。

```
