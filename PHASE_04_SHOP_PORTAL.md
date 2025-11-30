# PHASE_04_SHOP_PORTAL - 仕様設計書 (v1.0 Draft)

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 4：SHOP_PORTAL）は、

`shop.self-datsumou.net`（オーナーポータル／加盟店向けサポート UI）の仕様を定める。

- 主体は **owners（オーナー）** であり、
1オーナーに紐づく複数の店舗（shops）がぶら下がる構造とする。
- SHOP_PORTAL は、以下を 1 つの UI 群として提供する。
    - オーナー単位の状況把握（お知らせ／請求／売上サマリ／店舗ごとの問い合わせサマリ）
    - 店舗単位の運営・サポート UI（顧客問い合わせ対応／備品発注／各種申請／簡易分析）
    - 本部とのやり取り（AI相談／MTG予約／キャンペーン／デザイン依頼）

### 1.2 本フェーズで実現する主な機能

**オーナーレベル（オーナーダッシュボード）**

- 本部からのお知らせ（アナウンス）を受け取る
- オーナー情報（契約名義・請求先等）は「初期入力時のみ」オーナー編集可
→ 完了後の編集権限は HQ のみ
- ロイヤリティ等の **当月請求書発行状況・過去請求書閲覧**
- 売上サマリー（全店舗合算）
- 店舗ごとの問い合わせ状況サマリー（open/pending/closed 件数など）

**店舗レベル（店舗ダッシュボード）**

- 顧客からの問い合わせ（help./line. 経由）の「要対応分」の受付・返信
- 店舗ごとの基本契約・オプション契約情報の閲覧（編集不可）
- 本部が販売する備品発注
→ 発注分は当月のロイヤリティ請求に自動加算される前提
- 本部への各種申請（キャンペーン作成依頼、デザイン依頼 など）
→ 金額がかかるものは HQ が請求書にプラスオン
- 売上データからの簡易分析
（お試し体験獲得数、メニューごとの購入数など）

### 1.3 対応デバイスとレイアウト方針

- **メインターゲット：PC／タブレット（横幅 1024px 前後）**
    - 左サイドバーにオーナーレベルのメニュー
    - メインペインに店舗ダッシュボードや一覧・詳細
- **スマホ（縦長）もレスポンシブ対応**
    - 下部タブバーやドロップダウンでオーナーメニューを切り替え
    - 「オーナーダッシュボード → 店舗一覧 → 店舗詳細」の階層構造は PC と共通

### 1.4 UI デザイン・カラー方針（ブランド準拠）

- SHOP_PORTAL の UI デザイン（カラー／雰囲気）は、**ハイジ公式ページ（[https://self-datsumou.com](https://self-datsumou.com/)）とブランド整合を取ることを前提とする**。
- 方針レベルでの決定事項：
    - ベースカラー：公式サイトと同系統の「明るく清潔感のある白／淡色系」を採用
    - アクセントカラー：公式サイトのブランドカラー（ロゴ・ボタン色）を踏襲し、
    SHOP_PORTAL でも「ハイジらしさ」が一目でわかる配色とする
    - 禁止事項：
        - SHOP_PORTAL 独自で、ブランドコンセプトと乖離するダークテーマ／原色多用を導入しない
        - 本番環境では公式ブランドガイドラインに反する独自色の上書きを行わない
- 具体的な色コード・フォント指定は、別途「ブランドガイドライン／UIデザインガイド」にて管理し、
本仕様書では **「公式ページのブランドに準拠する」という原則レベルの宣言** にとどめる。

---

## 2. 前フェーズからの継承ポイント（抜粋）

詳細は Phase0〜3 の仕様書および `db_fields.yaml` を SoR とし、本フェーズでは要点のみ記載する。

### 2.1 ドメイン構成と役割

- help.：顧客向け Web チュートリアル（AI＋メール）
- line.：顧客向け LINE チュートリアル（AI＋LIFF／リッチメニュー）
- **shop.：加盟店ポータル（オーナー向けステータス／請求／AI相談／申請）**
- ai.：本部ポータル（AI-INBOX／AIチューニング／KPI）

### 2.2 コア層との関係

- SYS_CORE：sys_params／api_call_logs／sys_events／tickets／通知コア 等の共通基盤
- API_COMM：STORES予約 API などを業務 API としてラップ
- AI_CORE：help./line./shop./ai. 共通の AI 頭脳
    - SHOP_PORTAL では `channel="shop"`, `user_role="owner"` を標準とする

### 2.3 DB BOX（BOX_INTENT）

SHOP_PORTAL が主に利用する BOX：

- owners／owner_contacts（オーナー単位情報）
- shops／shop_secrets／shop_links／shop_internal_info
- options_master／shop_options
- sys_params（scope=global/shop）
- sales_records（売上）
- invoices／invoice_items（請求）
- tickets／ticket_messages（問い合わせ）
- ai_sessions／ai_turns（AI 会話ログ）
- hq_meeting_slots／hq_meetings（本部 MTG枠／予約）
- supply_items／shop_supplies（備品マスタ／発注）
- hq_requests（各種申請；本フェーズで新規追加）
- campaign_requests／campaign_banners／banner_templates（キャンペーン・バナー）
- design_requests（デザイン依頼 DR）

---

## 3. 新規決定事項（SHOP_PORTAL 固有仕様）

### 3.1 情報構造・ロール

- 主軸は **owner → shops** の階層構造
    - ログイン単位：owners（＋必要に応じて owner_contacts）
    - 1 オーナーに 1..N 店舗が紐づく
- ロールと権限：
    - SHOP ロール（オーナー／店舗担当者）：
        - 自分の owner_id に紐づく shop のみ閲覧・操作可
    - HQ ロール（本部）：
        - 全 owner／shop に対して横断的に閲覧・編集可（別 UI：ai. 側）

### 3.2 機能モジュール（骨）

**オーナーレベル**

1. オーナーダッシュボード
    - 本部お知らせ（hq_announcements）
    - オーナー情報（初期入力ウィザード＋完了後は閲覧のみ）
    - 当月／過去の請求サマリ
    - 店舗別問い合わせサマリ
2. 請求・売上一覧
    - invoices／invoice_items（オーナー全体の請求）
    - sales_records（全店舗合算＋フィルタ）
3. オーナー向け AI 相談
    - `AI_CORE_REQUEST.channel="shop", user_role="owner"`
4. 本部 MTG 予約
    - hq_meeting_slots／hq_meetings を利用した枠選択・予約

**店舗レベル**

1. 店舗ダッシュボード
    - 店舗別の問い合わせサマリ／簡易売上サマリ
2. 顧客問い合わせ（要対応分）
    - `tickets.shop_id = 店舗` かつ `status in (open, pending)` の一覧＆返信
3. 店舗契約・オプション情報（閲覧のみ）
    - shops／options_master＋shop_options
4. 備品発注
    - supply_items → shop_supplies への発注
    - 月次請求生成処理で invoices／invoice_items に取り込み
5. 各種申請（hq_requests）
    - キャンペーン申請・運営系申請・その他依頼
6. デザイン依頼（design_requests）
    - テンプレバナー（デザイナーA）／新規クリエイティブ（デザイナーB）を含む
7. 簡易売上分析
    - お試し体験数／メニュー別売上など

### 3.3 オーナー情報編集ポリシー

- `owners.is_profile_complete=false` の間のみ SHOP_PORTAL から編集可能
- `is_profile_complete=true` に遷移した後は、SHOP_PORTAL 上は閲覧のみ
- 再編集が必要な場合：
    - SHOP_PORTAL からは「修正申請」を出すのみ（hq_requests）
    - 実際の更新操作は HQ 側 UI から行う

### 3.4 顧客問い合わせの扱い

- help./line. で AI が一次対応し、エスカレーション条件を満たした問い合わせは tickets に登録される
- SHOP_PORTAL は `tickets.shop_id = 該当店舗` のうち、
「店舗判断で対応してよいもの」を一覧化
- クレーム／賠償／FC契約などの危険カテゴリは AI_CORE／HELP_UI 側で HQ に直行させる前提
（詳細ルールは AI_CORE／HELP_UI 側仕様に準拠）

### 3.5 備品発注フロー（骨）

- 店舗ダッシュボードから supply_items の一覧を表示し、数量入力→発注
- 発注は `shop_supplies` に記録
    - 発注ごとに `ordered_at`, `unit_price`, `quantity`, `status`
- 月次請求生成処理で、`status='ordered'` かつ締め日以前のものを集計し、
invoices／invoice_items に加算する

### 3.6 各種申請（hq_requests）

- SHOP_PORTAL の「申請」メニューから起票
    - request_type（campaign_plan / signage_change / design_request / other など）
    - shop_id／owner_id／内容
- `hq_requests` テーブルで状態管理：
    - status：requested / in_progress / need_confirm / completed / cancelled
- 金額が発生する申請は HQ 側で actual_cost／offer_price を設定し、
invoices／invoice_items に反映

### 3.7 デザイン依頼フロー（キャンペーン＋DR）

デザイン系は以下の二層構造で扱う。

1. **キャンペーン申請（campaign_requests）＋テンプレ／デザイナーA**
2. **デザイン依頼 DR（design_requests）＋デザイナーB**

骨となる方針：

- 金額・請求は必ず SHOP_PORTAL 上で HQ が握る
- デザイナーはオーナーに対して **一切金額を伝えない**
- Chatwork は「制作・ファイルやりとり」の裏方チャネル
- **DR-ID のない仕事は正式案件として扱わない**

（詳細は別章「4.3 デザイン関連テーブル／3.7.x デザイン依頼フロー」で定義）

### 3.8 🔧 パラメータ候補（SHOP_PORTAL 全体）

パラメータ自体は `SHOP_PORTAL_DEFAULT_PARAMS.md` 側で管理し、ここではキーと意味だけ示す。

- `shop_portal.dashboard_default_range_days`
ダッシュボードでデフォルト表示する期間（日数）
- `shop_portal.announcement_max_items`
表示するお知らせ件数上限
- `shop_portal.tickets_page_size`
問い合わせ一覧の 1ページあたり件数
- `shop_portal.analytics_default_range_days`
簡易分析のデフォルト期間
- `shop_portal.supply_order_cutoff_day`
月度請求に含める備品発注の締め日
- `shop_portal.owner_ai_max_turns_before_escalation_suggest`
オーナーAI相談で HQ エスカレーションを提案するまでのターン数
- `shop_portal.owner_ai_max_turns_before_force_escalation`
自動エスカレーションするターン数
- `shop_portal.meeting_slots_visible_days_ahead`
MTG スロットを何日先まで見せるか
- `shop_portal.design.dr_code_prefix` / `shop_portal.design.cr_code_prefix`
DR-ID／CR-ID のプレフィックス
- `shop_portal.design_designerA_offer_price_zero`
デザイナーA案件を常にオーナー請求なし（offer_price=0）とするか

---

## 4. データ構造 / I/O 定義

### 4.1 既存テーブルの利用（概要）

- owners／owner_contacts／shops／shop_*／options_master／shop_options／sys_params
→ オーナー／店舗／設定の SoR
- sales_records／invoices／invoice_items
→ 売上・請求
- tickets／ticket_messages
→ 顧客・オーナー・HQ すべての問い合わせ器
- ai_sessions／ai_turns
→ AI 会話ログ（owner AI 相談もここに統合）
- hq_meeting_slots／hq_meetings
→ 本部 MTG 枠・予約

### 4.2 新規テーブル（Phase4 提案）

### 4.2.1 HQ アナウンス

```sql
CREATE TABLE hq_announcements (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(191) NOT NULL,
  body TEXT NOT NULL,
  level ENUM('info','warning','critical') NOT NULL DEFAULT 'info',
  target_scope ENUM('all','owner','shop') NOT NULL DEFAULT 'all',
  target_owner_id BIGINT UNSIGNED NULL,
  target_shop_id BIGINT UNSIGNED NULL,
  published_at DATETIME NOT NULL,
  expires_at DATETIME NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 4.2.2 各種申請（hq_requests）

```sql
CREATE TABLE hq_requests (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  owner_id BIGINT UNSIGNED NOT NULL,
  shop_id BIGINT UNSIGNED NULL,
  request_type VARCHAR(64) NOT NULL COMMENT 'campaign_plan / signage_change / design_request など',
  title VARCHAR(191) NOT NULL,
  description TEXT NOT NULL,
  estimated_cost INT NULL COMMENT '見積金額（税抜）',
  actual_cost INT NULL COMMENT '確定金額（税抜）',
  status ENUM('requested','in_progress','need_confirm','completed','cancelled') NOT NULL DEFAULT 'requested',
  created_by_role ENUM('SHOP','HQ') NOT NULL DEFAULT 'SHOP',
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  CONSTRAINT fk_requests_owner FOREIGN KEY (owner_id) REFERENCES owners(id),
  CONSTRAINT fk_requests_shop FOREIGN KEY (shop_id) REFERENCES shops(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 4.2.3 キャンペーン／バナー関連

```sql
CREATE TABLE campaign_requests (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  cr_code VARCHAR(32) NOT NULL UNIQUE COMMENT '人間可読ID: CR-YYYY-NNNN',
  owner_id BIGINT UNSIGNED NOT NULL,
  shop_id BIGINT UNSIGNED NOT NULL,
  campaign_name VARCHAR(191) NOT NULL,
  target_menu_codes JSON NULL,
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  media_json JSON NULL COMMENT 'X, Instagram, POP など',
  template_code VARCHAR(64) NOT NULL COMMENT 'banner_templates.template_code',
  copy_text TEXT NULL,
  status ENUM(
    'draft','submitted','needs_revision','resubmitted',
    'approved','banner_in_production','banner_completed','cancelled'
  ) NOT NULL DEFAULT 'draft',
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  CONSTRAINT fk_campaign_owner FOREIGN KEY (owner_id) REFERENCES owners(id),
  CONSTRAINT fk_campaign_shop FOREIGN KEY (shop_id) REFERENCES shops(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

```sql
CREATE TABLE campaign_request_comments (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  campaign_request_id BIGINT UNSIGNED NOT NULL,
  author_role ENUM('OWNER','HQ') NOT NULL,
  author_id BIGINT UNSIGNED NULL,
  body TEXT NOT NULL,
  created_at DATETIME NOT NULL,
  CONSTRAINT fk_campaign_comment FOREIGN KEY (campaign_request_id)
    REFERENCES campaign_requests(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

```sql
CREATE TABLE banner_templates (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  template_code VARCHAR(64) NOT NULL UNIQUE,
  name VARCHAR(191) NOT NULL,
  description TEXT NULL,
  asset_id BIGINT UNSIGNED NOT NULL,
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

```sql
CREATE TABLE campaign_banners (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  campaign_request_id BIGINT UNSIGNED NOT NULL,
  variant_type ENUM('template_only','designer_A','designer_B') NOT NULL,
  template_code VARCHAR(64) NULL,
  asset_id BIGINT UNSIGNED NOT NULL,
  notes TEXT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  CONSTRAINT fk_campaign_banners FOREIGN KEY (campaign_request_id)
    REFERENCES campaign_requests(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 4.2.4 デザイン依頼（DR）

```sql
CREATE TABLE design_requests (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  dr_code VARCHAR(32) NOT NULL UNIQUE COMMENT 'DR-YYYY-NNNN',
  owner_id BIGINT UNSIGNED NOT NULL,
  shop_id BIGINT UNSIGNED NOT NULL,
  origin_type ENUM('campaign_banner_A','new_creative','other') NOT NULL,
  campaign_request_id BIGINT UNSIGNED NULL,
  designer_type ENUM('A','B') NOT NULL,
  title VARCHAR(191) NOT NULL,
  description TEXT NOT NULL,
  status ENUM(
    'draft','submitted','estimate_pending','estimate_proposed',
    'estimate_rejected','approved','in_production','in_revision',
    'delivered','completed','cancelled'
  ) NOT NULL DEFAULT 'draft',
  internal_cost INT NULL COMMENT 'デザイナーへの原価（税抜）',
  offer_price INT NULL COMMENT 'オーナーへの提示価格（税抜）',
  price_status ENUM('none','estimate_proposed','approved','rejected')
    NOT NULL DEFAULT 'none',
  asset_id BIGINT UNSIGNED NULL COMMENT '最終納品ファイル',
  chatwork_room_id VARCHAR(64) NULL COMMENT 'オーナー／デザイナーBルーム 等',
  chatwork_root_message_id VARCHAR(64) NULL COMMENT '案件スレッドの起点メッセージID',
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  CONSTRAINT fk_dr_owner FOREIGN KEY (owner_id) REFERENCES owners(id),
  CONSTRAINT fk_dr_shop FOREIGN KEY (shop_id) REFERENCES shops(id),
  CONSTRAINT fk_dr_campaign FOREIGN KEY (campaign_request_id)
    REFERENCES campaign_requests(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

```sql
CREATE TABLE design_request_comments (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  design_request_id BIGINT UNSIGNED NOT NULL,
  author_role ENUM('OWNER','HQ') NOT NULL,
  author_id BIGINT UNSIGNED NULL,
  body TEXT NOT NULL,
  created_at DATETIME NOT NULL,
  CONSTRAINT fk_dr_comment FOREIGN KEY (design_request_id)
    REFERENCES design_requests(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

---

## 5. フロー / 画面遷移

### 5.1 ログイン〜店舗選択

1. owner が shop. にアクセスし、メールアドレス＋パスワード等でログイン
2. JWT に `owner_id`, `role='SHOP'`, `shop_ids` を付与
3. shop_ids が 1 件なら、その店舗を前提にオーナーダッシュボードへ
4. 複数店舗の場合は「店舗一覧」からメイン店舗を選択（後から切替可能）

### 5.2 オーナーダッシュボード

1. hq_announcements から未読お知らせを取得し、上部カードで表示
2. 当月請求／売上サマリを集計表示
3. 店舗ごとの問い合わせサマリ（open/pending/closed 件数）を一覧表示
4. 店舗名クリックで店舗ダッシュボードへ遷移

### 5.3 店舗ダッシュボード（shop 単位）

1. `tickets.shop_id = shop_id` のサマリ（open/pending/closed）
2. 「顧客問い合わせ」タブ：
    - 要対応 tickets 一覧
    - チケット詳細では、顧客メッセージ・AI 要約・店舗／HQ 返信をタイムライン表示
3. 「備品発注」タブ：
    - supply_items 一覧から数量入力→発注→shop_supplies に登録
4. 「申請・デザイン」タブ：
    - hq_requests／campaign_requests／design_requests を一覧表示
    - 新規キャンペーン申請／デザイン依頼フォーム起票
5. 「分析」タブ：
    - 簡易売上分析指標を期間選択で表示

### 5.4 デザイン依頼フロー（要約）

- デザイナーA（テンプレ）：
    - キャンペーン申請（campaign_requests）→ HQ 承認 →
        
        テンプレ適用 or デザイナーA依頼（design_requests type='campaign_banner_A'）
        
    - Chatwork 本部／デザイナーAルームで制作・納品
    - 最終バナーは campaign_banners に紐づけ、SHOP_PORTAL から配布
- デザイナーB（新規クリエイト）：
    - デザイン依頼フォーム（design_requests origin_type='new_creative'）→ DR-ID 発行
    - HQ が internal_cost／offer_price 設定 → オーナーに見積提示 → 承認
    - Chatwork オーナー／デザイナーB（＋本部）ルームで仕様詰め～校正
    - 納品ファイルを assets へアップロード → design_requests.asset_id に紐づけ
    - offer_price を月次請求で invoices／invoice_items に反映

---

## 6. 通知 / 外部連携

### 6.1 Chatwork 連携

- デザイナーA：
    - design_requests 作成時：本部／デザイナーAルームに「DR-ID＋キャンペーン情報」を自動投稿
- デザイナーB：
    - design_requests が approved になったタイミングで、オーナー／デザイナーBルームに
        
        「【DR-XXXX】新規案件が承認されました」と投稿
        

### 6.2 AI チャット連携（オーナー向け）

- 以下のイベント時に、オーナー向け AI チャットへシステムメッセージを送信：
    - 新しいお知らせ（重要度が高いもの）
    - キャンペーンバナー完成
    - デザイン依頼の見積提示・納品完了
    - HQ から案件コメント（campaign_requests／design_requests コメント）追加

### 6.3 メール／その他通知

- 請求書発行／MTG 確定／重要な申請ステータス変更などは、
    
    billing_email／owner_contacts.email を宛先にメール通知
    
    （通知ルール詳細は SYS_CORE 通知仕様に従う）
    

---

## 7. セキュリティ・権限・ログ

### 7.1 権限

- SHOP ロール：
    - JWT に含まれる owner_id／shop_ids の範囲のみ操作可能
    - 他オーナー／他店舗のデータにはアクセス不可
- HQ ロール：
    - ai. 側 UI で全体を横断閲覧可能（SHOP_PORTAL とは別 UI）

### 7.2 PII／ログポリシー

- 顧客の氏名は、店舗が既に持っている情報として表示可
- メールアドレス／電話番号はマスク済み表現（末尾4桁のみなど）
- AI ログ（ai_turns.raw_text）は SHOP_PORTAL では直接閲覧不可
    
    → 要約のみ表示し、詳細は HQ 用 AI-INBOX で扱う
    
- 重要イベント（発注／申請／価格承認／請求生成など）は sys_events にイベント記録

---

## 8. 未決課題 / 次フェーズへの引き継ぎ

- クレーム系問い合わせを「どこまで店舗に直接見せるか」の最終ライン
- 簡易分析の指標範囲（最小セットは本仕様の通りだが、追加指標は KPI フェーズと調整）
- hq_requests と design_requests／campaign_requests の責務境界（将来の統合・整理の余地）
- ブランドガイドライン（カラーコード／フォント／コンポーネント）の別途明文化
    
    → Phase4 では「公式サイト準拠」という原則だけ定義し、具体ガイドは UI デザイン側で管理
