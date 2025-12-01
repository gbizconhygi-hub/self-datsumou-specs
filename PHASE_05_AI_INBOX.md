# PHASE_05_AI_INBOX - 仕様設計書 (v1.0 Draft)

フェーズ番号: Phase 5

フェーズ名: AI-INBOX（HQ向け INBOX／本部コンソール）

対応要素: `ai.self-datsumou.net`（HQポータル / AI-INBOX）:contentReference[oaicite:0]{index=0}

---

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 5：AI-INBOX）は、`ai.self-datsumou.net` 上に

- 全チャネル（help./line./shop./OP_MAIL）から流入する **問い合わせ／申請／アラートを統合管理する INBOX**
- AIコア（意図分類／FAQ／テンプレ／トーン）を HQ から調整する **AI設定コンソール**
- owners／shops／fee_*／sales_records／invoices 等の SoR を HQ目線で扱う **FC運営ビュー**

を構築する。:contentReference[oaicite:1]{index=1}

本フェーズのゴール：

- 顧客向けチュートリアル（HELP_UI／LINE_UI）、SHOP_PORTAL から発生した **tickets／ai_sessions** を、HQ が「抜け漏れなく・優先度順に」処理できる状態をつくる
- HQ が回答した内容・クローズした案件を **FAQ／AIナレッジに還元** するループを用意する
- owners／shops／請求・備品・キャンペーン・DR 等を「オーナー単位のタイムライン」で捉えられるようにする
- ロイヤリティ等の請求書発行フローを **AI-INBOX を起点に一元管理** する（Cron は下書き生成まで）

### 1.2 本フェーズで扱う範囲

本フェーズで仕様確定するもの：

1. HQ向け INBOX 画面
    - tickets 一覧（SLA／放置アラート／AIフラグ付き）
    - 顧客 Help／LINE からの本部エスカレーション、SHOP_PORTAL からの問い合わせ、OP_MAIL（電話代行メール）由来チケットの扱い
2. チケット詳細／対応画面
    - ticket_messages／ai_turns／関連オブジェクトの統合ビュー
    - AIアシスト（要約／返信ドラフト）※送信は必ず人間経由
    - incident グルーピング（重複「統合」ではなく、関連付け）
3. FAQ／intent／AI設定管理 UI
    - ai_intents_master／ai_qa_entries／ai_answer_templates／ai_core_params の編集 UI
    - HELP_UI／LINE_UI／SHOP_PORTAL／AI-INBOX 向け DEFAULT_TEXTS／DEFAULT_PARAMS との関係
4. オーナー／店舗／請求管理ビュー
    - owners／owner_contacts／shops／shop_internal_info／fee_*／sales_records／invoices／invoice_items の HQビュー
    - 店舗リスクフラグ／内部メモ表示
    - 月次請求サイクル（draft 自動生成＋手動発行）と、既存請求書命名規則を用いた PDF 発行
5. MTG／電話相談管理
    - hq_meeting_slots／hq_meetings を利用した MTG／電話枠の管理
    - SHOP_PORTAL からの MTG予約（direct_slot／request_and_confirm）のインターフェース
6. 補助ビュー／メタ情報
    - オーナー単位タイムライン（問い合わせ／請求／申請／MTG 等の時系列ビュー）
    - パラメータ変更履歴ビュー（sys_params／shop_params／shop_options／AI設定）
    - HQタグ（ticket_tag_master／ticket_tags）によるチケットラベル付け

本フェーズでは扱わないもの（他フェーズで詳細化）：

- KPIダッシュボード（売上・AI自己解決率・店舗別KPI グラフ等） → KPI_ANALYTICS フェーズ
- STORES売上の自動取込・RemoteLOCK本実装 → Phase9（拡張統合層）
- 電話代行（OP_MAIL）のメール解析ロジック（受信トリガ／スコアリング）は Phase7（OP_MAIL）で詳細化
    - 本フェーズでは `tickets.source='operator_mail'` の扱いのみ規定する

---

## 2. 前フェーズからの継承ポイント（抜粋）

詳細は Phase0〜4 の仕様書および `db_fields.yaml` を SoR とし、本章では AI-INBOX に直結するもののみ記載する。

### 2.1 ドメイン構成と役割

- help.：顧客向け Web チュートリアル（AI＋メール）:contentReference[oaicite:4]{index=4}
- line.：顧客向け LINE チュートリアル（AI＋LIFF／リッチメニュー）:contentReference[oaicite:5]{index=5}
- shop.：加盟店ポータル（オーナー向けステータス／請求／AI相談／申請）:contentReference[oaicite:6]{index=6}
- ai.：本部ポータル（AI-INBOX／AI設定／KPI入口）
- assets.：静的アセット／PDF／画像等の配信サーバー:contentReference[oaicite:7]{index=7}

### 2.2 コア層との関係

- SYS_CORE（Phase1）：
    - sys_params／api_call_logs／sys_events／tickets／通知コア 等の共通基盤
- AI_CORE（Phase2）：
    - AI_CORE_REQUEST／RESPONSE、ai_intents_master／ai_qa_entries／ai_sessions／ai_turns／ai_logs 等の AI頭脳
- API_COMM（Phase2）：
    - STORES予約 API／通知ラッパ（ChatWork／メール／SMS） 等の業務 API:contentReference[oaicite:10]{index=10}
- HELP_UI／LINE_UI（Phase3）：
    - tutorial_api、本人確認ポリシー、名寄せ戦略（customer_links）:contentReference[oaicite:11]{index=11}
- SHOP_PORTAL（Phase4）：
    - オーナーダッシュボード／店舗ダッシュボード／owner_ai／MTG予約／備品発注／キャンペーン／デザイン依頼／請求表示

AI-INBOX はこれらのレイヤに **新しい業務ロジックを極力持たず**、

- 「既存レイヤが作った tickets／ai_sessions／sales_records／invoices を HQ 目線で統合閲覧・操作する UI」
- 「AI_CORE／SYS_CORE／API_COMM／SHOP_PORTAL のパラメータ・マスタを編集する UI」

として振る舞う。

### 2.3 DB BOX（BOX_INTENT）

AI-INBOX が主に利用する BOX：

- owners／owner_contacts（オーナー情報）:contentReference[oaicite:13]{index=13}
- shops／shop_secrets／shop_links／shop_internal_info
- options_master／shop_options（店舗オプション）
- sys_params（scope=global/shop/channel）
- sales_records（売上／体験フラグ）:contentReference[oaicite:14]{index=14}
- invoices／invoice_items（請求ヘッダ／明細）:contentReference[oaicite:15]{index=15}
- tickets／ticket_messages（問い合わせ）:contentReference[oaicite:16]{index=16}
- ai_sessions／ai_turns／ai_logs（AI会話ログ・要約）
- hq_meeting_slots／hq_meetings（本部MTG枠／予約）:contentReference[oaicite:17]{index=17}
- shop_supplies／shop_fee_addons／shop_fee_package_assignments（備品／オプション料金）
- hq_chat_threads／hq_chat_messages／hq_chat_links（オーナー向け AI相談・HQチャット）:contentReference[oaicite:19]{index=19}
- portal_users（本部／オーナー共通ログインユーザ）:contentReference[oaicite:20]{index=20}

---

## 3. 対応ドメイン／コンポーネント

### 3.1 [ai.self-datsumou.net](http://ai.self-datsumou.net/) の役割

- HQスタッフ専用ポータル
- 主な機能群：
    - INBOX（チケット一覧／フィルタ／SLA）
    - チケット詳細／対応画面（AIアシスト付き）
    - FAQ／intent／AI設定管理
    - オーナー／店舗／請求管理
    - MTG／電話相談管理
    - 補助ビュー（オーナータイムライン／パラメータ履歴／タグ）

### 3.2 グローバルナビ（IA）

想定ナビゲーション：

1. INBOX
    - デフォルト画面。全チャンネルからの未解決チケット一覧。
2. チケット検索／履歴
    - 期間／店舗／オーナー／チャネル／カテゴリ／タグで検索。
3. AI／FAQ 管理
    - intent／Q&A／テンプレ／AIパラメータ／AIトーンの編集。
4. オーナー・店舗・請求
    - owners／shops／fee_*／sales_records／invoices の閲覧・編集。
5. MTG・電話相談
    - hq_meeting_slots／hq_meetings の管理、SHOP_PORTAL からの予約確認。
6. システム・設定
    - パラメータ履歴／監視（api_call_logs／sys_events）簡易ビュー。
    - portal_users の HQ側利用ルール（本仕様では DDL変更無し）。

---

## 4. INBOX 一覧（tickets）

### 4.1 目的

- 全チャネルから発生した「要対応」チケットを HQ が漏れなく把握し、優先度順に処理できる状態を作る。
- 顧客エスカレーション／オーナー相談／OP_MAIL緊急電話代行などを一元管理する。

### 4.2 対象チケット

標準では次の tickets を一覧対象とする：

- `tickets.status IN ('open','pending_owner','pending_admin')`:contentReference[oaicite:21]{index=21}
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
    - category や intent から `system_*`／重大クレーム系を絞り込む

OP_MAIL（電話代行報告メール）について：

- Phase4／base仕様では「電話代行オペレーター報告メールの解析は Push 型トリガで ticket 化」とされている。
- 本フェーズでは、解析後のチケットは `tickets.source='portal_admin'` として扱う。
    - `category` で `operator_mail` などのサブ種別を持たせ、INBOX上では「電話代行（緊急）」バッジを表示。

### 4.3 一覧カラム

最低限、以下を表示：

- 受付日時：tickets.created_at
- 最終更新：tickets.last_message_at
- ステータス：tickets.status
- 起点：source（ユーザー向けラベルに変換）
- 店舗：[shops.name](http://shops.name/)（shop_id が null の場合は「店舗未特定」）
- オーナー：owners.contract_holder_name
- 件名：tickets.subject
- 最終送信者：tickets.last_sender（customer／owner／admin）
- AIステータス：
    - 「AIのみでクローズ候補」／「AIがエスカレーションした」等（ai_logs／ai_sessionsに基づく）
- SLA：残り時間 or 期限切れラベル（due_at 計算結果）

### 4.4 操作

- 行クリック → チケット詳細画面へ遷移
- チェックボックス選択 → 一括操作（担当者割当／ステータス変更 等）
- フィルタ：
    - 日付範囲、source、status、shop、owner、category、last_sender
    - AIエスカレーション有無、タグ（後述 ticket_tags）

### 🔧 パラメータ候補（INBOX一覧）

- `ai_inbox.default_list_days`
    - 種別: int（日数）
    - 初期値: 7
    - 説明: INBOXに表示するデフォルトの期間（過去何日分）
- `ai_inbox.page_size`
    - 種別: int
    - 初期値: 50
    - 説明: 1ページあたりの表示件数
- `ai_inbox.autorefresh_interval_sec`
    - 種別: int（秒）
    - 初期値: 0（自動更新なし）
    - 説明: INBOX 自動更新間隔
- `ai_inbox.show_ai_flags`
    - 種別: bool
    - 初期値: true
    - 説明: AIエスカレーション／self_resolved フラグの表示 ON/OFF
- `ai_inbox.op_mail_badge_category_prefix`
    - 種別: string
    - 初期値: `"operator_mail"`
    - 説明: OP_MAIL由来チケットに対する category プレフィックス

---

## 5. チケット詳細・対応画面

### 5.1 目的

- 1件の案件について「顧客／オーナー／HQ／AI」のやりとりと、店舗・請求・MTG等の関連情報を 1画面で把握し、
HQが判断・返信できる状態を作る。

### 5.2 画面構成

- 左ペイン：メッセージタイムライン
    - ticket_messages（sender_role=customer／owner／admin／system）
    - ai_turns（AIの応答ログ）
    - 時系列に統合表示し、AIメッセージには「AI」ラベルを付与
- 右上：コンテキストパネル
    - 店舗：shops（ステータス／リスクレベル／内部メモ）
    - オーナー：owners／owner_contacts
    - 関連 AI セッション：ai_sessions／ai_logs（self_resolved／resolution_path 等）
    - 関連 MTG：hq_meetings（status／日時）
    - 関連申請：hq_requests／campaign_requests／design_requests
    - 請求情報：最新 invoices（当該 shop／owner の直近請求）
- 右下：操作パネル
    - ステータス変更（open／pending_owner／pending_admin／resolved／closed）
    - 担当者変更（assigned_to）
    - タグ付け（HQタグ：後述 ticket_tags）
    - AIアシスト（要約／ドラフト）

### 5.3 AIアシスト（HQ向け）

- AI_CORE_REQUEST を `channel="ai"`, `user_role="hq"` で呼び出し、
    - 「スレッド要約」
    - 「最新メッセージに対する返信ドラフト」
    を生成する。
- AI_CORE_RESPONSE.reply_type は `draft` とし、**AIから直接送信しない**。
    
    HQスタッフはドラフトを編集し、送信チャネル（メール／LINE／SHOP_PORTAL）を選択して送信。
    
- トーン方針は base仕様と同様、「ボット感NG」「人間が丁寧に説明しているような文体」を踏襲する。:contentReference[oaicite:23]{index=23}

### 5.4 incident グルーピング（重複統合ではなく関連付け）

- tickets に `incident_key`（任意文字列）を追加し、
    - 同一 incident_key を持つ tickets を「同じ案件グループ」とみなす
- AI-INBOX 上の挙動：
    - チケット詳細画面に「関連チケット」セクションを表示し、同じ incident_key のチケットを一覧
    - HQが「関連付け」ボタンを押すことで、現在のチケットの incident_key を親チケットと同一に設定
    - 間違えた場合は incident_key を外すことで元に戻せる
- 自動提案：
    - ai_turns／subject／customer_links／shop_id 等を元に「類似チケット候補」を提示するが、incident_key の設定は常に人間の操作で行う（自動設定しない）。

### 🔧 パラメータ候補（チケット詳細／AIアシスト）

- `ai_inbox.ai_draft_temperature`
    - 種別: float
    - 初期値: 0.4
    - 説明: HQ向け返信ドラフト生成時の温度
- `ai_inbox.ai_summary_max_tokens`
    - 種別: int
    - 初期値: 512
    - 説明: スレッド要約の最大トークン数
- `ai_inbox.ai_assist_enable`
    - 種別: bool
    - 初期値: true
    - 説明: HQ向けAIアシスト機能のON/OFF
- `ai_inbox.incident_suggestion_enable`
    - 種別: bool
    - 初期値: true
    - 説明: 類似チケット候補の自動表示ON/OFF

---

## 6. FAQ／intent／AI設定管理 UI

### 6.1 目的

- AI_CORE の「頭脳」（意図分類／FAQ／テンプレ／トーン）をコード変更無く HQ から調整できるようにする。:contentReference[oaicite:24]{index=24}

### 6.2 対象テーブル

- ai_intents_master（intent 定義・routing_policy 等）
- ai_qa_entries（チャネル別 Q&A テンプレ）
- ai_answer_templates（詳細回答テンプレ）
- ai_core_params／ai_settings（モデル・温度・トークン・禁止語など）:contentReference[oaicite:25]{index=25}
- ai_tone_policies（チャネル別トーンポリシー）:contentReference[oaicite:26]{index=26}

### 6.3 画面構成

1. intent 一覧
    - カラム：intent_key／super_category／parent_category／routing_policy／auto_answer可否
    - 編集可能項目：
        - routing_policy（auto_ok／human_required／escalate_prefer）
        - allow_channels（help／line／shop／ai）
2. Q&A一覧
    - intent ごとに ai_qa_entries をチャネル別表示（help／line／shop／ai）
    - Q&A編集画面：
        - 質問パターン（title／faq_text）
        - answer_template（AIが参考にするテンプレ）
3. AIプロファイル設定
    - モデル名（gpt-5.1 など）
    - temperature／top_p／max_tokens
    - 信頼度しきい値（intent_confidence_threshold）
    - 危険ワード／緊急ワード
4. UIテキスト／パラメータ編集リンク
    - HELP_UI／LINE_UI／SHOP_PORTAL／AI-INBOX 向けの DEFAULT_TEXTS／DEFAULT_PARAMS に遷移

### 🔧 パラメータ候補（AI設定管理）

- `ai_inbox.max_intents_per_page`（初期値: 100）
- `ai_inbox.max_qa_entries_per_page`（初期値: 100）
- `ai_inbox.intent_edit_role`（初期値: hq_admin のみ）
- `ai_inbox.ai_param_edit_role`（初期値: hq_admin のみ）
- `ai_inbox.hot_edit_allowed`（初期値: false）
    - true の場合、本番環境でも即時反映可／false の場合 draft→承認フローを挟む（将来拡張）

---

## 7. オーナー／店舗／請求管理ビュー

### 7.1 目的

- 「このオーナー／店舗は今どういう契約・料金・備品・請求状況か」を HQ が一目で把握できる状態を作る。
- SHOP_PORTAL でオーナーが見ている情報の「裏側」を HQ から操作・確認できるようにする。

### 7.2 店舗種別・請求プロファイル

### 7.2.1 店舗所有区分（ownership_type）

shops に以下カラムを追加する：

```sql
ALTER TABLE shops
  ADD COLUMN ownership_type ENUM('fc','hq','test')
    NOT NULL DEFAULT 'fc'
    COMMENT '店舗所有区分（FC／直営HQ／テスト）'
    AFTER shop_status;

```

- `fc` … FC店舗（加盟店）
- `hq` … 直営店舗（本部運営）
- `test` … テスト用店舗（請求・KPIから除外）

### 7.2.2 請求プロファイル（billing_profile）

```sql
ALTER TABLE shops
  ADD COLUMN billing_profile ENUM('fc_standard','fc_semi_manual','none')
    NOT NULL DEFAULT 'fc_standard'
    COMMENT '請求プロファイル（通常FC／半直営・手動／請求対象外）'
    AFTER ownership_type;

```

- `fc_standard`
    - 通常FC店舗。月次請求フローにフル参加（自動下書き＋一括発行）
- `fc_semi_manual`
    - 半直営（特殊オーナー店舗）。
    - 下書き請求書は作るが、一括発行対象からはデフォルトで除外（後述 issue_ready_flag）。
    - 将来「通常フローに乗せる」場合は billing_profile を `fc_standard` に変更すればよい。
- `none`
    - 請求対象外（直営 HQ など）。
    - 本システムでは請求書レコード自体を生成しない。

### 7.3 請求サイクル（invoices／invoice_items）

### 7.3.1 既存仕様との整合

- `spec_v1.4_base.md` では `cron_invoices.php` を「毎月1日 03:00／請求PDF生成・送信」と定義している。
    
    [spec_v1.4_base](https://github.com/gbizconhygi-hub/self-datsumou-specs/blob/6938e309f25d371c6443e8deb053fd6bb7d1f185/spec_v1.4_base.md)
    
- AI-INBOX 導入後の v1.4 では、この役割を以下のように **差分上書き** する：
    - cron_invoices.php は
        - **請求書の draft（下書き）生成** と
        - 自動計上項目（ロイヤリティ／備品 等）の積み上げ
            
            までを担当する。
            
    - 「PDF生成・送信」は AI-INBOX 側で HQ が 1日に行う **手動発行** によって実施する。

### 7.3.2 invoices テーブル差分

db_fields.yaml での invoices 定義：

[Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)

- owner_id, shop_id, billing_month, total_amount, status(`draft/sent/paid/void`), pdf_path, sent_at, paid_at, created_at, updated_at

本フェーズでは以下カラムを追加する：

```sql
ALTER TABLE invoices
  ADD COLUMN issue_ready_flag TINYINT(1) NOT NULL DEFAULT 1
    COMMENT '一括発行対象フラグ（0=除外）'
    AFTER total_amount;

```

運用ポリシー：

- `billing_profile='fc_standard'` の店舗 → draft生成時に `issue_ready_flag=1`（一括発行候補）
- `billing_profile='fc_semi_manual'` の店舗 → draft生成時に `issue_ready_flag=0`（デフォルトで一括発行対象外）
- `billing_profile='none'` の店舗 → invoices レコード自体を生成しない

※ `status='sent'` は「請求書発行済み／加盟店通知済み」を意味し、実装上は `sent` を `issued` と読み替えて良い。

### 7.3.3 invoice_items テーブル差分

既存定義：id／invoice_id／item_type／ref_id／description／amount／created_at

[Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)

店舗単位で明細を積み上げるため、以下カラムを追加する：

```sql
ALTER TABLE invoice_items
  ADD COLUMN shop_id INT(11) NULL
    COMMENT 'どの店舗に紐づく明細か（FC店舗のみ）'
    AFTER invoice_id;

```

- ロイヤリティ／システム利用料／オプション／備品／その他の行は、必ず `shop_id` をセットする。
- 1枚の請求書（invoices）は owner_id＋billing_month 単位で扱い、
    
    invoice_items はその中で店舗単位の明細として積み上げる。
    

### 7.3.4 月次サイクル

- 毎月1日 03:00（cron_invoices.php）
    - 対象：`ownership_type='fc'` かつ `billing_profile!='none'` の店舗を持つオーナー
    - owner_id＋billing_month（来月分）単位で `invoices` に draft を生成
    - 期間内（1日〜月末）の自動計上項目（ロイヤリティ／システム／備品など）を `invoice_items` に店舗単位で追加
- 1日〜月末
    - 自動項目の追加は随時行われる（sales_records／shop_supplies／shop_fee_addons etc. から）
    - HQスタッフは AI-INBOX の請求書詳細画面から
        - 明細行の追加・修正・削除
        - 開業月の2か月分／日割り調整など
            
            を行える（status が draft の間は自由）
            
- 翌月1日（AI-INBOXからの手動発行）
    - 「請求書発行」画面にて、前サイクル分の draft を一覧表示
    - `issue_ready_flag=1` のものはデフォルトでチェックON（一括発行候補）
    - HQが必要に応じてチェックを外し、「一括発行」ボタンを押す
    - 発行された請求書に対して：
        - status を `sent` に変更
        - `sent_at` に当日日時、支払期限（due_date 相当）は「当月末日」として UI 上／PDF 上で表現
        - PDF を生成し、既存命名規則に従って保存し、pdf_path に格納
        - billing_email／owner_contacts.email へ通知（SYS_CORE 通知仕様準拠）
            
            [Create PHASE_04_SHOP_PORTAL](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/d49e953804dfd09bcca2d7759da29cd3d86f1ce4)
            

### 7.3.5 PDF命名規則

既存運用ルールに準拠する：

> 【請求書】ハイジロイヤリティ等YYYY年MM月分_オーナー名御中（店舗名1,店舗名2,・・・）
> 
- YYYY年MM月分 … 請求対象月（例：2025年06月分）
- オーナー名 … owners.contract_holder_name
- （店舗名1,店舗名2,・・・）… 当該請求サイクルで請求対象となる FC店舗の shops.name をカンマ区切りで列挙

AI-INBOX／SHOP_PORTAL でDLする際も、このファイル名がそのままダウンロードされる。

### 7.4 オーナー／店舗ビュー

- オーナー一覧
    - owners／owner_contacts のリスト（contract_holder_name／owner_status／chatwork_room_id／billing_email 等）
    - フィルタ：owner_status（active/suspended/closed）
- 店舗一覧
    - shops のリスト（code／name／ownership_type／billing_profile／shop_status／region_name 等）
    - フィルタ：ownership_type（fc／hq／test）、shop_status
- オーナー詳細：タイムラインタブ（後述 9章）を含む

### 🔧 パラメータ候補（請求関連）

- `ai_inbox.invoice_default_due_days`
    - 初期値: 当月末固定（将来、日数指定に拡張可能）
- `ai_inbox.invoice_issue_time`
    - 初期値: 1日 09:00（UI上の「発行想定時間」表示）
- `ai_inbox.invoice_notify_email_template_key`
    - 請求書発行時に使用するメールテンプレキー
- `ai_inbox.invoice_show_on_portal_profiles`
    - 初期値: `['fc_standard']`
    - SHOP_PORTAL に請求書一覧を表示する billing_profile のセット

---

## 8. MTG／電話相談管理（hq_meeting_slots／hq_meetings）

### 8.1 MTG種別（meeting_type）と予約モード

MTG種別は hq_meeting_slots.mode（online／phone／visit）＋別途 sys_params で論理種別を持つ：

- `routine_checkin` … 定例運営相談（1:1、online）
- `trouble_consult` … トラブル／クレーム相談（複数 HQ参加可）
- `fc_contract` … 契約・ロイヤリティ・条件変更系
- `marketing_planning` … 集客／キャンペーン相談
- `phone_light` … 軽め電話相談（10〜15分）

各種別に対して：

- `booking_mode` … `direct_slot` / `request_and_confirm`
- `min_hq_staff` … 1 / 2 / 3

これらは sys_params に JSON で格納する（詳細は DEFAULT_PARAMS 側で定義）。

### 8.2 スロット管理（hq_meeting_slots）

db_fields.yaml 定義：slot_start／slot_end／mode／capacity／status 等。

[Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)

- 外部カレンダーサービス（Google Calendar 等）との連携は Phase9 で詳細化するが、
    
    本フェーズでは「本部が公開してよい相談枠」として slots を管理する。
    
- `status='available'` の slots のみ SHOP_PORTAL から予約対象とする。

### 8.3 SHOP_PORTAL 側フロー（参照）

- `booking_mode='direct_slot'`
    - SHOP_PORTAL から `hq_meeting_slots` を取得し、カレンダー形式で表示
    - オーナーが枠を選択 → hq_meetings を `status='confirmed'` で作成
- `booking_mode='request_and_confirm'`
    - SHOP_PORTAL では「希望日時フォーム」を出し、hq_meetings を `status='requested'` で作成
    - HQ は AI-INBOX の MTG画面で slots を確認し、手動で `slot_id` をアサインして `status='confirmed'` に変更
        
        → SHOP_PORTAL に確定日時を通知
        

### 8.4 AI-INBOX 側の連携

- チケット詳細画面／owner_ai 対応画面から「MTG提案」ボタンを提供
    - meeting_type を選択
    - booking_mode に応じて：
        - direct_slot → SHOP_PORTAL の MTG予約画面への deep link を生成し、AI／HQメッセージで送信
        - request_and_confirm → hq_meetings を `requested` で作成し、オーナーには「希望日時入力フォーム」への link を案内

### 🔧 パラメータ候補（MTG／電話）

- `hq_mtg.default_visible_days`（初期値: 30）
- `hq_mtg.max_requests_per_month_per_shop`（初期値: 3）
- `hq_mtg.ai_suggestable_types`（初期値: `['routine_checkin','trouble_consult','fc_contract']`）
- `hq_mtg.enable_phone_light`（初期値: true）

---

## 9. オーナー単位タイムラインビュー

### 9.1 目的

- 特定オーナーについて「最近何が起きているか」を一目で把握するための時系列ビューを提供する。

### 9.2 対象イベント

- tickets（owner配下の全 shop）
- hq_meetings（MTG）
- invoices（請求）
- shop_supplies（備品発注）
- hq_requests／campaign_requests／design_requests 等の申請
- リスクレベル変更（shop_internal_info の更新：audit_logs から抽出）
    
    [Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)
    

### 9.3 実装方針

- 追加のテーブルは作らず、owners.id をキーに各テーブルからイベントを UNION し、AI-INBOX側で整形して表示する。
- イベント種別ごとに色／アイコンで区別する。

---

## 10. パラメータ変更履歴ビュー

### 10.1 対象

- sys_params／shop_params／shop_options
- AI設定（ai_settings／ai_core_params／ai_tone_policies 等）

### 10.2 ログの扱い

- 既存の audit_logs テーブルを利用する。
    
    [Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)
    
    - entity: 'sys_params'／'shop_params'／'shop_options'／'ai_settings' 等
    - entity_id: 対象レコードID（または key 名）
    - before_value／after_value: 変更前後の JSON
- Phase5 以降、これらのパラメータ更新処理は **必ず audit_logs に1レコード書き込む** ことを要求仕様とする。

### 10.3 画面

- システム全体の履歴一覧
- 店舗詳細画面から「この店舗に関するパラメータ履歴のみ」をフィルタ表示

---

## 11. HQタグ（ticket_tag_master／ticket_tags）

### 11.1 目的

- intent／category だけでは表現しづらい「HQ内部用の分類」（例：RemoteLOCK案件／解約検討／法務確認など）を、柔軟に付与できるようにする。

### 11.2 DDL

```sql
CREATE TABLE ticket_tag_master (
  id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  tag_key VARCHAR(64) NOT NULL UNIQUE COMMENT '内部キー: remote_lock, legal, resignation など',
  label VARCHAR(255) NOT NULL COMMENT '表示名: RemoteLOCK, 法務確認 など',
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE ticket_tags (
  ticket_id INT(11) NOT NULL,
  tag_id INT(11) NOT NULL,
  PRIMARY KEY (ticket_id, tag_id),
  CONSTRAINT fk_ticket_tags_ticket
    FOREIGN KEY (ticket_id) REFERENCES tickets(id),
  CONSTRAINT fk_ticket_tags_tag
    FOREIGN KEY (tag_id) REFERENCES ticket_tag_master(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 11.3 画面

- チケット詳細画面で、tag_master から複数選択。
- INBOX一覧にタグ列（ラベル）を表示。
- フィルタ条件としてタグを指定可能。

### 🔧 パラメータ候補（HQタグ）

- `ai_inbox.tag_edit_role`（初期値: hq_admin／hq_operator）
- `ai_inbox.max_tags_per_ticket`（初期値: 5）

---

## 12. セキュリティ・権限・ログ

### 12.1 認証・権限

- 認証は既存 `portal_users` テーブルを利用する。
    
    [Add files via upload - [Repo na…](https://github.com/gbizconhygi-hub/self-datsumou-specs/commit/3357de36ba82067bf68b322ef889dd77d150214f)
    
    - `role='admin'` → HQユーザ（ai.／shop.管理者）
    - `role='owner'` → オーナー（shop. ログイン）
- AI-INBOX は `role='admin'` のみアクセス可能とし、将来 `portal_users` にロール追加が必要な場合は Phase0/1 側で差分定義する。

### 12.2 ログ・監査

- 重要操作（請求書発行／請求書金額の手修正／リスクレベル変更／パラメータ変更など）は audit_logs に記録。
- tickets／ticket_messages／ai_turns／sys_events の保持期間・匿名化方針は Phase1/2 のポリシーに従う。

---

## 13. 🔧 パラメータ化ポリシー（Phase5補足）

本フェーズで新たに導入されるパラメータは、原則として：

- sys_params（scope=global／shop／channel）に格納し、
- AI-INBOXから編集可能とする（編集権限は hq_admin が基本）

主要カテゴリ：

- INBOX／SLA／アラート関連
- AIアシスト／FAQ候補フロー
- MTG・電話相談枠
- 請求書（billing_profile ごとの挙動／自動積み上げ項目のON/OFF）
- HQタグ関連
- タイムライン／履歴ビューの期間・件数

詳細なキー一覧・初期値は `AI_INBOX_DEFAULT_PARAMS.md` にて定義する。

---

## 14. 未決課題／次フェーズへの引き継ぎ

### 14.1 Phase5内での未決課題

1. 開業月の「2か月分＋日割り」ロジックの自動化レベル
    - 自動計上をどこまで行い、どこからを「HQの手修正前提」とするかの閾値
2. incident グルーピングの UX 詳細
    - 自動候補の提示条件（どの程度類似していれば候補に出すか）
3. 半直営（fc_semi_manual）の請求運用
    - 現時点では一括発行対象外とするが、将来「部分自動化」する際のシナリオ

### 14.2 他フェーズへの引き継ぎ

- KPI_ANALYTICS フェーズ
    - AI-INBOXで集約された tags／incident_key／SLA 情報を用いたKPI可視化
- Phase7：OP_MAIL
    - `tickets.source='portal_admin'` かつ OP_MAILカテゴリの生成ロジック（メール解析）
- Phase9：拡張統合層
    - hq_meeting_slots／hq_meetings と Google Calendar 等の双方向連携
    - STORES売上の完全自動取込と `sales_records`／invoices の自動連動

---

以上をもって、PHASE_05_AI_INBOX の v1.0 Draft とする。
