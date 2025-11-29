# PHASE_03_LINE_UI - 仕様設計書 (PHASE_03_LINE_UI.md Draft)

## 1. 目的・範囲

本書は、「加盟店サポート自動化プロジェクト v1.4」における

**PHASE_03_LINE_UI（line.：LINE公式アカウント上の顧客向けチュートリアル UI）** のコア仕様を定義する。

本フェーズの目的は：

- 1つの LINE 公式アカウントで、**全店舗の顧客対応を一元的に捌く**こと
- HELP_UI（help.）と同じ AI_CORE／SYS_CORE／API_COMM／DB BOX を利用しつつ、
    - LINE特有の要素（LINE ID・リッチメニュー・LIFF・トーク履歴）を活かした
    - 「本人特定前提」「店舗文脈を持った」チュートリアル体験を実現すること
- 店舗ごとの予約方式（チケット／定額プラン／お試し専用予約ページ／都度払い専用予約ページ 等）や
    
    店舗LINEへの丸投げなどの差異を、**すべて DB／パラメータで吸収**し、
    
    コード修正を不要にすること
    

を達成することである。

本書は以下を対象とし、文言・細かなUIレイアウト・具体パラメータ値は別ファイルに委ねる。

- line_tutorial_api（`/api/tutorial/message`）の I/O 契約（HELP_UIとの共通化を含む）
- LINE Webhook → line_gateway → line_tutorial_api → AI_CORE／API_COMM → line_response_adapter のフロー
- LINE ID と STORES 会員情報の名寄せロジック（customer_links）と、店舗リレーションBOX（line_user_shop_links）
- 予約フロー差を吸収するための `shop_links`／`shop_options` の拡張
- リッチメニュータブ構造と、店舗タイプ／ユーザー状態ごとのプロファイル切り替えルール
- セキュリティ・権限・ログ方針（骨レベル）

文面・トーン・NG表現・具体テキストは `LINE_UI_POLICY.md`／`LINE_UI_DEFAULT_TEXTS.md`、

しきい値・回数・TTL 等の具体値は `LINE_UI_DEFAULT_PARAMS.md` で管理する。

## 2. 前フェーズからの継承ポイント

### 2.1 共通アーキテクチャ

- サブドメイン構成：
    - `help.` … Webチュートリアル（メール返信）
    - `line.` … 本フェーズ対象（LINE公式アカウント／LIFF）
    - `shop.` … オーナーポータル／加盟店向けUI
    - `ai.` … 本部向け管理／AI-INBOX 等
    - `assets.` … メディア／添付ファイルなど
- `.env` は `/self-datsumou.net/.env` に一元配置。
    
    各アプリは SYS_CORE の `SysConfig` を通じて .env／sys_params／DBを参照する。
    
- SoRポリシー：
    - テーブル定義は `db_fields.yaml`（Phase0 DB_CORE）が真実
    - 設定値は「DB（shops/shop_secrets/options_master/shop_options 等） → sys_params → .env → コードデフォルト」の優先順
    - 意味が重複する sys_params キーを作らない（二重管理禁止）

### 2.2 BOX／SYS_CORE／API_COMM／AI_CORE

- DB BOX：
    - owners／owner_contacts／shops／shop_secrets／shop_links／shop_internal_info／options_master＋shop_options／sales_records／tickets 等
- SYS_CORE：
    - `SysConfig`／`SysLogger`／`SysEscalation`／`HttpClient`／各種 Client（Stores／Line／OpenAI／ChatWork 等）
- API_COMM：
    - STORES／RemoteLOCK／通知系を「業務API」として抽象化、
    - `ApiCommResult` で統一フォーマットのレスポンスを返す
    - `customer_links` テーブルで `external_user_id`（LINE userId 等）と STORES 顧客IDを名寄せ
- AI_CORE：
    - help.／line.／shop.／ai. 共通の頭脳（`shared/ai_core.php`）
    - `AI_CORE_REQUEST`／`AI_CORE_RESPONSE` を介して、意図分類／回答生成／フォールバック／ログを統合

### 2.3 HELP_UI からの継承

- チュートリアルAPI `/api/tutorial/message` の基本構造：
    - `channel`／`version`／`shop_code`／`session_id`／`event_type`／`payload`／`client_meta`／`identity_token`
- セッションステート：
    - `STATE_NEW`／`STATE_NORMAL_AI`／`STATE_AWAIT_IDENTITY`／`STATE_ESCALATED`／`STATE_SELF_RESOLVED` 等
- 本人確認フロー：
    - `identity_token` による本人状態の管理
    - STORES会員情報・予約情報にアクセスが必要な intent では本人確認必須
- エスカレーション：
    - AI_CORE decision／ターン数閾値／ユーザー明示要望をトリガに、`tickets` にチケット登録
    - 二次対応チャネル（LINE／メール／電話）の最終判断は人間が行う

本フェーズでは、

**HELP_UI と同一エンドポイント／ステートマシン／AI_CORE／API_COMM を使いつつ、
LINE特有の I/OとUI構造を追加する**。

## 3. 新規決定事項（LINE_UI 固有の仕様）

### 3.1 LINE公式アカウント運用ポリシー

- LINE公式アカウントは **1つで全店舗対応** とする。
- shops レコードが増減しても、LINE_UI／他チャネルのコード修正が不要になる設計を最優先とする。
- 禁止事項：
    - `if ($shopId === 3) { ... }` など shop_id ベタ書きによる条件分岐
    - 対応店舗リストをコード内配列で持つ実装案
- 店舗ごとの差異は必ず DB／パラメータで吸収する：
    - `options_master`＋`shop_options`
    - `sys_params(scope=shop)`
    - 必要であれば新BOXテーブル追加（DDLを仕様書に明記）

### 3.2 3レイヤー構造（本人・店舗・会話）

LINE UI における顧客の識別は、次の3レイヤーに分けて扱う：

1. **Aレイヤ：本人名寄せ（誰か？）**
    - SoR：`customer_links`
        - `external_user_id = line_user_id`
        - `stores_customer_id`（STORES会員）
    - 一度名寄せに成功したら、基本的に長期維持（TTLはAI_CORE／セキュリティ側で管理）
2. **Bレイヤ：LINEユーザー⇔店舗リレーション（どの店舗と関係があるか？）**
    - SoR：新BOX `line_user_shop_links`（後述）
    - 「メイン店舗」「明示的に選んだ店舗」「訪問実績のある店舗」「ハイジ全体のみ」の情報を保持
3. **Cレイヤ：会話／チケット単位の店舗文脈（今回の話はどこ店の話か？）**
    - SoR：`ai_sessions.shop_id`／`tickets.shop_id`
    - shopポータルで表示するLINE履歴は、この Cレイヤを参照してフィルタ

### 3.3 LIFF の役割とモード

LIFF は次のモードを持つ：

- `mode=identity`
    - LINE ID ↔ STORES会員の名寄せ
- `mode=store_select`
    - 利用店舗／質問店舗の選択・変更
- `mode=reservations`
    - 予約一覧／詳細表示（将来的にキャンセル等もここから）
- `mode=coupon_center`（将来用）
    - クーポン一覧

LIFF の詳細UIや文言は `LINE_UI_POLICY`／`LINE_UI_DEFAULT_TEXTS` 側で定義し、

本仕様書では「どの場面でどの mode を使うか」までを扱う。

### 3.4 リッチメニュー・タブ構造

- リッチメニューは「タブバー＋タブごとのボタンエリア」という2層構造で設計する。
- タブ種別（論理）：
    - `tab_home` … メイン操作（お試し／予約／解錠／AI相談）
    - `tab_booking` … 予約・チケット・プラン関連
    - `tab_support` … 利用方法・入室注意・忘れ物などサポート系
    - `tab_account` … 店舗選択／会員連携／マイページ
    - `tab_coupon` … クーポンタブ（将来用）
- 1タブにつき「大きめボタン」は最大3つまでとし、詰め込みすぎを防ぐ。
- Messaging API の `richmenuswitch` アクション＋リッチメニューエイリアスを用いてタブ切替を実現する。

### 3.5 予約方式の差異と「予約プロファイル」

- 店舗ごとの予約方式（デフォルト構成／お試し予約ページ導入／都度払い予約ページ導入／店舗LINEへ丸投げ）は、
    - `shop_links`（予約系URL）
    - `options_master`＋`shop_options`（予約方式フラグ）
        
        から自動判定される「予約プロファイル」として扱う。
        
- LINE側のコードは、「予約プロファイル」から導線（スロット→URL or LIFF）を組み立てるだけとし、
    
    店舗ごとの差異はすべてDB設定で吸収する。
    

## 4. データ構造 / I/O定義（line_tutorial_api / LINE Webhook 連携など）

### 4.1 LINE Webhook → line_gateway → line_tutorial_api

### 4.1.1 LineInternalEvent（line_gateway 内部表現）

```json
{
  "line_user_id": "Uxxxx",
  "line_channel_id": "C_GLOBAL_MAIN",
  "shop_id": null,                   // この時点では未解決でもよい
  "event_kind": "USER_TEXT",         // USER_TEXT | POSTBACK_ACTION | LIFF_RETURN | SYSTEM
  "reply_token": "abcd...",
  "message_text": "脇の予約変更できますか？",
  "postback_data": "SLOT_TRIAL_RESERVE_PAGE",
  "liff_payload": null,
  "received_at_utc": "2025-11-29T12:34:56Z"
}

```

- `shop_id` は `line_user_shop_links`／`customer_links`／予約実績を参照して
    
    line_gateway 内の resolver で決定（未特定の場合は null のままチュートリアルAPIに渡してもよい）。
    

### 4.1.2 tutorial_api エンドポイント

- URL: `/api/tutorial/message`
- Method: `POST`
- 物理エンドポイントは HELP_UI と共通で、`channel="line"` によって挙動を切り替える。

リクエスト（共通構造）：

```json
{
  "channel": "line",
  "version": "v1",
  "shop_code": "ngy_kanayama",    // shop_idから導出できればセット。未確定なら null
  "session_id": "uuid-or-null",
  "event_type": "user_message" | "shortcut" | "liff_return" | "system",
  "payload": { ... },             // event_type別
  "client_meta": { ... },
  "identity_token": "..."         // 名寄せ済みなら任意で付与
}

```

### event_type 別 payload の骨

- `user_message`（普通のトーク）

```json
"payload": {
  "text": "脇の予約変更できますか？",
  "media_ids": [],
  "line_user_id": "Uxxxx",
  "source": "direct"
}

```

- `shortcut`（リッチメニュー／クイックリプライ／Flexボタン）

```json
"payload": {
  "shortcut_key": "SLOT_TRIAL_RESERVE_PAGE", // or ACTION_START_TUTORIAL 等
  "source": "richmenu" | "quick_reply" | "flex_button",
  "line_user_id": "Uxxxx"
}

```

- `liff_return`（LIFF完了イベント）

```json
"payload": {
  "flow": "identity" | "store_select" | "other",
  "line_user_id": "Uxxxx",
  "identity_token": "..." ,     // flow=identity のとき
  "selected_shop_code": "ngy_kanayama" // flow=store_select のとき
}

```

- `system`（HELP_UIと共通。障害時などのシステム起因イベント）

```json
"client_meta": {
  "line_user_id": "Uxxxx",
  "user_agent": "line-ios",
  "locale": "ja-JP"
}

```

### 4.1.3 レスポンス構造（LINE向け）

```json
{
  "session_id": "uuid",
  "state": "STATE_NORMAL_AI",
  "mode": "ai" | "fallback",
  "identity_status": "anonymous" | "linked" | "pending",
  "messages": [
    // ai_message / system_banner / form_bubble / rich_action / flex_card 等
  ],
  "meta": {
    "ai_available": true,
    "api_comm_available": true,
    "ai_decision": {
      "intent": "reservation_change",
      "super_category": "reservation",
      "parent_category": "change",
      "escalation_required": false,
      "identity_required": true
    }
  }
}

```

- `identity_status`：
    - `anonymous` … 名寄せ前
    - `linked` … `customer_links` による名寄せ済み
    - `pending` … LIFF 中 or 名寄せ処理進行中
        
        → リッチメニュー構成や案内文の切り替えに利用
        
- `messages[]` の type：
    - `ai_message` … 通常のテキスト吹き出し
    - `system_banner` … 障害案内・重要なお知らせ
    - `form_bubble` … 本人確認／追加情報入力フォーム（LIFF／Flex＋postback等で表現）
    - `rich_action` … ボタン＋説明（LINEテンプレ対応）
    - `flex_card` … 予約内容／店舗情報／クーポンなどのカード表現（論理構造）

`messages[].type` ごとの詳細 JSON Schema は別ファイル（I/O Schema）に切り出す。

### 4.2 AI_CORE／API_COMM との連携（骨）

- line_tutorial_api は、HELP_UIと同じく `AI_CORE_REQUEST` を組み立てて AI_CORE に問い合わせる。
    - `channel = "line"`
    - `actor_type = "customer"`
    - `tenant_id = shops.merchant_public_id`（STORES merchant ID）
    - `meta.channel_specific.line_user_id = line_user_id`
- 本人確認や予約照会が必要な場合のみ API_COMM を呼ぶ：
    - 顧客特定：`customer.resolve_from_identity_token()` または `customer.resolve_from_line()` など
    - 予約一覧取得：`reservation.list_upcoming()`
    - 解錠番号取得：予約情報＋店舗ごとのキー発行パターンに基づく処理（API_COMM or 専用モジュール）

AI_CORE と API_COMM の詳細仕様は Phase2 に委譲し、本フェーズでは

「どのタイミングで呼ぶか」「どの I/O で連携するか」までの骨を定義する。

### 4.3 新DB BOX／フィールド（要約）

### 4.3.1 `line_user_shop_links`

LINEユーザーと店舗の関係BOX：

- 主なカラム：
    - `line_user_id` … LINE userId
    - `shop_id` … shops.id（NULL=ハイジ全体／HQ）
    - `relation_type` … `primary`／`explicit_selected`／`visited`／`hq_only`
    - `is_anonymous` … 名寄せ前=1／名寄せ済=0
    - `last_used_at` … 最後にこの店舗前提でやりとりした日時
- 利用方針：
    - 「今このユーザーはどの店舗前提か？」を解決する第一候補
    - 匿名レコードは `last_used_at + line_ui.anonymous_shop_link_retention_days` 経過でクリーンアップ

### 4.3.2 `shop_links` 予約系 link_type 拡張（例）

- `stores_reserve_ticket_pass` … 回数券＆月謝予約ページ
- `stores_reserve_trial_spot` … お試し体験専用都度払い予約ページ
- `stores_reserve_spot` … 通常メニュー用都度払い予約ページ
- `stores_product_trial_ticket` … お試し体験チケット商品ページ
- `stores_products_plan_list` … チケット／定額プラン一覧ページ
- `stores_mypage_url` … STORES予約マイページURL

### 4.3.3 `options_master`／`shop_options` 予約方式フラグ（例）

- `booking_has_trial_ticket`
- `booking_has_trial_spot_page`
- `booking_has_spot_page`
- `booking_trial_ticket_behaves_like_spot`
- `booking_local_qr_pay_enabled`
- `booking_redirect_to_store_line`
- `line_ui_enabled`（この店舗を総合LINE対象にするか）

### 4.3.4 リッチメニュープロファイル BOX（概要）

- `line_richmenu_profiles`
    - `profile_code`（例：`default_store`／`trial_reserve`／`spot_reserve`／`store_line_redirect` 等）
    - `tab_code`（`tab_home`／`tab_booking`／`tab_support`／`tab_account`／`tab_coupon`）
- `line_richmenu_profile_slots`
    - `profile_id`／`slot_position`（例：`home_main_1`）／`slot_code`（`SLOT_TRIAL_RESERVE_PAGE` 等）
- `line_richmenu_profile_assignments`
    - `(shop_id, state_code)` → `profile_code`

詳細なDDLは PHASE_03_LINE_UI_DB セクション／DB_CORE 拡張として別途記載する。

## 5. フロー / 画面遷移（共通処理フロー）

### 5.1 共通フロー：LINEトーク → AI回答

1. ユーザーが LINE トークで発話 or ボタン操作
2. LINE Messaging API → `line_webhook` に Webhook イベント送信
3. `line_gateway` が `LineInternalEvent` を生成し、`line_user_shop_links`／`customer_links`／予約情報から `shop_id` を解決（可能なら）
4. `LineInternalEvent` を `tutorial_api` リクエストに変換（`event_type`／`payload` など）
5. `tutorial_api(channel="line")` が AI_CORE／API_COMM を呼び分けて `messages[]` を生成
6. `line_response_adapter` が `messages[]` を LINEテキスト／テンプレ／Flexに変換し返信
7. `ai_sessions`／`ai_turns`／`sys_events` 等にログを保存

### 5.2 フローA：初回ユーザー（店舗未選択・名寄せ前）

1. `line_user_shop_links` に該当レコードなし（`anonymous_no_store`）
2. HOMEタブでは：
    - 「店舗を選ぶ」（`SLOT_STORE_SELECT`）
    - 「サロンの利用方法」（`SLOT_HOW_TO_USE`）
    - 「メッセージによるお問い合わせ」（`SLOT_MESSAGE_CONTACT`）
        
        のような構成
        
3. 店舗を選ぶ操作：
    - タップ → `shortcut` → LIFF `mode=store_select`
    - `store_select` LIFFで都道府県→店舗名の一覧から選択
    - 完了後 `liff_return(flow="store_select")` → `line_user_shop_links` に `relation_type=primary` で登録
4. 以降は `anonymous_with_store` 状態として、店舗前提の会話を開始可能

### 5.3 フローB：一般質問（店舗関係なし）

1. intent が「全店共通の一般FAQ」と判定された場合
2. `ai_sessions.shop_id = null`（＋ owner_id=HQ）で会話を記録
3. 回答は「ハイジ全体としての回答」とし、店舗名は出さない
4. shopポータルには表示せず、本部側のビューからのみ閲覧

### 5.4 フローC：店舗固有の質問（予約系）

1. intent が予約／解錠／店舗設備など店舗固有と判定された場合
2. `line_user_shop_links` の `primary` or `explicit_selected` を見て、仮の `shop_id` を決定
3. 回答文の冒頭で：
    - 「現在【◯◯店】をご利用中という前提でお話ししますね」
        
        と店舗前提を明示（文言は DEFAULT_TEXTS）
        
    - 併せて「店舗を変更する」Quick Reply／ボタンを提示
4. ユーザーが店舗変更を選択した場合：
    - 実際に予約がある店舗等を候補に出し、選択された店舗を
        
        `line_user_shop_links(relation_type="explicit_selected")` に追加
        
    - そのセッションの `ai_sessions.shop_id` も更新

### 5.5 フローD：本人確認／名寄せ（LIFF）

1. intent が「予約内容確認」「解錠番号」「キャンセル」等で `identity_required = true`
2. `identity_status="anonymous"` のとき：
    - tutorial_api が `rich_action` として「会員連携（本人確認）」ボタンを返し、
    - ボタンタップで LIFF `mode=identity` を開く
3. ユーザーが LIFF 内で STORES ログイン完了
4. `liff_return(flow="identity", identity_token="...")` が line_gateway 経由で tutorial_api に届く
5. API_COMM 経由で identity_token → STORES顧客ID を解決し、`customer_links` を更新
6. `line_ui.auto_retry_after_identity = true` の場合：
    - 直前の intent を自動で再実行し、元の「予約確認」や「解錠番号」処理まで進める

### 5.6 フローE：予約確認（SLOT_RESERVATION_STATUS）

1. ユーザーが「予約確認」スロット or メッセージから予約確認 intent を発火
2. API_COMM で直近の未来予約を取得
3. tutorial_api は `flex_card` で以下のみ表示：
    - 予約日時（強調表示）
    - 店舗名
    - メニュー名（必要に応じて）
4. 解錠番号は一切表示せず、必要に応じて `SLOT_UNLOCK_CODE` を案内する

### 5.7 フローF：解錠番号確認（SLOT_UNLOCK_CODE）

1. 名寄せ済み（`identity_status="linked"`）＋予約が1件に特定可能な場合のみ実行
2. API_COMM／キー発行ロジックから解錠番号を取得または生成
3. Flexカードで以下をセット表示：
    - 予約日時（日時誤認を防止するため特に強調）
    - 店舗名
    - （任意）メニュー名・プラン種別
    - 解錠番号
4. セキュリティ上の理由で条件を満たさない場合：
    - 解錠番号は表示せず、本人確認や予約選択を促すフォーム／メッセージに切り替える

### 5.8 フローG：店舗LINEへの丸投げ店舗

1. `shop_options.booking_redirect_to_store_line = 1` の店舗
2. HOMEタブ／BOOKINGタブの予約系スロットは基本的に `SLOT_STORE_LINE_JUMP` のみ有効
3. 押下時は `shop_links(link_type='store_line_add')` 等のURLへ遷移
4. 予約確認／解錠番号については総合LINE側でも提供するかどうかを別途ポリシーで決定

## 6. 通知 / 外部連携

### 6.1 tickets との連携

- チケット起票の基本ルールは HELP_UI と共通：
    - AI_CORE の `escalation_required` or 閾値超過 or ユーザー明示希望によって `tickets` にレコード作成。
- LINE_UI固有：
    - `tickets.source_channel = "line"`
    - `tickets.shop_id` に会話文脈の店舗（Cレイヤ）をセット
    - 必要に応じて `tickets.metadata` に `line_user_id` や `ai_session_id`、
        
        「LINEトーク履歴参照用URL」等を格納（PII方針に従う）
        

### 6.2 LINE返信／メール返信の分岐

- 一次対応（AI／即時人手）は基本的に LINE トーク上で行う。
- エスカレーション後の二次対応チャネル（LINE / メール / 電話）は、
    
    **加盟店／本部が shopポータル／aiポータル上で判断**する。
    
- AIは `ai_decision` などに
    
    「この案件は電話推奨」「LINEで返しても問題なし」などの**おすすめ情報**を付与することはあるが、
    
    実際にどのチャネルを使うかは人間が決定する（自動判断しない）。
    

### 6.3 sys_events へのログ方針（骨）

- LINE_UI 上の主要操作（リッチメニュースロットタップ／LIFF起動／解錠番号表示など）は、
    
    `sys_events` に `component="line_ui"` として記録する方針とする。
    
- 具体的なイベント名／payloadスキーマは、ログ／分析設計フェーズで定義し、
    
    本フェーズでは「LINE_UI も sys_events による計測対象とする」方針のみを定める。
    

## 7. セキュリティ・権限

### 7.1 LINE ID と会員情報の扱い

- `line_user_id` は「外部チャネル固有ID」として扱い、
    
    Aレイヤ（customer_links）で STORES顧客IDと名寄せする。
    
- LINE_UI 側では、必要な場面でのみ STORES 顧客情報／予約情報を API_COMM 経由で参照し、
    
    不要なPIIを長期保存しない。
    
- `line_user_shop_links` は「店舗との関係」だけを保持し、
    
    氏名・電話番号などのPIIは保持しない。
    

### 7.2 PII・ログポリシー

- AIログ（ai_turns／ai_sessions）は AI_CORE の方針に従い、
    
    一定期間後にマスキング／集約される。
    
- `line_user_id` の保存方法（平文 or ハッシュ）は
    
    `line_ui.hash_line_user_id` パラメータで切り替え可能とする。
    
- Webhook生データの生ログは短期（数日）でローテーションし、
    
    長期は抽象化済みログ（LineInternalEvent／sys_events）を利用する。
    

### 7.3 権限・運用

- LINE公式アカウント／LIFFの発行・設定管理は本部（HQ）のみ。
- 店舗（オーナー／スタッフ）は
    - LINE_UIの文言・頻度・一部スロットの有効／無効は変更できても、
    - Webhook URL／LIFF ID／リッチメニュー alias の根本設定は変更できない前提とする。

## 8. 未決課題 / 次フェーズへの引き継ぎ

1. **line_user_shop_links の詳細DDL／インデックス最適化**
    - 現行案で問題ないか、DB_CORE／インフラ側で検証が必要。
2. **予約系 link_type／option_code の正式名称とコメント**
    - `db_fields.yaml`／DB_BOX_INTENT に追加し、他フェーズ（予約集計・分析）との整合を取る。
3. **リッチメニュー画像と座標設計**
    - `slot_position` と実際のピクセル座標をどう紐づけるかの設計は UI 実装フェーズで必要。
4. **クーポン機能（coupon_center）の BOX とフロー**
    - クーポンのSoR（自前DB／STORESクーポン／外部ツール）と
        
        LINE_UIでの出し方（タイミング・対象店舗）を別フェーズで定義する。
        
5. **ログ／分析イベントの具体設計**
    - `sys_events` に送るイベント名／payload を定義し、
        
        BI／ダッシュボードにどう載せるかはログ設計フェーズで扱う。
        
6. **セキュリティレビュー**
    - 解錠番号表示フローを含め、
        
        RemoteLOCK／STORESとの連携条件・リトライ制限・不正利用検知ロジックなどは、
        
        別途セキュリティレビューが必要。
        

---

以上をもって、PHASE_03_LINE_UI の**骨レベル仕様**をひとまず完成とする。

文言／UIラフ／パラメータ値は、`LINE_UI_POLICY.md`／`LINE_UI_DEFAULT_TEXTS.md`／`LINE_UI_DEFAULT_PARAMS.md` で肉付けしていく。
