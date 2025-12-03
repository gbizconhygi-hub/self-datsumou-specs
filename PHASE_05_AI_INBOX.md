# PHASE_05_AI_INBOX - 仕様設計書 (v1.1 Draft｜ロイヤリティ特例差分反映版)

フェーズ番号: Phase 5

フェーズ名: AI-INBOX（HQ向け INBOX／本部コンソール）

対応要素: `ai.self-datsumou.net`（HQポータル / AI-INBOX）

---

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 5：AI-INBOX）は、`ai.self-datsumou.net` 上に

- 全チャネル（help.／line.／shop.／OP_MAIL※）から流入する**問い合わせ／申請／アラートを統合管理する INBOX**
- AIコア（意図分類／FAQ／テンプレ／トーン）を HQ から調整する**AI設定コンソール**
- owners／shops／fee_*／sales_records／invoices 等の SoR を HQ目線で扱う**FC運営ビュー（オーナー・店舗・請求管理）**

を構築する。

※ OP_MAIL（電話代行メール解析）は Phase7 実装対象だが、

本フェーズで **チケットとしての見え方／優先度** だけ先に決めておく。

### 1.2 本フェーズで扱う範囲

本フェーズで仕様確定するもの：

1. HQ向け INBOX 画面
    - tickets 一覧（SLA／放置アラート／AIフラグ／HQタグ）
    - 顧客エスカレーション（HELP／LINE）・SHOP_PORTAL からの問い合わせ・OP_MAIL由来チケットの扱い
2. チケット詳細／対応画面
    - ticket_messages／ai_turns／関連オブジェクトの統合ビュー
    - AIアシスト（要約／返信ドラフト）※送信は必ず人間経由
    - incident グルーピング（物理マージではなく、関連付け）
3. FAQ／intent／AI設定管理 UI
    - ai_intents_master／ai_qa_entries／ai_answer_templates／ai_core_params の編集 UI
    - HELP_UI／LINE_UI／SHOP_PORTAL／AI-INBOX 向け DEFAULT_TEXTS／DEFAULT_PARAMS との関係
4. オーナー／店舗／請求管理ビュー
    - owners／owner_contacts／shops／shop_internal_info／fee_*／sales_records／invoices／invoice_items の HQビュー
    - 店舗リスクフラグ／内部メモ表示
    - 月次請求サイクル（draft 自動生成＋手動発行）
    - **ロイヤリティ特例ルール（shop_royalty_relief_rules）とアラート** ←今回差分
    - 既存請求書命名規則を用いた PDF 発行
5. MTG／電話相談管理
    - hq_meeting_slots／hq_meetings を利用した MTG／電話枠の管理
    - SHOP_PORTAL からの MTG予約（direct_slot／request_and_confirm）のインターフェース
6. 補助ビュー／メタ情報
    - オーナー単位タイムライン（問い合わせ／請求／申請／MTG 等の時系列ビュー）
    - パラメータ変更履歴ビュー（sys_params／shop_params／shop_options／AI設定）
    - HQタグ（ticket_tag_master／ticket_tags）によるチケットラベル付け

本フェーズでは扱わないもの（他フェーズで詳細化）：

- KPIダッシュボード（売上・AI自己解決率・店舗別KPI グラフ等） → Phase6 KPI_ANALYTICS
- STORES売上の自動取込＆RemoteLOCK本格統合 → 拡張仕様（Phase9相当：現プロジェクト外）
- 実運用後のAIチューニング（Phase8）：本番リリース後の運用フェーズ

---

## 2. 前フェーズからの継承ポイント（抜粋）

### 2.1 ドメイン構成と役割

- help.：顧客向け Web チュートリアル（AI＋メール）
- line.：顧客向け LINE チュートリアル（AI＋LIFF／リッチメニュー）
- shop.：加盟店ポータル（オーナー向けステータス／請求／AI相談／申請）
- ai.：本部ポータル（AI-INBOX／AI設定／KPI入口）
- assets.：静的アセット／PDF／画像等の配信サーバー

### 2.2 コア層との関係

- SYS_CORE（Phase1）：
sys_params／api_call_logs／sys_events／audit_logs／tickets／通知共通処理 等
- AI_CORE（Phase2）：
AI_CORE_REQUEST／AI_CORE_RESPONSE、ai_intents_master／ai_qa_entries／ai_sessions／ai_turns／ai_logs 等
- API_COMM（Phase2）：
STORES予約 API／通知ラッパ（ChatWork／メール／SMS） 等の業務 API
- HELP_UI／LINE_UI（Phase3）：
tutorial_api、本人確認ポリシー、名寄せ（customer_links）
- SHOP_PORTAL（Phase4）：
オーナーポータル UI、owner_ai、備品発注、キャンペーン／デザイン依頼、請求表示

AI-INBOX はこれらのレイヤに **新しい業務ロジックを極力持たず**、

- 「各レイヤが作った tickets／ai_sessions／sales_records／invoices を HQ 目線で統合閲覧・操作する UI」
- 「AI_CORE／SYS_CORE／API_COMM／SHOP_PORTAL のパラメータ・マスタを編集する UI」

として振る舞う。

### 2.3 使用テーブル（BOX）

AI-INBOX が主に利用する BOX：

- owners／owner_contacts
- shops／shop_secrets／shop_links／shop_internal_info
- options_master／shop_options
- sys_params／shop_params
- sales_records
- invoices／invoice_items
- tickets／ticket_messages
- ai_sessions／ai_turns／ai_logs
- hq_meeting_slots／hq_meetings
- shop_supplies／shop_fee_addons／shop_fee_package_assignments
- hq_chat_threads／hq_chat_messages／hq_chat_links
- portal_users（HQユーザー認証）
- shop_royalty_history（既存）＋ **shop_royalty_relief_rules（本フェーズで追加）**

---

## 3. [ai.self-datsumou.net](http://ai.self-datsumou.net/) の役割と IA

### 3.1 役割

- HQスタッフ専用ポータル
- 主な機能群：
    - INBOX（チケット一覧／フィルタ／SLA／タグ）
    - チケット詳細／対応画面（AIアシスト付き）
    - FAQ／intent／AI設定管理
    - オーナー／店舗／請求／ロイヤリティ特例管理
    - MTG／電話相談管理
    - タイムライン／パラメータ履歴／タグ管理

### 3.2 グローバルナビ（IA）

想定ナビゲーション：

1. INBOX
2. チケット検索／履歴
3. AI・FAQ管理
4. オーナー・店舗・請求
5. MTG・電話相談
6. システム・設定（監視／履歴）

---

## 4. INBOX 一覧（tickets）

### 4.1 目的

- 全チャネルから発生した「要対応」チケットを HQ が漏れなく把握し、
優先度順に処理できる状態を作る。
- 顧客エスカレーション／オーナー相談／OP_MAIL緊急電話代行などを一元管理する。

### 4.2 対象チケット

標準では次の tickets を一覧対象とする：

- `status IN ('open','pending_owner','pending_admin')`
- かつ `last_message_at >= (現在日付 - ai_inbox.default_list_days)`

チャネル別タブ例：

- すべて
- 顧客起点
    - `source IN ('stores_form','customer_line','hp_form')`
- 加盟店起点
    - `source = 'portal_owner'`
- 本部起点／OP_MAIL
    - `source = 'portal_admin'`（OP_MAIL由来を含む）
- システム／アラート
    - category or parent_category で `system_*`／重大クレーム等

OP_MAIL（電話代行）について：

- メール解析・ticket 化は Phase7(OP_MAIL) で実装。
- AI-INBOX では、`source='portal_admin'` かつ `category` に `operator_mail` を含むものを
「電話代行（緊急）」として強調表示する。

### 4.3 一覧カラム

少なくとも以下を表示：

- 受付日時：created_at
- 最終更新：last_message_at
- ステータス：status
- 起点：source（HELP／LINE／加盟店ポータル／OP_MAIL など）
- 店舗：[shops.name](http://shops.name/)
- オーナー：owners.contract_holder_name
- 件名：subject
- 最終送信者：last_sender（customer／owner／admin／system）
- AIステータス：AIエスカレーション／自己解決など
- SLA：残り時間または期限切れ／重大度

### 4.4 操作

- 行クリック → チケット詳細画面へ遷移
- チェックボックス選択 → 一括操作（担当者割当／ステータス変更 等）
- フィルタ：
    - 日付範囲、source、status、shop、owner、category、tag（HQタグ）、AIエスカレーション有無 など

### 🔧 パラメータ候補（INBOX一覧）

- `ai_inbox.default_list_days`（int）
- `ai_inbox.page_size`（int）
- `ai_inbox.autorefresh_interval_sec`（int）
- `ai_inbox.show_ai_flags`（bool）
- `ai_inbox.op_mail_badge_category_prefix`（string）

---

## 5. チケット詳細・対応画面

### 5.1 目的

- 1件の案件について「顧客／オーナー／HQ／AI」のやりとりと、
店舗・請求・MTG 等の関連情報を 1画面で把握し、HQが判断・返信できる状態を作る。

### 5.2 画面構成

- 左ペイン：メッセージタイムライン
    - ticket_messages（sender_role=customer／owner／admin／system）
    - ai_turns（AIの応答ログ）
    - 時系列に統合表示し、AIメッセージには「AI」ラベルを付与
- 右上：コンテキストパネル
    - 店舗：shops（ステータス／リスクレベル／内部メモ）
    - オーナー：owners／owner_contacts
    - 関連 AI セッション：ai_sessions／ai_logs
    - 関連 MTG：hq_meetings
    - 関連申請：hq_requests／campaign_requests／design_requests
    - 請求情報：最新 invoices（当該 owner／shop）
- 右下：操作パネル
    - ステータス変更
    - 担当者変更
    - HQタグの追加／削除
    - AIアシスト（要約／返信ドラフト）

### 5.3 AIアシスト（HQ向け）

- AI_CORE_REQUEST を `channel="ai"`, `user_role="hq"` で呼び出し、
    - 「スレッド要約」
    - 「最新メッセージに対する返信ドラフト」
    を生成する。
- AI_CORE_RESPONSE.reply_type は `draft` とし、AIから直接送信しない。
- HQスタッフはドラフトを編集し、送信チャネル（メール／LINE／SHOP_PORTAL）を選択して送信する。
- トーンは「ボット感NG」「人間が丁寧に説明している」方針に合わせる。

### 5.4 incident グルーピング（関連付け）

- tickets に `incident_key`（任意文字列）を追加する。
- 同一 incident_key を持つ tickets を「同じ案件グループ」として扱う。
- 挙動：
    - チケット詳細画面に「関連チケット」セクションを表示し、同じ incident_key の tickets を一覧表示。
    - HQが「関連付け」ボタンを押すことで incident_key を設定／解除できる。
    - 類似候補（同じ customer_links／同じ shop／近い日時／似たintent）は AI が提示するが、
    incident_key 設定は人間操作のみ（自動マージしない）。

### 🔧 パラメータ候補（チケット詳細／AIアシスト）

- `ai_inbox.ai_draft_temperature`（float）
- `ai_inbox.ai_summary_max_tokens`（int）
- `ai_inbox.ai_assist_enable`（bool）
- `ai_inbox.incident_suggestion_enable`（bool）

---

## 6. FAQ／intent／AI設定管理 UI

### 6.1 目的

- AI_CORE の「頭脳」（意図分類／FAQ／テンプレ／トーン）を、
コード変更なしに HQ から調整できるようにする。

### 6.2 対象テーブル

- ai_intents_master
- ai_qa_entries
- ai_answer_templates
- ai_core_params／ai_settings
- ai_tone_policies など

### 6.3 画面構成

1. インテント一覧
    - intent_key／super_category／parent_category／routing_policy／allowed_channels
    - routing_policy（auto_ok／human_required／escalate_prefer）を変更可能
2. Q&A一覧
    - チャネル別（help／line／shop／ai）の FAQ／Q&A テンプレートを表示／編集
3. AIプロファイル設定
    - モデル名／temperature／max_tokens／信頼度しきい値／NGワード など
4. UIテキスト／パラメータ編集リンク
    - HELP_UI／LINE_UI／SHOP_PORTAL／AI-INBOX の DEFAULT_TEXTS／DEFAULT_PARAMS へジャンプ

### 🔧 パラメータ候補（AI設定管理）

- `ai_inbox.max_intents_per_page`
- `ai_inbox.max_qa_entries_per_page`
- `ai_inbox.intent_edit_role`（通常：hq_admin）
- `ai_inbox.ai_param_edit_role`（通常：hq_admin）
- `ai_inbox.hot_edit_allowed`（bool）

---

## 7. オーナー／店舗／請求／ロイヤリティ特例ビュー

### 7.1 店舗所有区分（ownership_type）

`shops` に以下カラムを追加：

```sql
ALTER TABLE shops
  ADD COLUMN ownership_type ENUM('fc','hq','test')
    NOT NULL DEFAULT 'fc'
    COMMENT '店舗所有区分（FC／直営HQ／テスト）'
    AFTER shop_status;

```

- FC店舗（加盟店：請求対象）
- `hq` … 直営店舗（請求システムの対象外）
- `test` … テスト用店舗（請求・KPIから除外）

### 7.2 請求プロファイル（billing_profile）

`shops` に以下カラムを追加：

```sql
ALTER TABLE shops
  ADD COLUMN billing_profile ENUM('fc_standard','fc_semi_manual','none')
    NOT NULL DEFAULT 'fc_standard'
    COMMENT '請求プロファイル（通常FC／半直営・手動／請求対象外）'
    AFTER ownership_type;

```

- `fc_standard`
    - 通常の FC店舗。月次請求フロー（draft自動生成＋一括発行）にフル参加。
- `fc_semi_manual`
    - 半直営（特殊オーナー店舗）。
    - 下書き請求書は作るが、一括発行対象からデフォルトで除外。
- `none`
    - 請求対象外（直営 HQ 等）。請求書レコードを生成しない。

### 7.3 請求ヘッダ（invoices）差分

`invoices` に一括発行制御フラグを追加：

```sql
ALTER TABLE invoices
  ADD COLUMN issue_ready_flag TINYINT(1) NOT NULL DEFAULT 1
    COMMENT '一括発行対象フラグ（0=除外）'
    AFTER total_amount;

```

- `billing_profile='fc_standard'` の店舗 → draft生成時に `issue_ready_flag=1`
- `billing_profile='fc_semi_manual'` の店舗 → draft生成時に `issue_ready_flag=0`
- `billing_profile='none'` の店舗 → invoicesレコード自体を生成しない

### 7.4 請求明細（invoice_items）差分

`invoice_items` に店舗単位の紐付けを追加：

```sql
ALTER TABLE invoice_items
  ADD COLUMN shop_id BIGINT UNSIGNED NULL
    COMMENT 'どの店舗に紐づく明細か（FC店舗のみ）'
    AFTER invoice_id;

```

- 1枚の請求書（invoices）は owner＋billing_month 単位。
- invoice_items はその中で店舗単位の明細として積み上げる。
- ロイヤリティ／システム利用料／オプション／備品など、すべて `shop_id` をセットする。

### 7.5 月次請求サイクル

1. **毎月1日：draft 自動生成（cron）**
    - 対象：`ownership_type='fc'` かつ `billing_profile!='none'` の店舗を1つ以上持つオーナー。
    - owner＋billing_month（来月分ロイヤリティ）単位で `invoices` に draft を生成。
    - 以降、当月1日〜月末の自動計上項目（ロイヤリティ／システム／備品など）を `invoice_items` に店舗単位で追加。
2. **1日〜月末：自動積み上げ＋手動調整**
    - 自動：`sales_records`／`shop_supplies` 等から invoice_items に追加。
    - 手動：HQが AI-INBOX の請求書詳細画面から
        - 明細行の追加／修正／削除
        - 開業月の2か月分／日割り調整 などを行う（status=draft の間）。
3. **翌月1日：HQによる手動発行**
    - AI-INBOXの「請求書発行」画面で、前サイクル分 draft を一覧表示。
    - `issue_ready_flag=1` の請求書はデフォルトでチェックON（一括発行対象）。
    - HQが確認し、「一括発行」を押すと：
        - status を `sent` に変更
        - issue_date に当日／due_date は当月末（日付計算でUIに表示）
        - PDFを生成し、既存命名規則で保存：
            
            `【請求書】ハイジロイヤリティ等YYYY年MM月分_オーナー名御中（店舗名1,店舗名2,・・・)`
            
        - billing_email／owner_contacts.email へ通知
    - `billing_profile='fc_semi_manual'` の請求書 draft は `issue_ready_flag=0` で表示され、
        
        一括発行には含まれない（必要なら個別発行）。
        
4. **直営店舗**
    - `ownership_type='hq'`／`billing_profile='none'` の店舗は、
        
        本請求フローの対象外とする（請求書レコードを作成しない）。
        

### 7.6 ロイヤリティ特例ルール（shop_royalty_relief_rules）※今回差分

### 7.6.1 目的

- 「一時的にロイヤリティを下げるが、特定条件を満たしたら元に戻したい」
    
    といった **特例ルールの条件部分** をデータで管理する。
    
- 条件達成時に AI-INBOXにアラート（sys_events or tickets）を上げることで、
    
    HQが「戻しタイミング」を見逃さないようにする。
    
- ルール自体は追加／廃止しやすい「自由設計に近い箱」としておく。

### 7.6.2 DDL（追加テーブル）

```sql
CREATE TABLE shop_royalty_relief_rules (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  shop_id BIGINT UNSIGNED NOT NULL COMMENT '対象店舗ID',
  rule_type ENUM(
    'monthly_sales_over',        -- 月間売上が閾値を超えたら
    'months_since_open_over'     -- オープンからの経過月が閾値を超えたら
  ) NOT NULL COMMENT 'ルール種別',
  threshold_value BIGINT NOT NULL COMMENT '閾値（rule_typeに応じた解釈：金額 or 月数など）',
  target_royalty_history_id BIGINT UNSIGNED NULL
    COMMENT '条件達成時に「戻し」候補とするshop_royalty_history.id（NULLなら標準値に戻す運用）',
  alert_only TINYINT(1) NOT NULL DEFAULT 1
    COMMENT '1=アラートのみ（ロイヤリティ変更は必ず人手で行う）',
  is_active TINYINT(1) NOT NULL DEFAULT 1 COMMENT 'ルール有効フラグ',
  notes VARCHAR(255) NULL COMMENT 'ルールの説明（例：売上30万円超えたら通常ロイヤリティに戻す 等）',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  CONSTRAINT fk_relief_shop FOREIGN KEY (shop_id) REFERENCES shops(id),
  CONSTRAINT fk_relief_history FOREIGN KEY (target_royalty_history_id) REFERENCES shop_royalty_history(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 7.6.3 評価タイミングとロジック

- 評価実行は月次バッチ（Phase6/7で設計する `cron_sales_import` or 専用cron）に組み込む。
- 挙動（例）：
    1. `is_active=1` の `shop_royalty_relief_rules` を全件取得。
    2. ルールごとに以下を評価：
        - `rule_type='monthly_sales_over'`
            
            → 対象店舗の当月売上合計（sales_records集計）が `threshold_value` を超えているか。
            
        - `rule_type='months_since_open_over'`
            
            → `shops.open_date` からの経過月が `threshold_value` を超えているか。
            
    3. 条件達成時：
        - 自動でロイヤリティは変更しない（v1.4では `alert_only=1` を標準とする）。
        - アラート方法：
            - sys_events に `event_type='royalty_relief_condition_met'` を記録
                
                or
                
            - tickets に `source='system'`／`category='royalty_relief'` のチケットを自動起票（INBOXに乗る）

### 7.6.4 AI-INBOX側 UI（店舗詳細 → ロイヤリティセクション）

店舗詳細画面に **「ロイヤリティ設定／特例」パネル** を追加：

- 表示：
    - 現在のロイヤリティ（shops.royalty_*）
    - 過去履歴（shop_royalty_history 列挙）
    - 特例ルール一覧（shop_royalty_relief_rules）
        - rule_type／threshold_value／is_active／notes／「条件達成済み？」フラグ（直近評価結果）
- 操作（v1.4）：
    - 特例ルールの新規登録／編集／有効・無効切り替え
        - rule_type（プルダウン）
        - threshold_value
        - target_royalty_history_id（任意）
        - alert_only（チェックボックス：デフォルトON）
    - アラート（チケット or イベント）を確認した上で
        
        「標準ロイヤリティに戻す」「指定のhistoryに戻す」ボタンでロイヤリティを戻す
        
        - 裏では shop_royalty_history の end_dateセット＋新規history挿入
        - 変更内容は audit_logs に記録

### 🔧 パラメータ候補（請求・ロイヤリティ関連）

- `ai_inbox.owner_list_page_size`
- `ai_inbox.shop_list_page_size`
- `ai_inbox.default_billing_month_range`
- `ai_inbox.invoice_default_due_days`
- `ai_inbox.invoice_issue_time`
- `ai_inbox.invoice_notify_email_template_key`
- `ai_inbox.invoice_show_on_portal_profiles`（デフォルト: ["fc_standard"]）
- `ai_inbox.royalty_relief_check_enable`（bool）
- `ai_inbox.royalty_relief_check_lookback_months`（int）
- `ai_inbox.royalty_relief_alert_channel`（"ticket" / "sys_event"）
- `ai_inbox.royalty_relief_rule_edit_role`（通常: "hq_admin"）

---

## 8. MTG／電話相談管理（hq_meeting_slots／hq_meetings）

※ここは前回案と変わらず。要旨のみ：

- meeting_type：`routine_checkin`／`trouble_consult`／`fc_contract`／`marketing_planning`／`phone_light` など
- 各 meeting_type に `booking_mode`（direct_slot／request_and_confirm）を定義。
- SHOP_PORTAL からの予約 → hq_meeting_slots／hq_meetings を利用。
- AI-INBOXからチケット詳細画面経由で「この案件でMTG提案」ボタン。

🔧 主なパラメータ：`hq_mtg.default_visible_days`／`hq_mtg.max_requests_per_month_per_shop`／`hq_mtg.ai_suggestable_types`／`hq_mtg.enable_phone_light`

---

## 9. オーナー単位タイムラインビュー

- owners 詳細画面に「タイムライン」タブを追加。
- 対象イベント：
    - tickets
    - hq_meetings
    - invoices
    - shop_supplies
    - hq_requests／campaign_requests／design_requests
    - リスクレベル変更（shop_internal_info＋audit_logs）
- 追加テーブルは不要。owners.id をキーに複数テーブルからイベントを集約して時系列表示。

🔧 パラメータ：`ai_inbox.owner_timeline_lookback_days`（int）

---

## 10. パラメータ変更履歴ビュー

- sys_params／shop_params／shop_options／AI設定の変更は audit_logs に記録（entity／entity_id／before／after）。
- AI-INBOX「システム・設定」画面に「パラメータ変更履歴」タブを追加し、閲覧可能にする。
- 店舗詳細画面から「この店舗に関するパラメータ履歴」だけ絞るビューも提供。

🔧 パラメータ：`ai_inbox.param_history_lookback_days`（int）

---

## 11. HQタグ（ticket_tag_master／ticket_tags）

### 11.1 目的

- intent／category だけでは表現しづらい「HQ内部用分類」（RemoteLOCK案件／解約検討／法務確認 等）を柔軟に付与する。

### 11.2 DDL

```sql
CREATE TABLE ticket_tag_master (
  id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  tag_key VARCHAR(64) NOT NULL UNIQUE COMMENT '内部キー',
  label VARCHAR(255) NOT NULL COMMENT '表示名',
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE ticket_tags (
  ticket_id INT(11) NOT NULL,
  tag_id INT(11) NOT NULL,
  PRIMARY KEY (ticket_id, tag_id),
  CONSTRAINT fk_ticket_tags_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
  CONSTRAINT fk_ticket_tags_tag FOREIGN KEY (tag_id) REFERENCES ticket_tag_master(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 11.3 画面

- チケット詳細画面で tag_master から複数選択。
- INBOX一覧にタグ列（バッジ）を表示。
- フィルタ条件としてタグを指定可能。

🔧 パラメータ：`ai_inbox.tag_edit_role`／`ai_inbox.max_tags_per_ticket`

---

## 12. セキュリティ・権限・ログ

- 認証：`portal_users` を利用。`role='admin'` を HQ として ai. にアクセス可能。
- 権限：
    - hq_admin … 全機能・全パラメータ編集可
    - hq_operator … INBOX対応／タグ付け／FAQ採用可（パラメータやロイヤリティ特例ルールは不可 or 制限）
    - hq_viewer（将来） … 閲覧のみ
- 重要操作（請求書発行／ロイヤリティ変更／リスクレベル変更／パラメータ変更）は audit_logs に必ず記録。

---

## 13. パラメータ化ポリシー（Phase5 補足）

- 通知タイミング・閾値・一覧件数・フィルタ条件・ロイヤリティ特例チェック条件など、
    
    固定値は sys_params／shop_params に外だしする。コード直書きは禁止。
    
- 新規機能セクションごとに「🔧 パラメータ候補」を列挙し、
    
    AI_INBOX／KPI／CRON_SYS用 DEFAULT_PARAMS に集約する。
    

---

## 14. 未決課題・他フェーズへの引き継ぎ

### 14.1 未決課題（Phase5 内で詰める余地があるもの）

- 開業月の「2か月分＋日割り」ロイヤリティ自動化レベル
- 半直営（fc_semi_manual）の実運用詳細（どこまで invoice_items を積み上げるか／内部管理にどこまで使うか）
- incident グルーピングの UX 細部（候補表示基準）

### 14.2 他フェーズへの引き継ぎ

- Phase6（SALES_LINK／KPI_ANALYTICS）
    - sales_records と tickets／ai_logs／shop_royalty_relief_rules を使ったKPI設計。
- Phase7（OP_MAIL／CRON_SYS）
    - ロイヤリティ特例チェックcron／OP_MAILメール解析→tickets化のバッチ設計。
- 拡張フェーズ（STORES自動取込・RemoteLOCK本格連携）
    - APIベースで sales_records を自動同期し、RemoteLOCK障害案件をKPIに反映する拡張。

以上、ロイヤリティ特例ルール差分を反映した PHASE_05_AI_INBOX 仕様書 v1.1 Draft とする。
